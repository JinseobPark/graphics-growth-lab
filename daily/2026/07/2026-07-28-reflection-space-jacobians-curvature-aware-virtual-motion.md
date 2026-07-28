---
title: "Reflection-Space Jacobians and Curvature-Aware Virtual Motion"
date: "2026-07-28"
category: "Graphics"
tags: ["GPU", "Ray Tracing", "Specular", "Temporal Reprojection", "Jacobian", "Curvature", "PSR", "Motion Vector", "Compute Shader", "Memory Layout"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-07-28 - Reflection-Space Jacobians and Curvature-Aware Virtual Motion

## 1. 오늘의 개념

**Reflection-Space Jacobian**은 현재 화면의 작은 위치 변화가 반사 경로를 통과한 뒤 이전 프레임의 반사 좌표 변화로 어떻게 변환되는지를 나타내는 **국소 미분 매핑(local differential mapping)**이다.

일반적인 temporal reprojection은 한 pixel의 현재 좌표 `p`에 motion vector를 더해 이전 좌표를 찾는다.

`p_prev = p + motion`

이 식은 주변 pixel도 거의 같은 방향과 크기로 움직인다는 **국소 평행이동(local translation)**을 암묵적으로 가정한다. 그러나 곡면 거울, 굴곡진 금속, 강한 normal variation, PSR(Primary Surface Replacement) 경로에서는 반사 영상이 단순히 이동하지 않는다.

- 확대 또는 축소된다.
- 회전하거나 전단(shear)된다.
- 한 방향으로 길게 늘어난다.
- reflection fold 근처에서 방향이 뒤집히거나 correspondence가 끊긴다.

이때 현재 좌표에서 이전 좌표로 가는 함수 `F`를 다음처럼 본다.

`p_prev = F(p_current)`

한 pixel 주변의 변화는 2×2 Jacobian으로 근사할 수 있다.

`J = ∂F / ∂p`

motion vector가 **한 점의 이동량**을 설명한다면, Jacobian은 그 점 주변의 작은 pixel footprint가 이전 프레임에서 어떤 모양과 크기로 변형되는지를 설명한다.

오늘 다루는 핵심은 다음 두 가지다.

1. **Specular reflection의 virtual motion은 위치 변화뿐 아니라 국소 변형률을 가진다.**
2. **반사면 curvature는 reflected direction의 미분에 직접 들어가므로 temporal stability를 결정한다.**

## 2. 한 줄 핵심

**반사 영상의 motion vector는 한 pixel의 이동만 알려주지만, reflection-space Jacobian은 주변 footprint의 확대·축소·회전·전단까지 설명하며, 곡률이 큰 반사면일수록 이 국소 변형을 무시한 history reprojection이 쉽게 깨진다.**

## 3. 왜 중요한가

### Motion vector 하나로는 curved reflection을 충분히 표현할 수 없다

평면 거울에서 카메라와 reflected object가 부드럽게 움직이는 경우, 반사 영상은 일정 구간에서 거의 rigid motion처럼 보일 수 있다. 이 상황에서는 2D 또는 2.5D virtual motion vector만으로도 history를 비교적 안정적으로 추적할 수 있다.

반면 곡면에서는 인접한 두 primary ray가 서로 다른 surface normal을 만나고, 반사된 ray의 각도 차이가 크게 증폭될 수 있다. 결과적으로 화면에서 1 pixel 떨어진 두 점이 이전 프레임에서는 2 pixel, 5 pixel 또는 서로 반대 방향으로 벌어질 수 있다.

이때 단일 history sample만 가져오면 다음 문제가 나타난다.

- 반사 무늬가 표면 위를 흐르는 **specular swimming**
- 확대 영역에서 history가 부족해지는 **undersampling**
- 축소 영역에서 서로 다른 feature가 섞이는 **over-accumulation**
- 곡률 경계에서 발생하는 **reflection ghosting**
- convex/concave 전환이나 silhouette 부근의 **history foldover**

### Curvature는 reflected direction의 변화율을 결정한다

입사 방향을 `I`, 단위 normal을 `N`이라고 하면 이상적인 반사 방향은 다음과 같다.

`R = I - 2 × dot(I, N) × N`

이를 미분하면 reflected direction의 변화 `dR`에는 `dI`뿐 아니라 normal 변화 `dN`이 포함된다.

`dR = dI - 2 × [ (dot(dI, N) + dot(I, dN)) × N + dot(I, N) × dN ]`

평면에서는 주변 위치가 변해도 normal이 거의 일정하므로 `dN ≈ 0`이다. 곡면에서는 `dN`이 surface curvature와 연결되며, 작은 위치 변화가 reflected direction의 큰 변화로 확대될 수 있다.

즉 curvature는 단순한 shading detail이 아니라 다음을 결정하는 temporal geometry 정보다.

- reflection motion의 민감도
- virtual position의 국소 확대율
- history footprint의 방향성
- disocclusion threshold
- history confidence
- spatial filter radius와 anisotropy

### Jacobian은 reprojection의 신뢰도를 정량화할 수 있다

`J`의 singular value를 `σmax`, `σmin`이라고 하면 다음 정보를 얻을 수 있다.

- `σmax > 1`: 이전 프레임 footprint가 특정 방향으로 확대됨
- `σmin < 1`: 특정 방향으로 압축됨
- `σmax / σmin`: anisotropy 정도
- `det(J)`: 면적 변화와 orientation 변화
- `det(J) ≈ 0`: mapping이 접히거나 한 방향 correspondence가 붕괴하는 위험 구간

따라서 Jacobian은 단순한 추가 motion data가 아니라, temporal filter가 history를 얼마나 믿어야 하는지를 판단하는 **geometry-derived confidence signal**로 사용할 수 있다.

## 4. 구현 관점

### 4.1 Reflection mapping을 함수로 본다

현재 pixel `p`에서 시작한 camera ray가 primary reflector `A`를 만나고, reflected ray가 PSR 또는 secondary hit `B`를 찾는다고 하자.

현재 프레임의 virtual position을 `Xv_t(p)`, 이전 프레임의 대응 virtual position을 `Xv_{t-1}(p)`라고 하면 이전 screen coordinate는 다음과 같다.

`F(p) = Project(VP_prev, Xv_{t-1}(p))`

일반적인 motion vector는 `F(p) - p`만 저장한다. Reflection-space Jacobian은 이 함수의 screen-space 미분이다.

`J = [ ∂F.x/∂p.x  ∂F.x/∂p.y ]`

`    [ ∂F.y/∂p.x  ∂F.y/∂p.y ]`

PSR 경로가 여러 delta bounce를 통과한다면 전체 미분은 chain rule로 연결된다.

`J_total = J_projection · J_virtualization · J_delta_chain · J_primary_ray`

각 단계는 서로 다른 역할을 가진다.

- `J_primary_ray`: 인접 screen pixel이 camera ray direction을 얼마나 바꾸는가
- `J_delta_chain`: reflection/refraction event가 ray differential을 어떻게 변환하는가
- `J_virtualization`: physical PSR position을 virtual world-space position으로 어떻게 펼치는가
- `J_projection`: virtual position 변화가 이전 screen coordinate로 어떻게 투영되는가

실무에서는 이 전체 행렬을 완전한 analytic form으로 구현하지 않고, 일부 단계는 analytic derivative, 일부는 finite difference 또는 local buffer derivative로 구성할 수 있다.

### 4.2 반사 방향 미분과 curvature

Reflection operator를 `R(I, N)`이라고 하면 방향 differential은 입사 방향 differential과 normal differential을 모두 필요로 한다.

- `dI`: camera ray 또는 이전 bounce의 directional differential
- `dN`: surface normal field의 변화

`dN`은 tangent plane 위의 위치 변화와 curvature tensor의 관계로 생각할 수 있다.

`dN ≈ -S · dX_tangent`

여기서 `S`는 shape operator이며 principal curvature 정보를 포함한다. 정확한 differential geometry를 저장하지 않더라도, 실시간 renderer에서는 주변 world position과 normal 차이로 curvature proxy를 얻을 수 있다.

`κx ≈ length(N(x+1) - N(x)) / max(length(X(x+1) - X(x)), ε)`

`κy ≈ length(N(y+1) - N(y)) / max(length(X(y+1) - X(y)), ε)`

이 값은 정확한 principal curvature가 아니라 screen-sampled normal variation이다. 하지만 다음 용도로는 충분히 유용하다.

- virtual motion confidence 감소
- disocclusion threshold 확대
- high-curvature pixel의 history length 제한
- Jacobian 계산이 불안정한 구간 표시
- normal-map detail과 geometric curvature 분리 검증

중요한 점은 **shading normal의 고주파 변화와 실제 geometry curvature를 동일하게 취급하지 않는 것**이다. Normal map만으로 큰 curvature를 추정하면 virtual motion을 과도하게 불안정하게 만들 수 있다.

실무에서는 다음처럼 역할을 나누는 편이 안정적이다.

- geometric normal: motion, disocclusion, curvature baseline
- shading normal: BRDF orientation과 micro-detail
- roughness/normal variance: shading normal이 만드는 추가 lobe spread

### 4.3 Jacobian을 구하는 세 가지 방식

#### A. Analytic ray differentials

Camera ray에 screen x/y에 대한 origin과 direction derivative를 함께 전달하고, intersection·reflection·refraction마다 미분을 갱신한다.

장점:

- path geometry와 직접 연결된다.
- planar/curved surface 차이를 구조적으로 표현한다.
- footprint와 texture LOD 같은 다른 시스템에도 재사용 가능하다.

단점:

- animated/deformed geometry의 previous correspondence까지 포함하면 복잡도가 빠르게 증가한다.
- traversal payload와 register pressure가 커진다.
- shading normal과 geometric normal의 contract가 불명확하면 결과가 흔들린다.

#### B. Neighbor ray 또는 finite difference

현재 pixel과 인접 pixel에서 virtual position 또는 previous UV를 계산하고 차분한다.

`dFdx ≈ F(p + Δx) - F(p)`

`dFdy ≈ F(p + Δy) - F(p)`

`J = [dFdx, dFdy]`

장점:

- 구현 개념이 단순하고 analytic derivative가 없는 복잡한 reflector에도 적용 가능하다.
- 전체 path mapping의 실제 결과를 직접 측정한다.

단점:

- 추가 ray 또는 추가 virtual-path evaluation 비용이 크다.
- boundary에서 다른 surface/path를 읽으면 derivative가 폭발한다.
- low-resolution reflection buffer에서는 quantization noise가 커진다.

#### C. Motion/virtual-position buffer의 local derivative

이미 계산한 `previousUV`, virtual position, virtual normal buffer에서 인접 texel을 읽어 Jacobian을 추정한다.

장점:

- 별도 ray traversal 없이 compute pass에서 처리 가능하다.
- Vulkan, DirectX, OpenGL compute pipeline에 쉽게 배치할 수 있다.
- debug visualization이 쉽다.

단점:

- geometry boundary와 path topology change를 명시적으로 마스킹해야 한다.
- compute shader에는 fragment derivative가 자동 제공되지 않으므로 neighbor load, shared memory tile 또는 subgroup quad operation이 필요하다.
- 이전 UV가 이미 잘못 계산되었다면 Jacobian도 잘못된다.

실무적인 baseline은 **virtual motion buffer를 먼저 만들고, 2×2 quad 또는 3×3 neighborhood에서 local Jacobian과 confidence를 후처리하는 방식**이다.

### 4.4 Jacobian validation

인접 pixel의 derivative를 사용할 때는 같은 reflection correspondence인지 먼저 확인해야 한다.

다음 항목이 다르면 derivative sample을 invalid로 보는 편이 안전하다.

- primary object/material ID
- PSR object/material ID
- PSR bounce count
- reflection path signature
- virtual depth continuity
- geometric normal continuity
- reflector generation/revision ID

유효한 Jacobian이라도 수치적으로 위험할 수 있다.

- `abs(det(J)) < ε`: 거의 collapse된 mapping
- `det(J) < 0`: local orientation flip 또는 fold 가능성
- `σmax`가 매우 큼: 확대가 지나쳐 history sample density 부족
- `σmin`이 매우 작음: 여러 current pixel이 좁은 previous 영역으로 압축
- condition number `σmax / σmin`이 큼: 강한 anisotropy

`det(J) < 0`이 항상 물리적으로 틀렸다는 뜻은 아니다. 곡면 반사 mapping은 orientation을 뒤집을 수 있다. 그러나 단일 bilinear history lookup 관점에서는 높은 위험 신호이므로 confidence를 크게 낮추는 것이 합리적이다.

### 4.5 History confidence로 사용하는 방법

가장 저렴한 활용은 full Jacobian warp를 수행하지 않고 confidence만 조절하는 것이다.

예시적인 metric은 다음과 같다.

`scaleError = abs(log(max(σmax, ε))) + abs(log(max(σmin, ε)))`

`anisotropy = log(max(σmax / max(σmin, ε), 1))`

`Cj = exp(-ks × scaleError - ka × anisotropy)`

최종 specular history confidence는 다음 정보를 결합할 수 있다.

`Cspec = Cdepth × Cnormal × Cmaterial × Cmotion × Cjacobian × Ctopology`

이 방식은 reflection-space Jacobian을 별도 denoiser API 입력으로 직접 전달하지 않더라도, application-provided specular confidence 또는 reactive mask에 반영할 수 있다.

추가 정책 예시는 다음과 같다.

- `σmax`가 크면 history weight 감소
- `σmin`이 작으면 neighborhood clamping 강화
- anisotropy가 크면 isotropic filter radius 제한
- fold 위험이 있으면 history reset 또는 1~2 frame fast history만 허용
- high curvature이지만 mapping이 연속적이면 hard reject 대신 soft confidence 감소

### 4.6 GPU memory layout

Full 2×2 Jacobian을 저장하면 원소가 네 개다.

- `RGBA16_FLOAT`: 8 bytes/pixel
- 1920×1080: 약 15.8 MiB
- 3840×2160: 약 63.3 MiB

Temporal ping-pong까지 사용하면 비용이 두 배가 될 수 있으므로 대부분의 경우 Jacobian은 persistent history로 저장할 필요가 없다. 현재 frame의 motion/virtual G-buffer에서 계산해 temporal pass 직전에 소비하는 transient resource가 적합하다.

더 저렴한 표현은 다음과 같다.

- `RG16_FLOAT`: `σmax`, `σmin`
- `R16_FLOAT`: log area scale 또는 max stretch
- `R8_UNORM`: Jacobian-derived confidence
- `R8_UINT`: invalid/fold/topology flag
- motion vector derivative를 temporal pass에서 on-the-fly 계산

추천 baseline:

- `RG16_FLOAT`: virtual motion
- `R16_FLOAT`: virtual depth delta 또는 hit-distance guide
- `R8_UNORM`: specular confidence
- `R8_UINT`: path validity mask
- full Jacobian은 debug mode 또는 고품질 reflection path에서만 사용

Half precision은 대부분의 smooth mapping에서 충분하지만 `det(J) ≈ 0`인 구간에서는 상대 오차가 커진다. 따라서 determinant나 singular value를 직접 저장하기보다 clamp된 log scale 또는 confidence로 변환해 저장하는 편이 안정적이다.

### 4.7 Rendering pipeline 배치

한 가지 가능한 pipeline은 다음과 같다.

1. Ray tracing pass에서 physical hit, PSR data, virtual position, virtual normal, path signature 생성
2. Virtual motion pass에서 current/previous virtual correspondence 계산
3. Jacobian pass에서 neighbor virtual UV를 읽고 local derivative와 confidence 생성
4. Temporal denoising pass에서 depth·normal·material·Jacobian confidence 결합
5. Spatial filtering에서 curvature와 anisotropy를 edge-stopping에 반영
6. Debug overlay에서 stretch, determinant, fold mask, history length 시각화

Vulkan 또는 DirectX render graph에서는 virtual motion UAV write 이후 Jacobian compute read 사이의 resource barrier가 필요하다. Jacobian 결과가 temporal pass에서 UAV/SRV로 사용된다면 추가 transition도 명시해야 한다.

OpenGL에서는 image store 이후 `glMemoryBarrier` 범위를 정확히 설정해야 하며, compute tile에서 neighbor를 읽는다면 dispatch boundary와 texture clamp 정책을 분리해야 한다.

### 4.8 Debug visualization

Reflection temporal bug는 최종 color만 보면 원인을 찾기 어렵다. 다음 debug view가 특히 유용하다.

- virtual motion magnitude와 direction
- `σmax` heatmap
- `σmax / σmin` anisotropy
- `det(J)` sign
- curvature proxy
- PSR bounce count
- reflection path signature mismatch
- Jacobian confidence
- accumulated specular history length

카메라만 회전하는 정적 장면부터 시작해 planar mirror, convex mirror, concave mirror, animated reflector 순으로 검증하면 motion contract가 어느 단계에서 깨지는지 구분하기 쉽다.

## 5. 내 관심 분야와 연결

### Real-time rendering / game engine

Reflection-space Jacobian은 ray-traced reflection, glossy GI, specular denoising, temporal upscaling 사이의 contract를 설명하는 개념이다. 단순히 denoiser 설정을 조절하는 것이 아니라, engine이 어떤 geometry guide를 생성해야 하는지 보여준다.

특히 renderer architecture 관점에서는 다음 모듈의 경계가 중요하다.

- path tracer: virtual correspondence와 path identity 생성
- G-buffer system: geometric/shading normal 분리
- motion system: current/previous transform 관리
- render graph: transient Jacobian/confidence resource 관리
- denoiser: history validation과 accumulation
- debug system: mapping distortion 시각화

### C++ / Vulkan / DirectX / OpenGL

C++ 쪽에서는 virtual G-buffer descriptor, path signature, resource lifetime, frame index, previous transform snapshot을 명확히 관리해야 한다. GPU shader에서 수학을 정확히 구현해도 이전 frame data ownership이 틀리면 virtual motion은 안정될 수 없다.

API별 핵심은 비슷하다.

- Vulkan: image layout, pipeline barrier, subgroup quad operation
- DirectX 12: UAV barrier, resource state, Wave/Quad intrinsics
- OpenGL: image load/store synchronization, shared-memory tile
- CUDA/Warp: ray/path differential 또는 buffer-based Jacobian을 kernel pipeline으로 분리

### CFD / scientific visualization

곡면 반사 자체뿐 아니라 Jacobian 관점은 scientific visualization에도 확장된다.

- flow-map advection에서 local stretch와 compression 판단
- streamline reprojection의 screen-space footprint 추적
- volume rendering의 temporal sample warping
- deforming mesh의 feature correspondence
- slice/clip plane animation에서 history validity 판단

즉 **한 점의 velocity와 주변 field deformation은 다르다**는 원리는 fluid advection과 reflection reprojection 모두에 공통적이다.

### Semiconductor 3D visualization

반도체 구조는 thin layer, high-curvature corner, 반복 pattern, 강한 metallic response를 자주 포함한다. 이런 장면에서 specular history가 잘못 누적되면 얇은 gate, trench edge, rounded corner가 흔들리거나 번져 보일 수 있다.

Jacobian-derived confidence는 다음에 활용할 수 있다.

- thin metallic feature의 temporal ghosting 감소
- high-curvature corner의 history length 제한
- 반복 pattern 사이의 surface identity 검증
- LOD 또는 geometry revision 시 reflection history reset
- section view와 reflective material을 함께 사용할 때 virtual motion 안정화

## 6. 머릿속에 남길 질문 3개

1. **현재 pixel의 motion vector가 정확해도, 주변 reflection mapping의 Jacobian이 크게 변하면 왜 single-sample history reprojection은 실패하는가?**
2. **Geometric curvature와 normal-map variation을 temporal motion 관점에서 어떻게 분리해야 하는가?**
3. **Full 2×2 Jacobian을 저장하지 않고도 singular value, determinant, confidence 중 어떤 정보만 남기면 가장 높은 품질 대비 메모리 효율을 얻을 수 있는가?**

## 7. graphics engineer 면접 질문 1개와 답변

### 질문

곡면 반사에서 일반적인 2D motion vector만으로 temporal reprojection을 수행하면 ghosting과 swimming이 발생하는 이유를 설명하고, 이를 완화할 수 있는 GPU-friendly 방법을 제안해보세요.

### 답변

2D motion vector는 현재 pixel 하나가 이전 frame의 어느 좌표로 이동했는지만 표현한다. 이는 주변 pixel도 거의 같은 translation을 가진다는 가정에 가깝다. 하지만 곡면 반사에서는 인접 pixel이 다른 surface normal을 만나고, reflection operator의 normal derivative가 curvature에 따라 크게 달라진다. 따라서 reflection image는 이동뿐 아니라 확대·축소·회전·전단을 가진다.

이를 국소적으로 `p_prev = F(p_current)`라는 mapping으로 보고 `J = ∂F/∂p`를 계산하면 footprint deformation을 추정할 수 있다. GPU-friendly baseline은 ray tracing pass에서 virtual previous UV 또는 virtual position을 만들고, 후속 compute pass가 2×2 quad나 3×3 neighborhood를 읽어 finite difference Jacobian을 계산하는 방식이다.

Full Jacobian을 항상 저장할 필요는 없다. singular value 기반 stretch, anisotropy, determinant 위험도를 `R8_UNORM` confidence로 압축해 specular history weight에 곱할 수 있다. Surface ID, PSR path signature, virtual depth continuity가 깨지는 boundary에서는 derivative를 invalid 처리해야 한다. 이 방식은 analytic ray differential보다 저렴하면서도 curved reflection의 temporal instability를 motion-vector-only 방식보다 잘 감지한다.

## 8. 포트폴리오 / 커리어 연결

이 개념은 단순한 ray-traced reflection 구현보다 한 단계 높은 **temporal reconstruction system design** 역량을 보여준다.

포트폴리오에서는 다음 구조로 표현할 수 있다.

- planar/convex/concave reflector별 virtual motion 비교
- motion-only와 Jacobian-confidence 방식의 ghosting 비교
- curvature, stretch, anisotropy, fold mask debug view
- `RGBA16F` full Jacobian과 `R8` confidence 방식의 memory/performance 비교
- camera motion, reflector motion, reflected object motion을 분리한 failure analysis
- render graph resource lifetime과 barrier 설계

면접에서는 “반사 ray를 쏠 수 있다”보다 “반사 signal을 시간적으로 안정화하기 위해 어떤 geometry contract가 필요한가”를 설명하는 것이 graphics engineer로서 더 강한 차별점이 된다.

Nintendo, Unity, engine/rendering 팀 관점에서도 이 주제는 다음 능력을 함께 드러낸다.

- linear algebra와 differential reasoning
- GPU bandwidth와 transient resource 최적화
- ray tracing과 raster G-buffer의 연결
- temporal artifact debugging
- API-independent rendering architecture 설계

## 9. 내일 이어서 볼 개념

**Elliptical History Footprints and Jacobian-Aware Temporal Filtering**

오늘은 Jacobian이 reflection footprint의 변형을 설명하고 history confidence를 조절할 수 있다는 점을 다뤘다. 다음에는 2×2 Jacobian을 실제 history sampling에 사용해 previous-frame footprint를 ellipse로 해석하고, anisotropic history lookup·MIP selection·clamping radius를 결정하는 방법을 이어서 본다.

## 10. 참고 키워드

- Reflection-Space Jacobian
- Temporal Reprojection
- Specular Virtual Motion
- Primary Surface Replacement (PSR)
- Ray Differentials
- Path Differentials
- Shape Operator
- Principal Curvature
- Reflection Differential
- Virtual World Space
- Motion Vector
- 2.5D Motion
- Singular Value Decomposition (SVD)
- Determinant
- Condition Number
- Anisotropic Footprint
- History Confidence
- Disocclusion
- Reflection Fold
- Specular Swimming
- Compute Shader
- Subgroup Quad Operation
- Transient Resource
- NVIDIA NRD
- Homan Igehy, *Tracing Ray Differentials*
- Suykens & Willems, *Path Differentials and Applications*
