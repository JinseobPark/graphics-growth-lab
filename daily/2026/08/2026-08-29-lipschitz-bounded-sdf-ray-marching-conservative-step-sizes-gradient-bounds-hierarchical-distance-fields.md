---
title: "Lipschitz-Bounded SDF Ray Marching: Conservative Step Sizes, Gradient Bounds, and Hierarchical Distance Fields"
date: "2026-08-29"
category: Graphics
tags: [GPU, Rendering, SDF, Level Set, Sphere Tracing, Lipschitz Bound, Gradient Bound, Ray Marching, Hierarchical Distance Field, Empty Space Skipping, CUDA, Vulkan, Compute Shader, Memory Layout, C++]
level: intermediate
---

# [Daily Graphics Growth] 2026-08-29 - Lipschitz-Bounded SDF Ray Marching: Conservative Step Sizes, Gradient Bounds, and Hierarchical Distance Fields

## 1. 오늘의 개념
어제는 **min/max hierarchy**가 coarse level에서도 zero crossing 가능성을 보수적으로 유지해 surface false-negative를 막는 구조를 봤다. 오늘은 그 다음 질문인 **“현재 ray가 surface를 건너뛰지 않고 얼마나 멀리 이동해도 되는가?”**를 다룬다.

정확한 Signed Distance Field(SDF) `d(x)`는 표면까지의 실제 signed distance를 나타내므로 일반적인 위치에서 `|∇d| = 1`이고 1-Lipschitz 성질을 가진다. 따라서 ray marching에서 현재 점 `x`의 `|d(x)|`만큼 이동해도 그 구간 안에 surface가 없다는 것을 보장할 수 있다. 이것이 **Sphere Tracing**의 핵심 직관이다.

하지만 simulation에서 얻은 level-set `phi`, trilinear-resampled grid, CSG 결과, advection 이후의 field는 대개 exact SDF가 아니다. 이때 `abs(phi)`를 그대로 step size로 사용하면 field가 실제 surface distance를 과대평가하는 영역에서 zero crossing을 건너뛸 수 있다.

보다 일반적으로 scalar field `f`가 어떤 영역에서 Lipschitz constant `L`을 만족한다면

`|f(x) - f(y)| <= L ||x - y||`

이고 surface 위의 점 `y`에서는 `f(y)=0`이므로

`dist(x, surface) >= |f(x)| / L`

이라는 conservative lower bound를 얻는다. 즉 exact SDF가 아니더라도 **유효한 upper bound `L`을 알고 있다면 `|f|/L`을 안전한 step certificate로 사용할 수 있다.**

오늘의 핵심은 이 `L`을 단일 global constant로 두는 대신 brick/node 단위의 **Local Lipschitz Bound** 또는 **Gradient Bound**로 관리하고, 어제의 min/max hierarchy와 결합해 **range culling + distance stepping**을 동시에 수행하는 것이다.

## 2. 한 줄 핵심
> Sphere tracing의 안전성은 “SDF라는 이름”이 아니라 **현재 field가 surface distance를 얼마나 보수적으로 bound하는가**에 달려 있으며, approximate level-set에서는 `step <= |phi| / L` 형태의 Lipschitz-aware step과 hierarchy-level bound를 함께 써야 overshoot 없이 큰 구간을 건너뛸 수 있다.

## 3. 왜 중요한가
Real-time implicit rendering에서는 ray 한 개가 field를 몇 번 평가하는지가 성능을 거의 직접 결정한다. Step이 지나치게 작으면 정확하지만 느리고, 지나치게 크면 빠르지만 surface를 통과하는 **overshoot**가 발생한다. SDF ray marching은 이 두 문제 사이에서 “안전한 최대 step”을 계산한다는 점이 중요하다.

문제는 production field가 textbook SDF와 다르다는 것이다. Level-set advection, smoothing, boolean operation, resampling, low-precision storage, sparse brick border 처리 등은 모두 `|∇phi| ~= 1` 조건을 깨뜨릴 수 있다. 특히 thin film이나 trench처럼 feature가 얇은 경우 한 번의 overshoot가 곧 geometry disappearance로 이어진다.

Global Lipschitz bound 하나만 사용하면 correctness는 얻을 수 있지만 가장 가파른 영역 하나 때문에 전체 volume의 step이 작아질 수 있다. 반면 **local bound**는 공간적으로 smooth한 영역에서 큰 step을 허용하고, gradient가 큰 영역만 보수적으로 걷게 해 성능과 안전성을 동시에 개선할 수 있다. Segment Tracing 계열 연구도 local Lipschitz bound를 이용해 classical sphere tracing보다 field evaluation 수를 줄이는 방향을 보여준다.

또 하나 중요한 점은 **`|∇phi| = 1`과 rendering normal은 같은 목적이 아니라는 것**이다. Shading에서는 `normalize(∇phi)`가 필요하지만, distance quality를 판단할 때 gradient를 먼저 normalize하면 field가 얼마나 SDF에서 벗어났는지 정보를 잃는다. Renderer는 normalized normal과 raw gradient magnitude를 구분해야 한다.

## 4. 구현 관점
### 4.1 Safe step의 수학적 contract
현재 sample point를 `x`, zero set을 `S={p | phi(p)=0}`, 해당 영역의 Lipschitz upper bound를 `L >= 0`이라 하자.

`|phi(x)| = |phi(x)-phi(y)| <= L ||x-y||` for `y in S`

이므로

`distance(x,S) >= |phi(x)| / L`

이다. 따라서 `L`이 실제 Lipschitz constant보다 작지 않다면

`delta_t_safe <= |phi(x)| / L`

은 surface를 건너뛰지 않는 conservative step이 된다.

여기서 중요한 단어는 **upper bound**다. 실제 `L`보다 작은 값을 저장하면 step이 너무 커져 correctness가 깨진다. 반대로 `L`을 크게 잡으면 안전하지만 step이 작아져 성능만 손해 본다.

Exact SDF는 1-Lipschitz이며 미분 가능한 대부분의 점에서 `|∇phi|=1`이다. 다만 medial axis처럼 distance function이 미분 불가능한 위치가 있으므로 “모든 점에서 gradient norm이 정확히 1”이라고 단순화하면 안 된다.

### 4.2 Local bound의 유효 영역을 넘어가면 안 된다
Brick A에 대해 계산한 `L_A`는 Brick A 내부에서만 유효할 수 있다. 그런데 `|phi|/L_A`가 brick 경계를 넘어가는 길이라면 그 이후 영역의 gradient가 더 클 가능성이 있다.

따라서 local bound 기반 traversal은 개념적으로 다음 두 조건을 동시에 만족해야 한다.

- distance certificate: `step <= |phi| / L_local`
- domain certificate: 그 `L_local`이 step 전체 구간에 대해 유효해야 함

실무적으로는 local step을 현재 node/brick의 exit distance로 제한하거나, 더 큰 parent node가 보장하는 `L_parent`로 승격해 step을 계산하는 식의 **hierarchical validity**가 필요하다.

즉 local Lipschitz bound는 단순 scalar metadata가 아니라 **“이 bound가 어느 spatial domain을 커버하는가”**까지 포함하는 traversal contract다.

### 4.3 Hierarchical Distance Field
어제의 node가 `[phi_min, phi_max]`를 가졌다면 오늘은 다음 metadata를 함께 생각할 수 있다.

`NodeMeta = { phiMin, phiMax, Lmax, approximationError, childMask, payloadHandle, epoch }`

- `phiMin / phiMax`: zero crossing을 배제할 수 있는지 판단
- `Lmax`: node domain에서의 gradient/Lipschitz upper bound
- `approximationError`: quantization, interpolation, coarse reconstruction 오차의 상한
- `payloadHandle`: fine field가 resident할 때의 실제 data 위치
- `epoch`: field와 bound가 같은 snapshot인지 확인

Parent의 `Lmax`는 최소한 child들의 `Lmax`보다 작아서는 안 된다. Coarse hierarchy로 올라갈수록 bound가 느슨해질 수 있지만 더 큰 spatial domain에 대해 유효해진다는 장점이 있다.

이 구조에서는 ray가 먼저 min/max로 “완전히 empty인가?”를 확인하고, surface 가능성이 없으면 node extent를 크게 skip한다. Surface possibility가 남아 있으면 `Lmax`를 이용한 distance step 또는 child descend를 선택한다.

### 4.4 Gradient bound와 interpolation error를 분리한다
Grid field에서 `L`을 단순히 sampled `max(|∇phi|)`로 두면 충분하지 않을 수 있다. 실제 renderer는 trilinear interpolation된 continuous field를 평가하므로 bound는 **sample point**가 아니라 interpolation domain 전체를 감싸야 한다.

또 FP16 quantization, brick compression, coarse representation을 사용하면 저장된 `phi_hat`과 실제 target field `phi` 사이에

`|phi_hat(x) - phi(x)| <= epsilon`

같은 approximation error가 생길 수 있다. 이때 step을 `|phi_hat|/L`로만 계산하면 surface 근처에서 오차가 distance를 과대평가할 수 있다. 개념적으로는

`distance_lower_bound ~= max(|phi_hat| - epsilon, 0) / L`

처럼 **value error를 먼저 제거한 conservative distance**가 더 안전하다.

즉 production renderer에서는 `gradient bound`와 `value approximation bound`가 서로 다른 metadata라는 점이 중요하다.

### 4.5 Conservative quantization
`Lmax`를 FP16으로 저장한다면 일반 round-to-nearest가 실제 bound보다 작은 값으로 내려갈 수 있다. `L`은 upper bound여야 하므로 저장 시 **round-up / safety inflation** 방향이 필요하다.

반대로 shader에서 `invL = 1/L`을 직접 저장한다면 safe multiplier는 실제 값보다 커지면 안 되므로 **round-down**이 필요하다.

- store `L`: 보수적으로 크게
- store `1/L`: 보수적으로 작게

이 차이는 작은 bit-width metadata를 설계할 때 매우 중요하다. 일반 rendering parameter compression과 correctness-critical bound compression은 같은 규칙을 쓰면 안 된다.

### 4.6 Reinitialization quality와 Eikonal residual
Level-set reinitialization(redistancing)은 zero level set을 유지하면서 field를 다시 distance-like하게 만드는 과정이며 이상적으로는 Eikonal equation

`|∇phi| = 1`

을 회복시키는 방향이다.

Renderer 관점에서는 `e = ||∇phi| - 1|`을 **Eikonal residual** 또는 distance-quality signal로 볼 수 있다. `e`가 작으면 `abs(phi)`에 가까운 aggressive step을 허용할 수 있고, `e`가 큰 영역은 hierarchy의 conservative `Lmax`에 더 의존해야 한다.

중요한 점은 reinitialization 완료 여부를 boolean flag 하나로 취급하기보다, field quality가 공간적으로 다를 수 있다는 것이다. Simulation update가 일부 brick만 수정했다면 dirty brick만 distance quality가 나빠질 수 있다.

### 4.7 GPU memory layout
Ray traversal은 node metadata를 매우 자주 읽으므로 metadata footprint가 곧 bandwidth다. Conceptually 다음처럼 hot/cold split을 생각할 수 있다.

**Hot traversal metadata**
- packed `phiMin/phiMax`
- packed `Lmax` 또는 `invL`
- child/residency mask
- compact payload index

**Cold validation/debug metadata**
- approximation error detail
- dirty/reinitialization epoch
- statistics
- source brick version

GPU에서는 SoA 또는 compact AoSoA가 coherent traversal에서 유리할 수 있다. 반대로 모든 node가 큰 C++ AoS struct를 그대로 미러링하면 한 step에서 쓰지 않는 field까지 cache line에 같이 들어와 metadata bandwidth가 커질 수 있다.

### 4.8 Divergence와 step distribution
Lipschitz-aware marching은 correctness를 높이지만 ray마다 `L_local`이 달라져 step count 분산이 커질 수 있다. 한 warp 안에서 일부 ray는 empty region을 크게 건너뛰고, 일부는 steep gradient 영역에서 작은 step을 반복하면 **SIMT divergence**가 발생한다.

따라서 성능 지표는 평균 step 수만 보면 부족하다.

- ray당 step count의 평균 / p95 / p99
- overshoot 또는 missed-intersection count
- local `L` distribution
- node descend count
- hierarchy metadata bytes per ray
- L1/L2 hit rate
- active lane ratio
- reinitialization 전후 traversal cost

이 지표를 같이 봐야 “더 conservative해서 느려진 것인지”, “hierarchy가 너무 세분화되어 metadata traffic이 늘어난 것인지”, “gradient가 실제로 망가져 step이 줄어든 것인지”를 구분할 수 있다.

### 4.9 C++ / compute / rendering pipeline contract
CUDA compute가 level-set을 업데이트하고 Vulkan rendering이 동일 field를 읽는 구조라면 field data와 `Lmax` hierarchy는 하나의 logical snapshot으로 publish되어야 한다.

예를 들어 compute가 `phi`만 새 frame으로 갱신했는데 renderer가 이전 frame의 `Lmax`를 사용하면, 오래된 bound가 새 field의 gradient를 과소평가할 수 있다. 이것은 단순 visual mismatch가 아니라 ray-marching correctness bug가 될 수 있다.

따라서 C++ resource contract는 개념적으로

`DistanceFieldView { phi, rangeHierarchy, lipschitzHierarchy, errorHierarchy, fieldEpoch, boundEpoch }`

처럼 field와 bound의 version 관계를 명시하는 편이 좋다. Cross-API semaphore는 GPU execution visibility를 보장하지만, **어떤 resource들이 같은 semantic version인가**는 application architecture가 보장해야 한다.

## 5. 내 관심 분야와 연결
Semiconductor process emulation의 level-set은 deposition, etch, boolean operation, advection을 반복하면서 perfect SDF에서 벗어나기 쉽다. 특히 thin oxide, sidewall, trench corner처럼 geometric scale이 작은 영역에서는 `abs(phi)`를 무조건 step으로 쓰는 것이 위험하다.

이때 `phi`와 함께 brick별 `Lmax`와 approximation error를 유지하면 renderer는 simulation field를 별도 완전 재거리화(redistancing)하지 않은 순간에도 conservative하게 traversal할 수 있다. Reinitialization 직후에는 step이 커지고, simulation update 직후에는 수정된 brick에서만 step이 줄어드는 **quality-aware traversal**이 가능해진다.

Marching Cubes와도 연결된다. Min/max hierarchy는 zero-crossing candidate를 줄이고, Lipschitz/error bound는 coarse node가 surface로부터 얼마나 떨어져 있는지 판단하는 추가 information이 될 수 있다. Ray marching과 mesh extraction이 서로 다른 consumer여도 같은 conservative hierarchy를 공유할 수 있다는 점이 중요하다.

Sparse NanoVDB/brick volume에서는 topology sparsity와 distance quality를 분리해 볼 수 있다. Active voxel이 있다는 사실은 surface가 가까운지, field가 distance-like한지 알려주지 않는다. Sparse topology 위에 `range + Lmax + error` metadata를 별도 acceleration layer로 두면 GPU rendering과 simulation validation을 동시에 지원할 수 있다.

## 6. 머릿속에 남길 질문 3개
1. **Exact SDF가 아닌 level-set에서도 `|phi|/L`이 conservative step이 될 수 있는 이유는 무엇이며, 여기서 `L`은 어떤 방향의 bound여야 하는가?**
2. **Brick-local `Lmax`가 안전하더라도 ray step이 brick 경계를 넘어가면 왜 그 보장이 깨질 수 있는가?**
3. **`Lmax`를 FP16으로 저장할 때 round-to-nearest가 위험한 이유와, `L` 자체를 저장할 때와 `1/L`을 저장할 때 conservative rounding 방향이 왜 반대인가?**

## 7. graphics engineer 면접 질문 1개와 답변
### 질문
**Simulation에서 생성된 level-set field를 sphere tracing으로 렌더링하려고 합니다. `|∇phi|`가 1이 아닐 수 있을 때 surface overshoot를 막으면서도 fixed small step보다 빠르게 marching하려면 어떤 구조를 설계하겠습니까?**

### 답변
`abs(phi)`를 그대로 distance로 가정하지 않고, spatial region마다 field의 Lipschitz/gradient upper bound `L`을 유지해 `abs(phi)/L`을 conservative distance lower bound로 사용한다. Bound는 실제 gradient를 과소평가하면 안 되며, local bound를 사용할 경우 그 bound가 유효한 brick/node 범위를 step이 넘어가지 않도록 domain validity도 함께 관리해야 한다.

여기에 min/max hierarchy를 결합해 zero crossing이 불가능한 node는 node extent 단위로 skip하고, surface 가능성이 있는 영역에서만 Lipschitz-aware marching으로 내려간다. Quantization이나 interpolation 오차가 있다면 `epsilon`을 따로 유지해 `max(abs(phi)-epsilon,0)/L`처럼 value error도 보수적으로 반영한다.

GPU 구현에서는 hot metadata를 compact하게 배치하고 `L`은 upward-conservative하게 quantize한다. Simulation이 field를 갱신할 때 range/Lipschitz/error hierarchy도 같은 epoch로 publish해 오래된 bound가 새 field를 과소평가하는 상황을 막는다. 성능은 평균 ray-step뿐 아니라 p95/p99 step count, hierarchy fetch, cache hit, active lane ratio와 missed-intersection 검증을 함께 본다.

## 8. 포트폴리오 / 커리어 연결
이 주제를 포트폴리오에서 강하게 보여주는 포인트는 “sphere tracing을 구현했다”가 아니라 **numerical field quality를 GPU traversal contract로 변환했다**는 것이다.

설명할 수 있는 구조는 다음과 같다.

- exact SDF와 arbitrary level-set의 차이를 명확히 구분
- `|phi|/L` 기반 conservative step derivation
- global bound 대신 hierarchical local Lipschitz bound 사용
- min/max zero-crossing hierarchy와 distance-step hierarchy 역할 분리
- quantization/interpolation error를 별도 `epsilon`으로 관리
- FP16 metadata에서도 conservative rounding 유지
- field/bound epoch를 함께 publish하는 C++ resource contract
- CUDA simulation → Vulkan rendering cross-API synchronization과 semantic snapshot 연결
- 평균 성능이 아니라 tail step count, divergence, cache traffic, correctness를 함께 profiling

이 정도로 설명하면 rendering shader 최적화뿐 아니라 numerical simulation, GPU memory layout, sparse data, synchronization, correctness를 하나의 시스템으로 설계하는 graphics/visualization engineer 역량을 보여줄 수 있다.

## 9. 내일 이어서 볼 개념
**Hybrid SDF Intersection Refinement: Bracketing, Secant/Bisection, and Sub-Voxel Surface Accuracy**

오늘은 surface를 건너뛰지 않고 **가까이 접근하는 방법**을 봤다. 다음에는 ray가 zero crossing 근처에 도착했을 때 단순 `abs(phi) < epsilon` 종료 대신 sign bracket을 만들고, secant/bisection 또는 safeguarded root refinement로 intersection을 sub-voxel 정확도로 확정하는 방법을 본다.

특히 sparse/trilinear level-set에서 step termination epsilon, normal stability, depth precision, TAA/reprojection jitter, mesh-vs-ray surface consistency를 하나의 intersection-quality 문제로 연결한다.

## 10. 참고 키워드
- Sphere Tracing
- Signed Distance Field (SDF)
- Level Set
- Lipschitz Continuity
- Lipschitz Constant
- Local Lipschitz Bound
- Gradient Bound
- Conservative Distance Bound
- Distance Lower Bound
- Eikonal Equation
- Eikonal Residual
- Redistancing / Reinitialization
- Overshoot
- Hierarchical Distance Field
- Min/Max Hierarchy
- Segment Tracing
- Value Error Bound
- Conservative Quantization
- Outward Rounding
- Trilinear Interpolation
- Sparse Volume
- Brick Hierarchy
- GPU Ray Marching
- SIMT Divergence
- CUDA / Vulkan Interop
- John C. Hart, *Sphere Tracing: A Geometric Method for the Antialiased Ray Tracing of Implicit Surfaces* (1996)
- Galin et al., *Segment Tracing Using Local Lipschitz Bounds* (Computer Graphics Forum, 2020)
- Barbier et al., *Lipschitz Pruning: Hierarchical Simplification of Primitive-Based SDFs* (Computer Graphics Forum, 2025)
