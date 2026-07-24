---
title: "Diffuse/Specular Signal Separation and Roughness-Aware Specular Denoising"
date: "2026-07-24"
category: "Graphics"
tags: ["GPU", "Denoising", "Diffuse", "Specular", "Roughness", "Hit Distance", "Temporal Reprojection", "Compute Shader", "Memory Layout", "Ray Tracing"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-07-24 - Diffuse/Specular Signal Separation and Roughness-Aware Specular Denoising

## 1. 오늘의 개념

**Diffuse/Specular Signal Separation**은 primary surface의 조명 결과를 최종 색으로 합친 뒤 denoising하지 않고, diffuse radiance와 specular radiance를 서로 다른 temporal·spatial signal로 유지하는 설계다.

두 신호는 같은 pixel과 geometry에서 출발해도 성질이 다르다.

- **Diffuse**는 넓은 반구 방향의 에너지가 섞인 저주파 성분에 가깝다.
- **Specular**는 view direction, normal, roughness, Fresnel, reflected hit position에 강하게 의존한다.
- 낮은 roughness의 반사는 작은 카메라 이동에도 화면상 위치가 크게 달라질 수 있다.
- 높은 roughness의 반사는 더 부드럽지만 Monte Carlo variance는 여전히 클 수 있다.

**Roughness-Aware Specular Denoising**은 roughness를 이용해 temporal accumulation, history rejection, reflection reprojection, spatial filter radius를 다르게 조절하는 방식이다.

## 2. 한 줄 핵심

**Diffuse와 specular는 같은 색의 두 채널이 아니라 서로 다른 motion·frequency·history model을 요구하는 별도 신호이며, specular denoising의 핵심 제어값은 roughness와 reflected hit distance다.**

## 3. 왜 중요한가

Diffuse와 specular를 합친 lighting을 그대로 누적하면 필터는 상충하는 요구를 동시에 만족해야 한다.

Diffuse는 비교적 긴 history와 넓은 spatial support로 저주파 noise를 안정적으로 줄일 수 있다. 반면 mirror에 가까운 specular는 선명한 reflection edge를 가지며 reflected object의 움직임과 camera parallax에 민감하다. 긴 history를 잘못 적용하면 reflection이 표면 위에 붙어 따라오는 **specular ghosting**이 발생한다.

반대로 specular responsiveness에 맞춰 전체 history를 짧게 만들면 diffuse GI는 충분히 수렴하지 못한다.

Microfacet 모델에서 흔히 사용하는 관계는 다음과 같다.

`alpha = perceptualRoughness²`

`alpha`가 작을수록 specular lobe는 좁고 signal frequency와 motion sensitivity가 높다. `alpha`가 커질수록 더 큰 filter footprint를 허용할 수 있다. 따라서 roughness는 appearance parameter이면서 동시에 denoiser의 temporal·spatial bandwidth를 결정하는 control signal이다.

## 4. 구현 관점

### 4.1 Primary hit에서 신호를 분리한다

Path tracer 또는 hybrid renderer는 첫 표면에서 다음 신호를 구분해 출력해야 한다.

- diffuse radiance + diffuse hit distance
- specular radiance + specular hit distance
- normal + roughness
- view-space depth
- motion vector
- material/object identity
- diffuse/specular history confidence

Probabilistic lobe selection을 사용해 한 pixel에서 한 종류의 ray만 추적하더라도 선택되지 않은 lobe를 다른 lobe의 값으로 채우면 안 된다. 선택 확률을 반영한 radiance estimator와 hit-distance reconstruction 정책이 필요하다.

### 4.2 Diffuse와 specular의 temporal model

Temporal accumulation은 개념적으로 다음과 같다.

`H_t = lerp(C_t, Reproject(H_{t-1}), w_history)`

하지만 `w_history`는 두 신호에서 같으면 안 된다.

Diffuse history는 surface identity, depth, normal, lighting revision이 유효하면 비교적 안정적이다. Specular history는 여기에 roughness 변화, reflection hit distance, reflected surface motion, view direction, curvature까지 추가로 확인해야 한다.

따라서 `diffConfidence`와 `specConfidence`를 별도로 관리하는 편이 안전하다. 같은 lighting revision mask를 공유할 수는 있지만 specular confidence는 reflection tracking 결과에 의해 추가 감쇠되어야 한다.

### 4.3 Roughness-aware history

Roughness 정책은 단순히 “거칠수록 history를 많이 쓴다”가 아니다.

- **Low roughness**: 작은 reprojection 오차도 잘 보인다. virtual motion이 부정확하면 history를 짧게 유지한다.
- **Medium roughness**: reflection detail과 noise가 함께 존재해 confidence tuning이 가장 중요하다.
- **High roughness**: 더 넓은 filtering을 허용할 수 있지만 geometry boundary를 넘는 blur는 금지한다.

개념적인 specular history weight는 다음과 같다.

`w_spec = w_base × C_surface × C_motion × C_hit × C_roughness × C_lighting`

각 confidence는 `[0, 1]` 범위이며 depth·normal·material identity, reflected motion, hit-distance 일치, roughness revision, lighting change를 표현한다. 강한 불일치는 reject하고 애매한 영역은 연속적으로 감쇠하는 방식이 threshold popping을 줄인다.

### 4.4 Hit distance와 virtual motion

Primary surface motion vector만으로는 reflection 내부의 움직임을 설명할 수 없다. 반사 표면이 정지해 있어도 거울 속 물체는 움직일 수 있기 때문이다.

Specular first-bounce hit distance를 `t_hit`라고 하면 reflection hit point는 다음처럼 생각할 수 있다.

`P_reflected = P_primary + R × t_hit`

이 reflected position을 이전 프레임으로 projection하면 surface motion과 다른 **specular virtual motion**을 얻을 수 있다.

낮은 roughness에서는 하나의 representative hit point가 signal motion을 비교적 잘 설명한다. 높은 roughness에서는 넓은 방향 분포를 통합하므로 단일 hit point의 의미가 약해진다. 이때 surface motion과 virtual motion을 roughness에 따라 혼합하거나 confidence를 낮추는 정책이 필요하다.

Hit distance는 denoise 대상 BRDF lobe 안에서 나온 값이어야 한다. Diffuse contribution에 적합한 direct-light distance를 specular tracking에 넣으면 reprojection이 깨질 수 있다.

### 4.5 Roughness-aware spatial filtering

Specular filter radius는 roughness와 variance를 함께 고려한다.

`r_spec = lerp(r_min, r_max, f(roughness))`

실제 radius는 다음 조건으로 조절된다.

- 높은 variance 또는 짧은 history: radius 증가
- 낮은 roughness: radius 감소
- depth·normal·material boundary: radius 감소
- hit-distance discontinuity: reflection edge 보존
- thin geometry와 높은 curvature: aggressive blur 금지

Edge-stopping weight는 개념적으로 다음 요소를 결합한다.

`w = w_kernel × w_depth × w_normal × w_roughness × w_luminance × w_hit × w_material`

Roughness가 달라지면 specular lobe 폭과 transport distribution 자체가 달라진다. 따라서 roughness boundary를 무시하면 highlight 폭과 reflection sharpness가 서로 섞인다.

### 4.6 Rendering pipeline

대표적인 흐름은 다음과 같다.

`Ray/Path Trace → Diffuse/Specular Classification → Material Demodulation → Hit-Distance Reconstruction → Separate Temporal Reprojection → Separate Variance/Confidence Update → Roughness-Aware Spatial Filtering → Material Remodulation → Composition`

Diffuse와 specular는 composition 직전까지 별도로 유지하는 것이 debug와 tuning에 유리하다.

필수 debug view는 raw diffuse/specular, hit distance, roughness, history length, virtual motion amount, confidence, rejection mask다.

### 4.7 GPU resource와 memory layout

대표적인 resource 구성은 다음과 같다.

- diffuse radiance + hit distance: `RGBA16_FLOAT`
- specular radiance + hit distance: `RGBA16_FLOAT`
- normal + roughness: packed `RGBA16_FLOAT` 또는 octahedral normal
- diffuse/specular confidence: 각각 `R8_UNORM`
- history length: `R8_UINT` 또는 packed metadata
- material/object identity: `R16_UINT` 또는 `R32_UINT`

1920×1080의 `RGBA16_FLOAT` 한 장은 약 15.8 MiB다. Diffuse와 specular의 current/history ping-pong 네 장만으로 약 63.3 MiB가 필요하며 moments와 spatial intermediate가 추가되면 bandwidth가 빠르게 증가한다.

Render graph에서는 다음 trade-off를 본다.

- combined dispatch: descriptor·dispatch overhead 감소, branch와 register pressure 증가
- separate dispatch: specialization과 occupancy 개선, pass 수 증가
- transient aliasing: lifetime이 겹치지 않는 intermediate 재사용
- radiance/hit-distance packing: fetch 수 감소
- G-buffer 재사용과 denoiser 전용 repack 사이의 bandwidth 비교

하나의 compute dispatch가 두 신호를 처리하는 것과 하나의 history model로 두 신호를 합치는 것은 전혀 다른 선택이다. API가 Vulkan, DirectX, Metal, WebGPU 중 무엇이든 logical separation은 유지되어야 한다.

### 4.8 흔한 실패 패턴

1. Combined lighting에 하나의 motion vector만 사용한다.
2. Roughness를 최종 shading에만 사용하고 denoiser policy에는 반영하지 않는다.
3. Specular lobe와 무관한 distance를 specular hit distance로 전달한다.
4. Diffuse와 specular의 history confidence를 완전히 공유한다.
5. Roughness boundary에서 spatial filtering을 제한하지 않는다.
6. Scene scale 대비 FP16 hit-distance range와 precision을 검토하지 않는다.

## 5. 내 관심 분야와 연결

### Real-time rendering과 game engine

Denoiser는 post-process blur가 아니라 명확한 **signal contract**를 가진 subsystem이다. C++ render graph와 RHI는 radiance가 demodulated 상태인지, hit-distance 단위가 무엇인지, roughness encoding이 perceptual인지 linear alpha인지, reset ownership이 어디에 있는지 명시해야 한다.

### Vulkan / DirectX / Metal / WebGPU

별도 diffuse/specular history는 resource transition과 descriptor 수를 늘린다. Combined compute dispatch로 비용을 줄일 수 있지만 history, confidence, reset reason은 논리적으로 분리해야 한다. Vulkan과 DirectX에서는 transient aliasing과 async compute overlap을, WebGPU에서는 storage texture format과 bind-group 제한을 고려한다.

### CFD와 scientific visualization

Pressure 같은 smooth field, material interface, shock indicator, particle-derived density는 서로 다른 frequency와 motion model을 가진다. 이를 하나의 filter policy로 처리하지 않는다는 점에서 diffuse/specular separation과 같은 architecture pattern이다.

### Semiconductor 3D visualization

Material appearance, geometry identity, doping scalar, lighting component를 최종 RGB 이전에 분리하면 얇은 layer와 물질 경계가 temporal filtering으로 번지는 문제를 줄일 수 있다.

## 6. 머릿속에 남길 질문 3개

1. Primary surface motion이 정확해도 mirror reflection history가 틀릴 수 있는 이유는 무엇이며 hit distance는 이를 어떻게 보완하는가?
2. Roughness가 높아질수록 filter radius를 넓힐 수 있지만 depth·normal·material guidance가 여전히 필요한 이유는 무엇인가?
3. Diffuse와 specular를 하나의 compute dispatch에서 처리하더라도 logical history와 confidence를 분리해야 하는 이유는 무엇인가?

## 7. graphics engineer 면접 질문 1개와 답변

### 질문

왜 real-time ray tracing denoiser에서 diffuse와 specular를 분리하고 specular에는 roughness와 hit distance를 추가로 사용하는가?

### 답변

Diffuse와 specular는 동일한 primary surface에서 생성되지만 spatial frequency와 temporal motion이 다르다. Diffuse는 넓은 방향의 에너지를 통합해 비교적 저주파이며 surface motion을 중심으로 reprojection할 수 있다. Specular는 view direction과 roughness에 민감하고, 낮은 roughness에서는 reflected object가 primary surface와 다른 motion을 가진다.

같은 history weight와 filter radius를 사용하면 diffuse는 충분히 수렴하지 못하거나 specular가 ghosting된다. Hit distance로 reflected point 또는 virtual motion을 추정하면 reflection 내부의 motion을 더 잘 따라갈 수 있다. Roughness는 specular lobe의 폭을 나타내므로 낮은 roughness에서는 작은 spatial radius와 엄격한 reprojection을, 높은 roughness에서는 더 넓은 filtering을 허용하는 기준이 된다.

실무적으로는 diffuse/specular radiance, hit distance, history confidence, moments를 별도로 유지하고 depth·normal·roughness·material identity로 각각의 history를 검증한다.

## 8. 포트폴리오 / 커리어 연결

강한 포트폴리오 증거는 최종 이미지보다 architecture와 failure analysis다.

- Combined denoising과 separated denoising 구조 비교
- Diffuse/specular별 raw signal과 결과
- Roughness 구간별 history length와 filter radius 시각화
- Primary motion과 specular virtual motion 비교
- Reflected object motion에서 ghosting 원인 분석
- `RGBA16_FLOAT` history 구성의 VRAM·bandwidth 산정
- Separate dispatch와 combined dispatch의 GPU timing 비교

면접에서는 “roughness로 blur radius를 바꿨다”보다 왜 roughness가 signal frequency와 reprojection validity를 바꾸는지, hit distance가 왜 필요한지, separation이 render graph와 memory layout에 어떤 비용을 만드는지 설명하는 것이 중요하다.

## 9. 내일 이어서 볼 개념

**Specular Virtual Motion and Reflection Reprojection**

다음에는 primary surface motion만으로 반사를 추적할 수 없는 이유와 reflected hit position, parallax, virtual history, roughness 기반 surface/virtual motion blending을 더 깊게 살펴본다.

## 10. 참고 키워드

- Diffuse / Specular Signal Separation
- Roughness-Aware Denoising
- Perceptual Roughness / Alpha Roughness
- Specular Lobe Width
- Reflection Hit Distance
- Virtual Motion / Specular Tracking
- Temporal Accumulation / History Confidence
- Hit-Distance Reconstruction
- Material Demodulation / Remodulation
- REBLUR / RELAX
- NVIDIA Real-Time Denoisers (NRD)
- VNDF Specular Sampling
- Compute Shader Ping-Pong Resources
- Render Graph Transient Aliasing
- 공식 참고: [NVIDIA Real-Time Denoisers](https://github.com/NVIDIA-RTX/NRD)
