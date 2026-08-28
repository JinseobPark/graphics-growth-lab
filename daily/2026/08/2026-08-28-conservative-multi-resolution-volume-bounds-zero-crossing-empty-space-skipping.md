---
title: "Conservative Multi-Resolution Volume Bounds: Min/Max Envelopes, Zero-Crossing Preservation, and Residency-Aware Empty-Space Skipping"
date: "2026-08-28"
category: Graphics
tags: [GPU, Volume Rendering, Sparse Volume, SDF, Level Set, MinMax Hierarchy, Empty Space Skipping, Zero Crossing, Ray Marching, Vulkan, CUDA, Compute Shader, Memory Layout, C++]
level: intermediate
---

# [Daily Graphics Growth] 2026-08-28 - Conservative Multi-Resolution Volume Bounds: Min/Max Envelopes, Zero-Crossing Preservation, and Residency-Aware Empty-Space Skipping

## 1. 오늘의 개념
어제의 **Residency-Aware Multi-Resolution Fallback**은 fine brick이 없을 때 coarse parent나 mip tail을 사용해 품질을 단계적으로 낮추는 방법이었다. 오늘은 그 coarse representation이 단순 저해상도 값이 아니라 fine field를 놓치지 않는 **conservative bound**가 되어야 한다는 점을 본다.

기본 표현은 node마다 값의 interval을 유지하는 것이다.

`I(N) = [v_min(N), v_max(N)]`

부모는 child 범위를 합쳐 `v_min(parent)=min(v_min(child_i))`, `v_max(parent)=max(v_max(child_i))`로 만든다. SDF/level-set `phi`에서 `phi_min <= 0 <= phi_max`라면 그 영역은 zero crossing을 배제할 수 없다. 반대로 interval 전체가 양수 또는 음수라면, bound가 실제 field를 감싸고 field가 연속이라는 조건 아래 zero crossing이 없다고 판정할 수 있다.

Sparse residency와 연결하면 fine payload가 non-resident여도 coarse metadata만으로 **skip / descend / fallback / request**를 결정할 수 있다. 즉 hierarchy는 LOD가 아니라 traversal correctness를 위한 acceleration contract가 된다.

## 2. 한 줄 핵심
> Multi-resolution volume hierarchy는 평균 mip가 아니라 **fine field를 보수적으로 감싸는 min/max·opacity·distance bound를 유지해, sparse residency에서도 surface false-negative 없이 empty space를 건너뛰게 하는 구조**여야 한다.

## 3. 왜 중요한가
평균값과 존재성 보존은 목적이 다르다. 일반 mipmap의 평균값은 시각적 저주파 근사에는 좋지만, 얇은 positive/negative feature를 희석해 SDF의 zero crossing을 놓칠 수 있다. 그래서 **visual LOD**와 **conservative metadata**를 분리해야 한다.

Empty-space skipping에서 빈 영역을 비어 있지 않다고 판단하면 성능만 손해 보지만, surface가 있는 영역을 empty라고 판단하면 geometry가 사라진다. Conservative structure는 일부 불필요한 traversal을 허용하더라도 surface false-negative를 막는 쪽이 우선이다.

Density, temperature, doping 같은 scalar field에서는 brick의 `[v_min, v_max]`와 transfer function을 결합한다. 이 범위 전체에서 alpha가 0이면 현재 시각화 조건에서 brick을 skip할 수 있다. 같은 data라도 transfer function이 바뀌면 visual occupancy가 달라진다는 점이 중요하다.

Sparse residency에서는 작은 metadata가 큰 역할을 한다. Fine brick이 VRAM에 없어도 coarse bound가 resident하면 “확실히 empty인가?”, “surface 가능성이 있는가?”, “fine request가 필요한가?”를 판단할 수 있다.

또한 level-set은 항상 perfect SDF가 아니다. Advection이나 boolean operation 뒤에는 `|grad phi| ~= 1`이 깨질 수 있으므로 `abs(phi)`를 그대로 안전한 ray step으로 해석하면 위험하다. **Zero-crossing range bound**와 **distance lower bound**는 별개의 문제다.

## 4. 구현 관점
### 4.1 Min/Max hierarchy의 invariant
Parent interval은 자신이 덮는 모든 fine sample과 실제 sampling domain의 값을 포함해야 한다. 이 invariant가 깨지면 hierarchy는 빠르더라도 안전하지 않다.

Trilinear interpolation은 voxel corner 값의 convex combination이므로, interpolation에 참여하는 sample domain을 정확히 포함한 min/max는 voxel 내부 값을 감쌀 수 있다. Brick border, halo, ghost voxel을 사용한다면 hierarchy build 범위와 shader sampling 범위도 일치해야 한다.

### 4.2 Zero-crossing test와 distance skipping은 다르다
`phi_min <= 0 <= phi_max`이면 surface possibility가 있으므로 descend 또는 detailed sampling이 필요하다. Interval이 한쪽 sign이면 zero crossing을 배제할 수 있다.

하지만 이것만으로 큰 ray step이 안전한 것은 아니다. Exact SDF는 1-Lipschitz 특성을 가지며 일반적으로 `|phi(x)-phi(y)| <= L||x-y||` 형태의 bound를 생각할 수 있다. `L`이 커질수록 같은 `phi` 값에서도 안전한 step은 작아진다. 따라서 hierarchy에는 필요에 따라 min/max 외에 gradient/Lipschitz estimate와 approximation error를 함께 둘 수 있다.

### 4.3 Reinitialization 상태도 renderer contract다
Level-set reinitialization 직후에는 `phi`가 distance-like field에 가까워져 distance-based skipping을 더 신뢰할 수 있다. 반대로 advection 직후에는 zero-crossing hierarchy는 유효해도 `abs(phi)` 기반 step은 보수적으로 제한해야 할 수 있다.

C++ 쪽에서도 단순 texture handle보다 개념적으로 `PhiFieldView { data, hierarchy, distanceQuality, errorBound, epoch }` 같은 semantic contract가 더 안전하다.

### 4.4 FP16 bound packing에는 outward rounding이 필요하다
Bound metadata는 작고 자주 읽히므로 FP16이 매력적이다. 하지만 round-to-nearest로 stored minimum이 실제 최소보다 커지거나 stored maximum이 실제 최대보다 작아지면 conservative invariant가 깨질 수 있다.

따라서 압축 후에도 `storedMin <= trueMin`, `storedMax >= trueMax`가 유지되도록 **outward rounding** 또는 safety margin이 필요하다. 일반 value compression과 conservative metadata compression은 목적이 다르다.

### 4.5 Payload와 metadata lifetime을 분리한다
Fine payload는 sparse map/unmap 대상이지만 hierarchy metadata는 더 안정적으로 resident하는 편이 좋다. Node metadata는 `rangeMin`, `rangeMax`, `errorBound`, `residentLod`, `payloadHandle`, `residencyEpoch`, `dirtyEpoch`, `childMask` 등을 가질 수 있다.

`payloadHandle`이 invalid여도 range metadata는 유효할 수 있다. Traversal은 먼저 metadata로 skip/descend/fallback을 결정하고, 실제 fine sample이 필요할 때 payload residency를 확인한다.

### 4.6 Bounds에도 epoch가 필요하다
Simulation이 field를 갱신했는데 min/max hierarchy가 한 frame 늦으면 새 payload와 오래된 bound가 섞일 수 있다. 이 조합은 잘못된 skip으로 이어질 수 있으므로 `VolumeVersion N = { payload, bounds, residency }`처럼 같은 logical epoch로 publish하는 편이 안전하다.

### 4.7 Bottom-up reduction은 bandwidth problem이다
Leaf brick의 min/max를 계산하고 parent를 반복 reduction하면 계산량보다 memory traffic이 지배적일 수 있다. GPU에서는 contiguous child layout, coalesced load, compact parent write, dirty-region update, shared-memory reduction과 occupancy trade-off가 중요하다.

### 4.8 Pointerless layout과 traversal coherence
Morton order, breadth-first level array, linear octree 같은 pointerless hierarchy는 node footprint와 random load를 줄이는 데 유리하다. 반면 hierarchy traversal 자체는 ray마다 다른 depth를 만들어 warp divergence를 유발한다.

따라서 profiling은 ray-step 수뿐 아니라 skipped distance ratio, hierarchy node fetch count, L1/L2 hit rate, active lane ratio, payload fetch reduction, metadata bandwidth를 함께 봐야 한다.

## 5. 내 관심 분야와 연결
SDF/level-set 기반 semiconductor emulation에서 `phi=0`이 material surface라면 얇은 layer, trench, sidewall은 coarse average에서 쉽게 사라질 수 있다. `phi_min/phi_max` envelope를 따로 유지하면 rendering LOD가 낮아져도 surface candidate region을 보존할 수 있다.

Marching Cubes도 corner sign change를 이용하므로 같은 min/max hierarchy를 mesh extraction 전의 broad culling structure로 재사용할 수 있다.

Doping/temperature 같은 scalar heatmap은 min/max와 transfer function을 결합해 현재 시각화에서 보이지 않는 brick을 skip할 수 있다. **Data Occupancy**와 **Visual Occupancy**를 분리하는 관점이 중요하다.

NanoVDB 같은 sparse structure는 “어디에 데이터가 존재하는가”를 compact하게 저장한다. 오늘의 hierarchy는 “그 데이터 범위에서 surface/opacity가 존재할 가능성이 있는가”를 판단하는 traversal acceleration layer다.

CUDA가 field와 bounds를 만들고 Vulkan이 렌더링한다면 `data ready`만 signal하는 것으로 부족하다. Bound hierarchy까지 같은 epoch로 consumer-visible해야 한다. Cross-API synchronization을 **semantic snapshot visibility** 관점으로 보는 것이 중요하다.

## 6. 머릿속에 남길 질문 3개
1. **`phi_min <= 0 <= phi_max`가 의미하는 것은 surface가 반드시 있다는 것인가, 아니면 surface를 배제할 수 없다는 것인가?**
2. **Fine payload는 non-resident지만 coarse min/max hierarchy는 resident할 때 renderer가 안전하게 결정할 수 있는 것과 결정하면 안 되는 것은 무엇인가?**
3. **FP16 min/max 압축에서 round-to-nearest가 conservative invariant를 깨뜨릴 수 있는 이유는 무엇인가?**

## 7. graphics engineer 면접 질문 1개와 답변
### 질문
**Sparse SDF volume에서 empty-space skipping을 할 때 왜 평균 mip 대신 min/max hierarchy가 필요하며, min/max만 있으면 sphere tracing처럼 큰 step을 안전하게 사용할 수 있나요?**

### 답변
평균 mip은 시각적 근사에는 좋지만 얇은 positive/negative feature를 희석해 zero crossing을 놓칠 수 있다. 반면 `[phi_min, phi_max]`가 fine field를 보수적으로 감싸면 `phi_min <= 0 <= phi_max`인 영역은 surface possibility를 유지할 수 있고, interval이 한쪽 sign이면 zero crossing을 배제할 수 있다.

하지만 min/max는 **range bound**이지 자동으로 **distance lower bound**가 되는 것은 아니다. 큰 ray step을 안전하게 쓰려면 field가 true SDF에 가까운지, Lipschitz/gradient bound가 어떤지, node extent와 approximation error가 얼마인지가 추가로 필요하다. 따라서 zero-crossing bound와 distance-step bound를 분리하는 설계가 좋다.

## 8. 포트폴리오 / 커리어 연결
이 주제를 포트폴리오에서는 “octree로 empty space를 skip했다”보다 다음 구조로 설명하는 편이 강하다.

- visual LOD와 conservative hierarchy 분리
- min/max envelope로 zero-crossing possibility 보존
- transfer-function-aware opacity classification
- always-resident metadata + sparse fine payload
- payload/bound/residency epoch를 같은 frame snapshot으로 관리
- FP16 packing에서도 conservative quantization 유지
- ray steps, skipped distance, L2 hit rate, active-lane ratio, metadata bytes를 함께 profiling

이렇게 설명하면 numerical correctness, GPU memory, compute reduction, sparse residency, rendering traversal을 하나의 시스템 문제로 연결하는 graphics engineer 관점을 보여줄 수 있다.

## 9. 내일 이어서 볼 개념
**Lipschitz-Bounded SDF Ray Marching: Conservative Step Sizes, Gradient Bounds, and Hierarchical Distance Fields**

오늘은 “surface가 있을 가능성을 놓치지 않는” range hierarchy를 봤다. 다음에는 “ray가 실제로 얼마나 멀리 이동해도 안전한가”를 다루며 true SDF의 1-Lipschitz 성질, approximate level-set의 gradient bound, conservative distance lower bound, sphere-tracing overshoot와 reinitialization quality를 연결한다.

## 10. 참고 키워드
- Conservative Bounds
- Min/Max Hierarchy
- Interval Arithmetic
- Zero-Crossing Preservation
- Signed Distance Field (SDF)
- Level Set
- Lipschitz Bound
- Gradient Bound
- Sphere Tracing
- Empty-Space Skipping
- Transfer-Function-Aware Skipping
- Opacity Bound
- Volume Ray Casting
- Brick Hierarchy
- Linear Octree
- Morton Order
- Sparse Residency
- Residency Epoch
- Conservative Quantization
- Outward Rounding
- Trilinear Interpolation Bound
- GPU Reduction
- John C. Hart, *Sphere Tracing* (1996)
- SparseLeap
- Residency Octree
