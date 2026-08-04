---
title: "Variance-Guided À-Trous Wavelet Filtering and Edge-Stopping Functions"
date: "2026-08-04"
category: Graphics
tags: ["GPU", "Denoising", "À-Trous Wavelet", "Edge-Stopping Function", "SVGF", "Compute Shader", "Memory Layout", "Ray Tracing", "Scientific Visualization"]
level: intermediate
---

# [Daily Graphics Growth] 2026-08-04 - Variance-Guided À-Trous Wavelet Filtering and Edge-Stopping Functions

## 1. 오늘의 개념

전날의 **Moment-Preserving History Reconstruction and Variance Reinitialization**에서는 temporal reprojection이 끊긴 영역의 평균과 second moment를 일관되게 복구하고, 그 결과로 얻은 variance가 다음 필터의 신뢰도 판단을 이끌도록 만들었다. 오늘은 그 variance를 실제 공간 필터의 반경과 강도로 연결하는 단계다.

오늘의 개념은 **Variance-Guided À-Trous Wavelet Filtering**과 **Edge-Stopping Functions**이다. À-trous 필터는 매 iteration마다 kernel tap의 간격만 넓혀가며 동일한 해상도에서 점점 큰 spatial footprint를 형성하는 다중 스케일 필터다. 여기에 depth, normal, luminance, material, roughness 같은 guide를 이용한 edge-stopping weight를 곱하면 geometry와 shading boundary를 넘지 않으면서 noise를 줄일 수 있다.

Pixel `p`의 입력 signal을 `L(p)`, 주변 tap을 `q`, 기본 wavelet kernel을 `h(p,q)`, edge-stopping weight를 `w(p,q)`라 하면 한 단계의 출력은 다음과 같이 볼 수 있다.

`L'(p) = Σ_q h(p,q) · w(p,q) · L(q) / Σ_q h(p,q) · w(p,q)`

À-trous iteration `i`의 tap stride는 보통 다음처럼 증가한다.

`s_i = 2^i`

따라서 5×5 kernel을 사용하더라도 iteration이 진행될수록 실제로 참조하는 화면 반경은 `1, 2, 4, 8, ...` pixel 단위로 커진다. Tap 수는 고정되지만 필터가 보는 공간적 규모는 빠르게 확대된다.

핵심은 단순히 넓게 blur하는 것이 아니다. **Variance는 얼마나 강하게 필터링할지 결정하고, edge-stopping function은 어디까지 필터링해도 되는지 결정한다.**

## 2. 한 줄 핵심

**À-trous 필터는 고정된 tap 수로 다중 스케일 noise를 제거하고, variance와 geometry-aware edge weight는 detail과 surface boundary를 보존하는 필터의 제동 장치다.**

## 3. 왜 중요한가

실시간 ray tracing, path tracing, stochastic shadow, diffuse GI, glossy reflection은 낮은 sample count에서 고주파 Monte Carlo noise를 만든다. Temporal accumulation은 정지하거나 천천히 움직이는 영역에서는 강력하지만, disocclusion, 빠른 motion, lighting change, animated normal에서는 history가 짧아지고 spatial reconstruction의 역할이 커진다.

일반 Gaussian blur를 적용하면 noise는 줄지만 다음 문제가 발생한다.

- foreground와 background가 depth edge를 넘어 섞인다.
- 서로 다른 normal을 가진 면의 lighting이 누출된다.
- 밝은 specular highlight가 주변 diffuse surface로 번진다.
- 얇은 geometry와 silhouette이 사라진다.
- 재질 경계를 넘어 albedo 또는 roughness 특성이 혼합된다.

반대로 edge 조건을 지나치게 엄격하게 만들면 유효한 이웃을 거의 사용하지 못해 noise가 남는다. 즉 denoiser는 항상 두 실패 모드 사이에서 균형을 잡는다.

- **과도한 smoothing**: light leaking, halo, detail loss, reflection smear
- **과도한 edge rejection**: residual noise, temporal shimmer, isolated pixel, convergence 저하

Variance-guided À-trous 구조가 중요한 이유는 이 균형을 pixel 단위로 조절할 수 있기 때문이다. Variance가 큰 pixel은 luminance 차이에 더 관대하게 반응해 넓은 support를 사용하고, variance가 작은 pixel은 작은 차이도 detail일 가능성이 높다고 보고 더 보수적으로 필터링한다.

SVGF 계열의 핵심 관점도 temporal accumulation으로 effective sample count를 늘리고, spatiotemporal luminance variance를 추정한 뒤, 그 variance를 hierarchical image-space wavelet filter의 강도 조절에 사용하는 것이다. 이는 temporal filter와 spatial filter를 서로 독립된 두 단계가 아니라 **통계 상태를 공유하는 하나의 reconstruction pipeline**으로 본다는 의미다.

## 4. 구현 관점

### 4.1 À-trous가 만드는 다중 스케일 footprint

대표적인 1D B-spline kernel은 다음과 같다.

`h = [1, 4, 6, 4, 1] / 16`

2D kernel은 이를 separable outer product로 확장할 수 있다.

`H(x,y) = h(x) · h(y)`

Iteration `i`에서는 kernel coefficient는 그대로 두고 tap offset에 stride `s_i = 2^i`를 곱한다.

- iteration 0: offset `{-2,-1,0,1,2}`
- iteration 1: offset `{-4,-2,0,2,4}`
- iteration 2: offset `{-8,-4,0,4,8}`
- iteration 3: offset `{-16,-8,0,8,16}`

일반 MIP blur처럼 해상도를 줄였다가 복원하지 않으므로 각 단계의 결과는 원본 해상도를 유지한다. 이 때문에 depth, normal, motion, material ID 같은 full-resolution guide와 직접 비교하기 쉽다.

5×5 sparse kernel은 iteration마다 25개 tap을 사용하지만, logical radius는 지수적으로 증가한다. 따라서 작은 iteration 수로 넓은 영역을 다룰 수 있다. 다만 stride가 커질수록 texture cache locality가 나빠지고, 얇은 구조를 건너뛰는 sparse sampling 문제가 커질 수 있다.

### 4.2 최종 tap weight의 구조

실무적인 tap weight는 하나의 값이 아니라 여러 조건의 곱으로 구성된다.

`w(p,q) = w_l(p,q) · w_z(p,q) · w_n(p,q) · w_m(p,q) · w_r(p,q) · w_c(p,q)`

각 항의 의미는 다음과 같다.

- `w_l`: luminance 또는 signal similarity
- `w_z`: depth 또는 world-position continuity
- `w_n`: normal orientation similarity
- `w_m`: material/surface identity compatibility
- `w_r`: roughness 또는 lobe compatibility
- `w_c`: confidence/history validity

이 곱셈 구조는 어느 하나의 조건이 강하게 불일치하면 전체 weight를 낮춘다. 그러나 모든 항을 연속적인 exponential weight로 만들 필요는 없다. Material ID, topology revision, object ID처럼 의미가 불연속적인 guide는 hard rejection이 더 자연스럽다.

`w_m(p,q) = 1  if materialId_p == materialId_q, else 0`

반면 depth와 normal은 작은 차이를 허용하는 soft weight가 보통 더 안정적이다.

### 4.3 Variance-guided luminance weight

Luminance difference 기반 weight의 대표적인 형태는 다음과 같다.

`w_l(p,q) = exp(-|Y_p - Y_q| / (φ_l + ε))`

여기서 `Y`는 luminance이고, `φ_l`은 허용할 luminance 차이의 규모다. Variance-guided filtering에서는 이를 local standard deviation과 연결한다.

`φ_l(p) = k_l · sqrt(max(σ_p², σ_floor²))`

Variance가 크면 `φ_l`이 커져 luminance 차이에 관대해진다. Noise가 큰 영역에서 이웃 값이 크게 달라도 실제 detail이 아니라 stochastic noise일 수 있기 때문이다. 반대로 variance가 작으면 작은 luminance 차이도 실제 edge일 가능성이 커지므로 weight가 빠르게 줄어든다.

Central pixel variance만 사용하면 noisy outlier가 필터 반경을 지나치게 넓힐 수 있다. 다음과 같은 대안이 있다.

- center와 tap variance의 평균 사용
- 작은 same-surface neighborhood에서 variance를 선행 blur
- temporal history length에 따른 variance confidence 보정
- firefly-clamped variance와 raw radiance variance 분리
- repaired history에는 variance inflation 적용

대표적인 대칭형 scale은 다음처럼 생각할 수 있다.

`φ_l(p,q) = k_l · sqrt(max(σ_p² + σ_q², σ_floor²))`

이는 두 pixel 모두 불확실할 때 더 넓은 support를 허용한다. 다만 variance buffer 자체가 noisy하면 weight가 frame마다 크게 바뀌어 flicker를 만들 수 있으므로 variance guide도 안정화가 필요하다.

### 4.4 Depth edge-stopping function

단순한 view-space depth 차이는 카메라에서 멀어질수록 같은 world-space 거리 변화에 대해 민감도가 달라진다. 따라서 고정 threshold만 사용하면 가까운 곳에서는 과도하게 섞이고 먼 곳에서는 지나치게 끊길 수 있다.

기본 형태는 다음과 같다.

`w_z(p,q) = exp(-|z_p - z_q| / (φ_z + ε))`

`φ_z`는 다음 요소와 연결할 수 있다.

- local depth gradient
- projection scale
- kernel stride
- pixel footprint의 world-space 크기
- surface slope

예를 들어 local depth gradient를 `∇z_p`라고 하면 stride가 커질수록 허용 오차도 증가시키는 형태를 생각할 수 있다.

`φ_z = k_z · (|∇z_p| · |Δx| + z_relative + ε)`

여기서 `|Δx|`는 현재 tap의 pixel offset 크기다. 이 방식은 같은 평면 위에서 perspective와 slope 때문에 발생하는 자연스러운 depth 변화는 허용하고, 실제 disocclusion boundary는 차단하려는 목적을 가진다.

World position을 직접 비교하면 의미가 명확하지만 3채널 bandwidth가 필요하다. 많은 real-time pipeline은 linear viewZ와 screen-space gradient만으로 비슷한 판단을 만든다.

### 4.5 Normal edge-stopping function

Normal weight는 보통 dot product에 기반한다.

`w_n(p,q) = max(dot(n_p, n_q), 0)^φ_n`

`φ_n`이 크면 작은 normal 차이에도 빠르게 weight가 줄어든다. Smooth surface에서는 geometric normal보다 shading normal을 사용하면 bump/normal map detail을 보존할 수 있지만, animated normal이나 고주파 normal map은 유효한 support를 지나치게 끊을 수 있다.

Signal에 따라 guide normal 선택이 달라질 수 있다.

- diffuse irradiance: geometric normal 또는 low-frequency shading normal
- glossy reflection: shading normal과 roughness를 함께 고려
- shadow visibility: normal보다 depth와 blocker topology가 더 중요할 수 있음
- scientific scalar field: surface normal 대신 scalar gradient direction 사용 가능

Normal encoding도 품질에 영향을 준다. Octahedral encoding, `R10G10B10A2_UNORM`, `RG16_SNORM` 등은 bandwidth와 precision 사이의 trade-off를 만든다. Decode 비용은 작지만 quantization으로 인해 threshold 근처의 weight가 흔들릴 수 있다.

### 4.6 Material, roughness, surface identity

Depth와 normal만 같아도 서로 다른 shading domain일 수 있다. 예를 들어 coplanar한 두 재질이 맞닿아 있으면 geometry guide만으로는 경계를 구분하지 못한다.

Material/surface guide로 고려할 수 있는 값은 다음과 같다.

- material ID
- object/instance ID
- primitive 또는 surface ID
- roughness class
- emissive/non-emissive flag
- topology revision
- level-set material label

Roughness는 특히 specular signal에서 중요하다. Roughness가 낮은 mirror-like lobe와 높은 glossy lobe는 동일한 radiance difference를 다르게 해석해야 한다. 일반적으로 rough surface는 넓은 spatial support를 허용할 수 있지만, sharp reflection은 작은 geometric 변화에도 reflected content가 크게 바뀌므로 강한 boundary 보존이 필요하다.

Material ID를 hard gate로 사용하면 leakage는 줄지만 작은 decal, texture-driven roughness, material blending 경계에서 support가 파편화될 수 있다. 따라서 엔진에서는 signal별로 다음 정책을 분리하는 편이 좋다.

- hard identity gate
- roughness difference soft weight
- albedo demodulation 이후 material gate 완화
- emissive 및 thin geometry 전용 policy

### 4.7 Demodulation과 illumination-space filtering

Albedo가 포함된 final color를 직접 필터링하면 texture edge가 lighting noise로 오인되거나, 반대로 texture variation 때문에 유효한 lighting support가 차단될 수 있다.

Diffuse signal은 흔히 다음처럼 분해할 수 있다.

`radiance ≈ diffuseAlbedo · irradiance`

필터는 albedo를 제거한 irradiance-like signal에 적용하고 이후 다시 albedo를 곱할 수 있다. 이를 **demodulation/remodulation** 관점으로 볼 수 있다.

- demodulated domain: lighting noise를 더 직접적으로 비교 가능
- remodulated domain: 최종 material appearance 복원

Specular는 단순 roughness 하나로 완전히 demodulate하기 어렵고, BRDF lobe와 reflected radiance의 관계가 view-dependent하다. 따라서 specular denoiser는 roughness, hit distance, virtual motion, normal, view vector 같은 guide를 더 강하게 요구한다.

### 4.8 Iteration별 filter strength

모든 iteration에 같은 edge threshold를 사용하면 큰 stride에서 support가 지나치게 줄어들 수 있다. Tap 사이의 실제 거리가 커지기 때문이다.

Iteration scale `s_i`를 기준으로 threshold를 조정할 수 있다.

`φ_z(i) ∝ s_i`

`φ_l(i) ∝ f(σ²_i, s_i)`

하지만 threshold를 단순히 stride에 비례해 넓히면 큰 scale에서 edge leakage가 증가한다. 따라서 iteration이 진행될수록 다음 정보가 중요해진다.

- 이전 iteration에서 필터된 variance
- depth gradient와 normal cone
- surface identity purity
- valid support coverage
- thin geometry 또는 silhouette mask

초기 iteration은 local noise 제거에 집중하고, 후반 iteration은 넓은 low-frequency noise를 줄인다. Signal이 이미 안정적이거나 variance가 threshold 아래라면 후반 iteration을 중단하거나 output을 유지할 수 있다.

### 4.9 Variance propagation

Color만 ping-pong하고 variance를 최초 값으로 고정하면 후반 iteration의 luminance threshold가 현재 signal 상태와 맞지 않는다. 한 단계에서 noise가 줄었는데도 초기 variance가 크면 다음 단계가 과도하게 blur할 수 있다.

따라서 À-trous pass에서 color와 함께 variance 또는 second moment를 전파하는 방법이 사용된다.

가중 평균의 variance를 완전히 정확하게 추적하려면 tap correlation까지 알아야 하지만, screen-space filter에서는 보통 근사적인 propagation을 사용한다.

- 동일 weight로 first/second moment 필터링
- variance에 squared normalized weight를 적용
- filtered luminance neighborhood에서 variance 재추정
- iteration별 conservative floor 유지

독립 표본이라는 가정 아래 normalized weight `a_i`를 사용하면 평균의 variance는 대략 다음과 같다.

`σ_out² ≈ Σ_i a_i² σ_i²`

그러나 screen-space neighbor는 강하게 상관되어 있으므로 실제 uncertainty를 과소평가할 수 있다. 따라서 variance floor, correlation factor, history class에 따른 inflation이 필요하다.

이 문제는 내일 이어서 볼 **Variance Stabilization Across À-Trous Scales and Filtered Moment Propagation**의 핵심이 된다.

### 4.10 Compute shader 실행 구조

가장 직접적인 구조는 iteration마다 full-screen compute dispatch를 수행하고, 두 개의 radiance/moment texture를 ping-pong하는 방식이다.

- input radiance/moments: SRV 또는 sampled texture
- output radiance/moments: UAV 또는 storage texture
- depth/normal/material/roughness: read-only guide
- iteration index와 stride: constant/push constant

5×5 kernel은 pixel당 최대 25개 signal fetch와 guide fetch를 요구한다. Arithmetic보다 memory bandwidth와 texture cache behavior가 지배적이기 쉽다.

초기 iteration은 stride가 작아 group shared memory tile이 효과적이다. 예를 들어 8×8 thread group에 2-pixel halo를 포함해 guide와 signal을 적재하면 중복 fetch를 줄일 수 있다. 그러나 stride가 8 또는 16으로 커지면 필요한 halo가 너무 커져 shared memory 효율이 급격히 떨어진다.

실무적인 최적화 선택은 다음과 같다.

- early iterations만 shared memory 사용
- late iterations는 texture cache 기반 sparse fetch
- center guide를 register에 유지
- hard gate용 metadata를 먼저 읽고 radiance fetch를 생략
- variance가 낮은 tile은 subgroup ballot으로 pass-through
- invalid/out-of-range tap은 mirror clamp보다 명시적으로 reject
- diffuse/specular를 한 pass에 묶을지 signal별로 분리할지 bandwidth 기준으로 판단
- half precision accumulation은 overflow와 weight normalization 오차를 확인

Separable 1D pass 두 번으로 tap 수를 줄일 수 있지만, nonlinear edge weight가 각 방향과 중간 결과에 의존하므로 원래 2D joint filter와 완전히 동일하지 않다. 품질과 비용을 비교한 엔진 정책이 필요하다.

### 4.11 Memory layout과 resource contract

1080p 기준 대표적인 texture 크기는 다음과 같다.

- `RGBA16_FLOAT` radiance: 약 15.8 MiB
- `RG16_FLOAT` moments: 약 7.9 MiB
- `R16_FLOAT` variance: 약 4.0 MiB
- `R32_FLOAT` viewZ: 약 7.9 MiB
- `R10G10B10A2_UNORM` normal/roughness: 약 7.9 MiB
- `R16_UINT` material/surface ID: 약 4.0 MiB

Radiance와 moments를 각각 ping-pong하면 transient memory가 빠르게 커진다. Render graph에서는 다음 수명 분석이 중요하다.

- temporal history는 persistent resource
- À-trous intermediate는 transient resource
- diffuse/specular가 동시에 처리되지 않으면 alias 가능
- final iteration output은 후속 composite input과 alias 불가
- guide texture는 G-buffer와 공유하되 layout transition 최소화

예시적인 C++ 측 resource contract는 다음 정보를 명시해야 한다.

- signal domain: radiance, irradiance, visibility, occlusion
- color encoding/exposure domain
- normal encoding과 coordinate space
- depth convention과 projection parameters
- variance semantics와 precision
- checkerboard/full-resolution 여부
- valid rectangle과 dynamic resolution scale
- iteration count와 maximum stride
- diffuse/specular/shadow policy

Shader가 guide의 의미를 추측하게 만들기보다 render graph 단계에서 resource semantics를 명시하는 편이 디버깅과 API 이식에 유리하다.

### 4.12 API별 관점

- **OpenGL**: image load/store와 texture sampling을 조합하며 pass 사이에 적절한 memory barrier가 필요하다.
- **Vulkan**: sampled image와 storage image layout, descriptor set 재사용, pipeline barrier, transient image aliasing이 핵심이다.
- **Direct3D 12**: SRV/UAV state transition, UAV barrier, descriptor heap 관리, transient resource heap aliasing을 고려한다.
- **WebGPU**: storage texture format 제약, bind group layout, ping-pong texture, workgroup memory 한도와 브라우저별 shader compilation 특성이 중요하다.
- **CUDA**: surface/texture object를 이용한 2D locality와 kernel fusion이 가능하지만 graphics interop synchronization 비용을 함께 봐야 한다.

알고리즘은 API-agnostic하지만 resource state와 dispatch granularity는 backend마다 달라진다. 포트폴리오에서는 동일한 filtering abstraction을 여러 API에 매핑하는 구조가 graphics engineer의 설계 역량을 보여준다.

### 4.13 대표적인 failure mode

**Light leaking**

Depth/normal threshold가 느슨하거나 material identity가 없을 때 밝은 surface가 어두운 surface로 번진다.

**Dark halo**

Edge 근처에서 유효 tap 수가 비대칭적으로 줄고 normalization이 불안정할 때 경계 주변이 어두워질 수 있다.

**Residual noise islands**

Normal map 또는 material ID가 지나치게 세분화되어 support가 끊기면 noise가 작은 섬처럼 남는다.

**Overblurred specular**

Roughness와 virtual motion을 무시하고 diffuse와 같은 threshold를 사용하면 reflection detail이 사라진다.

**Stride skipping**

큰 À-trous stride가 얇은 geometry를 건너뛰며 우연히 반대편 surface를 참조할 수 있다. Surface identity와 depth gradient가 이를 방지해야 한다.

**Variance feedback instability**

필터된 variance가 너무 빠르게 작아지면 후반 pass가 갑자기 edge-sensitive해지고 frame 간 filter strength가 흔들린다. 반대로 variance floor가 너무 크면 계속 과도한 blur가 유지된다.

## 5. 내 관심 분야와 연결

### 실시간 렌더링과 ray-traced effect

사용자가 관심을 가진 real-time rendering, ray tracing, compute shader에서 À-trous 필터는 단순 post-process blur가 아니라 temporal state와 G-buffer guide를 결합하는 핵심 reconstruction stage다. PBR renderer에서 diffuse GI, glossy reflection, soft shadow를 각각 다른 signal policy로 운영하는 구조는 엔진 설계 관점에서 중요하다.

### C++ rendering pipeline

C++ 측에서는 pass 자체보다 resource semantics가 더 중요하다. Temporal accumulation이 만든 radiance, moments, variance, history length를 À-trous pass가 어떤 domain으로 받는지 명확해야 한다. Render graph, descriptor binding, transient memory aliasing, dispatch scheduling을 함께 설명할 수 있으면 단순 shader 구현을 넘어 pipeline ownership을 보여준다.

### GPU 최적화

À-trous는 fixed tap count 때문에 계산량을 예측하기 쉽지만, iteration stride가 증가할수록 cache locality가 달라진다. Early-pass shared memory, late-pass texture cache, subgroup early-out, guide-first rejection은 GPU architecture와 memory hierarchy를 이해하는 좋은 사례다.

### CFD 및 scientific visualization

CFD scalar field, pressure, temperature, density, vorticity magnitude에도 edge-aware multiscale filtering 관점을 적용할 수 있다. 이때 depth/normal 대신 다음 guide가 사용될 수 있다.

- material 또는 phase ID
- scalar gradient magnitude
- interface normal
- confidence/measurement uncertainty
- cell refinement level

단, scientific visualization에서는 필터가 물리적 discontinuity를 제거하거나 값을 보존하지 못하면 해석을 왜곡할 수 있다. 따라서 렌더링용 denoising과 simulation data smoothing을 엄격히 분리해야 한다. 화면에 표시되는 stochastic volume rendering noise는 필터링할 수 있지만, 원본 simulation field 자체를 동일한 규칙으로 변경해서는 안 된다.

### Level-set, voxel, semiconductor 구조

Level-set surface와 multi-material voxel에서는 surface identity와 material label이 매우 강한 edge guide가 된다. Oxide, silicon, metal, photoresist 사이를 hard gate로 구분하고, 동일 material 내부에서는 SDF gradient와 surface normal을 soft weight로 사용할 수 있다.

Doping heatmap이나 process visualization에서는 농도 변화가 실제 물리적 gradient인지 sampling noise인지 구분해야 한다. Material boundary를 넘지 않는 filter와 uncertainty-aware color reconstruction은 시각적 안정성을 높일 수 있지만, 정량 값을 보여주는 probe와 filtered display는 별도 resource로 유지해야 한다.

### WebGPU와 cross-platform graphics

WebGPU에서는 storage texture format과 workgroup memory가 제한적이므로 `RGBA16Float` 지원 여부, ping-pong binding, dynamic iteration uniform, workgroup 크기를 명확히 설계해야 한다. 동일 알고리즘을 Vulkan/DirectX/WebGPU backend에 구현할 때 resource abstraction과 shader permutation 관리가 좋은 포트폴리오 주제가 된다.

## 6. 머릿속에 남길 질문 3개

1. Variance가 큰 영역에서 luminance threshold를 넓히는 것이 noise 제거에는 유리하지만, 실제 lighting edge까지 blur하는 것을 어떤 geometry 및 material guide가 막아야 하는가?
2. À-trous stride가 커질수록 logical footprint는 넓어지지만 tap 수는 고정된다. 이 sparse sampling이 thin geometry와 high-frequency reflection에 만드는 failure mode는 무엇인가?
3. Diffuse GI, glossy reflection, shadow visibility, scientific scalar field에 동일한 edge-stopping function을 사용할 수 없는 이유는 무엇이며, signal policy는 어떤 항목을 분리해야 하는가?

## 7. graphics engineer 면접 질문 1개와 답변

**질문:** Variance-guided À-trous denoiser에서 variance와 depth/normal edge-stopping function은 각각 어떤 역할을 하며, threshold를 잘못 설정하면 어떤 artifact가 발생합니까?

**답변:** Variance는 현재 pixel의 signal uncertainty를 나타내며 luminance difference를 얼마나 noise로 허용할지 결정한다. Variance가 크면 luminance weight의 scale을 넓혀 더 많은 이웃을 사용하고, variance가 작으면 작은 차이도 실제 detail로 판단해 필터를 보수적으로 만든다. Depth와 normal edge-stopping function은 서로 다른 surface나 orientation 사이의 radiance mixing을 제한한다. Threshold가 너무 느슨하면 foreground/background light leaking, halo, specular smear가 생기고, 너무 엄격하면 유효 support가 부족해 residual noise와 temporal shimmer가 남는다. 또한 iteration stride가 커질수록 depth 변화 허용 범위와 surface identity 검증을 함께 조절해야 하며, diffuse와 specular는 roughness와 virtual motion 특성이 다르므로 별도 policy가 필요하다. GPU 관점에서는 25-tap 반복 pass가 bandwidth 중심이므로 early iteration의 shared-memory tiling, metadata-first rejection, transient ping-pong resource 관리가 주요 최적화 지점이다.

## 8. 포트폴리오 / 커리어 연결

À-trous denoiser를 포트폴리오에 넣을 때 결과 이미지 한 장보다 다음 구조를 보여주는 것이 더 강하다.

- temporal accumulation → moments/variance → À-trous filtering의 data flow
- diffuse/specular/shadow별 signal policy table
- depth, normal, material, roughness guide의 역할과 failure case
- iteration별 variance, valid tap count, history length debug view
- edge leakage와 residual noise의 비교 장면
- GPU timing, bandwidth, occupancy, cache behavior 분석
- `RGBA16F`, moments, variance, guide resource의 memory budget
- OpenGL/Vulkan/DirectX/WebGPU 중 둘 이상의 backend mapping
- dynamic resolution과 checkerboard input 처리 방식

면접에서는 “À-trous shader를 작성했다”보다 “왜 variance가 luminance threshold를 조절하고, 왜 material identity가 geometry guide를 보완하며, 왜 high-stride pass에서 shared memory가 불리해지는가”를 설명하는 것이 더 높은 수준의 역량을 보여준다.

게임 엔진·그래픽스 C++ 포지션에서는 다음 능력으로 연결된다.

- stochastic rendering의 통계적 특성 이해
- G-buffer guide와 post-processing pipeline 설계
- compute shader와 GPU memory hierarchy 최적화
- temporal/spatial artifact 진단
- render graph resource lifetime 및 synchronization 관리
- signal별 품질 정책과 디버그 도구 설계

Scientific visualization 포지션에서는 data fidelity와 display reconstruction을 분리해 설명하는 것이 중요하다. 원본 field는 보존하고, stochastic rendering output만 edge-aware filtering하는 구조를 제시하면 시각적 품질과 과학적 정확성의 경계를 이해하고 있음을 보여준다.

## 9. 내일 이어서 볼 개념

**Variance Stabilization Across À-Trous Scales and Filtered Moment Propagation**

오늘은 variance가 luminance edge-stopping scale을 제어한다는 점을 살펴봤다. 다음에는 color와 함께 moments/variance를 여러 À-trous iteration에 전달할 때 생기는 bias, neighbor correlation, variance collapse, uncertainty floor 문제를 다룬다. 특히 `Σ a_i² σ_i²` 형태의 근사, filtered moment propagation, scale별 variance floor, iteration confidence를 연결해 볼 것이다.

## 10. 참고 키워드

- À-trous wavelet transform
- edge-avoiding wavelet filter
- variance-guided filtering
- Spatiotemporal Variance-Guided Filtering, SVGF
- edge-stopping function
- cross-bilateral filter
- luminance variance
- temporal moments
- first/second raw moment
- depth gradient
- normal similarity
- material identity
- roughness-aware filtering
- diffuse/specular demodulation
- RELAX denoiser
- ping-pong resource
- compute shader tiling
- subgroup/wave early-out
- transient resource aliasing
- Dammertz et al., *Edge-Avoiding À-Trous Wavelet Transform for Fast Global Illumination Filtering*, HPG 2010
- Schied et al., *Spatiotemporal Variance-Guided Filtering: Real-Time Reconstruction for Path-Traced Global Illumination*, HPG 2017
- NVIDIA NRD RELAX
