---
title: "Occluder Selection and Two-Phase GPU Visibility: Depth-Only Prepasses, Coarse Occluder Rasterization, and Late Hi-Z Rejection"
date: "2026-09-10"
category: Graphics
tags: [GPU, Rendering, GPU-Driven Rendering, Occlusion Culling, Occluder Selection, Depth Prepass, Hi-Z, HZB, Meshlet, Mesh Shader, Indirect Draw, Vulkan, CUDA, Dynamic Geometry, Memory Layout, C++]
level: intermediate
---

# [Daily Graphics Growth] 2026-09-10 - Occluder Selection and Two-Phase GPU Visibility: Depth-Only Prepasses, Coarse Occluder Rasterization, and Late Hi-Z Rejection

## 1. 오늘의 개념

어제는 **Temporal Hi-Z Occlusion Culling**에서 previous-frame depth를 current-frame truth가 아니라 temporal evidence로 취급해야 한다는 점을 봤다. Camera motion, dynamic SDF update, disocclusion 때문에 history가 불확실하면 conservative하게 visible/unknown으로 남기는 것이 correctness에 중요했다.

오늘은 그 다음 단계다.

> **Previous-frame history에 덜 의존하면서도 full depth prepass의 중복 비용을 피하려면, current frame에서 어떤 geometry를 먼저 occluder로 렌더하고 어떤 geometry를 뒤에서 Hi-Z로 제거해야 하는가?**

핵심은 **Two-Phase GPU Visibility**다.

```text
Phase A
Stable / High-Value Occluder Selection
        ↓
Depth-Only or Reduced-Geometry Raster
        ↓
Current-Frame Partial HZB

Phase B
Remaining Candidate Culling
        ↓
Compact Visible Worklist
        ↓
Indirect / Mesh-Task Rendering
```

이 구조의 목적은 모든 geometry를 depth-only로 한 번 더 그리는 것이 아니다. **현재 frame에서 큰 occlusion value를 가진 일부 geometry만 먼저 depth에 넣고**, 그 depth로 훨씬 많은 후속 geometry work를 제거하는 것이다.

오늘의 가장 중요한 geometry rule은 다음이다.

> **Occludee를 검사할 때는 실제 geometry를 모두 포함하는 outer-conservative bound가 필요하지만, occluder proxy는 실제 opaque geometry 바깥으로 나가면 안 되는 inner-conservative representation이어야 한다.**

Bounding box는 좋은 occludee bound가 될 수 있지만, 그대로 occluder로 rasterize하면 실제 geometry가 없는 픽셀까지 depth를 써서 뒤의 visible geometry를 잘못 제거할 수 있다.

---

## 2. 한 줄 핵심

> Two-phase visibility의 성패는 **“가장 큰 object”가 아니라 `saved downstream work / occluder raster cost`가 높은 안정적인 opaque geometry를 먼저 depth에 넣고, occluder는 inner-conservative, occludee bound는 outer-conservative라는 서로 반대의 보수성(conservativeness)을 지키는 것**에 달려 있다.

---

## 3. 왜 중요한가

Full depth prepass는 매우 단순한 current-frame occlusion source다.

```text
All Opaque Geometry
    ↓ depth-only
Full Depth
    ↓ HZB
Main Pass
```

하지만 geometry가 많거나 mesh shader/vertex shader cost가 큰 장면에서는 같은 geometry를 depth와 main pass에서 두 번 처리할 수 있다. Dynamic mesh/SDF에서는 매 frame meshlet이 재생성되거나 relocation되므로 depth-prepass용 geometry fetch 자체도 싸지 않을 수 있다.

반대로 previous-frame HZB만 사용하면 depth prepass는 피할 수 있지만 어제 본 것처럼 temporal mismatch가 생긴다.

Two-phase visibility는 그 사이를 노린다.

- **Phase A:** 적은 비용으로 current-frame occlusion을 만든다.
- **Phase B:** current HZB를 사용해 나머지 geometry를 GPU에서 제거한다.

좋은 occluder 몇 개가 화면의 큰 영역을 덮으면 수천 개 meshlet의 task/mesh shader 실행, vertex/index fetch, primitive setup, rasterization, fragment work를 줄일 수 있다.

반대로 occluder selection이 나쁘면:

- depth prepass 비용 증가
- HZB build 비용 증가
- culling compute 증가
- main pass에서도 geometry를 다시 처리
- 결국 full rendering보다 느려질 수 있다

따라서 중요한 것은 단순 occluded object 수가 아니다.

개념적으로 occluder의 가치는 다음과 같이 볼 수 있다.

```text
OccluderValue
≈
Expected Covered Screen Area
× Expected Hidden Work Behind It
× Temporal / Geometric Stability
---------------------------------
Depth Raster Cost + Selection Cost
```

정확한 공식이라기보다 **profiler가 무엇을 측정해야 하는지 알려주는 mental model**이다.

2021년 *Hierarchical Raster Occlusion Culling*도 temporal coherence로 찾은 occluder를 이용하고, occludee group을 coarse하게 rasterize하며, occluder filtering을 추가해 scalable한 online occlusion culling을 구성한다. 핵심 메시지는 동일하다. Occlusion culling에서는 hierarchy 자체뿐 아니라 **무엇을 rasterize하고 무엇을 test할지 선택하는 비용 모델**이 중요하다.

---

## 4. 구현 관점

### 4.1 Occludee bound와 occluder proxy는 보수성 방향이 반대다

이 구분이 오늘의 핵심이다.

#### Occludee test bound — Outer-Conservative

실제 geometry를 반드시 포함해야 한다.

```text
Actual Geometry ⊂ Test Bound
```

예:

- bounding sphere
- AABB
- conservative meshlet bound

Bound가 조금 크면 false positive, 즉 실제로 가려졌지만 visible로 남는 성능 손해만 생긴다.

#### Occluder proxy — Inner-Conservative

Proxy가 쓰는 depth/coverage는 실제 opaque geometry 안쪽에 있어야 한다.

```text
Occluder Proxy ⊂ Actual Opaque Coverage
```

Proxy가 실제 silhouette 밖으로 나가면 geometry가 존재하지 않는 픽셀에 depth를 쓰고 뒤 geometry를 false-negative cull할 수 있다.

그래서 **AABB는 occludee test에는 좋지만 occluder raster proxy로는 일반적으로 안전하지 않다.**

### 4.2 Coarse LOD가 자동으로 safe occluder는 아니다

“Triangle 수가 적은 LOD를 depth proxy로 쓰면 되지 않는가?”라는 생각이 자연스럽지만, 일반 mesh simplification은 silhouette를 실제 geometry 바깥으로 이동시킬 수 있다.

Occlusion proxy가 안전하려면 단순 polygon count가 아니라 **coverage relation**이 중요하다.

Safe한 방향:

- 원본 opaque surface의 subset
- inward-offset proxy
- 보수적으로 축소된 hull
- 실제 surface보다 camera에서 멀거나 내부에 있는 proxy
- offline/compute 단계에서 occlusion-safe로 분류된 reduced mesh

주의할 방향:

- geometry 바깥으로 팽창한 convex hull
- arbitrary decimated LOD
- bounding box
- thin hole이나 trench를 막아버린 proxy

Visual LOD의 error metric과 occluder proxy의 error metric은 다르다.

Visual LOD는 screen-space image error를 줄이면 되지만,
occluder proxy는 **잘못된 extra coverage가 0에 가까워야 한다.**

### 4.3 좋은 occluder의 특징

대표적으로 다음 조건이 유리하다.

- **Opaque**
- **Large projected screen area**
- camera에 비교적 가까움
- motion이 작음
- geometry/topology가 안정적
- low-cost depth raster
- 뒤에 많은 geometry가 존재할 가능성이 큼
- thin/porous하지 않음
- alpha-tested coverage가 복잡하지 않음

즉 world-space 크기보다 **screen-space coverage**가 중요하다.

매우 큰 object가 멀리 있어 20 pixel밖에 차지하지 않으면 가치가 낮다.

반대로 중간 크기 wall이 화면의 절반을 덮으면 매우 좋은 occluder다.

### 4.4 `Projected Area / Raster Cost`는 좋은 1차 heuristic이다

완벽한 “뒤에 얼마나 많은 geometry가 있는가”를 미리 계산하면 occlusion culling보다 더 비쌀 수 있다.

그래서 저렴한 score를 사용한다.

개념적으로:

```text
score =
projectedArea
× opacityConfidence
× stability
× depthPriority
----------------
estimatedRasterCost
```

`estimatedRasterCost`는 다음 metadata에서 근사할 수 있다.

- triangle/meshlet count
- vertex count
- alpha test 여부
- deformation cost
- material depth-shader cost

중요한 점은 exact physical model이 아니라 **selection overhead가 saved work보다 작아야 한다는 것**이다.

### 4.5 Front-to-back priority는 occluder value를 높인다

Depth occlusion은 camera에 가까운 geometry가 먼저 기록될수록 유리하다.

Phase A occluder candidate를 대략적인 depth key로 front-to-back ordering하면:

- nearer depth가 먼저 만들어짐
- 뒤쪽 Phase A geometry도 early-Z로 빨리 제거 가능
- partial depth가 더 강한 occlusion source가 됨

다만 full sort가 비쌀 수 있다.

GPU에서는 exact sort보다:

- coarse depth bins
- tile/depth bucket
- approximate front-to-back
- previous-frame ordering reuse

가 더 실용적일 수 있다.

### 4.6 Full sort보다 bucket/binning이 나을 수 있다

수십만 occluder candidate를 매 frame radix sort하면 selection 자체가 병목이 될 수 있다.

대안:

```text
Depth Bins:
near
mid-near
mid
far
```

또는 screen tile coverage 기준 binning을 사용할 수 있다.

Occlusion은 정확한 painter ordering이 아니라 **충분히 좋은 near coverage를 빨리 만드는 것**이 목적이므로 approximate ordering이 허용되는 경우가 많다.

### 4.7 Occluder budget은 고정 개수보다 GPU work budget으로 본다

`top 100 occluders` 같은 fixed count는 object complexity가 다양할 때 불안정하다.

더 의미 있는 budget은:

```text
max depth meshlets
max depth triangles
max projected raster pixels
max Phase-A GPU time target
```

같은 work-oriented budget이다.

대형 wall 3개와 작은 meshlet 100개는 같은 “3개 vs 100개 object”로 비교할 수 없다.

### 4.8 Phase A depth pipeline은 가능한 단순해야 한다

Phase A의 목적은 shading이 아니라 occlusion source 생성이다.

따라서 logical pipeline은 다음처럼 작다.

```text
Position / Minimal Geometry Fetch
        ↓
Depth Transform
        ↓
Depth Test / Write
```

필요하지 않은 것:

- lighting
- PBR material sampling
- normal/tangent fetch
- expensive texture sampling
- unnecessary interpolants

다만 alpha-tested material은 실제 opaque coverage를 알기 위해 texture fetch가 필요할 수 있다. 이 경우 Phase A cost가 급격히 올라갈 수 있다.

그래서 foliage/fence처럼 alpha-test가 강한 geometry는 좋은 occluder가 아닐 수 있다.

### 4.9 Mesh shader에서는 occluder-only meshlet path를 따로 생각할 수 있다

Meshlet renderer라면 Phase A가 모든 render payload를 읽을 필요가 없다.

Occluder metadata:

```text
OccluderMeshlet
- logical meshlet ID
- position stream range
- primitive range
- conservative proxy flag
- projected-area estimate
- stability bits
```

Main rendering metadata:

```text
RenderMeshlet
- full attribute range
- material
- normals/tangents
- UV
- shading flags
```

Depth path와 shading path의 hot data를 분리하면 bandwidth를 줄일 수 있다.

NVIDIA의 mesh shader 자료도 task shader에서 cluster culling을 수행하고, surviving cluster만 mesh shader로 전달하는 구조를 보여준다. Two-phase visibility에서도 같은 **work reduction before expensive payload fetch** 원칙이 반복된다.

### 4.10 Phase A occluder가 실제로 hidden이어도 correctness 문제는 아니다

선택한 occluder가 다른 geometry 뒤에 완전히 가려져 있을 수 있다.

그 occluder를 depth pass에 그려도 앞의 depth test에 실패하므로 보통 추가 depth를 만들지 않는다.

즉 잘못된 occluder selection은 주로 **성능 손해**다.

반면 occluder proxy가 실제 geometry보다 크게 rasterize되는 것은 **correctness 손해**다.

이 둘을 구분해야 한다.

### 4.11 Partial Depth에서 HZB를 만들 수 있다

HZB가 scene 전체 depth를 포함할 필요는 없다.

Phase A에서 selected occluder만 depth에 썼다면 그 depth로 만든 pyramid는 **부분적인 occlusion source**다.

Coverage가 없는 영역에서는 background/far depth가 유지되어 candidate를 cull하지 못할 뿐이다.

이것은 안전한 false positive다.

즉 current-frame partial HZB의 의미는:

> **확실히 알고 있는 occluder만 포함한 conservative depth evidence**

다.

### 4.12 Standard-Z / Reversed-Z semantics는 그대로 유지된다

어제와 연결해서:

#### Standard-Z

```text
near = 0
far = 1
LESS
HZB = max reduction
```

#### Reversed-Z

```text
near = 1
far = 0
GREATER
HZB = min reduction
```

Partial depth에서도 같은 원칙이 유지된다.

Clear depth는 “occluder가 없음”을 표현하므로 candidate를 잘못 cull하지 않는 방향이어야 한다.

### 4.13 MSAA depth를 HZB로 만들 때도 conservative resolve가 필요하다

MSAA depth에는 pixel마다 여러 sample depth가 있다.

Occlusion culling용 single depth로 축약할 때는 모든 sample을 실제로 occlude할 수 있는 방향을 골라야 한다.

즉 object가 sample 사이 gap에서 보일 가능성이 있는데 aggressive average depth를 사용하면 위험하다.

Occlusion hierarchy에서는 visual antialiasing보다 **sample coverage conservativeness**가 우선이다.

### 4.14 Phase B는 “나머지 candidate”를 current HZB로 test한다

Phase B의 input은 대략 다음과 같다.

```text
Candidate Meshlet / Chunk Bounds
Current Partial HZB
Generation / Epoch
```

Output:

```text
Visible Worklist
Indirect Command / Mesh Task Count
```

이때 Phase A에서 이미 rasterized한 geometry를 main shading에서 다시 그릴지, visibility/material pass와 결합할지는 renderer architecture에 따라 달라진다.

Depth-only prepass라면 main shading에서 depth-equal/less-equal을 사용할 수 있고,
visibility-buffer architecture에서는 Phase A와 main visibility pass를 더 강하게 통합할 수도 있다.

### 4.15 Phase A와 Phase B 사이 synchronization은 명확하다

Logical dependency:

```text
Graphics Phase A
DEPTH_STENCIL_ATTACHMENT_WRITE
        ↓
HZB Compute
SHADER_SAMPLED_READ
        ↓
HZB Compute Write
SHADER_STORAGE_WRITE
        ↓
Phase B Cull Compute
SHADER_SAMPLED_READ
        ↓
Visible Work / Indirect Write
        ↓
DRAW_INDIRECT + TASK/MESH/GRAPHICS READ
```

즉:

- depth write → HZB read
- HZB write → cull read
- cull write → indirect read
- cull write → task/mesh shader storage read

를 구분해야 한다.

하나의 “graphics barrier”로 뭉개면 correctness는 얻을 수 있어도 overlap과 scheduling freedom을 잃을 수 있다.

### 4.16 Async compute가 항상 이득은 아니다

HZB build와 Phase B culling을 compute queue로 옮길 수 있다.

하지만 Phase A depth가 끝나야 current HZB를 만들 수 있으므로 dependency chain은 존재한다.

```text
Phase A Graphics
    ↓ semaphore
HZB + Cull Compute
    ↓ semaphore
Phase B Graphics
```

Queue를 나누면 queue synchronization과 memory contention이 생긴다.

따라서 async compute의 가치는:

- compute units 여유
- graphics raster bottleneck
- memory bandwidth
- HZB/cull latency
- overlap 가능한 다른 rendering pass

와 함께 봐야 한다.

### 4.17 Previously visible/stable occluder는 좋은 Phase A 후보가 된다

어제 temporal history는 hard rejection truth로 쓰기 어렵다고 했다.

하지만 **occluder selection priority**에는 매우 유용하다.

이전 frame에 visible했고:

- motion 작음
- topology unchanged
- projected area 큼
- opaque
- current frustum 안

이면 Phase A 우선순위를 높일 수 있다.

여기서는 history가 틀려도 보통 occluder가 depth test에서 실패하거나 screen coverage가 줄어드는 성능 문제로 끝난다.

즉 temporal history는 **cull authority보다 occluder scheduling hint로 쓰는 것이 더 안전하다.**

### 4.18 Dynamic SDF에서는 “stable occluder class”가 중요하다

Process visualization에서 모든 mesh chunk가 같은 update rate를 가지지 않는다.

예:

```text
substrate       → 거의 static
wafer bulk      → stable
large oxide     → 비교적 stable
etch front      → dynamic
thin deposition → dynamic
moving iso-front→ dynamic
```

Stable layer/chunk는 Phase A occluder로 높은 점수를 줄 수 있다.

Dirty SDF brick은:

```text
occluderStability = low
```

로 내려 temporal/history 기반 우선순위에서 제외할 수 있다.

### 4.19 Cross-section / transparency mode에서는 occluder semantics가 바뀐다

Scientific/semiconductor viewer는 cut plane, X-ray, transparency를 자주 사용한다.

이때 평소 좋은 occluder였던 substrate가:

- clipping plane으로 잘림
- transparency 적용
- ghost rendering
- layer isolation

되면 더 이상 occlusion proof source가 아니다.

따라서 occluder eligibility는 geometry property만이 아니라 **render mode state**를 포함해야 한다.

### 4.20 Occluder selection metadata의 SoA

GPU selection pass가 자주 읽는 필드:

```text
OccluderCandidateSoA
- projectedArea[]
- depthKey[]
- rasterCost[]
- opacityClass[]
- stabilityBits[]
- geometryHandle[]
- generation[]
```

Depth raster에 필요한 physical geometry는 selection 후에 resolve한다.

이렇게 하면 selection 단계가 full render metadata를 읽지 않아도 된다.

또 relocation-safe architecture와 연결해 candidate에는 logical handle을 유지할 수 있다.

### 4.21 Top-K exact selection이 필요하지 않을 수 있다

Occlusion value score의 순위가 완벽할 필요는 없다.

다음과 같은 approximate selection이 가능하다.

- score threshold
- per-depth-bin quota
- per-screen-tile quota
- subgroup local winners
- previous-frame selected set reuse
- fixed GPU time budget까지 append

목표는 최적 조합 문제를 정확히 푸는 것이 아니라 **선택 비용보다 충분히 큰 downstream 절감**을 얻는 것이다.

### 4.22 Tile coverage를 고려하면 duplicate occluder를 줄일 수 있다

Large wall A와 wall B가 화면에서 거의 같은 영역을 덮으면 둘 다 Phase A에 넣어도 두 번째 occluder의 marginal value가 작다.

단순 projected-area score는 이 중복을 모른다.

더 발전된 heuristic은:

```text
Marginal Occluder Value
≈
new screen coverage
× hidden-work estimate
```

를 본다.

완전한 coverage optimization은 비싸므로 low-resolution screen tiles에 occupancy bitset을 두고 “새 tile을 얼마나 덮는가” 정도를 근사할 수 있다.

이것은 occluder selection을 **screen-space set coverage problem**으로 보는 관점이다.

### 4.23 Phase A depth cost와 Phase B saving을 프레임별로 측정한다

Two-phase visibility는 scene dependent하다.

낮은 occlusion scene:

```text
Phase A cost > Phase B saving
```

일 수 있다.

그래서 runtime telemetry가 중요하다.

대표 지표:

- selected occluder count
- Phase A meshlet/triangle count
- Phase A depth time
- partial HZB build time
- Phase B cull time
- HZB rejected candidate count
- rejected mesh shader invocations
- reduced fragment invocations
- total visibility system time
- final main-pass time
- saved downstream time estimate

Occlusion이 낮은 camera angle에서는 Phase A budget을 줄이거나 temporal-only/frustum-only 경로로 전환할 수 있다.

### 4.24 Overdraw와 duplicated geometry work의 균형

Depth prepass의 대표적인 장점은 overdraw 감소다.

하지만 geometry-heavy scene에서는 vertex/mesh processing을 두 번 수행한다.

따라서 renderer 병목에 따라 답이 달라진다.

#### Fragment-bound

Depth prepass/occluder pass가 강하게 유리할 수 있음.

#### Geometry/mesh-shader-bound

Selective occluder만 depth에 넣는 것이 더 유리할 수 있음.

#### Bandwidth-bound

Position-only depth stream, compressed proxy, stable meshlet table이 중요.

즉 Two-phase visibility는 “항상 depth prepass를 하자”가 아니라 **현재 병목에 맞춰 occluder raster budget을 제한하는 방법**이다.

### 4.25 Debug view는 occluder와 occludee를 분리해 보여준다

Useful overlays:

```text
Phase A selected occluder
Rejected by Phase B HZB
Visible after Phase B
Unsafe proxy rejected
Dirty/stability bypass
History-prioritized occluder
```

추가로:

- occluder score
- projected area
- depth bin
- raster cost
- screen tile coverage
- HZB mip
- rejection reason

을 보여주면 “왜 이 meshlet을 먼저 그렸는가?”를 디버깅하기 쉽다.

---

## 5. 내 관심 분야와 연결

### Semiconductor / CFD visualization

이 구조는 dynamic scientific geometry에 특히 잘 맞는다.

반도체 process viewer를 생각하면:

- substrate/wafer bulk는 넓고 안정적인 occluder
- etched/deposited front는 dynamic
- thin oxide는 occluder로는 약할 수 있음
- trench/cavity는 proxy가 hole을 막으면 절대 안 됨
- cutaway/transparency mode에서는 occluder eligibility 변경

즉 sparse brick 또는 material layer에 **occluder stability/eligibility metadata**를 붙이는 구조가 자연스럽다.

### SDF → Meshlet → Visibility pipeline

```text
CUDA / Warp SDF
    ↓
Incremental Mesh Extraction
    ↓
Meshlet + Conservative Bounds
    ↓
Occluder Candidate Metadata
    ↓
Phase A Current Depth
    ↓
Partial HZB
    ↓
Phase B Meshlet Cull
    ↓
Indirect Mesh Tasks
```

이 pipeline은 CPU readback 없이 GPU-stay-GPU로 유지할 수 있다.

### Sparse volume과 screen-space visibility의 연결

Sparse brick은 world-space hierarchy,
HZB는 screen-space hierarchy다.

Two-phase visibility에서는 이 둘을 연결한다.

```text
Brick/Chunk stability
      ↓
Occluder priority
      ↓ project
Screen coverage
      ↓
Depth hierarchy
```

즉 sparse spatial data structure가 rendering workload scheduling까지 영향을 준다.

### Vulkan / engine 관점

이 주제는:

- mesh shader
- indirect count
- compute culling
- depth resource
- synchronization
- queue scheduling
- relocation-safe handles
- hot/cold metadata

를 한 번에 연결한다.

Game-engine graphics role에서 강한 포인트는 **“occlusion culling을 한다”**가 아니라:

> 어떤 representation은 occludee bound로만 안전하고, 어떤 representation은 occluder로도 안전한지 구분하고, 그 차이가 false-positive와 false-negative에 어떤 영향을 주는지 설명할 수 있는가

이다.

---

## 6. 머릿속에 남길 질문 3개

1. **왜 AABB는 좋은 occludee test bound가 될 수 있지만 일반적으로 occluder proxy로 rasterize하면 위험하며, outer-conservative와 inner-conservative의 차이는 무엇인가?**
2. **Phase A occluder budget을 object 개수가 아니라 GPU work budget으로 잡는다면 projected area, raster cost, stability, hidden-work estimate 중 어떤 값이 실제 profiler와 가장 잘 상관될까?**
3. **Dynamic SDF viewer에서 cut plane·transparency·dirty brick이 자주 바뀔 때 occluder eligibility를 geometry metadata와 render-mode state 중 어디까지 결합해야 snapshot consistency를 유지할 수 있을까?**

---

## 7. graphics engineer 면접 질문 1개와 답변

### 질문

**“Occlusion culling용 depth를 빠르게 만들기 위해 큰 object의 AABB를 먼저 depth-only로 rasterize하면 좋은 최적화 아닌가요?”**

### 답변

일반적으로 안전하지 않다.

AABB는 actual geometry를 포함하는 **outer-conservative bound**이기 때문에 occludee를 검사할 때는 좋다. Bound가 실제 geometry보다 커도 일부 가려진 object를 visible로 남기는 false positive가 생길 뿐 rendering correctness는 유지된다.

하지만 같은 AABB를 occluder로 rasterize하면 실제 geometry가 없는 공간에도 depth가 기록된다. 그 뒤쪽에 실제 visible object가 있으면 HZB가 그 object를 완전히 가렸다고 판단할 수 있고, 이것은 false negative다.

Occluder proxy는 반대로 actual opaque coverage 안쪽에 있어야 한다.

```text
Occludee Bound:
Actual Geometry ⊂ Bound

Occluder Proxy:
Proxy Coverage ⊂ Actual Opaque Geometry
```

따라서 arbitrary visual LOD나 convex hull도 자동으로 safe occluder가 아니다. Simplification이 silhouette를 바깥으로 이동시키거나 실제 hole을 막으면 위험하다.

Production renderer에서는 large projected area, opaque/stable state, low raster cost를 가진 geometry를 Phase A 후보로 선택하고, actual geometry 또는 inner-conservative proxy만 depth에 넣는 것이 안전하다. 이후 partial current-frame HZB로 나머지 meshlet을 cull한다.

핵심은:

> **Occlusion culling에서 “보수적”이라는 말은 occludee와 occluder에 대해 서로 반대 방향을 뜻한다.**

---

## 8. 포트폴리오 / 커리어 연결

이 주제는 graphics engineer가 visibility algorithm을 단순 API 기능이 아니라 **cost model + geometry semantics + GPU scheduling**으로 이해한다는 것을 보여주기 좋다.

### Rendering

- depth prepass
- early-Z
- Hi-Z/HZB
- occlusion culling
- overdraw
- Standard-Z / Reversed-Z
- alpha-tested occluder semantics

### Geometry

- outer-conservative occludee bound
- inner-conservative occluder proxy
- silhouette-safe simplification
- meshlet / coarse LOD
- thin feature와 hole preservation

### GPU Compute

- occluder scoring
- approximate top-K / binning
- screen-tile coverage
- Phase B worklist compaction
- indirect command/count generation

### Memory Layout

- `OccluderCandidateSoA`
- position-only depth payload
- logical geometry handle
- stability/generation bits
- culling hot metadata와 shading cold metadata 분리

### Vulkan

- depth attachment write
- HZB compute read/write
- compute → draw-indirect synchronization
- task/mesh shader work generation
- async compute trade-off

### Dynamic Simulation / Visualization

- dirty SDF brick
- topology change
- material layer visibility
- cut plane / transparency
- stable substrate vs dynamic surface front

면접에서는 다음처럼 설명할 수 있으면 강하다.

> **“Full depth prepass는 current occlusion은 정확하지만 geometry work를 두 번 지불할 수 있습니다. 그래서 projected coverage와 raster cost가 좋은 stable opaque meshlet만 Phase A에 넣고 current partial HZB를 만든 뒤, 나머지를 Phase B에서 cull합니다. 이때 occludee bound는 outer-conservative, occluder proxy는 inner-conservative여야 하며, 성능 평가는 occluder pass 비용 대비 줄어든 downstream mesh/raster work로 합니다.”**

이 답변은 algorithm, GPU architecture, memory layout, Vulkan synchronization을 한 번에 연결한다.

---

## 9. 내일 이어서 볼 개념

**Visibility-Driven Front-to-Back GPU Scheduling: Depth-Key Binning, Meshlet Reordering, and Overdraw-Aware Work Prioritization**

오늘은 **어떤 geometry를 먼저 occluder로 사용할 것인가**를 봤다.

내일은 그 선택 결과를 더 일반적인 rendering order 문제로 확장한다.

```text
Temporal Hi-Z
    ↓
Occluder Selection
    ↓
Current Partial HZB
    ↓
Visibility-Driven Work Ordering
    ↓
Front-to-Back Meshlet Scheduling
```

다음 개념에서는:

- exact sort vs depth binning
- screen-tile binning
- front-to-back ordering
- material locality와 depth locality의 충돌
- meshlet reorder
- indirect worklist reorder
- overdraw와 cache locality trade-off
- transparent pass와 opaque pass의 차이
- GPU sort/scan 비용
- temporal order reuse

를 중심으로 이어간다.

---

## 10. 참고 키워드

- Occluder Selection
- Two-Phase Visibility
- Depth-Only Prepass
- Selective Depth Prepass
- Current-Frame Hi-Z / HZB
- Partial Depth Pyramid
- Occludee Bound
- Occluder Proxy
- Outer-Conservative Bound
- Inner-Conservative Proxy
- False Positive / False Negative
- Early-Z
- Overdraw
- Reversed-Z
- Meshlet
- Mesh Shader / Task Shader
- GPU-Driven Rendering
- Indirect Draw / Indirect Count
- Projected Screen Area
- Occluder Value
- Occluder Filtering
- Front-to-Back Ordering
- Depth Binning
- Screen-Tile Coverage
- Worklist Compaction
- Stable Occluder
- Dynamic Occluder
- Dirty SDF Brick
- Visibility Epoch
- CUDA-Vulkan Interop
- Depth Attachment Synchronization
- Hi-Z Compute Pass
- Jiří Bittner et al., **“Coherent Hierarchical Culling: Hardware Occlusion Queries Made Useful,” Computer Graphics Forum, 2004**
- Oliver Mattausch et al., **“CHC++: Coherent Hierarchical Culling Revisited,” Computer Graphics Forum, 2008**
- Gi Beom Lee et al., **“Hierarchical Raster Occlusion Culling,” Computer Graphics Forum, 2021**
- Khronos Vulkan Documentation Project — **Mesh Shader Culling**
- Khronos Vulkan Documentation Project — **GPU Rendering and Multi-Draw Indirect**
- Khronos Vulkan Specification — **Occlusion Queries**
- NVIDIA — **Using Mesh Shaders for Professional Graphics**
- NVIDIA — **Introduction to Turing Mesh Shaders**
