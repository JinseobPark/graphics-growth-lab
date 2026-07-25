---
title: "Specular Virtual Motion and Reflection Reprojection"
date: "2026-07-25"
category: "Graphics"
tags: ["GPU", "Ray Tracing", "Denoising", "Specular", "Reflection Reprojection", "Motion Vector", "Hit Distance", "Temporal Accumulation", "Compute Shader", "Memory Layout"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-07-25 - Specular Virtual Motion and Reflection Reprojection

## 1. 오늘의 개념

**Specular Virtual Motion**은 거울이나 낮은 roughness 표면에 보이는 반사 영상의 움직임을, 반사 표면 자체의 motion vector가 아니라 **가상 반사 공간(virtual reflected space)**의 움직임으로 추적하는 개념이다.

일반적인 G-buffer motion vector는 primary surface의 현재 위치가 이전 프레임의 어디에 있었는지를 나타낸다. 그러나 반사 신호는 primary surface에 붙어 있는 texture가 아니다. 정지한 거울 위에서도 카메라가 움직이거나 거울 속 물체가 움직이면 반사 영상은 별도로 이동한다.

따라서 specular history를 안정적으로 재사용하려면 다음 두 motion model을 구분해야 한다.

- **Surface motion-based reprojection**: 반사 표면 자체를 따라 history를 재투영한다.
- **Virtual motion-based reprojection**: 반사되어 보이는 가상 물체의 위치를 따라 history를 재투영한다.

실무에서는 두 후보를 roughness, hit distance, curvature, parallax, secondary-object motion confidence에 따라 선택하거나 혼합한다.

## 2. 한 줄 핵심

**거울 반사의 history는 거울 표면이 아니라 거울 속 가상 세계를 따라 움직여야 하며, 이를 위해 specular hit distance와 virtual motion을 별도 reprojection signal로 관리한다.**

## 3. 왜 중요한가

Primary motion vector만 사용하면 denoised reflection이 표면에 붙어 따라오는 **reflection sticking**, 카메라 회전 시 반사가 늦게 끌려오는 **specular ghosting**, 얇은 highlight가 흔들리는 **shimmering**이 발생한다.

특히 낮은 roughness에서는 반사 영상의 spatial frequency가 높기 때문에 1 pixel 이하의 reprojection 오차도 쉽게 드러난다. 반대로 높은 roughness에서는 하나의 pixel이 넓은 reflected-direction distribution을 통합하므로 단일 virtual point가 신호 전체의 motion을 정확히 표현하지 못한다.

즉 specular temporal accumulation은 단순히 “정확한 motion vector 하나”를 찾는 문제가 아니다. 표면 motion과 가상 motion 중 어떤 모델이 현재 pixel의 reflected radiance를 더 잘 설명하는지 평가하는 **model selection problem**에 가깝다.

이 구분은 denoising뿐 아니라 DLSS Ray Reconstruction 같은 reconstruction system에도 중요하다. 외부 denoiser가 reflection motion을 이해하려면 application이 specular motion vector를 직접 제공하거나, specular hit distance와 camera matrix를 통해 virtual geometry를 재구성할 수 있게 해야 한다.

## 4. 구현 관점

### 4.1 Primary surface motion은 무엇을 추적하는가

현재 pixel의 primary world position을 `X_t`, 이전 프레임의 동일한 표면 위치를 `X_{t-1}`이라고 하자.

Primary motion candidate는 개념적으로 다음과 같이 얻는다.

`uv_prev_surface = Project(VP_prev, X_{t-1})`

`mv_surface = uv_current - uv_prev_surface`

여기서 `X_{t-1}`은 rigid object transform, skinning, deformation, camera transform을 반영한 이전 위치다. 이 motion은 primary surface의 identity를 추적하는 데는 적합하지만, 그 표면 위에 보이는 reflection content의 identity를 보장하지는 않는다.

### 4.2 Reflected hit point와 virtual point

현재 primary position에서 specular ray를 추적해 secondary hit point `P_hit`과 hit distance `h`를 얻었다고 하자.

`P_hit = X_t + R_t × h`

여기서 `R_t`는 reflected ray direction이다.

중요한 점은 `P_hit`을 일반 카메라로 그대로 이전 프레임에 projection하는 것만으로는 거울 영상의 screen-space 위치를 완전히 설명할 수 없다는 것이다. 반사 영상은 reflector를 통과해 보이는 **virtual image**이기 때문이다.

평면 거울과 secondary hit data를 명확히 알고 있다면 mirror plane에 대해 hit point를 반사한 가상 점을 만들 수 있다.

`P_virtual = P_hit - 2 × (dot(N_mirror, P_hit) + d_mirror) × N_mirror`

그 뒤 이전 프레임의 reflector transform과 secondary object transform을 반영한 `P_virtual_prev`를 이전 view-projection matrix로 투영한다.

`uv_prev_virtual = Project(VP_prev, P_virtual_prev)`

이 방식은 평면 거울과 tracked secondary geometry에서 가장 해석하기 쉽다. 하지만 arbitrary glossy surface, noisy hit distance, probabilistic ray sampling에서는 완전한 secondary correspondence를 얻기 어렵다.

### 4.3 Hit-distance 기반 virtual tracking proxy

Real-time denoiser는 더 저렴한 근사로 primary surface와 camera view vector를 이용한 virtual proxy를 사용한다.

`X_virtual = X_t - V_t × h_tracking`

여기서 `V_t`는 surface에서 camera를 향하는 view vector이며, 부호는 engine convention에 따라 달라질 수 있다. 이 점은 실제 reflected object position이 아니라 **반사 영상의 screen-space parallax를 설명하기 위한 tracking proxy**다.

`h_tracking`은 raw hit distance를 그대로 쓰기보다 다음 요소를 고려해 안정화한다.

- probabilistic lobe selection으로 비어 있는 pixel의 hit-distance reconstruction
- temporal accumulation된 hit distance
- firefly 또는 outlier 제거
- roughness와 view angle에 따른 hit-distance normalization
- sky 또는 miss pixel의 별도 처리

Raw hit distance가 frame마다 크게 흔들리면 virtual motion 자체가 noise가 되어 reflection이 끓어 보인다.

### 4.4 Surface candidate와 virtual candidate를 동시에 평가한다

Temporal pass는 보통 두 개의 previous coordinate를 만든다.

`uv_s = uv_prev_surface`

`uv_v = uv_prev_virtual`

각 위치에서 history를 읽은 뒤 별도의 validation을 수행한다.

- previous depth와 current expected depth
- normal과 roughness
- material 또는 object identity
- hit distance consistency
- reflection lobe compatibility
- disocclusion 여부
- scene revision과 lighting revision

개념적인 confidence는 다음과 같다.

`C_surface = C_depth × C_normal × C_material × C_primaryMotion`

`C_virtual = C_hit × C_roughness × C_curvature × C_parallax × C_secondaryMotion`

최종 history는 단순한 고정 lerp보다 confidence에 따라 candidate를 선택하거나 정규화된 weight로 결합하는 편이 안전하다.

`H = (C_surface × H_surface + C_virtual × H_virtual) / max(C_surface + C_virtual, epsilon)`

두 candidate가 서로 다른 geometry를 가리키면 blending이 double image를 만들 수 있으므로, confidence 차이가 큰 경우에는 더 강한 candidate 하나를 선택하는 정책이 유리하다.

### 4.5 Roughness가 virtual motion의 사용량을 결정한다

낮은 roughness에서는 반사 방향이 좁고 mirror-like behavior가 강하므로 하나의 virtual point가 signal motion을 비교적 잘 설명한다.

높은 roughness에서는 여러 reflected direction이 섞이므로 하나의 hit point나 virtual point가 대표성을 잃는다. 이때는 surface motion의 비중을 높이거나 history length를 줄이며 spatial filtering에 더 의존한다.

개념적인 virtual history amount는 다음과 같이 생각할 수 있다.

`A_virtual = pow(1 - roughness, k) × C_hit × C_curvature × C_parallax`

- `roughness → 0`: virtual motion 비중 증가
- curvature 증가: planar mirror assumption 신뢰도 감소
- hit-distance variance 증가: virtual motion 신뢰도 감소
- extreme parallax: 작은 world-space 오차가 큰 screen-space 오차로 확대

Roughness threshold 하나로 surface와 virtual motion을 갑자기 전환하면 temporal popping이 생길 수 있으므로 연속적인 confidence가 더 안정적이다.

### 4.6 Curved reflector의 한계

단순 virtual point 모델은 평면 거울에서 가장 잘 맞는다. Concave 또는 convex reflector에서는 surface normal이 위치마다 달라지고 reflected rays가 수렴하거나 발산한다.

이 경우 동일한 hit distance라도 screen-space magnification이 달라진다. 즉 reflector는 사실상 lens처럼 작동한다.

대응 방식은 다음과 같다.

- local normal derivative로 curvature를 추정한다.
- curvature가 크면 virtual history amount를 낮춘다.
- reflection footprint와 ray cone 정보를 함께 사용한다.
- planar proxy 대신 curvature-adjusted virtual depth를 사용한다.
- history validation threshold를 curved surface에서 별도로 조절한다.

Curvature 보정이 불완전한 상태에서 긴 virtual history를 유지하는 것보다 confidence를 낮추고 빠르게 재수렴시키는 편이 시각적으로 안전하다.

### 4.7 Dynamic reflected object

Specular hit distance와 camera matrix만으로는 camera-driven parallax는 추정할 수 있지만, reflected secondary object 자체의 animation을 완전히 알 수는 없다.

Dynamic reflection을 정확히 추적하려면 다음 중 하나가 필요하다.

- secondary hit의 object ID와 previous transform
- secondary hit position의 explicit previous-world position
- application-generated specular motion vector
- 실패 가능성을 반영한 낮은 history confidence와 빠른 rejection

Primary object는 정지해 있고 mirror 속 character만 움직이는 장면이 대표적인 검증 사례다. Primary motion만 보면 motion이 0이지만 specular motion은 0이 아니다.

### 4.8 Sky와 infinite-distance reflection

Environment map이나 sky miss는 유한한 hit point가 없다. 매우 큰 hit distance를 억지로 저장하면 FP16 precision과 reprojection 안정성이 깨진다.

이 경우에는 finite virtual position 대신 reflected direction을 이전 camera orientation으로 변환해 angular reprojection을 계산하는 편이 적합하다.

- translation에는 거의 반응하지 않는다.
- camera rotation과 environment rotation에는 반응한다.
- local reflection geometry와 다른 validation path가 필요하다.

따라서 hit-distance buffer에는 miss sentinel 또는 validity bit를 함께 관리해야 한다.

### 4.9 Rendering pipeline

대표적인 흐름은 다음과 같다.

`Specular Ray Trace → Hit-Distance Reconstruction → Surface Motion Candidate → Virtual Motion Candidate → Candidate Validation → Roughness/Curvature-Based Selection → Specular Temporal Accumulation → Variance Update → Spatial Filtering`

External reconstruction system에 전달할 경우에는 중간 결과를 다음 형태로 export할 수 있다.

- explicit specular motion vector
- 또는 specular hit distance + camera matrices

Internal denoiser라면 virtual motion generation과 temporal accumulation을 하나의 compute pass로 합쳐 full-resolution motion texture write를 생략할 수 있다.

### 4.10 GPU resource와 memory layout

1920×1080 기준 대표적인 resource 비용은 다음과 같다.

- specular motion vector `RG16_FLOAT`: 약 7.9 MiB
- specular hit distance `R16_FLOAT`: 약 4.0 MiB
- virtual history confidence `R8_UNORM`: 약 2.0 MiB
- secondary object ID `R16_UINT`: 약 4.0 MiB

External denoiser integration에서는 `RG16_FLOAT` specular motion vector가 명확한 contract를 제공한다. 반면 engine 내부 pass라면 motion을 register에서 계산해 바로 history sampling에 사용함으로써 write/read bandwidth를 줄일 수 있다.

`R16_FLOAT` hit distance는 메모리 절감에 유리하지만 scene scale이 크면 far distance에서 quantization이 커진다. 다음 기준을 확인해야 한다.

- world unit과 meter의 대응
- maximum trace distance
- camera-relative rendering 사용 여부
- hit distance normalization 여부
- miss sentinel encoding

Motion vector는 API보다 convention이 더 중요하다.

- current-to-previous인지 previous-to-current인지
- pixel unit인지 normalized UV인지
- Y축 방향이 위인지 아래인지
- jitter가 포함되는지
- render resolution과 output resolution 중 어느 공간인지

Vulkan, DirectX, Metal, WebGPU 모두 같은 수학을 사용할 수 있지만, 이 contract가 불명확하면 specular history는 즉시 불안정해진다.

### 4.11 Compute shader 관점

이 pass는 arithmetic보다 texture fetch와 history validation이 지배적이다.

- surface와 virtual candidate를 같은 thread에서 계산해 G-buffer fetch를 공유한다.
- 두 history candidate를 모두 읽기 전에 roughness와 validity로 early classification한다.
- low-roughness tile과 high-roughness tile의 branch divergence를 관찰한다.
- candidate selection 결과를 debug buffer로 출력할 수 있게 한다.
- motion generation과 temporal accumulation을 fuse할 때 register pressure와 occupancy를 함께 본다.
- external API에 motion buffer를 전달해야 한다면 compact `RG16_FLOAT`를 우선 검토한다.

필수 debug view는 다음과 같다.

- surface motion
- virtual motion
- 두 motion의 screen-space 차이
- hit distance와 hit-distance variance
- virtual history amount
- selected candidate
- curvature confidence
- final history length

### 4.12 흔한 실패 패턴

1. Specular history에 primary surface motion만 사용한다.
2. Actual secondary hit point를 mirror transform 없이 일반 camera로 projection한다.
3. Noisy raw hit distance를 virtual depth로 바로 사용한다.
4. Roughness가 높은 pixel에서도 mirror-like virtual motion을 강제한다.
5. Curved reflector에서 planar assumption을 그대로 유지한다.
6. Dynamic secondary object motion을 무시하면서 긴 history를 유지한다.
7. Sky miss를 매우 큰 finite hit distance로 처리한다.
8. Jittered matrix와 non-jittered motion vector를 혼합한다.
9. Motion vector unit과 direction convention을 engine과 denoiser 사이에서 다르게 해석한다.

## 5. 내 관심 분야와 연결

### Real-time rendering과 game engine

Specular motion은 ray tracing subsystem만의 문제가 아니라 render graph 전체의 data contract다. G-buffer, ray tracing pass, temporal denoiser, upscaler가 motion convention, roughness encoding, hit-distance unit, camera matrices를 공유해야 한다.

C++ engine architecture에서는 `SpecularReprojectionInputs`와 같은 명시적인 구조로 resource와 convention을 묶는 편이 좋다. 단순 texture handle 목록보다 다음 metadata가 중요하다.

- coordinate space
- motion direction
- distance scale
- validity encoding
- previous-frame ownership
- reset condition

### Vulkan / DirectX / Metal / WebGPU

Internal denoiser는 compute shader 안에서 virtual motion을 생성해 bandwidth를 줄일 수 있다. 외부 denoiser나 upscaler integration에서는 export texture와 resource barrier가 추가된다.

Vulkan과 DirectX에서는 async compute overlap과 transient resource aliasing을 고려할 수 있다. Metal에서는 texture format과 heap reuse를, WebGPU에서는 storage texture format, bind group 수, half precision 지원 범위를 확인해야 한다.

### CFD와 scientific visualization

이 개념은 “화면에 보이는 신호의 motion이 proxy geometry의 motion과 다를 수 있다”는 일반 원리로 확장된다.

예를 들어 volume rendering에서 entry surface는 고정되어 있어도 내부 scalar feature가 이동하면 temporal history는 entry point가 아니라 sampled feature의 motion을 따라야 한다. 반사 표면과 virtual image의 관계가 volume proxy와 내부 physical feature의 관계와 유사하다.

### Semiconductor 3D visualization

Glossy wafer, metal layer, thin-film stack을 ray-traced reflection으로 표시할 때 primary layer motion만으로는 reflected structure를 안정적으로 추적할 수 없다. Geometry revision ID, secondary material ID, reflected hit distance를 함께 관리하면 공정 step 변경 후 잘못된 reflection history가 남는 문제를 줄일 수 있다.

### Computer vision과 reprojection

Surface correspondence와 appearance correspondence를 구분한다는 점은 optical flow와도 연결된다. Geometry가 정지해 있어도 view-dependent appearance는 이동할 수 있으며, specular flow는 Lambertian optical flow 가정을 위반한다.

## 6. 머릿속에 남길 질문 3개

1. 정지한 평면 거울의 primary motion vector가 0이어도 camera movement에 따라 specular virtual motion이 0이 아닐 수 있는 이유는 무엇인가?
2. 낮은 roughness에서는 virtual motion이 유리하지만 높은 roughness에서는 surface motion 또는 짧은 history가 더 안전한 이유는 무엇인가?
3. Specular hit distance만으로 camera parallax는 추적할 수 있어도 dynamic reflected object의 motion을 완전히 복원하기 어려운 이유는 무엇인가?

## 7. graphics engineer 면접 질문 1개와 답변

### 질문

왜 ray-traced reflection의 temporal reprojection에 primary surface motion vector만 사용하면 안 되며, specular virtual motion은 어떻게 구성하는가?

### 답변

Primary motion vector는 거울이나 glossy surface 자체의 frame-to-frame correspondence만 표현한다. 하지만 reflection radiance는 view-dependent signal이므로 reflected object와 camera parallax에 따라 primary surface와 다른 screen-space motion을 가진다. 따라서 specular temporal pass는 surface-motion candidate와 virtual-motion candidate를 별도로 만든다.

Virtual candidate는 specular hit distance, view vector, previous camera matrix를 이용한 virtual tracking proxy 또는 planar mirror에 대해 secondary hit point를 반사한 virtual point로 구성할 수 있다. 두 candidate는 depth, normal, roughness, hit distance, curvature, material identity로 검증하고, 낮은 roughness와 낮은 curvature에서는 virtual motion의 비중을 높인다. 높은 roughness, noisy hit distance, curved reflector, untracked dynamic secondary object에서는 virtual confidence를 낮추고 surface motion 또는 짧은 history로 fallback한다.

## 8. 포트폴리오 / 커리어 연결

이 개념은 단순 denoising filter 구현보다 **temporal signal ownership과 motion model 설계 능력**을 보여준다.

포트폴리오에서 가치 있는 설명 포인트는 다음과 같다.

- primary motion과 specular motion을 분리한 이유
- hit distance를 tracking signal로 사용하는 방식
- planar, curved, dynamic reflection에서의 failure mode
- roughness 기반 candidate selection
- full-resolution motion buffer와 fused compute pass의 bandwidth trade-off
- debug visualization을 통한 ghosting 원인 분석
- external denoiser contract와 engine RHI integration

Graphics engineer 면접에서는 “reflection이 왜 표면에 붙어 보이는가”를 surface correspondence, view-dependent appearance, secondary motion, temporal validation 관점으로 설명할 수 있어야 한다. 이는 game engine rendering, real-time ray tracing, reconstruction, scientific visualization 모두에 연결되는 고급 실무 주제다.

## 9. 내일 이어서 볼 개념

**Primary Surface Replacement (PSR) and Virtual World-Space G-Buffers**

Pure mirror 또는 여러 delta bounce 뒤의 첫 non-mirror surface를 새로운 primary surface처럼 다루고, virtual position·normal·viewZ·motion vector를 재구성하는 방법을 이어서 본다.

## 10. 참고 키워드

- Specular Virtual Motion
- Reflection Reprojection
- Surface Motion-Based Reprojection
- Virtual Motion-Based Reprojection
- Specular Motion Vector
- Specular Hit Distance
- Reflected Hit Point
- Virtual Image
- Planar Mirror Transform
- Roughness-Aware History
- Curvature-Aware Reprojection
- Secondary Object Motion
- Reflection Sticking
- Specular Ghosting
- Hit-Distance Reconstruction
- Temporal Validation
- Primary Surface Replacement
- NVIDIA NRD / REBLUR
- DLSS Ray Reconstruction
- Ray Tracing Gems II Chapter 49

Primary references:

- [NVIDIA Real-Time Denoisers (NRD)](https://github.com/NVIDIA-RTX/NRD)
- [NVIDIA Streamline DLSS Ray Reconstruction Programming Guide](https://github.com/NVIDIA-RTX/Streamline/blob/main/docs/ProgrammingGuideDLSS_RR.md)
- [Ray Tracing Gems II](https://developer.nvidia.com/ray-tracing-gems-ii)
