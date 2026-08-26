---
title: "Residency-Aware Multi-Resolution Fallback: Mip Tails, Coarse Bricks, and Error-Bounded Rendering Under Memory Pressure"
date: "2026-08-27"
category: Graphics
tags: [GPU, Sparse Residency, Multi-Resolution, Mip Tail, Sparse Volume, Brick Streaming, Error Bound, Vulkan, CUDA, Ray Marching, SDF, Level Set, Memory Layout, Compute Shader, C++]
level: intermediate
---

# [Daily Graphics Growth] 2026-08-27 - Residency-Aware Multi-Resolution Fallback: Mip Tails, Coarse Bricks, and Error-Bounded Rendering Under Memory Pressure

## 1. 오늘의 개념
어제는 **Temporal Residency Prediction and Eviction Hysteresis**를 통해 GPU feedback이 가진 지연을 prediction으로 숨기고, resident brick이 경계에서 반복적으로 들어왔다 나가는 **thrashing**을 hysteresis로 줄이는 방법을 봤다.

하지만 prediction과 hysteresis를 잘 설계해도 sparse residency에는 피할 수 없는 순간이 존재한다.

- camera가 갑자기 이동한다.
- ray/volume traversal이 예상하지 못한 region을 만난다.
- VRAM budget이 급격히 줄어든다.
- upload/decompression이 늦어진다.
- fine brick은 요청됐지만 아직 `consumer-visible` 상태가 아니다.

이때 renderer가 선택할 수 있는 가장 나쁜 대응은 **"fine brick이 없으니 그냥 0을 읽는다"** 또는 **"빈 공간으로 간주한다"**는 것이다. Sparse API가 non-resident access를 안전하게 처리해 주더라도, 그것이 시각적으로 올바른 결과를 의미하지는 않는다.

오늘의 주제인 **Residency-Aware Multi-Resolution Fallback**은 이 문제를 다음과 같이 바꿔 생각한다.

> **가장 원하는 해상도의 data가 없을 때, 현재 resident한 더 coarse한 representation 중 오차를 통제할 수 있는 최선의 대체물을 선택한다.**

즉 sparse residency를 단순한 `resident / non-resident` binary 문제로 보지 않고 다음과 같은 계층으로 본다.

`Fine Brick -> Coarse Parent Brick -> Coarser Ancestor -> Mip Tail / Root Representation -> Safe Fallback`

여기서 중요한 것은 **fallback이 단순 LOD 저하가 아니라 availability와 quality를 동시에 다루는 contract**라는 점이다.

- **Availability contract**: 지금 GPU가 실제로 접근 가능한 representation은 무엇인가?
- **Quality contract**: 그 representation이 현재 pixel/ray에 허용되는 error budget 안에 있는가?

좋은 sparse renderer는 "page miss가 발생했는가"만 보는 것이 아니라, **현재 resident한 representation 중 어느 수준까지 내려가면 frame을 깨뜨리지 않고 계속 렌더링할 수 있는가**를 판단한다.

## 2. 한 줄 핵심
> Sparse rendering의 안정성은 fine page의 완전한 residency보다 **항상 resident한 coarse fallback chain과 명시적인 error budget을 결합해, memory pressure를 hole이 아니라 통제된 품질 저하로 변환하는 것**에서 나온다.

## 3. 왜 중요한가

### 3.1 Non-resident read의 API 의미와 시각적 의미는 다르다
Vulkan sparse image에서 `residencyNonResidentStrict`가 지원되면 non-resident 영역을 읽었을 때 0처럼 동작하도록 정의할 수 있다. 지원되지 않더라도 access 자체는 안전하지만 읽은 값은 undefined일 수 있다.

하지만 volume density, SDF, material field에서 `0`은 결코 중립적인 값이 아니다.

- density `0`은 완전히 빈 공간을 의미할 수 있다.
- SDF `0`은 오히려 surface 자체를 의미한다.
- material id `0`은 특정 재질을 가리킬 수 있다.
- doping/temperature scalar `0`은 실제 물리 값처럼 보일 수 있다.

따라서 hardware/API가 정의하는 safe access와 renderer가 원하는 **semantic fallback**은 분리되어야 한다.

### 3.2 Sparse residency의 진짜 품질 문제는 "miss"보다 "무슨 값을 보여 주는가"다
Fine brick miss 자체는 피하기 어렵다. 중요한 것은 miss 이후의 결과다.

두 renderer를 비교해 보자.

- Renderer A: fine brick이 없으면 hole, black voxel, undefined sample이 보인다.
- Renderer B: fine brick이 없으면 coarse parent로 내려가고, parent도 없으면 root/mip-tail representation을 사용한다.

같은 VRAM budget이라도 B는 **resolution degradation**으로 실패하고, A는 **semantic corruption**으로 실패한다.

Production renderer에서는 전자가 훨씬 다루기 쉽다. 품질 저하는 측정·예측·완화할 수 있지만, 잘못된 값은 temporal instability, false surface, missing geometry, NaN propagation 같은 더 큰 문제로 이어진다.

### 3.3 Mip tail은 단순 메모리 규칙이 아니라 "always-resident floor"로 해석할 수 있다
Sparse mipmapped image에서는 작은 mip level들이 개별 sparse block으로 관리되지 않고 **mip tail**이라는 opaque region에 묶인다.

Vulkan은 `imageMipTailFirstLod`, `imageMipTailSize`, `imageMipTailOffset` 등을 통해 그 영역을 보고하며, CUDA sparse mipmapped array도 `miptailFirstLevel`, `miptailSize` 형태의 layout 정보를 제공한다.

이 mip tail을 항상 resident하게 유지하면 renderer 입장에서는 중요한 성질이 생긴다.

> **아무리 fine residency가 무너져도 최소한 coarse representation은 존재한다.**

즉 mip tail은 단순한 API 특수 케이스가 아니라 sparse texture/volume의 **quality floor**가 될 수 있다.

Custom brick hierarchy에서도 같은 원리를 적용할 수 있다. Hardware mip tail이 없더라도 root 또는 몇 단계의 coarse brick을 별도 pool에 항상 resident하게 두면 동일한 fallback floor를 만들 수 있다.

### 3.4 Memory pressure를 binary failure가 아니라 graceful degradation으로 바꿀 수 있다
VRAM budget이 줄어들 때 두 가지 정책이 가능하다.

1. fine brick을 계속 유지하다 budget을 넘으면 갑자기 miss가 발생한다.
2. screen-space importance가 낮은 region부터 더 coarse한 resident level로 내려간다.

두 번째는 메모리 pressure를 품질 정책으로 흡수한다.

예를 들어 8K 또는 3D dense field를 유지할 수 없는 상황에서도 다음과 같은 동작이 가능하다.

- 가까운 surface: LOD 0
- 중간 거리: LOD 1
- 주변부/periphery: LOD 2
- occluded/low-importance region: coarse root only

즉 **resident bytes와 visual error를 교환하는 명시적 trade-off**가 된다.

### 3.5 Simulation/visualization에서는 "coarse fallback 가능"과 "compute correctness"를 구분해야 한다
Rendering은 coarse data로 품질을 낮추면서 계속 진행할 수 있다. 하지만 simulation update가 같은 방식으로 fallback해도 되는 것은 아니다.

예를 들어 level-set advection, CFD stencil, diffusion, reinitialization은 특정 grid resolution과 neighbor dependency를 전제로 할 수 있다. Fine cell이 없다고 coarse parent 값을 그대로 대입하면 numerical operator 자체가 바뀔 수 있다.

따라서 integrated simulation/visualization pipeline에서는 다음을 분리하는 것이 중요하다.

- **Rendering residency**: quality-degradable, coarse fallback 허용 가능
- **Simulation residency**: correctness-critical, active region/halo는 fine residency가 필요할 수 있음

같은 sparse volume을 공유하더라도 consumer별 residency contract는 다를 수 있다.

## 4. 구현 관점

### 4.1 Page table은 "resident인가?"보다 "최선의 resident LOD가 무엇인가?"를 표현해야 한다
가장 단순한 sparse page table은 logical brick마다 physical slot 하나만 저장한다.

`LogicalBrick -> PhysicalSlot or Invalid`

Multi-resolution fallback에서는 이보다 richer한 metadata가 필요하다.

개념적으로는 다음 정보가 중요하다.

- finest requested LOD
- finest resident LOD
- fallback parent/ancestor
- physical slot 또는 image region
- residency epoch
- quality/error metadata
- transition/hysteresis state

핵심은 shader가 miss를 만날 때 CPU에 질문하는 것이 아니라, **현재 frame의 residency snapshot 안에서 즉시 사용할 수 있는 fallback chain을 결정할 수 있어야 한다**는 점이다.

### 4.2 Desired LOD와 Resident LOD를 분리한다
렌더링에서 필요한 해상도는 camera distance만으로 정해지지 않는다.

- projected pixel footprint
- ray cone / differential
- volume ray step size
- surface curvature
- SDF gradient
- local field frequency
- viewport resolution
- foveated/peripheral importance

이런 정보로부터 먼저 **Desired LOD**가 결정된다.

그러나 실제 memory 상태는 다를 수 있다.

`Desired LOD = 0`

`Available Resident LOD = 2`

이때 renderer는 LOD 2를 사용하되, 이것이 acceptable한지 error metric으로 판단한다. 따라서 **LOD selection과 residency selection은 같은 단계가 아니다.**

- LOD policy: 품질 관점에서 무엇이 필요한가?
- Residency policy: 그중 무엇이 현재 존재하는가?

이 둘을 분리해야 memory pressure가 rendering quality에 어떤 영향을 주는지 분석할 수 있다.

### 4.3 Fallback은 coarse ancestor 탐색으로 모델링할 수 있다
3D brick hierarchy를 octree-like 구조로 생각하면 fine brick의 parent는 더 넓은 공간을 절반 해상도로 표현한다.

개념적인 fallback sequence는 다음과 같다.

`LOD0 brick -> LOD1 parent -> LOD2 parent -> ... -> root`

여기서 중요한 것은 traversal cost다. 매 sample마다 여러 ancestor를 탐색하면 page-table traffic이 커질 수 있다.

그래서 runtime metadata에는 현재 logical region에 대해 **best resident ancestor**를 캐시한 view가 유용하다.

예를 들어 renderer가 보는 read-only snapshot은 사실상 다음 함수를 제공한다고 볼 수 있다.

`BestAvailable(logicalPosition, desiredLod) -> {residentLod, physicalLocation, errorBound}`

이 contract는 Vulkan sparse image, CUDA sparse array, custom brick pool 등 backend가 달라도 상위 C++ renderer에서 동일하게 유지될 수 있다.

### 4.4 Coarse representation은 단순 downsample이 아니라 "무엇을 보존해야 하는가"에 따라 달라진다
다중 해상도 volume에서 가장 흔한 실수는 모든 field를 평균 내리면 된다고 생각하는 것이다.

하지만 field의 semantic에 따라 coarse representation의 보존 조건은 다르다.

#### Density / scalar volume
평균값은 저주파 표현에는 유용하지만 작은 고밀도 feature를 약화시킬 수 있다.

따라서 coarse brick에는 평균뿐 아니라 다음 보조 통계가 유용할 수 있다.

- min / max
- variance
- gradient magnitude bound
- opacity bound

이 정보는 단순 shading뿐 아니라 "이 coarse level을 사용해도 되는가"를 판단하는 error metadata가 된다.

#### SDF / level-set `phi`
SDF를 단순 평균하면 얇은 surface의 zero crossing이 사라질 수 있다.

Surface 존재 여부를 보존하려면 coarse cell이 적어도 다음 정보를 유지하는 방식이 더 안전하다.

`phi_min <= 0 <= phi_max`

이 조건이면 coarse cell 내부에 zero crossing이 존재할 가능성을 보수적으로 유지할 수 있다.

즉 coarse SDF는 시각적 smoothing representation과 별개로 **conservative surface-existence envelope**를 가질 수 있다.

#### Material / categorical data
Material ID는 평균할 수 없다. Coarse representation은 dominant material, bit mask, occupancy mask, material set처럼 categorical semantics를 유지해야 한다.

따라서 multi-resolution fallback은 texture LOD의 일반화가 아니라 **data semantics-aware hierarchy**로 보는 편이 정확하다.

### 4.5 Error-bounded fallback의 핵심은 "coarse를 쓸 수 있는가"를 수치화하는 것이다
Memory pressure에서 단순히 "resident한 가장 fine한 LOD"를 쓰는 것만으로는 품질을 예측하기 어렵다.

더 강한 구조는 각 hierarchy level에 **error bound**를 저장하는 것이다.

예를 들어 scalar field에서 parent가 child를 근사할 때 최대 deviation을

`E_l = max |f_child - f_parent_reconstructed|`

처럼 생각할 수 있다.

렌더링 시에는 이 world/field-space error가 현재 pixel에 얼마나 크게 보이는지 projected error로 변환한다.

개념적으로

`E_screen ≈ E_world * ProjectionScale(distance, FOV, viewport)`

와 같은 관계를 생각할 수 있다.

그 결과 fallback 선택은 다음 목표를 가진다.

> **현재 resident한 level 중 `E_screen <= ErrorBudget`을 만족하는 가장 저렴한 representation을 선택한다.**

이렇게 하면 VRAM budget이 줄어도 quality 정책이 명확해진다.

- budget 여유: 낮은 error threshold
- memory pressure: threshold를 점진적으로 완화
- critical view: threshold 유지, 다른 region을 더 coarse하게

즉 quality와 memory가 하나의 scheduler에서 연결된다.

### 4.6 Volume rendering에서는 radiometric error까지 고려할 수 있다
Volume ray marching에서 field error가 곧 pixel error와 같지는 않다.

Density 오차가 최종 color에 미치는 영향은 다음에 따라 달라진다.

- step length
- transfer function
- opacity accumulation
- transmittance
- lighting/shadowing

예를 들어 이미 앞쪽에서 transmittance가 거의 0이라면 뒤쪽 brick의 fine detail은 pixel에 거의 영향을 주지 않는다. 반대로 얇고 높은 extinction feature는 작은 density error도 silhouette나 attenuation을 크게 바꿀 수 있다.

따라서 advanced residency policy에서는 단순 geometric distance보다 **estimated contribution / transmittance importance**를 error budget에 포함할 수 있다.

이는 sparse volume residency와 ray marching이 별도 subsystem이 아니라 서로 정보를 주고받을 수 있다는 뜻이다.

### 4.7 Mip tail과 custom coarse-brick pool은 역할은 비슷하지만 제약이 다르다
**Hardware/API mip tail**

- sparse mipmapped image layout의 일부다.
- 작은 mip level이 opaque memory region으로 묶인다.
- implementation이 보고한 first LOD/size/stride 규칙을 따라야 한다.
- 항상 resident하게 두면 확실한 texture-level fallback floor가 된다.

**Custom coarse-brick pool**

- logical hierarchy를 application이 직접 정의한다.
- SDF min/max, material mask, simulation metadata처럼 texture mip보다 richer한 representation을 담을 수 있다.
- bind granularity와 별개로 application-defined LOD를 만들 수 있다.
- 대신 page table, lifetime, compaction, synchronization contract를 직접 관리해야 한다.

즉 regular filtered scalar image에는 mip tail이 자연스럽고, semantic-rich sparse simulation field에는 custom coarse hierarchy가 더 유연할 수 있다.

### 4.8 Coarse/Fine 경계는 seam과 temporal popping의 원인이 된다
같은 frame에서 인접한 두 brick이 서로 다른 LOD를 사용하면 boundary artifact가 생길 수 있다.

3D texture filtering에서는 특히 다음 문제가 중요하다.

- trilinear footprint가 neighbor brick을 넘는다.
- 한쪽은 fine resident, 다른 쪽은 coarse fallback이다.
- gradient/normal reconstruction이 서로 다른 scale에서 계산된다.
- iso-surface 위치가 LOD 간 약간 달라진다.

따라서 fallback architecture는 단순히 "parent를 찾았다"에서 끝나지 않고 **cross-LOD continuity**를 고려해야 한다.

대표적인 설계 요소는 다음과 같은 형태다.

- brick border/apron data
- cross-level filter compatibility
- parent-child overlap region
- temporal LOD hysteresis
- child가 새로 resident됐을 때 짧은 transition window

특히 temporal prediction으로 fine brick이 뒤늦게 도착하면 화면은 `coarse -> fine`으로 전환된다. 이 전환이 매 frame 반복되면 residency thrash는 줄었어도 **LOD shimmer**가 남을 수 있다.

어제 다룬 hysteresis는 memory state뿐 아니라 quality transition에도 적용될 수 있다.

### 4.9 Residency metadata는 frame 중간에 바뀌지 않는 snapshot으로 보는 것이 안전하다
Sparse bind/map은 비동기 queue operation일 수 있고, upload/decompression 완료 시점도 서로 다르다.

그러므로 renderer가 한 dispatch 안에서 보는 page table이 중간에 바뀌면 다음 문제가 생길 수 있다.

- 같은 ray가 앞 sample과 뒤 sample에서 서로 다른 residency view를 본다.
- parent/child transition이 비결정적으로 섞인다.
- physical slot reuse와 stale mapping이 충돌한다.

C++/render graph 관점에서는 다음과 같은 역할 분리가 자연스럽다.

- `ResidencyManager`: 다음 epoch의 mapping을 계산
- `ResidencyTable[N]`: 현재 frame consumer가 읽는 immutable snapshot
- `ResidencyTable[N+1]`: feedback/prediction 결과로 갱신 중인 next snapshot
- sparse bind / upload completion signal 이후 swap

이렇게 보면 page table은 단순 resource lookup buffer가 아니라 **frame-level ABI**다.

Shader, CUDA compute, Vulkan rendering이 같은 sparse field를 공유한다면 모두 동일한 residency epoch 의미를 이해해야 한다.

### 4.10 Memory pressure controller는 bytes가 아니라 quality curve를 봐야 한다
Residency manager가 `residentBytes <= budget`만 만족하면 기술적으로는 성공할 수 있다. 하지만 사용자가 보는 품질은 전혀 다를 수 있다.

더 좋은 관측 지표는 다음을 함께 본다.

- fine-residency hit rate
- fallback rate by LOD
- mip-tail/root fallback rate
- average / P95 projected error
- error-budget violation rate
- coarse->fine transition count
- residency churn bytes
- bind/map operation count
- visible hole / invalid sample count
- frame time과 VRAM usage

특히 portfolio나 실제 엔진 분석에서는 **VRAM budget을 줄였을 때 visual error와 frame time이 어떻게 변하는지**를 curve로 보는 것이 좋다.

`VRAM Budget -> Fallback Distribution -> Error -> Frame Time`

이 그래프는 sparse residency architecture가 단순히 메모리를 줄였는지, 아니면 **품질을 예측 가능하게 낮추면서 성능을 유지했는지**를 보여준다.

## 5. 내 관심 분야와 연결

### 5.1 SDF / Level-Set 기반 3D emulation
Level-set `phi = 0` surface를 sparse brick으로 저장하는 파이프라인에서는 fine brick miss가 곧 geometry hole로 이어질 수 있다.

Residency-aware fallback을 사용하면 coarse level에서도 surface 존재 가능성을 유지할 수 있다. 특히 parent brick이 `phi_min / phi_max` 또는 distance-error bound를 함께 저장하면 **fine field가 없을 때도 zero-crossing을 보수적으로 추적하는 구조**를 만들 수 있다.

이는 marching cubes, ray-marched SDF, cross-section visualization 모두와 연결된다.

### 5.2 NanoVDB와 sparse residency의 역할 분리
NanoVDB 같은 sparse data structure는 **어디에 값이 존재하는가**를 압축하는 구조다. Sparse image/array residency는 **어떤 physical memory page가 현재 GPU에 backing되어 있는가**를 제어한다.

Multi-resolution fallback은 이 두 계층 사이의 bridge가 된다.

- NanoVDB hierarchy 또는 custom tree: logical multi-resolution / topology
- sparse tile pool: physical residency
- fallback metadata: 현재 resident한 ancestor와 error bound

즉 sparse data structure와 sparse residency를 동시에 사용하는 경우에도 둘을 하나로 뭉개지 않고 역할을 분리할 수 있다.

### 5.3 CUDA simulation -> Vulkan rendering interop
CUDA가 fine field를 생산하고 Vulkan이 같은 volume을 visualization하는 구조에서는 producer와 consumer의 요구가 다르다.

CUDA simulation은 active narrow band에서 fine brick을 반드시 필요로 할 수 있고, Vulkan renderer는 화면에서 멀리 있는 region을 coarse fallback으로 볼 수 있다.

따라서 공통 residency manager는 `simulation-critical`, `render-fine`, `render-coarse` 같은 서로 다른 importance class를 합쳐 budget을 결정하는 구조로 확장할 수 있다.

이때 coarse fallback이 있기 때문에 rendering이 simulation의 fine working set을 불필요하게 밀어내지 않도록 priority를 설계하기 쉬워진다.

### 5.4 GPU-driven rendering / ray tracing
Ray tracing에서는 secondary ray가 camera frustum 밖의 region을 갑자기 요구할 수 있으므로 prediction이 raster view보다 어렵다.

항상 resident한 coarse hierarchy는 이런 **unpredictable traversal**에 특히 강하다. Fine brick을 기다리지 않고 coarse representation으로 ray를 계속 진행할 수 있기 때문이다.

향후 ray cone, path throughput, transmittance와 error bound를 결합하면 "이 ray에는 어느 LOD까지 필요한가"를 더 정교하게 판단할 수 있다.

### 5.5 Graphics engineer 관점
이 주제는 단순 sparse texture API 지식보다 다음 역량을 동시에 보여준다.

- GPU virtual memory / residency 이해
- multi-resolution representation 설계
- shader-side resource indirection
- C++ resource lifetime과 frame epoch
- numerical / visual error model
- performance-quality trade-off 분석
- simulation과 rendering의 consumer contract 분리

즉 graphics engineer에게 중요한 **"API를 아는 것"에서 "quality와 memory를 함께 설계하는 것"으로 넘어가는 주제**다.

## 6. 머릿속에 남길 질문 3개

1. **Sparse page miss를 0 또는 empty로 처리하는 것과 coarse resident ancestor로 fallback하는 것은 최종 image의 failure mode를 어떻게 다르게 만드는가?**
2. **SDF/level-set volume에서 단순 average mipmap이 zero-crossing을 잃을 수 있다면, coarse representation은 어떤 conservative metadata를 추가로 가져야 하는가?**
3. **VRAM pressure가 커졌을 때 LOD를 낮추는 정책을 distance가 아니라 error budget으로 제어하면 어떤 profiling 지표가 새로 필요해지는가?**

## 7. graphics engineer 면접 질문 1개와 답변

**질문**  
대형 sparse 3D volume을 렌더링하는데 fine brick이 frame 안에 제때 resident하지 못해 hole과 flicker가 발생한다고 하자. VRAM을 크게 늘리지 않고 이 문제를 구조적으로 완화하는 renderer architecture를 설명해 보세요.

**답변**  
핵심은 page miss를 exceptional failure로 두지 않고 **multi-resolution fallback을 정상적인 rendering path로 만드는 것**이다.

먼저 logical volume에 fine-to-coarse hierarchy를 두고, 최소 몇 단계의 coarse representation 또는 mip tail/root를 항상 resident하게 유지한다. Shader가 원하는 `desired LOD`를 결정한 뒤 해당 level이 없으면 page table에서 현재 사용할 수 있는 **best resident ancestor**로 fallback한다.

하지만 무조건 coarse parent를 쓰는 것만으로는 충분하지 않다. 각 level에는 parent가 child를 얼마나 정확히 근사하는지 나타내는 error metadata를 두고, projected screen error나 volume contribution 기준의 budget 안에서 fallback이 허용되는지 판단한다. SDF라면 단순 평균 대신 min/max `phi` 같은 conservative bound를 유지해 zero crossing이 사라지지 않도록 한다.

또한 residency table은 frame 동안 immutable snapshot으로 읽고, next-frame mapping은 별도의 epoch에서 갱신해 sparse bind, upload, physical-slot reuse와의 race를 피한다. Fine brick이 새로 도착할 때는 LOD hysteresis 또는 transition state를 사용해 coarse/fine 전환의 temporal popping을 줄인다.

마지막으로 성능 평가는 resident byte만 보지 않고 fine hit rate, fallback LOD distribution, error-budget violation, map/unmap churn, visible artifact rate와 frame time을 함께 측정한다. 이렇게 하면 memory pressure가 **undefined data나 hole이 아니라 측정 가능한 quality degradation**으로 나타난다.

## 8. 포트폴리오 / 커리어 연결
이 개념을 포트폴리오에 녹일 때 강한 포인트는 "sparse volume을 만들었다"보다 **memory pressure에서도 품질이 예측 가능하게 유지되는 architecture를 설명할 수 있다는 것**이다.

설명 구조는 다음처럼 잡을 수 있다.

**Problem**  
Fine-grained sparse residency는 VRAM을 절약하지만, page miss가 visible hole과 temporal flicker를 만든다.

**Architecture**  
`Logical Brick Hierarchy -> Desired LOD -> Residency Lookup -> Best Resident Ancestor -> Error Check -> Sampling`

**Memory Model**  
- fine tile pool
- always-resident coarse/root pool 또는 mip tail
- compact residency/error metadata
- frame-stable residency epoch

**Quality Contract**  
- screen-space error budget
- SDF zero-crossing conservative bound
- temporal LOD hysteresis

**Performance Evidence**  
- VRAM budget별 fallback distribution
- error-budget violation rate
- page miss가 hole로 이어지는 비율
- mapping churn
- frame time

이런 설명은 Vulkan/CUDA API 지식뿐 아니라 **GPU memory architecture, numerical representation, rendering quality, C++ lifetime design을 하나의 시스템 문제로 연결하는 능력**을 보여준다.

면접에서도 sparse residency를 단순히 `vkQueueBindSparse()` 또는 CUDA sparse array API 이름으로 설명하는 것보다, **"non-resident fine data가 있을 때 renderer가 어떤 semantic을 유지해야 하는가"**까지 이야기할 수 있으면 훨씬 강한 답변이 된다.

## 9. 내일 이어서 볼 개념
**Conservative Multi-Resolution Volume Bounds: Min/Max Envelopes, Zero-Crossing Preservation, and Residency-Aware Empty-Space Skipping**

오늘은 coarse fallback을 사용할 수 있으려면 각 level의 error를 알아야 한다는 점을 봤다. 내일은 그 error metadata 자체를 더 깊게 들어간다.

특히 다음 질문으로 이어진다.

> **SDF, density, occupancy volume의 coarse brick이 fine data를 대체할 때, 어떤 bound를 저장해야 surface를 놓치지 않으면서도 ray marching과 empty-space skipping을 더 빠르게 만들 수 있는가?**

이를 통해 `min/max envelope`, `zero-crossing preservation`, `opacity bound`, hierarchical empty-space skipping, conservative traversal을 연결한다.

## 10. 참고 키워드
- Sparse Residency
- Multi-Resolution Fallback
- Mip Tail
- `VkSparseImageMemoryRequirements`
- `imageMipTailFirstLod`
- `VK_SPARSE_IMAGE_FORMAT_SINGLE_MIPTAIL_BIT`
- `residencyNonResidentStrict`
- CUDA Sparse Mipmapped Array
- `miptailFirstLevel`
- `miptailSize`
- `cuMemMapArrayAsync`
- Coarse Brick Hierarchy
- Best Resident Ancestor
- Desired LOD vs Resident LOD
- Error-Bounded Rendering
- Screen-Space Error
- Ray Cone / Ray Differential
- Sparse Volume Rendering
- SDF / Level Set
- Zero-Crossing Preservation
- `phi_min / phi_max`
- Conservative Bounds
- Opacity / Transmittance Bound
- Residency Page Table
- Frame Epoch
- Double-Buffered Residency Metadata
- Temporal LOD Hysteresis
- Brick Border / Apron
- VRAM Working Set
- Graceful Degradation
- C++ Resource Lifetime
- Vulkan Sparse Image
- CUDA Sparse Array
