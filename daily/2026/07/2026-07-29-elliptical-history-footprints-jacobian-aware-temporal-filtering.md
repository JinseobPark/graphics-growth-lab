---
title: "Elliptical History Footprints and Jacobian-Aware Temporal Filtering"
date: "2026-07-29"
category: "Graphics"
tags: ["GPU", "Temporal Filtering", "Jacobian", "Anisotropic Filtering", "EWA", "Ray Differentials", "Specular", "Reprojection", "Compute Shader", "Memory Layout"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-07-29 - Elliptical History Footprints and Jacobian-Aware Temporal Filtering

## 1. 오늘의 개념

**Elliptical History Footprint**는 현재 pixel 하나가 이전 프레임의 history image에서 단일 점이 아니라, 방향과 크기를 가진 **타원형 영역(elliptical region)**에 대응한다는 관점이다.

일반적인 temporal reprojection은 현재 pixel 좌표 `p`에 motion vector를 적용해 이전 좌표 하나를 찾는다.

`p_prev = F(p_current)`

`history = Sample(previousHistory, p_prev)`

이 방식은 현재 pixel 주변의 작은 영역이 이전 프레임에서도 거의 같은 크기와 모양을 유지한다는 가정을 포함한다. 그러나 전날 다룬 **Reflection-Space Jacobian**처럼 mapping의 국소 미분이 존재하면, 현재 pixel footprint는 이전 프레임에서 확대·축소·회전·전단된 타원으로 변환된다.

현재 pixel 주변의 작은 좌표 변화 `δp`에 대해 다음과 같이 선형화할 수 있다.

`F(p + δp) ≈ F(p) + J · δp`

여기서 `J = ∂F/∂p`는 2×2 Jacobian이다. 현재 pixel footprint를 원형 또는 정사각형 support로 근사하면, `J`가 이를 이전 프레임의 anisotropic ellipse로 변형한다.

이때 temporal filter의 질문은 더 이상 “이전 프레임의 어느 한 점을 읽을 것인가?”가 아니다.

- 이전 history의 어느 **영역**이 현재 pixel에 기여하는가?
- 그 영역은 어느 방향으로 길게 늘어나는가?
- 어느 MIP level 또는 몇 개의 anisotropic tap이 필요한가?
- footprint가 geometry boundary를 넘을 때 어떤 sample을 버려야 하는가?
- mapping이 fold 또는 singular 상태에 가까울 때 history를 얼마나 신뢰할 수 있는가?

오늘의 핵심은 texture filtering에서 사용되는 **anisotropic footprint**와 **Elliptical Weighted Average(EWA)** 관점을 temporal history sampling으로 확장하는 것이다.

## 2. 한 줄 핵심

**Motion vector가 history의 중심 좌표만 알려준다면, Jacobian은 현재 pixel이 이전 프레임에서 차지하는 타원형 footprint를 알려주며, temporal filter는 이 타원의 크기·방향·조건수에 따라 anisotropic sampling, MIP 선택, history confidence를 함께 결정해야 한다.**

## 3. 왜 중요한가

### 단일 bilinear sample은 국소 변형을 표현하지 못한다

Standard TAA 또는 denoiser의 reprojection은 보통 이전 좌표에서 bilinear sample 하나를 읽는다. 이 방식은 mapping이 거의 translation일 때 효율적이고 안정적이다.

그러나 curved reflection, refraction, non-linear projection, foveated rendering, dynamic resolution, virtual world-space reprojection에서는 current-to-previous mapping이 방향별로 다른 scale을 가진다.

예를 들어 Jacobian의 singular value가 다음과 같다고 하자.

`σmax = 4.0`

`σmin = 0.8`

현재 pixel footprint는 이전 프레임에서 한 방향으로 약 4배 늘어나고, 직교 방향으로는 거의 유지된다. 이 상황에서 center sample 하나만 읽으면 넓은 영역의 대표값을 한 texel에 맡기므로 다음 문제가 나타난다.

- history의 고주파 패턴이 시간에 따라 깜빡이는 **temporal aliasing**
- 곡면 반사 무늬가 표면 위를 미끄러지는 **specular swimming**
- 확대되는 방향에서 history 정보가 부족해지는 **undersampling**
- 축소 또는 folding 부근에서 서로 다른 feature가 섞이는 **history smearing**
- 화면 미세 이동만으로 반사 패턴이 불연속적으로 바뀌는 **reprojection instability**

### History sample도 reconstruction filter다

Temporal accumulation은 과거 값을 단순히 복사하는 과정이 아니라, 현재 pixel이 나타내는 신호를 이전 프레임의 discrete samples로부터 재구성하는 **reconstruction problem**이다.

Texture mapping에서 screen pixel이 texture space의 넓은 영역을 덮으면 anisotropic filtering이 필요하다. 동일하게 temporal reprojection에서도 current pixel이 previous history의 넓고 비스듬한 영역에 대응하면 방향성을 가진 prefilter가 필요하다.

차이는 history texture가 일반 color texture보다 훨씬 복잡하다는 점이다.

- depth와 normal이 다르면 같은 surface가 아닐 수 있다.
- material ID와 object ID가 다르면 평균내면 안 된다.
- PSR path signature가 다르면 reflection topology가 바뀐 것이다.
- history length와 variance도 함께 일관성을 유지해야 한다.
- disocclusion 영역에는 유효한 이전 signal 자체가 없다.

따라서 temporal ellipse는 단순한 image-space blur kernel이 아니라 **surface-validity constraint가 포함된 anisotropic reconstruction domain**이다.

### Jacobian의 크기뿐 아니라 조건수가 중요하다

`J`의 Singular Value Decomposition(SVD)을 다음처럼 쓸 수 있다.

`J = U · diag(σ1, σ2) · Vᵀ`

- `U`: previous-frame ellipse의 주축 방향
- `σ1`, `σ2`: 각 축의 stretch
- `V`: current footprint 내부에서의 입력 방향

`σmax / σmin`은 condition number 또는 anisotropy ratio의 근사다.

- 두 singular value가 모두 1에 가까움: bilinear reprojection이 충분할 가능성이 높음
- `σmax`만 큼: major axis 방향의 anisotropic sampling 필요
- 두 값이 모두 큼: 넓은 previous region을 저주파화하는 prefilter 필요
- `σmin`이 0에 가까움: mapping이 한 방향으로 collapse되는 near-singular 상태
- condition number가 매우 큼: 작은 motion 또는 normal noise가 history 좌표를 크게 흔드는 민감한 상태

즉 footprint는 sample radius뿐 아니라 **reprojection stability 자체를 측정하는 기하학적 신호**다.

### 반사와 과학 시각화 모두에서 같은 문제가 나타난다

이 개념은 ray-traced reflection에만 국한되지 않는다.

- volume ray marching에서 카메라 이동과 adaptive step이 이전 ray footprint를 비등방적으로 변형할 수 있다.
- CFD scalar field의 slice 또는 streamline visualization에서 camera-space reprojection이 field feature를 한 방향으로 늘일 수 있다.
- sparse voxel LOD 전환에서는 current cell footprint가 이전 level의 여러 texel에 대응할 수 있다.
- level-set surface가 재메시되면 screen-space mapping은 유지되어 보여도 topology correspondence는 끊길 수 있다.
- dynamic resolution이나 foveated rendering은 screen domain 자체가 non-linear mapping을 가진다.

따라서 Jacobian-aware temporal filtering은 rendering, simulation visualization, reconstruction pipeline을 연결하는 일반적인 graphics engineering 문제다.

## 4. 구현 관점

### 4.1 Pixel footprint를 covariance로 표현한다

현재 pixel footprint를 중심이 0인 isotropic Gaussian 또는 disk로 근사한다고 하자. 간단한 covariance 표현은 다음과 같다.

`Σcurrent = r² · I`

여기서 `r`은 current pixel support radius이며, `I`는 2×2 identity matrix다.

Jacobian을 통해 이전 프레임으로 변환된 covariance는 다음과 같다.

`Σhistory = J · Σcurrent · Jᵀ`

`Σcurrent = r²I`이면 다음처럼 단순화된다.

`Σhistory = r² · J · Jᵀ`

`Σhistory`의 eigenvector는 ellipse의 major/minor axis 방향을 나타내고, eigenvalue의 제곱근은 각 축의 scale을 나타낸다.

실제 temporal reconstruction에서는 최소 footprint와 불확실성을 추가할 수 있다.

`Σeffective = J · Σcurrent · Jᵀ + Σbase + Σmotion`

- `Σbase`: bilinear reconstruction, camera jitter, 최소 1-texel support
- `Σmotion`: motion quantization, virtual hit uncertainty, deformation error
- roughness 또는 ray-cone spread는 별도의 signal footprint로 합성 가능

이 표현의 장점은 회전된 ellipse를 axis-aligned radius 두 개로 억지로 근사하지 않고, 방향성을 2×2 symmetric matrix로 유지한다는 점이다.

### 4.2 Ellipse의 quadratic form

History center를 `c = F(p)`라고 하고 candidate sample 위치를 `x`라고 하면 offset은 다음과 같다.

`d = x - c`

Ellipse 내부 여부는 quadratic form으로 판정할 수 있다.

`q = dᵀ · Σeffective⁻¹ · d`

`q ≤ 1`이면 1-sigma 계열 footprint 안에 있다고 볼 수 있다. 실제 finite support kernel은 `q ≤ k²` 형태로 support를 넓힐 수 있다.

Gaussian 형태의 EWA weight는 다음처럼 표현할 수 있다.

`w_ewa(d) = exp(-0.5 · q)`

Compact polynomial kernel을 사용하면 exponential 비용을 줄이면서 finite support를 만들 수 있다.

중요한 점은 EWA weight만으로 sample을 합치지 않는 것이다. Temporal history에서는 geometry guide weight가 추가되어야 한다.

`w_total = w_ewa · w_depth · w_normal · w_id · w_path · w_confidence`

각 항은 다음 의미를 가진다.

- `w_depth`: previous depth 또는 virtual viewZ correspondence
- `w_normal`: geometric normal continuity
- `w_id`: object/material/surface identity
- `w_path`: PSR path signature 또는 reflection topology
- `w_confidence`: 이전 history의 통계적 신뢰도

결과적으로 타원은 sample 후보의 공간적 범위를 정하고, guide는 후보가 같은 signal domain에 속하는지 결정한다.

### 4.3 SVD를 temporal policy로 변환한다

Full SVD를 모든 pixel에서 정확히 계산할 필요는 없다. 2×2 matrix의 `JᵀJ` 또는 `JJᵀ`는 closed-form eigenvalue를 가지므로 singular value를 비교적 저렴하게 추정할 수 있다.

`A = J · Jᵀ`

`trace = A00 + A11`

`detA = A00 · A11 - A01²`

`λ1,2 = 0.5 · (trace ± sqrt(max(trace² - 4detA, 0)))`

`σ1,2 = sqrt(max(λ1,2, 0))`

이 값은 다음 temporal policy로 연결된다.

| Footprint 상태 | 해석 | Temporal policy |
|---|---|---|
| `σmax ≈ σmin ≈ 1` | 거의 rigid reprojection | center bilinear + 일반 history weight |
| `σmax > 1`, `σmin ≈ 1` | 한 방향 stretch | major-axis anisotropic taps |
| `σmax > 1`, `σmin > 1` | 면적 확대 | guide-aware prefilter 또는 history MIP |
| `σmin ≪ 1` | 한 방향 collapse | history length 제한, strong clamp |
| `σmax/σmin` 매우 큼 | ill-conditioned mapping | confidence 감소, footprint radius cap |
| `det(J)` 부호가 이웃과 다름 | fold 또는 orientation transition 가능성 | boundary rejection 또는 history reset |

`det(J) < 0` 자체가 항상 잘못된 것은 아니다. Orientation-reversing mapping도 수학적으로 유효할 수 있다. 더 위험한 신호는 `det(J)`가 0에 가까워지거나, neighborhood에서 부호가 급변하거나, path identity가 함께 바뀌는 경우다.

### 4.4 History sampling 방식

#### A. Center sample + Jacobian confidence

가장 저렴한 방식은 기존 bilinear reprojection을 유지하고, Jacobian에서 만든 confidence만 history weight에 곱하는 것이다.

- 장점: texture fetch 증가가 거의 없다.
- 단점: anisotropic footprint를 실제로 integration하지 않으므로 aliasing 감소는 제한적이다.
- 용도: narrow glossy reflection, moderate curvature, bandwidth가 매우 제한된 플랫폼

#### B. Major-axis anisotropic taps

Ellipse의 major axis 방향으로 3~8개의 sample을 배치하고 EWA/guide weight를 적용한다.

- minor axis는 bilinear 또는 작은 MIP footprint로 처리한다.
- major axis 길이가 길수록 tap 간격 또는 tap 수가 증가한다.
- tap 수는 quantized bucket으로 제한하면 shader divergence를 줄일 수 있다.

이 방식은 hardware anisotropic texture filtering의 아이디어와 유사하지만, temporal guide validation이 pixel마다 필요하므로 일반 sampler state만으로 해결되지 않는다.

#### C. History MIP + anisotropic taps

Validity-aware history pyramid가 존재한다면 minor axis에 맞는 MIP level을 선택하고, major axis 방향으로 여러 tap을 읽는 구조가 가능하다.

개념적인 LOD는 다음처럼 생각할 수 있다.

`lod_minor ≈ log2(max(r · σmin, 1))`

그 후 major axis 길이에 따라 여러 sample을 배치한다.

이 방식은 broad footprint에서 fetch 수를 줄일 수 있지만, 일반 color MIP처럼 history를 단순 평균하면 서로 다른 surface가 이미 섞여버릴 수 있다. 따라서 MIP 생성 단계부터 depth, normal, ID, validity, history length를 보존하는 별도 contract가 필요하다.

#### D. Stochastic footprint sampling

Ellipse 내부에서 frame마다 한두 개의 sample을 stochastic하게 선택하고 temporal accumulation으로 적분하는 방식도 가능하다.

- 장점: 고정 tap 수로 넓은 footprint를 커버할 수 있다.
- 단점: history lookup 자체에 추가 noise가 생기며, low-discrepancy sequence와 confidence 설계가 필요하다.
- denoiser와 temporal upscaler가 이미 stochastic reconstruction을 수행하는 pipeline에서는 고려할 수 있다.

### 4.5 Geometry boundary와 topology invalidation

Local Jacobian은 mapping이 neighborhood에서 미분 가능하다는 가정 위에 있다. 다음 위치에서는 이 가정이 쉽게 깨진다.

- foreground/background silhouette
- mirror edge와 reflection fold
- 서로 다른 object 또는 material boundary
- PSR path bounce count가 바뀌는 위치
- dynamic mesh의 topology revision
- Marching Cubes remeshing으로 triangle identity가 바뀐 위치
- sparse voxel chunk 또는 LOD boundary
- disocclusion과 newly visible region

따라서 neighbor finite difference로 `J`를 계산할 때는 좌우/상하 texel이 같은 domain인지 먼저 검증해야 한다.

유효성 contract의 예시는 다음과 같다.

- stable object ID 일치
- material class 일치
- geometric normal dot product threshold
- current/previous virtual depth continuity
- PSR path signature 또는 bounce class 일치
- geometry revision ID 일치
- active viewport 및 dynamic-resolution tile 일치

Neighbor가 invalid이면 derivative를 0으로 두는 것보다 해당 axis를 unavailable로 표시하는 편이 안전하다. 0 derivative는 실제로는 “변화 없음”을 의미하므로 collapse된 footprint로 오해될 수 있다.

### 4.6 Footprint clamping

이론적인 Jacobian이 매우 큰 ellipse를 만들더라도 실시간 temporal filter가 무제한 radius를 사용할 수는 없다.

실무적인 footprint cap은 다음 요소를 함께 반영한다.

- 최대 texel radius
- 최대 anisotropy ratio
- 최대 history MIP level
- roughness별 specular lobe support
- current signal variance
- surface boundary까지의 거리
- history sample validity ratio
- frame budget에 따른 tap bucket

큰 footprint를 무조건 넓게 평균내는 것은 aliasing을 줄이지만 reflection detail을 과도하게 제거할 수 있다. 따라서 `σmax`가 커질수록 radius만 늘리는 것이 아니라 history confidence를 줄이고 current sample 비중을 높이는 정책이 함께 필요하다.

`historyWeight = baseWeight · jacobianConfidence · validSampleRatio`

`jacobianConfidence`는 예를 들어 다음 신호에서 감소할 수 있다.

- high condition number
- determinant magnitude가 매우 작음
- determinant sign discontinuity
- curvature proxy가 큼
- path signature instability
- large virtual motion divergence

### 4.7 GPU memory layout

#### Full Jacobian 저장

2×2 Jacobian을 `RGBA16_FLOAT`로 저장하면 pixel당 8 bytes다.

1920×1080 기준:

- 약 16.6 MB/frame, 즉 약 15.8 MiB
- read와 write를 모두 포함하면 pass bandwidth가 빠르게 증가
- FP16은 일반적인 screen UV derivative에는 충분할 수 있지만 large virtual coordinate 또는 near-singular case에서 precision 검증이 필요

장점은 후속 pass가 SVD, determinant, ellipse axis를 자유롭게 계산할 수 있다는 점이다.

#### Packed ellipse 저장

Full `J` 대신 다음을 저장할 수 있다.

- major axis direction `float2`
- `log2(σmax)`
- `log2(σmin)` 또는 confidence

`RGBA16_FLOAT`이면 저장 크기는 동일하지만 후속 계산이 줄어든다. Axis direction을 octahedral-like 1D angle 또는 signed normalized representation으로 더 압축할 수도 있다.

#### Scalar confidence만 저장

`R8_UNORM` confidence만 저장하면 1080p에서 약 2.1 MB다.

- 장점: bandwidth가 작고 기존 temporal pass에 쉽게 결합된다.
- 단점: 실제 anisotropic sampling 방향과 MIP level을 복원할 수 없다.

따라서 renderer의 목표에 따라 세 단계로 나눌 수 있다.

1. `R8` confidence-only
2. packed ellipse parameters
3. full `RGBA16F` Jacobian

#### Tap bandwidth

이전 radiance가 `RGBA16_FLOAT`이고 8-tap sampling을 수행하면 radiance read만으로 1080p에서 frame당 약 132.7 MB가 필요하다. 60 FPS에서는 약 8 GB/s 수준이며, depth·normal·ID·moments·history length fetch가 추가되면 실제 비용은 훨씬 커진다.

따라서 성능은 ALU보다 memory traffic에 의해 제한될 가능성이 높다.

GPU 관점의 핵심 고려사항은 다음과 같다.

- tap pattern을 2/4/8개 bucket으로 quantize
- nearby pixels가 비슷한 major axis를 가질 때 texture cache locality 활용
- guide data를 compact format으로 유지
- repeated depth/normal fetch를 shared memory tile 또는 subgroup exchange로 줄임
- Jacobian 생성과 confidence 계산의 pass fusion 가능성
- history radiance와 moments의 binding 및 lifetime 최적화

### 4.8 Compute shader와 render graph contract

일반적인 render graph에서는 다음 resource가 관여한다.

**Persistent history resources**

- previous radiance 또는 demodulated signal
- previous moments/variance
- previous history length
- previous depth/viewZ
- previous normal/roughness
- previous object/material/path identity

**Current-frame resources**

- current signal
- motion 또는 previous UV
- current depth/normal/ID
- Jacobian 또는 packed ellipse
- disocclusion/confidence mask

**Transient resources**

- local derivative buffer
- ellipse parameters
- valid-tap ratio
- optional guide-aware MIP levels

C++ renderer에서는 resource format만큼 coordinate convention이 중요하다.

- motion이 pixel unit인지 normalized UV인지
- `old = new + MV`인지 반대 방향인지
- Jacobian이 pixel-to-pixel인지 UV-to-UV인지
- current/previous jitter가 제거된 matrix인지
- dynamic resolution scale이 Jacobian에 포함되었는지
- viewport offset과 subrect가 반영되었는지

이 contract가 흔들리면 ellipse axis는 시각적으로 그럴듯해 보여도 실제 sampling radius가 resolution에 따라 달라지는 버그가 발생한다.

Compute shader에서 neighbor derivative를 만들 경우 fragment derivative에 의존할 수 없으므로 explicit texture load, workgroup tile, subgroup quad operation 중 하나가 필요하다. Boundary pixel과 inactive lane을 포함한 subgroup 연산은 별도의 validity 처리가 필요하다.

### 4.9 수치 안정성

Near-singular Jacobian에서 `Σ⁻¹`을 직접 계산하면 수치가 폭발할 수 있다.

안정화 방법은 다음 원칙을 따른다.

- eigenvalue 또는 singular value에 minimum clamp 적용
- maximum footprint radius 제한
- determinant가 작을 때 inverse 대신 confidence rejection
- FP16 저장 전에 log-domain scale 압축 고려
- NaN/INF를 history reset 신호로 처리
- neighbor path mismatch에서 derivative를 계산하지 않음

Covariance에 작은 isotropic term을 더하는 regularization도 유용하다.

`Σregularized = Σeffective + ε²I`

하지만 regularization은 invalid mapping을 valid하게 만드는 해결책이 아니다. Topology가 끊긴 위치는 ellipse를 안정화하는 것이 아니라 history correspondence를 거부해야 한다.

## 5. 내 관심 분야와 연결

### C++ 렌더링 엔진

이 주제는 shader 수식보다 **resource contract와 render graph 설계**가 더 어렵다. C++ 측에서는 history buffer ping-pong, dynamic resolution, camera cut, geometry revision, PSR mode 변경을 하나의 temporal reset policy로 통합해야 한다.

Engine-level 관점에서 중요한 질문은 다음과 같다.

- Jacobian을 어느 pass가 소유하는가?
- reflection, denoiser, temporal upscaler가 같은 motion convention을 공유하는가?
- packed footprint가 API-independent format으로 표현되는가?
- transient Jacobian buffer를 다른 temporal effect가 재사용할 수 있는가?
- debug view가 production resource와 동일한 데이터를 보는가?

### Vulkan · DirectX · OpenGL · WebGPU

이 개념은 특정 API보다 compute-centric pipeline contract에 가깝다.

- Vulkan/D3D12: explicit resource state, descriptor lifetime, async compute scheduling
- OpenGL: image load/store barrier와 texture feedback 방지
- WebGPU: storage texture와 sampled texture 역할 분리, explicit neighbor load 기반 derivative
- 공통: FP16 storage 지원, normalized/integer guide format, ping-pong history 관리

WebGPU에서도 full ray tracing API가 없어도 screen-space reflection, temporal accumulation, volume rendering의 history footprint 분석에 동일한 수학을 적용할 수 있다.

### CFD와 scientific visualization

CFD scalar field 또는 vector field의 temporal visualization에서는 camera motion만이 아니라 simulation 자체의 advection이 correspondence를 결정한다.

- screen-space motion: camera와 geometry projection
- field-space motion: velocity field에 따른 feature 이동
- topology motion: vortex 생성·소멸, iso-surface 분리·결합

Jacobian-aware footprint는 field-space mapping이 한 방향으로 stretch되는 shear flow에서 유용하다. 다만 물리 field가 실제로 변한 경우 history를 넓게 평균내면 수치 feature가 왜곡될 수 있으므로, temporal visualization에서는 geometry confidence뿐 아니라 simulation timestamp, timestep, field revision을 함께 사용해야 한다.

### Voxel · Marching Cubes · Level-Set

Sparse voxel과 adaptive LOD에서는 화면의 current footprint가 previous LOD의 여러 cell을 덮을 수 있다. Elliptical footprint는 screen-space sampling density를 설명하지만, chunk revision이나 remeshing이 발생하면 local mapping만으로 surface identity를 보장할 수 없다.

특히 level-set에서 topology가 변하는 구간은 Jacobian이 유한하게 보여도 correspondence가 존재하지 않을 수 있다. 이 경우 geometry revision ID 또는 stable region label이 temporal validity의 상위 조건이 된다.

### Ray tracing과 denoising

Specular signal은 roughness와 hit distance에 따라 자체적인 ray footprint를 가진다. Reflection-space Jacobian은 screen mapping footprint를 설명하고, roughness/ray cone은 path-space signal spread를 설명한다.

두 footprint를 결합하면 다음 질문으로 이어진다.

- screen deformation과 BRDF lobe spread를 하나의 covariance로 합칠 수 있는가?
- sharp reflection과 rough reflection에 같은 anisotropic history kernel을 적용해야 하는가?
- virtual hit uncertainty와 geometric curvature를 어떻게 분리할 것인가?

이 관점은 단순 denoising parameter tuning보다 훨씬 깊은 **signal-space reconstruction design**으로 연결된다.

## 6. 머릿속에 남길 질문 3개

1. **현재 pixel footprint를 `J · Jᵀ` 기반 ellipse로 만들었을 때, geometry boundary를 넘는 sample을 공간 kernel이 아니라 surface identity contract로 제거해야 하는 이유는 무엇인가?**

2. **`σmax`가 크고 `σmin`이 작아 condition number가 높은 반사 영역에서, filter radius 확대와 history confidence 감소 중 어느 쪽이 artifact를 더 안정적으로 줄이는가?**

3. **Reflection-space Jacobian, roughness-derived ray footprint, motion uncertainty를 하나의 covariance로 합칠 때 서로 다른 의미의 blur를 과도하게 중복 적용하지 않으려면 어떤 분리 기준이 필요한가?**

## 7. graphics engineer 면접 질문 1개와 답변

### 질문

곡면 반사에 temporal reprojection을 적용할 때 motion vector 하나만 사용하는 방식의 한계를 설명하고, 2×2 Jacobian을 이용해 history sampling을 개선하는 GPU-friendly 구조를 제안해보세요.

### 답변

Motion vector는 current pixel 중심이 previous frame의 어느 좌표에 대응하는지만 표현한다. 곡면 반사에서는 인접 pixel이 서로 다른 normal과 reflected direction을 가지므로 reflection image가 translation뿐 아니라 scale, rotation, shear를 가진다. 따라서 center bilinear sample 하나는 current pixel이 previous history에서 차지하는 넓고 비등방적인 footprint를 재구성하지 못해 temporal aliasing, swimming, ghosting이 발생한다.

Current-to-previous mapping을 `F(p)`라고 두고 local Jacobian `J = ∂F/∂p`를 계산하면 current pixel footprint를 previous frame의 ellipse로 근사할 수 있다. Isotropic current covariance `Σc`를 사용하면 previous covariance는 `Σh = JΣcJᵀ`이며, eigenvector가 ellipse axis, eigenvalue의 제곱근이 axis scale이 된다.

GPU-friendly baseline은 ray pass 또는 reprojection pass에서 previous UV를 만들고, 후속 compute pass가 2×2 quad나 3×3 neighborhood의 finite difference로 Jacobian을 추정하는 방식이다. 이후 `σmax`, `σmin`, determinant, condition number를 이용해 2/4/8-tap anisotropic history lookup 또는 confidence-only policy를 선택한다.

각 tap에는 depth, geometric normal, object/material ID, PSR path signature 검증을 적용해야 한다. Mapping이 near-singular이거나 determinant sign이 neighborhood에서 급변하거나 path identity가 바뀌면 footprint를 넓히는 대신 history를 reject한다. Full Jacobian은 `RGBA16F`, 저비용 경로는 packed ellipse 또는 `R8` confidence로 저장할 수 있으며, 실제 병목은 ALU보다 history radiance와 guide texture의 memory bandwidth가 될 가능성이 높다.

## 8. 포트폴리오 / 커리어 연결

이 개념은 “TAA를 구현했다”보다 한 단계 높은 **temporal reconstruction architecture** 역량을 보여준다.

포트폴리오에서 드러낼 수 있는 내용은 다음과 같다.

- planar, convex, concave reflector의 Jacobian/ellipse debug visualization
- motion-only reprojection과 Jacobian-aware history sampling 비교
- major/minor singular value, determinant, condition number heatmap
- center sample, 4-tap, 8-tap, MIP-assisted 방식의 quality/performance 비교
- surface ID와 PSR path validation 유무에 따른 edge leakage 비교
- full `RGBA16F` Jacobian, packed ellipse, `R8` confidence의 memory trade-off
- 1080p/1440p/4K에서 tap bandwidth와 frame time 분석
- dynamic resolution과 camera jitter convention을 포함한 render graph diagram
- curved reflection뿐 아니라 volume, voxel, CFD visualization에 같은 footprint model을 적용한 확장성

면접에서는 다음 역량을 함께 보여줄 수 있다.

- differential reasoning과 linear algebra
- texture filtering 이론을 temporal reconstruction에 전이하는 능력
- GPU cache와 bandwidth 중심의 성능 분석
- C++ render graph와 shader resource contract 설계
- visual artifact를 geometry, signal, sampling 문제로 분해하는 디버깅 능력
- API-independent architecture 사고

Nintendo, Unity, real-time engine, visualization 팀 관점에서도 이 주제는 rendering feature 하나보다 더 넓은 문제 해결 능력을 보여준다. 특히 “왜 blur radius를 늘렸는가”가 아니라 “어떤 mapping footprint와 validity contract에서 그 radius가 도출되는가”를 설명할 수 있다는 점이 강한 차별점이다.

## 9. 내일 이어서 볼 개념

**Validity-Aware History Mipmaps and Guide-Preserving Downsampling**

오늘은 Jacobian으로 previous-frame footprint를 ellipse로 해석하고 anisotropic tap 또는 MIP 선택에 연결했다. 다음에는 temporal history의 MIP pyramid를 일반 color average로 만들면 surface leakage가 발생하는 이유를 다루고, depth·normal·ID·validity·moments를 보존하는 guide-aware downsampling과 history pyramid resource layout을 이어서 본다.

## 10. 참고 키워드

- Elliptical History Footprint
- Jacobian-Aware Temporal Filtering
- Elliptical Weighted Average (EWA)
- Anisotropic Reconstruction
- Temporal Reprojection
- Reflection-Space Jacobian
- Ray Differentials
- Path Differentials
- Covariance Propagation
- Singular Value Decomposition (SVD)
- Major Axis / Minor Axis
- Condition Number
- Determinant
- Mapping Fold
- Specular Virtual Motion
- Primary Surface Replacement (PSR)
- Guide-Aware Filtering
- History Confidence
- Disocclusion
- Validity Mask
- Surface Identity
- History MIP Pyramid
- Demodulated Radiance
- Compute Shader
- GPU Memory Bandwidth
- Subgroup Quad Operation
- Dynamic Resolution
- Foveated Rendering
- [Homan Igehy, Tracing Ray Differentials](https://graphics.stanford.edu/papers/trd/)
- [Paul Heckbert, Fundamentals of Texture Mapping and Image Warping](https://www2.eecs.berkeley.edu/Pubs/TechRpts/1989/5504.html)
- [Greene and Heckbert, Elliptical Weighted Average Filter](https://www.ri.cmu.edu/publications/creating-raster-omnimax-images-from-multiple-perspective-views-using-the-elliptical-weighted-average-filter/)
- [NVIDIA, Texture Level of Detail Strategies for Real-Time Ray Tracing](https://research.nvidia.com/publication/2019-03_texture-level-detail-strategies-real-time-ray-tracing)
- [NVIDIA Real-Time Denoisers (NRD)](https://github.com/NVIDIA-RTX/NRD)
