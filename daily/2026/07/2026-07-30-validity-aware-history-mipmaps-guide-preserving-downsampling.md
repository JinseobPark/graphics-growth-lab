---
title: "Validity-Aware History Mipmaps and Guide-Preserving Downsampling"
date: "2026-07-30"
category: Graphics
tags: ["GPU", "Temporal Filtering", "Mipmap", "Guide-Aware Filtering", "Downsampling", "History Buffer", "Surface Identity", "Compute Shader", "Memory Layout", "Denoising"]
level: intermediate
---

# [Daily Graphics Growth] 2026-07-30 - Validity-Aware History Mipmaps and Guide-Preserving Downsampling

## 1. 오늘의 개념

전날에는 **Elliptical History Footprint**와 **Jacobian-Aware Temporal Filtering**을 통해 현재 pixel이 이전 프레임에서 단일 texel이 아니라 크기와 방향을 가진 영역에 대응한다는 관점을 다뤘다. Footprint가 넓어질수록 여러 history sample을 읽거나 더 낮은 MIP level을 선택하는 편이 sampling 관점에서는 자연스럽다.

그러나 temporal history는 일반 color texture와 다르다. 서로 다른 표면, 깊이, 법선, material, object identity, disocclusion 상태가 한 화면 안에 공존하며, 단순 평균으로 MIP pyramid를 만들면 서로 섞이면 안 되는 정보가 상위 level에서 이미 혼합된다.

**Validity-Aware History Mipmap**은 history radiance만 축소하는 것이 아니라 다음 정보를 함께 보존하는 temporal pyramid다.

- 유효한 history sample의 비율인 **coverage / validity**
- 표면 깊이 범위인 **depth interval**
- 평균 normal과 방향 분산을 나타내는 **normal cone / normal coherence**
- 동일 표면 여부를 나타내는 **surface identity / material ID / object ID**
- temporal moments, variance, history length, confidence
- remeshing·LOD 전환·simulation update를 나타내는 **revision metadata**

**Guide-Preserving Downsampling**은 2×2 또는 그 이상의 source footprint를 줄일 때 radiance 평균만 계산하지 않고, 어떤 sample들이 같은 signal domain에 속하는지 판단할 수 있는 guide를 함께 축약하는 방식이다.

핵심은 MIP level을 단순한 저해상도 color image로 보지 않고, 특정 화면 영역에 포함된 history의 **통계적 요약(statistical summary)**과 **기하학적 유효성 계약(geometric validity contract)**으로 보는 것이다.

## 2. 한 줄 핵심

**Temporal history MIP은 color의 평균 pyramid가 아니라, 유효 sample의 가중합·coverage·depth range·normal coherence·surface identity를 함께 보존하는 guide-aware pyramid여야 하며, 그렇지 않으면 넓은 footprint를 읽는 순간 geometry leakage와 ghosting이 상위 MIP에서 구조적으로 발생한다.**

## 3. 왜 중요한가

### 일반 color MIP의 평균은 surface boundary를 모른다

일반적인 color downsampling은 2×2 texel을 다음처럼 평균내는 형태로 이해할 수 있다.

`C_mip = (C0 + C1 + C2 + C3) / 4`

이 계산은 네 texel이 동일한 연속 신호를 sample한다고 가정한다. Albedo texture나 environment map처럼 texture domain이 연속적으로 정의된 경우에는 합리적이다.

하지만 screen-space history에서는 2×2 block 안에서도 다음 상황이 발생한다.

- foreground object와 background가 절반씩 포함됨
- 한 texel은 disocclusion으로 history가 없음
- 서로 다른 material ID가 경계를 공유함
- curved reflection의 virtual surface identity가 달라짐
- level-set remeshing으로 이전 topology와 대응하지 않음
- sparse volume에서 빈 공간과 실제 값 0이 함께 존재함

이때 단순 평균은 invalid sample을 검은색 또는 0으로 취급해 radiance를 어둡게 만들거나, 서로 다른 surface의 radiance를 섞어 경계 색을 만든다. 한 번 상위 MIP에 섞인 정보는 이후 depth test나 object ID test로 완전히 분리할 수 없다.

### 넓은 footprint일수록 MIP 내부의 오염이 크게 보인다

Jacobian으로 구한 history footprint가 큰 경우 `LOD ≈ log2(footprint size)`에 따라 낮은 해상도의 history를 읽고 싶어진다. 이는 bandwidth와 aliasing 측면에서 유리하다.

하지만 MIP pyramid가 geometry-aware하지 않으면 다음 artifact가 발생한다.

- foreground 색이 background reflection으로 번지는 **surface leakage**
- thin geometry가 상위 MIP에서 사라지는 **thin-feature collapse**
- invalid history가 평균에 포함되는 **dark halo**
- 서로 다른 object history가 섞이는 **cross-object ghosting**
- rough reflection에서 boundary가 과도하게 확장되는 **specular bleeding**
- remeshed iso-surface가 이전 topology의 색을 끌고 오는 **topology ghosting**

즉 footprint-aware sampling을 제대로 하려면, 먼저 footprint가 읽을 수 있는 history pyramid 자체가 surface semantics를 보존해야 한다.

### Downsampling은 정보 손실이 아니라 정보 요약 문제다

한 MIP texel은 source의 여러 texel을 대표한다. 이때 하나의 평균 color만 저장하면 source 영역의 상태를 충분히 설명하지 못한다.

예를 들어 같은 평균 color를 가진 두 block을 생각할 수 있다.

- Block A: 네 texel 모두 동일한 안정적 surface
- Block B: 두 개의 서로 다른 object가 절반씩 포함됨

평균 color만 보면 둘은 같지만 temporal reuse 관점에서는 전혀 다르다. Block A는 신뢰할 수 있고, Block B는 boundary 또는 mixed region으로 취급해야 한다.

따라서 history pyramid의 각 texel은 최소한 다음 질문에 답할 수 있어야 한다.

- 유효한 source가 얼마나 있었는가?
- 하나의 surface가 지배적인가, 여러 identity가 섞였는가?
- depth 범위가 좁은가 넓은가?
- normal 방향이 일관적인가?
- history confidence와 age가 충분한가?
- 상위 MIP sample을 현재 pixel에 사용할 수 있는가?

### GPU 최적화와 품질이 직접 연결된다

MIP pyramid는 큰 footprint를 소수의 texture lookup으로 처리하게 해준다. 하지만 guide texture까지 여러 개 만들면 memory와 bandwidth가 증가한다.

결국 설계 목표는 다음 두 가지를 동시에 만족하는 것이다.

1. 넓은 history footprint를 적은 sample 수로 재구성한다.
2. 상위 MIP에서도 surface identity와 validity를 잃지 않는다.

이 문제는 단순 filter tuning이 아니라 **resource contract, reduction operator, memory layout, render graph scheduling**을 함께 설계하는 graphics engineering 문제다.

## 4. 구현 관점

### 4.1 Radiance와 validity를 분리해 생각한다

History radiance `C_i`와 sample validity `v_i`를 분리한다.

- `v_i = 1`: 유효한 history
- `v_i = 0`: disocclusion, reset, out-of-bounds, topology mismatch 등으로 무효
- 연속 confidence를 사용하면 `v_i ∈ [0, 1]`

단순 평균 대신 다음과 같은 **normalized weighted reduction**을 사용할 수 있다.

`W = Σ_i (k_i · v_i)`

`C̄ = Σ_i (k_i · v_i · C_i) / max(W, ε)`

여기서 `k_i`는 2×2 box weight 또는 downsampling kernel weight다.

이 방식의 중요한 성질은 invalid sample을 radiance 0으로 평균내지 않는다는 점이다. 유효 sample이 하나뿐이면 그 sample이 대표값이 되고, 전부 invalid이면 상위 texel도 invalid가 된다.

Coverage는 다음처럼 저장할 수 있다.

`coverage = W / Σ_i k_i`

- `coverage = 1`: source footprint 전체가 유효
- `coverage = 0.5`: 절반만 유효
- `coverage = 0`: 유효 history 없음

Coverage는 radiance normalization과 분리되어야 한다. Color는 valid sample만으로 normalize하되, coverage는 상위 MIP의 신뢰도와 filter weight를 낮추는 metadata로 사용된다.

### 4.2 Premultiplied representation과 normalized convolution

Radiance pyramid를 만들 때 다음 두 표현이 가능하다.

#### A. 매 level에서 normalized color를 저장

- `color = weightedColor / weight`
- `weight = coverage 또는 accumulated weight`

읽을 때 color와 weight를 함께 사용한다.

#### B. weighted sum과 weight를 저장

- `sumColor = Σ(v_i · C_i)`
- `sumWeight = Σ(v_i)`

필요할 때 `sumColor / sumWeight`로 복원한다.

B 방식은 여러 level을 다시 downsample할 때 associative reduction에 더 가깝다. 단, HDR radiance의 누적 범위가 커지므로 FP16 overflow와 precision 문제가 커질 수 있다. 실제 GPU pipeline에서는 pre-exposed radiance, level별 normalization, FP32 accumulator 후 FP16 storage를 조합하는 형태가 현실적이다.

이 구조는 image processing의 **normalized convolution**과 유사하다. Signal 값과 certainty를 별도로 누적한 뒤 certainty 합으로 normalize한다.

### 4.3 Surface identity는 평균낼 수 없는 categorical data다

Object ID, material ID, mesh region ID, simulation revision ID는 연속값이 아니라 범주형 값이다. 이를 `R32_UINT`로 저장해도 일반 linear sampling이나 평균 reduction을 적용할 수 없다.

2×2 source의 identity가 모두 같다면 상위 texel은 해당 ID를 그대로 유지할 수 있다.

`id0 == id1 == id2 == id3 → stable ID`

하나라도 다르면 다음 선택지가 있다.

- **Mixed flag**를 세우고 상위 texel의 identity를 무효화
- 가장 큰 coverage를 가진 **dominant ID**를 저장하되 purity를 별도 저장
- bit mask 또는 small-set encoding으로 여러 ID를 요약
- stable region hash와 revision ID를 결합

Temporal history에서는 dominant ID만 저장하는 방식이 위험할 수 있다. 3:1 비율로 foreground가 우세하더라도 minority background가 color 평균에 포함되면 이미 leakage가 생긴다.

보수적인 baseline은 다음과 같다.

- identity가 모두 같을 때만 radiance를 함께 축소
- identity가 다르면 mixed texel로 표시
- mixed texel은 더 낮은 confidence 또는 더 세밀한 MIP 사용

고급 방식에서는 current pixel의 identity와 일치하는 source만 선택적으로 reduction할 수 있지만, 한 texel 안에 여러 identity별 color를 보존하려면 layered representation이 필요해 memory가 급격히 증가한다.

### 4.4 Depth는 평균보다 범위가 중요하다

Depth를 하나의 평균값으로 축소하면 foreground와 background가 섞인 block이 중간 깊이처럼 보인다. 이 값은 실제 surface를 나타내지 않는다.

History guide pyramid에는 다음 depth summary가 유용하다.

- `z_min`: source의 최소 depth
- `z_max`: source의 최대 depth
- `z_rep`: 대표 depth, 예를 들어 nearest depth 또는 identity-matched weighted depth
- `z_variance`: depth spread의 통계값

현재 sample depth `z_c`가 상위 MIP texel과 호환되는지는 interval test로 평가할 수 있다.

`z_c ∈ [z_min - τ, z_max + τ]`

그러나 range가 너무 넓으면 여러 surface가 포함된 mixed region일 가능성이 높다.

`depthCompactness = exp(-α · (z_max - z_min) / max(|z_rep|, ε))`

Depth range는 perspective scene에서 view-space linear depth로 유지하는 편이 해석하기 쉽다. Scientific visualization처럼 좌표 범위가 크거나 단위가 매우 다양한 경우 FP16 range와 precision을 별도로 검토해야 한다.

### 4.5 Normal은 평균 방향과 coherence를 함께 저장한다

Normal을 단순 평균한 뒤 normalize하면 서로 반대 방향의 normal이 상쇄되어 불안정한 결과가 나올 수 있다.

Weighted normal sum을 다음처럼 정의한다.

`N_sum = Σ_i (w_i · n_i)`

`N_mean = normalize(N_sum)`

Normal coherence는 다음처럼 근사할 수 있다.

`coherence = |N_sum| / max(Σ_i w_i, ε)`

- `coherence ≈ 1`: normal이 거의 같은 방향
- `coherence ≈ 0`: 방향이 크게 분산되거나 상쇄됨

Normal cone angle은 다음처럼 근사 가능하다.

`coneAngle ≈ acos(clamp(coherence, 0, 1))`

정확한 최대 angular deviation을 저장하려면 `max(acos(dot(N_mean, n_i)))`를 사용할 수 있다. 평균 normal과 cone angle을 함께 저장하면 current normal이 상위 MIP의 surface orientation 범위에 들어가는지 검사할 수 있다.

`dot(n_current, N_mean) ≥ cos(coneAngle + τ_n)`

Sharp edge, curved reflection, coarse geometry에서는 cone이 넓어진다. 이 값은 sample rejection뿐 아니라 history confidence와 LOD cap에도 사용할 수 있다.

### 4.6 Moments와 variance의 축소

Temporal denoising에서 luminance first/second moment를 저장한다고 하자.

`m1 = E[L]`

`m2 = E[L²]`

`variance = max(m2 - m1², 0)`

MIP reduction에서도 valid weight를 사용해 moments를 축소할 수 있다.

`m1_mip = Σ(w_i · m1_i) / W`

`m2_mip = Σ(w_i · m2_i) / W`

여기서 중요한 점은 상위 footprint 안의 평균 차이 자체도 variance에 포함된다는 것이다. 서로 다른 source texel의 mean이 다르면, 단순히 각 texel의 variance만 평균내는 것보다 상위 level variance가 커져야 한다.

Law of Total Variance 관점에서는 다음 두 항이 필요하다.

`Var_total = E[Var_local] + Var(E_local)`

- `E[Var_local]`: 각 source history 내부의 noise variance
- `Var(E_local)`: footprint 내부 signal mean의 공간적 차이

두 항을 구분하면 geometry edge와 실제 Monte Carlo noise를 더 잘 분리할 수 있다. 다만 mixed surface에서 큰 variance가 나온다고 해서 그 영역을 단순히 넓게 blur하면 leakage가 커지므로 identity와 depth guide가 상위 조건이다.

### 4.7 History length와 confidence는 보수적으로 축소한다

History length는 sample count와 유사하지만 단순 평균이 항상 안전하지 않다.

예를 들어 source history length가 `[32, 32, 32, 1]`이면 평균은 24.25지만, footprint의 일부는 거의 새 history다. 넓은 MIP sample 전체를 24-frame history로 신뢰하면 lag와 ghosting이 생길 수 있다.

가능한 reduction policy는 다음과 같다.

| Metadata | 공격적 reduction | 보수적 reduction |
|---|---|---|
| History length | weighted mean | minimum 또는 낮은 percentile |
| Confidence | weighted mean | minimum, harmonic mean, product-like reduction |
| Validity | average coverage | minimum validity + coverage 병행 |
| Revision | dominant revision | all-equal check |

실시간 renderer에서는 모든 metadata를 지나치게 보수적으로 줄이면 상위 MIP가 거의 사용되지 않는다. 따라서 다음처럼 역할을 분리할 수 있다.

- coverage: 유효 sample 비율
- identity purity: 같은 surface 비율
- depth compactness: geometry separation 정도
- normal coherence: 방향 일관성
- history confidence: lighting 변화와 temporal stability

최종 상위 MIP confidence는 이들을 곱하거나 min-combine한 값으로 구성할 수 있다.

`confidence_mip = coverage · identityPurity · depthCompactness · normalCoherence · temporalConfidence`

곱셈은 한 요소가 낮으면 빠르게 0으로 수렴한다. 실제 production에서는 각 항의 curve와 floor를 조정하거나, hard rejection과 soft confidence를 분리한다.

### 4.8 MIP 생성 pass 구조

#### Multi-pass downsampling

각 MIP level을 순차적으로 생성한다.

`Mip 0 → Mip 1 → Mip 2 → ...`

장점:

- 구현과 디버깅이 단순함
- level별 UAV/SRV dependency가 명확함
- custom guide reduction을 적용하기 쉬움
- 각 level의 debug visualization이 편리함

단점:

- dispatch와 barrier 수가 증가함
- 작은 MIP level에서 GPU utilization이 낮음
- 여러 guide texture를 각각 처리하면 pass 수가 많아짐

#### Single-pass downsampling

AMD FidelityFX SPD와 같은 구조는 하나의 dispatch에서 여러 MIP level을 생성해 dispatch overhead와 intermediate synchronization을 줄인다. Wave operation 또는 LDS(Local Data Share)를 활용해 작은 level을 workgroup 내부에서 계속 축소한다.

History pyramid에서 single-pass 구조를 사용할 때는 color average 하나가 아니라 다음 reduction을 함께 처리해야 한다.

- radiance weighted sum
- validity weight
- depth min/max
- normal sum/coherence
- identity equality/mixed flag
- moments
- history metadata

Reduction operator가 복잡할수록 register pressure, LDS usage, format conversion 비용이 증가한다. 따라서 첫 구조는 multi-pass가 검증에 유리하고, 병목이 확인된 뒤 pass fusion 또는 SPD-style scheduling을 검토하는 흐름이 합리적이다.

### 4.9 MIP 선택과 history footprint의 연결

Jacobian `J`의 최대 singular value를 `σmax`라고 하면 isotropic MIP 선택은 대략 다음처럼 표현할 수 있다.

`lod = log2(max(σmax, 1))`

하지만 anisotropy가 큰 경우 `σmax`만 따라 MIP을 낮추면 minor axis 방향까지 과도하게 blur된다.

`anisotropy = σmax / max(σmin, ε)`

- anisotropy가 낮음: trilinear MIP sampling이 적절
- anisotropy가 높음: moderate MIP + major-axis taps가 더 적절
- near-singular mapping: MIP 확대보다 history confidence 감소 또는 rejection이 안전

Validity-aware pyramid를 사용하더라도 hardware trilinear sampling을 그대로 적용하면 서로 다른 MIP level의 metadata가 다시 선형 혼합될 수 있다.

예를 들어 MIP L과 L+1을 각각 읽고 다음처럼 처리하는 방식이 더 명시적이다.

1. 각 level의 radiance와 guide를 별도로 fetch
2. current surface와 각 level의 validity를 독립적으로 평가
3. valid level만 fractional LOD weight로 normalize

`C = (w0 · v0 · C0 + w1 · v1 · C1) / max(w0 · v0 + w1 · v1, ε)`

이것은 cross-MIP에서도 normalized convolution을 적용하는 형태다.

### 4.10 Resource layout과 memory cost

2D texture의 전체 MIP chain 면적은 다음 geometric series에 가깝다.

`1 + 1/4 + 1/16 + ... ≈ 4/3`

즉 base level만 저장할 때보다 전체 chain은 약 33.3%의 추가 texel을 사용한다.

1080p `RGBA16_FLOAT` history radiance를 예로 들면:

- base level: 약 15.8 MiB
- 추가 MIP level: 약 5.3 MiB
- 전체 chain: 약 21.1 MiB

4K에서는:

- base level: 약 63.3 MiB
- 추가 MIP level: 약 21.1 MiB
- 전체 chain: 약 84.4 MiB

여기에 guide pyramid가 추가된다.

| Resource | 연구용 baseline | production packing 관점 |
|---|---|---|
| Radiance | `RGBA16_FLOAT` | diffuse/specular 분리, pre-exposed FP16 |
| Validity/Coverage | `R16_FLOAT` | `R8_UNORM` |
| Depth range | `RG32_FLOAT` | scene scale에 따라 `RG16_FLOAT` |
| Mean normal | `RGBA16_FLOAT` | oct normal `RG16_SNORM` |
| Normal cone/coherence | `R16_FLOAT` | `R8_UNORM` |
| Surface identity | `R32_UINT` | packed ID + mixed bit |
| Moments | `RG16_FLOAT` | luminance range에 따라 FP16/FP32 |
| History length/confidence | `RG16_FLOAT` | `R8_UINT + R8_UNORM` |

모든 guide를 full chain으로 만들 필요는 없다.

- depth·normal·identity는 작은 MIP 몇 level까지만 유지
- 매우 coarse level은 history sample 자체를 금지
- specular만 별도 pyramid 사용
- full-resolution identity와 coarse coverage를 결합
- packed metadata texture로 bandwidth를 줄임

실제 병목은 MIP 생성 ALU보다 여러 source texture read와 destination write에서 발생할 가능성이 높다. RenderDoc, PIX, Nsight Graphics에서 pass별 bytes read/write, cache hit, occupancy를 함께 보는 이유다.

### 4.11 C++ render graph와 API contract

C++ renderer에서는 history pyramid를 단일 texture resource의 여러 subresource로 관리하거나, level별 texture로 분리할 수 있다.

단일 resource 방식:

- descriptor 수가 적음
- hardware sampling과 `textureLod` 사용이 편리함
- level별 UAV view와 subresource barrier 관리가 필요함

분리 resource 방식:

- render graph dependency가 명확함
- format을 level별로 다르게 구성 가능
- descriptor와 resource 관리 비용 증가

API별 공통 고려점은 다음과 같다.

- Vulkan: mip별 image view, storage image layout transition, subresource barrier
- Direct3D 12: mip slice별 UAV descriptor, resource state transition 또는 UAV barrier
- OpenGL: `glBindImageTexture`의 level binding, texture barrier와 dispatch ordering
- WebGPU: mip별 texture view, storage binding format 제한, dispatch 간 pass ordering

Integer identity는 filtered sampler가 아니라 exact load가 필요하다. Radiance는 sampled texture, guide는 storage/read-only texture, identity는 integer texture로 분리하는 resource contract가 명확하다.

### 4.12 대표 failure mode와 debug view

#### Invalid-as-zero

Invalid history를 color 0으로 평균내 dark halo가 생긴다.

Debug:

- coverage heatmap
- weighted sum과 normalized color 비교

#### Dominant-ID leakage

상위 MIP이 dominant ID를 유지하지만 minority surface color가 평균에 포함된다.

Debug:

- identity purity
- mixed flag
- source ID distribution

#### Depth-average phantom surface

Foreground/background 평균 depth가 실제로 존재하지 않는 중간 surface를 만든다.

Debug:

- `z_min`, `z_max`, `z_max-z_min`

#### Normal cancellation

서로 다른 normal이 평균에서 상쇄되어 mean normal이 불안정하다.

Debug:

- normal coherence
- cone angle

#### Coarse-level overuse

Footprint가 크다는 이유만으로 지나치게 coarse MIP을 사용해 thin feature와 sharp reflection이 사라진다.

Debug:

- selected LOD
- `σmax`, `σmin`, anisotropy
- MIP confidence

#### Stale revision reuse

Level-set remeshing, voxel chunk rebuild, simulation timestep change 후 이전 revision의 history가 상위 MIP에 남는다.

Debug:

- revision ID
- topology reset mask
- per-level valid coverage

## 5. 내 관심 분야와 연결

### CFD와 scientific visualization

CFD scalar field의 값 0과 invalid cell은 의미가 다르다. Invalid cell을 0으로 평균내면 boundary layer, shock, vortex core 주변의 값을 인위적으로 낮춘다.

Validity-aware pyramid는 다음 시각화에 연결된다.

- scalar heatmap의 empty/invalid mask 보존
- adaptive mesh refinement(AMR) level 간 coverage 보존
- vector field의 평균 방향과 magnitude 분리
- streamline density history의 valid region 축소
- iso-surface temporal accumulation의 topology revision 관리

Vector field는 normal과 유사하게 방향 평균만 저장하면 cancellation이 발생한다. 평균 vector, magnitude statistics, directional coherence를 분리하는 구조가 필요하다.

### Level-Set · Marching Cubes · semiconductor process emulation

Level-set surface는 공정 step마다 geometry가 변하고, deposition·etch 과정에서 topology가 생성·소멸할 수 있다. Screen-space motion이 작더라도 이전 surface와 현재 surface가 동일하다는 보장은 없다.

History pyramid에 다음 metadata를 연결할 수 있다.

- material ID
- process step ID
- level-set revision ID
- mesh generation epoch
- connected-component 또는 region label

특히 doping heatmap이나 material boundary를 temporal filtering할 때 서로 다른 material의 scalar value를 상위 MIP에서 평균내면 물리적으로 존재하지 않는 혼합 영역이 시각화될 수 있다. Color만 guide-aware하게 만드는 것이 아니라, 원본 scalar field의 validity와 material identity를 먼저 보존하는 편이 안전하다.

### Sparse voxel · octree · NanoVDB

Sparse volume에서는 empty space와 resident brick이 명확히 구분된다. History MIP의 coverage는 sparse residency와 유사한 역할을 한다.

- brick residency
- voxel validity
- LOD revision
- material occupancy
- active voxel fraction

Sparse texture 또는 virtual texture를 사용할 경우 coarse history MIP이 존재해도 필요한 fine tile이 resident하지 않을 수 있다. Direct3D 12 Sampler Feedback과 같은 기술은 어떤 MIP region이 실제로 요청되는지 기록해 streaming 결정을 돕는다. Temporal history에서도 footprint-driven LOD 통계를 통해 필요한 history level과 tile을 제한하는 방향으로 확장할 수 있다.

### Real-time rendering과 game engine

Game engine에서는 TAA, ray-traced reflection denoising, GI reconstruction, volumetric fog history, screen-space effects가 모두 temporal resource를 가진다.

공통 history pyramid abstraction을 만들면 다음 contract를 공유할 수 있다.

- radiance/signal
- validity/coverage
- depth interval
- normal cone
- identity/revision
- moments/confidence

하지만 모든 effect가 같은 guide를 요구하는 것은 아니다.

- diffuse GI: depth·normal·material identity 중심
- specular: roughness·virtual hit·path identity 추가
- volumetric: transmittance·depth interval·phase-related metadata
- CFD visualization: field revision·cell validity·physical range

따라서 engine architecture에서는 fixed G-buffer pyramid보다 **effect-specific reduction policy를 가진 generic pyramid framework**가 더 확장성이 높다.

## 6. 머릿속에 남길 질문 3개

1. **상위 history MIP에서 radiance는 valid sample만으로 normalize하면서 coverage는 별도로 유지해야 하는 이유는 무엇이며, invalid sample을 color 0으로 취급하면 어떤 bias가 생기는가?**

2. **Depth average 대신 depth interval, normal average 대신 normal coherence/cone, object ID 평균 대신 identity purity를 저장해야 하는 공통 원리는 무엇인가?**

3. **Jacobian footprint가 매우 크더라도 MIP texel의 identity가 mixed이고 depth range가 넓다면, 더 coarse MIP을 선택하는 것과 history confidence를 낮추는 것 중 어떤 정책이 더 안전한가?**

## 7. graphics engineer 면접 질문 1개와 답변

### 질문

Temporal denoiser에서 큰 reprojection footprint를 처리하기 위해 history MIP pyramid를 도입하려고 합니다. 일반 color mipmap을 그대로 사용할 때 발생하는 문제와, GPU-friendly한 guide-preserving downsampling 구조를 설명해보세요.

### 답변

일반 color mipmap은 source texel이 동일한 연속 신호에 속한다고 가정하고 평균을 계산한다. Screen-space history에서는 한 2×2 block 안에 foreground/background, 서로 다른 object나 material, disocclusion, invalid history가 함께 존재할 수 있다. 이를 단순 평균하면 상위 MIP에 surface leakage, dark halo, cross-object ghosting이 생성되고, 이후 depth나 ID validation으로 이미 혼합된 radiance를 분리할 수 없다.

GPU-friendly baseline은 radiance와 validity weight를 함께 축소하는 normalized reduction이다. 각 source의 confidence 또는 validity를 `v_i`라고 하면 `sumColor = Σ(v_i C_i)`, `sumWeight = Σv_i`를 계산하고, 상위 color는 `sumColor / sumWeight`로 만든다. Coverage는 별도로 저장해 coarse texel의 유효 면적을 나타낸다.

Guide는 signal 종류에 따라 다른 reduction operator를 사용한다. Depth는 평균보다 min/max interval, normal은 normalized mean과 coherence 또는 cone angle, object/material ID는 all-equal check와 mixed flag, moments는 first/second moment, history length와 confidence는 보수적 min 또는 weighted policy를 사용한다. Current pixel은 상위 MIP sample을 읽은 뒤 depth interval, normal cone, identity purity, coverage를 검사해 history weight를 결정한다.

C++ render graph에서는 mip별 UAV/SRV view와 dependency를 관리하고, compute shader가 level별 2×2 reduction을 수행하는 multi-pass 방식이 검증에 유리하다. 성능이 문제가 되면 wave operation과 LDS를 사용하는 single-pass downsampler 구조로 합칠 수 있다. `RGBA16F` radiance chain은 base level 대비 약 33% 추가 memory가 필요하며, guide pyramid까지 포함하면 bandwidth가 주요 병목이 된다. 따라서 packed metadata, 제한된 최대 MIP, diffuse/specular 선택적 pyramid, pass fusion을 통해 비용을 조절한다.

## 8. 포트폴리오 / 커리어 연결

이 주제는 단순히 “mipmap을 만들었다”가 아니라, texture filtering 이론을 temporal reconstruction과 geometry semantics에 적용하는 능력을 보여준다.

포트폴리오에서 드러낼 수 있는 내용은 다음과 같다.

- 일반 box-filter history MIP과 validity-aware MIP 비교
- foreground/background 경계에서 leakage·dark halo 시각화
- coverage, depth range, normal cone, identity purity debug view
- Jacobian-selected LOD와 MIP confidence heatmap
- bilinear/trilinear hardware sampling과 explicit normalized cross-MIP sampling 비교
- multi-pass compute와 single-pass downsampling의 GPU time·bandwidth 비교
- `RGBA16F` guide baseline과 packed production layout의 memory 비교
- thin geometry, curved reflection, disocclusion, remeshing failure case 분석
- ray-traced reflection과 CFD/level-set visualization에 같은 abstraction 적용
- render graph resource lifetime과 subresource barrier diagram

면접에서는 다음 역량을 연결해 설명할 수 있다.

- categorical data와 continuous signal의 reduction 차이
- normalized convolution과 temporal validity
- G-buffer guide의 의미와 failure mode
- compute shader reduction, wave operation, shared memory
- GPU bandwidth와 texture format trade-off
- C++ resource abstraction과 API portability
- artifact를 sampling·geometry·statistics 문제로 분해하는 사고

Nintendo, Unity, real-time engine, scientific visualization 팀 모두에서 중요한 점은 feature 이름보다 문제 정의다. “왜 일반 MIP이 실패하는가”, “어떤 정보는 평균낼 수 없는가”, “어떤 metadata가 coarse footprint의 신뢰도를 설명하는가”를 명확히 말할 수 있으면 rendering pipeline과 simulation visualization을 함께 이해하는 engineer로 차별화된다.

## 9. 내일 이어서 볼 개념

**Surface-Constrained Trilinear Sampling and Cross-Mip Validity**

오늘은 history pyramid를 만드는 단계에서 radiance와 geometry guide를 어떻게 보존할지 다뤘다. 다음에는 두 MIP level 사이를 trilinear interpolation할 때 validity·identity·depth interval이 다시 섞이는 문제를 분석하고, level별 validation 후 valid weight만 normalize하는 cross-MIP reconstruction, anisotropic taps와 coarse-level rejection policy로 이어간다.

## 10. 참고 키워드

- Validity-Aware History Mipmap
- Guide-Preserving Downsampling
- Normalized Convolution
- Coverage / Validity Weight
- Premultiplied Signal
- Weighted Sum / Weight Pyramid
- Surface Identity
- Identity Purity
- Mixed Surface Flag
- Depth Interval
- Depth Min-Max Pyramid
- Normal Cone
- Normal Coherence
- Temporal Moments
- Law of Total Variance
- History Length
- History Confidence
- Disocclusion
- Geometry Leakage
- Cross-Object Ghosting
- Thin-Feature Collapse
- Jacobian-Aware LOD
- Elliptical History Footprint
- Anisotropic Sampling
- Cross-Mip Validation
- Compute Shader Reduction
- Wave Operations
- Local Data Share (LDS)
- Subresource Barrier
- GPU Memory Bandwidth
- Sparse Residency
- Sampler Feedback
- Level-Set Revision
- Topology Identity
- [NVIDIA Research, Spatiotemporal Variance-Guided Filtering](https://research.nvidia.com/labs/rtr/publication/schied2017spatiotemporal/)
- [NVIDIA Real-Time Denoisers (NRD)](https://github.com/NVIDIA-RTX/NRD)
- [AMD FidelityFX Single Pass Downsampler](https://gpuopen.com/manuals/fidelityfx_sdk/techniques/single-pass-downsampler/)
- [Microsoft Direct3D 12 Sampler Feedback Specification](https://microsoft.github.io/DirectX-Specs/d3d/SamplerFeedback.html)
