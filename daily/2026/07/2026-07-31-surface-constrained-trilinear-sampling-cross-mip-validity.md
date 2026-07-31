---
title: "Surface-Constrained Trilinear Sampling and Cross-Mip Validity"
date: "2026-07-31"
category: Graphics
tags: ["GPU", "Temporal Filtering", "Trilinear Sampling", "Mipmap", "History Buffer", "Surface Validity", "Compute Shader", "Memory Layout", "Denoising"]
level: intermediate
---

# [Daily Graphics Growth] 2026-07-31 - Surface-Constrained Trilinear Sampling and Cross-Mip Validity

## 1. 오늘의 개념

전날에는 **Validity-Aware History Mipmap**과 **Guide-Preserving Downsampling**을 통해 temporal history의 상위 MIP이 단순 color 평균이 아니라 coverage, depth range, normal coherence, surface identity, confidence를 함께 보존해야 한다는 점을 다뤘다.

오늘은 그 pyramid를 읽는 단계인 **Surface-Constrained Trilinear Sampling**을 다룬다. 일반적인 trilinear filtering은 두 인접 MIP level에서 bilinear filtering을 수행한 뒤 fractional LOD로 두 결과를 보간한다. 2D texture 기준으로 최대 8개 texel이 결과에 기여하지만, 하드웨어는 각 texel이 같은 object·material·surface인지, history가 유효한지 알지 못한다.

Temporal history, reflection history, sparse volume, level-set field에서는 화면상 인접한 sample이 서로 다른 signal domain에 속할 수 있다. 따라서 위치 기반 interpolation weight에 depth, normal, identity, coverage, revision, temporal confidence를 곱하고 valid weight만 다시 normalize해야 한다.

**Cross-Mip Validity**는 fine level과 coarse level이 각각 유효한지만 보는 것이 아니라, 두 level이 현재 pixel에 대해 같은 surface domain을 대표하는지도 평가하는 개념이다. Fine sample이 현재 object와 일치해도 coarse texel이 mixed surface라면 단순 LOD fraction으로 둘을 섞어서는 안 된다.

이 용어는 특정 API의 표준 기능이라기보다, hardware texture filtering의 연속 신호 가정과 temporal·scientific data의 불연속성을 연결하는 graphics engineering design pattern으로 이해하는 편이 정확하다.

## 2. 한 줄 핵심

**History pyramid를 읽을 때는 8-tap trilinear weight에 depth·normal·identity·coverage·revision 기반 compatibility를 곱해 normalize하고, 두 MIP level의 유효 support가 다르면 LOD fraction보다 cross-mip confidence를 우선해야 surface leakage와 ghosting을 막을 수 있다.**

## 3. 왜 중요한가

일반적인 trilinear filtering은 다음 구조다.

`L = floor(lod)`, `t = frac(lod)`

`C0 = BilinearSample(texture[L], uv)`

`C1 = BilinearSample(texture[L + 1], uv)`

`C = lerp(C0, C1, t)`

이 계산은 주변 texel이 같은 연속 신호라는 전제를 사용한다. 하지만 screen-space history는 여러 surface domain이 한 texture에 겹쳐 저장된 구조다. 한 2×2 footprint 안에 foreground, background, disocclusion, 다른 material, 다른 topology revision이 동시에 들어올 수 있다.

이때 단순 filtering은 다음 artifact를 만든다.

- silhouette 주변의 dark halo와 cross-object ghosting
- thin geometry 뒤에서 나타나는 background history
- rough reflection의 specular bleeding
- curved reflection의 virtual-surface leakage
- level-set remeshing 이후의 topology ghosting
- sparse volume에서 missing sample이 실제 scalar zero처럼 평균되는 dilution

또한 coarse MIP은 여러 fine sample의 요약이다. Radiance가 valid sample만으로 정상적으로 normalize되었더라도 coverage가 낮고 depth interval이 넓거나 identity purity가 낮다면 현재 surface에 재사용할 수 없는 sample일 수 있다.

Jacobian-aware temporal filtering은 footprint가 넓을 때 낮은 MIP을 선택해 sample 수와 aliasing을 줄인다. 그러나 LOD가 올라갈수록 하나의 texel이 더 넓은 영역을 대표하므로 surface purity와 valid support가 더 중요해진다. 결국 LOD 선택은 footprint size와 valid support 사이의 타협이다.

실무에서는 품질과 비용도 충돌한다. Hardware trilinear는 한 번의 sample 명령으로 처리되지만 constrained sampling은 explicit tap, guide fetch, compatibility 계산, normalization이 필요하다. 따라서 fast path, metadata packing, fallback 정책, cache locality를 함께 설계해야 한다.

## 4. 구현 관점

### 4.1 8-tap weighted sampling

명시적 LOD `λ`에 대해 다음을 정의한다.

`l0 = floor(λ)`

`l1 = min(l0 + 1, maxMip)`

`t = frac(λ)`

각 level의 2×2 texel에 대한 bilinear weight를 `b_m,k`라고 하면 일반 trilinear weight는 다음과 같다.

`w_0,k = (1 - t) · b_0,k`

`w_1,k = t · b_1,k`

각 tap의 surface compatibility를 `g_m,k`라고 두면:

`ŵ_m,k = w_m,k · g_m,k`

`C = Σ_m Σ_k (ŵ_m,k · C_m,k) / max(Σ_m Σ_k ŵ_m,k, ε)`

`g`는 binary rejection mask일 수도 있고 `[0, 1]`의 soft confidence일 수도 있다.

### 4.2 Compatibility 구성

`g = g_valid · g_depth · g_normal · g_identity · g_revision · g_temporal`

- **Validity / coverage**: `g_valid = coverage · historyConfidence`
- **Depth**: current linear depth가 candidate의 `[z_min, z_max]`에 들어가는지 평가한다. Interval이 지나치게 넓으면 compactness penalty를 준다.
- **Normal**: mean normal과 normal cone을 사용해 current normal이 허용 방향 범위에 들어가는지 평가한다.
- **Identity**: object ID, material ID, virtual surface ID가 맞아야 한다. Coarse texel이 dominant ID와 purity를 저장한다면 ID match 시 purity를 weight로 사용한다.
- **Revision**: level-set remeshing, chunk rebuild, simulation reset이 발생했으면 generation counter 또는 topology epoch가 일치해야 한다.
- **Temporal**: history length, moments, variance, disocclusion confidence를 반영한다.

Hard rejection은 surface identity·revision처럼 의미가 불연속적인 값에 적합하다. Depth·normal·coverage는 soft confidence로 두면 경계에서 갑작스러운 popping을 줄일 수 있다.

### 4.3 Per-level normalization과 Cross-Mip blend

각 MIP level에서 먼저 normalized result와 effective support를 구한다.

`A_m = Σ_k (b_m,k · g_m,k · C_m,k)`

`W_m = Σ_k (b_m,k · g_m,k)`

`C_m = A_m / max(W_m, ε)`

그 다음 level weight를 계산한다.

`q0 = (1 - t) · supportConfidence(W0, metadata0)`

`q1 = t · supportConfidence(W1, metadata1)`

`C = (q0 · C0 + q1 · C1) / max(q0 + q1, ε)`

이 구조에서는 fractional LOD가 coarse level을 강하게 요구해도, 해당 level의 coverage·purity·compactness가 낮으면 fine level 비중을 높일 수 있다.

두 level 사이의 **cross-level consistency**도 검사한다.

- dominant surface ID가 같은가?
- representative depth 차이가 허용 범위 안인가?
- mean normal이 유사한가?
- revision ID가 같은가?
- coarse interval이 fine interval을 합리적으로 포함하는가?

Coarse level이 fine level과 다른 surface aggregate로 넘어갔다면 coarse weight를 낮추거나 제거한다.

### 4.4 Fallback 정책

총 effective support가 임계값보다 작으면 억지로 normalize하지 않는다.

가능한 fallback:

1. 한 단계 finer MIP으로 재시도
2. 가장 신뢰도 높은 단일 tap 사용
3. base-level nearest valid history 검색
4. current-frame sample 사용
5. history accumulation weight를 0으로 설정

Temporal denoising에서는 잘못된 history를 쓰는 것보다 history를 버리고 순간적인 noise를 허용하는 편이 안전하다. Scientific visualization에서는 missing data와 physical zero가 다른 의미이므로 signal semantics에 맞는 fallback이 필요하다.

### 4.5 Fast path와 Slow path

Coverage가 거의 1이고 depth interval이 compact하며 normal coherence와 identity purity가 높은 interior 영역은 hardware trilinear fast path를 사용할 수 있다.

Silhouette, mixed region, low-purity coarse MIP, topology revision boundary에서는 explicit tap을 평가하는 slow path를 사용한다. Classification pass를 별도로 두거나 coarse metadata를 먼저 읽어 경로를 선택할 수 있다.

단, branch divergence와 추가 texture fetch가 생기므로 실제 성능은 boundary pixel 비율과 GPU architecture에 따라 달라진다.

### 4.6 API와 shader 관점

- **OpenGL / GLSL**: `textureLod`, `texelFetch`, `textureGather`를 사용한다. Integer identity texture는 linear filtering이 불가능하므로 nearest 또는 exact fetch가 필요하다.
- **Vulkan / SPIR-V**: sampler의 `mipmapMode`와 explicit image fetch를 분리한다. Format filterability와 image layout transition을 확인한다.
- **Direct3D / HLSL**: `SampleLevel`, `Texture2D.Load`, `Gather`를 조합한다. SRV/UAV typed format compatibility도 고려한다.
- **WebGPU / WGSL**: `textureSampleLevel`, `textureLoad`를 사용하며 integer guide는 non-filtering 경로로 관리한다.

### 4.7 Memory layout

대표 resource 구성:

- history radiance: `RGBA16_FLOAT`
- coverage / confidence / history length: `RGBA8_UNORM` 또는 `R8_UNORM`
- depth min/max: `RG16_FLOAT` 또는 `RG32_FLOAT`
- mean normal + cone: octahedral normal과 cone angle을 packed format으로 저장
- identity / revision: `R32_UINT`, `RG16_UINT`, 또는 hash-packed integer
- temporal moments: `RG16_FLOAT`

Guide를 모두 별도 texture로 두면 bandwidth와 descriptor pressure가 커진다. 반대로 지나친 packing은 decode ALU, precision loss, format filterability 제약을 만든다. Categorical data와 continuous data는 분리하고, 함께 읽는 continuous metadata는 같은 cache line에 들어가도록 묶는 편이 좋다.

### 4.8 GPU 성능과 디버깅

수동 8-tap sampling은 guide까지 포함하면 pixel당 수십 번의 memory access가 될 수 있다.

최적화 포인트:

- `Gather`로 2×2 component fetch 축소
- coarse metadata로 early accept/reject
- branchless compatibility weight
- half storage + FP32 accumulator
- diffuse/specular별 필요한 guide 분리
- invalid tap이 많은 영역에서 early-out
- full-resolution과 half-resolution history 경로 분리

필수 debug view:

- requested LOD와 effective LOD
- 각 level의 support weight
- rejected tap count와 reason bit mask
- coverage, purity, depth compactness, normal coherence
- cross-mip consistency score
- fallback reason와 final history confidence

## 5. 내 관심 분야와 연결

### Real-time rendering과 denoising

Temporal denoising에서는 history reprojection 후 넓은 footprint를 읽을 때 surface-constrained sampling이 disocclusion, object boundary, glossy reflection의 잘못된 history를 제한한다. Specular history는 roughness, curvature, hit distance, virtual motion에 민감하므로 diffuse보다 stricter compatibility가 필요할 수 있다.

### Level-set / voxel / semiconductor emulation

Level-set 공정 형상은 remeshing과 topology change가 빈번하다. Material ID와 topology revision을 guide로 사용하면 이전 surface branch의 history가 현재 형상으로 섞이는 것을 막을 수 있다.

3D volume에서 일반 trilinear interpolation은 8개 voxel을 섞는다. Active mask, material ID, transform revision을 certainty로 사용하면 빈 voxel과 실제 값 0을 분리하고 thin-film boundary의 scalar leakage를 줄일 수 있다.

### CFD / scientific visualization

Obstacle boundary, phase interface, cut-cell, AMR level boundary에서는 화면상 또는 grid상 인접 sample을 무조건 섞으면 물리적 의미가 깨질 수 있다. Fluid/solid mask, boundary condition type, AMR revision을 validity로 사용하고 coarse/fine level의 cross-level consistency를 평가하는 구조로 확장할 수 있다.

### C++ 엔진 구조

Shader 내부 로직만이 아니라 다음 resource contract를 C++ render graph에 명시해야 한다.

- signal texture와 validity/guide texture
- MIP count와 coordinate convention
- pre-exposure state
- frame index와 reset epoch
- surface identity와 revision semantics
- pyramid build pass와 consumer pass의 synchronization

Algorithm, resource ownership, barrier, lifetime을 함께 설명할 수 있어야 graphics engineer 실무·면접에서 강점이 된다.

## 6. 머릿속에 남길 질문 3개

1. **Coarse MIP이 footprint에는 적합하지만 surface purity가 낮다면 aliasing 감소와 surface leakage 방지 중 무엇을 우선해야 하는가?**
2. **Depth·normal·identity·revision 중 어떤 guide를 hard rejection으로 두고 어떤 guide를 soft confidence로 둘 것인가?**
3. **수동 8-tap 비용을 줄이기 위해 어떤 metadata 조건에서 hardware trilinear fast path를 안전하게 허용할 수 있는가?**

## 7. graphics engineer 면접 질문 1개와 답변

### 질문

**Temporal history MIP을 sampling할 때 일반 hardware trilinear filtering을 그대로 사용하면 왜 문제가 생기며, 어떻게 개선할 수 있습니까?**

### 답변

Hardware trilinear filtering은 두 인접 MIP level의 주변 texel을 위치 기반으로 섞으며, sample들이 같은 연속 신호라는 가정을 사용합니다. 하지만 temporal history에는 서로 다른 object, depth, normal, material, disocclusion 상태가 인접할 수 있고 coarse MIP은 여러 surface가 섞인 aggregate일 수 있습니다.

개선 방법은 각 tap의 interpolation weight에 coverage, depth compatibility, normal cone, surface identity, revision, temporal confidence를 곱한 뒤 valid weight 합으로 normalize하는 것입니다. 두 MIP level은 개별 support를 계산하고, level 간 blend에는 LOD fraction뿐 아니라 support confidence와 cross-level consistency를 사용합니다. Support가 부족하면 finer MIP, nearest valid history, current frame 또는 history reset으로 fallback합니다.

성능을 위해 stable interior에서는 hardware trilinear fast path를 사용하고 silhouette·mixed region에서만 explicit constrained path를 실행할 수 있습니다. 구현 시 integer guide의 non-filterable 특성, metadata packing, bandwidth, render graph synchronization까지 함께 고려해야 합니다.

## 8. 포트폴리오 / 커리어 연결

이 주제는 “denoising filter를 추가했다”보다 **temporal data representation과 sampling contract를 설계했다**는 서사로 연결하기 좋다.

포트폴리오에서 강조할 항목:

- Jacobian 기반 requested LOD와 validity 기반 effective LOD의 분리
- coverage·depth interval·normal cone·identity purity의 역할
- hardware fast path와 explicit slow path 조건
- diffuse/specular별 compatibility 차이
- level-set topology revision과 history invalidation 연계
- GPU capture에서 bandwidth와 frame-time 비교
- artifact별 debug visualization과 failure analysis

좋은 시각 자료는 일반 trilinear와 constrained sampling의 silhouette 비교, tap rejection overlay, requested/effective LOD heatmap, cross-mip consistency score, fast-path 비율이다.

면접에서는 “Temporal history는 일반 color texture와 달리 surface semantics가 있으므로 MIP generation과 sampling 모두 validity contract를 유지해야 한다”고 설명하고, reduction metadata·cross-mip compatibility·fallback·GPU layout을 하나의 pipeline으로 연결하면 좋다.

## 9. 내일 이어서 볼 개념

**Hierarchical History Search and Validity-Driven LOD Fallback**

오늘은 요청된 두 MIP level 안에서 valid tap을 선별하고 cross-mip blend를 조정했다. 내일은 local support가 부족할 때 finer/coarser level을 탐색하고 nearest valid history 또는 current-frame fallback을 결정하는 계층적 검색 구조를 살펴본다.

연결 포인트: coarse-to-fine search, validity pyramid early termination, bounded iteration, fallback radius, nearest valid surface, GPU divergence, history rejection과 noise 증가의 trade-off.

## 10. 참고 키워드

- Surface-Constrained Sampling
- Guide-Aware Interpolation
- Validity-Aware Trilinear Filtering
- Cross-Mip Validity
- Normalized Convolution
- Certainty-Weighted Filtering
- Temporal History Pyramid
- Fractional LOD / Effective LOD
- Surface Identity / Identity Purity
- Depth Interval / Normal Cone
- Coverage / History Confidence
- Revision ID / Topology Epoch
- Explicit Tap Filtering
- `textureLod` / `texelFetch` / `textureGather`
- `SampleLevel` / `Texture2D.Load`
- `textureSampleLevel` / `textureLoad`
- Fast Path / Slow Path
- Sparse Volume Validity
- AMR Cross-Level Sampling

참고 기반: Khronos OpenGL texture filtering 문서, Microsoft Direct3D texture filtering 문서, NVIDIA NRD 문서, Knutsson·Westin의 normalized convolution 연구.
