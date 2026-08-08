---
title: "BRDF-Lobe-Aware Firefly Rejection and Energy-Preserving Specular Clamping"
date: "2026-08-08"
category: Graphics
tags: ["GPU", "Path Tracing", "Specular", "Firefly", "BRDF", "Robust Statistics", "Temporal Denoising", "Compute Shader", "Memory Layout", "Real-Time Rendering"]
level: intermediate
---

# [Daily Graphics Growth] 2026-08-08 - BRDF-Lobe-Aware Firefly Rejection and Energy-Preserving Specular Clamping

## 1. 오늘의 개념

어제의 **Material Demodulation and Roughness-Aware Specular Signal Reconstruction**에서는 material response를 제거한 illumination-like signal을 denoise하고, specular는 roughness와 hit distance에 따라 temporal/spatial support를 다르게 해석해야 한다는 점을 다뤘다.

오늘은 그 흐름에서 바로 나타나는 다음 문제, **Firefly**를 다룬다. Firefly는 path tracing이나 stochastic lighting에서 드물게 발생하는 매우 큰 sample contribution이 화면의 한두 pixel을 비정상적으로 밝게 만드는 현상이다. 특히 low-spp real-time renderer에서는 이 한 번의 outlier가 temporal history에 들어가 여러 frame 동안 번지거나, upscaler가 이를 실제 detail로 해석해 더 크게 강조할 수 있다.

Monte Carlo estimator의 단일 sample contribution을 단순화하면 다음과 같이 볼 수 있다.

`C = L_i · f_r(l, v) · |n·l| / p(l)`

- `L_i`: incident radiance
- `f_r`: BRDF
- `p(l)`: sampled direction의 PDF

Firefly는 대개 **분자는 큰데 PDF가 작은 경우**에 생긴다. 예를 들어 매우 밝은 emissive surface를 낮은 확률로 맞히거나, 좁은 glossy/specular lobe에서 sampling distribution과 실제 integrand가 잘 맞지 않으면 `C`가 순간적으로 매우 커질 수 있다.

전날 다룬 material demodulation도 firefly와 직접 연결된다.

`I_spec = L_spec / max(M_spec, ε)`

여기서 `M_spec`이 작은 영역에서는 noisy radiance와 variance가 함께 증폭될 수 있다. 즉 firefly는 단순히 "밝은 pixel" 문제가 아니라 **sampling estimator, BRDF lobe, material demodulation, temporal accumulation이 만나는 지점에서 발생하는 signal-conditioning 문제**다.

가장 단순한 해결은 absolute luminance clamp다.

`Y' = min(Y, T)`

하지만 이 방식은 세 가지 문제가 있다.

1. 진짜 specular highlight도 잘릴 수 있다.
2. scene exposure나 light intensity에 따라 threshold가 무의미해진다.
3. 잘린 에너지 `ΔY = Y - T`가 사라져 estimator에 bias를 추가한다.

그래서 production renderer에서는 **BRDF-lobe-aware outlier detection**과 **local energy를 가능한 한 보존하는 robust reconstruction**을 함께 생각하는 편이 더 좋다.

오늘의 핵심 구조는 다음과 같다.

1. **Outlier detection**: 현재 sample이 주변/과거의 같은 specular lobe에 비해 얼마나 비정상적인지 판단한다.
2. **Lobe-aware thresholding**: roughness, normal, view direction, hit distance, material identity를 사용해 비교 가능한 sample 집합을 만든다.
3. **Robust clamping/reweighting**: hard clamp 대신 confidence와 robust weight를 사용해 outlier의 영향만 줄인다.
4. **Energy-aware reconstruction**: 제거된 high-energy residual을 버리기보다 compatible support 또는 별도 residual path에서 처리한다.
5. **Temporal protection**: 한 번의 firefly가 history moment와 variance를 오염시키기 전에 차단한다.

여기서 중요한 점은 "energy-preserving"이 곧 Monte Carlo estimator의 strict unbiasedness를 의미하지는 않는다는 것이다. Image-space redistribution이나 temporal residual 처리 역시 위치와 시간 축에서 bias를 만들 수 있다. 실시간 graphics에서의 목표는 **hard clamp보다 에너지 손실을 줄이면서, temporal stability와 perceptual quality를 높이는 것**에 가깝다.

## 2. 한 줄 핵심

**Specular firefly는 밝기 자체가 아니라 같은 BRDF lobe에서 기대되는 분포를 벗어난 sample로 판단해야 하며, hard clamp로 에너지를 삭제하기보다 robust weight와 residual redistribution으로 history 오염을 막는 것이 핵심이다.**

## 3. 왜 중요한가

### 3.1 Firefly 하나가 여러 frame을 오염시킨다

1 spp 또는 그 이하의 stochastic renderer에서는 한 pixel의 outlier가 단 한 frame만 보이고 끝나지 않는다.

Temporal accumulation을 단순화하면

`H_t = (1 - α) H_{t-1} + α X_t`

이다. `X_t`가 정상 sample보다 수십~수백 배 큰 firefly라면 `H_t`가 크게 튀고, 이후 여러 frame 동안 history가 천천히 원래 값으로 돌아온다.

더 나쁜 점은 first/second moment까지 함께 누적하는 경우다.

`m1 = E[X]`

`m2 = E[X²]`

`variance = m2 - m1²`

Firefly는 제곱되는 `m2`를 훨씬 더 강하게 오염시킨다. 그러면 variance-guided filter는 해당 영역을 "매우 noisy"한 것으로 판단해 과도한 blur를 적용하거나, history clamp 범위를 비정상적으로 넓혀 추가 ghosting을 허용할 수 있다.

즉 firefly suppression은 beauty buffer를 깨끗하게 만드는 후처리가 아니라 **temporal statistics를 보호하는 전처리**에 가깝다.

### 3.2 Specular highlight와 firefly는 밝기만 보면 구분하기 어렵다

매우 밝은 pixel이라고 해서 모두 firefly는 아니다.

- 태양이 glossy car paint에 반사된 highlight
- 작은 emissive sign이 mirror reflection에 잡힌 경우
- 금속 표면의 grazing-angle Fresnel peak

이들은 실제로 존재해야 하는 high-energy signal이다.

따라서 `luminance > 10` 같은 global threshold는 physically meaningful하지 않다. 같은 luminance라도 roughness와 reflected footprint에 따라 의미가 완전히 다르다.

낮은 roughness에서는 BRDF lobe가 좁기 때문에 한두 pixel에 매우 높은 peak가 생기는 것이 정상일 수 있다. 반대로 높은 roughness에서는 energy가 넓은 방향으로 퍼지므로 주변과 완전히 동떨어진 single-pixel spike는 outlier일 가능성이 더 높다.

그래서 threshold는 절대 밝기가 아니라 **lobe-conditioned local distribution**을 기준으로 잡는 것이 합리적이다.

### 3.3 Demodulation은 outlier를 더 날카롭게 만들 수 있다

전날처럼 demodulated specular signal을

`I = L / M`

로 정의하면

`Var(I) ≈ Var(L) / M²`

이다. 작은 material factor에서는 variance와 extreme value가 동시에 확대된다.

따라서 demodulation 이후의 firefly suppression은 반드시 다음을 구분해야 한다.

- 실제 incident illumination이 매우 밝은 경우
- material divisor가 작아서 값이 커진 경우
- sampling PDF 또는 lobe probability가 작아서 estimator weight가 커진 경우

이 세 가지는 결과 luminance만 보면 비슷하지만 원인이 다르다. 가능하다면 renderer는 radiance뿐 아니라 `roughness`, `hitT`, lobe identity, sampling confidence 같은 metadata를 denoiser 쪽에 함께 전달하는 편이 좋다.

### 3.4 Upscaler와 post-process는 outlier를 더 눈에 띄게 만든다

Real-time pipeline에서는 denoiser 뒤에 TAA, DLSS/FSR/XeSS, sharpening, bloom, tone mapping이 이어진다.

작은 firefly 하나가

`noisy specular -> denoiser history -> upscaler -> sharpening/bloom`

경로를 통과하면 원본보다 훨씬 넓은 영역에서 flicker 또는 sparkle로 보일 수 있다.

NVIDIA NRD도 REBLUR/RELAX에 anti-firefly option을 제공하며, REBLUR 설정에는 `fireflySuppressorMinRelativeScale`처럼 상대적 outlier suppression 강도를 조절하는 항목이 있다. 이는 production denoiser에서 firefly가 단순한 sampling-side 문제가 아니라 reconstruction-side stability 문제로도 취급된다는 좋은 예다.

### 3.5 Hard clamp는 안정적이지만 energy bias가 크다

Hard clamp의 장점은 명확하다.

- 매우 싸다.
- branchless하게 구현하기 쉽다.
- 최악의 outlier를 확실하게 제한한다.

하지만

`Y' = min(Y, T)`

이면 잘린 양

`ΔY = max(Y - T, 0)`

만큼 총 radiance가 사라진다.

Firefly의 상당수가 실제 light transport에서 나오는 rare event라면, 이 에너지를 무조건 삭제하는 것은 장면을 체계적으로 어둡게 만들 수 있다. 특히 small bright light, caustic-like transport, glossy indirect reflection에서 누적 bias가 눈에 띌 수 있다.

따라서 quality-oriented renderer에서는 **값을 제한하되 excess energy를 어떻게 다룰지**까지 설계해야 한다.

## 4. 구현 관점

### 4.1 Firefly suppression은 temporal accumulation 전에 두는 것이 유리하다

대표적인 흐름은 다음처럼 생각할 수 있다.

`Raw Specular Sample`

`-> Demodulation`

`-> Lobe-Aware Outlier Classification`

`-> Robust Sample / Residual Split`

`-> Temporal Accumulation`

`-> Spatial Reconstruction`

`-> Remodulation`

핵심은 outlier가 main history와 moment buffer에 들어가기 전에 영향도를 제한하는 것이다.

Temporal history를 이미 오염시킨 뒤 clamp하면 현재 frame의 spark는 줄일 수 있어도 `m1`, `m2`, history confidence가 여러 frame 동안 잘못된 상태로 남는다.

### 4.2 Absolute luminance보다 robust local statistic을 사용한다

HDR radiance는 긴 tail을 가지므로 linear luminance에서 mean/stddev를 바로 계산하면 한 개의 firefly가 통계 자체를 끌어올릴 수 있다.

그래서 log luminance를 생각할 수 있다.

`y = log2(1 + luminance(L))`

compatible neighborhood 또는 stable temporal history에서

`μ = E[y]`

`σ² = E[y²] - μ²`

를 구하고, candidate가

`y > μ + kσ`

인지를 본다.

Log domain은 dynamic range를 줄여 FP16 storage와 threshold stability에 유리하다. 또한 exposure가 변해도 linear domain의 fixed threshold보다 상대적으로 안정적이다.

다만 mean/stddev 역시 완전히 robust한 statistic은 아니다. 더 aggressive한 설계에서는 median/MAD, trimmed mean, Huber weight 같은 robust estimator 개념을 사용할 수 있다.

예를 들어 Huber-style weight는 개념적으로

`r = (y - μ) / max(σ, ε)`

`w = 1                         if |r| <= δ`

`w = δ / |r|                   otherwise`

처럼 outlier contribution을 0으로 버리지 않고 영향만 줄인다.

이 방식은 binary reject보다 temporal flicker가 적고, extreme sample이 조금씩은 energy에 기여하도록 만들 수 있다.

### 4.3 BRDF-lobe-aware comparison set

Specular에서 neighborhood statistic을 만들 때 단순 3x3 pixel 평균을 쓰면 서로 다른 reflected surface를 섞기 쉽다.

Candidate `j`가 center `i`와 같은 distribution에 속하는지를 다음 guide로 판단할 수 있다.

`W_ij = W_depth · W_normal · W_roughness · W_hitT · W_material · W_reflection`

- `W_depth`: 같은 primary surface인지
- `W_normal`: normal 차이가 specular lobe에서 허용 가능한지
- `W_roughness`: lobe width가 비슷한지
- `W_hitT`: secondary hit distance가 비슷한지
- `W_material`: material/lobe identity가 같은지
- `W_reflection`: reflected direction 또는 virtual hit가 유사한지

특히 roughness는 threshold와 support radius에 동시에 영향을 준다.

#### 낮은 roughness

- true highlight도 매우 높을 수 있다.
- spatial support가 작다.
- local mean이 안정적이지 않다.
- aggressive luminance clamp는 mirror detail을 죽이기 쉽다.
- 대신 accurate reprojection, hitT consistency, reflected direction consistency를 더 강하게 본다.

#### 높은 roughness

- lobe가 넓어 neighborhood support를 더 확보할 수 있다.
- isolated spike가 distribution에서 벗어났다고 판단하기 쉬워진다.
- robust local statistic을 더 신뢰할 수 있다.

따라서 단순히 `roughness가 낮으면 threshold를 낮춘다` 또는 반대로 `높이면 threshold를 낮춘다` 같은 1차원 rule은 충분하지 않다. **Roughness는 threshold 값보다 "어떤 sample을 비교 대상으로 인정할지"를 결정하는 parameter**라고 보는 편이 정확하다.

### 4.4 Lobe-aware threshold의 개념적 형태

Linear luminance threshold를 다음처럼 생각할 수 있다.

`T = μ_Y + k(α, N·V, C_hit) · σ_Y`

여기서

- `α`: linear roughness
- `N·V`: grazing angle 여부
- `C_hit`: hit-distance/reprojection confidence

낮은 roughness와 grazing angle에서는 legitimate Fresnel peak 가능성이 크므로 `k`를 넓게 두고, 대신 same-lobe validation을 엄격하게 가져가는 편이 자연스럽다.

높은 roughness에서 충분한 compatible neighborhood가 확보되면 `k`를 더 작게 해 isolated spike를 적극적으로 줄일 수 있다.

또한 history confidence가 낮거나 disocclusion 직후라면 temporal statistic을 신뢰하기 어려우므로 threshold를 지나치게 좁게 두면 정상 signal까지 제거할 수 있다.

### 4.5 Hard clamp보다 soft reweighting

Hard clamp:

`L_core = L · min(1, T / max(Y, ε))`

는 chromaticity를 유지하면서 luminance만 제한하는 간단한 형태다.

좀 더 부드럽게는 clamp ratio `q = T / Y`를 바로 쓰지 않고 smooth weight를 사용할 수 있다.

`w = saturate(F(r))`

`L_robust = lerp(L_reference, L, w)`

여기서 `r`은 robust standardized residual이고 `L_reference`는 같은 lobe의 temporal/spatial estimate다.

이 구조의 장점은 outlier를 0으로 버리지 않는다는 것이다. GPU에서도 `step`, `min`, `rcp`, `lerp` 중심으로 branchless하게 구성할 수 있다.

### 4.6 Energy-preserving residual split

Hard clamp에서 잘리는 residual을

`R = L - L_core`

로 분리할 수 있다.

그러면 signal을 두 경로로 나눠 생각할 수 있다.

- `Core`: stable temporal history에 바로 들어가는 bounded signal
- `Residual`: rare high-energy contribution을 별도 confidence/low-frequency path에서 처리하는 signal

최종적으로

`L_final = Core_filtered + Residual_reconstructed`

형태로 합치면, residual을 삭제하는 방식보다 integrated energy를 더 잘 보존할 수 있다.

Residual은 일반 signal보다 훨씬 불안정하므로 같은 history length와 같은 spatial kernel을 쓰는 것이 적절하지 않을 수 있다. 예를 들어 residual은 더 넓은 same-lobe support에 분산하거나, 낮은 temporal weight로 천천히 축적하는 쪽이 안정적일 수 있다.

중요한 점은 이 방식도 strict unbiased estimator는 아니라는 것이다. Pixel 위치와 frame 시점이 바뀌기 때문이다. 하지만 hard clamp처럼 excess energy를 완전히 삭제하지 않으므로 **perceptual stability와 local energy retention 사이의 더 좋은 절충**을 만들 수 있다.

### 4.7 Compatible neighborhood로 excess energy를 재분배하는 관점

한 tile 또는 lobe-consistent neighborhood `N(i)`에서 outlier `i`의 excess energy를

`ΔL_i = L_i - L_core,i`

라고 하자.

호환 가능한 neighbor에 normalized weight `ŵ_ij`로 분배하면

`L'_j = L_j + ŵ_ij · ΔL_i`

`Σ_j ŵ_ij = 1`

이므로 그 local support 안에서는 energy sum을 유지할 수 있다.

그러나 여기에는 trade-off가 있다.

- wrong-surface neighbor로 분배하면 light leaking이 생긴다.
- very sharp reflection에서는 실제 energy가 한 pixel에 집중되어야 할 수 있다.
- moving reflection에서는 frame-to-frame redistribution pattern이 달라져 shimmer가 생길 수 있다.

따라서 low-roughness에서는 redistribution을 최소화하고 temporal/reference 기반 reweighting을 선호하며, rough surface에서만 더 넓은 support를 허용하는 편이 합리적이다.

### 4.8 Sampling-side metadata를 사용할 수 있다면 더 좋다

Firefly를 image-space luminance만으로 판단하면 "왜 이 sample이 큰지"를 알 수 없다.

Renderer가 다음 정보를 제공할 수 있다면 outlier classification이 훨씬 강해진다.

- path throughput magnitude
- BSDF PDF
- light sampling PDF
- MIS weight
- probabilistic lobe selection probability
- path depth
- emissive hit 여부

예를 들어 큰 radiance가 나오더라도 MIS weight와 PDF가 정상이고 stable temporal reference와 일치한다면 실제 highlight일 가능성이 높다.

반대로 `throughput / pdf`가 비정상적으로 큰 rare event라면 firefly confidence를 높일 수 있다.

이 지점은 denoiser와 integrator의 책임 경계를 보여준다. Firefly 문제를 완전히 post-process로 떠넘기기보다, **sample generation 단계에서 estimator quality를 개선하고 reconstruction 단계에서는 남은 outlier만 robust하게 처리하는 구조**가 가장 좋다.

### 4.9 GPU compute shader와 memory layout

대표적인 resource contract를 생각하면 다음과 같다.

**입력**

- `SpecularRadiance` : `RGBA16F` 또는 `RGB16F + hitT`
- `NormalRoughness` : packed normal + roughness
- `ViewZ`
- `MaterialID`
- `MotionVector`
- `HistoryMoments`
- `HistoryConfidence`

**중간/출력**

- `RobustSpecular`
- `ResidualSpecular` 또는 clamp ratio
- updated moments
- optional debug mask

HDR linear `m2`를 FP16에 저장하면 bright sample의 제곱에서 overflow 또는 precision loss가 커질 수 있다. Firefly suppression용 statistic은 `log2(1 + luminance)`를 사용해 `RG16F`에 first/second moment를 저장하는 방식이 practical하다.

Debug가 필요하지 않은 production path에서는 clamp mask를 별도 texture로 만들지 않고, existing confidence channel 또는 temporary register에서만 유지해 bandwidth를 줄일 수 있다.

3x3~5x5 neighborhood statistic을 compute shader로 계산한다면 tile + halo를 shared memory에 올릴 수 있다. 다만 specular validity는 depth/normal/roughness/hitT를 함께 읽으므로 shared-memory footprint와 register pressure가 커질 수 있다. 단순히 radiance만 cache하는 것보다 **guide texture read 수와 occupancy를 함께 프로파일링**해야 한다.

Wave/subgroup reduction은 tile의 log-luminance mean 또는 outlier count를 빠르게 구할 때 유용하지만, surface boundary를 넘는 lane을 무조건 합치면 안 된다. 결국 subgroup optimization에서도 lobe-validity mask가 먼저다.

### 4.10 C++ / render graph 관점

Engine-side에서는 firefly suppression을 독립적인 "미화용 post effect"보다 denoising signal contract의 일부로 보는 것이 좋다.

개념적인 render graph는 다음과 같다.

`TraceSpecular`

`-> MaterialDemodulation`

`-> LobeAwareFireflySuppression`

`-> TemporalReprojection`

`-> Variance/MomentUpdate`

`-> SpatialDenoise`

`-> Remodulation`

C++ resource lifetime 관점에서는 `ResidualSpecular`가 다음 pass까지 필요하지 않다면 transient aliasing 후보가 된다. 반대로 debug visualization과 quality metric을 위해 clamp ratio를 보존하면 bandwidth와 memory cost가 추가된다.

Graphics engineer 입장에서는 "firefly clamp를 넣었다"보다 다음을 설명할 수 있어야 한다.

- suppression이 sampling side인지 reconstruction side인지
- history 오염을 어느 pass에서 차단하는지
- roughness에 따라 compatible support가 왜 달라지는지
- energy loss를 어떻게 측정했는지
- false positive로 실제 highlight가 손실되는지
- 1080p/1440p/4K에서 GPU cost가 어떻게 증가하는지

이런 설명이 구현의 성숙도를 만든다.

## 5. 내 관심 분야와 연결

### Real-Time Rendering / Game Engine

Path tracing, RTGI, reflection denoising에서 firefly는 단순 noise보다 사용자에게 훨씬 강하게 인지된다. 특히 glossy floor, wet road, metallic prop처럼 sharp reflection이 많은 게임 장면에서는 sparse outlier가 shimmer와 sparkle로 보인다.

Engine 관점에서는 integrator의 MIS/sampling quality, denoiser의 robust statistics, upscaler의 temporal behavior까지 연결해서 봐야 한다.

### GPU Compute / Shader Optimization

Lobe-aware filtering은 radiance뿐 아니라 normal, roughness, depth, hitT, material ID를 읽기 때문에 전형적인 bandwidth-heavy compute pass다.

Optimization 포인트는 arithmetic 몇 줄보다

- guide packing
- cache locality
- shared-memory tile 크기
- wave/subgroup mask
- transient texture aliasing
- debug resource 제거

같은 memory-system 설계에 있다.

### CFD / Scientific Visualization

Firefly라는 용어는 path tracing에서 주로 쓰이지만, 개념적으로는 sparse outlier가 temporal/spatial reconstruction을 오염시키는 문제다. CFD scalar field, particle visualization, Monte Carlo uncertainty visualization에서도 extreme sample을 단순 삭제하면 실제 rare event를 잃을 수 있다.

따라서 "outlier suppression과 feature preservation을 어떻게 구분할 것인가"라는 질문은 scientific visualization에서도 그대로 적용된다.

### Semiconductor 3D Visualization

Doping heatmap이나 sparse volumetric signal에서도 작은 영역의 high-value concentration이 실제 feature인지 numerical artifact인지 구분해야 한다. 특히 log-scale colormap을 쓰는 경우 robust statistic과 dynamic-range-aware filtering 사고방식이 직접 연결된다.

### Graphics API / Engine Architecture

D3D12, Vulkan, WebGPU 모두 이 문제 자체를 API가 해결해주지는 않는다. 핵심은 compute pass resource binding과 synchronization이다.

Firefly suppression이 temporal history를 수정하는 pass라면 UAV/barrier contract와 read/write ownership을 명확히 해야 한다. Render graph에서 noisy input, stable history, moment buffer, residual buffer의 lifetime을 잘 정의하는 것이 중요하다.

## 6. 머릿속에 남길 질문 3개

1. **같은 luminance peak라도 low-roughness mirror highlight와 high-roughness firefly를 구분하려면 roughness 외에 어떤 metadata가 가장 강력한가?**
2. **Hard clamp로 잃는 energy와 residual redistribution으로 생기는 spatial/temporal bias 중 실제 게임 화면에서는 어느 쪽이 더 눈에 띄는가?**
3. **Firefly suppression을 integrator의 sample-weight 단계, denoiser 입력 단계, temporal history 단계 중 어디에 두는 것이 가장 안정적인가?**

## 7. graphics engineer 면접 질문 1개와 답변

**Q. Path-traced specular signal에서 firefly를 줄이기 위해 radiance를 일정한 최대값으로 clamp하면 왜 충분하지 않으며, 더 나은 설계는 무엇인가?**

**A.** Fixed radiance clamp는 구현이 싸고 extreme value를 빠르게 제한하지만 세 가지 문제가 있다. 첫째, scene exposure와 light intensity에 의존하므로 threshold가 portable하지 않다. 둘째, low-roughness reflection이나 grazing Fresnel처럼 실제로 존재해야 하는 bright highlight를 firefly로 잘못 제거할 수 있다. 셋째, clamp된 excess energy를 버리기 때문에 estimator가 체계적으로 어두워지는 bias가 생긴다.

더 나은 방식은 center pixel과 비교 가능한 **BRDF-lobe-consistent neighborhood/history**를 만들고, depth·normal·roughness·hit distance·material identity를 사용해 local reference distribution을 추정하는 것이다. 그 뒤 log luminance 기반 robust statistic이나 Huber-style weight로 outlier의 영향만 줄인다. 필요하다면 bounded core와 high-energy residual을 분리해 residual을 별도 temporal/spatial path에서 복원함으로써 hard clamp보다 energy loss를 줄일 수 있다. 중요한 것은 이 suppression을 temporal moment accumulation 전에 적용해 한 번의 outlier가 history와 variance를 오염시키지 않도록 하는 것이다.

면접에서는 여기에 "strict unbiasedness와 real-time perceptual stability는 별개의 목표"라는 점까지 설명하면 좋다. Image-space redistribution은 local energy를 보존할 수 있어도 Monte Carlo 의미에서 완전히 unbiased한 것은 아니다.

## 8. 포트폴리오 / 커리어 연결

Firefly suppression은 작은 기능처럼 보이지만 graphics portfolio에서 매우 좋은 주제가 될 수 있다. 이유는 **sampling theory, BRDF, temporal reconstruction, robust statistics, GPU optimization을 한 번에 보여줄 수 있기 때문**이다.

좋은 포트폴리오 설명은 단순히 "firefly를 제거했다"가 아니라 다음 흐름을 가진다.

- Firefly 발생 원인을 `f · Li · cos / pdf` 관점에서 설명
- fixed clamp가 실제 highlight와 energy를 손상시키는 문제 제시
- roughness/hitT/normal 기반 same-lobe support 정의
- robust weight 또는 residual split 설계
- temporal moment contamination 감소 확인
- reference image 대비 energy error 측정
- clamp ratio / residual heatmap / history length debug view 제공
- GPU pass 비용과 memory bandwidth 분석

비교 지표도 FPS 하나보다 다양하게 잡는 편이 좋다.

- isolated outlier count
- temporal luminance variance
- reference 대비 mean luminance bias
- highlight preservation rate
- ghosting recovery time
- compute dispatch time
- transient memory usage

Nintendo, Unity, engine/rendering 팀 면접에서는 이런 주제가 특히 유리하다. 단순 shader effect가 아니라 **rendering equation에서 출발해 production temporal pipeline의 안정성까지 연결하는 엔지니어링 사고**를 보여주기 때문이다.

또한 C++ engine layer에서 pass ordering, resource lifetime, debug visualization, GPU profiling까지 함께 설명하면 shader programmer를 넘어 renderer/engine engineer 관점의 역량을 보여줄 수 있다.

## 9. 내일 이어서 볼 개념

**Reservoir Weight Stabilization and Boiling Suppression in ReSTIR**

오늘은 이미 생성된 specular sample에서 outlier의 영향을 줄이는 reconstruction-side 문제를 봤다. 내일은 한 단계 upstream으로 이동해 **ReSTIR / reservoir sampling에서 큰 reservoir weight가 왜 temporal boiling을 만들고, weight normalization·spatial reuse·visibility 변화가 어떻게 안정성과 bias에 영향을 주는지**를 본다.

이 흐름을 이어가면

`BRDF sampling`

`-> rare high-weight sample`

`-> reservoir reuse`

`-> firefly / boiling`

`-> temporal denoising`

까지 sampling과 reconstruction을 하나의 pipeline으로 연결해서 볼 수 있다.

## 10. 참고 키워드

- Firefly / Hot Pixel / Outlier Suppression
- Monte Carlo Estimator
- `f_r · L_i · cosθ / PDF`
- Multiple Importance Sampling (MIS)
- Path Throughput
- Specular BRDF / Microfacet BRDF
- GGX / Trowbridge-Reitz
- Roughness / Specular Lobe Width
- Hit Distance (`hitT`)
- Grazing-Angle Fresnel
- Material Demodulation / Remodulation
- Robust Statistics
- Log Luminance
- Median Absolute Deviation (MAD)
- Huber Loss / Huber Weight
- Winsorization / Soft Clamping
- Temporal Moment Contamination
- First / Second Moment
- Variance Inflation
- History Confidence
- Energy-Preserving Residual
- Residual Redistribution
- Same-Surface / Same-Lobe Neighborhood
- REBLUR / RELAX
- NVIDIA NRD `enableAntiFirefly`
- NVIDIA NRD `fireflySuppressorMinRelativeScale`
- Reservoir Sampling / ReSTIR
- Boiling Artifact
- Compute Shader
- Shared Memory / Tile + Halo
- Wave / Subgroup Reduction
- Transient Resource Aliasing
- D3D12 / Vulkan / WebGPU Render Graph
- PBRT v4, *Film and Imaging* - Fireflies and sample clamping: https://pbr-book.org/4ed/Cameras_and_Film/Film_and_Imaging
- NVIDIA NRD README: https://github.com/NVIDIA-RTX/NRD
- NVIDIA NRD settings (`NRDSettings.h`): https://github.com/NVIDIA-RTX/NRD/blob/master/Include/NRDSettings.h
