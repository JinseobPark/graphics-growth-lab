---
title: "Curvature-Aware Adaptive Surface Extraction: Feature Radius, QEF Conditioning, and Error-Bounded LOD"
date: "2026-09-03"
category: Graphics
tags: [GPU, Rendering, SDF, Level Set, Dual Contouring, QEF, Curvature, Principal Curvature, Adaptive Meshing, LOD, Octree, Sparse Volume, CUDA, Vulkan, Compute Shader, Memory Layout, C++]
level: intermediate
---

# [Daily Graphics Growth] 2026-09-03 - Curvature-Aware Adaptive Surface Extraction: Feature Radius, QEF Conditioning, and Error-Bounded LOD

## 1. 오늘의 개념
최근 노트에서는 refined SDF hit point에서 gradient와 normal을 같은 reconstructed field에 맞추고, curvature가 Hessian 같은 second-order information을 요구한다는 점까지 연결했다.

오늘은 그 geometric signal을 **surface extraction의 refinement decision**으로 넘긴다.

핵심 질문은 다음이다.

> **모든 voxel cell을 같은 해상도로 polygonize하지 않고, 실제 surface detail이 필요한 곳에만 mesh density를 집중하려면 무엇을 기준으로 해야 하는가?**

SDF/level-set에서 local feature scale을 나타내는 대표적인 값은 principal curvature `k1`, `k2`다. 큰 curvature magnitude는 작은 radius의 feature를 의미한다.

`r_feature ≈ 1 / max(|k1|, |k2|)`

따라서 cell size `h`가 feature radius에 비해 너무 크면 corner, neck, trench sidewall, rounded edge가 하나의 cell 안에 뭉개질 수 있다. 반대로 거의 평평한 영역은 더 큰 cell로도 비슷한 geometric error를 유지할 수 있다.

Dual Contouring 계열에서는 각 active cell 안에 하나의 representative vertex를 두고, surface crossing에서 얻은 point/normal constraint를 이용해 **Quadratic Error Function(QEF)** 을 최소화한다. Curvature-aware adaptive extraction은 단순히 “curvature가 크면 subdivide”하는 문제에 그치지 않는다. 실제 production 구조에서는 다음 세 가지를 함께 봐야 한다.

- geometric feature scale
- QEF의 numerical conditioning
- screen/world-space error budget

오늘은 이 세 요소를 GPU-friendly adaptive hierarchy와 연결한다.

## 2. 한 줄 핵심
> Adaptive SDF meshing의 핵심은 **curvature로 local feature radius를 추정하고, QEF conditioning으로 cell 내부의 feature ambiguity를 감지하며, world/screen-space error budget으로 refinement를 제한해 필요한 곳에만 geometry·bandwidth·memory를 쓰는 것**이다.

## 3. 왜 중요한가
Dense Marching Cubes는 해상도가 단순하고 GPU parallelization이 쉽지만, 모든 cell에 동일한 spatial resolution을 강요한다. 3D process visualization이나 large sparse field에서는 surface가 차지하는 영역이 전체 volume의 일부이고, 그 안에서도 실제 detail은 더 국소적이다.

예를 들어 wafer의 넓은 평면과 nano-scale corner가 같은 cell size를 요구하지 않는다. 평면은 coarse mesh로 충분하지만 trench corner, thin layer edge, narrow gap은 작은 geometric error budget이 필요하다.

Curvature는 이러한 local detail의 한 proxy다. `|k|`가 크면 normal이 짧은 거리 안에서 빠르게 변하고, 작은 feature radius가 존재한다는 뜻이다. 하지만 curvature만으로 refinement를 결정하면 세 가지 문제가 생긴다.

첫째, **sharp feature에서는 curvature가 수치적으로 매우 크거나 정의되지 않을 수 있다.** 따라서 curvature estimate 자체의 confidence가 필요하다.

둘째, **QEF가 잘-conditioned인지**도 중요하다. 거의 평행한 normal constraint만 모이면 vertex 위치의 일부 방향이 약하게 결정되어 tiny numerical noise가 큰 위치 변화로 증폭될 수 있다.

셋째, 최종 목적은 field-space accuracy가 아니라 renderer/analysis가 요구하는 error다. 멀리 있는 object는 높은 curvature가 있어도 screen-space 영향이 작을 수 있고, 반대로 metrology view에서는 sub-pixel이 아닌 world-unit error가 핵심일 수 있다.

즉 좋은 adaptive extractor는 geometry signal 하나가 아니라 **feature scale + solver confidence + consumer error**를 함께 본다.

## 4. 구현 관점
### 4.1 Curvature를 feature radius로 해석한다
principal curvature를 `k1`, `k2`라 하면 local radius scale을

`r1 = 1 / max(|k1|, eps)`

`r2 = 1 / max(|k2|, eps)`

처럼 볼 수 있다.

cell edge length를 `h`라 하면 `h / r_feature`는 “이 cell이 surface bending scale에 비해 얼마나 큰가”를 나타내는 무차원 지표가 된다.

이 지표는 절대 curvature threshold보다 해상도 변화에 강하다. 예를 들어 LOD가 바뀌어 `h`가 두 배가 되면 같은 surface curvature라도 refinement pressure가 자연스럽게 커진다.

다만 curvature estimate는 derivative stencil scale과 field reconstruction scale에 의존하므로, `k` 자체와 함께 **measurement scale**을 resource metadata로 유지해야 한다.

### 4.2 QEF는 point fit이 아니라 plane fit이다
Dual Contouring의 classical QEF는 surface crossing에서 얻은 point `pi`와 normal `ni`에 대해

`E(x) = Σ (ni · (x - pi))²`

를 최소화한다.

행렬 형태로는

`A x ≈ b`

`A_i = ni^T`

`b_i = ni · pi`

이며 normal equation은

`A^T A x = A^T b`

가 된다.

중요한 점은 QEF가 단순 point centroid가 아니라 **여러 tangent plane의 교차에 가까운 위치**를 찾는다는 것이다. 그래서 sharp edge/corner를 cell 내부에서 보존할 가능성이 높다.

### 4.3 QEF conditioning은 geometry ambiguity signal이다
`A^T A`의 eigenvalue 또는 singular value를 보면 constraint가 공간을 얼마나 잘 제한하는지 알 수 있다.

- 세 방향이 충분히 독립적이면 corner-like constraint가 강하다.
- 두 방향만 강하면 edge-like feature가 된다.
- 한 방향만 강하면 거의 planar surface다.

작은 singular value는 해당 방향의 vertex 위치가 약하게 결정된다는 뜻이다. 따라서 QEF condition number가 나쁘다는 것은 단순 numerical issue만이 아니라 **현재 cell의 Hermite data가 위치를 충분히 결정하지 못한다는 geometry signal**이기도 하다.

GPU에서는 full generic solver보다 symmetric 3×3 matrix의 spectral structure, truncated SVD, eigenvalue thresholding 같은 구조가 더 예측 가능한 선택이 될 수 있다.

### 4.4 Regularization은 bias와 stability의 trade-off다
Ill-conditioned QEF에 작은 regularization을 넣으면 vertex가 비정상적으로 멀리 튀는 현상을 줄일 수 있다.

대표적인 형태는 cell center 또는 mass point `m`을 향한 penalty를 더하는 것이다.

`E'(x) = E(x) + λ ||x - m||²`

이는 solver stability를 높이지만 sharp feature 위치를 center 방향으로 bias할 수 있다.

따라서 `λ`는 “수치 안정성 parameter”인 동시에 **geometric fidelity parameter**다. 포트폴리오/면접에서는 이 trade-off를 설명할 수 있어야 한다.

### 4.5 Cell clamp는 safety net이지 feature detector가 아니다
QEF minimizer가 현재 cell 밖으로 나갈 수 있다. 이를 단순 clamp하면 topological instability를 줄일 수 있지만, 원래 최적 plane intersection을 왜곡할 수 있다.

더 중요한 것은 왜 minimizer가 cell 밖으로 갔는지를 보는 것이다.

- noisy normal
- inconsistent crossing point
- under-constrained QEF
- coarse cell이 여러 feature를 동시에 포함
- LOD/halo mismatch

즉 out-of-cell vertex는 단순 좌표 문제보다 **refinement 필요성의 진단 신호**로 해석할 수 있다.

### 4.6 Refinement score는 여러 signal의 조합이다
한 cell의 refinement priority를 개념적으로

`Score = w_c * CurvatureError + w_q * QEFAmbiguity + w_s * ScreenError + w_r * ResidencyCost`

처럼 볼 수 있다.

여기서
- `CurvatureError`: `h * max(|k1|, |k2|)` 같은 scale-relative bending
- `QEFAmbiguity`: condition number, residual, out-of-cell distance
- `ScreenError`: projected cell size 또는 projected geometric error
- `ResidencyCost`: fine brick loading/creation 비용

로 분리할 수 있다.

실제 핵심은 weighted sum 자체가 아니라 **각 signal이 무엇을 보호하는지 분리하는 것**이다.

### 4.7 World-space error와 screen-space error는 목적이 다르다
Simulation/metrology에서는 `μm` 단위 geometric error가 중요할 수 있다. Renderer에서는 projected pixel error가 더 직접적이다.

World-space geometric error는 curvature와 cell size로 근사할 수 있고, small patch에서 chord error는 대략 curvature와 `h²`에 비례하는 형태로 증가한다.

Screen-space error는 그 world-space error를 camera projection과 depth에 따라 pixel footprint로 바꾼 것이다.

따라서 하나의 hierarchy가 여러 consumer를 지원한다면

- geometry validity threshold
- rendering LOD threshold
- analysis/metrology threshold

를 분리하는 것이 좋다.

### 4.8 Adaptive hierarchy의 crack 문제
인접 cell이 서로 다른 refinement level을 가지면 face connectivity가 깨질 수 있다. Adaptive octree/brick hierarchy에서는 다음 contract가 중요하다.

- neighboring level difference 제한
- transition cell 또는 stitching rule
- shared vertex ownership
- deterministic edge-crossing classification
- 동일 field epoch 사용

Refinement 자체보다 **LOD boundary의 watertight connectivity**가 더 어려운 경우가 많다.

### 4.9 Sparse volume과 adaptive mesh hierarchy는 같은 트리가 아닐 수 있다
Sparse SDF brick hierarchy는 memory residency와 sampling을 최적화한다. Adaptive surface hierarchy는 geometric error와 connectivity를 최적화한다.

둘을 같은 octree로 강제로 합치면 구조는 단순해지지만 서로 다른 refinement pressure가 충돌할 수 있다.

예를 들어
- field는 coarse brick 하나로 충분히 resident
- surface는 corner 때문에 더 fine mesh가 필요

할 수 있다.

반대로 field는 simulation 때문에 fine resident지만 화면상 멀리 있어 mesh는 coarse로 충분할 수도 있다.

따라서 production 구조에서는 `Field LOD`와 `Mesh LOD`를 logical layer로 분리하고, mapping metadata로 연결하는 편이 유연하다.

### 4.10 GPU execution pipeline
Adaptive extraction을 GPU 관점으로 보면 대략 다음 stage로 분리된다.

1. active cell classification
2. curvature/QEF/error metadata evaluation
3. refinement flag generation
4. prefix-sum/compaction
5. leaf-cell QEF solve
6. topology/connectivity generation
7. vertex/index buffer compaction
8. indirect draw 또는 downstream mesh processing

이 구조에서 핵심 memory-layout 질문은 **per-cell hot metadata를 얼마나 작게 유지할 것인가**다.

예를 들어 hot SoA에는
- active mask
- level
- QEF residual
- feature scale
- refinement flag
- compacted output index

정도만 두고, full Hermite constraints나 debug statistics는 cold buffer로 분리할 수 있다.

### 4.11 QEF solver와 register pressure
한 thread가 cell 하나의 QEF를 풀 때 많은 crossing point/normal을 register에 유지하면 register pressure가 높아질 수 있다.

반대로 constraints를 global/shared memory에 저장하면 bandwidth가 늘어난다.

따라서 practical design은
- accumulation matrix `A^T A`의 6개 symmetric component
- vector `A^T b`의 3개 component
- scalar residual-related accumulator

처럼 **constraint stream을 compact sufficient statistics로 줄이는 방향**이 중요하다.

### 4.12 Temporal stability
Dynamic level-set에서 cell refinement가 threshold 근처를 오가면 mesh LOD가 frame마다 toggle될 수 있다.

Residency에서 eviction hysteresis가 필요했던 것처럼 adaptive meshing에서도
- split threshold
- merge threshold
- minimum lifetime
- feature-confidence decay

같은 temporal hysteresis가 필요하다.

Curvature가 noisy한 field라면 hysteresis 없이 adaptive mesh를 만들었을 때 geometry shimmer가 shading instability보다 더 크게 보일 수 있다.

## 5. 내 관심 분야와 연결
Semiconductor process visualization에서는 geometry scale의 dynamic range가 크다.

- wafer/substrate의 넓은 평면
- thin oxide layer
- trench/gate corner
- narrow spacer
- rounded etch/deposition profile

를 동일 mesh resolution로 다루는 것은 비효율적이다.

SDF/level-set 기반 pipeline에서 curvature와 QEF를 함께 사용하면 surface가 실제로 복잡한 부분에 mesh budget을 집중할 수 있다. 특히 CUDA compute에서 SDF가 업데이트된 뒤 전체 dense Marching Cubes를 매번 수행하는 대신, **changed brick + high-error cell** 위주로 extraction work를 제한하는 architecture와 자연스럽게 연결된다.

NanoVDB/sparse grid를 사용하는 경우에도 field sparsity와 mesh adaptivity를 구분해서 생각할 수 있다. Field residency는 scalar access를 위한 것이고, mesh refinement는 geometric representation error를 위한 것이다.

Babylon.js/WebGL/WebGPU frontend로 전달할 때는 adaptive mesh가 vertex count와 PCIe/WebView transfer를 줄이는 장점도 있다. 다만 LOD topology가 frame마다 바뀌면 object ID, selection, measurement anchor 같은 application-level state가 깨질 수 있으므로 stable logical cell/feature ID가 필요하다.

CFD/level-set 관점에서는 curvature-aware mesh extraction이 단순 시각화 최적화가 아니라 interface diagnostics에도 유용하다. High-curvature region은 numerical instability, pinch-off, thin feature와 관련될 수 있어 renderer와 solver 분석이 같은 signal을 공유할 수 있다.

## 6. 머릿속에 남길 질문 3개
1. **높은 curvature와 ill-conditioned QEF가 동시에 나타났을 때, 이것은 실제 sharp feature인가 아니면 noisy derivative/insufficient sampling 때문인가?**
2. **Field LOD와 Mesh LOD를 같은 hierarchy로 묶는 것이 언제 유리하고, 언제 두 hierarchy를 분리해야 하는가?**
3. **screen-space error가 작은데 metrology world-space error가 큰 cell은 rendering과 engineering visualization에서 각각 어떻게 다른 refinement policy를 가져야 하는가?**

## 7. graphics engineer 면접 질문 1개와 답변
### 질문
**“Adaptive Dual Contouring에서 curvature threshold만으로 cell subdivision을 결정하면 왜 충분하지 않나요?”**

### 답변
Curvature는 local bending scale을 알려주기 때문에 좋은 refinement signal이지만 세 가지 한계가 있다.

첫째, discrete SDF의 curvature는 second derivative라 noise와 LOD/halo error에 매우 민감하다. 높은 curvature가 실제 feature인지 derivative artifact인지 confidence가 필요하다.

둘째, Dual Contouring의 vertex placement는 QEF가 결정하므로 QEF conditioning과 residual을 함께 봐야 한다. 낮은 curvature surface라도 constraint가 불충분하거나 서로 충돌하면 vertex가 불안정할 수 있고, 반대로 sharp feature는 QEF의 독립적인 normal 방향 때문에 안정적으로 포착될 수 있다.

셋째, 최종 error budget은 consumer에 따라 다르다. renderer는 projected pixel error를, simulation/metrology는 world-space geometric error를 더 중요하게 볼 수 있다.

따라서 production-quality adaptive extractor는 보통 **curvature/feature scale + QEF conditioning/residual + world/screen-space error + temporal hysteresis**를 함께 사용하고, sparse field residency는 별도의 cost/availability constraint로 취급하는 편이 안전하다.

## 8. 포트폴리오 / 커리어 연결
이 주제는 graphics engineer 포트폴리오에서 단순 polygonization을 넘어 **geometry processing과 GPU systems를 연결하는 설계 역량**을 보여주기 좋다.

강조할 수 있는 포인트는 다음과 같다.

- **Geometry:** principal curvature, feature radius, Hermite constraint, QEF
- **Numerics:** conditioning, singular value, regularization, residual
- **GPU:** per-cell sufficient statistics, compaction, indirect dispatch/draw, register pressure
- **Memory:** sparse field hierarchy와 adaptive mesh hierarchy의 분리
- **Rendering:** world-space vs screen-space error, temporal LOD hysteresis
- **Application:** measurement/selection ID 안정성, simulation visualization
- **Modern research awareness:** 2026년 SIGGRAPH의 *Dual Contouring of Signed Distance Data*처럼 discrete SDF만으로 sharp feature를 보존하는 quadratic optimization 기반 contouring 흐름

면접에서는 “Marching Cubes보다 Dual Contouring이 sharp feature에 유리하다” 수준보다, **왜 QEF가 tangent-plane intersection을 표현하고 conditioning이 feature semantics와 연결되는가**까지 설명할 수 있으면 강하다.

## 9. 내일 이어서 볼 개념
**Topology-Stable Adaptive Dual Contouring: Octree Transitions, Manifoldness, and Crack-Free GPU Connectivity**

오늘은 adaptive cell을 “어디서 쪼갤지”에 집중했다. 다음은 실제로 더 어려운 문제인 **서로 다른 LOD의 cell을 어떻게 crack 없이 연결하고 manifold topology를 유지할 것인가**로 이어진다.

학습 흐름은

`SDF gradient/curvature → feature-aware refinement → QEF vertex placement → topology-stable adaptive connectivity`

로 이어진다.

## 10. 참고 키워드
- Signed Distance Field (SDF)
- Level Set
- Principal Curvature
- Feature Radius
- Dual Contouring
- Hermite Data
- Quadratic Error Function (QEF)
- QEF Conditioning
- Singular Value / Condition Number
- Regularization
- Adaptive Octree
- World-Space Error
- Screen-Space Error
- LOD Hysteresis
- Sparse Volume
- GPU Compaction / Prefix Sum
- Indirect Draw / Indirect Dispatch
- SoA / Hot-Cold Metadata
- Ju et al., **Dual Contouring of Hermite Data**, SIGGRAPH 2002
- Carrera et al., **Dual Contouring of Signed Distance Data**, SIGGRAPH 2026
- Kohlbrenner & Alexa, **Contouring Signed Distance Fields by Approximating Gradients**, Computer Graphics Forum, 2026
