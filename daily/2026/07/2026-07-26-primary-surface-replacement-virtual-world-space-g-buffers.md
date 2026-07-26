---
title: "Primary Surface Replacement (PSR) and Virtual World-Space G-Buffers"
date: "2026-07-26"
category: "Graphics"
tags: ["GPU", "Ray Tracing", "Denoising", "Primary Surface Replacement", "Virtual World Space", "G-Buffer", "Specular", "Motion Vector", "Compute Shader", "Memory Layout"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-07-26 - Primary Surface Replacement (PSR) and Virtual World-Space G-Buffers

## 1. 오늘의 개념

**Primary Surface Replacement (PSR)**은 카메라 광선이 순수 거울과 같은 **delta surface**를 한 번 이상 통과한 뒤 처음 만나는 non-delta surface를, denoising과 temporal reconstruction 관점에서 새로운 primary surface처럼 취급하는 기법이다.

일반적인 reflection ray path를 다음과 같이 생각할 수 있다.

`Camera → Mirror₀ → Mirror₁ → ... → First Non-Delta Surface B → Lighting Path`

기존 G-buffer는 첫 번째 물리적 교차점인 `Mirror₀`의 depth, normal, motion vector, material을 저장한다. 그러나 화면에 실제로 보이는 noisy radiance의 구조와 움직임은 거울 표면보다 그 뒤에 반사되어 보이는 `B`의 geometry와 더 강하게 연결된다.

PSR은 `B`를 논리적인 primary hit으로 승격하고, 거울 경로를 접어 없앤 것처럼 보이는 **virtual world space**에서 다음 guide를 다시 만든다.

- virtual position / virtual viewZ
- virtual normal
- PSR roughness와 material identity
- virtual motion vector
- PSR 이후의 diffuse/specular hit distance

핵심은 단순히 secondary hit의 G-buffer를 복사하는 것이 아니다. **PSR surface의 material data와, mirror chain을 반영해 변환한 virtual geometry data를 결합한 새로운 G-buffer contract**를 만드는 것이다.

## 2. 한 줄 핵심

**PSR은 거울 자체가 아니라 거울 속 첫 non-mirror surface를 denoiser의 primary surface로 바꾸고, 그 surface를 virtual world space의 depth·normal·motion으로 표현해 reflection history가 실제 반사 영상의 구조를 따라가게 만든다.**

## 3. 왜 중요한가

Pure mirror reflection에서 primary G-buffer를 그대로 사용하면 denoiser가 잘못된 surface identity를 기준으로 판단한다.

예를 들어 정지한 평면 거울 속에서 character가 움직이는 장면을 생각하자.

- primary depth는 거울 깊이로 고정된다.
- primary normal은 거울 normal이다.
- primary motion vector는 0일 수 있다.
- 그러나 반사 영상의 edge, depth ordering, material, motion은 character를 따른다.

이 상태에서 primary G-buffer 기반 temporal accumulation을 수행하면 다음 문제가 발생한다.

- 움직이는 반사 물체의 history가 거울 표면에 붙는 **reflection sticking**
- 반사 영상 경계가 geometry edge로 인식되지 않는 **cross-surface bleeding**
- 거울 속 disocclusion을 감지하지 못하는 **specular ghosting**
- 서로 다른 reflected material 사이의 history 혼합
- mirror chain이 길어질수록 커지는 motion mismatch

PSR을 사용하면 denoiser는 거울을 사이에 둔 실제 장면을 직접 보는 것처럼 동작할 수 있다. reflected object의 virtual depth와 normal로 edge-stopping을 수행하고, virtual motion으로 history를 재투영하며, PSR material ID로 surface mixing을 제한한다.

다만 PSR은 모든 glossy reflection을 해결하는 일반 기법은 아니다. 하나의 대표 surface correspondence가 명확한 **delta 또는 near-delta path**에서 가장 강하다. Rough surface에서는 한 pixel이 넓은 BRDF lobe의 여러 방향을 통합하므로 단일 PSR point가 전체 신호를 대표하지 못한다.

## 4. 구현 관점

### 4.1 PSR은 어디에서 시작하고 어디에서 멈추는가

Path traversal 중 각 bounce를 다음처럼 분류한다.

- **Delta event**: 이상적인 mirror, 완전한 refraction처럼 방향이 사실상 하나로 결정되는 event
- **Non-delta event**: diffuse 또는 유한 폭의 glossy lobe처럼 여러 방향에 에너지가 분포하는 event

카메라의 첫 hit이 delta라면 path를 계속 추적하고, 처음 만나는 non-delta hit을 `B_psr`로 선택한다.

`A₀ = primary mirror hit`

`A₁ ... Aₙ = additional delta hits`

`B_psr = first non-delta hit`

실무에서 roughness threshold만으로 delta 여부를 결정하면 material tuning에 따라 PSR이 불안정하게 켜졌다 꺼질 수 있다. 따라서 renderer의 BSDF 분류와 동일한 기준을 사용해야 한다.

- ideal delta lobe 여부
- sampled lobe type
- transmission/reflection event
- roughness와 normal-map variance
- path throughput과 lobe probability
- 최대 PSR bounce 수

PSR validity가 frame마다 바뀌면 virtual G-buffer identity가 바뀌므로 history를 유지해서는 안 된다. `PSR_VALID`, `PSR_BOUNCE_COUNT`, `PSR_PATH_SIGNATURE` 같은 metadata가 필요하다.

### 4.2 물리적 hit과 virtual guide를 구분한다

PSR에서는 세 종류의 위치를 구분해야 한다.

1. **Physical primary position** `A₀`: 카메라가 처음 만난 거울 표면
2. **Physical PSR position** `B`: 실제 scene 안의 첫 non-delta hit
3. **Virtual PSR position** `Bᵥ`: 거울 chain을 제거한 것처럼 카메라 쪽 공간에 펼친 위치

Denoiser에 전달되는 것은 보통 `B` 자체가 아니라 `Bᵥ`에서 계산한 viewZ와 motion이다. Material과 roughness는 `B`에서 가져오지만, geometry guide는 virtual space에서 만들어진다.

이 분리는 매우 중요하다.

- `B`의 material은 reflected object의 identity를 제공한다.
- `Bᵥ`의 position은 화면에서 reflection이 보이는 위치와 parallax를 제공한다.
- mirror surface의 curvature와 animation은 `B → Bᵥ` 변환에 영향을 준다.

### 4.3 평면 거울의 reflection transform

평면이 단위 normal `N`과 plane equation `dot(N, X) + d = 0`으로 표현될 때, 점 `X`의 반사 위치는 다음과 같다.

`ReflectPoint(X) = X - 2 × (dot(N, X) + d) × N`

방향 벡터 `V`는 translation 항 없이 반사한다.

`ReflectVector(V) = V - 2 × dot(N, V) × N`

하나의 평면 거울에서는 physical PSR point `B`를 mirror plane에 대해 반사한 점이 virtual image position의 직관적인 표현이 된다.

여러 개의 delta reflector가 있다면 각 reflection transform을 path 순서에 맞게 합성한다. Position을 camera-visible virtual space로 펼칠 때와 PSR normal을 primary 쪽으로 되돌릴 때는 transform order를 명시적으로 관리해야 한다.

특히 normal은 PSR에서 primary 방향으로 path를 역추적하며 mirror transform을 적용한다.

`N_virtual = R₀(R₁(...Rₙ(N_psr)))`

행렬 표기와 row/column vector convention에 따라 실제 곱 순서는 달라지지만, 개념적으로는 **PSR에서 출발해 delta chain을 역순으로 접어 primary camera space까지 가져온다.**

### 4.4 Virtual position은 같은 pixel ray 위에 있어야 한다

Flat mirror의 이상적인 경우, virtual PSR point는 현재 pixel의 camera ray와 일치하는 위치에 나타난다. 따라서 denoiser용 proxy를 다음과 같이 생각할 수 있다.

`B_virtual = CameraPosition + PrimaryViewRay × virtualDistance`

`virtualDistance`는 단순한 camera-to-primary depth가 아니라 mirror chain을 따라 누적된 optical path와 reflector geometry를 반영한다.

정확한 planar reflection transform을 사용할 수 있다면 physical PSR point를 reflection plane들로 펼쳐 `B_virtual`을 얻는다. 일반적인 real-time denoiser에서는 보다 저렴한 형태로 다음 정보를 누적할 수 있다.

- camera에서 첫 mirror까지의 거리
- delta segment들의 path length
- 각 reflector에서의 curvature correction
- PSR의 virtualized distance

PSR viewZ는 `B_virtual`을 denoiser가 사용하는 view matrix로 변환해 얻는다.

`viewZ_psr = TransformToViewSpace(B_virtual).z`

여기서 현재 pixel UV와 virtual position projection이 크게 어긋난다면 transform order, matrix convention, normal orientation, ray direction 중 하나가 잘못되었을 가능성이 높다.

### 4.5 Virtual normal은 PSR normal을 mirror chain으로 되접은 결과다

Denoiser의 normal guide는 physical mirror normal이 아니라 PSR surface의 normal을 virtual space로 변환한 값이어야 한다.

`N_virtual = FoldNormalThroughDeltaChain(N_psr)`

이 normal은 다음 용도로 사용된다.

- spatial edge-stopping
- temporal normal validation
- local curvature estimation
- specular lobe tracking
- material boundary preservation

Normal map이 적용된 PSR surface라면 shading normal과 geometric normal의 역할을 분리할 필요가 있다.

- geometric normal: stable reprojection과 disocclusion
- shading normal: BRDF와 local signal shape

Denoiser에 지나치게 noisy한 shading normal을 제공하면 curvature가 과장되고 history rejection이 증가할 수 있다. 반대로 geometric normal만 제공하면 bump detail을 가로질러 blur가 발생할 수 있다. Engine에서는 normal encoding precision과 roughness anti-aliasing 정책을 함께 봐야 한다.

### 4.6 PSR motion vector는 virtual position의 frame-to-frame correspondence다

현재 frame의 virtual position을 `Bᵥ_t`, 이전 frame의 대응 virtual position을 `Bᵥ_{t-1}`이라고 하자.

`uv_prev = Project(VP_prev, Bᵥ_{t-1})`

`MV_xy = uv_prev - uv_current`

정확한 부호는 engine contract에 따라 달라지지만, 핵심은 `Bᵥ_{t-1}`을 올바르게 재구성하는 것이다.

이전 virtual position에는 다음 변화가 모두 포함되어야 한다.

- previous camera transform
- mirror object의 previous transform
- mirror plane normal의 변화
- PSR object의 previous transform 또는 deformation
- delta bounce chain의 topology 변화
- curved reflector의 local magnification 변화

현재 `B_virtual`을 이전 view-projection matrix로 단순 projection하는 방식은 camera motion만 반영한다. 움직이는 mirror나 reflected object가 있으면 잘못된 motion이 된다.

가능하다면 2D motion보다 **2.5D motion**을 사용한다.

`MV_z = viewZ_prev - viewZ_current`

2.5D motion은 screen position뿐 아니라 virtual depth 변화까지 제공하므로 dynamic PSR에서 history rejection이 더 안정적이다.

### 4.7 PSR의 hit distance 의미

PSR에서 가장 자주 혼동되는 값은 hit distance다.

Virtual viewZ는 camera부터 PSR이 보이는 virtual 위치까지의 geometry guide다. 반면 denoiser의 diffuse/specular hit distance는 **PSR을 새로운 primary surface로 보았을 때 그 이후 signal path의 거리**다.

즉 mirror chain까지의 거리를 noisy signal hit distance에 그대로 더하는 것이 아니다.

- virtual viewZ: camera와 PSR의 virtual geometry 관계
- lobe hitT: PSR에서 출발한 diffuse/specular sample이 다음 hit까지 이동한 거리

PSR 이후 diffuse와 specular radiance는 분리해야 하며, 각 lobe의 hitT도 해당 lobe 안에서 선택된 ray의 의미를 유지해야 한다. 이 contract가 깨지면 specular tracking과 spatial weight가 잘못된다.

### 4.8 Virtual G-buffer contract

PSR을 external denoiser 또는 engine denoising pass에 전달할 때 resource 의미를 명확히 고정해야 한다.

| Resource | PSR에서의 의미 |
|---|---|
| Normal + Roughness | virtual-space PSR normal + physical PSR roughness |
| ViewZ | `B_virtual`의 linear view-space Z |
| Motion Vector | virtual PSR position의 frame-to-frame motion |
| Material ID | physical PSR surface의 stable material identity |
| Diffuse/Specular Radiance | PSR surface에서 분리된 demodulated signal |
| Diffuse/Specular HitT | PSR 이후 각 lobe의 in-lobe hit distance |
| PSR Validity | 현재 pixel이 PSR contract를 사용하는지 여부 |
| Path Signature | bounce count, reflector identity, lobe chain의 compact signature |

PSR validity가 0인 pixel은 일반 primary G-buffer path를 사용한다. 동일 frame 안에서 PSR pixel과 non-PSR pixel이 섞일 수 있으므로 shader branch와 resource packing이 명확해야 한다.

### 4.9 Rendering pipeline 배치

대표적인 pipeline은 다음과 같다.

`Primary Ray Trace → Delta-Path Traversal → First Non-Delta Hit Selection → Virtual Position/Normal/Motion Construction → PSR Material Demodulation → Diffuse/Specular Signal Packing → Temporal Denoising → Spatial Filtering → Material Remodulation → Upscaling/TAA`

PSR guide 생성 위치는 크게 두 가지다.

**Ray tracing pass에서 직접 생성**

- physical path 정보를 이미 register에 보유한다.
- bounce transform과 PSR identity를 정확히 계산할 수 있다.
- 별도 reconstruction pass를 줄일 수 있다.
- output bandwidth가 증가한다.

**후속 compute pass에서 생성**

- path trace output을 compact하게 저장하고 guide 생성을 분리한다.
- debug와 algorithm 교체가 쉽다.
- delta-chain metadata read가 추가된다.
- 여러 full-resolution intermediate buffer가 필요할 수 있다.

Internal denoiser라면 PSR guide 생성과 temporal reprojection을 한 compute pass로 합쳐 virtual motion texture write를 피할 수 있다. External denoiser integration에서는 명시적인 resource contract가 더 중요하므로 별도 texture가 실용적이다.

### 4.10 GPU memory layout과 bandwidth

1920×1080 full resolution 기준 대략적인 단일 texture 비용은 다음과 같다.

- virtual normal + roughness `R10G10B10A2_UNORM`: 약 7.9 MiB
- virtual viewZ `R32_FLOAT`: 약 7.9 MiB
- 2.5D motion `RGBA16_FLOAT`: 약 15.8 MiB
- material ID `R16_UINT`: 약 4.0 MiB
- PSR validity / bounce metadata `R8_UINT`: 약 2.0 MiB

이 guide들만 약 37.6 MiB이며, previous-frame copy까지 유지하면 75 MiB 수준으로 증가할 수 있다. Diffuse/specular history와 moments, hit distance까지 포함하면 denoising subsystem의 VRAM과 bandwidth가 빠르게 커진다.

메모리 최적화 관점에서는 다음 선택지가 있다.

- virtual position을 저장하지 않고 viewZ와 camera ray로 reconstruct한다.
- PSR validity와 bounce count를 material ID 또는 metadata texture의 spare bit에 pack한다.
- 2D motion이면 `RG16_FLOAT`, 2.5D motion이면 API가 지원하는 compact 3-channel 또는 `RGBA16_FLOAT`를 선택한다.
- viewZ range가 제한적이면 `R16_FLOAT`를 고려하되 large scene precision을 검증한다.
- normal은 octahedral encoding으로 precision 대비 bandwidth를 최적화한다.
- PSR generation과 denoiser input packing을 fuse해 intermediate write/read를 제거한다.
- non-PSR pixel에서는 불필요한 delta-chain metadata를 쓰지 않는다.

Full-screen PSR guide pass는 arithmetic보다 memory bandwidth에 제한될 가능성이 높다. Transform 몇 번을 줄이는 것보다 resource 수와 round trip을 줄이는 것이 더 큰 효과를 낼 수 있다.

### 4.11 실패하기 쉬운 장면

#### Curved mirror

Flat mirror에서는 virtual transform이 하나의 rigid reflection으로 설명된다. Curved mirror에서는 pixel마다 local reflector frame과 magnification이 다르다. PSR normal과 viewZ만 대략 변환하면 temporal reprojection artefact가 남을 수 있다.

#### Animated reflector

Water, deforming metal, animated normal map은 reflector transform 자체가 시간에 따라 바뀐다. Primary motion과 PSR object motion만으로는 reflection motion을 설명할 수 없다.

#### Roughness transition

Delta와 non-delta 경계에서 PSR이 on/off되면 guide identity가 갑자기 바뀐다. Hysteresis, path signature validation, confidence fade가 필요하다.

#### Multiple mirror chain

Bounce 하나의 transform 오류가 뒤의 모든 virtual guide에 누적된다. Bounce order와 handedness, normal orientation을 debug할 수 있어야 한다.

#### Mixed pixel과 geometric edge

한 pixel footprint 안에 mirror와 non-mirror가 함께 포함되면 단일 PSR validity가 불충분할 수 있다. Temporal upscaler와 checkerboard sampling까지 고려하면 coverage 또는 confidence가 필요하다.

#### PSR miss

Delta chain 끝이 sky 또는 miss라면 finite PSR surface가 없다. 이 경우 virtual G-buffer 대신 reflected direction 기반 environment reprojection path를 사용해야 한다.

### 4.12 Debug visualization

PSR은 결과 영상만 보고 오류 원인을 찾기 어렵다. 다음 debug view가 실무적으로 유용하다.

- PSR validity mask
- delta bounce count
- reflector object ID chain hash
- physical primary normal vs virtual PSR normal
- primary viewZ vs virtual viewZ
- primary motion vs virtual motion
- virtual motion residual
- PSR material ID
- current/previous path signature mismatch
- flat-mirror analytic solution과의 error heatmap

가장 먼저 검증할 장면은 정지한 평면 거울, 움직이는 카메라, 움직이는 reflected object의 조합이다. 이 장면에서 virtual depth와 motion이 맞지 않으면 curved mirror로 확장해도 안정화되지 않는다.

## 5. 내 관심 분야와 연결

### Real-time rendering과 game engine

PSR은 ray-traced reflection 품질을 filter kernel 하나가 아니라 **G-buffer ownership과 coordinate-space design** 문제로 바라보게 한다. Vulkan, DirectX 12, Metal의 ray tracing pipeline 또는 ray query 기반 renderer에서 path tracing front-end와 denoiser interface를 설계할 때 직접 연결된다.

### C++ / RHI architecture

Engine에서는 PSR을 shader 내부 트릭으로 숨기기보다 resource semantic으로 명시하는 편이 좋다.

- `PrimaryGuideSet`
- `PSRGuideSet`
- `SignalHitDistanceSet`
- `TemporalMetadata`

이처럼 guide의 의미를 타입과 render-graph resource에 드러내면 backend별 Vulkan image layout, D3D12 resource state, Metal texture usage를 일관되게 관리할 수 있다.

### GPU compute와 memory layout

PSR은 계산 자체보다 full-resolution guide traffic이 병목이 되기 쉽다. Virtual position을 reconstruct할지, motion을 texture로 저장할지, denoiser pass와 fuse할지 결정하는 과정은 GPU architecture와 cache/bandwidth 이해를 요구한다.

### CFD / scientific visualization

PSR의 직접적인 사용처는 reflection denoising이지만, 개념은 scientific visualization에도 확장된다. 화면에 보이는 signal이 현재 rasterized surface가 아니라 다른 coordinate domain의 sample을 나타낼 때, filtering guide를 physical surface가 아닌 **signal-owning virtual domain**에서 구성해야 한다는 원칙이다.

예를 들어 reflection probe, portal visualization, mirrored inspection view, transformed slice view에서는 현재 display surface의 depth/normal보다 실제 data domain의 identity와 motion이 temporal reconstruction에 더 중요할 수 있다.

### Semiconductor 3D visualization

Wafer와 metal surface의 강한 reflection, inspection-style ray tracing, virtual camera view를 지원한다면 PSR은 reflected geometry의 temporal stability와 edge preservation에 연결된다. 특히 매우 얇은 pattern과 high-frequency geometry는 잘못된 virtual motion에서 ghosting이 즉시 드러난다.

### WebGPU

WebGPU에는 범용 hardware ray tracing 표준 기능이 아직 제한적일 수 있지만, precomputed mirror path, planar reflection, screen-space virtual view를 compute pipeline으로 구성할 때 동일한 G-buffer contract와 bandwidth 사고방식을 적용할 수 있다.

## 6. 머릿속에 남길 질문 3개

1. PSR에서 material과 roughness는 physical PSR hit에서 가져오면서 normal과 viewZ는 virtual space로 변환해야 하는 이유는 무엇인가?
2. 움직이는 mirror 속 움직이는 object를 정확히 추적하려면 current virtual point의 previous projection만으로 왜 충분하지 않은가?
3. Rough reflection에서 단일 PSR point의 대표성이 무너지는 이유를 BRDF lobe width와 pixel footprint 관점에서 어떻게 설명할 수 있는가?

## 7. graphics engineer 면접 질문 1개와 답변

### 질문

**Pure mirror reflection을 temporal denoising할 때 Primary Surface Replacement를 사용하는 이유와, denoiser에 전달해야 할 virtual G-buffer 정보를 설명해보세요.**

### 답변

일반 primary G-buffer는 카메라가 처음 만난 mirror surface의 depth, normal, motion을 담지만, 화면에 보이는 reflection radiance의 geometry와 motion은 mirror 뒤에 반사되어 보이는 surface를 따른다. Primary guide만 사용하면 reflected object의 edge와 disocclusion을 인식하지 못하고 history가 mirror에 붙는 ghosting이 발생한다.

PSR은 delta reflection chain 뒤의 첫 non-delta surface를 새로운 primary surface로 선택한다. Material과 roughness는 그 physical PSR hit에서 가져오고, position과 normal은 mirror chain을 펼친 virtual world space로 변환한다. Denoiser에는 virtual viewZ, virtual normal, PSR roughness, stable material ID, virtual motion vector, PSR 이후의 diffuse/specular radiance와 lobe hit distance를 제공한다.

Virtual motion은 previous camera뿐 아니라 mirror와 PSR object의 previous transform, bounce chain 변화까지 포함해야 한다. Flat mirror에서는 reflection transform으로 정확히 구성할 수 있지만, curved mirror와 rough surface에서는 단일 virtual point의 correspondence가 약해지므로 curvature-aware correction, 낮은 history confidence, 또는 non-PSR fallback이 필요하다.

## 8. 포트폴리오 / 커리어 연결

PSR을 포트폴리오에 포함하면 단순 ray tracing 효과 구현보다 높은 수준의 문제 해결 능력을 보여줄 수 있다.

강조할 수 있는 포인트는 다음과 같다.

- primary surface와 signal-owning surface를 구분한 설계
- delta path traversal과 first non-delta hit classification
- virtual position·normal·motion의 coordinate transform
- physical material data와 virtual geometry guide의 결합
- 2D/2.5D motion vector 선택과 temporal validation
- flat, curved, animated mirror의 failure mode 분석
- PSR path signature와 history invalidation
- external denoiser integration contract
- fused compute pass와 explicit G-buffer 사이의 bandwidth trade-off
- Vulkan/D3D12/Metal RHI resource state와 render graph 연결

Graphics engineer 면접에서는 PSR을 “거울을 더 잘 그리는 기법”으로만 설명하기보다, **denoiser가 어떤 surface를 primary identity로 보아야 하는가를 재정의하는 virtual G-buffer architecture**로 설명하는 것이 좋다.

## 9. 내일 이어서 볼 개념

**Reflection-Space Jacobians and Curvature-Aware Virtual Motion**

Flat mirror의 rigid reflection transform이 curved reflector에서 왜 깨지는지, reflector normal derivative와 screen-space Jacobian이 reflection magnification, motion divergence, history confidence에 어떤 영향을 주는지 이어서 본다.

## 10. 참고 키워드

- Primary Surface Replacement (PSR)
- Virtual World Space
- Virtual G-Buffer
- Delta Reflection Chain
- First Non-Delta Hit
- Virtual Position
- Virtual Normal
- Virtual ViewZ
- PSR Motion Vector
- 2.5D Motion Vector
- Reflection Transform
- Mirror Plane Matrix
- Path Signature
- Stable Material ID
- Signal Ownership
- Specular Denoising
- Reflection Reprojection
- Curvature-Aware Motion
- NVIDIA Real-Time Denoisers (NRD)
- REBLUR / RELAX
- Ray Tracing Gems II

Primary references:

- [NVIDIA Real-Time Denoisers (NRD)](https://github.com/NVIDIA-RTX/NRD)
- [Ray Tracing Gems II](https://developer.nvidia.com/ray-tracing-gems-ii)
