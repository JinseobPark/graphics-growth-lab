---
title: "Moment-Preserving History Reconstruction and Variance Reinitialization"
date: "2026-08-03"
category: Graphics
tags: ["GPU", "Temporal Denoising", "Temporal Moments", "Variance", "History Reconstruction", "Compute Shader", "Memory Layout", "Ray Tracing", "Scientific Visualization"]
level: intermediate
---

# [Daily Graphics Growth] 2026-08-03 - Moment-Preserving History Reconstruction and Variance Reinitialization

## 1. 오늘의 개념

전날의 **Confidence-Weighted History Inpainting**은 temporal reprojection이 실패한 pixel에서 같은 surface에 속하는 주변 history를 찾아 radiance를 복구했다. 그러나 color만 그럴듯하게 채워도 temporal denoiser의 내부 통계가 복구된 signal과 맞지 않으면 다음 frame부터 문제가 다시 발생한다.

오늘의 개념은 **Moment-Preserving History Reconstruction**과 **Variance Reinitialization**이다. 핵심은 복구한 history의 평균값만 만드는 것이 아니라, 그 값이 얼마나 불확실한지 나타내는 **first raw moment**, **second raw moment**, **variance**, **effective history length**를 서로 일관된 상태로 다시 구성하는 것이다.

Luminance를 확률 변수 `L`로 보면 기본 temporal statistics는 다음과 같다.

- first raw moment: `m1 = E[L]`
- second raw moment: `m2 = E[L²]`
- variance: `σ² = max(m2 - m1², 0)`

Temporal accumulation은 단순 color blending이 아니라 매 pixel에 작은 통계 추정기를 유지하는 과정이다. History가 direct reprojection으로 이어졌을 때는 기존 moments를 갱신하면 되지만, neighborhood에서 빌린 history나 disocclusion 직후의 history는 기존 시간축 표본과 동일한 의미를 갖지 않는다. 따라서 moments와 variance를 보수적으로 재초기화해야 한다.

## 2. 한 줄 핵심

**복구된 history의 평균만 재사용하지 말고 raw moments를 함께 재구성하며, current sample과 reconstructed history의 불일치를 variance에 반영해야 temporal filter가 거짓 확신을 갖지 않는다.**

## 3. 왜 중요한가

Temporal denoiser의 spatial filter radius, accumulation weight, anti-lag, history clipping은 variance에 크게 의존한다. 잘못된 variance는 두 방향의 artifact를 만든다.

- **Variance underestimation**: 실제로 불안정한 영역을 안정적이라고 판단해 history를 과도하게 신뢰한다. 결과는 ghosting, stale lighting, specular trail, disocclusion smear다.
- **Variance overestimation**: 안정적인 영역도 noisy하다고 판단해 spatial blur와 current-frame weight를 과도하게 높인다. 결과는 detail loss, temporal shimmer, 느린 convergence다.

특히 history inpainting에서는 여러 source pixel의 평균값이 비슷해 보여도 각 source가 서로 다른 시간적 mean을 가질 수 있다. Source variance를 단순 평균하면 source mean 간 차이가 사라져 최종 variance가 과소평가된다.

가중치 `a_i`의 합이 1이고 각 source의 mean과 variance가 `μ_i`, `σ_i²`라면 mixture의 올바른 variance는 다음과 같다.

`μ = Σ a_i μ_i`

`σ² = Σ a_i(σ_i² + μ_i²) - μ²`

이는 다음 두 불확실성을 모두 포함한다.

1. 각 source 내부의 시간적 변동인 **within-source variance**
2. source mean 사이의 차이인 **between-source variance**

단순히 `Σ a_i σ_i²`만 사용하면 두 번째 항을 잃는다. 따라서 history reconstruction에서 central variance를 직접 평균하기보다 raw moments를 모은 뒤 `m2 - m1²`로 variance를 복원하는 방식이 더 안전하다.

SVGF는 temporal accumulation으로 유효 표본 수를 늘리고 luminance moments에서 얻은 variance를 hierarchical wavelet filter의 강도 제어에 사용한다. NVIDIA NRD의 RELAX도 radiance history와 second moment를 함께 다루며, history confidence와 history length를 통해 temporal responsiveness를 조정한다. 즉 moments는 부가 metadata가 아니라 denoiser의 다음 의사결정을 지배하는 핵심 state다.

## 4. 구현 관점

### 4.1 Direct history의 일반적인 moment update

Current luminance가 `L_c`, reprojected history moments가 `m1_h`, `m2_h`이고 current sample의 blend weight가 `α`라면 raw-moment accumulation은 다음과 같이 표현할 수 있다.

`m1_t = (1 - α) m1_h + α L_c`

`m2_t = (1 - α) m2_h + α L_c²`

`σ_t² = max(m2_t - m1_t², 0)`

History length `n`을 명시적으로 사용할 때는 `α ≈ 1 / (n + 1)` 형태가 가능하다. 실시간 denoiser에서는 반응성을 위해 maximum history length, signal type, roughness, motion, confidence에 따라 `α`를 조정한다.

중요한 점은 first moment와 second moment에 동일한 blending semantics가 적용되어야 한다는 것이다. Color는 강하게 history를 믿고 second moment만 빠르게 current로 당기거나, 반대로 moments만 오래 유지하면 mean과 variance의 의미가 분리된다.

### 4.2 Neighborhood history의 moment-preserving reconstruction

같은 surface로 검증된 source history `i`에 normalized weight `a_i`를 부여한다.

`m1_r = Σ a_i m1_i`

`m2_r = Σ a_i m2_i`

`σ_r² = max(m2_r - m1_r², 0)`

이 방식은 각 source의 second raw moment에 이미 포함된 내부 variance와 source mean 간 차이를 함께 보존한다. 반대로 다음 방식은 피해야 할 통계적 함정이다.

`잘못된 예: σ_r² = Σ a_i σ_i²`

이 식은 서로 다른 평균을 가진 source의 혼합 불확실성을 잃는다. Edge 근처, moving light, glossy reflection, dynamic volume에서는 이 차이가 크게 나타난다.

### 4.3 Current sample을 포함한 보수적 재초기화

Reconstructed history가 direct temporal trajectory를 따르지 않았다면 그대로 full history로 채택하기보다 current sample과 mixture를 만든다. History trust를 `β`라 하면 다음과 같이 생각할 수 있다.

`m1_new = β m1_r + (1 - β) L_c`

`m2_new = β m2_r + (1 - β) L_c²`

`σ_new² = max(m2_new - m1_new², σ_floor²)`

이 식은 단순한 선형 blend처럼 보이지만 variance를 전개하면 current와 history mean 차이에서 오는 항이 자연스럽게 포함된다.

`σ_new² = β σ_r² + β(1 - β)(m1_r - L_c)²`

Current sample과 reconstructed mean이 크게 다르면 두 번째 항이 증가한다. 즉 lighting change나 잘못된 inpainting source가 들어왔을 때 variance가 자동으로 커지고, 다음 spatial/temporal stage가 해당 history를 덜 신뢰하게 된다.

`β`는 다음 요소에 의해 제한된다.

- source support confidence
- same-surface compatibility
- source distance와 search level
- effective sample count
- direct/repaired/reset classification
- motion 및 disocclusion severity
- signal class와 roughness

Borrowed history는 보통 direct history보다 낮은 `β`와 낮은 maximum history length를 갖는다.

### 4.4 Variance floor와 cold-start 문제

Disocclusion pixel에서 current sample 하나만 사용하면 `m1 = L_c`, `m2 = L_c²`이므로 계산상 variance는 0이다. 하지만 표본 하나로 얻은 0은 “완벽히 안정적”이라는 뜻이 아니라 “아직 분산을 추정할 정보가 없다”는 뜻이다.

따라서 cold-start variance에는 최소한의 uncertainty floor가 필요하다.

`σ_init² = max(σ_temporal², σ_spatial² · k_s, σ_min²)`

여기서 `σ_spatial²`는 동일 surface로 제한된 작은 neighborhood에서 얻은 spatial luminance variance이고, `k_s`는 spatial estimate를 temporal uncertainty로 변환하는 보수 계수다. Firefly에 민감한 signal에서는 mean/variance 대신 median absolute deviation과 같은 robust scale estimate가 보조적으로 사용될 수 있다.

Variance floor는 고정 상수만으로 결정하기보다 signal magnitude와 연동하는 편이 낫다.

`σ_min² = a + b · m1²`

이 형태는 어두운 영역의 절대 노이즈와 밝은 영역의 상대 노이즈를 동시에 다룰 수 있다.

### 4.5 Effective history length의 재해석

Reconstructed history는 여러 neighbor가 제공한 값이지만, 이들을 독립적인 temporal samples처럼 합산하면 안 된다. Screen-space neighbor는 동일한 ray sequence, blue-noise pattern, shading event를 공유해 상관관계가 높을 수 있다.

가중치 기반 support 규모는 다음 effective sample count로 추정할 수 있다.

`N_eff = (Σ w_i)² / Σ(w_i²)`

그러나 이 값은 spatial support의 균등성을 나타낼 뿐 실제 temporal sample count와 동일하지 않다. 따라서 reconstructed history length는 다음처럼 cap이 필요하다.

`n_repair = min(n_source_weighted, n_repair_max, k_corr · N_eff)`

`k_corr`는 source correlation을 반영하는 1 이하의 계수로 볼 수 있다. 실무적으로 중요한 것은 정확한 통계적 unbiasedness보다 repaired history가 빠르게 재검증되고 direct history로 전환되도록 하는 것이다.

### 4.6 Raw moment와 central moment의 저장 선택

Temporal state로 저장할 수 있는 대표 형식은 두 가지다.

1. **Raw moments 저장**: `m1`, `m2`
2. **Mean + variance 저장**: `μ`, `σ²`

Raw moments는 weighted reconstruction과 mixture 계산이 간단하다. 여러 source를 합칠 때 각각의 `m1`, `m2`를 동일 weight로 누적한 뒤 variance를 다시 계산하면 된다.

Mean + variance 형식은 사람이 해석하기 쉽지만 mixture 시 `σ² + μ²`로 second raw moment를 복원해야 한다. 단순 variance interpolation은 between-source variance를 놓칠 수 있다.

FP16에서는 `m2 = L²`의 dynamic range가 빠르게 증가한다. NVIDIA NRD 문서도 RELAX가 second moment를 추적하므로 HDR signal의 제곱이 FP16 범위에 들어오도록 sane range나 비공격적 color compression이 필요하다고 설명한다. 따라서 raw radiance가 매우 큰 renderer에서는 다음 선택이 중요하다.

- exposure 적용 후 moments 계산
- luminance compression 또는 log-like mapping
- firefly clamp와 moment input clamp 분리
- moment buffer만 FP32 사용 여부
- RGB moments 대신 luminance moments 사용

Compression을 사용하면 variance는 compressed domain의 값이므로 filter threshold와 reconstruction semantics도 같은 domain에서 정의되어야 한다.

### 4.7 Signal별 variance semantics

- **Diffuse GI**: 비교적 넓은 spatial support와 긴 history가 가능하며 luminance variance가 noise estimate로 잘 작동한다.
- **Glossy specular**: reflected hit, roughness, virtual motion이 바뀌면 mean change가 빠르므로 repaired history cap과 variance inflation이 더 강해야 한다.
- **Hard shadow**: binary-like visibility는 luminance Gaussian 가정과 다르다. first moment를 visibility probability로 보면 `p(1-p)` 형태의 variance 해석이 가능하다.
- **Ambient occlusion**: `[0,1]` bounded signal이라 FP16 range 문제는 작지만 geometry disocclusion과 thin geometry가 moments를 흔든다.
- **Volumetric radiance**: transmittance와 emission의 multiplicative structure 때문에 linear luminance variance만으로는 깊이 방향 불확실성을 충분히 표현하지 못할 수 있다.

하나의 variance 정책을 모든 signal에 공통 적용하면 memory는 단순해지지만 artifact 특성은 달라진다. 엔진에서는 signal policy table이 history cap, variance floor, clamp strength, spatial radius를 분리하는 구조가 적합하다.

### 4.8 GPU compute와 memory access

Moment reconstruction은 arithmetic 자체는 가볍지만 previous radiance, moments, confidence, history length, depth, normal, material ID를 함께 읽으므로 bandwidth 중심 pass가 되기 쉽다.

대표적인 최적화 관점은 다음과 같다.

- Guide metadata를 먼저 읽고 invalid tap의 radiance/moment fetch 생략
- Tile 단위 shared memory에 depth, normal, confidence, compact moment를 적재
- Direct history valid pixel은 neighborhood search와 reconstruction을 건너뛰는 fast path
- Repaired pixel mask를 compaction해 sparse dispatch를 구성할지, full-screen divergence를 감수할지 비교
- FP16 vector load에 맞춰 radiance와 second moment를 `RGBA16_FLOAT`로 pack
- Variance는 다음 pass에서만 사용된다면 persistent history가 아니라 transient `R16_FLOAT`로 생성
- Wave/subgroup ballot으로 tile 내 repair 필요 여부를 판정

1080p 기준 대략적인 resource 크기는 다음과 같다.

- `RGBA16_FLOAT`: 약 15.8 MiB
- `RG16_FLOAT`: 약 7.9 MiB
- `R16_FLOAT`: 약 4.0 MiB
- `RG8_UNORM`: 약 4.0 MiB

Radiance RGB와 luminance second moment를 하나의 `RGBA16_FLOAT`에 pack하면 fetch 수가 줄지만, color precision과 moment dynamic range가 같은 format에 묶인다. 별도 `RG16_FLOAT` moments는 flexibility가 높지만 texture traffic과 descriptor 수가 증가한다.

### 4.9 C++ render graph와 state contract

C++ render graph 관점에서 moment reconstruction pass의 resource contract는 다음 상태를 명확히 구분해야 한다.

- current noisy signal
- direct reprojected history
- repaired history source
- previous first/second moments
- previous history length와 confidence
- current G-buffer guide
- reconstruction class: direct / repaired / reset
- output moments, variance, effective history length

Camera cut, resolution change, exposure discontinuity, transfer-function change, topology revision은 moments reset 조건이 된다. 특히 auto exposure가 moments 계산 전후 어느 위치에 적용되는지 frame 간 일관되어야 한다. Exposure convention이 바뀌면 radiance history뿐 아니라 `m1`과 `m2`도 동일 scale로 변환되어야 한다.

Radiance가 `s`배 변환되면 moments는 다음처럼 변한다.

`m1' = s · m1`

`m2' = s² · m2`

Second moment를 first moment와 같은 비율로 scaling하면 variance가 잘못된다. Dynamic exposure를 지원하는 temporal pipeline에서 자주 놓치는 contract다.

API 관점에서는 D3D12/HLSL의 UAV/SRV barrier, Vulkan의 storage image access mask와 layout transition, OpenGL의 image/texture memory barrier, WebGPU의 pass usage separation이 중요하다. Ping-pong history resource는 previous frame의 immutable SRV와 current frame의 UAV를 분리해 same-frame feedback을 방지한다.

### 4.10 Failure mode와 debug view

대표적인 failure mode는 다음과 같다.

- Repaired color만 갱신하고 old moments를 유지해 variance와 mean이 불일치
- Neighbor variance를 직접 평균해 between-source variance 손실
- Single-sample reset에서 variance가 0으로 시작해 history가 즉시 과신됨
- Source history length를 그대로 복사해 borrowed history가 오래 고착됨
- FP16 second moment overflow 또는 precision collapse
- Exposure 변화에서 `m2`를 `s²`가 아닌 `s`로 보정
- Repaired history가 다음 pixel의 source가 되어 한 frame 안에 moments가 연쇄 확산
- Negative variance가 발생했을 때 단순 clamp만 하고 원인을 숨김

유용한 debug view:

- first moment / second moment / reconstructed variance
- direct / repaired / reset classification
- within-source variance와 between-source variance 분리 표시
- current-history mean delta
- variance floor가 적용된 pixel
- `N_eff`와 final effective history length
- moment overflow/clamp mask
- FP16 vs FP32 moment difference heatmap
- exposure-rescaled history validation

## 5. 내 관심 분야와 연결

### 실시간 렌더링과 게임 엔진

Ray-traced reflection, GI, shadow denoiser에서 품질 문제는 대부분 temporal history가 실패한 이후 드러난다. Graphics engineer는 motion vector와 reprojection만 설명하는 데서 끝나지 않고, rejected history가 moments와 confidence를 어떤 상태로 남기는지 설명할 수 있어야 한다.

특히 glossy reflection에서는 color가 비슷해 보여도 reflected hit distribution이 바뀔 수 있다. Virtual motion이나 hit distance가 불안정한 영역에 낮은 variance가 남으면 specular trail이 길게 유지된다. Roughness-aware history cap과 between-source variance 보존은 반사 denoising의 안정성과 직접 연결된다.

### Level-set, voxel, Marching Cubes

Level-set 기반 공정 emulation은 geometry가 매 step 재구성되므로 이전 frame의 mesh topology identity가 쉽게 깨진다. Surface history를 same-material voxel이나 interface neighborhood에서 복구하더라도 process-step revision과 signed-distance 변화가 크다면 moments를 그대로 이어서는 안 된다.

이 경우 current field와 reconstructed history 차이는 geometry evolution uncertainty로 해석할 수 있다. Variance inflation은 렌더링 history의 신뢰도를 낮추는 역할이며 solver의 물리적 데이터 자체를 변경하지 않는다.

### CFD와 scientific visualization

Stochastic volume rendering, particle splatting, path-traced scientific visualization에서도 temporal moments는 noise와 detail을 구분하는 데 사용될 수 있다. 다만 scalar field 변화나 transfer-function 변경이 radiance 변화로 이어질 때, variance 증가가 Monte Carlo noise인지 실제 simulation change인지 분리해야 한다.

Simulation time-step ID, AMR block revision, transfer-function hash를 history validity에 포함하면 실제 물리 변화가 old moments에 흡수되어 늦게 보이는 문제를 줄일 수 있다.

### GPU 최적화

이 주제는 알고리즘과 memory layout의 결합을 보여준다. `RGBA16_FLOAT` 한 번의 vector fetch로 RGB와 second moment를 읽는 구조, moments를 별도 texture로 분리하는 구조, direct-history fast path, sparse repair dispatch는 모두 품질과 bandwidth의 trade-off다.

RenderDoc, PIX, Nsight에서 pass duration만 보는 것이 아니라 cache hit, texture bytes, wave occupancy, divergent branch, FP16 overflow debug view를 함께 보는 관점이 필요하다.

## 6. 머릿속에 남길 질문 3개

1. **여러 neighbor history를 합칠 때 variance를 직접 평균하면 왜 source mean 간 차이가 사라지며, raw moments를 합치면 어떤 항이 보존되는가?**
2. **Disocclusion 직후 current sample 하나의 계산상 variance가 0일 때, 이를 실제 confidence 100%로 해석하지 않으려면 어떤 spatial prior와 variance floor가 필요한가?**
3. **Repaired history의 `N_eff`, confidence, history length는 서로 어떤 의미 차이가 있으며 왜 하나의 숫자로 완전히 대체할 수 없는가?**

## 7. graphics engineer 면접 질문 1개와 답변

**질문:** Temporal denoiser에서 reprojection이 실패한 pixel의 radiance를 주변 valid history로 복구했다. First moment, second moment, variance, history length를 어떻게 재구성해야 하며 단순 variance averaging이 왜 위험한가?

**답변:** 각 source의 normalized weight로 first raw moment `m1`과 second raw moment `m2`를 각각 합산하고, 최종 variance는 `max(m2 - m1², 0)`으로 복원한다. Variance를 직접 평균하면 각 source 내부 variance만 남고 source mean 사이의 차이에서 오는 between-source variance가 사라져 불확실성을 과소평가할 수 있다. Reconstructed history는 direct temporal trajectory가 아니므로 current sample과 낮은 history trust로 mixture를 만들며, 이때 current와 history mean 차이가 variance inflation 항으로 반영되도록 한다. History length는 source 값을 그대로 복사하지 않고 confidence, support, `N_eff`, repair cap으로 줄인다. Single-sample reset은 계산상 variance가 0이므로 same-surface spatial variance와 signal-dependent floor를 사용해 cold-start uncertainty를 유지한다. GPU에서는 radiance, moments, confidence, history length, G-buffer guide를 일관된 domain과 exposure convention으로 관리하고 FP16 second-moment overflow를 주의해야 한다.

## 8. 포트폴리오 / 커리어 연결

포트폴리오에서는 “temporal denoising 구현”이라는 표현보다 **history state의 통계적 일관성을 설계했다**는 점을 보여주는 것이 강하다.

설명할 핵심 항목:

- raw moments와 central variance의 차이
- mixture variance와 between-source variance
- direct / repaired / reset state machine
- current-history discrepancy 기반 variance inflation
- cold-start variance floor
- effective sample count와 history length의 구분
- HDR compression과 FP16 second-moment 범위
- exposure 변화 시 `m1`, `m2` scale contract
- signal별 diffuse/specular/shadow variance policy
- packed memory layout과 bandwidth 비교

시각 자료는 다음 조합이 효과적이다.

- noisy input / denoised output
- mean only reconstruction / moment-preserving reconstruction 비교
- variance underestimation으로 발생한 ghosting 사례
- direct / repaired / reset mask
- current-history delta와 variance heatmap
- FP16 overflow 또는 clamp 영역
- GPU timing과 persistent memory footprint

면접에서는 `reprojection → validation → repair → moment reconstruction → variance reinitialization → temporal accumulation → spatial filtering`의 전체 흐름을 한 번에 설명하면 rendering pipeline과 statistical filtering을 함께 이해하고 있음을 보여줄 수 있다.

## 9. 내일 이어서 볼 개념

**Variance-Guided À-Trous Wavelet Filtering and Edge-Stopping Functions**

오늘은 temporal history의 mean과 variance를 일관되게 재구성했다. 다음은 이 variance를 실제 spatial filter radius와 weight에 연결하는 단계다. À-trous wavelet filter가 step width를 늘리며 receptive field를 확장하는 방식, luminance variance와 depth·normal·hit-distance 기반 edge-stopping function이 noise 제거와 detail preservation을 어떻게 균형 잡는지 살펴본다.

## 10. 참고 키워드

- Temporal Moments
- First Raw Moment / Second Raw Moment
- Luminance Variance
- Moment-Preserving Reconstruction
- Mixture Variance
- Within-Source Variance / Between-Source Variance
- Variance Reinitialization
- Effective Sample Count
- History Confidence / History Length
- Disocclusion / History Repair
- SVGF (Spatiotemporal Variance-Guided Filtering)
- NVIDIA NRD RELAX / REBLUR
- À-Trous Wavelet Filter
- Edge-Stopping Function
- FP16 Dynamic Range
- HDR Luminance Compression
- Temporal Anti-Lag
- Compute Shader / Shared Memory / Subgroup
- Render Graph / Persistent History Resource
- Scientific Visualization Temporal Stability

### 참고 자료

- [NVIDIA Research - Spatiotemporal Variance-Guided Filtering](https://research.nvidia.com/publication/2017-07_spatiotemporal-variance-guided-filtering-real-time-reconstruction-path-traced)
- [NVIDIA RTX - NRD Repository and Integration Notes](https://github.com/NVIDIA-RTX/NRD)
- [NVIDIA NRD - RELAX Temporal Accumulation Shader](https://github.com/NVIDIA-RTX/NRD/blob/master/Shaders/RELAX_TemporalAccumulation.cs.hlsl)
- [Microsoft Direct3D 12 Raytracing Samples - Real-Time Denoised Ambient Occlusion](https://learn.microsoft.com/en-us/samples/microsoft/directx-graphics-samples/d3d12-raytracing-samples-win32/)
