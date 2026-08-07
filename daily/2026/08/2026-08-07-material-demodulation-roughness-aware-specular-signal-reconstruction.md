---
title: "Material Demodulation and Roughness-Aware Specular Signal Reconstruction"
date: "2026-08-07"
category: Graphics
tags: ["GPU", "Denoising", "Material Demodulation", "Specular", "Roughness", "BRDF", "Hit Distance", "Compute Shader", "Memory Layout", "Ray Tracing", "NRD", "Real-Time Rendering"]
level: intermediate
---

# [Daily Graphics Growth] 2026-08-07 - Material Demodulation and Roughness-Aware Specular Signal Reconstruction

## 1. 오늘의 개념

전날의 **Frequency-Separated Denoising and Detail-Preserving Residual Reconstruction**에서는 noisy shaded color를 저주파 base와 고주파 residual로 분리해 서로 다른 filter policy를 적용하는 관점을 다뤘다. 오늘은 그보다 한 단계 더 앞선 분해, 즉 **재질 응답(Material Response)** 과 **조명 신호(Illumination Signal)** 를 분리하는 **Material Demodulation**을 살펴본다.

실시간 path tracing 또는 ray-traced GI의 primary hit에서 관측되는 radiance는 대략 다음처럼 생각할 수 있다.

`L_out = M_diff · I_diff + M_spec · I_spec`

- `M_diff`: diffuse albedo, metalness, geometry term 등 primary material에 의해 결정되는 diffuse material factor
- `M_spec`: Fresnel, roughness, normal/view configuration 등으로 결정되는 specular material factor
- `I_diff`, `I_spec`: stochastic sampling 결과로 얻은 illumination-like signal

최종 shaded radiance를 그대로 denoise하면 texture, normal map, Fresnel 변화, roughness 변화도 모두 noise와 같은 image-space frequency로 보인다. Material demodulation은 다음과 같이 noisy radiance에서 material factor를 제거한다.

`I_diff ≈ L_diff / max(M_diff, ε)`

`I_spec ≈ L_spec / max(M_spec, ε)`

그 후 더 부드럽고 surface-consistent한 illumination space에서 temporal/spatial reconstruction을 수행하고, 마지막 composition에서 다시 material factor를 곱한다.

`L̂_diff = Î_diff · M_diff`

`L̂_spec = Î_spec · M_spec`

하지만 diffuse와 specular는 같은 방식으로 다루면 안 된다. Diffuse material factor는 상대적으로 안정적이지만, specular response는 **roughness, Fresnel, view direction, normal, sampled reflection direction**에 매우 민감하다. 특히 낮은 roughness에서는 아주 작은 방향 변화가 완전히 다른 reflected surface를 가리킬 수 있다.

따라서 오늘의 핵심은 단순한 `color / material`이 아니라 다음 세 가지를 하나의 pipeline으로 보는 것이다.

1. **Demodulation**: 재질 고주파를 illumination에서 분리한다.
2. **Roughness-aware reconstruction**: specular lobe 폭에 맞춰 temporal/spatial support를 바꾼다.
3. **Remodulation**: denoised illumination을 현재 frame의 material response와 정확히 다시 결합한다.

NVIDIA NRD 역시 application side에서 material을 de-modulate한 입력을 기대하며, diffuse와 specular radiance를 primary hit에서 분리하고 roughness 및 hit distance를 denoising guide로 사용한다. 이 구조는 특정 SDK에만 해당하는 구현 규칙이 아니라, modern real-time reconstruction에서 **signal과 shading response의 의미를 분리하는 설계 원리**로 이해할 수 있다.

## 2. 한 줄 핵심

**재질의 고주파 변화는 먼저 제거하고 illumination을 denoise한 뒤 다시 material response를 곱해야 하며, specular는 roughness와 hit distance에 따라 filter footprint와 temporal confidence를 별도로 조절해야 한다.**

## 3. 왜 중요한가

### 최종 shaded color를 직접 denoise하면 material detail이 noise로 보인다

예를 들어 같은 diffuse indirect lighting을 받는 checker texture가 있다고 하자.

`L_diff = albedo · I_diff`

실제 `I_diff`는 공간적으로 거의 일정해도 albedo가 흰색/검은색으로 반복되면 최종 `L_diff`에는 강한 high-frequency edge가 생긴다. Denoiser가 이를 illumination 변화라고 판단하면 두 가지 문제가 생긴다.

- 강한 edge stopping을 사용하면 texture boundary마다 filter support가 끊겨 noise가 남는다.
- 약한 edge stopping을 사용하면 texture 자체가 blur된다.

Demodulation 후에는 albedo variation이 제거되므로 illumination space는 훨씬 smooth해지고, 넓은 spatial support와 긴 temporal history를 사용하기 쉬워진다.

### Specular는 diffuse보다 훨씬 더 view-dependent하다

Microfacet specular BRDF를 단순화하면 다음처럼 표현된다.

`f_s(l, v) = D(h) · G(l, v) · F(v, h) / (4 · NoL · NoV)`

여기서 roughness는 주로 normal distribution function `D`의 폭을 결정한다.

- **낮은 roughness**: 매우 좁은 lobe, mirror-like reflection, 작은 angular error에도 reflected point가 크게 이동
- **높은 roughness**: 넓은 lobe, 여러 방향의 reflected energy가 한 pixel에 평균적으로 기여

따라서 같은 화면 거리의 두 specular pixel이라도 roughness가 다르면 적절한 denoising footprint가 다르다.

낮은 roughness에서 filter radius를 크게 쓰면 reflected image가 옆 surface와 섞이며 reflection detail이 무너진다. 반대로 높은 roughness에서 radius를 지나치게 작게 쓰면 broad glossy reflection에 Monte Carlo noise가 계속 남는다.

### Hit distance는 specular footprint의 공간적 의미를 알려준다

Specular reflection은 primary surface에서 끝나지 않는다. 반사 ray가 얼마나 멀리 이동해 secondary surface를 만났는지, 즉 **specular hit distance (`hitT`)** 는 reprojection과 spatial footprint 해석에 중요한 guide가 된다.

직관적으로 동일한 angular lobe width `θ`에 대해 reflected surface에서 차지하는 world-space footprint는 hit distance가 길수록 커진다.

`footprint_world ∝ hitT · tan(θ)`

small-angle 근사에서는

`footprint_world ≈ hitT · θ`

이고 microfacet roughness가 증가하면 effective `θ` 역시 커진다. 따라서 roughness와 hit distance는 서로 독립적인 guide가 아니라 **specular lobe가 실제 scene에서 얼마나 넓은 영역을 대표하는지**를 함께 설명한다.

### Demodulation은 noise distribution 자체도 바꾼다

`I = L / M`

이라면 material factor `M`이 작을 때 noise도 함께 증폭된다.

`Var(I) ≈ Var(L) / M²`

즉 demodulation은 material detail을 제거해 filtering을 쉽게 만들지만, 작은 divisor 영역에서는 variance와 firefly를 크게 키울 수 있다. 특히 specular factor가 매우 작거나 probabilistic lobe selection의 probability가 작을 때 극단적인 값이 발생할 수 있다.

따라서 production pipeline의 material demodulation은 단순 나눗셈이 아니라 다음 계약을 필요로 한다.

- minimum stable material factor
- valid diffuse/specular lobe 정의
- probabilistic lobe selection의 probability clamp 정책
- HDR/pre-exposure space 통일
- demodulated variance의 스케일 변환
- remodulation에서 동일한 material convention 사용

결국 material demodulation은 **quality optimization**이면서 동시에 **signal representation design**이다.

## 4. 구현 관점

### 4.1 Diffuse와 specular signal을 primary hit에서 분리한다

Denoising 관점에서는 최종 indirect radiance 하나보다 다음처럼 signal class를 명시하는 편이 좋다.

`DiffuseSignal  = (radiance_diff, hitT_diff, variance_diff)`

`SpecularSignal = (radiance_spec, hitT_spec, variance_spec)`

이 분리는 rendering equation의 의미와도 맞고 filter policy를 독립적으로 설정할 수 있게 한다.

Diffuse는 주로 surface-local한 irradiance reconstruction 문제에 가깝고, specular는 reflected scene의 virtual image reconstruction 문제에 더 가깝다.

### 4.2 Diffuse demodulation

Lambertian diffuse만 생각하면 material factor는 대략 다음 형태다.

`M_diff ≈ albedo / π`

실제 PBR pipeline에서는 metalness, layered material, transmission, clearcoat, energy compensation에 따라 factor의 의미가 달라질 수 있다. 중요한 점은 **denoiser 입력과 final composition이 같은 factorization을 공유해야 한다는 것**이다.

Diffuse demodulation은 상대적으로 안정적이다.

- albedo가 primary G-buffer에서 정확히 재생성 가능
- view dependence가 작음
- temporal reprojection 후 current-frame albedo로 remodulation 가능
- texture detail을 denoiser에서 분리하기 쉬움

다만 albedo가 매우 작은 surface에서는 `L / albedo`가 불안정해질 수 있으므로 denominator floor 또는 dark-material-specific confidence가 필요하다.

### 4.3 Specular demodulation은 단순 F0 division이 아니다

Specular material factor를 `F0` 또는 roughness 하나로 표현하면 충분하지 않다. 실제 specular response는 Fresnel, NDF, masking-shadowing, view/normal configuration에 의존한다.

따라서 production denoiser의 `M_spec`은 보통 정확한 BRDF value 자체라기보다 **stable reconstruction factor**에 가깝다.

좋은 specular demodulation factor는 다음 특성을 가져야 한다.

1. material 변화로 생기는 high-frequency shading을 충분히 제거한다.
2. 매우 작은 값으로 떨어져 radiance를 폭발시키지 않는다.
3. current/previous frame에서 temporal consistency가 있다.
4. remodulation할 때 원래 material appearance를 다시 복원할 수 있다.
5. sampling PDF와 lobe selection probability의 역할과 혼동되지 않는다.

특히 `radiance / BRDF / PDF` 형태를 무조건 demodulated signal로 정의하면 여러 importance sampling term이 중첩되어 extreme value가 만들어질 수 있다. **sampling estimator의 normalization과 material demodulation은 서로 다른 층의 문제**다.

### 4.4 Roughness-aware spatial support

Specular spatial kernel radius를 roughness에 따라 바꾸는 가장 단순한 직관은 다음과 같다.

`r_spec = r_min + k · roughness^p`

하지만 screen-space radius는 hit distance와 projection에도 영향을 받는다.

개념적으로는

`r_spec ∝ Project( hitT · lobeWidth(roughness) )`

로 볼 수 있다.

실무에서는 exact BRDF cone projection보다 다음 guide를 조합한 heuristic이 더 흔하다.

- linear roughness
- normalized hit distance
- view-space depth
- normal similarity
- reflected hit consistency
- specular history confidence

낮은 roughness 영역에서는 작은 kernel, 강한 hit-distance/normal rejection, 짧거나 엄격한 temporal validation이 중요하다. 높은 roughness에서는 더 큰 support를 허용할 수 있지만, material boundary를 넘어가면 안 된다.

### 4.5 Roughness transition에서 생기는 seam

Roughness를 threshold로 둘로 나누면

`roughness < 0.2 -> glossy path`

`roughness >= 0.2 -> rough path`

같은 hard branch가 생길 수 있다. 이 경우 roughness animation, normal map 변화, mip transition에서 filter footprint가 frame마다 갑자기 바뀌어 seam 또는 pumping artifact가 발생한다.

따라서 filter strength는 연속 함수로 구성하는 편이 안정적이다.

`w_r = smoothstep(r0, r1, roughness)`

`radius = lerp(radius_glossy, radius_rough, w_r)`

Temporal accumulation length, spatial radius, history clamp width도 같은 roughness transition band를 공유하면 pipeline behavior가 더 예측 가능하다.

### 4.6 Temporal reconstruction과 roughness

낮은 roughness specular는 mirror-like detail 때문에 temporal reprojection 정확도 요구가 높다.

History candidate를 평가할 때 다음 조건이 중요하다.

`C_history = C_surface · C_normal · C_roughness · C_hitT · C_motion`

- `C_surface`: 같은 primary surface인지
- `C_normal`: reflected direction 변화가 허용 범위인지
- `C_roughness`: lobe width가 유사한지
- `C_hitT`: reflected hit depth가 일관적인지
- `C_motion`: motion vector 또는 virtual motion이 맞는지

높은 roughness는 broad lobe 때문에 약간의 geometric mismatch를 더 잘 숨기지만, 낮은 roughness는 작은 reprojection error도 double-image와 ghosting으로 바로 보인다.

이때 roughness가 낮다고 무조건 history length를 줄이는 것이 최선은 아니다. 정확한 virtual motion과 hit distance가 있다면 mirror-like signal도 긴 history를 사용할 수 있다. 핵심은 **roughness가 history rejection threshold와 motion model의 정확도 요구를 결정한다는 것**이다.

### 4.7 Variance propagation

Demodulation 전 radiance variance를 `σ_L²`, material factor를 `M`이라고 하면 이상적인 단순 스케일에서는

`σ_I² = σ_L² / M²`

Remodulation 후에는

`σ_L,out² = M² · σ_I,out²`

이다.

하지만 `M` 자체가 frame마다 변화하거나 normal/roughness가 stochastic하게 변하는 material에서는 material factor의 uncertainty도 존재한다. 실시간 renderer에서는 이를 완전한 random variable로 모델링하기보다는 material change를 history confidence 또는 disocclusion-like rejection으로 처리하는 편이 현실적이다.

즉 variance buffer는 signal noise의 통계이고, material mismatch는 별도의 validity problem으로 유지하는 것이 pipeline을 해석하기 쉽다.

### 4.8 GPU resource layout

대표적인 denoising resource는 다음처럼 구성할 수 있다.

| Resource | 예시 format | 역할 |
|---|---:|---|
| Demodulated Diffuse | `RGBA16F` | diffuse illumination + auxiliary channel |
| Demodulated Specular | `RGBA16F` | specular illumination + auxiliary channel |
| Diffuse/Spec HitT | `RG16F` | 두 lobe의 hit distance packing |
| Normal | `RG16_SNORM` 또는 packed octahedral | edge/reprojection guide |
| Roughness | `R8_UNORM` / packed G-buffer | specular lobe width guide |
| Material ID | `R8_UINT` / bit field | cross-material rejection |
| Variance/Moments | `RG16F` / `RGBA16F` | temporal statistics |
| History Confidence | `R8_UNORM` / packed | history validity |

Memory bandwidth를 줄이기 위해 normal, roughness, material ID를 하나의 G-buffer texture에 pack할 수 있지만, compute shader에서 매 sample마다 decode cost가 발생한다. 반대로 별도 texture는 access pattern이 단순해지지만 descriptor 수와 bandwidth가 늘어난다.

Denoiser가 5×5 또는 À-trous support를 여러 번 읽는다면 G-buffer guide는 높은 reuse를 갖는다. 따라서 tile/shared-memory caching이 효과적일 수 있지만, radiance와 guide를 모두 shared memory에 올리면 occupancy가 빠르게 떨어질 수 있다. 실제 병목은 ALU보다 texture load와 cache locality일 가능성이 높다.

### 4.9 Compute shader 관점

Specular reconstruction compute pass의 논리적 단계는 다음과 같다.

`Load center -> classify roughness -> gather neighbors -> evaluate surface/lobe compatibility -> weighted reconstruction -> update variance/confidence`

Neighbor weight의 개념형은 다음처럼 쓸 수 있다.

`w = w_kernel · w_depth · w_normal · w_roughness · w_hitT · w_material`

여기서 specular에서는 `w_roughness`와 `w_hitT`가 diffuse보다 훨씬 중요한 역할을 한다.

또한 roughness가 낮은 pixel과 높은 pixel이 같은 wave에 섞이면 branch divergence가 생길 수 있다. 하지만 roughness를 discrete path로 과도하게 분리하면 quality seam이 생긴다. 따라서 shader architecture는 **continuous weight 계산 + 일부 coarse tile classification** 정도의 균형이 적절하다.

### 4.10 C++ / Render Graph contract

C++ render graph에서는 demodulation과 denoising 사이의 resource 의미가 명확해야 한다.

예를 들어 resource name만 `SpecularColor`로 두면 다음 ambiguity가 생긴다.

- raw radiance인가?
- demodulated illumination인가?
- pre-exposed 값인가?
- denoised 값인가?
- remodulated 값인가?

실무에서는 resource contract를 의미 중심으로 분리하는 편이 좋다.

- `SpecularRadianceRaw`
- `SpecularIlluminationDemodulated`
- `SpecularIlluminationHistory`
- `SpecularIlluminationDenoised`
- `SpecularRadianceResolved`

이런 naming은 단순 가독성보다 debugging과 RenderDoc capture 해석에 직접적인 차이를 만든다. 특히 exposure mismatch, double-remodulation, wrong roughness source 같은 integration bug를 찾기 쉬워진다.

## 5. 내 관심 분야와 연결

### Real-time Rendering / Game Engine

Material demodulation은 ray-traced reflection, RTGI, path-traced renderer뿐 아니라 modern temporal reconstruction의 기본적인 signal decomposition 사고방식과 연결된다. Game engine에서는 lighting pass와 material pass가 분리되어 있을수록 denoiser와 upscaler에 더 의미 있는 auxiliary buffer를 제공할 수 있다.

특히 engine graphics engineer 관점에서는 단순히 “denoiser SDK를 붙였다”보다 다음 질문에 답할 수 있어야 한다.

- denoiser가 받는 radiance의 물리적 의미는 무엇인가?
- diffuse/specular split은 어느 bounce에서 이루어지는가?
- roughness는 perceptual roughness인가 linear roughness인가?
- hit distance는 primary distance를 포함하는가?
- denoised output은 어떤 material factor로 remodulation되는가?

이런 contract를 설명하는 능력이 실제 renderer integration 역량에 가깝다.

### GPU / Compute / Memory

Demodulation 자체는 ALU 비용이 작지만, 전체 denoising pipeline에서는 resource 수를 증가시킬 수 있다. Diffuse/specular radiance, hit distance, moments, history, guide buffer를 모두 유지하면 1440p/4K에서 bandwidth와 VRAM 비용이 빠르게 증가한다.

따라서 graphics engineer는 algorithm accuracy뿐 아니라 다음 trade-off를 함께 본다.

- recompute material factor vs store material factor
- packed G-buffer vs separate guide textures
- FP16 vs FP32 moments
- single-pass fused demodulation vs independent pass
- transient resource aliasing
- async compute 사용 가능성

### Scientific Visualization / Semiconductor Visualization

Material demodulation의 핵심 사고방식은 scientific visualization에도 적용할 수 있다. 예를 들어 scalar field visualization에서 최종 color가

`Color = TransferFunction(value) · Lighting`

형태라면, temporal denoising이나 accumulation을 final color에 적용할 때 transfer function의 sharp boundary가 noise와 섞일 수 있다. Lighting-like component와 visualization mapping을 분리하면 데이터 feature boundary를 흐리지 않고 shading만 안정화할 수 있다.

반도체 3D 구조의 doping heatmap, material classification, cut-surface rendering에서도 **데이터 의미(signal semantics)** 와 **시각적 shading response** 를 분리하는 설계가 중요하다. Denoising뿐 아니라 temporal accumulation, LOD, interpolation에서도 같은 원리가 반복된다.

## 6. 머릿속에 남길 질문 3개

1. **왜 diffuse albedo demodulation은 비교적 단순한데 specular demodulation은 roughness, Fresnel, view direction까지 고려해야 하는가?**

2. **낮은 roughness와 높은 roughness의 specular signal에 동일한 spatial radius와 history rejection threshold를 사용하면 각각 어떤 artifact가 발생하는가?**

3. **Demodulation factor가 매우 작을 때 radiance와 variance가 폭증하는 문제를 해결하면서도 energy bias를 최소화하려면 어떤 signal contract가 필요한가?**

## 7. graphics engineer 면접 질문 1개와 답변

### 질문

**Ray-traced diffuse/specular radiance를 왜 material-demodulated space에서 denoise하는 것이 유리하며, specular reconstruction에서 roughness와 hit distance는 어떤 역할을 합니까?**

### 답변

최종 shaded radiance를 직접 denoise하면 material texture, normal map, Fresnel, roughness 변화가 모두 image-space high frequency로 포함되어 실제 Monte Carlo noise와 구분하기 어렵다. Diffuse의 경우 radiance를 albedo-like material factor로 나누면 illumination이 더 smooth해져 temporal/spatial filter가 넓은 support를 사용할 수 있고, 마지막에 current-frame material factor를 다시 곱해 texture detail을 복원할 수 있다.

Specular는 view-dependent BRDF이기 때문에 단순한 albedo division으로 충분하지 않다. Roughness는 specular lobe의 angular width를 결정해 필요한 filter footprint와 neighbor acceptance 범위를 바꾼다. 낮은 roughness는 mirror-like detail 때문에 작은 spatial support와 정확한 reprojection이 필요하고, 높은 roughness는 broad lobe라 더 넓은 support를 허용할 수 있다.

Hit distance는 reflected ray가 실제로 어느 거리의 surface를 대표하는지 알려준다. 같은 angular lobe라도 hit distance가 길면 world-space footprint가 커지므로 roughness와 함께 spatial reconstruction scale과 specular reprojection validity를 판단하는 guide가 된다.

마지막으로 demodulation factor가 작으면 signal과 variance가 `1/M`, `1/M²`에 비례해 커질 수 있으므로 denominator floor, lobe validity, probability clamp, pre-exposure, consistent remodulation contract가 필요하다. 즉 좋은 integration은 단순한 filter 구현이 아니라 **signal factorization, statistical stability, reprojection semantics, resource layout을 함께 설계하는 문제**다.

## 8. 포트폴리오 / 커리어 연결

이 개념은 graphics 포트폴리오에서 “ray-traced denoiser 구현”보다 더 깊은 설계 역량을 보여줄 수 있다.

포트폴리오에서 설명 가치가 높은 항목은 다음과 같다.

- raw shaded radiance와 demodulated illumination의 visual comparison
- diffuse/specular branch 분리 이유
- roughness에 따른 filter footprint 변화
- hit-distance-aware specular rejection
- material boundary leakage와 reflection ghosting failure case
- FP16/FP32 variance precision 비교
- demodulation/remodulation 단계의 render graph 구조
- RenderDoc에서 각 intermediate buffer의 의미를 설명하는 capture

면접에서는 다음 레벨의 질문으로 자연스럽게 확장된다.

- 왜 specular motion vector가 surface motion vector와 다를 수 있는가?
- roughness가 낮아질수록 history reprojection이 어려워지는 이유는 무엇인가?
- denoiser input과 path tracer sampling strategy가 왜 함께 설계되어야 하는가?
- material factor를 shader에서 재계산할지 buffer에 저장할지 어떤 기준으로 결정하는가?

Nintendo·Unity·real-time engine·rendering R&D 역할에서는 특정 SDK 이름보다 **rendering equation의 signal semantics를 GPU pipeline과 연결해서 설명하는 능력**이 강한 차별점이 된다.

## 9. 내일 이어서 볼 개념

**BRDF-Lobe-Aware Firefly Rejection and Energy-Preserving Specular Clamping**

오늘은 material factor를 제거하면서 작은 denominator, probability, narrow specular lobe 때문에 demodulated signal이 극단적인 값을 가질 수 있다는 문제까지 연결했다. 다음에는 specular firefly가 단순 outlier인지 실제 높은 에너지 highlight인지 구분하는 방법, luminance clamp가 만드는 bias, neighborhood statistics와 BRDF lobe probability를 이용한 robust clamping, 그리고 energy를 지나치게 잃지 않는 anti-firefly policy를 이어서 본다.

## 10. 참고 키워드

- Material Demodulation / Remodulation
- Diffuse Illumination vs Specular Illumination
- Microfacet BRDF / GGX / VNDF
- Fresnel / `F0`
- Linear Roughness / Perceptual Roughness
- Specular Hit Distance (`hitT`)
- Specular Lobe Width
- Roughness-Aware Spatial Filtering
- Hit-Distance-Aware Reprojection
- Demodulated Variance
- Probabilistic Diffuse / Specular Split
- Importance Sampling / PDF / MIS / ReSTIR
- NVIDIA NRD Material Demodulation
- NVIDIA NRD REBLUR / RELAX
- DLSS Ray Reconstruction Specular Hit Distance
- Temporal Reconstruction
- History Confidence
- Surface Compatibility
- G-buffer Packing
- Octahedral Normal Encoding
- FP16 HDR / Pre-Exposure
- Compute Shader / Wave / Subgroup
- Render Graph Resource Contract
- RenderDoc Debugging
