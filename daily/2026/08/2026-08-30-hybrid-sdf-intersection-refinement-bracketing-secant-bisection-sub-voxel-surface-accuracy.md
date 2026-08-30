---
title: "Hybrid SDF Intersection Refinement: Bracketing, Secant/Bisection, and Sub-Voxel Surface Accuracy"
date: "2026-08-30"
category: Graphics
tags: [GPU, Rendering, SDF, Level Set, Ray Marching, Sphere Tracing, Root Finding, Bracketing, Secant Method, Bisection, Brent Method, Trilinear Interpolation, Sub-Voxel Accuracy, CUDA, Vulkan, Compute Shader, Memory Layout, C++]
level: intermediate
---

# [Daily Graphics Growth] 2026-08-30 - Hybrid SDF Intersection Refinement: Bracketing, Secant/Bisection, and Sub-Voxel Surface Accuracy

## 1. 오늘의 개념
어제는 approximate SDF/level-set에서 **Lipschitz bound**를 이용해 `step <= |phi| / L` 형태의 conservative step을 만들고, surface를 건너뛰지 않으면서 빠르게 근처까지 접근하는 방법을 봤다. 오늘은 그 다음 단계인 **“surface 근처까지 왔을 때, 실제 교점 `t*`를 어떻게 안정적으로 sub-voxel 수준까지 좁힐 것인가?”**를 다룬다.

Ray를

`x(t) = o + t d`

라고 하고 scalar field를 `phi(x)`라 하면 ray-surface intersection은 1차원 함수

`g(t) = phi(o + t d)`

의 root

`g(t*) = 0`

을 찾는 문제가 된다.

Sphere tracing은 surface까지 안전하게 접근하는 데 강하지만, 마지막에 흔히 쓰는 `abs(phi) < epsilon` 종료 조건은 **field-space residual**이지 직접적인 **ray-parameter error**나 **world-space geometric error**가 아니다. 특히 field가 exact SDF가 아니거나 local gradient magnitude가 1과 크게 다르면 `|phi|`가 작다는 사실만으로 교점 위치가 충분히 정확하다고 말할 수 없다.

그래서 실무적인 implicit renderer는 traversal과 refinement를 분리해서 생각하는 편이 좋다.

- **Traversal phase**: Lipschitz-aware sphere tracing / hierarchy traversal로 surface 후보 구간까지 빠르게 접근
- **Refinement phase**: sign-changing interval 또는 certified local interval을 만든 뒤 root-finding으로 `t`를 좁힘

오늘의 핵심은 refinement에서 **Bracketing + Secant + Bisection**을 조합하는 것이다. Secant는 잘 behaved한 구간에서 빠르고, Bisection은 bracket이 유지되는 한 매우 강한 convergence guarantee를 제공한다. GPU에서는 이 둘을 고정 iteration 수의 hybrid rule로 묶으면 성능 예측성과 robustness를 동시에 얻기 쉽다.

## 2. 한 줄 핵심
> Sphere tracing은 “안전하게 가까이 가는 알고리즘”이고, 최종 intersection accuracy는 **sign bracket을 유지한 채 secant의 빠른 수렴과 bisection의 보수적 보장을 결합하는 별도의 root refinement 단계**로 다루는 것이 안정적이다.

## 3. 왜 중요한가
### 3.1 `abs(phi) < epsilon`은 geometric error가 아니다
Exact SDF라면 `|phi(x)|`가 surface까지의 실제 거리와 일치하므로 field residual과 geometric error가 거의 같은 의미를 갖는다. 하지만 다음 상황에서는 그렇지 않다.

- level-set advection 이후 `|∇phi| != 1`
- CSG / smoothing / resampling으로 distance property가 깨짐
- FP16 quantization 또는 compressed brick representation
- coarse LOD / fallback field 사용
- trilinear reconstruction 자체의 interpolation error

예를 들어 `|∇phi|`가 큰 영역에서는 매우 작은 `|phi|` 변화가 작은 spatial displacement에 해당할 수 있고, 반대로 gradient가 작다면 `|phi|`가 작아도 root까지 상당한 거리 차이가 남을 수 있다.

즉 rendering correctness 관점에서는 **field tolerance `epsilon_phi`와 ray-space tolerance `epsilon_t`를 분리**해야 한다.

### 3.2 Bracket은 단순한 숫자 두 개가 아니라 correctness certificate다
연속 함수 `g(t)`에 대해

`g(t_a) * g(t_b) <= 0`

이면 Intermediate Value Theorem에 의해 `[t_a, t_b]` 안에 최소 하나의 root가 존재한다.

이 bracket은 refinement가 실패하지 않게 만드는 가장 중요한 상태다. Secant나 Newton step이 빠르더라도 bracket 밖으로 나가거나 grazing configuration에서 잘못된 방향으로 진행하면 root를 놓칠 수 있다. 반면 bracket을 유지한 채 interval width를 계속 줄이면 최소한 “root가 이 범위 안에 있다”는 geometric uncertainty가 직접 줄어든다.

### 3.3 Sub-voxel accuracy의 의미를 정확히 이해해야 한다
Grid SDF에서 refinement를 많이 하면 voxel size보다 훨씬 작은 `t` interval을 얻을 수 있다. 이것을 흔히 **sub-voxel intersection accuracy**라고 부를 수 있다.

하지만 이는 원래 continuous geometry를 sub-voxel 수준으로 복원했다는 뜻이 아니다. 실제로 얻는 것은 보통 **trilinearly reconstructed scalar field의 zero set**에 대한 정밀 교점이다.

즉 두 종류의 error를 분리해야 한다.

- **solver error**: 현재 reconstructed field의 root를 얼마나 정확히 찾았는가
- **representation error**: 그 reconstructed zero set이 실제 target geometry와 얼마나 다른가

Root refinement는 첫 번째를 크게 줄일 수 있지만 두 번째는 field resolution, reconstruction scheme, quantization, simulation quality에 의해 결정된다.

### 3.4 Thin feature와 measurement에서 특히 중요하다
얇은 oxide, trench sidewall, narrow gap, contact edge처럼 geometry scale이 voxel 크기와 비슷한 경우 intersection `t`의 작은 오차가 thickness, depth, angle 같은 측정량에 바로 영향을 준다.

그래서 visualization이 단순히 “예뻐 보이는 것”을 넘어 metrology-like query, picking, cross-section measurement까지 담당한다면 final root refinement는 rendering quality가 아니라 **measurement stability**의 일부가 된다.

## 4. 구현 관점
### 4.1 Ray-surface intersection을 1D root 문제로 본다
Ray

`x(t) = o + t d`

에 대해

`g(t) = phi(x(t))`

를 정의하면 implicit intersection은 `g(t)=0`의 root다.

GPU shader 입장에서는 3D field sampling problem이지만, root solver 입장에서는 **1D scalar function**이다. 이 관점 전환이 중요하다. Traversal은 3D hierarchy와 residency를 이용하지만 refinement에 들어온 순간 필요한 상태는 대부분 다음처럼 줄어든다.

`BracketState = { tLo, tHi, phiLo, phiHi }`

여기에 field/brick version이 필요하면 `epoch` 또는 compact handle이 붙는다.

### 4.2 Bracket 생성은 sphere tracing과 별도의 단계다
Exact SDF sphere tracing은 원칙적으로 surface를 overshoot하지 않고 한쪽에서 접근하기 때문에, 단순히 이전 sample과 현재 sample만 봐서는 sign change가 생기지 않을 수 있다. 따라서 “sphere tracing이 끝났으니 자동으로 bracket이 생겼다”고 생각하면 안 된다.

Bracket은 대체로 다음 정보 중 하나를 이용해 만든다.

- hierarchy/cell traversal에서 얻은 **local interval**
- zero crossing 가능성이 확인된 voxel/brick entry-exit interval
- 마지막 safe sample 근처에서 field continuity를 보존하는 controlled local probe
- 이미 overshoot가 감지된 approximate field에서의 adjacent opposite-sign samples

중요한 것은 bracket이 **현재 reconstruction과 같은 field snapshot**에서 계산되어야 한다는 점이다. `phiLo`는 이전 epoch, `phiHi`는 새 epoch라면 sign change 자체가 의미를 잃는다.

### 4.3 Bisection: 느리지만 geometric uncertainty를 확실히 줄인다
Bracket `[tLo, tHi]`에서 midpoint

`tMid = 0.5 * (tLo + tHi)`

를 평가하고 sign에 따라 한쪽 endpoint를 교체하면 interval width는 iteration마다 절반이 된다.

초기 interval width를 `W0`라 하면 `N`번 이후

`W_N = W0 / 2^N`

이다.

예를 들어 voxel-sized interval에서 10번이면 약 `1/1024 voxel`, 16번이면 약 `1/65536 voxel` 폭까지 줄어든다. 물론 실제 geometry accuracy가 그 정도라는 의미는 아니지만, **solver uncertainty 자체는 명확하게 bound**할 수 있다.

GPU 관점에서 bisection의 장점은 분명하다.

- derivative 불필요
- 분모 안정성 문제 없음
- fixed iteration count 설계가 쉬움
- 각 iteration의 cost가 거의 일정

단점은 선형 convergence라 field sample 수가 늘어난다는 점이다.

### 4.4 Secant: local field shape를 이용해 빠르게 root를 예측한다
두 endpoint `(tLo, phiLo)`, `(tHi, phiHi)`를 잇는 직선이 `phi=0`과 만나는 위치는

`tSec = tHi - phiHi * (tHi - tLo) / (phiHi - phiLo)`

이다.

Field가 root 주변에서 거의 linear하면 midpoint보다 훨씬 가까운 위치를 한 번에 고를 수 있다. Trilinear grid를 ray가 통과할 때 `g(t)`는 일반적으로 cell 내부에서 cubic 형태가 될 수 있지만, 충분히 작은 bracket에서는 local linear approximation이 잘 맞는 경우가 많다.

문제는 **순수 secant method는 bracket을 보존하지 않는다**는 점이다. 두 최근 iterate를 계속 사용하면 다음 점이 현재 interval 밖으로 나갈 수 있고, denominator가 작거나 grazing root에서 수렴이 불안정해질 수 있다.

그래서 renderer에서는 보통 “secant를 proposal로 쓰고 bracket은 절대 버리지 않는” 방식이 더 안전하다.

### 4.5 Bracketed Secant + Bisection의 hybrid rule
개념적으로 refinement iteration은 다음 구조를 가진다.

1. secant candidate `tSec` 계산
2. candidate가 bracket 내부에 있고 수치적으로 충분히 안정적인지 검사
3. 안정적이면 secant candidate 사용
4. 아니면 midpoint 사용
5. 새 sample의 sign으로 bracket 갱신

이 구조는 Brent 계열 root finder가 가진 철학과 비슷하다. Brent method는 bracket을 유지하면서 bisection과 interpolation 기반 step을 결합해 robustness와 빠른 convergence를 동시에 노린다. Production GPU shader에서는 full Brent 구현보다 더 단순한 fixed-rule hybrid가 instruction count와 branch predictability 측면에서 유리할 수 있다.

### 4.6 Secant denominator와 grazing intersection
Secant formula의 denominator

`phiHi - phiLo`

가 매우 작다면 두 endpoint의 field value가 거의 같다는 뜻이다. 이 경우 interpolation point가 멀리 튈 수 있다.

특히 ray가 surface를 거의 접선 방향으로 지나가는 **grazing intersection**에서는

`dg/dt = ∇phi · d`

가 0에 가까워질 수 있다. Root 주변에서 `g(t)`가 매우 평평해지므로 secant/Newton 계열 method가 불안정해진다.

따라서 hybrid solver는 다음 종류의 상태를 구분해야 한다.

- steep crossing: secant가 효과적
- shallow/grazing crossing: bisection 비중 증가
- multiple crossing 가능성이 있는 interval: bracket 자체를 더 잘 isolate해야 함

“secant가 실패하면 midpoint”라는 fallback은 단순하지만 GPU에서 상당히 강한 safety valve가 된다.

### 4.7 Newton refinement와의 비교
Gradient를 이미 갖고 있다면

`g'(t) = ∇phi(x(t)) · d`

이므로 Newton step

`tNew = t - g(t) / g'(t)`

도 가능하다. Root 근처에서 derivative가 안정적이면 매우 빠르게 수렴한다.

하지만 implicit renderer에서는 다음 문제가 있다.

- `∇phi · d ≈ 0`인 grazing ray
- quantized/noisy gradient
- brick border의 gradient discontinuity
- coarse/fine LOD 경계
- Newton step이 bracket 밖으로 나가는 문제

그래서 Newton은 “항상 더 좋은 solver”라기보다 **gradient quality가 높은 경우에만 bracket 내부 candidate로 허용할 수 있는 추가 proposal**로 보는 편이 안전하다.

GPU 실무에서는 solver sophistication보다 field sample cost, divergence, live register 수가 더 큰 병목이 될 수도 있다.

### 4.8 Termination은 `epsilon_phi`보다 `epsilon_t` 중심으로 본다
Refinement의 목표가 intersection position이라면 가장 직접적인 종료 기준은

`tHi - tLo <= epsilon_t`

이다.

이 값은 ray parameter 기준의 geometric uncertainty를 직접 bound한다. 여기에 보조적으로

`abs(phi(tCandidate)) <= epsilon_phi`

를 함께 볼 수 있다.

`epsilon_t`는 고정 world-space 값만 쓰는 것보다 다음 요소와 연결하는 편이 합리적이다.

- local voxel size
- current LOD cell size
- camera pixel footprint
- target measurement tolerance
- ray distance에 따른 floating-point resolution

예를 들어 화면에서 한 pixel보다 훨씬 작은 error까지 refinement하는 것은 rendering에는 낭비일 수 있지만, picking/metrology path라면 더 작은 tolerance가 의미가 있다.

### 4.9 Trilinear field에서 “정확한 root”의 의미
Grid sample을 trilinear interpolation하는 경우 voxel 내부 field는

`phi(x,y,z) = c0 + c1 x + c2 y + c3 z + c4 xy + c5 yz + c6 zx + c7 xyz`

형태다.

Ray에서 `x,y,z`가 모두 `t`의 1차식이므로 `g(t)`는 일반적으로 **3차 polynomial**이 된다. 이론적으로 cell 내부 root를 polynomial solve로 구할 수도 있다.

하지만 production renderer에서 iterative refinement가 여전히 매력적인 이유가 있다.

- texture hardware가 제공하는 실제 interpolation semantics와 동일한 field evaluator 사용 가능
- quantization/compression/custom reconstruction을 그대로 포함 가능
- solver를 representation별로 바꾸지 않아도 됨
- cubic closed-form의 수치 안정성과 복잡도를 피할 수 있음
- coarse/fine mixed representation에도 동일한 refinement framework 적용 가능

즉 “analytic cubic solve가 가능하다”와 “GPU production에서 그것이 가장 좋은 선택이다”는 다른 문제다.

### 4.10 Precision: FP16 field와 FP32 root state를 구분한다
Field가 FP16으로 저장되어 있어도 `tLo/tHi`와 secant arithmetic은 FP32로 유지하는 편이 일반적으로 안전하다. Root state까지 FP16으로 줄이면 bracket width가 작아질수록 endpoint가 같은 representable value로 collapse할 수 있다.

반대로 field sample 자체가 FP16 quantized라면 FP32 solver iteration을 무한히 늘려도 representation error는 줄지 않는다. 결국 최종 accuracy는

`total error ≈ representation error + interpolation error + solver error + floating-point error`

의 합으로 봐야 한다.

특히 world coordinate가 매우 크고 local feature가 작은 장면에서는 `o + t d` 계산에서 유효 자릿수가 손실될 수 있다. 이 경우 object/local coordinate 또는 brick-local coordinate에서 refinement하는 것이 단순한 precision 최적화가 아니라 geometry stability에 직접 영향을 준다.

### 4.11 GPU memory layout과 queue 설계
Refinement가 필요한 ray만 별도 queue에 넣는다면 각 item은 비교적 작은 상태로 표현할 수 있다.

**Hot refinement state**
- `tLo`, `tHi`
- `phiLo`, `phiHi`
- compact field/brick handle
- field epoch

**Cold/debug state**
- initial bracket width
- iteration count
- fallback-to-bisection count
- final residual
- failure reason

AoS 형태로 모든 값을 크게 묶기보다, large batch refinement에서는 `tLo/tHi`, `phiLo/phiHi`, handle처럼 매 iteration 읽는 값과 debug metadata를 분리하는 것이 cache/coalescing 측면에서 유리할 수 있다.

또 refinement를 모든 ray가 수행하도록 만들면 already-missed ray와 already-converged ray까지 같은 kernel path에 남아 divergence가 커질 수 있다. Wavefront renderer라면 **candidate compaction → fixed-count refinement pass**가 자연스럽고, megakernel이라면 refinement iteration 수를 제한해 live state와 divergence를 통제하는 것이 중요하다.

### 4.12 Branch divergence와 fixed iteration budget
CPU root solver는 tolerance를 만족하면 즉시 빠져나오는 dynamic loop가 자연스럽다. GPU에서는 lane마다 convergence 속도가 다르면 warp 내부 divergence가 커진다.

그래서 graphics workload에서는 다음 trade-off가 생긴다.

- dynamic early exit: 평균 sample 수 감소, divergence 증가 가능
- fixed iteration count: 약간의 over-computation, execution coherence 향상

Bracket 폭이 이미 작다는 전제라면 `4~8`회 정도의 fixed hybrid iteration만으로도 큰 개선을 얻는 경우가 많다. 중요한 것은 iteration 수 자체보다 **초기 bracket quality와 field sample cost**다.

### 4.13 Field epoch와 refinement correctness
CUDA simulation이 `phi`를 갱신하고 Vulkan renderer가 intersection을 계산하는 구조라면 bracket 생성과 refinement가 같은 logical field snapshot을 사용해야 한다.

예를 들어 frame N에서 생성한 `(tLo, phiLo)`를 들고 있다가 frame N+1의 field로 `phiHi` 또는 intermediate sample을 평가하면 bracket invariant가 깨질 수 있다.

따라서 refinement resource contract는 개념적으로

`HitCandidate { tLo, tHi, phiLo, phiHi, fieldHandle, fieldEpoch, lodLevel }`

처럼 field identity를 포함해야 한다.

External semaphore나 timeline semaphore는 execution ordering을 보장하지만, **이 bracket이 어느 semantic field version에 속하는지**는 application-level data contract가 보장해야 한다.

### 4.14 Refinement 이후 normal 계산
교점 위치 `x(t*)`가 안정되면 shading normal은 같은 reconstructed field에서 계산한 gradient를 사용하는 것이 일관성이 좋다.

`n = normalize(∇phi(x(t*)))`

하지만 intersection은 trilinear field를 기준으로 찾고 normal은 다른 smoothing/filtering field에서 계산하면 silhouette position과 shading orientation의 semantic mismatch가 생길 수 있다.

특히 brick border에서 finite-difference gradient를 쓰는 경우 neighboring brick residency/halo 정책에 따라 normal이 흔들릴 수 있다. 이 문제는 내일 다룰 **gradient-consistent surface attributes**와 직접 연결된다.

## 5. 내 관심 분야와 연결
Semiconductor process emulation에서 level-set 또는 SDF는 deposition, etch, oxidation, CMP 같은 공정 이후의 3D 구조를 표현하는 핵심 representation이 될 수 있다. 이 field를 직접 ray-march해 surface를 보여주거나 cross-section/picking에 사용한다면 오늘의 refinement는 다음 문제와 직접 연결된다.

첫째, **thin film thickness**다. Oxide나 resist layer 두께가 voxel 크기와 비슷할 때 두 surface의 intersection `t`를 각각 coarse `abs(phi)<epsilon`으로 잡으면 thickness가 camera angle이나 step history에 따라 흔들릴 수 있다. Bracketed refinement는 같은 reconstructed zero set에 대해 더 안정적인 front/back intersection을 제공한다.

둘째, **trench depth / sidewall angle / CD-like measurement**다. 화면에 그릴 때는 0.2 voxel 오차가 잘 보이지 않아도 측정값에서는 누적될 수 있다. Rendering path와 measurement path가 같은 intersection primitive를 공유하면 visual picking과 numerical query 사이의 차이를 줄일 수 있다.

셋째, **CUDA simulation → Vulkan/WebGPU visualization** 같은 GPU-stay-GPU pipeline이다. Simulation field가 sparse brick, NanoVDB-like hierarchy, custom ColumnStack/voxel representation으로 존재한다면 traversal 단계는 representation-specific일 수 있지만, 일단 `g(t)` evaluator와 local bracket을 제공하면 refinement layer는 비교적 representation-independent하게 만들 수 있다.

넷째, **Marching Cubes mesh와 direct implicit rendering의 차이**다. Mesh extraction은 surface position을 vertex interpolation 시점에 결정하고 이후 rasterization은 그 mesh를 따른다. Direct implicit rendering은 ray마다 zero set을 다시 solve한다. 같은 source field라도 두 경로의 geometric error budget과 temporal behavior가 다르므로, portfolio에서 이 차이를 설명할 수 있으면 단순 API 사용이 아니라 representation-level reasoning을 보여줄 수 있다.

## 6. 머릿속에 남길 질문 3개
1. **Sphere tracing이 surface를 안전하게 접근했다면, 왜 마지막 `abs(phi)<epsilon`만으로는 충분하지 않고 별도의 bracketed root refinement가 필요한가?**
2. **Secant method가 bisection보다 빠를 수 있는데도 production GPU intersection에서 bracket을 절대 버리지 않는 설계가 중요한 이유는 무엇인가?**
3. **Sub-voxel root accuracy를 얻었다고 해서 실제 geometry가 sub-voxel 정확도로 복원되었다고 말할 수 없는 이유는 무엇인가?**

## 7. graphics engineer 면접 질문 1개와 답변
### 질문
**SDF ray marcher에서 surface 근처에 도달한 뒤 intersection을 더 정확하게 만들고 싶습니다. Secant, bisection, Newton 중 어떤 전략을 선택하겠습니까? GPU에서의 trade-off까지 설명해 주세요.**

### 답변
먼저 intersection을 `g(t)=phi(o+td)=0`인 1D root 문제로 보고, 가장 중요한 것은 **root가 존재하는 bracket을 유지하는 것**이라고 답할 수 있다.

Bisection은 연속 함수와 sign-changing bracket만 있으면 interval을 매 iteration 절반으로 줄이므로 매우 robust하지만 convergence가 선형이다. Secant는 endpoint의 function value를 이용해 root 위치를 interpolation하므로 잘 behaved한 crossing에서는 훨씬 빠르지만, 순수 secant method는 bracket을 유지하지 않아 unstable할 수 있다. Newton은 `g'(t)=∇phi·d`를 이용해 root 근처에서 빠르게 수렴할 수 있지만 grazing ray에서 derivative가 작거나 gradient가 noisy한 경우 위험하다.

따라서 GPU production path에서는 **bracketed hybrid**가 좋은 선택이다. Secant 또는 Newton candidate가 bracket 내부이고 numerically stable할 때만 사용하고, 그렇지 않으면 midpoint bisection으로 fallback한다. 이렇게 하면 fast proposal과 guaranteed interval shrink를 결합할 수 있다.

GPU에서는 root-finder의 이론적 iteration 수뿐 아니라 field sample cost, warp divergence, live register 수가 중요하다. 그래서 CPU 스타일의 복잡한 adaptive solver보다, 초기 bracket을 충분히 작게 만든 뒤 소수의 fixed-count secant/bisection iteration을 수행하는 구조가 더 예측 가능한 경우가 많다. 또한 final tolerance는 `abs(phi)`만 보지 않고 `tHi-tLo` 같은 geometric interval width를 함께 보는 것이 좋다.

마지막으로 sub-voxel solver accuracy와 field representation accuracy를 구분해야 한다. FP16/trilinear field의 root를 매우 정밀하게 찾아도 원래 geometry와의 오차는 field resolution과 interpolation에 의해 남아 있다.

## 8. 포트폴리오 / 커리어 연결
이 개념은 포트폴리오에서 단순한 “SDF ray marching 구현”보다 훨씬 깊은 graphics engineering 역량을 보여주는 소재다.

좋은 설명 포인트는 **correctness contract → numerical method → GPU execution → data representation**을 한 흐름으로 연결하는 것이다.

예를 들어 architecture 설명에서 다음 차이를 명확히 말할 수 있다.

- Hierarchical/Lipschitz traversal은 **surface를 놓치지 않고 candidate interval까지 도달하는 단계**
- Bracketed refinement는 **candidate interval 안에서 geometric uncertainty를 줄이는 단계**
- Gradient evaluation은 **refined zero set에서 shading attribute를 만드는 단계**
- Sparse residency/epoch은 이 세 단계가 **같은 field snapshot**을 사용하도록 만드는 data-lifetime contract

Performance 섹션에서는 단순 FPS보다 다음 지표가 더 설득력이 있다.

- average / p95 sphere-tracing step count
- refinement field evaluations per hit
- secant acceptance ratio
- bisection fallback ratio
- final bracket width distribution
- hit-position temporal variance
- FP16 vs FP32 field storage에 따른 intersection error
- voxel size 대비 root solver error
- refinement queue occupancy와 active-lane ratio

이런 프로파일링 관점은 rendering engineer, engine programmer, GPU compute engineer 면접에서 “알고리즘을 알고 있다”를 넘어 **왜 이 구조가 robust하며 어디가 실제 병목인지 설명할 수 있는 사람**이라는 신호가 된다.

또 simulation/visualization 제품에서는 동일 zero-set intersection primitive를 rendering, picking, probe measurement, cross-section query에 재사용할 수 있다는 점이 architecture적으로 중요하다. 하나의 numerical contract가 여러 feature의 consistency를 높이는 사례이므로 실무 경험 설명에도 매우 좋은 소재다.

## 9. 내일 이어서 볼 개념
**Gradient-Consistent SDF Surface Attributes: Analytic Trilinear Gradients, Normal Stability, and Curvature-Aware Shading**

오늘은 ray parameter `t*`를 안정적으로 찾는 데 집중했다. 하지만 교점이 정확해져도 normal이 noisy하거나 field reconstruction과 일치하지 않으면 specular highlight, silhouette-adjacent shading, curvature visualization이 흔들릴 수 있다.

내일은 refined hit point에서

- trilinear interpolant의 analytic gradient
- central difference와 analytic derivative의 차이
- brick border / halo / sparse residency에서 normal continuity
- gradient normalization이 감추는 field-quality 정보
- Hessian/curvature estimate의 비용과 안정성
- FP16 gradient storage vs on-the-fly reconstruction
- intersection field와 shading field를 다르게 쓸 때 생기는 semantic mismatch

를 중심으로 **geometry solver의 정확도를 shading attribute의 정확도로 연결**한다.

## 10. 참고 키워드
- **Sphere Tracing / Signed Distance Bound**
- **Ray-Implicit Surface Intersection**
- **Root Bracketing**
- **Intermediate Value Theorem**
- **Bisection Method**
- **Secant Method**
- **Regula Falsi / False Position**
- **Brent Method / Brent-Dekker Method**
- **Newton-Raphson Refinement**
- **Grazing Intersection**
- **Trilinear Interpolation**
- **Sub-Voxel Surface Accuracy**
- **Representation Error vs Solver Error**
- **Field-Space Tolerance vs Geometric Tolerance**
- **Implicit Surface Gradient**
- **Wavefront Refinement Queue**
- **SIMT Divergence**
- **FP16 Field / FP32 Root State**
- **Field Epoch / Snapshot Consistency**
- **Hart 1996, Sphere Tracing: A Geometric Method for the Antialiased Ray Tracing of Implicit Surfaces**
  - https://utensil.github.io/forest/hart1996sphere/
- **SciPy `brentq` documentation — bracketed hybrid root finding**
  - https://docs.scipy.org/doc/scipy/reference/generated/scipy.optimize.brentq.html
- **Boost.Math Root Finding Algorithms**
  - https://beta.boost.org/doc/libs/1_86_0/libs/math/doc/html/root_finding.html
- **Knoll et al., Fast Ray Tracing of Arbitrary Implicit Surfaces with Interval and Affine Arithmetic**
  - https://diglib.eg.org/items/bf588fc9-fbbe-463b-825c-ba3834019a6c
