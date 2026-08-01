---
title: "Hierarchical History Search and Validity-Driven LOD Fallback"
date: "2026-08-01"
category: Graphics
tags: ["GPU", "Temporal Filtering", "History Buffer", "Mipmap", "LOD", "Validity", "Denoising", "Compute Shader", "Memory Layout", "Render Graph"]
level: intermediate
---

# [Daily Graphics Growth] 2026-08-01 - Hierarchical History Search and Validity-Driven LOD Fallback

## 1. 오늘의 개념

전날에는 **Surface-Constrained Trilinear Sampling**과 **Cross-Mip Validity**를 통해, 요청된 두 MIP level의 sample이 현재 surface와 호환되는지를 depth·normal·identity·coverage·revision으로 검증하는 방법을 살펴봤다.

오늘은 요청된 LOD에서 충분한 valid support를 얻지 못했을 때의 다음 단계인 **Hierarchical History Search**와 **Validity-Driven LOD Fallback**을 다룬다.

일반적인 texture sampling은 미분값이나 footprint로부터 하나의 fractional LOD를 정하고, 해당 위치의 두 인접 MIP을 읽으면 끝난다. 그러나 temporal history와 denoising history는 다음 이유로 그 LOD가 사용 불가능할 수 있다.

- disocclusion으로 history가 비어 있음
- foreground와 background가 coarse MIP에서 혼합됨
- object·material·topology revision이 현재 frame과 불일치함
- specular virtual motion이 크게 변해 reprojection confidence가 낮음
- sparse volume 또는 AMR grid에서 요청 영역이 비활성 상태임

**Hierarchical History Search**는 요청 LOD 하나를 절대적인 답으로 보지 않고, MIP hierarchy를 valid history 후보 집합으로 해석한다. 목표 LOD 주변의 finer/coarser level을 제한적으로 탐색하고, surface compatibility·coverage·footprint mismatch·temporal confidence를 종합해 가장 안전한 level 또는 level 조합을 선택한다.

핵심은 “가장 가까운 LOD”가 아니라 **현재 pixel의 signal domain을 보존하면서 충분한 support를 제공하는 가장 신뢰할 수 있는 LOD**를 찾는 것이다.

## 2. 한 줄 핵심

**요청 LOD의 history가 유효하지 않다면 무조건 base level이나 coarse level로 점프하지 말고, MIP hierarchy를 제한적으로 탐색하여 surface validity와 footprint 적합도가 가장 높은 level을 선택해야 ghosting·surface leakage·과도한 blur를 동시에 줄일 수 있다.**

## 3. 왜 중요한가

Temporal filtering에서 history rejection은 필요하지만, rejection 이후 항상 current-frame sample만 사용하면 noise가 크게 증가하고 convergence가 반복적으로 초기화된다. 반대로 validity가 낮은 history를 억지로 사용하면 ghosting과 light leakage가 남는다.

Hierarchical search는 이 두 극단 사이의 복구 경로다.

예를 들어 Jacobian으로 계산된 footprint가 `LOD 3.4`를 요구한다고 하자. `Mip 3`과 `Mip 4`가 모두 mixed surface라면 일반 trilinear 결과는 사용할 수 없다. 그러나 `Mip 2`에는 현재 object의 valid sample이 충분히 남아 있을 수 있고, `Mip 5`에는 surface purity는 낮지만 넓은 coverage가 있을 수 있다. 이때 최적 선택은 signal 특성에 따라 달라진다.

- **Finer fallback**: surface detail과 identity를 잘 보존하지만 footprint보다 작은 영역을 읽으므로 aliasing과 variance가 증가할 수 있다.
- **Coarser fallback**: 넓은 support로 noise를 줄이지만 다른 surface와의 혼합, lag, over-blur 위험이 커진다.
- **Current-frame fallback**: temporal contamination은 없지만 순간 noise와 flicker가 증가한다.

따라서 fallback은 단순한 `lod--` 또는 `lod++` 규칙이 아니라 다음 trade-off를 평가해야 한다.

1. surface correctness
2. valid sample coverage
3. requested footprint와 실제 footprint의 차이
4. temporal age와 history confidence
5. signal 종류—diffuse, glossy specular, shadow, scalar field

NVIDIA NRD의 HistoryFix·history confidence·fast history 개념도 같은 문제를 다른 형태로 다룬다. history가 짧거나 불안정한 영역은 주변 signal과 짧은 history를 활용해 복구하며, 충분히 신뢰할 수 없는 main history는 accumulation을 가속하거나 제한한다. 즉 현대 denoiser는 history를 binary valid/invalid로만 다루지 않고 **신뢰도와 복구 단계가 있는 계층적 자원**으로 취급한다.

## 4. 구현 관점

### 4.1 기본 입력

현재 pixel `p`에 대해 다음 값이 있다고 가정한다.

- requested fractional LOD: `λ`
- current guide: depth `z`, normal `n`, surface/material ID `id`, revision `r`
- level별 history signal: `H_m`
- level별 metadata: coverage, depth interval, normal cone, identity purity, history confidence

각 MIP level `m`에서 surface-constrained sample을 수행해 다음을 얻는다.

- normalized signal `C_m`
- effective valid support `W_m`
- metadata compatibility `G_m`
- temporal confidence `T_m`

level confidence는 다음처럼 구성할 수 있다.

`Q_m = W_m · G_m · T_m`

여기에 requested footprint와의 차이를 나타내는 LOD penalty를 추가한다.

`P_m = exp(-α · |m - λ|)`

finer level과 coarser level의 위험이 다르므로 비대칭 penalty도 가능하다.

`P_m = exp(-α_f · max(0, λ - m) - α_c · max(0, m - λ))`

- `α_f`: finer fallback의 aliasing·variance 비용
- `α_c`: coarser fallback의 blur·surface mixing 비용

최종 score의 한 예는 다음과 같다.

`S_m = Q_m · P_m`

실제 엔진에서는 diffuse와 specular에 서로 다른 `α_f`, `α_c`, threshold를 사용해야 한다. Glossy specular는 coarse level의 surface mixing에 특히 민감하므로 `α_c`를 크게 둘 수 있다.

### 4.2 탐색 순서

#### Target-centered search

요청 LOD 주변을 중심으로 탐색한다.

`l = round(λ)`

후보 순서 예:

`l, l-1, l+1, l-2, l+2 ...`

이 방식은 requested footprint와 가까운 후보를 먼저 평가한다. 단, level마다 surface-constrained 2×2 tap을 수행하면 비용이 빠르게 증가하므로 최대 탐색 반경을 2~3 level로 제한하는 편이 현실적이다.

#### Finer-first search

현재 surface identity 보존을 우선한다.

`floor(λ), floor(λ)-1, ...`

Silhouette, thin geometry, material boundary, topology revision 직후에 적합하다. 다만 fine level까지 내려가도 valid support가 적으면 high-frequency noise가 커질 수 있다.

#### Coarser-first search

variance 감소와 넓은 coverage를 우선한다.

`ceil(λ), ceil(λ)+1, ...`

저주파 diffuse indirect lighting이나 매우 rough한 reflection에 유리할 수 있다. 반면 coarse metadata의 identity purity와 depth compactness가 충분히 높아야 한다.

실무적으로는 **target-centered search + signal별 방향 bias**가 균형이 좋다. Diffuse는 약한 coarse bias, sharp specular는 strong finer bias를 사용할 수 있다.

### 4.3 Early accept와 search termination

모든 level을 끝까지 평가하는 대신 confidence가 충분한 첫 후보를 accept할 수 있다.

`if S_m >= acceptThreshold: accept m`

그러나 first-valid 방식은 level score가 비슷한 영역에서 popping을 만들 수 있다. 더 안정적인 방식은 제한된 후보를 평가한 뒤 softmax 또는 normalized score로 blend하는 것이다.

`w_m = exp(k · S_m)`

`C = Σ(w_m · C_m) / Σw_m`

다만 서로 다른 MIP이 같은 surface domain인지 먼저 확인해야 한다. Cross-mip identity·revision이 다르면 blend하지 않고 단일 후보를 선택한다.

Search 종료 조건:

- confidence가 threshold를 넘음
- dominant identity와 revision이 정확히 일치함
- coverage와 depth compactness가 충분함
- 최대 search radius 도달
- sky·background·inactive voxel로 분류됨

### 4.4 Validity-driven fallback ladder

권장되는 논리적 fallback 단계는 다음과 같이 계층화할 수 있다.

1. requested LOD의 constrained trilinear result
2. 인접 MIP level의 valid candidate
3. 가장 가까운 finer level의 high-purity sample
4. 제한된 screen-space neighborhood의 same-surface history
5. fast history 또는 short-term accumulation
6. current-frame sample
7. history weight 0으로 reset

이 순서는 모든 signal에 고정되는 규칙이 아니라 policy table로 관리하는 편이 좋다. 예를 들어 hard shadow는 spatially 가까운 다른 shadow sample을 사용할 수 있지만, mirror reflection은 잘못된 history를 쓰는 것보다 current sample로 reset하는 편이 안전하다.

### 4.5 Confidence hysteresis

LOD 선택이 frame마다 바뀌면 MIP popping과 temporal shimmer가 생긴다. Previous selected LOD `m_prev`를 저장하거나, history confidence에 hysteresis를 적용할 수 있다.

- 새 후보가 기존 후보보다 충분히 높은 score일 때만 전환
- LOD 변화량을 frame당 제한
- 선택된 LOD를 fractional value로 temporal smooth
- disocclusion·revision change에서는 hysteresis를 즉시 해제

하지만 stale LOD를 오래 유지하면 surface 변화에 늦게 반응하므로 hysteresis도 history confidence와 함께 감소해야 한다.

### 4.6 GPU shader 구조

Data-dependent loop는 warp/wave divergence를 만들 수 있으므로 탐색 횟수를 compile-time constant로 제한하는 편이 좋다.

예시 구조:

- 후보 level 3~5개를 고정 생성
- 각 후보의 coarse metadata를 먼저 읽음
- 명백히 invalid한 candidate는 signal fetch를 생략
- surviving candidate에만 explicit tap 수행
- subgroup ballot로 tile 전체가 fast path인지 판단

Compute shader에서는 workgroup shared memory에 current guide tile과 border를 적재할 수 있다. 다만 여러 MIP의 signal을 shared memory에 모두 올리는 것은 용량과 bank conflict 비용이 크므로, guide 재사용 효과가 큰 level에만 적용하는 편이 낫다.

Wave-level optimization:

- `WaveActiveAllTrue` / subgroup all: tile 전체가 requested LOD에서 valid하면 search 생략
- ballot mask: fallback이 필요한 lane만 slow path
- quad operation: 2×2 guide coherence와 edge detection

Branchless 구현이 항상 최선은 아니다. Invalid 영역이 적다면 uniform fast path와 분리된 slow pass가 더 효율적일 수 있다.

### 4.7 Memory layout

Full MIP chain의 texel 수는 base level의 약 `4/3`이다.

1080p `RGBA16_FLOAT` history 기준:

- base level: 약 15.8 MiB
- full MIP chain: 약 21.1 MiB

여기에 metadata pyramid가 추가된다.

권장 분리:

- signal: `RGBA16_FLOAT` 또는 signal별 `RG16_FLOAT`
- coverage + temporal confidence + history length: `RGBA8_UNORM`
- depth min/max: `RG16_FLOAT`
- normal mean + cone: packed `R10G10B10A2` 또는 octahedral encoding
- identity + revision: `R32_UINT` 또는 `RG16_UINT`
- selected LOD / fallback reason: debug 또는 compact persistent channel

Hierarchical search는 같은 pixel coordinate에서 여러 MIP을 읽으므로 texture cache locality는 비교적 좋다. 하지만 metadata와 signal이 여러 resource로 분리되면 descriptor pressure와 bandwidth가 증가한다. Coarse rejection에 필요한 metadata를 하나의 compact texture에 묶고, signal은 candidate가 살아남은 경우에만 읽는 구조가 효율적이다.

### 4.8 C++ render graph와 resource contract

엔진 측에서는 history pyramid를 단순 texture 하나가 아니라 다음 contract를 가진 temporal resource로 관리해야 한다.

- current/previous frame dimensions와 dynamic-resolution scale
- valid rect와 viewport origin
- MIP count와 build policy
- topology/reset epoch
- signal type과 pre-exposure 상태
- history confidence semantics
- build pass → search pass → accumulation pass 사이의 barrier

D3D12·Vulkan에서는 모든 MIP을 하나의 resource로 관리하더라도 subresource state와 UAV barrier를 명확히 해야 한다. MIP pyramid를 compute로 생성한 직후 같은 frame에 sampling한다면 write-after-read/read-after-write dependency가 render graph에 올바르게 표현되어야 한다.

API별 핵심 수단:

- **Direct3D / HLSL**: `Texture2D.Load`, `SampleLevel`, `Gather`, wave intrinsics
- **Vulkan / GLSL**: `texelFetch`, `textureLod`, subgroup operations
- **OpenGL / GLSL**: explicit LOD sampling과 image/texture barrier
- **WebGPU / WGSL**: `textureLoad`, `textureSampleLevel`, workgroup memory, storage-texture pass 분리

### 4.9 Debug view와 품질 평가

필수 debug view:

- requested LOD / selected LOD / previous selected LOD
- level별 `Q_m`, `P_m`, `S_m`
- fallback 단계와 reason code
- valid support와 identity purity
- depth interval width와 normal cone angle
- fast history 사용량
- current-frame reset mask

평가 장면:

- thin geometry와 빠른 camera pan
- glossy reflection 위의 moving light
- animated normal·water surface
- level-set topology split/merge
- sparse volume의 active/inactive boundary
- dynamic-resolution 전환

평균 frame time만으로는 부족하다. slow-path pixel 비율, 후보 level당 fetch 수, cache miss, divergence, history reset rate를 함께 측정해야 한다.

## 5. 내 관심 분야와 연결

### Real-time rendering과 denoising

Reflection history는 surface motion뿐 아니라 virtual motion, roughness, hit distance, curvature에 의해 validity가 달라진다. Hierarchical search는 reflection footprint에 맞는 LOD가 invalid할 때, sharp surface detail을 유지할지 넓은 support를 얻을지 signal별로 결정하는 정책 계층이 된다.

Diffuse indirect lighting은 low-frequency 성분이 많아 coarse fallback을 더 허용할 수 있지만, material boundary와 disocclusion에서는 identity purity를 우선해야 한다. 이 차이를 policy로 표현할 수 있으면 denoiser를 단순 blur가 아니라 signal-aware temporal system으로 설명할 수 있다.

### Level-set / voxel / semiconductor emulation

공정 step 사이에서 topology와 material region이 변하면 이전 volume pyramid의 일부 level만 재사용 가능할 수 있다. Revision ID와 active mask를 이용한 hierarchical search는 현재 voxel이 속한 material branch를 유지하면서 사용할 수 있는 가장 안정적인 resolution을 선택하는 구조로 확장된다.

Thin film이나 narrow trench에서는 coarse voxel level이 서로 다른 material을 섞을 가능성이 높다. 이 경우 coarser fallback penalty를 크게 하고, fine level 또는 current data를 우선해야 한다.

### CFD / scientific visualization

AMR(Adaptive Mesh Refinement), cut-cell, phase interface에서는 requested resolution의 cell이 비활성·미정의일 수 있다. Fine/coarse hierarchy를 탐색하되 fluid/solid mask, boundary-condition type, refinement epoch가 일치하는 candidate만 선택하면 missing data와 physical zero를 구분할 수 있다.

Streamline·volume rendering에서도 empty space와 invalid sample을 같은 0으로 처리하지 않고 validity hierarchy를 사용하면 경계의 scalar dilution과 false interpolation을 줄일 수 있다.

### Graphics C++ 엔지니어링

이 개념은 shader 수식만으로 끝나지 않는다. C++에서 다음을 설계하고 설명할 수 있어야 한다.

- signal별 fallback policy table
- temporal resource lifetime과 reset 조건
- MIP build와 consumer pass scheduling
- packed metadata format과 precision budget
- GPU profiling counter와 debug visualization
- quality mode별 search radius와 fast-path 비율

면접과 포트폴리오에서는 “history가 invalid하면 버린다”보다 “validity hierarchy, confidence, footprint mismatch를 기반으로 복구하며 비용을 제한한다”는 설명이 훨씬 강한 시스템 설계 역량을 보여준다.

## 6. 머릿속에 남길 질문 3개

1. **Requested LOD가 invalid할 때 finer level의 높은 surface purity와 coarser level의 넓은 sample support 중 무엇을 signal별로 우선해야 하는가?**
2. **Level search를 first-valid 방식으로 끝낼 것인가, 여러 candidate score를 비교·blend할 것인가? 두 방식의 temporal stability와 GPU 비용 차이는 무엇인가?**
3. **History fallback이 자주 발생하는 영역에서 더 많은 search를 허용하는 것과 history를 빠르게 reset하는 것 중 어느 쪽이 장기적인 품질과 responsiveness에 유리한가?**

## 7. graphics engineer 면접 질문 1개와 답변

### 질문

**Temporal history sampling에서 요청된 MIP level이 invalid할 때 어떤 fallback 전략을 사용하며, GPU 비용은 어떻게 제한하겠습니까?**

### 답변

요청 LOD를 footprint에 가장 적합한 시작점으로 사용하되, 해당 level의 coverage·depth·normal·surface ID·revision·history confidence를 검증합니다. Confidence가 부족하면 요청 LOD 주변의 소수 MIP level을 target-centered 순서로 탐색하고, 각 후보에 surface validity와 LOD mismatch penalty를 적용해 score를 계산합니다.

Finer fallback은 surface detail을 보존하지만 variance와 aliasing이 늘고, coarser fallback은 support가 넓지만 blur와 cross-surface leakage 위험이 있습니다. 따라서 diffuse와 specular에 서로 다른 policy를 적용합니다. Sharp specular와 thin geometry는 finer bias를, low-frequency diffuse는 제한적인 coarse bias를 줄 수 있습니다.

GPU 비용은 search radius를 고정된 2~3 level로 제한하고, compact coarse metadata로 후보를 먼저 reject하며, tile 전체가 valid한 경우 hardware sampling fast path를 사용해 줄입니다. Fallback이 필요한 lane만 slow path로 보내고, valid candidate가 없으면 fast history 또는 current-frame sample로 전환합니다. 또한 selected LOD, fallback reason, slow-path 비율을 debug/profiling data로 노출해 품질과 비용을 함께 조정합니다.

## 8. 포트폴리오 / 커리어 연결

이 주제는 포트폴리오에서 단순 denoising 효과보다 **temporal data reliability architecture**를 보여주기에 좋다.

설명 구조:

1. 문제: requested footprint의 history가 disocclusion·mixed surface·topology change로 invalid해짐
2. 기존 방식의 한계: fixed LOD sampling, binary rejection, unconditional base-level fallback
3. 설계: validity metadata pyramid, bounded hierarchical search, signal-aware scoring
4. GPU 최적화: fast/slow path, fixed candidate count, metadata-first rejection, subgroup classification
5. 검증: selected-LOD view, fallback reason, ghosting/noise/performance 비교

게임 엔진·렌더링 팀에서는 temporal stability, resource layout, render graph synchronization, profiling을 한 번에 논의할 수 있다. Scientific visualization·반도체 모델링에서는 같은 구조를 AMR, sparse volume, topology revision, material boundary 문제에 연결할 수 있어 사용자의 렌더링과 시뮬레이션 경험을 하나의 강점으로 묶어준다.

## 9. 내일 이어서 볼 개념

**Confidence-Weighted History Inpainting and Same-Surface Neighborhood Recovery**

MIP hierarchy 안에서도 valid candidate를 찾지 못했을 때, screen-space 또는 surface-space neighborhood에서 같은 signal domain의 history를 복구하는 방법을 살펴본다. 단순 dilation과 달리 depth·normal·identity·motion·variance를 이용해 contamination을 제한하고, inpainting 결과의 confidence가 temporal accumulation에 어떻게 전달되어야 하는지 연결한다.

## 10. 참고 키워드

- Hierarchical History Search
- Validity-Driven LOD Fallback
- HistoryFix
- Fast History / Short-Term History
- History Confidence
- MIP Pyramid / MIP Chain
- Surface-Constrained Sampling
- Cross-Mip Validity
- Disocclusion Recovery
- Temporal Accumulation
- Footprint Mismatch
- Finer-First / Coarser-First Search
- Signal-Aware Policy
- Wave/Subgroup Classification
- Metadata-First Rejection
- NVIDIA Real-Time Denoisers (NRD): https://github.com/NVIDIA-RTX/NRD
- Direct3D 12 Sampler Feedback specification: https://microsoft.github.io/DirectX-Specs/d3d/SamplerFeedback.html
- AMD FidelityFX Single Pass Downsampler: https://gpuopen.com/fidelityfx-spd/
