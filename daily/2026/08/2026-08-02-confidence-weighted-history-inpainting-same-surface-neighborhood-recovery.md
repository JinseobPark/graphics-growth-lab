---
title: "Confidence-Weighted History Inpainting and Same-Surface Neighborhood Recovery"
date: "2026-08-02"
category: Graphics
tags: ["GPU", "Temporal Filtering", "History Inpainting", "Denoising", "History Confidence", "Compute Shader", "G-Buffer", "Memory Layout", "Scientific Visualization"]
level: intermediate
---

# [Daily Graphics Growth] 2026-08-02 - Confidence-Weighted History Inpainting and Same-Surface Neighborhood Recovery

## 1. 오늘의 개념

전날의 **Hierarchical History Search**와 **Validity-Driven LOD Fallback**은 요청한 LOD의 temporal history가 유효하지 않을 때 주변 MIP level에서 대체 history를 찾는 방법이었다. 오늘은 그 다음 단계인 **Confidence-Weighted History Inpainting**과 **Same-Surface Neighborhood Recovery**를 다룬다.

Temporal reprojection이 실패한 pixel은 disocclusion, motion-vector mismatch, topology change, reflection-space distortion 때문에 history를 잃는다. History inpainting은 이 빈 영역을 단순히 주변 color로 메우는 blur가 아니다. 현재 pixel과 동일한 surface domain에 속하는 주변의 valid history만 제한적으로 빌리고, 결과에 낮은 confidence와 짧은 effective history length를 부여하는 복구 과정이다.

Same-surface 판정에는 screen-space 거리뿐 아니라 linear depth, normal, material/object ID, roughness, topology revision, virtual-surface identity가 함께 사용된다. 핵심은 값을 채우는 것보다 **어디에서 빌려왔고 얼마나 믿을 수 있는지**를 보존하는 것이다.

## 2. 한 줄 핵심

**Reprojection hole은 가장 가까운 색으로 메우는 것이 아니라, 같은 surface의 valid history를 confidence-weighted 방식으로 재구성하고 그 결과를 ‘빌린 history’로 낮게 평가해야 edge leakage 없이 temporal convergence를 이어갈 수 있다.**

## 3. 왜 중요한가

History를 즉시 reset하면 contamination은 막을 수 있지만 새로 드러난 영역의 noise, silhouette flicker, specular sparkle가 반복된다. 반대로 unconstrained dilation은 foreground history를 background로 번지게 하여 ghosting과 color bleeding을 만든다.

Same-surface recovery는 두 극단 사이의 복구 단계다.

1. 직접 reprojection history가 valid하면 그대로 사용한다.
2. invalid하면 같은 surface의 인접 valid history를 찾는다.
3. 복구 history는 confidence·history length·variance를 낮춰 사용한다.
4. 충분한 support가 없으면 current sample로 reset한다.

NVIDIA NRD의 **HistoryFix**, fast history, history confidence도 같은 방향을 따른다. NRD는 history confidence를 `[0, 1]` 범위로 받아 temporal lag를 줄이고, 짧거나 불안정한 history를 별도 복구 단계에서 처리한다. SVGF 역시 temporal moments와 variance를 이용해 spatial filter 강도를 조절한다. 현대 denoiser는 history를 binary valid/invalid가 아니라 신뢰도와 통계를 가진 signal로 취급한다.

## 4. 구현 관점

### 4.1 Surface-compatible weight

현재 pixel `p`의 direct history가 invalid일 때 neighborhood `Ω(p)`의 후보 `q`를 평가한다.

`w(p,q) = w_s · w_z · w_n · w_id · w_r · w_e · c(q)`

- `w_s`: screen-space distance
- `w_z`: depth 또는 world-position 차이
- `w_n`: normal alignment
- `w_id`: material/object/virtual-surface identity
- `w_r`: roughness와 signal class
- `w_e`: topology/revision consistency
- `c(q)`: source history confidence

Material/object ID와 revision은 soft weight보다 hard rejection에 가깝다. 다른 topology epoch나 명확히 다른 material이면 후보에서 제외한다.

복구 signal은 다음 형태로 생각할 수 있다.

`H_repair(p) = Σ w(p,q) H(q) / Σ w(p,q)`

그러나 `Σw`가 작으면 값이 계산되어도 신뢰할 수 없다. Total support, valid candidate count, dominant identity ratio, depth/normal spread를 함께 평가해야 한다.

### 4.2 복구 confidence와 history length

복구된 history는 정확한 motion trajectory에서 얻은 값이 아니므로 direct history와 동일하게 취급하면 안 된다.

`c_repair = c_support · c_surface · c_distance · c_source`

복구 confidence에는 direct history보다 낮은 상한을 둔다. History length도 source 값을 그대로 복사하지 않고 confidence에 따라 줄인다.

`n_repair = min(n_source, n_repair_max) · c_repair`

이렇게 해야 inpainted value가 오래 고착되어 lag와 ghosting을 만드는 것을 막을 수 있다.

### 4.3 Effective sample count

같은 total weight라도 여러 이웃이 고르게 지지하는 결과와 한 pixel에 의존하는 결과는 다르다.

`N_eff = (Σw)^2 / Σ(w²)`

`N_eff`가 크면 여러 compatible source가 결과를 지지한다. `N_eff ≈ 1`이면 사실상 한 source에 의존하므로 confidence를 낮게 유지해야 한다.

### 4.4 Same-surface guide 설계

- **Depth**: 단순 absolute epsilon보다 depth-relative threshold나 reconstructed world-space plane distance가 안정적이다.
- **Normal**: geometry continuity에는 geometric normal, BRDF/specular domain에는 shading normal과 roughness가 더 적합하다.
- **Identity**: material ID만으로 geometry continuity를 보장할 수 없으므로 object/instance ID, revision epoch를 함께 사용한다.
- **Reflection**: glossy specular는 가까운 pixel도 다른 reflected hit을 볼 수 있으므로 virtual position, hit distance, roughness 검증이 필요하다.
- **Volume**: surface가 없는 영역에서는 AMR block, sparse-brick ID, scalar range, opacity classification이 domain guide가 된다.

### 4.5 Neighborhood와 search policy

Fixed square kernel은 단순하지만 edge를 가로지르는 tap이 많다. Cross/diamond pattern, edge-aligned kernel, small-radius-first search, hierarchical search가 대안이다.

- diffuse GI: 비교적 넓은 same-surface support 허용
- glossy specular: 작은 radius와 강한 virtual-surface 검증
- hard shadow: blocker/light identity 고려
- hair/transparency: coverage와 strand/material identity 필요

논리적 recovery ladder는 `direct reprojection → small-radius repair → wider/hierarchical repair → fast history blend → current reset`으로 볼 수 있다.

### 4.6 Moments와 variance

Color만 복구하고 temporal moments를 그대로 두면 variance가 signal과 불일치한다. 반대로 neighboring moments를 단순 평균하면 서로 다른 mean이 섞이면서 variance가 과소평가될 수 있다.

Temporal state는 최소한 first moment `m1`, second moment `m2`, variance `max(m2 - m1², 0)`, effective history length를 함께 관리해야 한다. Borrowed history는 정확한 시간 축적 결과가 아니므로 moments를 보수적으로 재초기화해야 한다. 이 문제는 내일의 **Moment-Preserving History Reconstruction and Variance Reinitialization**으로 이어진다.

### 4.7 GPU compute shader

History repair는 neighborhood gather pass라서 arithmetic보다 memory access와 divergence가 중요하다.

- Compact depth/normal/ID/confidence guide를 workgroup shared memory에 적재
- Candidate guide test를 먼저 수행하고 통과한 경우에만 history signal fetch
- Subgroup ballot으로 repair lane mask 생성
- Tile 전체가 direct history valid이면 fast exit
- Search radius를 bounded constant로 제한해 wave divergence 억제
- Invalid pixel이 매우 적을 때는 compacted pixel list와 indirect dispatch 고려

이미 복구된 history가 다시 다른 pixel의 source가 되면 한 frame 안에서 uncontrolled dilation이 발생할 수 있다. Original valid history만 source로 허용하거나 repair generation을 제한해야 한다.

### 4.8 Memory layout

대표적인 persistent state는 다음과 같다.

- history signal: `RGBA16_FLOAT` 또는 signal별 `RG16_FLOAT`
- temporal moments: `RG16_FLOAT`
- history length + confidence: `RG8_UNORM` 또는 별도 compact texture
- validity/rejection reason: `R8_UINT`
- material/object/revision metadata: packed `R32_UINT`

1080p `RGBA16_FLOAT` texture 하나는 약 15.8 MiB다. Repair distance와 reason code는 가능한 한 transient resource로 두고 최종 confidence/history length에 결과를 압축하는 편이 유리하다.

### 4.9 C++ render graph

Repair pass는 current G-buffer, previous history signal/moments/confidence, direct validity를 읽고 repaired state를 쓴다. C++ render graph는 다음 contract를 관리해야 한다.

- camera cut와 history reset epoch
- dynamic-resolution scale과 valid rect
- signal type과 policy
- motion-vector space와 jitter convention
- object/topology revision
- transient/persistent resource aliasing
- previous SRV와 repaired UAV 사이의 barrier

D3D12/HLSL에서는 `Texture2D.Load`, `Gather`, groupshared memory, wave intrinsics를 사용한다. Vulkan/GLSL은 `texelFetch`, subgroup operations, storage image barrier가 핵심이다. WebGPU/WGSL에서는 `textureLoad`, storage texture, workgroup memory와 pass 간 usage 분리가 중요하다.

### 4.10 Failure mode와 debug view

대표 failure mode는 foreground history의 background 침투, normal map으로 인한 과도한 rejection, reflection virtual-hit 오판, repeated inpainting의 confidence 증폭, topology revision 누락, moments 불일치다.

필수 debug view:

- direct valid / repaired / reset 분류
- repair source distance와 direction
- total support와 `N_eff`
- dominant ID ratio
- source confidence / repaired confidence
- history length와 repair generation
- depth·normal·identity rejection mask

## 5. 내 관심 분야와 연결

### 실시간 렌더링과 게임 엔진

Ray-traced GI, reflection, shadow denoising에서 disocclusion hole은 피할 수 없다. Graphics engineer는 temporal accumulation만 설명하는 데서 끝나지 않고 reject 이후의 search, repair, confidence reduction, fallback까지 설계할 수 있어야 한다.

### Level-set, voxel, Marching Cubes

Level-set 기반 공정 emulation은 geometry가 매 step 재생성되므로 mesh vertex index가 temporal identity로 안정적이지 않다. Same-surface guide로 level-set sign, material field, closest-interface distance, process-step revision, voxel-to-world transform을 사용하는 편이 더 안전하다.

### CFD와 scientific visualization

Stochastic volume rendering이나 temporal sampling에서도 history hole이 발생한다. Surface 대신 AMR block, sparse-brick hierarchy, scalar range, gradient direction, transfer-function classification, simulation time-step revision을 compatibility guide로 사용할 수 있다. 이 복구는 rendering layer에만 적용되며 원본 solver data를 변경하지 않는다.

### GPU 최적화

품질은 guide 설계에서 결정되지만 성능은 tap 수, search radius, metadata packing, texture cache locality, shared-memory footprint, invalid mask sparsity, wave divergence에서 결정된다. 알고리즘과 hardware execution pattern을 함께 이해해야 한다.

## 6. 머릿속에 남길 질문 3개

1. **Direct reprojection history와 neighborhood에서 빌린 history는 confidence, history length, variance를 각각 얼마나 다르게 취급해야 하는가?**
2. **‘같은 surface’를 판정할 때 depth·normal·material ID·topology revision 중 무엇을 hard rejection으로 쓰고 무엇을 soft weight로 써야 하는가?**
3. **복구된 history의 연쇄 전파를 어떻게 제한해야 edge leakage와 과도한 dilation을 막을 수 있는가?**

## 7. graphics engineer 면접 질문 1개와 답변

**질문:** Temporal denoiser에서 reprojection이 실패한 pixel을 주변 history로 복구하려고 한다. 단순 spatial blur와 same-surface history inpainting의 차이, 필요한 GPU 상태를 설명하라.

**답변:** Spatial blur는 화면상 가까운 color를 평균하지만 same-surface inpainting은 depth, normal, material/object identity, roughness, topology revision이 호환되는 valid history만 사용한다. 후보는 거리, guide compatibility, source confidence로 가중하고 support가 충분할 때만 reconstruction을 만든다. 복구 결과는 정확한 temporal trajectory를 따른 값이 아니므로 direct history보다 낮은 confidence와 짧은 effective history length를 부여하고 temporal moments와 variance도 보수적으로 재초기화한다. GPU에서는 previous history signal, moments, confidence/history length, current G-buffer guide, validity mask가 필요하며 compact metadata, shared-memory tile, subgroup fast path, bounded search radius로 비용을 제어한다. Repaired value의 재전파도 제한해야 한다.

## 8. 포트폴리오 / 커리어 연결

포트폴리오에서는 “denoiser를 구현했다”보다 **history failure policy를 설계했다**는 점이 중요하다.

설명할 항목:

- direct reprojection validation
- same-surface compatibility 정의
- confidence와 history length semantics
- signal별 repair radius와 policy
- repeated propagation 제한
- moments/variance 재초기화
- GPU memory footprint와 bandwidth
- artifact debug view

비교 화면은 current-only reset, unconstrained dilation, same-surface repair, confidence/source-distance overlay를 함께 보여주는 것이 좋다. 면접에서는 `reject → search → repair → confidence reduction → fallback`의 state machine을 설명하면 엔진 실무 이해도를 보여줄 수 있다.

## 9. 내일 이어서 볼 개념

**Moment-Preserving History Reconstruction and Variance Reinitialization**

오늘은 invalid history signal을 same-surface neighborhood에서 복구했다. 다음은 복구된 color와 temporal statistics가 일치하도록 first/second moments, variance, effective sample count를 재구성하고 spatial filter가 과도하게 약해지거나 강해지지 않도록 variance를 reinitialize하는 방법을 살펴본다.

## 10. 참고 키워드

- Confidence-Weighted History Inpainting
- Same-Surface Neighborhood Recovery
- Temporal Reprojection / Disocclusion Handling
- HistoryFix / History Confidence / Fast History
- Surface-Aware Filtering / Cross-Bilateral Weight
- Effective Sample Count
- Temporal Moments / Variance Reinitialization
- G-Buffer Validation
- Material / Object / Topology Revision ID
- Compute Shader Neighborhood Gather
- Wave / Subgroup Operations
- NVIDIA Real-Time Denoisers (NRD): https://github.com/NVIDIA-RTX/NRD
- Spatiotemporal Variance-Guided Filtering (SVGF): https://research.nvidia.com/labs/rtr/publication/schied2017spatiotemporal/
