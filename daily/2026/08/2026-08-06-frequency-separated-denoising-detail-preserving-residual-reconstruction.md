---
title: "Frequency-Separated Denoising and Detail-Preserving Residual Reconstruction"
date: "2026-08-06"
category: Graphics
tags: ["GPU", "Denoising", "Frequency Separation", "Residual Reconstruction", "Compute Shader", "Memory Layout", "Ray Tracing", "SVGF", "NRD", "Scientific Visualization"]
level: intermediate
---

# [Daily Graphics Growth] 2026-08-06 - Frequency-Separated Denoising and Detail-Preserving Residual Reconstruction

## 1. 오늘의 개념

전날의 **Variance Stabilization Across À-Trous Scales and Filtered Moment Propagation**에서는 필터 scale이 커질수록 radiance와 variance가 같은 의미를 유지하도록 만드는 문제를 다뤘다. 오늘은 그 다음 단계로, 큰 spatial support가 필요한 저주파 noise 제거와 보존해야 하는 고주파 detail을 하나의 필터에 모두 맡기지 않는 **주파수 분리 기반 디노이징(Frequency-Separated Denoising)**을 살펴본다.

핵심 아이디어는 noisy signal `S`를 저주파 성분인 **base**와 고주파 성분인 **residual/detail**로 분해한 뒤 서로 다른 안정화 정책을 적용하는 것이다.

`S = B + R`

- `B`: broad illumination, smooth indirect lighting, low-frequency reflection energy처럼 큰 footprint로 안정화해도 되는 성분
- `R`: texture-driven contrast, normal-map detail, thin highlight, contact variation처럼 blur에 취약한 고주파 성분

Edge-aware low-pass operator를 `F_g`라고 하면 다음과 같이 표현할 수 있다.

`B = F_g(S)`

`R = S - B`

여기서 `g`는 depth, normal, roughness, material ID, motion validity 같은 guide이다. 최종 reconstruction은 단순히 `B + R`을 되돌리는 것이 아니라, 각 branch의 신뢰도와 noise 수준을 반영해 다음처럼 구성된다.

`S_out = D_low(B) + c_R · D_high(R)`

- `D_low`: 넓은 support를 사용하는 low-frequency denoising
- `D_high`: 작은 support, temporal stabilization, robust clamping을 사용하는 residual 처리
- `c_R`: residual confidence 또는 detail gain

주파수 분리는 완전한 Fourier decomposition이 아니다. 실시간 rendering pipeline에서는 screen-space, surface-aware filter가 만든 **operational frequency split**에 가깝다. 즉 어떤 성분이 low/high frequency인지 절대적인 주파수만으로 정하는 것이 아니라 surface continuity와 material response를 함께 고려한다.

이 관점은 다음 두 decomposition과 연결된다.

### Additive residual decomposition

`S = B + R`

- signed residual을 명시적으로 보존한다.
- low-frequency branch와 high-frequency branch를 독립적으로 제어하기 쉽다.
- base filter가 바뀌면 residual의 의미도 함께 바뀐다.

### Multiplicative material decomposition

`S = M · L`

`L = S / max(M, ε)`

- `M`: albedo, BRDF response, visibility-independent material factor
- `L`: demodulated illumination 또는 irradiance-like signal
- material detail을 제거한 뒤 lighting만 denoise하고 마지막에 remodulation한다.

Additive split은 image-space frequency 관점이고, multiplicative split은 light transport factorization 관점이다. 실무 denoiser는 두 방식을 함께 사용할 수 있다. 예를 들어 diffuse radiance를 albedo로 demodulate한 뒤, demodulated illumination 내부에서 다시 low-frequency base와 residual을 분리할 수 있다.

## 2. 한 줄 핵심

**저주파 illumination은 넓게 필터링하고 고주파 residual은 작고 신중하게 안정화해야, noise 제거와 texture·normal·specular detail 보존을 동시에 달성할 수 있다.**

## 3. 왜 중요한가

하나의 À-trous filter가 모든 frequency band를 처리하면 filter radius가 커질수록 다음 trade-off가 발생한다.

- radius가 작으면 broad low-frequency noise와 disocclusion noise가 남는다.
- radius가 크면 thin highlight, texture contrast, contact shadow, small geometric feature가 사라진다.
- edge-stopping weight를 강하게 하면 detail은 보존되지만 low-frequency noise가 support를 얻지 못한다.
- edge-stopping weight를 약하게 하면 noise는 줄지만 cross-surface leakage와 over-blur가 증가한다.

Variance-guided filtering만으로 이 문제를 완전히 해결하기 어려운 이유는 variance가 **noise와 실제 detail을 모두 high-frequency energy로 관측**하기 때문이다. 정상적인 normal-map 변화, glossy highlight, 얇은 emissive feature는 local variance를 크게 만들 수 있다. 필터 입장에서는 이것이 Monte Carlo noise인지 중요한 signal detail인지 즉시 구분하기 어렵다.

Frequency separation은 문제를 다음 두 질문으로 나눈다.

1. 넓은 spatial support를 가져도 되는 energy는 무엇인가?
2. detail로 간주해 제한된 support만 허용해야 하는 energy는 무엇인가?

이 분리는 특히 low-spp ray tracing에서 중요하다. Diffuse indirect lighting은 대체로 smooth하지만 albedo와 normal detail에 의해 최종 shaded color는 고주파가 될 수 있다. 최종 color 자체를 크게 blur하면 material detail까지 손상된다. 반면 lighting을 material response와 분리해 denoise하면 훨씬 공격적인 low-frequency reconstruction이 가능하다.

Reflection과 specular signal에서는 문제가 더 복잡하다.

- 낮은 roughness에서는 매우 얇고 높은 에너지의 highlight가 나타난다.
- 높은 roughness에서는 같은 반사 에너지가 넓은 lobe로 퍼진다.
- 한 pixel의 specular detail scale은 screen-space frequency뿐 아니라 BRDF roughness와 reflected hit distance에 의해 결정된다.
- specular residual을 diffuse residual과 같은 기준으로 처리하면 firefly 유지 또는 highlight collapse가 발생할 수 있다.

따라서 production denoiser에서 frequency separation은 단순한 sharpen pass가 아니라 **signal class, material parameter, temporal confidence를 포함한 reconstruction architecture**다.

또한 residual branch는 detail을 되살리는 동시에 noise를 다시 주입할 위험이 있다. `R = S - B`에는 실제 detail뿐 아니라 base filter가 제거하지 못한 noise도 포함된다. Residual을 그대로 합치면 low-frequency denoising 효과가 상쇄된다. 그래서 residual은 다음 정보와 함께 해석해야 한다.

- temporal consistency
- local variance
- history length
- surface compatibility
- material roughness
- luminance outlier 여부
- base/residual energy ratio

결국 frequency separation의 목적은 모든 high-frequency를 보존하는 것이 아니라, **시간적으로 반복되고 surface와 material에 의해 설명 가능한 high-frequency만 detail로 인정하는 것**이다.

## 4. 구현 관점

### 4.1 Decomposition operator의 조건

Base extraction filter `F_g`는 다음 조건을 만족해야 한다.

1. **Surface awareness**  
   Depth, normal, material boundary를 넘어가며 base가 섞이지 않아야 한다.

2. **Temporal consistency**  
   frame마다 base support가 크게 바뀌면 residual이 반대 방향으로 흔들리며 flicker가 발생한다.

3. **Energy stability**  
   base와 residual을 다시 합쳤을 때 전체 radiance energy가 불필요하게 증가하거나 감소하지 않아야 한다.

4. **Scale predictability**  
   어떤 detail scale이 base로 이동하고 어떤 scale이 residual에 남는지 roughness와 resolution 변화에 대해 예측 가능해야 한다.

가장 단순한 형태는 guide-aware blur다.

`B_p = (Σ_q w_pq S_q) / (Σ_q w_pq)`

`w_pq = w_kernel · w_depth · w_normal · w_material · w_roughness`

`R_p = S_p - B_p`

여기서 base extraction과 low-frequency denoising을 같은 pass로 생각하면 안 된다. Base extraction은 frequency band를 나누는 역할이고, low-frequency denoising은 그 band의 noise를 줄이는 역할이다. 같은 kernel을 사용할 수도 있지만 resource 의미와 variance propagation은 분리해 정의하는 편이 안전하다.

### 4.2 Residual은 signed signal이다

Residual은 양수와 음수를 모두 가진다.

`R = S - B`

따라서 unsigned 또는 clamped color buffer에 저장하면 dark-side detail이 손실된다. 예를 들어 highlight 주변의 local contrast는 center의 positive residual과 주변의 negative residual이 함께 있어야 에너지가 보존된다.

Residual storage에서 고려할 점은 다음과 같다.

- `RGBA16F`: signed HDR residual 저장에 일반적으로 적합하다.
- `R11G11B10_FLOAT`: 음수를 표현할 수 없어 residual buffer에는 부적합하다.
- `RGB9E5`: signed residual과 채널 독립 outlier 표현에 부적합하다.
- log encoding: 넓은 dynamic range를 줄일 수 있지만 zero와 sign 처리 규칙이 필요하다.
- luma/chroma separation: luminance residual은 강하게 관리하고 chroma residual은 더 작은 bandwidth로 처리할 수 있다.

FP16 residual은 대부분의 moderate-range signal에 충분하지만, pre-exposure 이전의 매우 밝은 emissive 또는 specular firefly에는 overflow와 quantization 문제가 생길 수 있다. 그래서 temporal/rendering pipeline 전체가 pre-exposed radiance를 사용하는지 resource contract를 명확히 해야 한다.

### 4.3 Low-frequency branch

Low-frequency base는 넓은 filter support를 사용할 수 있다.

`B_out = D_low(B, σ_B², guides)`

이 branch는 다음 특성을 가진다.

- 큰 À-trous stride
- 강한 temporal accumulation
- broad disocclusion repair
- coarse MIP 또는 hierarchical history 활용
- 상대적으로 느린 response
- lower-frequency variance 중심의 filter strength

하지만 base가 surface boundary를 넘으면 residual이 경계를 보상하려고 큰 signed value를 갖게 된다. 이후 residual confidence가 낮아지는 순간 그 보상이 사라져 halo가 보인다. 따라서 base branch의 cross-surface leakage는 residual branch가 해결할 수 있는 문제가 아니다.

Low-frequency branch의 variance는 이전 노트에서 다룬 estimator variance와 residual energy를 함께 사용할 수 있다.

`σ_B,next² = max(σ_prop², β · σ_local², σ_floor²)`

여기서 frequency split 자체가 local variation을 줄이므로 base variance와 original signal variance를 동일하게 해석하면 안 된다. Base extraction 이후의 variance는 base band의 uncertainty를 나타내야 한다.

### 4.4 High-frequency residual branch

Residual branch는 detail preservation을 담당하지만, aggressive filtering을 피해야 한다.

`R_out = D_high(R, σ_R², confidence, guides)`

대표적인 정책은 다음과 같다.

- 작은 spatial radius
- 짧은 temporal history
- stronger outlier clamp
- material/roughness-aware confidence
- center-dominant weight
- high motion과 disocclusion에서 residual attenuation
- residual magnitude에 따른 nonlinear gain

Residual confidence `c_R`는 다음과 같은 곱 형태로 구성할 수 있다.

`c_R = c_temporal · c_surface · c_material · c_variance · c_motion`

최종 reconstruction은 다음처럼 표현된다.

`S_out = B_out + c_R · R_out`

`c_R`이 0이면 detail이 모두 사라지는 문제가 있으므로, production pipeline에서는 완전 제거보다 frequency-dependent attenuation을 사용하는 경우가 많다.

`c_R' = c_min + (1 - c_min) · c_R`

또는 residual magnitude를 soft compression할 수 있다.

`R_comp = R / (1 + |R| / k)`

이 방식은 firefly와 unstable residual을 줄이면서 작은 detail은 상대적으로 유지한다. 다만 nonlinear compression은 에너지 bias를 만들므로 exposure와 tone mapping 전후 중 어느 공간에서 적용하는지 일관되어야 한다.

### 4.5 Residual variance의 의미

Residual variance는 original variance에서 base variance를 단순히 빼면 얻을 수 없다.

`R = S - B`

따라서

`Var(R) = Var(S) + Var(B) - 2Cov(S, B)`

Base는 `S`에서 만들어졌기 때문에 두 signal은 강하게 correlated되어 있다. `Var(S) - Var(B)` 같은 단순 계산은 음수 또는 과소평가를 만들 수 있다.

실시간에서는 exact covariance를 저장하기 어렵기 때문에 다음 근사를 고려한다.

- temporal residual moment를 별도로 추적
- residual absolute deviation 추적
- base filter response와 original response의 차이를 local energy로 사용
- residual history confidence가 낮을 때 variance floor 적용
- scale별 residual gain에 맞춰 variance도 `gain²`으로 조정

Residual gain을 `g`라고 하면 variance도 다음처럼 변한다.

`Var(gR) = g² Var(R)`

Color는 `g`로 줄였는데 variance를 그대로 유지하면 후속 pass가 residual을 실제보다 noisy하게 본다. 반대로 color만 sharpen하고 variance를 갱신하지 않으면 후속 temporal clamp가 지나치게 공격적으로 동작할 수 있다.

### 4.6 Additive split과 material demodulation

Diffuse shading을 단순화하면 다음과 같이 쓸 수 있다.

`C_diffuse ≈ A · E`

- `A`: albedo
- `E`: irradiance-like lighting

최종 shaded color `C_diffuse`를 직접 blur하면 albedo texture가 함께 흐려진다. Demodulation은 다음과 같다.

`E_est = C_diffuse / max(A, ε)`

`E_out = D(E_est)`

`C_out = A · E_out`

이 방식은 material high-frequency를 denoiser 입력에서 제거해 lighting을 더 smooth하게 만든다. 그러나 다음 예외가 있다.

- 매우 어두운 albedo에서 division이 noise를 증폭한다.
- textured metallic/specular material은 단순 곱 factorization이 성립하지 않는다.
- normal map은 albedo처럼 단순 remodulation할 수 없다.
- multiple scattering, visibility, BRDF lobe 변화는 material과 lighting이 비선형적으로 결합한다.

따라서 frequency-separated residual은 demodulation 실패 영역을 보완하는 역할을 할 수 있다. 예를 들어 base는 demodulated lighting에서 복원하고, normal-map-driven high-frequency response는 residual branch에서 제한적으로 재결합한다.

### 4.7 Diffuse와 specular의 서로 다른 frequency 기준

Diffuse signal에서는 surface color와 normal variation을 제외한 indirect illumination이 상대적으로 smooth하다. 반면 specular signal의 frequency는 roughness에 따라 크게 달라진다.

Roughness가 낮을수록:

- highlight가 좁다.
- reflected scene detail이 고주파다.
- 작은 motion error가 큰 radiance 변화로 나타난다.
- residual temporal confidence가 낮아지기 쉽다.

Roughness가 높을수록:

- lobe가 넓다.
- low-frequency branch 비중을 높일 수 있다.
- larger spatial footprint가 허용된다.
- hit-distance와 normal guide의 영향이 달라진다.

따라서 specular split scale `s_spec`는 다음 parameter의 함수로 볼 수 있다.

`s_spec = f(roughness, hitDistance, projectedFootprint, motion, variance)`

고정 blur radius 하나로 diffuse/specular를 동시에 분리하면 material마다 detail loss 특성이 달라진다.

### 4.8 GPU compute pass 구성

Render graph 관점의 대표적인 pass 흐름은 다음과 같다.

1. **Signal preparation**
   - pre-exposed radiance
   - guide normalization
   - optional demodulation
   - luminance/moment preparation

2. **Base extraction**
   - edge-aware local low-pass
   - base radiance 출력
   - signed residual 출력
   - base/residual energy metric 출력

3. **Low-frequency denoising**
   - temporal accumulation
   - variance-guided À-trous passes
   - large support reconstruction

4. **Residual stabilization**
   - small-support spatial filter
   - residual temporal validation
   - outlier compression
   - residual confidence 생성

5. **Reconstruction**
   - `base + confidence × residual`
   - optional remodulation
   - output moments/variance 갱신

6. **Post-reconstruction validation**
   - neighborhood clamp
   - exposure consistency
   - invalid pixel fallback

Compute shader에서는 base와 residual을 같은 pass에서 생성하면 original signal fetch를 재사용할 수 있다. 다만 base kernel이 크면 shared memory tile의 halo가 증가한다. `16×16` group에서 radius 2 kernel을 사용하면 `(16+4)×(16+4)` tile이 필요하고, radiance와 guide를 함께 적재하면 shared memory 사용량이 빠르게 늘어난다.

Base extraction과 residual writing은 bandwidth-heavy pass다.

- original radiance read
- depth/normal/material/roughness read
- base write
- residual write
- optional confidence/moment write

따라서 모든 중간 값을 별도 texture로 유지하기보다 liveness에 따라 transient resource aliasing을 고려할 수 있다. 예를 들어 base denoising이 끝난 뒤 raw signal이 더 이상 필요 없다면 해당 allocation을 residual stabilized buffer와 alias할 수 있다.

### 4.9 Memory layout과 resource format

한 가지 예시는 다음과 같다.

| Resource | 의미 | 후보 format |
|---|---|---|
| `SignalInput` | pre-exposed noisy RGB + optional hit distance | `RGBA16F` |
| `BaseSignal` | low-frequency RGB | `RGBA16F` |
| `ResidualSignal` | signed high-frequency RGB | `RGBA16F` |
| `Moments` | first/second luminance moment | `RG16F` 또는 `RG32F` |
| `Confidence` | residual/temporal confidence | `R8_UNORM` 또는 `R16F` |
| `Guides` | packed normal, roughness, material metadata | engine-specific packed formats |

`ResidualSignal`을 별도 full-resolution `RGBA16F`로 유지하면 4K에서 상당한 bandwidth와 memory를 사용한다. 이를 줄이는 방법은 다음과 같다.

- luminance residual + compact chroma
- residual을 base reconstruction 직전에 transient로 생성
- diffuse/specular branch별 필요한 channel만 저장
- confidence를 residual alpha에 packing
- low-amplitude residual을 lower precision으로 quantization
- tile-local residual을 shared memory에서 소비하고 global write 생략

다만 지나친 packing은 shader register pressure와 decode cost를 높인다. 특히 normal/roughness/material metadata를 한 buffer에 과도하게 압축하면 base extraction pass의 ALU cost가 증가하고 wave occupancy가 낮아질 수 있다.

### 4.10 C++ render graph contract

C++ 측에서는 buffer 이름보다 signal 의미를 명확히 하는 것이 중요하다.

예를 들어 다음 항목을 resource contract로 문서화할 수 있다.

- radiance가 pre-exposed인지 absolute HDR인지
- residual이 signed linear RGB인지 log-luma space인지
- base/residual의 합이 input과 정확히 일치하는지
- demodulation에 사용한 material factor
- confidence가 reconstruction gain인지 단순 validity인지
- moments가 input, base, residual 중 어느 signal을 표현하는지
- roughness가 perceptual roughness인지 squared roughness인지
- motion vector와 depth가 current/previous 어느 convention인지

이 contract가 불명확하면 각 pass는 개별적으로 정상이어도 전체 pipeline에서 ghosting, halo, exposure pop, specular detail loss가 발생한다.

Render graph dependency는 다음 흐름을 가져야 한다.

`Prepare → Split → LowDenoise → ResidualStabilize → Reconstruct`

Async compute를 사용할 경우 Split과 geometry/raster pass 사이의 guide resource ownership, UAV barrier, queue synchronization이 필요하다. Low-frequency denoising이 여러 iteration의 ping-pong texture를 사용한다면 transient lifetime과 queue overlap이 실제 memory peak를 줄이는지 확인해야 한다.

### 4.11 Temporal stability

Frequency split이 frame마다 바뀌면 detail이 base와 residual branch 사이를 이동하며 flicker가 발생한다. 이를 **band migration** 문제로 볼 수 있다.

원인은 다음과 같다.

- motion vector 오차
- roughness LOD 변화
- dynamic resolution 변화
- exposure 변화
- base filter support의 discontinuous rejection
- normal-map aliasing
- temporal history reset

안정화를 위해 split parameter와 residual confidence에도 temporal hysteresis를 둘 수 있다.

`c_R,t = lerp(c_R,t-1, c_R,current, α)`

다만 disocclusion에서는 과거 confidence를 유지하면 ghost detail이 남는다. 따라서 hysteresis는 surface validity와 history confidence가 확보된 경우에만 적용되어야 한다.

Dynamic resolution에서는 split radius가 pixel 단위인지 normalized screen-space 단위인지 결정해야 한다. 같은 world-space detail이 resolution 변화에 따라 base와 residual 사이를 이동하면 reconstruction character가 달라진다. Projected footprint와 roughness를 활용한 scale 정의가 더 안정적이다.

### 4.12 실패 모드

Frequency-separated denoising의 대표적인 실패 모드는 다음과 같다.

#### Halo

Base가 surface boundary를 넘어가고 residual이 이를 부분적으로 보상할 때 생긴다. Residual confidence가 낮아지면 보상이 사라져 밝거나 어두운 halo가 남는다.

#### Noise reinjection

Residual branch가 실제 detail과 Monte Carlo noise를 구분하지 못하고 noise를 다시 합친다.

#### Detail pumping

Residual confidence가 frame마다 변하며 texture와 highlight contrast가 숨 쉬듯 커졌다 작아진다.

#### Energy drift

Nonlinear residual compression, clamp, remodulation이 누적되어 평균 radiance가 원본과 달라진다.

#### Chromatic ringing

RGB channel별 residual scale이 달라 color edge 주변에 반대색 ring이 생긴다.

#### Roughness transition seam

Material roughness threshold에 따라 split policy가 급변하며 specular lobe 경계에서 seam이 나타난다.

#### Over-sharpening

Residual gain이 1보다 크거나 tone mapping 이후 detail enhancement를 중복 적용할 때 noise와 ringing이 강조된다.

Debug view는 최소한 다음을 분리해서 볼 수 있어야 한다.

- base
- raw residual
- stabilized residual
- residual confidence
- reconstructed output
- base/residual energy ratio
- split scale
- surface rejection mask

## 5. 내 관심 분야와 연결

### 실시간 ray tracing과 game rendering

Ray-traced diffuse GI, glossy reflection, shadow signal은 서로 frequency 특성이 다르다. Frequency-separated architecture는 모든 signal을 하나의 generic filter로 처리하는 대신 signal별 reconstruction contract를 설계하는 기반이 된다.

특히 game engine에서는 다음과 연결된다.

- material albedo를 보존하는 indirect lighting denoising
- roughness-aware reflection reconstruction
- normal map detail 유지
- RTXDI direct illumination의 high-frequency visibility 보존
- temporal upscaling 이전의 stable signal preparation
- dynamic resolution에서 detail scale 유지

### GPU compute와 shader optimization

이 구조는 단순한 image filter가 아니라 여러 compute pass와 transient resource로 구성된 pipeline이다. Graphics engineer 관점에서는 다음 능력을 보여준다.

- guide-aware compute kernel 설계
- shared memory와 global bandwidth trade-off
- signed HDR residual format 선택
- wave/subgroup reduction 활용
- pass fusion과 register pressure 비교
- render graph aliasing과 lifetime 관리
- async compute synchronization
- dynamic resolution과 history resource 대응

### CFD·Scientific Visualization

CFD scalar/vector field도 coarse structure와 fluctuation을 분리해 해석할 수 있다.

`field = mean/base + fluctuation/residual`

예를 들어 velocity magnitude, pressure, vorticity visualization에서 큰 흐름 구조는 low-frequency base에 해당하고, boundary layer와 vortex detail은 residual에 해당한다. 다만 scientific visualization에서는 residual이 단순 noise가 아니라 물리적으로 중요한 fluctuation일 수 있으므로, denoising보다 **uncertainty-aware multiscale visualization** 관점이 중요하다.

Monte Carlo volume rendering이나 sparse volume path tracing에서는 surface guide가 없거나 불완전하다. 이 경우 volume transmittance, density gradient, depth interval, layer identity를 guide로 사용한 frequency separation이 필요하다.

### Level-set·Voxel·Marching Cubes

Level-set field `φ`는 smooth signed distance-like base와 grid discretization 또는 topology detail로 나누어 생각할 수 있다. 하지만 geometry reconstruction에서 high-frequency residual을 무조건 제거하면 thin feature와 topology가 사라질 수 있다.

Rendering pipeline에서는 다음과 같은 연결이 가능하다.

- coarse distance field 기반 smooth shading
- high-frequency normal/detail reconstruction
- voxel LOD 경계의 residual 보정
- Marching Cubes mesh normal의 low/high frequency 분리
- temporal geometry revision에 따른 residual invalidation

### 반도체 공정 시각화

반도체 공정 결과의 doping concentration, material interface, etch/deposition surface는 smooth bulk field와 sharp interface가 함께 존재한다.

- bulk concentration gradient: low-frequency base
- junction boundary와 thin layer: high-frequency residual
- process step change: temporal history reset 또는 topology revision
- material ID boundary: 절대 넘어가면 안 되는 guide
- log-scaled concentration: decomposition space 선택이 중요

특히 log concentration heatmap에서는 linear-space residual과 log-space residual의 의미가 다르다. 물리량 보존이 중요한 분석 화면과 perceptual clarity가 중요한 presentation 화면은 서로 다른 split/reconstruction 정책을 가져야 한다.

## 6. 머릿속에 남길 질문 3개

1. **Residual이 실제 surface/material detail인지 Monte Carlo noise인지 판별하려면 temporal consistency, variance, roughness 중 어떤 신호를 가장 우선해야 하는가?**
2. **Additive base-residual split과 albedo/BRDF demodulation은 어떤 조건에서 서로 보완적이며, 어떤 조건에서는 같은 detail을 두 번 보존해 over-sharpening을 만들 수 있는가?**
3. **4K 실시간 pipeline에서 full-resolution signed residual buffer의 bandwidth를 줄이면서 chromatic detail과 HDR energy를 유지하려면 어떤 memory layout이 적절한가?**

## 7. graphics engineer 면접 질문 1개와 답변

### 질문

**저주파 noise를 제거하기 위해 filter radius를 키웠더니 texture와 specular highlight가 흐려졌다. Frequency-separated denoising pipeline을 설계한다면 signal decomposition, temporal 처리, GPU resource 관점에서 어떻게 설명하겠는가?**

### 답변

먼저 최종 shaded signal을 하나의 frequency distribution으로 취급하지 않고, 넓은 support를 허용할 수 있는 low-frequency base와 blur에 취약한 signed residual로 분리한다.

`S = B + R`

Base는 depth, normal, material ID, roughness를 사용하는 edge-aware filter로 추출하고, broad illumination과 smooth reflection energy를 포함하도록 한다. Base branch는 temporal accumulation과 variance-guided À-trous filtering을 사용해 큰 support로 안정화한다.

Residual branch는 실제 detail과 noise가 섞여 있으므로 작은 spatial radius, 짧은 history, outlier clamp, roughness와 motion 기반 confidence를 사용한다. 최종 출력은 다음처럼 구성한다.

`S_out = B_denoised + c_R · R_stabilized`

Diffuse signal에서는 albedo demodulation을 먼저 적용해 material texture가 lighting denoiser에 들어가지 않게 할 수 있다. Specular signal은 roughness와 reflected footprint에 따라 split scale이 달라지므로 diffuse와 같은 정책을 사용하지 않는다.

Temporal 관점에서는 frame마다 base/residual 경계가 이동하는 band migration을 막아야 한다. Split scale과 residual confidence에 hysteresis를 적용하되, disocclusion과 surface invalidation에서는 history를 즉시 줄인다.

GPU 관점에서는 base extraction과 residual 생성의 input fetch를 한 compute pass에서 공유할 수 있지만, 큰 kernel은 shared memory halo와 register pressure를 증가시킨다. `BaseSignal`, signed `ResidualSignal`, moments, confidence의 format과 lifetime을 명확히 하고, render graph transient aliasing으로 memory peak를 줄인다. Residual은 음수를 포함하므로 unsigned packed HDR format을 사용하면 안 된다.

마지막으로 base, raw residual, stabilized residual, confidence, reconstructed output을 각각 debug view로 제공해 halo, noise reinjection, energy drift를 분리해 진단한다.

## 8. 포트폴리오 / 커리어 연결

이 개념을 포트폴리오에서 설명할 때는 “blur를 줄였다”보다 **signal decomposition과 resource architecture를 설계했다**는 흐름이 중요하다.

좋은 설명 구조는 다음과 같다.

### 문제 정의

- low-spp radiance에서 broad noise와 high-frequency detail이 같은 buffer에 존재했다.
- 큰 filter radius는 noise를 줄였지만 material texture와 highlight를 손상시켰다.
- 강한 edge-stopping만으로는 residual noise와 unstable detail을 구분하기 어려웠다.

### 설계 결정

- low-frequency base와 signed residual로 signal을 분리했다.
- diffuse는 material demodulation 가능성을 분리해서 검토했다.
- specular는 roughness와 hit-distance 기반 split policy를 사용했다.
- base와 residual이 서로 다른 temporal history와 variance 의미를 갖도록 했다.
- residual confidence와 energy debug view를 추가했다.

### GPU 최적화

- split pass에서 original signal과 guides fetch를 재사용했다.
- base/residual intermediate의 lifetime을 render graph로 관리했다.
- signed residual에 적합한 FP16 format을 선택했다.
- pass fusion과 shared memory 사용이 occupancy에 미치는 영향을 비교했다.
- 1080p, 1440p, 4K에서 bandwidth와 frame time을 분리해 측정했다.

### 품질 평가

- flat indirect lighting 영역의 residual noise
- textured surface detail retention
- glossy highlight width
- moving camera에서 detail pumping
- disocclusion에서 ghost residual
- roughness transition seam
- HDR emissive 주변 energy drift

이 주제는 면접에서 rendering algorithm, statistical signal processing, shader implementation, GPU memory, render graph를 한 번에 연결할 수 있다. 특히 Nintendo·Unity·game engine graphics role에서 “효과를 구현했다”를 넘어 **품질 실패 모드와 pipeline contract를 설명하는 능력**을 보여주기 좋다.

## 9. 내일 이어서 볼 개념

**Material Demodulation and Roughness-Aware Specular Signal Reconstruction**

오늘은 image-space signal을 low-frequency base와 high-frequency residual로 분리하는 관점을 다뤘다. 다음에는 diffuse albedo, specular BRDF, roughness, hit distance를 이용해 material response와 illumination을 분리하고, diffuse와 specular가 서로 다른 remodulation 및 temporal reconstruction 정책을 가져야 하는 이유를 이어서 본다.

## 10. 참고 키워드

### 핵심 용어

- Frequency-Separated Denoising
- Low-Frequency Base
- High-Frequency Residual
- Detail-Preserving Reconstruction
- Additive Signal Decomposition
- Material Demodulation / Remodulation
- Signed HDR Residual
- Residual Confidence
- Band Migration
- Energy Preservation
- Edge-Aware Low-Pass Filter
- Variance-Guided Filtering
- À-Trous Wavelet Filter
- Temporal Accumulation
- Roughness-Aware Filtering
- Specular Lobe Footprint
- Hit-Distance Reconstruction
- Pre-Exposed Radiance
- Transient Resource Aliasing
- Render Graph Resource Contract

### 연결해서 찾아볼 자료

- **Spatiotemporal Variance-Guided Filtering: Real-time Reconstruction for Path Traced Global Illumination (SVGF)**
- **An Efficient Denoising Algorithm for Global Illumination**
- **Combining Analytic Direct Illumination and Stochastic Shadows**
- **NVIDIA Real-Time Denoisers: ReLAX / ReBLUR**
- **Gradient-Domain Rendering and Residual Reconstruction**
- **Demodulated Illumination Denoising**
- **Diffuse/Specular Signal Separation**
- **Roughness-Adaptive Spatial Filtering**
- **Monte Carlo Radiance Factorization**
