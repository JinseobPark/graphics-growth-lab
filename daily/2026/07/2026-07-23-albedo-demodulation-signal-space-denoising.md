---
title: "Albedo Demodulation and Signal-Space Denoising"
date: "2026-07-23"
category: "Graphics"
tags: ["GPU", "Denoising", "Albedo Demodulation", "Signal Space", "Radiance", "BRDF", "Compute Shader", "Memory Layout", "Scientific Visualization"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-07-23 - Albedo Demodulation and Signal-Space Denoising

## 1. 오늘의 개념

**Albedo Demodulation**은 noisy shading 결과에서 표면 재질이 곱해 만든 결정적 변화인 albedo 또는 BRDF factor를 제거한 뒤, 조명에 가까운 신호 공간에서 denoising하고 마지막에 현재 프레임의 재질을 다시 곱하는 방식이다.

가장 단순한 diffuse 모델에서는 화면에 관측되는 신호를 다음처럼 분해해 생각할 수 있다.

`C_noisy = L_noisy ⊙ M`

- `C_noisy`: Monte Carlo noise가 포함된 최종 diffuse shading
- `L_noisy`: 필터링하려는 incident illumination 또는 radiance 계열 신호
- `M`: diffuse albedo를 포함한 material response
- `⊙`: 채널별 곱

Demodulation과 remodulation은 다음과 같다.

`L_demod = C_noisy / max(M, ε)`

`C_filtered = Denoise(L_demod) ⊙ M_current`

핵심은 최종 shaded color가 아니라, **노이즈가 실제로 존재하는 물리적 신호에 더 가까운 공간(signal space)** 에서 시간적·공간적 통계를 계산하는 것이다.

Diffuse에서는 albedo가 대표적인 factor지만, specular에서는 단순히 `F0`로 나누는 것으로 충분하지 않다. Specular response는 roughness, `N·V`, Fresnel, microfacet distribution에 따라 크게 달라지므로 보통 preintegrated environment BRDF 또는 엔진의 BSDF 모델에서 계산한 material factor를 사용한다.

## 2. 한 줄 핵심

**Denoiser는 표면 무늬가 섞인 최종 색보다 재질을 제거한 illumination/radiance 신호를 다룰 때, 실제 노이즈와 deterministic material detail을 더 정확히 구분할 수 있다.**

## 3. 왜 중요한가

최종 shaded color를 그대로 temporal accumulation과 spatial filtering에 넣으면 albedo texture의 체크무늬, 로고, 미세 패턴이 조명 변화나 noise처럼 보인다. 그 결과 다음 문제가 발생한다.

- 밝고 어두운 albedo 경계에서 luminance variance가 불필요하게 증가한다.
- Edge-stopping filter가 재질 경계를 조명 경계로 오해한다.
- 넓은 spatial support가 서로 다른 material color를 섞어 color bleeding을 만든다.
- Temporal history가 albedo animation이나 texture streaming 변화에 끌려가 ghosting을 남긴다.

반대로 material factor를 제거하면 같은 조명을 받는 표면은 albedo가 달라도 더 비슷한 signal distribution을 갖는다. 이는 denoiser가 사용하는 평균, variance, history confidence, luminance difference를 더 안정적으로 만든다.

여기서 중요한 오해가 하나 있다. Demodulation은 Monte Carlo variance를 마법처럼 제거하지 않는다. Dark albedo로 나누면 절대값과 numerical error가 오히려 커질 수 있다. 이 기법의 실질적 이점은 **noise가 있는 신호와 deterministic shading factor를 분리하여 필터의 통계적 가정을 더 잘 맞추는 것**이다.

전날 다룬 À-trous filter가 “어떤 이웃을 어느 범위까지 섞을 것인가”를 결정했다면, 오늘의 demodulation은 “처음부터 무엇을 필터링해야 하는가”를 결정한다.

## 4. 구현 관점

### 4.1 Rendering pipeline에서의 위치

실무 파이프라인은 다음처럼 구성할 수 있다.

`Ray/Path Trace → Material Demodulation → Temporal Accumulation → Variance Estimation → À-Trous Filtering → Material Remodulation → Final Composition`

입력 resource는 보통 다음 계열로 나뉜다.

- Noisy diffuse signal
- Noisy specular signal
- Diffuse material factor
- Specular material factor
- Depth, normal, roughness, motion vector
- Stable object/material identity
- History confidence와 reactive metadata

Diffuse와 specular는 통계적 특성과 material response가 다르므로 가능한 한 별도 signal로 유지한다.

### 4.2 Diffuse demodulation

Lambertian 계열에서 엔진이 diffuse lighting과 albedo를 곱해 저장한다면, 다음 형태가 가장 직관적이다.

`diffuseDemod = noisyDiffuse / max(diffuseFactor, ε)`

여기서 `diffuseFactor`가 정확히 무엇인지는 renderer의 convention에 달려 있다.

- `albedo`
- `albedo / π`
- diffuse BSDF와 cosine term 일부를 포함한 factor
- denoiser integration layer가 정의한 material factor

중요한 것은 front-end와 back-end가 동일한 정의를 사용해야 한다는 점이다. 단위와 factor convention이 어긋나면 denoised 결과는 밝기 보존에 실패한다.

### 4.3 Dark albedo와 division stability

Material factor가 0 또는 매우 작으면 division은 `INF`, `NaN`, firefly를 만든다.

실무에서는 다음 정책을 함께 둔다.

- factor에 channel별 lower bound `ε` 적용
- 유효하지 않은 pixel은 별도 validity mask로 history reject
- demodulated radiance에 robust upper bound 적용
- finite check 후 zero 또는 neighborhood-safe value로 sanitize
- 완전한 black material은 denoiser 입력보다 composition 단계에서 처리

`ε`를 너무 크게 잡으면 어두운 재질의 lighting을 편향시키고, 너무 작게 잡으면 half precision overflow와 outlier가 증가한다. 따라서 scene exposure와 signal range를 기준으로 결정해야 한다.

### 4.4 Specular demodulation은 왜 더 어려운가

Specular는 `specularColor` 하나로 factorization되지 않는다.

`specFactor = EnvBRDF(F0, roughness, NdotV)`

`specDemod = noisySpecular / max(specFactor, ε)`

Specular factor는 다음에 민감하다.

- Linear roughness 또는 alpha roughness
- Fresnel reflectance `F0`
- View angle `N·V`
- Metalness workflow
- Multi-scattering compensation
- Clearcoat, anisotropy, transmission layer

따라서 엔진의 BRDF와 다른 근사식을 사용하면 remodulation 후 energy와 hue가 달라진다. 특히 roughness가 변하면 단순한 material color 변화가 아니라 specular lobe 자체가 변하므로 temporal history confidence를 낮추거나 reset해야 한다.

### 4.5 Current-frame factor로 remodulation하기

History에는 demodulated signal을 저장하고, 최종 composition에서는 **현재 프레임의 material factor**를 곱하는 것이 일반적이다.

이 구조의 장점은 diffuse albedo texture가 바뀌어도 과거의 material color가 history에 남지 않는다는 점이다.

- Diffuse albedo만 변경: illumination history를 재사용할 가능성이 높다.
- Roughness/F0 변경: specular transport response가 달라지므로 history를 감쇠해야 한다.
- Geometry 또는 normal 변경: surface identity validation이 우선이다.
- Emissive 변경: 독립 signal 또는 reactive mask가 필요하다.

즉, material을 제거했다고 해서 모든 material revision을 무시할 수 있는 것은 아니다. 어떤 parameter가 단순 modulation이고 어떤 parameter가 sampling distribution과 transport 자체를 바꾸는지 구분해야 한다.

### 4.6 Pre-exposure와 HDR signal range

Albedo는 무차원 factor이므로 algebra상 pre-exposed color를 demodulate해도 구조는 유지된다.

`(C · exposureScale) / M = (C / M) · exposureScale`

하지만 temporal history는 프레임 간 동일한 exposure space에서 비교되어야 한다. Auto exposure가 변하면 다음 중 하나가 필요하다.

- 이전 history를 현재 pre-exposure 기준으로 rescale
- exposure-independent radiance 공간에 history 저장
- 큰 exposure jump에서 history confidence 감소

Demodulation과 exposure normalization을 별개의 문제로 관리해야 한다.

### 4.7 GPU resource와 memory layout

대표적인 resource 구성을 보면 다음과 같다.

- Diffuse noisy/demodulated signal: `RGBA16_FLOAT`
- Specular noisy/demodulated signal: `RGBA16_FLOAT`
- Diffuse albedo 또는 factor: `R11G11B10_FLOAT`, `RGBA8_UNORM`, `RGBA16_FLOAT`
- Normal + roughness: packed `RGBA16_FLOAT` 또는 octahedral normal + 별도 roughness
- History confidence: `R8_UNORM`
- Stable material/object ID: `R16_UINT` 또는 `R32_UINT`

`RGBA16_FLOAT`는 픽셀당 8바이트이므로 1920×1080 한 장이 약 15.8 MiB다. Diffuse와 specular 각각에 temporal ping-pong과 spatial ping-pong을 별도로 두면 VRAM과 bandwidth가 빠르게 증가한다.

따라서 material factor를 별도 full-resolution texture로 저장할지, 기존 G-buffer에서 재구성할지 판단해야 한다.

- 별도 factor texture: shader가 단순하고 BRDF convention이 명확함
- G-buffer 재구성: memory traffic 감소, ALU와 shader 복잡도 증가
- Ray generation에서 inline demodulation: 추가 pass와 intermediate write 제거
- Denoiser front-end pack 단계와 fusion: resource transition과 bandwidth 절감

현대 GPU에서는 간단한 BRDF factor 계산의 ALU 비용보다 full-resolution texture read/write 비용이 더 클 수 있다.

### 4.8 Signal-space denoising의 일반 원칙

Signal-space denoising은 albedo에만 한정되지 않는다.

**관측 이미지에 deterministic mapping이 적용되기 전의 quantity를 필터링한다**는 원칙으로 확장할 수 있다.

- Shadow: 최종 lit color보다 visibility signal
- Ambient occlusion: shaded color보다 occlusion scalar
- Reflection: composite color보다 specular radiance와 hit distance
- Scientific visualization: colormap이 적용된 RGB보다 원래 scalar field
- Volume visualization: transfer function 결과보다 density, optical depth, radiance component를 목적에 맞게 분리

단, transfer function과 volume compositing은 비선형이므로 단순 division으로 역분해할 수 없는 경우가 많다. 이때는 물리량 공간에서 filter를 설계하거나, geometry/material boundary를 guidance로 사용해야 한다.

## 5. 내 관심 분야와 연결

### Real-time rendering과 game engine

Engine architecture 관점에서는 denoiser를 단일 post-process가 아니라 **signal contract를 가진 subsystem**으로 봐야 한다. Renderer는 어떤 signal이 diffuse인지, 어떤 factor가 제거되었는지, exposure space가 무엇인지, history reset 조건이 무엇인지 명확히 전달해야 한다.

Render graph에는 다음 dependency가 드러나야 한다.

- G-buffer material factor 생성
- noisy signal 생성
- denoiser front-end pack
- temporal/spatial denoising
- current material factor 기반 composition

### CFD와 scientific visualization

CFD scalar field에 colormap을 먼저 적용한 뒤 RGB 이미지를 blur하면, transfer function의 색 경계가 물리적 gradient처럼 취급된다. 반면 pressure, temperature, vorticity 같은 원래 scalar를 filtering하고 마지막에 colormap을 적용하면 visualization style과 physical signal을 분리할 수 있다.

다만 shock, interface, boundary layer처럼 실제 불연속이 중요한 경우에는 material ID, region ID, gradient magnitude를 edge-stopping guidance로 사용해야 한다.

### Semiconductor 3D visualization

Material color와 doping concentration heatmap을 최종 RGB에서 섞어 필터링하면 서로 다른 물질 경계에서 농도 색상이 번질 수 있다. Doping scalar, material identity, geometry coverage를 별도 signal로 유지한 뒤 최종 colormap과 material appearance를 합성하는 구조가 더 안전하다.

### WebGPU / Vulkan / DirectX / Metal

API가 달라도 핵심 resource model은 같다.

- demodulated signal은 storage texture/UAV로 기록
- temporal history는 ping-pong resource
- material ID는 integer load로 정확히 비교
- factor texture는 반드시 linear space로 sampling
- pass fusion 여부는 bandwidth, register pressure, occupancy를 함께 고려

## 6. 머릿속에 남길 질문 3개

1. 내가 필터링하는 값은 실제 stochastic signal인가, 아니면 material과 presentation mapping이 이미 섞인 최종 관측값인가?
2. Diffuse albedo 변경과 specular roughness 변경은 temporal history 관점에서 왜 같은 방식으로 처리하면 안 되는가?
3. Material factor를 texture로 저장하는 비용과 shader에서 재구성하는 비용 중 현재 GPU pipeline에서는 무엇이 더 큰가?

## 7. graphics engineer 면접 질문 1개와 답변

### 질문

왜 ray-traced diffuse 또는 specular 결과를 최종 shaded color 상태로 바로 denoising하지 않고 material demodulation을 수행하는가?

### 답변

최종 shaded color에는 Monte Carlo noise뿐 아니라 albedo texture와 BRDF가 만든 deterministic high-frequency variation이 함께 들어 있다. Denoiser가 이 값을 그대로 사용하면 material detail을 noise로 오해하거나 서로 다른 material color를 spatially 섞을 수 있다.

Material demodulation은 noisy signal에서 diffuse albedo 또는 specular BRDF factor를 제거해 illumination/radiance에 가까운 공간에서 temporal statistics와 spatial filtering을 수행하게 한다. 필터링 후 현재 프레임의 material factor를 다시 곱하면 material detail을 보존하면서 lighting noise를 줄일 수 있다.

다만 dark factor division에 따른 numerical instability, specular factor의 roughness·view dependence, exposure normalization, material revision에 따른 history invalidation을 함께 설계해야 한다.

## 8. 포트폴리오 / 커리어 연결

이 개념을 이해하고 설명할 수 있다는 것은 단순 blur shader 구현을 넘어, renderer의 **signal decomposition과 denoiser integration contract**를 설계할 수 있음을 보여준다.

포트폴리오에서는 다음 관점이 강한 증거가 된다.

- Final color denoising과 demodulated signal denoising의 비교
- Diffuse/specular 분리 architecture diagram
- Dark albedo, animated material, roughness change에 대한 failure analysis
- Albedo, demodulated radiance, variance, history confidence debug view
- Full-resolution factor buffer와 on-the-fly reconstruction의 bandwidth 비교
- CFD scalar-space filtering과 colormap-space filtering의 차이

게임 엔진·real-time rendering 면접에서는 “어떤 filter를 썼는가”보다 “어떤 signal을 왜 분리했는가”를 설명할 수 있는 능력이 더 높은 설계 역량으로 평가될 수 있다.

## 9. 내일 이어서 볼 개념

**Diffuse/Specular Signal Separation and Roughness-Aware Specular Denoising**

같은 geometry를 공유하더라도 diffuse와 specular는 spatial frequency, temporal stability, hit distance, roughness sensitivity가 다르다. 다음에는 두 signal을 왜 분리해야 하며, roughness와 hit distance가 specular history와 filter radius를 어떻게 결정하는지 이어서 본다.

## 10. 참고 키워드

- Albedo Demodulation / Remodulation
- Material Demodulation
- Signal-Space Denoising
- Radiance vs Irradiance
- BRDF Factorization
- Diffuse/Specular Separation
- Environment BRDF / Split-Sum Approximation
- `N·V`, `F0`, Linear Roughness
- Pre-Exposure / History Rescaling
- NRD Material Factors
- SVGF
- Edge-Aware Filtering
- Temporal Accumulation
- An Efficient Denoising Algorithm for Global Illumination
- NVIDIA Real-Time Denoisers Material Demodulation
- DLSS Ray Reconstruction Diffuse/Specular Albedo Inputs
