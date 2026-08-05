---
title: "Variance Stabilization Across À-Trous Scales and Filtered Moment Propagation"
date: "2026-08-05"
category: Graphics
tags: ["GPU", "Denoising", "À-Trous Wavelet", "Variance", "Statistical Moments", "Compute Shader", "Memory Layout", "SVGF", "Ray Tracing", "Scientific Visualization"]
level: intermediate
---

# [Daily Graphics Growth] 2026-08-05 - Variance Stabilization Across À-Trous Scales and Filtered Moment Propagation

## 1. 오늘의 개념

전날의 **Variance-Guided À-Trous Wavelet Filtering and Edge-Stopping Functions**에서는 variance가 필터 강도를 정하고, depth·normal·material·roughness 기반 edge-stopping weight가 필터가 넘어가도 되는 경계를 정한다는 구조를 살펴봤다. 오늘은 그 다음 단계다. À-trous iteration이 반복될 때 variance와 statistical moment를 어떻게 갱신해야 다음 scale의 판단이 무너지지 않는지를 다룬다.

오늘의 개념은 **Variance Stabilization Across À-Trous Scales**와 **Filtered Moment Propagation**이다.

À-trous filter는 iteration마다 sample stride를 `1, 2, 4, 8, ...`로 확대한다. Radiance만 필터링하고 variance를 이전 값 그대로 사용하면, 다음 iteration은 이미 smoothing된 signal과 아직 noisy한 variance를 함께 보게 된다. 반대로 variance를 단순 평균하면 smoothing 이후의 불확실성을 과소평가하거나, 실제 surface detail을 noise로 잘못 포함할 수 있다.

이를 이해하려면 서로 다른 두 종류의 variance를 분리해야 한다.

1. **Local distribution variance**: 주변 sample 값이 얼마나 서로 다른가. Noise뿐 아니라 실제 shading variation과 edge도 포함한다.
2. **Estimator variance**: 현재 필터 결과가 참값에서 얼마나 흔들릴 것으로 예상되는가. 독립 noise를 평균할수록 감소한다.

Pixel `i`의 평균을 `μ_i`, variance를 `σ_i²`, 정규화된 filter weight를 `a_i`라 하면 filtered mean은 다음과 같다.

`μ' = Σ_i a_i μ_i`,  단 `Σ_i a_i = 1`

First/second raw moment를 필터링하는 방식은 다음과 같다.

`m1' = Σ_i a_i m1_i`

`m2' = Σ_i a_i m2_i`

`σ_mix² = max(m2' - (m1')², 0)`

여기서 `σ_mix²`는 neighborhood가 가진 전체 분산으로, 각 sample 내부의 uncertainty와 sample 평균 사이의 차이를 함께 포함한다. 이는 **law of total variance**의 형태로 다음처럼 해석할 수 있다.

`σ_mix² = Σ_i a_i σ_i² + Σ_i a_i (μ_i - μ')²`

첫 항은 within-sample uncertainty, 두 번째 항은 between-sample variation이다. 하지만 필터 결과 자체의 estimator variance는 독립 noise를 가정하면 다음에 더 가깝다.

`σ_est² = Σ_i a_i² σ_i²`

즉 filtered moments로 얻는 variance와 weighted average의 uncertainty는 같은 값이 아니다. 실시간 denoiser에서 scale이 진행될수록 variance가 불안정해지는 핵심 이유가 바로 이 두 의미를 하나의 buffer로 뭉쳐 사용하기 때문이다.

## 2. 한 줄 핵심

**À-trous scale이 커질수록 radiance와 moment를 같은 support로 전파하되, ‘주변 값의 변화량’과 ‘필터 결과의 불확실성’을 구분해야 variance가 과도하게 붕괴하거나 팽창하지 않는다.**

## 3. 왜 중요한가

Variance-guided denoiser는 현재 variance를 다음 filter scale의 luminance weight와 filter radius 결정에 다시 사용한다. 따라서 variance update의 작은 오류가 다음 iteration에서 증폭된다.

Variance가 지나치게 작아지면 다음 문제가 생긴다.

- luminance edge-stopping weight가 너무 엄격해져 residual noise가 남는다.
- filter support가 scale마다 갑자기 끊겨 checkerboard 또는 speckle이 유지된다.
- disocclusion과 짧은 history 영역이 충분히 복구되지 않는다.
- frame마다 support가 미세하게 바뀌며 temporal shimmer가 발생한다.

Variance가 지나치게 커지면 반대 문제가 생긴다.

- 실제 highlight와 texture-driven lighting variation이 noise로 간주된다.
- glossy reflection이 넓게 퍼지고 contact detail이 사라진다.
- thin geometry와 silhouette을 넘어 light leaking이 발생한다.
- 후반 À-trous scale이 사실상 large-radius blur처럼 동작한다.

특히 filtered moment를 그대로 재사용할 때 주의할 점은 **filtering이 signal의 frequency content 자체를 바꾼다**는 것이다. 첫 scale의 출력은 raw Monte Carlo sample이 아니라 이미 주변 sample이 섞인 estimator다. 두 번째 scale의 tap들은 서로 독립적이지 않으며, 서로 겹치는 neighborhood를 공유한다. 독립성을 가정한 `Σ a_i² σ_i²`는 covariance를 무시하므로 uncertainty를 지나치게 낮게 잡을 수 있다.

정확한 식에는 covariance가 포함된다.

`Var(Σ_i a_i X_i) = Σ_i a_i² Var(X_i) + 2Σ_{i<j} a_i a_j Cov(X_i, X_j)`

실시간 pipeline에서 모든 covariance를 저장하는 것은 비현실적이다. 따라서 실제 엔진은 다음과 같은 근사를 사용한다.

- scale-dependent variance floor
- correlation inflation factor
- effective sample count 기반 보정
- residual energy와 propagated variance의 결합
- low-confidence/history-repaired 영역의 variance 재팽창

SVGF는 temporal accumulation으로 얻은 luminance moment와 variance를 hierarchical wavelet filtering에 사용한다. 핵심은 variance를 단순한 디버그 값이 아니라 **다음 reconstruction pass의 제어 신호(control signal)**로 취급한다는 점이다. 한 번 과소평가된 variance는 이후 scale의 support를 축소하고, 한 번 과대평가된 variance는 이후 scale의 edge 보존을 약화한다.

## 4. 구현 관점

### 4.1 첫 번째 moment와 두 번째 moment

Scalar luminance `Y`에 대해 raw moments는 다음과 같다.

`m1 = E[Y]`

`m2 = E[Y²]`

`σ² = max(m2 - m1², 0)`

Temporal accumulation 단계에서는 current sample과 reprojected history를 혼합해 `m1`, `m2`를 갱신할 수 있다. Spatial À-trous 단계에서는 radiance와 동일하거나 호환되는 edge-stopping weight로 moments를 필터링해야 한다.

Radiance는 geometry guide를 통과한 이웃만 사용했는데 moment는 일반 bilinear 또는 box filter로 갱신하면, 두 buffer가 서로 다른 모집단을 표현하게 된다. 그러면 `m2 - m1²`가 현재 radiance support와 맞지 않으며 edge 근처에서 variance spike 또는 collapse가 발생한다.

따라서 scale `k`의 정규화 weight를 `a_i^(k)`라고 하면 다음 관계를 유지하는 것이 기본이다.

`L^(k+1) = Σ_i a_i^(k) L_i^(k)`

`m1^(k+1) = Σ_i a_i^(k) m1_i^(k)`

`m2^(k+1) = Σ_i a_i^(k) m2_i^(k)`

다만 color radiance가 RGB이고 moment가 luminance scalar라면, radiance weight가 luminance에서 계산되는지, per-channel 차이를 포함하는지 resource contract를 명확히 해야 한다.

### 4.2 Mixture variance와 estimator variance

Filtered raw moment로 계산한 `σ_mix²`는 neighborhood 안에서 값이 얼마나 다양한지를 나타낸다.

`σ_mix² = Σ_i a_i (σ_i² + μ_i²) - (Σ_i a_i μ_i)²`

이를 전개하면 다음 두 요소가 나온다.

- `Σ_i a_i σ_i²`: 각 sample이 가진 기존 uncertainty
- `Σ_i a_i (μ_i - μ')²`: 이웃 평균 사이의 variation

두 번째 항은 edge나 실제 illumination gradient에서도 커진다. Edge-stopping function이 완벽하지 않다면 geometry boundary의 차이가 variance에 들어가고, 다음 scale이 그 경계를 noise라고 판단할 수 있다.

반면 output estimator uncertainty는 독립 가정 아래 다음과 같다.

`σ_est² = Σ_i a_i² σ_i²`

동일한 variance를 가진 `N`개의 sample을 균등 평균하면 `a_i = 1/N`이므로 다음이 된다.

`σ_est² = σ² / N`

이는 평균이 noise를 줄이는 직관과 맞는다. 그러나 실제 À-trous tap은 edge weight로 인해 균등하지 않고, 이전 scale의 결과가 서로 겹치므로 완전 독립도 아니다.

실무적으로는 두 variance를 모두 의미 있게 사용할 수 있다.

- `σ_mix²`: local detail/noise discrimination, edge-aware confidence
- `σ_est²`: 다음 scale의 expected noise magnitude, filter strength

Buffer를 두 개 유지할 수 없다면 다음과 같은 conservative combination이 가능하다.

`σ_next² = max(σ_est² · c_corr, k_residual · σ_residual², σ_floor²)`

여기서 `c_corr`는 correlation inflation, `σ_residual²`는 filtered mean 주변의 residual energy다.

### 4.3 Effective tap count

정규화 weight `a_i`로부터 effective sample count를 다음처럼 정의할 수 있다.

`N_eff = 1 / Σ_i a_i²`

모든 tap이 같은 weight를 가지면 `N_eff`는 실제 tap 수와 같다. 하나의 center tap만 지배하면 `N_eff`는 1에 가까워진다.

`N_eff`는 단순 tap count보다 유용하다.

- depth/normal rejection으로 support가 줄었는지 표현한다.
- 큰 kernel을 사용해도 실제로 유효한 sample이 적은 영역을 구분한다.
- variance reduction이 어느 정도 가능한지 추정한다.
- thin geometry, silhouette, glossy highlight에서 과도한 confidence를 방지한다.

간단한 근사는 다음과 같다.

`σ_est² ≈ σ_input² / N_eff`

하지만 이미 spatially filtered된 tap들은 상관되어 있으므로 scale이 커질수록 실제 감소폭은 이보다 작다. 이를 보정하기 위해 scale별 correlation factor를 둘 수 있다.

`σ_est,stable² = c_k · Σ_i a_i² σ_i²`

`c_k >= 1`

`c_k`는 고정 상수일 수도 있고, stride, filter iteration, local motion, history length에 따라 달라질 수도 있다. 중요한 것은 exact covariance를 복원하는 것이 아니라 **variance가 비현실적으로 0에 수렴하지 않도록 하는 것**이다.

### 4.4 Residual 기반 variance 재주입

Moment propagation만 사용하면 edge-aware filtering이 놓친 실제 residual noise를 빠르게 잃을 수 있다. 이를 보완하기 위해 filtered mean 주변의 residual energy를 측정할 수 있다.

`σ_residual² = Σ_i a_i (Y_i - μ')²`

이는 사실상 weighted local variance이며 noise와 detail을 모두 포함한다. 따라서 그대로 estimator variance로 사용하면 edge에서 과대평가된다. 하지만 geometry/material compatible tap만 사용하고 robust clamp를 적용하면 useful floor가 된다.

대표적인 안정화 형태는 다음과 같다.

`σ_next² = max(σ_prop², β_k · σ_residual², σ_floor,k²)`

- `σ_prop²`: squared-weight로 전파한 estimator variance
- `β_k`: scale별 residual injection 비율
- `σ_floor,k²`: precision 및 low-sample 영역을 위한 최소 variance

또는 smooth blend를 사용할 수 있다.

`σ_next² = lerp(σ_prop², σ_residual², r_k)`

여기서 `r_k`는 다음 조건에서 커질 수 있다.

- history length가 짧음
- disocclusion 또는 repaired history
- motion vector confidence가 낮음
- temporal clamp가 강하게 작동함
- input sample이 firefly clamp에 걸림

### 4.5 Scale별 variance floor와 gain

모든 À-trous iteration에서 같은 variance floor를 사용하면 HDR luminance 범위와 scale 변화에 대응하기 어렵다. Floor는 절대값뿐 아니라 signal magnitude에 상대적인 형태가 유용하다.

`σ_floor² = σ_abs² + (k_rel · |μ|)²`

이 식은 어두운 영역에서는 absolute floor가, 밝은 영역에서는 relative floor가 지배하도록 한다. HDR scene에서 second moment는 radiance 제곱을 포함하므로 FP16 범위와 precision 문제가 커진다. Exposure-normalized 또는 pre-exposed domain에서 moments를 저장하면 수치 안정성이 좋아진다.

Scale-dependent gain을 다음처럼 둘 수도 있다.

`σ_stable,k² = clamp(g_k · σ_next², σ_min,k², σ_max,k²)`

초기 scale은 raw noise를 많이 보므로 `g_k`를 보수적으로 유지하고, 후반 scale은 correlation과 edge leakage 위험이 커지므로 variance collapse를 막는 방향으로 조정한다.

단, scale마다 임의의 gain을 튜닝하면 scene-dependent artifact가 늘어난다. 실무에서는 gain보다 다음 정보를 우선 활용하는 편이 설명 가능성이 높다.

- `N_eff`
- history length
- reprojection confidence
- material/surface identity purity
- local depth/normal gradient
- filter stride

### 4.6 Luminance variance와 RGB signal

Variance를 luminance 하나로 관리하면 memory와 bandwidth가 줄고, filter strength를 perceptual brightness에 연결하기 쉽다. 하지만 강한 chromatic noise나 saturated lighting에서는 RGB channel의 불확실성이 다를 수 있다.

대안은 다음과 같다.

- luminance variance 1채널
- luma + chroma variance
- RGB per-channel moments
- YCoCg 또는 opponent color space moments
- scalar variance와 chroma clamp의 결합

실시간 엔진에서는 luminance moments가 일반적이지만, emissive neon, colored caustics, spectral-like effects에서는 luma만으로 chroma noise를 놓칠 수 있다. 반대로 RGB second moment는 buffer 비용과 bandwidth가 크게 증가한다.

Signal별 분리도 중요하다.

- diffuse radiance와 specular radiance는 별도 moments
- shadow visibility는 Bernoulli variance 구조
- hit distance는 radiance variance와 다른 통계
- scalar scientific field는 physical unit과 gradient 규모를 반영

### 4.7 GPU pass와 resource dependency

개념적인 render graph 흐름은 다음과 같다.

1. Temporal accumulation: radiance, `m1`, `m2`, history length 생성
2. Variance estimation: `max(m2 - m1², floor)`
3. À-trous scale `k`: guide-aware weight 계산
4. Radiance filtering
5. Filtered moment 또는 propagated variance 계산
6. `N_eff`, confidence, residual 기반 stabilization
7. 다음 scale의 radiance/variance 입력으로 ping-pong

한 pass에서 radiance와 moments를 동시에 처리하면 같은 tap과 guide를 재사용할 수 있다. 장점은 다음과 같다.

- depth/normal/material fetch 재사용
- 동일한 normalized weight 보장
- shared memory tile 재사용
- pass 수 및 global memory round trip 감소

단점은 register pressure와 UAV write 수 증가다. RGB radiance, moments, variance, weight sum, squared weight sum, residual을 동시에 유지하면 occupancy가 떨어질 수 있다.

분리 pass는 단순하지만 guide와 tap을 다시 읽으므로 bandwidth가 증가한다. 실제 선택은 shader register count, target GPU, resolution, tap 수, async compute 사용 여부에 따라 달라진다.

### 4.8 Memory layout

대표적인 full-resolution layout은 다음과 같다.

| Resource | 예시 Format | 역할 |
|---|---:|---|
| Filtered radiance ping/pong | `RGBA16F` × 2 | RGB signal + optional metadata |
| Moments ping/pong | `RG16F` × 2 | first/second luminance moments |
| Variance | `R16F` | stabilized variance |
| Confidence / history length | `R8_UNORM` 또는 `R16F` | temporal 신뢰도 |
| Normal + roughness | packed `RGBA16F`, `R10G10B10A2`, octahedral encoding | edge guide |
| Linear depth | `R32F` 또는 `R16F` | geometry guide |
| Material/surface ID | `R16_UINT` 또는 packed integer | hard compatibility gate |

1920×1080 기준 대략적인 texture 크기는 다음과 같다.

- `RGBA16F`: 약 15.8 MiB
- `RG16F`: 약 7.9 MiB
- `R16F`: 약 4.0 MiB
- `R8`: 약 2.0 MiB

Radiance와 moments를 모두 ping-pong하면 transient memory가 빠르게 증가한다. Render graph aliasing을 사용해 temporal pass 이후 더 이상 필요 없는 intermediate와 같은 heap 영역을 재사용하는 것이 중요하다.

Moments를 매 scale 유지하지 않고 variance만 ping-pong하는 방식도 가능하다. 그러나 first/second moment를 다시 계산할 수 없고, mixture variance와 estimator variance를 분리하기 어려워진다. 반대로 moments를 유지하면 flexibility는 커지지만 write bandwidth가 증가한다.

### 4.9 FP16 precision과 numerical stability

`m2 = E[Y²]`는 밝은 HDR value에서 빠르게 커진다. FP16에 raw radiance squared를 저장하면 overflow 또는 coarse quantization이 발생할 수 있다.

안정화 방법은 다음과 같다.

- pre-exposed radiance domain에서 moment 계산
- luminance clamp 또는 firefly suppression 이후 moment 계산
- `m1`, `m2`를 FP32 accumulator로 계산하고 FP16 texture에 저장
- variance 계산 시 `max(m2 - m1², 0)` 적용
- catastrophic cancellation을 줄이기 위한 centered moment 또는 compensated representation
- 매우 밝은 signal은 log-luminance moment로 별도 관리

`m2 - m1²`는 두 큰 수의 차이이므로 low variance 영역에서 cancellation이 발생한다. Shader 내부 계산은 FP32가 안전하며, storage만 FP16으로 압축하는 구조가 일반적으로 더 안정적이다.

### 4.10 C++ / shader resource contract

C++ render graph 측에서는 각 resource의 의미를 명확히 고정해야 한다.

- moments가 raw moment인지 centered moment인지
- exposure 적용 전인지 후인지
- variance가 local mixture인지 estimator uncertainty인지
- history-repaired pixel에 inflation이 적용되었는지
- current scale의 stride와 correlation factor가 무엇인지
- signal이 demodulated diffuse인지 raw radiance인지

이 계약이 불명확하면 shader pass는 같은 `R16F variance`를 서로 다른 의미로 해석한다. Denoiser는 수식보다 resource semantics가 더 자주 깨지는 시스템이다.

Debug view도 단순 variance grayscale만으로는 부족하다. 다음 항목을 함께 보면 failure mode를 구분하기 쉽다.

- `log2(variance)` heatmap
- `N_eff`
- weight sum과 maximum tap weight
- history length/confidence
- residual variance / propagated variance ratio
- selected filter scale
- surface identity rejection mask

## 5. 내 관심 분야와 연결

### 실시간 렌더링과 GPU denoising

Ray-traced reflection, diffuse GI, soft shadow는 낮은 sample count에서 temporal/spatial denoising에 의존한다. À-trous filter 자체보다 어려운 부분은 각 iteration이 다음 iteration에 전달하는 confidence와 variance 의미를 유지하는 것이다. 이 구조를 이해하면 SVGF 계열뿐 아니라 NRD, custom temporal denoiser, hybrid raster/ray tracing pipeline의 품질 문제를 더 체계적으로 분석할 수 있다.

### C++ 렌더링 엔진

Render graph에서는 radiance, moments, variance, guide buffer의 lifetime과 aliasing이 중요하다. 한 프레임에 여러 signal을 denoise하면 diffuse/specular/shadow마다 비슷한 transient texture가 반복된다. Resource format, ping-pong 전략, async compute dependency를 설계하는 능력은 그래픽스 엔지니어의 엔진 레벨 역량을 보여준다.

### Compute Shader와 memory bandwidth

25-tap À-trous pass에서 연산량보다 texture fetch와 UAV write가 병목이 되기 쉽다. Radiance와 moments를 한 pass에서 처리하면 guide fetch는 줄지만 register와 write bandwidth가 증가한다. Shared memory tile은 작은 stride에서 유리하지만 stride가 커지면 halo가 커져 효율이 떨어진다. Scale마다 같은 shader 구조를 고집하기보다 early scale과 late scale의 memory access pattern을 구분하는 관점이 필요하다.

### Voxel·level-set·반도체 구조 시각화

Monte Carlo radiance가 아니더라도 noisy scalar field, sparse sampling 결과, surface heatmap을 multi-scale filtering할 때 같은 문제가 나타난다. 예를 들어 doping concentration이나 CFD scalar field를 surface 위에 재구성할 때 local variance는 물리적 gradient와 sampling noise를 모두 포함한다. Estimator uncertainty와 실제 field variation을 분리하지 않으면 sharp junction, material interface, shock-like feature가 smoothing될 수 있다.

Level-set 기반 surface에서는 normal과 material label을 edge guide로 사용하고, topology revision 또는 process step ID를 hard gate로 둘 수 있다. 이때 variance는 단순 영상 통계가 아니라 simulation data confidence와 visualization reconstruction quality를 연결하는 값이 된다.

### WebGPU·Vulkan·DirectX·Metal

모든 modern graphics API에서 storage texture 또는 UAV 기반 ping-pong compute pass로 구성할 수 있다. 차이는 resource binding과 synchronization 방식에 있다.

- Vulkan: descriptor set, image layout transition, pipeline barrier
- Direct3D 12: UAV barrier, descriptor heap, transient resource aliasing
- Metal: texture usage, command encoder boundary, heap aliasing
- WebGPU: storage texture format 제한, bind group layout, explicit pass dependency

API보다 중요한 것은 동일한 statistical resource contract를 유지하는 것이다. `R16F variance`가 어느 pass에서 어떤 의미를 갖는지 정해져 있어야 backend별 구현도 일관된다.

## 6. 머릿속에 남길 질문 3개

1. Filtered moment에서 얻은 local mixture variance와 weighted average의 estimator variance는 어떤 상황에서 크게 달라지는가?
2. À-trous iteration이 진행되며 tap neighborhood가 서로 겹칠 때, covariance를 저장하지 않고 variance collapse를 막을 수 있는 가장 설명 가능한 근사는 무엇인가?
3. Variance가 실제 shading detail과 Monte Carlo noise를 모두 포함할 때, depth·normal·material guide 외에 어떤 confidence signal이 둘을 구분하는 데 도움이 되는가?

## 7. graphics engineer 면접 질문 1개와 답변

**질문:** À-trous denoiser에서 radiance는 점점 부드러워지는데 variance buffer를 단순히 같은 kernel로 평균하면 어떤 문제가 생기며, 어떻게 안정화할 수 있습니까?

**답변:** 단순 평균한 variance는 두 가지 의미를 혼합합니다. First/second moment를 같은 weight로 필터링해 `m2 - m1²`을 계산하면 neighborhood 내부의 기존 uncertainty뿐 아니라 이웃 평균 차이, 즉 실제 edge와 shading variation까지 포함한 mixture variance가 됩니다. 반면 filtered output의 estimator variance는 독립 noise 가정에서 `Σ a_i² σ_i²`로 감소합니다. À-trous scale이 반복되면 neighborhood가 겹쳐 sample correlation이 생기므로 squared-weight 공식만 사용하면 variance를 과소평가할 수 있습니다. 실무에서는 radiance와 moment에 동일한 geometry-aware support를 적용하고, `N_eff = 1/Σa_i²`, scale-dependent correlation inflation, residual variance, minimum floor, history confidence를 함께 사용해 variance가 0으로 붕괴하거나 edge에서 과도하게 팽창하지 않도록 합니다. 또한 variance buffer가 local variation인지 estimator uncertainty인지 resource contract로 명시해야 pass 간 의미가 일관됩니다.

## 8. 포트폴리오 / 커리어 연결

이 개념은 단순히 denoiser shader를 만들었다는 설명보다 한 단계 깊은 포트폴리오 서사를 만든다.

좋은 포트폴리오 설명은 다음 관점을 포함한다.

- temporal moments와 spatial variance가 어떻게 연결되는지
- multi-scale filter에서 variance semantics가 왜 변하는지
- mixture variance와 estimator uncertainty를 어떻게 구분했는지
- `N_eff`, history confidence, residual energy로 어떤 artifact를 제어했는지
- FP16 moment storage와 HDR precision 문제를 어떻게 분석했는지
- radiance/moment ping-pong과 render graph transient memory를 어떻게 설계했는지
- diffuse/specular/scientific scalar signal별 정책이 왜 다른지

면접에서는 결과 이미지보다 failure mode를 설명하는 능력이 강한 신호가 된다.

- 왜 후반 scale에서 highlight가 퍼졌는가?
- 왜 edge 근처 variance가 커졌는가?
- 왜 noise가 남는데 variance는 낮게 보이는가?
- 왜 history-repaired 영역에서 flicker가 발생하는가?
- 왜 FP16 second moment가 특정 exposure에서 깨지는가?

이 질문에 statistical meaning, GPU memory, shader support, render graph contract를 함께 연결해 답하면 graphics engineer로서의 시스템적 사고를 보여줄 수 있다.

## 9. 내일 이어서 볼 개념

**Frequency-Separated Denoising and Detail-Preserving Residual Reconstruction**

오늘은 scale 사이에서 mean과 variance를 안정적으로 전달하는 방법을 다뤘다. 다음에는 저주파 illumination과 고주파 detail/residual을 분리해, 큰 À-trous footprint가 noise를 제거하면서도 texture·normal·specular detail을 잃지 않도록 하는 frequency-separated reconstruction 관점으로 이어간다.

## 10. 참고 키워드

- Spatiotemporal Variance-Guided Filtering, SVGF
- À-Trous Wavelet Filter
- First Raw Moment, Second Raw Moment
- Local Mixture Variance
- Estimator Variance
- Law of Total Variance
- Covariance
- Effective Sample Count, `N_eff`
- Correlation Inflation
- Residual Variance
- Variance Floor
- Edge-Stopping Function
- Temporal Accumulation
- History Confidence
- Demodulation / Remodulation
- HDR Pre-Exposure
- FP16 Moment Precision
- Compute Shader
- Shared Memory / LDS
- Render Graph Resource Aliasing
- Transient Texture
- [Schied et al., Spatiotemporal Variance-Guided Filtering (HPG 2017)](https://doi.org/10.1145/3105762.3105770)
- [NVIDIA Real-time Denoisers (NRD)](https://github.com/NVIDIA-RTX/NRD)
