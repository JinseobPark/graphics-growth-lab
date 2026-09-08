---
title: "GPU-Driven Meshlet Visibility Pipelines: Hierarchical Culling, Worklist Compaction, and Indirect Count Synchronization"
date: "2026-09-08"
category: Graphics
tags: [GPU, Rendering, GPU-Driven Rendering, Meshlet, Mesh Shader, Task Shader, Visibility Culling, Hi-Z, Occlusion Culling, Indirect Draw, Stream Compaction, Vulkan, CUDA, Memory Layout, C++]
level: intermediate
---

# [Daily Graphics Growth] 2026-09-08 - GPU-Driven Meshlet Visibility Pipelines: Hierarchical Culling, Worklist Compaction, and Indirect Count Synchronization

## 1. 오늘의 개념

어제는 **Relocation-Safe GPU-Driven Rendering**에서 persistent logical handle과 physical placement를 분리하고, meshlet table과 indirect command가 동일한 relocation/render snapshot을 보도록 만드는 방법을 봤다.

오늘은 그 안정적인 meshlet table을 실제 rendering workload 감소로 연결한다.

> **수십만~수백만 개의 meshlet 후보 중 rasterization 또는 mesh shader 실행 가치가 있는 work만 GPU에서 계층적으로 걸러내고, compact한 worklist와 indirect count로 graphics pipeline에 넘기는 구조는 어떻게 설계해야 하는가?**

GPU-driven visibility pipeline의 목적은 단순히 “triangle을 많이 지우는 것”이 아니다. 더 정확한 목표는 다음과 같다.

- CPU가 개별 draw를 판단하지 않게 한다.
- 비싼 fine-grained test 전에 cheap coarse test를 수행한다.
- invisible work가 vertex/mesh shader, primitive setup, rasterization까지 내려가지 않게 한다.
- culling 결과를 GPU 내부의 **compacted worklist**로 바꾼다.
- worklist와 indirect command/count가 같은 frame snapshot을 보장하게 한다.
- conservative test를 사용해 **false negative**, 즉 실제 visible geometry를 잘못 제거하는 상황을 피한다.

Mesh shader pipeline에서는 optional task shader가 coarse work reduction과 amplification에 자연스럽게 맞는다. Khronos의 Vulkan Mesh Shader Culling sample도 task shader가 몇 개의 mesh shader workgroup을 launch할지 결정하고 payload를 전달하는 역할을 보여준다.

하지만 task shader가 항상 최적이라는 뜻은 아니다. 큰 scene hierarchy, async compute, multi-pass Hi-Z와 결합할 때는 별도 compute culling pass가 더 유연할 수 있다.

오늘의 핵심은 API stage보다 **visibility decision을 어떤 granularity와 memory flow로 배치하는가**다.

---

## 2. 한 줄 핵심

> GPU-driven meshlet visibility의 핵심은 **instance/chunk → meshlet → primitive 순으로 test 비용을 높이는 hierarchical rejection, subgroup/scan 기반 compact work generation, conservative bounds, 그리고 compute-write → indirect/task/mesh-read 사이의 정확한 synchronization을 하나의 pipeline contract로 설계하는 것**이다.

---

## 3. 왜 중요한가

GPU 성능 문제를 triangle count 하나로 설명하기 어려운 이유는 **실제로 실행된 work의 위치**가 다르기 때문이다.

같은 1M triangle이라도 다음 두 상황은 비용 구조가 다르다.

### 상황 A — coarse 단계에서 대부분 제거

- 100k meshlet candidate
- 90k가 instance/chunk 또는 meshlet bound test에서 제거
- 10k만 mesh shader/rasterization 진입

### 상황 B — rasterization 직전까지 모두 진행

- 100k meshlet candidate
- mesh shader가 전부 실행
- primitive 또는 depth test에서 마지막에 대부분 제거

최종 visible pixel 수는 비슷해도 B는 geometry fetch, shader invocation, primitive assembly, cache traffic을 더 많이 소비한다.

따라서 좋은 visibility system의 질문은

> **“얼마나 많이 cull했는가?”**

보다

> **“어떤 work를 어느 stage 이전에 제거했는가?”**

에 가깝다.

또 GPU-driven pipeline에서는 CPU가 culling 결과를 읽지 않아야 한다. CPU readback이 들어가면 GPU의 work generation 장점이 약해진다.

그래서 현대적인 흐름은 대체로 다음과 같다.

```text
Scene / Chunk Candidates
        ↓
Coarse Visibility
        ↓
Meshlet Visibility
        ↓
Compaction
        ↓
Indirect Command + Count
        ↓
Task/Mesh Shader or Indexed Draw
        ↓
Rasterization
```

여기서 culling, compaction, indirect execution이 모두 GPU 내부에서 연결된다.

### 3.1 False positive와 false negative는 비용이 다르다

Visibility culling에서 두 종류의 오판을 구분해야 한다.

**False positive**
- 실제 invisible인데 visible로 통과시킴
- 성능 손해
- 최종 rendering correctness는 유지

**False negative**
- 실제 visible인데 invisible로 제거
- 화면에서 geometry가 사라짐
- correctness 실패

그래서 bounding sphere/AABB quantization, cone culling, Hi-Z occlusion은 대체로 **conservative rejection**을 목표로 한다.

Graphics engineer 관점에서는 “culling accuracy”보다

> **rejection rule이 visible set의 superset을 유지하는가**

라는 표현이 더 정확하다.

---

## 4. 구현 관점

### 4.1 Visibility pipeline은 cost ladder로 본다

모든 meshlet에 가장 정확한 occlusion test를 처음부터 수행하는 것은 낭비일 수 있다.

일반적인 cost ladder는 다음처럼 생각할 수 있다.

```text
Cheap / Coarse
    ↓
Generation validity
    ↓
Instance or chunk frustum
    ↓
LOD / projected-size test
    ↓
Meshlet frustum
    ↓
Normal-cone backface test
    ↓
Hi-Z occlusion
    ↓
Optional primitive culling
Expensive / Fine
```

핵심 원칙은:

> **비용이 낮고 rejection power가 높은 test를 앞에 둔다.**

다만 test 순서는 scene 특성에 따라 달라진다.

예를 들어 semiconductor cross-section viewer처럼 거의 모든 geometry가 camera frustum 안에 있고 occlusion이 강하다면 frustum보다 Hi-Z의 중요도가 올라간다.

반대로 CAD-style exploded view처럼 occlusion이 적고 object가 넓게 퍼져 있다면 coarse frustum/chunk rejection이 더 중요할 수 있다.

### 4.2 두 단계 hierarchy: chunk/instance → meshlet

어제의 relocation-safe 구조와 자연스럽게 연결하면 다음 logical hierarchy가 된다.

```text
Surface / Mesh Handle
        ↓
Chunk Record
        ↓
Meshlet Range
        ↓
Meshlet Record
```

각 chunk에 다음과 같은 coarse metadata가 있다고 생각할 수 있다.

```text
ChunkCullData
- bounding sphere or AABB
- meshletOffset
- meshletCount
- LOD error
- generation
- visibility flags
```

먼저 chunk가 invisible이면 해당 meshlet range 전체를 읽지 않는다.

이것은 단순 ALU 절약 이상이다.

**meshlet descriptor memory traffic 자체를 제거**하기 때문이다.

Sparse SDF에서 brick 단위로 mesh를 생성한다면 brick/chunk hierarchy가 이미 존재하므로, 별도의 scene hierarchy를 만들지 않고도 coarse culling unit으로 재사용할 수 있다.

### 4.3 Bounds는 tightness와 update cost의 trade-off다

대표적인 bounding volume은 다음과 같다.

#### Bounding Sphere

장점:
- transform이 단순
- frustum test가 저렴
- rotation에 안정적

단점:
- 긴/평평한 geometry에서 loose할 수 있음

#### AABB

장점:
- axis-aligned geometry에 tight
- Hi-Z screen rect 계산과 연결하기 쉬움

단점:
- object transform이 있으면 world-space AABB update 비용이 존재
- 회전 geometry에는 loose해질 수 있음

#### OBB

장점:
- 더 tight할 수 있음

단점:
- test 비용과 metadata가 증가

GPU-driven culling에서는 가장 tight한 bound가 항상 좋은 것은 아니다.

bound fetch bytes, transform ALU, rejection rate를 함께 봐야 한다.

Dynamic SDF mesh에서는 geometry가 자주 변하므로 **cheap-to-update conservative bound**가 특히 중요하다.

### 4.4 Quantized bound는 반드시 outward-conservative해야 한다

Meshlet bound를 FP16 또는 packed integer로 줄이면 culling metadata bandwidth가 감소한다.

하지만 단순 nearest rounding으로 box가 실제 geometry보다 작아지면 false negative가 가능하다.

따라서 quantization의 의미는 보통 다음처럼 잡는 편이 안전하다.

```text
min bound → geometry 바깥 방향으로 round down
max bound → geometry 바깥 방향으로 round up
sphere radius → round up
```

즉 culling metadata compression에서도 이전 SDF Lipschitz bound와 같은 원칙이 반복된다.

> **conservative quantity는 rounding direction까지 semantic contract의 일부다.**

### 4.5 Frustum culling은 싸지만 hierarchy에 따라 효과가 달라진다

Bounding sphere에 대한 6-plane frustum test는 비교적 저렴하다.

각 plane `p=(n,d)`에 대해 sphere center `c`, radius `r`가 완전히 바깥이면 reject할 수 있다.

개념적으로:

`dot(n, c) + d < -r`

인 plane이 하나라도 있으면 invisible이다.

하지만 100만 meshlet에 직접 6-plane test를 하면 그것도 큰 비용이다.

그래서:

```text
Instance/Chunk Frustum
        ↓ pass
Meshlet Frustum
```

처럼 hierarchy를 둔다.

coarse parent가 reject되면 child test를 생략할 수 있다.

이때 parent bound는 child geometry 전체를 반드시 포함해야 한다.

### 4.6 Normal Cone은 cluster-level backface rejection이다

Meshlet의 triangle normal들이 비슷한 방향을 가진다면 normal cone으로 cluster 전체가 back-facing인지 근사할 수 있다.

Meshlet descriptor에 다음 정보를 둘 수 있다.

```text
cone axis
cone half-angle or cutoff
```

view direction과 cone의 관계를 보고 meshlet의 모든 primitive가 back-facing임이 확실할 때 reject한다.

이 test는 매우 저렴할 수 있지만 조건이 있다.

- two-sided material에는 적용할 수 없음
- mirrored transform에서 winding/normal orientation을 주의
- non-uniform scale에서는 normal transform과 cone 보수성이 깨질 수 있음
- highly curved meshlet은 cone이 넓어져 rejection power가 약해짐

즉 meshlet generation 단계에서 **vertex reuse만 최대화하는 clustering**과 **culling-friendly normal coherence**가 충돌할 수 있다.

NVIDIA의 mesh shader 자료도 meshlet descriptor에 bounding shape와 cone을 넣어 task shader에서 cluster culling하는 구조를 설명한다.

### 4.7 Hi-Z는 “depth test를 싸게 근사하는 hierarchy”다

Hi-Z(Hierarchical Z) 또는 depth pyramid는 이전 또는 현재 depth buffer를 여러 mip level로 축소한 구조다.

Meshlet의 screen-space bounding rectangle이 크면 coarse mip에서 몇 번의 sample만으로 occlusion 가능성을 검사할 수 있다.

개념은:

```text
World Bounds
   ↓ project
Screen Rectangle + Conservative Depth Range
   ↓ choose mip
Hi-Z Samples
   ↓
Visible / Occluded
```

중요한 것은 reduction convention이다.

일반적인 depth에서 **near=0, far=1, LESS test**를 사용한다면 완전한 occlusion을 보수적으로 확인하기 위해 tile의 farthest occluder 정보를 유지하는 방식이 필요하다.

Reversed-Z처럼 **near=1, far=0, GREATER test**를 사용하면 reduction과 비교 방향도 반대로 설계된다.

즉 단순히 “Hi-Z는 max pyramid”라고 외우면 안 된다.

> **depth convention + reduction operator + object conservative depth가 하나의 contract다.**

### 4.8 Hi-Z에서 가장 위험한 것은 false occlusion이다

Occlusion culling은 frustum test보다 correctness 위험이 크다.

object의 screen rect를 너무 작게 잡거나 nearest depth를 잘못 계산하면 실제 visible object를 제거할 수 있다.

따라서 보통 다음과 같은 conservative bias가 필요하다.

- screen-space rect를 약간 확장
- depth bound를 conservative하게 이동
- near-plane crossing object는 특별 처리
- camera motion이 큰 frame에서는 이전-frame Hi-Z rejection을 완화
- dynamic object/geometry에는 visibility hysteresis 적용

특히 previous-frame depth pyramid를 사용할 때는 **temporal lag**가 있다.

이 문제는 내일의 주제로 이어진다.

### 4.9 Previous-frame Hi-Z는 free lunch가 아니다

이전 frame depth는 current frame culling 전에 이미 존재하므로 scheduling이 편하다.

하지만 다음 상황에서 오래된 occluder 정보가 문제가 된다.

- camera translation/rotation
- occluder가 이동
- SDF surface가 etch/deposition으로 변화
- LOD/refinement로 silhouette가 변함
- formerly occluded object가 disocclude됨

그래서 previous-frame visibility를 hard truth로 사용하기보다:

- conservative reprojected region
- safety margin
- previously visible fast path
- uncertain candidate fallback

같은 방식으로 다루는 것이 안정적이다.

Visibility history는 correctness source라기보다 **work prioritization / confidence signal**에 가깝게 보는 편이 좋다.

### 4.10 Compute culling과 Task Shader culling은 같은 문제가 아니다

둘 다 meshlet rejection을 할 수 있지만 pipeline placement가 다르다.

#### Compute Culling

```text
Compute
  ↓
Visible Meshlet Buffer
  ↓
Indirect Commands
  ↓
Graphics
```

장점:
- async compute 가능성
- 여러 consumer가 visibility 결과 공유 가능
- Hi-Z / scene hierarchy와 복잡한 pass 구성에 유리
- scan/compaction을 자유롭게 구성

단점:
- worklist를 global memory에 써야 함
- compute → indirect/task/mesh synchronization 필요

#### Task Shader Culling

```text
Task Shader
  ↓ compact payload
Mesh Shader
```

장점:
- mesh pipeline 내부에서 local work reduction
- subgroup/shared payload로 global intermediate buffer를 줄일 수 있음
- producer-consumer 거리가 짧음

단점:
- cross-object/global hierarchy traversal에 덜 유연
- task payload 크기와 workgroup organization 제약
- visibility 결과를 다른 pass와 공유하기 어려움

Khronos sample은 task shader가 mesh shader launch 수를 결정하고 payload를 전달하는 구조를 보여주며, payload를 작게 유지하는 것이 권장된다.

실무적으로는 둘을 혼합할 수 있다.

```text
Compute: coarse instance/chunk culling
Task Shader: fine meshlet culling
Mesh Shader: optional primitive culling
```

### 4.11 Compaction이 visibility boolean을 실행 가능한 work로 바꾼다

culling 결과가 단순한 `visible[i] = true/false` 배열이라면 renderer는 아직 sparse candidate array를 다뤄야 한다.

실제 실행에는 compact list가 유리하다.

```text
input:
[0,1,0,1,1,0]

output:
[id1,id3,id4]
count = 3
```

대표적인 GPU 방법은 다음 두 가지다.

#### Atomic Append

visible thread가 global counter를 증가시키고 slot을 얻는다.

장점:
- 단순
- sparse output에 편함
- 작은 dirty region/update에 유리

단점:
- visibility가 높으면 atomic contention
- output order가 nondeterministic할 수 있음

#### Prefix Sum / Scan

predicate를 0/1로 만들고 exclusive scan으로 output index를 계산한다.

장점:
- deterministic compact order
- dense large workload에서 효율적
- exact output size와 offset을 얻기 쉬움

단점:
- 여러 단계와 scratch buffer 필요

GPU-driven renderer에서는 한 가지 방식만 고집하기보다 workload 규모와 density에 따라 선택하는 편이 좋다.

### 4.12 Subgroup compaction은 global atomic pressure를 줄인다

한 단계 더 나아가면 각 thread가 바로 global atomic을 호출하지 않고 subgroup 또는 workgroup 내부에서 먼저 compact할 수 있다.

개념적으로:

```text
predicate
   ↓
subgroup ballot
   ↓
local rank
   ↓
one reservation per subgroup/workgroup
   ↓
packed writes
```

장점:
- global atomic 수 감소
- contiguous writes
- cache behavior 개선

Task shader payload도 같은 원리를 사용한다.

visible meshlet lane만 local compact index를 얻어 payload에 넣고, mesh shader workgroup 수를 visible count로 설정할 수 있다.

### 4.13 Worklist는 logical ID를 저장하는 편이 relocation에 강하다

어제의 핵심과 직접 연결된다.

다음은 빠르지만 relocation-sensitive하다.

```text
VisibleMeshlet {
    uint64 vertexAddress;
    uint64 primitiveAddress;
}
```

반면 다음은 한 단계의 indirection이 있지만 relocation-safe하다.

```text
VisibleMeshlet {
    MeshHandle mesh;
    uint meshletLocalIndex;
    uint expectedGeneration;
}
```

그 뒤 render snapshot의 meshlet/chunk table에서 physical address를 resolve한다.

따라서 visibility pipeline과 allocator를 decouple하려면

> **worklist는 “무엇을 그릴 것인가”를 저장하고, “현재 어디에 있는가”는 render table에서 resolve한다.**

라는 역할 분리가 좋다.

### 4.14 Indirect count는 CPU readback을 없앤다

Compute pass가 최종 visible command count를 만들었을 때 CPU가 이를 readback하면 GPU-driven pipeline의 장점이 크게 줄어든다.

Vulkan의 indirect-count draw는 count buffer에서 실제 draw 수를 GPU가 직접 읽는다.

Mesh shader에서도 `vkCmdDrawMeshTasksIndirectCountEXT`는 device-side count를 사용하며, 최신 Vulkan refpage에서는 이 기능이 address-range 기반의 `vkCmdDrawMeshTasksIndirectCount2EXT`에 의해 superseded되었다고 명시한다.

Pipeline의 논리는 다음과 같다.

```text
Cull / Compact
    ↓
IndirectCommandBuffer
DrawCountBuffer
    ↓ barrier
Indirect Count Draw
```

여기서 `maxDrawCount`는 allocation capacity와 safety contract를 표현한다.

count buffer의 값만 맞고 command array가 다른 epoch이면 여전히 잘못된 snapshot이다.

### 4.15 Compute → Draw Indirect synchronization은 명시적이다

Compute shader가 indirect command/count를 쓰고 graphics command가 읽는다면 “같은 command buffer에 있으니 알아서 보인다”라고 생각하면 안 된다.

필요한 dependency의 의미는 개념적으로 다음과 같다.

```text
Producer:
COMPUTE_SHADER
SHADER_STORAGE_WRITE

        ↓ memory dependency

Consumer:
DRAW_INDIRECT
INDIRECT_COMMAND_READ
```

Khronos Vulkan synchronization example도 compute dispatch가 indirect buffer를 쓴 뒤 `DRAW_INDIRECT` stage가 읽도록 barrier를 두는 예시를 제공한다.

만약 task/mesh shader가 별도의 compacted worklist/meshlet table도 읽는다면 그 resource에는 추가로:

```text
TASK_SHADER / MESH_SHADER
SHADER_STORAGE_READ
```

쪽 visibility가 필요하다.

즉 **indirect command read와 shader storage read는 서로 다른 consumer access**다.

### 4.16 Barrier는 “모든 graphics”보다 정확한 consumer를 쓰는 편이 낫다

가장 넓은 barrier는 correctness를 얻기 쉽지만 overlap을 줄일 수 있다.

예를 들어:

```text
src = ALL_COMMANDS
dst = ALL_COMMANDS
```

는 reasoning은 쉽지만 scheduler 자유도를 줄일 수 있다.

GPU-driven pipeline에서는 resource별 소비 stage를 분리하면 더 명확하다.

- indirect command/count → `DRAW_INDIRECT`
- meshlet worklist → `TASK_SHADER` 또는 `MESH_SHADER`
- vertex/index payload → `MESH_SHADER` 또는 vertex/index input
- Hi-Z image → culling `COMPUTE_SHADER`

이렇게 해야 async compute와 graphics overlap을 설계하기 쉽다.

### 4.17 Async compute는 “queue를 나눴다”보다 dependency chain이 중요하다

Culling을 async compute queue에 둔다고 자동으로 빨라지지 않는다.

다음 조건을 같이 봐야 한다.

- Hi-Z가 언제 준비되는가
- graphics와 compute가 같은 memory bandwidth를 경쟁하는가
- indirect buffer가 어느 queue family에서 사용되는가
- queue ownership transfer가 필요한가
- semaphore wait 때문에 graphics가 결국 culling을 기다리는가
- culling이 critical path를 단축하는가

특히 현재-frame Hi-Z를 사용하면 depth prepass가 끝난 뒤 culling이 가능하므로 overlap window가 제한될 수 있다.

Previous-frame Hi-Z는 scheduling freedom이 크지만 temporal conservativeness 문제가 있다.

즉 async compute 여부는 visibility algorithm과 독립적인 선택이 아니다.

### 4.18 Fine primitive culling은 항상 이득이 아니다

Mesh shader는 workgroup 내부에서 triangle별 culling도 가능하다.

예:

- backface
- tiny primitive
- frustum
- conservative occlusion approximation

하지만 per-primitive test가 늘면:

- ALU 증가
- output compaction 증가
- shared/register pressure 증가
- mesh shader occupancy 감소

가 생긴다.

NVIDIA 자료도 primitive culling이 작은 triangle이 많거나 expensive output을 피할 수 있을 때 유리할 수 있지만 항상 이득인 것은 아니라고 설명한다.

따라서 granularity를 다음처럼 봐야 한다.

```text
Cull cost < Saved downstream work
```

Meshlet-level rejection이 이미 매우 강하다면 primitive-level 추가 test는 손해일 수 있다.

### 4.19 Screen-space size는 culling과 LOD를 함께 연결한다

Meshlet의 projected size가 매우 작으면 다음 선택지가 있다.

- 그대로 render
- coarser LOD 선택
- subpixel contribution threshold 아래에서 cull
- representative proxy로 merge

하지만 “작다”를 바로 cull로 연결하면 temporal popping이 생길 수 있다.

그래서 projected error와 hysteresis를 함께 두는 편이 좋다.

어제의 relocation-safe render table에 LOD metadata가 있고, 이전의 curvature-aware meshing에 world-space feature error가 있다면 다음 chain을 만들 수 있다.

```text
Curvature / Geometry Error
        ↓
World-space LOD Error
        ↓ projection
Screen-space Error
        ↓
LOD Selection / Culling
```

이렇게 geometry generation과 rendering LOD가 같은 error model로 연결된다.

### 4.20 Dynamic SDF geometry에서는 visibility bounds도 versioned resource다

Static asset에서는 meshlet bound가 asset build 결과로 고정된다.

Dynamic SDF mesh에서는:

- brick remesh
- meshlet rebuild
- relocation
- bound recompute

가 모두 일어날 수 있다.

따라서 meshlet visibility record도 다음 version과 맞아야 한다.

```text
geometry generation
meshlet generation
bound generation
relocation epoch
render snapshot
```

old bound + new geometry 조합은 위험하다.

bound가 작으면 false negative,
bound가 크면 성능 저하가 생긴다.

따라서 visibility metadata는 단순 optional optimization data가 아니라 **geometry correctness에 영향을 줄 수 있는 versioned resource**다.

### 4.21 Hot culling metadata와 render payload를 분리한다

Culling pass가 자주 읽는 필드는 제한적이다.

예:

```text
CullMeshlet
- sphere center/radius
- cone axis/cutoff
- mesh handle
- local meshlet ID
- LOD error
- flags
```

반면 mesh shader가 필요한 값:

```text
RenderMeshlet
- vertex offset
- primitive offset
- vertex count
- primitive count
- material/attribute metadata
```

이를 큰 AoS 하나로 묶으면 culling에서 사용하지 않는 render payload까지 cache line에 들어온다.

따라서 **Cull SoA / Render SoA / Cold Debug Metadata**로 나누는 구조가 bandwidth 측면에서 유리할 수 있다.

이것은 어제의 relocation table hot/cold split과 같은 설계 원리다.

### 4.22 Meshlet ordering도 culling efficiency에 영향을 준다

Visible meshlet을 compact하게 저장해도 source ordering이 random하면 bounds와 payload fetch locality가 나빠질 수 있다.

Ordering 후보:

- Morton / spatial order
- parent chunk contiguous range
- material grouping
- LOD grouping
- recently visible order

하지만 목표가 충돌할 수 있다.

예:

- spatial order → culling cache locality
- material order → shading/state coherence
- stable logical order → incremental update와 debug 용이

GPU-driven renderer에서는 하나의 perfect order보다

> **culling order와 render order를 동일하게 유지할 필요가 있는가**

를 먼저 질문하는 편이 좋다.

Visibility output을 후속 radix/binning pass로 재정렬하는 architecture도 가능하다.

### 4.23 Temporal visibility는 bandwidth optimization으로 볼 수 있다

지난 frame에 visible했던 meshlet은 이번 frame에도 visible할 확률이 높을 수 있다.

이를 활용하면:

- previously visible set 우선 검사
- invisible set은 coarse test 후 늦게 검사
- occlusion confidence/history 유지
- camera cut 시 history reset

같은 정책을 둘 수 있다.

하지만 history는 correctness authority가 아니다.

visibility history의 가장 안전한 역할은:

> **“무엇을 먼저 검사할 것인가”를 정하는 hint**

이다.

오늘의 핵심은 current-frame conservative visible set을 유지하는 것이고, 내일은 previous-frame Hi-Z와 temporal coherence를 더 깊게 본다.

### 4.24 CPU visibility와 GPU visibility의 역할을 완전히 동일하게 만들 필요는 없다

GPU-driven architecture라도 CPU가 아무것도 하지 않는 것은 아니다.

CPU는 다음 coarse state를 관리할 수 있다.

- scene streaming
- top-level object existence
- large world partition
- resource residency
- camera region
- high-level portal/zone state

GPU는:

- instance
- chunk
- meshlet
- primitive

같은 fine-grained visibility를 담당할 수 있다.

즉 좋은 architecture는 “전부 GPU”가 아니라

> **decision frequency와 data locality에 맞는 processor를 선택하는 것**

이다.

### 4.25 프로파일링에서 봐야 할 지표

Visibility system은 final FPS만 보면 병목 원인을 찾기 어렵다.

유용한 counters:

- candidate chunk count
- frustum-rejected chunk ratio
- candidate meshlet count
- frustum-rejected meshlet ratio
- cone-rejected meshlet ratio
- Hi-Z rejected ratio
- final visible meshlet count
- primitive input/output count
- task shader invocations
- mesh shader invocations
- mesh primitives generated
- indirect draw count
- worklist bytes written
- cull metadata bytes read
- atomic operations / candidate
- scan/compaction time
- Hi-Z sampling bandwidth
- L1/L2 hit rate
- task/mesh shader occupancy
- average meshlets per visible chunk
- false-occlusion debug count
- generation/epoch rejection count

특히 중요한 derived metric은:

```text
Saved downstream work / Culling cost
```

이다.

Culling pass가 1 ms를 쓰고 geometry pipeline에서 0.4 ms만 줄였다면 rejection ratio가 높아 보여도 실패한 optimization일 수 있다.

---

## 5. 내 관심 분야와 연결

### 5.1 Semiconductor SDF visualization

Process emulation에서는 넓고 평평한 영역과 작은 high-curvature feature가 동시에 존재한다.

예:

- substrate
- thin oxide
- trench
- gate stack
- spacer
- deposition overhang
- etched cavity

Curvature-aware adaptive meshing으로 geometry 수를 줄여도 camera에는 일부 영역만 보인다.

따라서 다음 hierarchy가 자연스럽다.

```text
Sparse SDF Brick
      ↓
Generated Mesh Chunk
      ↓
Meshlet Range
      ↓
Cull Metadata
      ↓
Visible Meshlet Worklist
      ↓
Vulkan Mesh Shader / Draw
```

brick AABB를 coarse visibility bound로 재사용하면 SDF hierarchy와 renderer hierarchy가 연결된다.

### 5.2 CUDA → Vulkan GPU-stay-GPU

사용자의 관심 pipeline에서는 CUDA가 simulation/mesh generation을 수행하고 Vulkan이 rendering을 소비할 수 있다.

이때 ideal flow는:

```text
CUDA:
dirty bricks
mesh update
meshlet rebuild
bounds update
        ↓ external sync
Vulkan Compute:
visibility + compaction
        ↓
Indirect Count
        ↓
Vulkan Graphics:
task/mesh shader
```

중간 CPU readback 없이 surface update부터 visibility까지 GPU에 남길 수 있다.

다만 핵심은 단순한 zero-copy가 아니다.

**geometry epoch, bound epoch, relocation epoch, visibility epoch**를 같은 published snapshot으로 묶는 것이 중요하다.

### 5.3 CFD / scientific visualization

CFD surface, iso-surface, streamline geometry는 camera와 timestep에 따라 visible set이 크게 달라질 수 있다.

특히 simulation geometry는 매 timestep 바뀌므로 static game asset보다 temporal occlusion history의 신뢰도가 낮을 수 있다.

따라서 visibility history를 강한 rejection rule보다 scheduling hint로 사용하는 설계가 더 안전하다.

### 5.4 Game engine / real-time rendering

이 개념은 다음 실무와 직접 연결된다.

- GPU-driven scene rendering
- mesh shader pipeline
- virtualized geometry
- runtime destructible mesh
- voxel terrain
- procedural geometry
- Nanite-style cluster thinking
- indirect command generation
- visibility buffer
- occlusion culling
- LOD selection

Unity/Nintendo류 engine role에서 중요한 것은 “mesh shader를 써봤다”보다 다음 질문에 답할 수 있는 것이다.

> **어떤 granularity에서 cull하고, false negative를 어떻게 막으며, compact work를 어떤 synchronization으로 graphics stage에 넘기는가?**

---

## 6. 머릿속에 남길 질문 3개

1. **Meshlet culling에서 rejection ratio가 높아도 전체 frame time이 개선되지 않을 수 있는 이유를 memory traffic, compaction cost, downstream work 관점에서 어떻게 설명할 수 있는가?**
2. **Previous-frame Hi-Z를 current-frame hard rejection source로 사용할 때 camera motion과 dynamic geometry가 false negative를 만드는 경로는 무엇인가?**
3. **Compute culling과 task-shader culling을 조합한다면 어느 hierarchy level을 global compute에 두고 어느 level을 task payload에 두는 것이 좋은지 어떤 profiler 지표로 결정할 수 있는가?**

---

## 7. graphics engineer 면접 질문 1개와 답변

### 질문

**“GPU compute shader가 visible meshlet list와 indirect draw count를 생성했습니다. 그 다음 바로 `vkCmdDrawMeshTasksIndirectCount*`를 호출하면 되는 것 아닌가요?”**

### 답변

같은 command buffer에 순서대로 기록되어 있다는 사실만으로 compute shader의 memory write가 indirect-command consumer와 mesh/task shader에 자동으로 올바르게 보인다고 가정하면 안 된다.

먼저 두 종류의 consumer를 구분해야 한다.

1. **Indirect command/count**
   - `DRAW_INDIRECT` stage에서 읽힌다.
   - compute의 shader write가 indirect command read에 visible해야 한다.

2. **Visible meshlet worklist / meshlet records**
   - task shader 또는 mesh shader가 storage buffer로 읽을 수 있다.
   - compute write가 `TASK_SHADER` / `MESH_SHADER` shader read에 visible해야 한다.

따라서 synchronization은 단순히 “compute 다음 graphics”가 아니라 **resource별 실제 consumer stage와 access type**을 표현해야 한다.

또 command buffer와 count buffer가 current epoch인데 meshlet bounds/table이 previous geometry epoch라면 memory barrier가 완벽해도 semantic correctness는 깨진다.

그래서 production GPU-driven renderer에서는:

- execution/memory synchronization
- geometry/bounds generation
- relocation snapshot
- visibility generation
- indirect command/count generation

을 같은 render snapshot contract로 묶어야 한다.

핵심은 **barrier는 memory ordering을 보장하지만, resource version의 의미까지 자동으로 맞춰주지는 않는다는 것**이다.

---

## 8. 포트폴리오 / 커리어 연결

이 주제는 graphics engineer 포트폴리오에서 “GPU 최적화”를 구체적인 시스템으로 설명하기 좋다.

### Geometry / Rendering 관점

- meshlet clustering과 bound quality
- frustum / cone / Hi-Z의 역할 분리
- false-positive vs false-negative
- projected error와 LOD
- mesh shader primitive emission

### GPU Compute 관점

- predicate generation
- subgroup ballot
- atomic append
- prefix sum / stream compaction
- work queue
- indirect count generation

### Vulkan 관점

- `VK_EXT_mesh_shader`
- task/mesh shader
- `vkCmdDrawMeshTasksIndirectCountEXT`
- `vkCmdDrawMeshTasksIndirectCount2EXT`
- `DRAW_INDIRECT` stage
- synchronization2
- compute-write → indirect-read dependency

### Memory Layout 관점

- culling hot metadata와 render payload 분리
- quantized conservative bounds
- mesh/chunk-level indirection
- logical meshlet ID
- spatial ordering
- cache locality

### Engine Architecture 관점

- persistent geometry snapshot
- visibility epoch
- dynamic SDF update
- previous-frame visibility history
- CPU/GPU responsibility split
- async compute scheduling

면접에서 강한 설명은

> “Hi-Z로 occlusion culling을 했다.”

에서 끝나지 않는다.

더 강한 설명은 다음 연결을 보여준다.

> **“coarse hierarchy로 descriptor traffic을 줄이고, meshlet-level conservative tests를 통과한 work만 subgroup/scan으로 compact하며, compute-generated command/count와 task/mesh-readable worklist에 서로 다른 consumer synchronization을 설정하고, culling 자체의 비용 대비 절감된 downstream work를 profiler로 검증한다.”**

이렇게 말할 수 있으면 algorithm, GPU architecture, Vulkan synchronization, memory layout을 함께 이해한다는 신호가 된다.

---

## 9. 내일 이어서 볼 개념

**Temporal Hi-Z Occlusion Culling for Dynamic Geometry: Reprojection Error, Conservative Depth Bounds, and Visibility Hysteresis**

오늘은 Hi-Z를 visibility ladder의 한 단계로 봤다.

내일은 그중 가장 까다로운 부분인 **previous-frame depth를 current-frame visibility에 사용할 때 생기는 시간적 불일치**를 깊게 본다.

학습 흐름은 다음과 같다.

```text
relocation-safe render tables
        ↓
hierarchical meshlet visibility
        ↓
temporal Hi-Z occlusion
        ↓
stable GPU-driven visibility across frames
```

다음 개념에서는 다음을 중심으로 연결한다.

- previous-frame depth pyramid
- reversed-Z reduction
- camera reprojection
- occluder motion
- disocclusion
- conservative screen/depth expansion
- visibility hysteresis
- camera-cut/history invalidation
- current-frame depth prepass와 previous-frame Hi-Z의 trade-off
- temporal false-negative 방지

---

## 10. 참고 키워드

- GPU-Driven Rendering
- Meshlet Visibility
- Hierarchical Culling
- Frustum Culling
- Cluster / Normal-Cone Culling
- Hi-Z / Hierarchical Z
- Depth Pyramid
- Reversed-Z
- Conservative Occlusion Culling
- False Positive / False Negative
- Task Shader
- Mesh Shader
- `VK_EXT_mesh_shader`
- `VkDrawMeshTasksIndirectCommandEXT`
- `vkCmdDrawMeshTasksIndirectCountEXT`
- `vkCmdDrawMeshTasksIndirectCount2EXT`
- Draw Indirect Count
- Multi-Draw Indirect (MDI)
- `VK_PIPELINE_STAGE_2_DRAW_INDIRECT_BIT`
- `VK_ACCESS_2_INDIRECT_COMMAND_READ_BIT`
- Synchronization2
- Stream Compaction
- Prefix Sum / Scan
- Atomic Append
- Subgroup Ballot
- Work Queue
- Work Amplification / Work Reduction
- Projected Screen-Space Error
- Conservative Quantization
- Visibility History
- Temporal Coherence
- Meshlet Bounds
- Cull Metadata SoA
- Indirect Command Buffer
- Count Buffer
- CUDA-Vulkan Interop
- Async Compute
- Khronos Vulkan Samples — **Mesh Shader Culling**
- Vulkan Documentation Project — **VK_EXT_mesh_shader**
- Vulkan Documentation Project — **Synchronization Examples**
- Vulkan Documentation Project — **GPU-Side Command Generation**
- Vulkan Documentation Project — **Multi-Draw Indirect (MDI)**
- NVIDIA — **Introduction to Turing Mesh Shaders**
- NVIDIA — **Using Mesh Shaders for Professional Graphics**
