---
title: "Temporal Hi-Z Occlusion Culling for Dynamic Geometry: Reprojection Error, Conservative Depth Bounds, and Visibility Hysteresis"
date: "2026-09-09"
category: Graphics
tags: [GPU, Rendering, GPU-Driven Rendering, Hi-Z, HZB, Occlusion Culling, Temporal Coherence, Reprojection, Reversed-Z, Meshlet, Mesh Shader, Vulkan, CUDA, Dynamic Geometry, Memory Layout, C++]
level: intermediate
---

# [Daily Graphics Growth] 2026-09-09 - Temporal Hi-Z Occlusion Culling for Dynamic Geometry: Reprojection Error, Conservative Depth Bounds, and Visibility Hysteresis

## 1. 오늘의 개념

어제는 **GPU-Driven Meshlet Visibility Pipelines**에서 `chunk/instance → meshlet → primitive` 계층으로 visibility cost를 줄이고, compact worklist와 indirect count를 GPU 내부에서 연결하는 구조를 봤다. 오늘은 그 흐름에서 가장 correctness 위험이 큰 **previous-frame Hi-Z(Hierarchical Z / HZB)** 를 다룬다.

Hi-Z는 depth buffer를 min/max reduction으로 mip hierarchy화한 뒤 object 또는 meshlet의 screen-space bounding rectangle을 coarse depth와 비교해 occlusion을 빠르게 판단한다. 문제는 이전 frame의 depth를 현재 frame에 재사용할 때 생긴다.

```text
Current:
camera pose
dynamic geometry
meshlet bounds
LOD / relocation snapshot

Previous:
depth buffer
occluder position
visibility history
HZB coverage
```

이때 previous HZB는 단순 texture가 아니라 **특정 camera·geometry snapshot이 특정 screen region을 가렸다는 temporal evidence**다.

핵심 질문은 다음이다.

> **camera motion, moving occluder, SDF deformation, disocclusion이 있는 상황에서 previous-frame Hi-Z를 사용하면서 실제 visible geometry를 잘못 제거하는 false negative를 어떻게 막는가?**

Temporal coherence는 강력한 **prediction**이지만 arbitrary dynamic scene에서 자동으로 current-frame **proof**가 되지는 않는다.

---

## 2. 한 줄 핵심

> Temporal Hi-Z의 핵심은 **이전 depth를 현재 visibility의 사실이 아니라 versioned evidence로 취급하고, reprojection coverage·conservative screen/depth expansion·history invalidation·visibility hysteresis를 통해 불확실한 경우는 visible/unknown으로 남기는 것**이다.

---

## 3. 왜 중요한가

Previous-frame Hi-Z의 장점은 current depth prepass를 기다리지 않고 frame 초반부터 GPU culling을 시작할 수 있다는 것이다.

```text
Previous HZB
   ↓
Early GPU Cull
   ↓
Compacted Work
   ↓
Rendering
```

반면 current-frame Hi-Z는 보통 다음과 같은 dependency를 가진다.

```text
Depth Prepass
   ↓
HZB Build
   ↓
Occlusion Cull
   ↓
Main Rendering
```

Previous HZB는 scheduling freedom이 크지만 다음 변화에 취약하다.

- camera 이동/회전
- occluder 이동·삭제
- occludee 이동
- SDF deposition/etch에 따른 surface 변화
- topology 변경
- meshlet rebuild/LOD 변경
- TAA jitter / dynamic resolution

가장 위험한 경우는 **stale occluder**다. 이전 frame에서 벽이 뒤쪽 meshlet을 가렸지만 이번 frame에서 벽이 이동했다면, previous depth를 그대로 믿는 순간 뒤쪽 meshlet을 잘못 cull할 수 있다.

이것은 단순 performance miss가 아니라 화면에서 geometry가 사라지는 correctness failure다.

고전적인 Coherent Hierarchical Culling(CHC) 계열이 이전 visibility를 hierarchy traversal과 query scheduling에 활용한 이유도 temporal coherence를 **work scheduling hint**로 사용하기 위해서다. 이전 결과를 current truth로 무조건 사용하는 것과는 다르다.

Dynamic SDF/CFD geometry에서는 history가 더 빨리 낡는다. surface 자체가 생성·삭제·변형되고 meshlet partition까지 바뀔 수 있기 때문이다.

---

## 4. 구현 관점

### 4.1 Previous HZB를 사용하는 두 방식

#### A. Current bound를 previous camera로 투영

현재 world-space bound를 `PrevViewProj`로 투영해 previous HZB와 비교한다.

```text
Current World Bound
    ↓ PrevViewProj
Previous Screen Rect
    ↓
Previous HZB
```

Static world geometry에서는 단순하지만 object가 이동했거나 SDF가 변형되었다면 previous frame 당시의 위치와 다르다. 따라서 motion uncertainty, swept bound, dirty-geometry 상태가 함께 필요하다.

#### B. Previous depth를 current frame으로 reproject

Previous depth를 world/view space로 복원한 뒤 current camera로 다시 투영한다.

```text
Previous Depth
    ↓ unproject
Previous World Position
    ↓ current projection / motion
Current Screen
    ↓
Reprojected History
```

이 방식은 current screen에서 바로 query할 수 있지만 **hole, many-to-one splat, disocclusion, moving occluder** 문제가 생긴다.

어느 방식이든 단순 matrix multiply만으로 temporal correctness가 해결되지는 않는다.

### 4.2 Reprojection hole은 `UNKNOWN`

이전 frame에서 foreground occluder 뒤에 숨은 background는 depth buffer에 존재하지 않는다. Camera가 이동해 그 background가 드러나면 reprojected history에 hole이 생긴다.

Occlusion culling에서는 이 hole을 주변 depth로 채워 occluder로 사용하면 위험하다.

```text
valid conservative history → proof candidate
hole / invalid             → UNKNOWN
```

UNKNOWN은 cull이 아니라 통과시키는 방향으로 처리해야 한다.

Temporal denoising에서는 hole filling이 품질 문제지만, occlusion culling에서는 잘못 채운 history가 실제 geometry를 없애는 correctness 문제다.

### 4.3 `DepthHistory`와 `HistoryValidity`를 분리한다

Temporal HZB는 논리적으로 다음 두 resource의 결합으로 보는 편이 좋다.

```text
DepthHistory
HistoryValidity
```

Validity를 무효화하는 대표 이벤트:

- camera cut
- projection/FOV 변경
- viewport/resolution 변경
- dynamic occluder update
- SDF topology change
- mesh representation/LOD change
- resource resize
- history generation mismatch

즉 HZB는 `depth + provenance`다.

### 4.4 Dirty SDF brick은 temporal visibility도 invalidate한다

Dynamic sparse SDF에서는 이미 dirty brick bitset이 존재할 수 있다.

```text
Dirty SDF Brick
    ↓
Dirty Geometry
    ↓
Dirty Meshlet / Bound
    ↓
Dirty Temporal Occlusion Region
```

Brick이 바뀌었다면 이전 frame에서 그 brick이 만든 occlusion도 더 이상 신뢰할 수 없다. 따라서 dirty brick의 projected region을 temporal HZB의 invalid region으로 전파할 수 있다.

이렇게 보면 simulation dirty metadata가 remeshing뿐 아니라 rendering history invalidation에도 사용된다.

### 4.5 Occluder motion과 occludee motion은 다르다

**Occludee motion**은 query bound 문제다.

- current/previous transform 차이
- swept bound
- screen-space expansion

**Occluder motion**은 history trust 문제다.

- 이전 HZB 자체가 stale
- moving occluder가 만든 tile invalidation
- current depth refresh 필요

이 둘을 같은 “motion bias”로 처리하면 reasoning이 흐려진다.

### 4.6 Screen rectangle은 uncertainty 방향으로 넓힌다

Motion/reprojection 오차가 있으면 projected rect를 넓힌다.

```text
R_safe = dilate(R_projected, Δscreen)
```

`Δscreen`의 원인은 다음과 같다.

- camera angular/linear motion
- object velocity
- bound quantization
- TAA jitter difference
- dynamic resolution mapping
- numerical reprojection error

Rect를 넓히면 culling power는 떨어지지만 false negative 위험은 줄어든다.

### 4.7 Depth bound는 camera 쪽으로 bias한다

Occludee의 nearest depth를 너무 멀게 추정하면 false occlusion이 가능하다.

따라서 uncertainty가 있을 때는 occludee depth bound를 camera 쪽으로 더 가깝게 확장하는 편이 conservative하다.

```text
z_near_safe = toward_camera(z_near, Δz)
```

Temporal Hi-Z에서 안전한 방향은 세 가지다.

```text
screen rect      → 더 넓게
object near depth→ 더 가깝게
invalid history  → UNKNOWN
```

### 4.8 Standard-Z와 Reversed-Z의 reduction은 반대다

**Standard-Z**

```text
near = 0
far  = 1
depth test = LESS
```

Tile 전체의 occlusion proof에는 가장 먼 occluder가 필요하므로 보통 **max reduction**을 사용한다.

**Reversed-Z**

```text
near = 1
far  = 0
depth test = GREATER
```

가장 먼 occluder는 더 작은 값이므로 대응되는 conservative hierarchy는 보통 **min reduction**이다.

핵심은 `Hi-Z = max`를 외우는 것이 아니라 **depth encoding + compare convention에서 reduction operator를 유도하는 것**이다.

### 4.9 Reversed-Z는 temporal mismatch를 해결하지 않는다

Reversed-Z는 floating-point depth precision을 크게 개선하지만 다음 문제는 그대로다.

- stale occluder
- disocclusion
- reprojection hole
- camera motion
- dynamic geometry

즉:

```text
Reversed-Z       → depth precision
Temporal validity→ time consistency
```

서로 다른 문제다.

### 4.10 Mip 선택은 coverage proof의 일부다

Object rect가 크면 coarse mip을 선택해 sample 수를 줄인다. 그러나 rect가 여러 texel을 덮는데 texel 하나만 읽는 식의 shortcut은 conservative proof를 깨뜨릴 수 있다.

Correctness는 다음의 조합에서 나온다.

- conservative pyramid reduction
- rect를 완전히 커버하는 mip/sample rule
- point/texel 기반 extremum fetch
- conservative object depth bound

따라서 mip selection은 단순 performance heuristic가 아니라 coverage contract와 함께 봐야 한다.

### 4.11 Bilinear filtering은 extremum semantics를 깨뜨릴 수 있다

HZB는 visual smoothness가 아니라 min/max extremum을 보존하는 resource다. 일반 bilinear filtering은 texel 사이 값을 interpolation하므로 conservative min/max 의미를 그대로 유지한다고 볼 수 없다.

Occlusion용 HZB는 보통 explicit texel/point sampling 관점으로 reasoning하는 편이 명확하다.

### 4.12 TAA jitter도 temporal coordinate state다

Camera transform이 동일해도 previous/current projection jitter가 다르면 thin meshlet의 screen rect가 이동한다.

따라서 temporal projection snapshot에는 다음이 함께 들어간다.

```text
camera pose
projection
jitter
viewport
resolution
```

Dynamic resolution까지 사용하면 history mapping도 함께 versioned되어야 한다.

### 4.13 Camera cut은 history reset 이벤트다

작은 motion은 conservative expansion으로 다룰 수 있지만 teleport, cinematic cut, 큰 FOV 변경은 temporal coherence 자체가 사라진다.

이 경우 previous HZB를 억지로 재사용하기보다 history를 invalid로 보는 편이 안전하다.

### 4.14 Visibility history는 binary보다 confidence state가 낫다

단순 `visibleLastFrame`보다 다음처럼 상태를 나누면 temporal 정책을 표현하기 쉽다.

```text
RecentlyVisible
Candidate
TemporallyOccluded
Unknown
```

함께 저장할 수 있는 metadata:

```text
occludedStreak
lastVisibleFrame
historyAge
generation
```

### 4.15 Hysteresis는 성능을 희생해 correctness 방향으로 bias한다

대표적인 정책은 한 번의 temporal occlusion 결과로 즉시 제거하지 않고 여러 번 연속으로 occluded일 때 강하게 cull하는 것이다.

또는 previous frame visible이었던 meshlet은 한 frame 정도 benefit of doubt를 줄 수 있다.

이 정책은 invisible work를 조금 더 render하지만 popping과 false-negative 가능성을 줄인다.

Hysteresis는 strict proof가 아니라 **uncertainty를 performance loss 방향으로 흡수하는 engineering policy**다.

### 4.16 Previously visible geometry를 먼저 current depth에 반영하는 hybrid

Pure previous-frame HZB와 full depth prepass 사이에 hybrid가 있다.

```text
Phase A:
previously visible / stable large occluders
    ↓ depth

Partial Current HZB
    ↓

Phase B:
previously occluded / unknown candidates
    ↓ cull & render
```

이 구조는 temporal coherence를 사용하면서 current-frame occlusion evidence를 빠르게 만든다.

CHC 계열의 “previous visibility를 먼저 활용해 current work를 scheduling한다”는 사고방식과 연결된다.

### 4.17 모든 geometry를 temporal occluder로 쓰지 않는다

Temporal proof source에 적합한 geometry:

- static 또는 motion이 작은 geometry
- large opaque surface
- stable LOD/meshlet representation
- update epoch가 안정적인 geometry

덜 적합한 geometry:

- rapidly moving
- topology-changing SDF region
- translucent
- thin/subpixel
- newly spawned

즉 occludee와 occluder의 eligibility policy를 분리할 수 있다.

### 4.18 History metadata도 hot/cold split이 가능하다

Millions of meshlets에서는 visibility history 4바이트 차이도 누적된다.

Hot data:

```text
visibility bits
age / streak
generation
```

Cold/debug data:

```text
last reason
last sampled mip
last HZB depth
source dirty brick
```

Depth history는 full-resolution pyramid, validity는 coarse tile mask/bitset로 분리하는 것도 bandwidth 절약에 유리하다.

### 4.19 History entry도 generation check가 필요하다

Meshlet slot이 free된 뒤 새 meshlet이 같은 physical slot을 재사용하면 old history를 상속해서는 안 된다.

```text
MeshletHandle = {index, generation}
```

History와 current meshlet generation이 다르면 visibility state를 reset해야 한다.

어제의 relocation-safe handle 원칙이 temporal data에도 그대로 적용된다.

### 4.20 Geometry epoch와 depth-history epoch를 분리한다

같은 frame 번호를 사용해도 producer timing은 다를 수 있다.

- CUDA는 이미 N+1 geometry를 생성
- Vulkan은 아직 N depth를 HZB로 사용
- relocation table은 다른 snapshot일 수 있음

따라서 다음을 구분한다.

```text
geometryEpoch
boundEpoch
depthHistoryEpoch
visibilityEpoch
relocationEpoch
```

Barrier가 맞아도 epoch 의미가 섞이면 semantic correctness는 깨진다.

### 4.21 CUDA-Vulkan interop에서 semaphore와 validity는 다른 역할이다

External semaphore는 작업 순서와 memory visibility를 전달한다.

그러나 어느 dirty brick의 old occlusion을 버려야 하는지는 semantic metadata가 결정한다.

```text
Synchronization != History Validity
```

GPU-stay-GPU pipeline에서도 이 구분이 중요하다.

### 4.22 Current/Previous hybrid는 scene motion에 따라 바뀔 수 있다

History invalid ratio가 낮은 static scene:

```text
Previous HZB aggressive
Current refresh small
```

Dynamic CFD/SDF timestep:

```text
Previous HZB conservative
Dirty region bypass
Current depth/HZB 비중 증가
```

즉 temporal strategy도 runtime scene statistics에 따라 조절할 수 있다.

### 4.23 Occlusion query와 HZB가 공유하는 교훈

Vulkan occlusion query는 실제 fragment test를 통과한 sample 수를 future rendering decision에 사용할 수 있다. HZB는 application이 depth hierarchy와 bound test를 직접 만든다는 차이가 있다.

공통점:

- visibility result에는 latency/history가 존재
- hierarchy와 temporal coherence가 중요
- test 자체에도 비용이 존재
- occlusion이 적은 scene에서는 overhead가 이득을 넘을 수 있음

### 4.24 Debugger는 “왜 cull됐는가”를 기록해야 한다

Temporal bug는 한 frame만 나타날 수 있다.

Reason code 예:

```text
CULLED_VALID_HIZ
VISIBLE_HISTORY_INVALID
VISIBLE_HYSTERESIS
BYPASS_DIRTY_GEOMETRY
BYPASS_CAMERA_CUT
BYPASS_REPROJECTION_HOLE
GENERATION_MISMATCH
```

Debug overlay에서는 query rect, selected mip, object near depth, HZB depth, history age, validity를 함께 보는 것이 좋다.

### 4.25 프로파일링에서 볼 지표

- temporal candidate count
- temporal reject ratio
- history-invalid bypass ratio
- reprojection hole ratio
- dirty-region invalidation ratio
- average HZB query mip
- HZB samples per candidate
- hysteresis pass-through count
- previously-visible reuse ratio
- depth pyramid/reprojection time
- history metadata bandwidth
- generation mismatch count
- culling으로 줄어든 task/mesh/raster work

핵심 derived metric:

```text
saved downstream GPU work
-------------------------
temporal visibility cost
```

Temporal system이 복잡해졌는데 downstream work가 충분히 줄지 않으면 최적화가 아니다.

---

## 5. 내 관심 분야와 연결

### Semiconductor process visualization

Deposition, etch, CMP처럼 surface 자체가 변하는 geometry에서는 previous depth가 빠르게 stale해질 수 있다.

예를 들어 이전 frame의 trench sidewall이 뒤쪽 structure를 가렸는데 etch로 sidewall이 줄어들었다면, 그 sidewall이 만든 previous occlusion은 즉시 신뢰도를 잃는다.

그래서 dirty SDF brick은 다음과 같이 연결할 수 있다.

```text
dirty field
 → dirty mesh
 → dirty bound
 → temporal occluder invalidation
```

### Sparse volume hierarchy

Sparse brick key를 그대로 visibility group key로 쓰면 field와 renderer의 spatial identity를 공유할 수 있다.

```text
Sparse Brick
  ↓
Mesh Chunk
  ↓
Meshlet Range
  ↓
Temporal Visibility Group
```

### GPU-stay-GPU

관심 pipeline과 연결하면:

```text
CUDA / Warp
  ↓
Dynamic SDF
  ↓
Dirty Brick Bitset
  ↓
Incremental Mesh / Meshlet
  ↓
Relocation-Safe Table
  ↓
Temporal History Invalidation
  ↓
Hi-Z Culling
  ↓
Indirect Mesh Task
  ↓
Vulkan Rendering
```

CPU readback 없이도 전체 흐름을 GPU에 남길 수 있다. 중요한 것은 zero-copy 자체보다 **snapshot semantics**다.

### CFD / scientific visualization

Turbulent iso-surface처럼 timestep 간 변화가 크면 history invalid ratio가 높다. 이 경우 temporal HZB보다 current-frame depth 기반 culling이 상대적으로 유리할 수 있다.

즉 visibility system은 scene motion statistics에 적응해야 한다.

### Game engine

이 개념은 large-world rendering, destruction, doors/moving walls, crowds, virtualized geometry, TAA jitter, dynamic resolution과 직접 연결된다.

Game-engine graphics role에서 중요한 설명은 “HZB를 쓴다”가 아니라 **previous depth의 false-negative 경로와 conservative fallback policy를 설명할 수 있는가**다.

---

## 6. 머릿속에 남길 질문 3개

1. **Previous-frame HZB가 meshlet을 occluded라고 판단했을 때, current-frame hard rejection으로 사용하려면 occluder motion과 reprojection coverage에 대해 어떤 추가 조건이 필요한가?**
2. **Dynamic SDF의 dirty brick mask를 temporal visibility invalidation과 연결한다면 field-space dirty region과 screen-space HZB invalid region 사이의 dependency를 어떻게 정의해야 하는가?**
3. **Visibility hysteresis가 invisible work를 더 render하는 비용을 만든다면, 어떤 profiler 지표로 hysteresis window의 적정 크기를 판단해야 하는가?**

---

## 7. graphics engineer 면접 질문 1개와 답변

### 질문

**“Previous-frame Hi-Z에서 meshlet의 bounding rectangle이 완전히 occluded라고 나왔습니다. 그러면 이번 frame에서도 안전하게 cull할 수 있나요?”**

### 답변

Arbitrary dynamic scene에서는 안전하다고 볼 수 없다.

Previous-frame HZB는 이전 camera와 이전 occluder geometry가 만든 depth다. 이번 frame에서 camera가 움직였거나 occluder가 이동·삭제·변형되면 이전 depth는 current coverage를 보장하지 않는다.

특히 disocclusion이 핵심이다. 이전 frame에서 foreground 뒤에 숨겨져 depth가 없던 background가 이번 frame에 드러날 수 있다. Reprojection hole이나 invalid tile을 주변 depth로 채워 occluder처럼 사용하면 실제 visible geometry를 false-negative cull할 수 있다.

Robust한 방향은 다음과 같다.

- previous depth를 history evidence로 취급
- invalid/hole 영역은 `UNKNOWN`
- screen rect는 motion/error만큼 확대
- object near depth는 camera 쪽으로 conservative bias
- moving/dirty occluder의 history는 invalidate
- camera cut에서는 history reset
- history entry는 generation check
- uncertain case에는 hysteresis 또는 visible bias
- 가능하면 stable occluder를 먼저 current depth에 반영한 뒤 second-stage Hi-Z 사용

또 Standard-Z `near=0/far=1, LESS`에서는 보통 max reduction, Reversed-Z `near=1/far=0, GREATER`에서는 대응되는 min reduction을 사용한다.

핵심은:

> **Temporal coherence는 predictor이고, current-frame occlusion proof가 되려면 spatial coverage와 temporal validity가 함께 보장되어야 한다.**

---

## 8. 포트폴리오 / 커리어 연결

이 주제는 **rendering algorithm → GPU compute → dynamic geometry → memory lifetime → Vulkan synchronization**을 하나로 설명하기 좋다.

포트폴리오에서 강조할 수 있는 관점:

- HZB min/max semantics와 reversed-Z
- previous/current depth trade-off
- disocclusion과 reprojection validity
- conservative screen/depth bounds
- visibility hysteresis
- SDF dirty-region history invalidation
- generation-checked temporal metadata
- depth/geometry/relocation epoch 분리
- CUDA producer와 Vulkan consumer의 semantic snapshot
- bit-packed visibility history와 hot/cold metadata
- reason-coded debug visualization
- culling cost 대비 saved downstream work profiler

면접에서 강한 설명은 다음과 같다.

> **“Previous-frame HZB는 depth prepass latency를 줄이지만 stale occluder가 false negative를 만들 수 있으므로 depth와 validity를 분리합니다. Dirty dynamic geometry와 camera cut은 history를 invalidate하고, uncertain region은 visible 방향으로 bias합니다. 그 위에 hysteresis를 사용하고, profiler에서는 temporal pass 비용 대비 실제 줄어든 mesh/task/raster work를 측정합니다.”**

---

## 9. 내일 이어서 볼 개념

**Occluder Selection and Two-Phase GPU Visibility: Depth-Only Prepasses, Coarse Occluder Rasterization, and Late Hi-Z Rejection**

오늘은 previous-frame HZB의 temporal uncertainty를 봤다. 다음은 current frame에서 **어떤 geometry를 먼저 occluder로 렌더해 신뢰 가능한 depth를 빠르게 만들 것인가**다.

학습 흐름:

```text
relocation-safe meshlet state
 → hierarchical GPU visibility
 → temporal Hi-Z validity
 → stable occluder selection
 → two-phase current-frame visibility
```

다음 개념은 다음을 연결한다.

- occluder value vs raster cost
- large opaque occluder selection
- previously visible occluder reuse
- depth-only / reduced-geometry prepass
- coarse proxy occluder
- phase A / phase B worklist
- partial current HZB
- late occlusion rejection
- overdraw vs duplicated geometry work
- GPU-driven depth-prepass scheduling

---

## 10. 참고 키워드

- Hierarchical Z / Hi-Z / HZB
- Temporal Occlusion Culling
- Previous-Frame Depth
- Current-Frame Depth
- Temporal Coherence
- Reprojection / Disocclusion
- Reprojection Hole
- Coverage Validity
- Conservative Occlusion
- False Positive / False Negative
- Screen-Space Dilation
- Conservative Near Depth
- Standard-Z / Reversed-Z
- Min / Max Depth Reduction
- TAA Jitter
- Camera Cut
- Visibility Hysteresis
- Occlusion Confidence
- Stable / Unstable Occluder
- Dynamic Geometry
- Dirty SDF Brick
- Geometry / Depth-History / Visibility Epoch
- Generation-Checked History
- GPU-Driven Rendering
- Meshlet Visibility
- Indirect Count
- CUDA-Vulkan Interop
- Coherent Hierarchical Culling (CHC)
- CHC++
- Jiří Bittner et al., **“Coherent Hierarchical Culling: Hardware Occlusion Queries Made Useful,” Computer Graphics Forum, 2004**
- Oliver Mattausch et al., **“CHC++: Coherent Hierarchical Culling Revisited,” Computer Graphics Forum, 2008**
- Gi Beom Lee et al., **“Hierarchical Raster Occlusion Culling,” Computer Graphics Forum, 2021**
- NVIDIA GPU Gems 2, **“Hardware Occlusion Queries Made Useful”**
- NVIDIA, **“Depth Precision Visualized”**
- Khronos Vulkan Documentation, **“Occlusion Queries”**
- Khronos Vulkan Samples, **“GPU Rendering and Multi-Draw Indirect”**
- Khronos Vulkan Documentation, **“Using Pipeline Barriers Efficiently”**
