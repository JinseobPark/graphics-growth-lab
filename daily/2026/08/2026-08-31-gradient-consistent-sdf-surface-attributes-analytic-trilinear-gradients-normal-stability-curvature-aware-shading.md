---
title: "Gradient-Consistent SDF Surface Attributes: Analytic Trilinear Gradients, Normal Stability, and Curvature-Aware Shading"
date: "2026-08-31"
category: Graphics
tags: [GPU, Rendering, SDF, Level Set, Implicit Surface, Trilinear Interpolation, Analytic Gradient, Surface Normal, Curvature, Hessian, Eikonal Equation, Sparse Volume, CUDA, Vulkan, Compute Shader, Memory Layout, C++]
level: intermediate
---

# [Daily Graphics Growth] 2026-08-31 - Gradient-Consistent SDF Surface Attributes: Analytic Trilinear Gradients, Normal Stability, and Curvature-Aware Shading

## 1. 오늘의 개념
어제는 **Hybrid SDF Intersection Refinement**를 통해 ray를

`x(t) = o + t d`

로 두고 `phi(x(t)) = 0`의 root를 bracketed secant/bisection으로 좁혀, 현재 reconstructed field의 zero set에 대해 sub-voxel 수준의 안정적인 hit point를 얻는 방법을 봤다.

오늘의 질문은 그 다음이다.

> **교점 위치가 정확해졌다면, 그 점에서 shading normal·gradient magnitude·curvature 같은 surface attribute도 같은 geometry를 설명하고 있는가?**

Implicit surface

`F(x, y, z) = 0`

에서 regular point의 surface normal 방향은

`n = normalize(∇F)`

이다. 따라서 SDF/level-set renderer에서 normal은 별도의 mesh attribute가 아니라 **scalar reconstruction의 derivative**다.

이 말은 곧 다음 contract를 뜻한다.

- intersection은 어떤 scalar field reconstruction에서 찾았는가?
- normal은 그 **같은 reconstruction의 gradient**인가?
- gradient가 sparse brick boundary나 LOD 경계를 넘을 때도 같은 field epoch를 보는가?
- curvature는 그 reconstruction이 제공할 수 있는 미분 연속성보다 더 높은 차수의 정보를 요구하고 있지 않은가?

특히 regular grid에서 가장 흔한 **trilinear interpolation**은 scalar 값 자체는 cell boundary에서 연속인 `C0` reconstruction이지만, 그 analytic gradient는 일반적으로 cell boundary에서 불연속이다. 즉 zero set의 위치는 연속적으로 보이더라도 normal이 cell 경계를 지날 때 갑자기 변할 수 있고, 이 변화는 diffuse보다 specular highlight에서 훨씬 크게 드러난다.

오늘은 이 문제를 **analytic trilinear gradient, finite-difference gradient, interpolated gradient, sparse brick halo, gradient magnitude, Hessian/curvature**의 관점에서 정리한다.

## 2. 한 줄 핵심
> SDF surface attribute의 안정성은 `normalize(∇phi)` 한 줄의 문제가 아니라, **intersection과 동일한 scalar reconstruction·좌표계·LOD·epoch에서 derivative를 정의하고, trilinear field의 gradient discontinuity와 second-order 한계를 명시적으로 관리하는 문제**다.

## 3. 왜 중요한가
### 3.1 정확한 hit point와 불안정한 normal은 동시에 존재할 수 있다
전날의 root refinement가 매우 정확해도 shading normal이 다른 reconstruction에서 계산되면 두 단계가 서로 다른 surface를 설명하게 된다.

예를 들어 intersection은 trilinear `phi`에서 찾았는데 normal은 주변 6개 trilinear sample의 central difference로 계산할 수 있다. 이 central difference는 시각적으로 더 smooth할 수 있지만, 엄밀히는 **현재 hit를 만든 trilinear polynomial의 exact derivative가 아니다**.

반대로 같은 8개 corner value에서 analytic gradient를 계산하면 intersection과 derivative가 수학적으로 일치하지만, cell boundary에서 gradient가 jump할 수 있다.

즉 production renderer에는 다음 trade-off가 있다.

- **reconstruction consistency**: hit와 normal이 같은 field를 정확히 설명
- **normal continuity**: frame/cell 이동에서 shading이 부드럽게 유지

둘은 trilinear reconstruction에서 자동으로 동시에 얻어지지 않는다.

### 3.2 Trilinear scalar는 연속이지만 gradient는 일반적으로 연속이 아니다
Local cell coordinate를 `(u,v,w) ∈ [0,1]^3`라 하고 8개 corner sample을 `phi_ijk`라 하자.

Trilinear interpolant는

`phi(u,v,w) = Σ phi_ijk B_i(u) B_j(v) B_k(w)`

이며

`B_0(s)=1-s`, `B_1(s)=s`

이다.

인접 cell이 shared vertex 값을 사용하면 scalar value는 경계에서 이어지므로 `C0`다. 하지만 각 cell의 x-direction derivative는 서로 다른 x-edge difference에서 만들어지므로 경계를 건널 때 일반적으로 같은 값을 갖지 않는다.

결과적으로 다음 artifact가 생길 수 있다.

- specular highlight crawling / popping
- grazing-angle normal flicker
- curvature heatmap의 cell pattern
- thin layer sidewall의 banding
- camera가 움직일 때 voxel-cell 구조가 shading에 드러나는 현상

### 3.3 Gradient normalization은 field quality 문제를 숨길 수 있다
Exact SDF는 smooth region에서 Eikonal property

`|∇phi| = 1`

을 만족한다.

하지만 실제 simulation/visualization field는 다음 이유로 쉽게 깨진다.

- level-set advection
- CSG/smooth boolean
- resampling
- coarse LOD
- quantization
- interpolation
- redistancing 주기 부족

Shading에서는 `normalize(gradient)`만 사용하면 방향은 얻을 수 있지만 `|∇phi|`의 이상을 완전히 버린다. 이 magnitude는 오히려 다음을 알려주는 좋은 diagnostic이다.

- distance-property degradation
- near-singular/flat gradient
- LOD transition 품질
- curvature estimate 신뢰도
- measurement confidence

따라서 **normal vector와 raw gradient magnitude는 의미가 다르다**.

### 3.4 Curvature는 normal보다 훨씬 더 민감하다
Normal은 1차 derivative를 사용하지만 curvature는 2차 derivative, 즉 **Hessian**에 의존한다.

Implicit surface `F=0`에서 `g=∇F`, Hessian을 `H_F`라 하면 한 가지 sign convention에서 mean curvature는

`H_mean = (|g|^2 trace(H_F) - g^T H_F g) / (2 |g|^3)`

형태로 표현할 수 있다. Normal orientation convention에 따라 부호는 반대가 될 수 있다.

문제는 trilinear field가 cell 내부에서 각 축에 대해 linear이므로

`∂²phi/∂u² = ∂²phi/∂v² = ∂²phi/∂w² = 0`

이라는 점이다. Mixed derivative는 존재할 수 있지만 pure second derivative는 0이고, cell boundary에서는 first derivative 자체가 jump한다.

즉 **trilinear reconstruction은 second-order geometry를 안정적으로 표현하기 위한 매끄러운 basis가 아니다**. Exact SDF에서 흔히 쓰는 `curvature ≈ div(normal)` 또는 `Laplacian(phi)` 직관을 discrete trilinear field에 무비판적으로 적용하면 cell-scale artifact가 생길 수 있다.

### 3.5 Sparse residency에서는 normal도 data-lifetime 문제다
Fine brick의 scalar value는 resident하지만 gradient를 위한 neighboring brick/halo가 아직 resident하지 않을 수 있다. 또는 hit는 fine LOD로 찾았는데 normal sample 중 하나가 coarse fallback을 사용할 수도 있다.

이 경우 단순 memory miss가 아니라

> **교점과 surface attribute가 서로 다른 reconstructed surface를 참조하는 semantic mismatch**

가 된다.

따라서 sparse renderer에서 normal stability는 filtering 문제뿐 아니라 **residency, LOD, epoch, halo ownership** 문제다.

## 4. 구현 관점
### 4.1 Trilinear interpolant의 analytic gradient
Cell 내부 local coordinate `(u,v,w)`에서 x-direction derivative는 4개의 x-edge difference를 bilinear하게 섞은 형태다.

`∂phi/∂u =`

`(1-v)(1-w)(phi100-phi000)`

`+ v(1-w)(phi110-phi010)`

`+ (1-v)w(phi101-phi001)`

`+ vw(phi111-phi011)`

`∂phi/∂v`, `∂phi/∂w`도 같은 구조로 얻을 수 있다.

이 derivative의 중요한 특성은 **현재 trilinear scalar와 정확히 일치한다는 것**이다. Intersection solver가 동일한 8개 corner sample의 trilinear reconstruction을 사용했다면, 이 gradient는 그 zero set의 true local normal이다.

수치적으로는 corner 값을 이미 register/shared memory에 가지고 있다면 추가 global-memory fetch 없이 multiply-add 중심으로 계산할 수 있다.

### 4.2 Grid-space gradient를 바로 normalize하면 안 되는 경우
Voxel spacing이 `(hx, hy, hz)`라면 local/index-space derivative를 world-space derivative로 바꿀 때 축별 scale이 들어간다.

`∇_world phi = (∂phi/∂u / hx, ∂phi/∂v / hy, ∂phi/∂w / hz)`

보다 일반적으로 grid-to-world affine transform의 linear part를 `A`라 하면

`∇_world phi = A^{-T} ∇_grid phi`

로 본다.

따라서 anisotropic voxel이나 non-uniform transform에서 index-space gradient를 먼저 normalize한 뒤 transform하면 방향이 틀릴 수 있다. **metric을 반영한 뒤 normalize**해야 한다.

또 non-uniform scale이 들어간 field는 scalar value 자체가 더 이상 world-space exact distance가 아닐 수 있으므로, normal transform correctness와 SDF distance-property correctness는 별개의 문제다.

### 4.3 Analytic trilinear gradient의 장점과 약점
**장점**
- intersection reconstruction과 정확히 일치
- 같은 8 corner를 재사용하면 bandwidth가 작음
- voxel spacing/transform을 명시적으로 반영하기 쉬움
- gradient magnitude를 raw 상태로 얻을 수 있음

**약점**
- cell boundary에서 gradient discontinuity
- trilinear cell pattern이 specular에 드러날 수 있음
- second-order attribute에 부적합
- hardware trilinear texture sample 하나만 사용하던 path라면 8 corner를 직접 읽는 구조로 바뀌며 instruction/load pattern이 달라질 수 있음

즉 “analytic이라서 항상 더 예쁘다”가 아니라 **geometry-consistent하지만 piecewise-smooth**한 선택이다.

### 4.4 Central difference of reconstructed field
GPU volume rendering에서 전통적으로 많이 쓰는 방식은 hit point 주변을 ±h만큼 offset해

`gx ≈ [phi(x+h ex) - phi(x-h ex)] / (2h)`

형태로 gradient를 구하는 것이다.

장점은 sample footprint가 cell 양쪽을 걸치면서 analytic trilinear gradient의 cell-boundary jump를 어느 정도 평균내기 때문에 시각적으로 더 smooth할 수 있다는 점이다.

하지만 중요한 차이가 있다.

- `h`가 reconstruction scale과 비슷하면 gradient는 사실상 **추가 smoothing kernel을 적용한 derivative**가 된다.
- hit point를 만든 exact trilinear polynomial의 derivative와는 다르다.
- 3축 central difference는 일반적으로 여러 scalar texture sample을 추가로 요구한다.

따라서 central difference는 단순 approximation error가 아니라 **다른 effective reconstruction을 선택하는 것**으로 이해하는 편이 좋다.

### 4.5 Precomputed/interpolated gradient field
다른 전략은 voxel vertex마다 gradient를 미리 계산한 뒤, shading 시 그 gradient vector를 trilinear interpolation하는 것이다.

이 방식의 장점은 shared vertex gradient가 동일하면 cell boundary에서 vector field가 연속적으로 이어져 normal continuity가 좋아질 수 있다는 점이다. GPU에서는 gradient volume을 별도 3D texture로 두면 하나의 vector texture sample로 surface normal candidate를 얻는 구조도 가능하다.

대신 다음 비용과 semantic 차이가 있다.

- scalar field 외에 gradient volume residency 필요
- field update마다 gradient 갱신 필요
- scalar `phi`와 gradient field의 epoch가 어긋날 수 있음
- interpolated gradient는 일반적으로 **원래 trilinear scalar의 exact derivative가 아님**
- normalized normal만 저장하면 `|∇phi|` diagnostic을 잃음

즉 memory를 써서 continuity와 sampling cost를 줄이는 대신 scalar-gradient consistency contract가 추가된다.

### 4.6 Higher-order reconstruction은 다른 해법이다
Trilinear reconstruction의 근본 한계는 scalar가 `C0`이고 gradient continuity가 보장되지 않는다는 점이다.

더 높은 품질이 필요하면 cubic/B-spline 계열처럼 **`C1` 이상 연속성**을 제공하는 reconstruction을 사용할 수 있다. 그러면 scalar와 analytic gradient를 같은 higher-order basis에서 얻어 geometry consistency와 normal continuity를 동시에 개선할 수 있다.

대신 support가 넓어지고 다음 비용이 생긴다.

- 더 많은 neighboring sample
- brick halo 확대
- sparse residency dependency 증가
- 더 많은 ALU/register
- reconstruction/filter kernel 설계 복잡도 증가

따라서 higher-order reconstruction은 단순 “normal filter”가 아니라 **field representation 자체의 변경**이다.

### 4.7 Gradient magnitude를 confidence signal로 본다
Shading용 unit normal은

`n = g / max(|g|, epsilon)`

으로 만들 수 있지만 raw `|g|`는 버리지 않는 것이 유용하다.

Exact SDF 기대값 `1`과 비교해

`eikonalError = ||g| - 1|`

를 보면 field quality를 정량화할 수 있다.

이 값은 다음과 연결할 수 있다.

- normal confidence
- root refinement confidence
- curvature validity
- redistancing 필요성
- coarse/fine LOD transition detector
- debug heatmap

특히 `|g|`가 매우 작은 지점은 implicit function의 regular point 조건에서 멀어지는 영역이므로 normal direction 자체가 수치적으로 불안정할 수 있다.

### 4.8 Sparse brick boundary와 halo
Central difference나 precomputed voxel gradient는 neighboring sample이 필요하다. Brick-local storage라면 border에서 다음 정책이 필요해진다.

- duplicated halo/ghost voxel
- neighbor brick lookup
- parent/coarse fallback
- boundary one-sided derivative

각 선택은 normal continuity와 memory overhead를 바꾼다.

**Duplicated halo**는 hot shading path를 단순하게 만들지만 brick update 시 halo synchronization이 필요하다. **Neighbor lookup**은 storage duplication은 줄이지만 pointer/handle indirection과 sparse miss 가능성이 커진다. **Coarse fallback**은 availability는 높이지만 fine hit + coarse normal이라는 LOD mismatch를 만들 수 있다.

그래서 scalar residency와 별도로

`AttributeValidity = { fieldEpoch, lodLevel, haloState }`

같은 semantic 상태를 생각하는 것이 중요하다.

### 4.9 Intersection과 normal의 LOD/epoch contract
전날 root refinement state가

`{ tLo, tHi, phiLo, phiHi, fieldHandle, fieldEpoch, lodLevel }`

을 유지했다면 surface attribute 단계도 같은 identity를 이어받는 편이 일관적이다.

개념적으로는

`SurfaceHit = { position, t, fieldHandle, fieldEpoch, lodLevel }`

`SurfaceAttributes = { normal, gradMagnitude, curvature?, validity }`

처럼 볼 수 있다.

여기서 핵심은 normal 계산이 **새로운 임의의 residency decision을 독립적으로 수행하지 않는 것**이다. Hit가 fine level의 zero set에서 나왔는데 normal만 parent field로 올라가면 silhouette와 lighting이 서로 다른 geometry를 설명할 수 있다.

필요하다면 “fine normal unavailable → entire surface query를 coarse representation으로 재평가”처럼 consistency를 우선하는 정책과 “hit는 유지하고 visually smooth fallback normal 사용”처럼 continuity를 우선하는 정책을 명확히 구분해야 한다.

### 4.10 Curvature: Hessian을 쓰기 전에 reconstruction order를 본다
Implicit curvature는 Hessian과 gradient를 통해 계산할 수 있지만, 2차 derivative는 1차 derivative보다 noise와 quantization에 훨씬 민감하다.

특히 trilinear interpolant는 cell 내부에서 pure second derivative가 0이므로, exact-SDF identity를 가정한

`curvature ∝ Laplacian(phi)`

같은 shortcut은 그대로 사용할 수 없다. Trilinear field의 Laplacian은 cell interior에서 구조적으로 퇴화하며, boundary에서는 derivative jump가 존재한다.

Curvature가 정말 필요한 경우에는 다음을 서로 다른 목표로 구분할 필요가 있다.

- **geometric curvature**: reconstructed zero set의 differential geometry
- **visual curvature signal**: cavity/edge accent를 위한 안정적인 low-frequency attribute
- **simulation curvature**: mean-curvature flow나 surface-tension term에 들어가는 numerical quantity

세 경우는 허용 가능한 smoothing, bias, precision이 다르다.

OpenVDB 같은 level-set library가 gradient, Laplacian, mean/Gaussian/principal curvature용 stencil을 별도로 제공하는 이유도 second-order quantity가 단순 normal 계산보다 넓은 neighborhood와 더 강한 numerical contract를 필요로 하기 때문이다.

### 4.11 Curvature-aware shading에서는 scale이 attribute의 일부다
Curvature는 sampling resolution에 매우 민감하므로 “curvature 값”만 저장하면 의미가 부족하다. 어떤 stencil radius, voxel size, smoothing scale에서 계산했는지가 중요하다.

그래픽스 관점에서는 high-frequency raw curvature를 그대로 specular color에 넣기보다 다음처럼 **scale-aware diagnostic**으로 보는 것이 안정적이다.

- convex/concave heatmap
- sidewall/corner 강조
- adaptive shading detail
- feature-size estimate
- normal smoothing 강도 조절
- measurement visualization

즉 curvature는 scalar 하나가 아니라

`CurvatureSample = { value, scale, confidence, fieldEpoch, lodLevel }`

에 가까운 정보다.

### 4.12 GPU memory layout: 저장할 것인가, 재구성할 것인가
3D volume에서 gradient를 별도 저장하면 memory footprint가 크게 늘어난다.

예를 들어 scalar `R16F`가 voxel당 2 byte라면 gradient를 `RGB16F`로 추가할 경우 6 byte가 더 들어가 scalar만 저장할 때보다 전체 working set이 크게 증가한다. Sparse volume에서는 resident brick 수와 cache behavior에 직접 영향을 준다.

반대로 on-the-fly analytic gradient는 8 corner scalar를 읽어 derivative를 계산하므로 storage는 작지만 ALU와 load pattern이 늘어난다.

따라서 대표적인 선택은 다음과 같다.

**Value-only field**
- storage 최소
- analytic/finite-difference gradient를 runtime 계산
- simulation update와 attribute consistency가 단순

**Value + raw gradient field**
- shading/sample cost 감소 가능
- memory/residency 증가
- field-gradient epoch synchronization 필요

**Value + packed normal field**
- 가장 compact한 shading attribute 가능
- gradient magnitude/curvature source 정보 손실
- geometry field와 normal field가 독립 representation이 됨

Graphics engineer 관점에서 중요한 것은 “어떤 format이 더 빠른가”보다 **bandwidth, ALU, residency, update frequency, semantic consistency의 전체 cost model**이다.

### 4.13 Shader pipeline 관점
Raster/mesh pipeline에서는 vertex normal interpolation이 익숙하지만, implicit renderer에서는 surface attribute가 **hit 이후 동적으로 생성되는 geometry output**이다.

Ray/compute path를 개념적으로 나누면

1. hierarchy / sphere-tracing traversal
2. bracketed intersection refinement
3. field-consistent gradient evaluation
4. optional curvature/confidence evaluation
5. material shading

순서가 된다.

Wavefront 구조라면 curvature가 필요한 hit만 별도 queue로 분리해 second-order work를 제한할 수 있고, megakernel에서는 gradient/Hessian 계산이 live register와 texture dependency를 늘릴 수 있다.

또 screen-space `ddx/ddy`는 volume의 object/world-space `∇phi`를 의미하지 않는다. Fragment derivative는 neighboring pixel invocation의 변화량을 나타내므로 implicit field gradient를 대체하는 개념이 아니다. Compute/ray tracing에서는 screen-space derivative availability 자체도 pipeline/model에 따라 다르다.

### 4.14 C++ resource contract
C++ 엔진에서는 scalar field와 derived attribute를 서로 독립 texture로만 취급하면 version mismatch를 놓치기 쉽다.

개념적으로는

`FieldSnapshot`
- field resource handle
- transform / voxel metric
- epoch
- LOD policy
- reconstruction mode

`SurfaceAttributePolicy`
- analytic trilinear / central difference / interpolated gradient
- derivative scale
- curvature scale
- fallback rule

를 분리하면 rendering backend가 CUDA/Vulkan/WebGPU 중 무엇이든 **같은 semantic contract**를 유지하기 쉽다.

특히 external-memory interop에서는 semaphore가 execution ordering을 보장해도 “gradient texture가 어느 phi epoch에서 파생되었는가”까지 자동으로 보장하지 않는다. Derived resource versioning은 application-level responsibility다.

### 4.15 Profiling에서 볼 지표
Normal/curvature pipeline은 FPS 하나만으로 비교하기 어렵다. 다음 지표가 architecture를 더 잘 설명한다.

- scalar fetches per hit
- gradient evaluation cost per hit
- extra gradient-volume bandwidth
- gradient cache hit rate
- `|∇phi|-1` distribution
- cell-boundary normal angular jump
- frame-to-frame normal angular variance
- specular temporal variance
- halo miss / fallback count
- hit LOD vs normal LOD mismatch count
- curvature confidence / invalid sample ratio
- fine brick residency 증가량

특히 **normal angular variance와 specular temporal variance**를 함께 보면 “수치적으로 약간 다른 normal”이 실제 화면 안정성에 얼마나 큰 영향을 주는지 확인하기 좋다.

## 5. 내 관심 분야와 연결
Semiconductor process emulation/visualization에서 level-set·SDF 기반 geometry를 직접 렌더링한다면 오늘의 개념은 단순 shading 품질을 넘어 geometry interpretation과 연결된다.

첫째, **thin film / sidewall visualization**이다. Oxide, PR, TiN, W 같은 얇은 layer의 sidewall이 voxel cell을 따라 이동할 때 analytic trilinear normal의 discontinuity가 specular banding으로 드러날 수 있다. 반대로 과도하게 smooth한 normal은 실제 sharp process feature를 시각적으로 둔화시킬 수 있다. 따라서 geometry-consistent normal과 visually stable normal의 trade-off를 설명할 수 있어야 한다.

둘째, **measurement와 rendering의 consistency**다. 전날 refined intersection으로 trench depth나 layer thickness를 측정하면서 normal은 별도 smoothed field에서 얻는다면, position measurement는 정확하지만 sidewall angle은 다른 surface를 기준으로 계산될 수 있다. `position`, `normal`, `gradient magnitude`가 같은 field snapshot에 속한다는 contract가 중요하다.

셋째, **CUDA simulation → Vulkan/WebGPU visualization** 구조다. Simulation이 `phi`를 GPU에서 갱신하고 renderer가 zero-copy로 읽는 경우 gradient를 미리 bake할지, renderer가 on-the-fly로 재구성할지 결정해야 한다. Precomputed gradient는 rendering 비용을 줄일 수 있지만 derived buffer 갱신과 external synchronization, residency를 추가한다. On-the-fly analytic gradient는 field와의 consistency가 강하지만 shading hot path의 bandwidth/ALU가 늘어난다.

넷째, **sparse representation**이다. NanoVDB-like hierarchy나 custom sparse brick/ColumnStack 구조에서 scalar payload는 sparse하게 유지하면서 gradient/curvature metadata를 어디까지 resident하게 할지 결정해야 한다. Normal은 first-order local data지만 curvature는 더 넓은 support가 필요하므로 동일한 residency 정책을 그대로 적용하기 어렵다.

다섯째, **Marching Cubes와 direct implicit rendering 비교**다. Marching Cubes mesh는 vertex normal을 scalar gradient에서 한 번 생성한 뒤 raster pipeline에서 interpolate할 수 있고, direct implicit rendering은 각 ray hit에서 gradient를 다시 평가한다. 동일 source field라도 temporal stability, cache behavior, attribute consistency, memory footprint가 다르다. 이 차이를 설명할 수 있으면 representation-level graphics engineering 역량을 보여줄 수 있다.

## 6. 머릿속에 남길 질문 3개
1. **Intersection은 trilinear field에서 정확히 찾았는데 normal은 central difference로 계산한다면, 두 결과가 서로 다른 effective reconstruction을 설명하게 되는 이유는 무엇인가?**
2. **Analytic trilinear gradient가 수학적으로 정확한데도 cell boundary에서 specular flicker가 생길 수 있는 이유는 무엇이며, 이를 단순 normalization으로 해결할 수 없는 이유는 무엇인가?**
3. **Curvature가 필요할 때 `Laplacian(phi)`를 바로 사용하기 전에 scalar reconstruction의 continuity order와 SDF property를 확인해야 하는 이유는 무엇인가?**

## 7. graphics engineer 면접 질문 1개와 답변
### 질문
**Grid SDF를 ray marching해서 얻은 hit point의 normal을 GPU에서 계산해야 합니다. Analytic trilinear gradient, central difference, precomputed gradient texture 중 무엇을 선택하겠습니까? 정확도·shading stability·memory bandwidth·sparse residency 관점에서 비교해 주세요.**

### 답변
먼저 세 방법이 단순히 같은 gradient를 다른 속도로 계산하는 방식이 아니라 **서로 다른 reconstruction contract를 갖는다**고 설명할 수 있다.

**Analytic trilinear gradient**는 hit를 만든 trilinear scalar의 exact derivative이므로 geometry consistency가 가장 강하다. 이미 8 corner value를 갖고 있다면 추가 global fetch 없이 계산할 수도 있다. 하지만 trilinear scalar는 `C0`이고 gradient는 cell boundary에서 일반적으로 불연속이므로 specular highlight가 cell을 따라 flicker할 수 있다.

**Central difference**는 hit 주변의 scalar 값을 여러 번 sampling해서 derivative를 근사한다. Sample footprint가 경계를 걸치기 때문에 normal이 더 smooth해질 수 있지만, 이는 현재 trilinear cell polynomial의 exact derivative가 아니라 추가 filtering이 들어간 effective gradient다. 따라서 texture fetch가 늘고, `h` 선택에 따라 bias와 smoothness가 달라진다.

**Precomputed gradient texture**는 runtime shading cost를 줄이고 shared vertex gradient를 interpolation해 continuity를 좋게 만들 수 있다. 대신 scalar field 외에 3-component volume을 유지해야 하므로 memory footprint와 sparse residency 비용이 커지고, simulation이 field를 갱신할 때 gradient resource도 같은 epoch로 갱신해야 한다. 또한 interpolated gradient는 원래 trilinear scalar의 exact derivative가 아닐 수 있다.

Sparse GPU renderer라면 선택 기준에 **halo availability와 LOD consistency**도 넣어야 한다. Fine field에서 hit를 찾고 gradient sample 일부만 coarse fallback을 쓰면 position과 normal이 다른 surface를 설명할 수 있다.

그래서 한 가지 정답보다 workload를 기준으로 선택한다고 답하는 것이 좋다. Measurement/picking consistency가 중요하고 corner data를 이미 읽는 path라면 analytic gradient가 매력적이고, visual volume rendering에서 specular continuity가 중요하면 filtered/interpolated gradient가 더 나을 수 있다. 매우 높은 normal/curvature 품질이 필요하다면 trilinear gradient를 억지로 보정하기보다 `C1` higher-order reconstruction을 고려하는 것이 representation 관점에서 더 일관된 해법이다.

마지막으로 어떤 방식이든 grid-to-world metric을 적용한 뒤 normalize하고, raw gradient magnitude와 field epoch를 유지해 field-quality와 version mismatch를 추적하는 것이 중요하다.

## 8. 포트폴리오 / 커리어 연결
이 주제는 포트폴리오에서 **“SDF normal을 계산했다”**보다 훨씬 강한 설명 소재가 된다. 핵심은 geometry solver와 shading을 별도 기능으로 보지 않고 하나의 continuous/discrete representation contract로 연결하는 것이다.

좋은 architecture 설명은 다음 흐름을 가진다.

- **Traversal correctness**: Lipschitz/hierarchy로 surface candidate를 놓치지 않음
- **Intersection correctness**: bracketed refinement로 zero-set root uncertainty를 줄임
- **Attribute correctness**: 같은 reconstruction에서 gradient/normal을 평가
- **Attribute stability**: cell/brick/LOD boundary의 derivative discontinuity 관리
- **Second-order validity**: curvature를 reconstruction order와 scale에 맞게 평가
- **GPU cost model**: bandwidth vs ALU vs residency vs derived-resource update

이 흐름은 rendering engineer뿐 아니라 simulation visualization, GPU compute, engine architecture 면접에서도 좋은 신호다. 특히 “analytic gradient가 더 정확한데 왜 화면은 더 불안정할 수 있는가?”라는 질문에 `C0 scalar / discontinuous derivative`로 답할 수 있으면 interpolation을 단순 texture-filtering API가 아니라 **numerical reconstruction**으로 이해하고 있음을 보여준다.

포트폴리오 지표도 단순 FPS보다 다음이 설득력 있다.

- hit-position error와 normal angular error를 분리
- cell-boundary normal jump histogram
- specular temporal variance
- value-only vs gradient-volume VRAM
- field update 후 derived-gradient latency
- sparse halo miss rate
- gradient magnitude/Eikonal error heatmap
- curvature scale별 stability 비교

또 `FieldSnapshot + SurfaceAttributePolicy` 같은 C++ abstraction을 설명하면 API-specific 코드를 넘어 **correctness contract를 엔진 구조로 옮기는 능력**을 드러낼 수 있다.

## 9. 내일 이어서 볼 개념
**Second-Order SDF Differential Geometry: Hessian Projection, Principal Curvatures, and Curvature-Stable Surface Analysis**

오늘은 gradient/normal을 중심으로 curvature가 왜 더 어려운지까지 연결했다. 내일은 second-order geometry 자체로 들어가 다음을 본다.

- implicit-surface Hessian과 tangent-plane projection
- principal curvature / principal direction
- exact SDF에서 Hessian의 geometric 의미
- `Hessian · normal ≈ 0` 관계와 Eikonal property
- finite-difference Hessian의 stencil/precision 비용
- trilinear field에서 second derivative가 갖는 구조적 한계
- curvature smoothing scale과 feature size
- sparse brick halo 및 multi-LOD curvature consistency
- curvature-driven visualization과 measurement confidence

즉 **surface normal의 방향 정보에서 surface가 어느 방향으로 얼마나 휘는지에 대한 second-order attribute로 확장**한다.

## 10. 참고 키워드
- **Implicit Surface Gradient**
- **Signed Distance Field (SDF)**
- **Level Set**
- **Eikonal Equation / `|∇phi| = 1`**
- **Trilinear Interpolation**
- **Analytic Trilinear Gradient**
- **Central Difference Gradient**
- **Interpolated Gradient Field**
- **C0 / C1 Continuity**
- **Surface Normal Stability**
- **Normal Angular Error**
- **Specular Temporal Stability**
- **Grid-to-World Jacobian / Inverse Transpose**
- **Anisotropic Voxel Spacing**
- **Gradient Magnitude / Eikonal Residual**
- **Hessian Matrix**
- **Mean Curvature / Gaussian Curvature / Principal Curvature**
- **Laplacian**
- **Sparse Brick Halo / Ghost Voxels**
- **LOD / Epoch Consistency**
- **Derived Resource Versioning**
- **GPU Volume Rendering**
- **Kohlbrenner & Alexa, Contouring Signed Distance Fields by Approximating Gradients, Computer Graphics Forum, 2026**
  - https://onlinelibrary.wiley.com/doi/10.1111/cgf.70373
- **NVIDIA GPU Gems, Chapter 39: Volume Rendering Techniques — central-difference gradient computation**
  - https://developer.nvidia.com/gpugems/gpugems/part-vi-beyond-triangles/chapter-39-volume-rendering-techniques
- **OpenVDB CurvatureStencil — gradient, Laplacian, mean/Gaussian/principal curvature stencils**
  - https://www.openvdb.org/documentation/doxygen/classopenvdb_1_1v13__0_1_1math_1_1CurvatureStencil.html
- **Csébfalvi, One Step Further Beyond Trilinear Interpolation and Central Differences: Triquadratic Reconstruction and its Analytic Derivatives, CGF 2023**
  - https://onlinelibrary.wiley.com/doi/10.1111/cgf.14753
- **VoxNeuS: Enhancing Voxel-Based Neural Surface Reconstruction via Gradient Interpolation, 2024**
  - https://arxiv.org/abs/2406.07170