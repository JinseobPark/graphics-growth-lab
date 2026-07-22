---
title: "À-Trous Wavelet Filtering and Edge-Stopping Functions"
date: "2026-07-22"
category: "Graphics"
tags: ["GPU", "SVGF", "À-Trous Wavelet", "Edge-Stopping", "Denoising", "Compute Shader", "Memory Bandwidth", "Scientific Visualization"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-07-22 - À-Trous Wavelet Filtering and Edge-Stopping Functions

## 1. 오늘의 개념

**À-Trous Wavelet Filtering**은 커널의 샘플 개수는 유지하면서 반복 단계마다 샘플 간격을 넓혀, 다운샘플링 없이 넓은 공간적 범위의 노이즈를 제거하는 계층적 필터다. `à trous`는 프랑스어로 “구멍이 있는(with holes)”이라는 뜻이며, 반복마다 커널 탭 사이에 빈 간격을 삽입하는 형태에서 이름이 왔다.

실시간 렌더링에서는 보통 다음과 같은 1차원 B-spline 계열 커널을 2차원으로 확장해 사용한다.

`h = [1, 4, 6, 4, 1] / 16`

반복 단계 `k`의 샘플 간격을 다음처럼 둔다.

`step_k = 2^k`

따라서 5×5 커널을 사용하더라도 실제 샘플 수는 단계마다 25개로 유지되고, 샘플 위치만 `1, 2, 4, 8, 16 ...` 픽셀 간격으로 멀어진다. 여러 반복을 통과한 결과는 큰 blur kernel을 한 번 적용한 것과 비슷한 넓은 support를 가지지만, 각 단계의 비용은 고정된 탭 수로 제한된다.

그러나 일반적인 Gaussian blur처럼 모든 이웃을 섞으면 depth discontinuity, surface normal 변화, material boundary, illumination edge까지 흐려진다. 이를 막는 장치가 **Edge-Stopping Function**이다. 중심 픽셀과 이웃 픽셀이 같은 표면과 같은 신호 분포에 속할 가능성이 낮으면 필터 가중치를 빠르게 줄인다.

전체 필터는 다음 형태로 생각할 수 있다.

`C_out(p) = Σ_q h_k(p, q) · W(p, q) · C(q) / Σ_q h_k(p, q) · W(p, q)`

여기서 `h_k`는 dilation이 적용된 spatial kernel이고, `W`는 depth·normal·luminance·identity 등의 edge-stopping weight를 곱한 값이다.

## 2. 한 줄 핵심

**À-trous filter는 적은 고정 탭으로 넓은 영역을 계층적으로 탐색하고, edge-stopping function은 그 넓은 필터가 서로 다른 표면과 신호를 섞지 못하도록 막는다.**

## 3. 왜 중요한가

Temporal accumulation은 여러 프레임의 샘플을 결합해 effective sample count를 높이지만, disocclusion·빠른 motion·history reset 직후에는 사용할 수 있는 과거 데이터가 부족하다. 이때 남은 stochastic noise를 공간적으로 줄이는 단계가 필요하다.

작은 Gaussian이나 bilateral filter만 반복하면 넓은 low-frequency noise를 제거하기 위해 많은 iteration이 필요하다. 반대로 큰 커널을 직접 사용하면 탭 수와 bandwidth가 급격히 증가한다. À-trous 방식은 커널 탭 수를 유지하면서 단계별 support를 확장해 이 문제를 완화한다.

SVGF(Spatiotemporal Variance-Guided Filtering)는 temporal moments와 luminance variance를 이용해 필터 강도를 제어하고, à-trous wavelet hierarchy를 통해 여러 공간 규모에서 noise와 detail을 분리한다. 전날 다룬 variance가 **얼마나 강하게 필터링할 것인가**를 결정한다면, 오늘의 edge-stopping function은 **어디까지 섞어도 되는가**를 결정한다.

이 구분이 중요하다.

- Variance가 높다: 더 넓은 spatial support가 필요할 수 있다.
- Depth 또는 normal이 끊긴다: variance가 높더라도 경계를 넘어가면 안 된다.
- Luminance가 다르다: noise인지 실제 illumination edge인지 variance와 history length를 함께 봐야 한다.
- Stable identity가 다르다: depth가 우연히 비슷해도 다른 객체라면 hard rejection이 안전하다.

좋은 spatial denoiser는 blur의 크기보다 **신뢰할 수 있는 이웃을 정의하는 방법**이 더 중요하다.

## 4. 구현 관점

### 4.1 À-trous iteration 구조

일반적인 단계는 다음과 같다.

`Temporal Accumulation → Variance Estimation → À-Trous Iteration 0 → 1 → 2 → ... → Final Composition`

각 iteration은 이전 결과를 읽어 새 결과를 기록하므로 두 개의 texture를 ping-pong한다.

- Iteration 0: `step = 1`
- Iteration 1: `step = 2`
- Iteration 2: `step = 4`
- Iteration 3: `step = 8`
- Iteration 4: `step = 16`

각 pass는 중심 픽셀 주변의 5×5 위치를 읽지만, 실제 offset은 `int2(xx, yy) * step`이 된다. 단일 pass의 최대 footprint는 단계가 커질수록 넓어지며, 여러 단계의 누적 support는 100픽셀 이상의 저주파 noise에도 영향을 줄 수 있다.

중심 픽셀은 weight 1로 먼저 누적하는 편이 안전하다. 모든 이웃의 edge-stopping weight가 0에 가까워져 분모가 사라지는 경우를 막고, 강한 경계에서 중심 신호가 소실되는 것을 방지한다.

### 4.2 Edge-stopping weight 구성

실무에서는 하나의 weight가 아니라 여러 guidance weight를 곱한다.

`W(p, q) = W_depth · W_normal · W_luminance · W_identity · W_material`

#### Depth weight

`W_depth = exp(-|z_p - z_q| / (phi_z + epsilon))`

고정 `phi_z`는 카메라 근처와 먼 곳에서 다른 화면 공간 오차를 만든다. 따라서 linear depth, depth gradient, pixel footprint, projection scale, 현재 iteration의 `step`을 이용해 허용 범위를 조절한다.

큰 step에서는 같은 평면 위에서도 depth 차이가 커질 수 있으므로 `phi_z` 역시 step과 local depth slope에 맞춰 증가해야 한다. 그렇지 않으면 후반 iteration이 모든 이웃을 거부해 넓은 support를 전혀 사용하지 못한다.

#### Normal weight

`W_normal = max(0, dot(n_p, n_q))^phi_n`

`phi_n`이 크면 작은 normal 차이에도 weight가 빠르게 감소해 sharp edge는 잘 보존하지만, normal map noise와 curved surface를 과도하게 분리할 수 있다. Geometric normal과 shading normal 중 무엇을 guidance로 사용할지도 중요하다.

- Geometric normal: 표면 identity와 silhouette 보존에 안정적
- Shading normal: bump·normal map detail 보존에 유리하지만 high-frequency 변화에 민감
- Hybrid: 큰 구조는 geometric normal, 세부 반응은 shading normal 또는 roughness로 보완

#### Luminance weight

`W_luminance = exp(-|L_p - L_q| / (phi_l · sqrt(variance_p) + epsilon))`

Luminance 차이를 variance로 정규화하면 noisy region에서는 더 넓게 평균화하고, 이미 안정적인 region에서는 실제 detail을 보존할 수 있다. 다만 variance가 지나치게 크면 실제 shadow boundary나 lighting change까지 noise로 간주할 수 있으므로 variance upper bound, history length, reactive signal을 함께 사용한다.

Diffuse와 specular는 통계적 성질이 다르다. Specular 신호는 roughness와 view direction에 민감하고 heavy-tail outlier가 많기 때문에 동일한 `phi_l`을 공유하면 diffuse는 덜 필터링되고 specular는 과도하게 번질 수 있다.

#### Identity와 material gate

`W_identity = objectId_p == objectId_q ? 1 : 0`

`W_material = materialClass_p == materialClass_q ? 1 : softWeight`

Depth와 normal은 연속적인 confidence를 만들고, stable object ID나 region ID는 명확한 불연속을 hard gate로 처리한다. 단, 매 프레임 변하는 draw index나 transient mesh ID를 사용하면 filter가 불필요하게 끊기므로 temporal stability가 보장된 identity가 필요하다.

### 4.3 Kernel weight와 guidance weight의 역할 분리

Spatial kernel `h`는 거리에 따른 기본 중요도를 정의한다. Edge-stopping weight는 scene feature 차이에 따른 신뢰도를 정의한다.

`w_total = h(offset) · W_depth · W_normal · W_luminance`

두 역할을 분리해서 생각하면 튜닝이 명확해진다.

- Kernel 변경: blur profile과 frequency response 변화
- Depth/normal threshold 변경: geometry leakage 변화
- Luminance threshold 변경: noise 제거와 illumination detail 보존의 균형 변화
- Iteration 수 변경: 최대 spatial scale 변화

모든 문제를 `phiColor` 하나로 해결하려 하면 원인을 구분하기 어렵다. Debug view에서는 각 weight와 최종 weight를 분리해 관찰해야 한다.

### 4.4 Variance propagation

입력 픽셀의 noise가 서로 독립적이라고 가정하면 weighted average의 variance는 weight의 제곱으로 전파된다.

`variance_out = Σ_i (w_i² · variance_i) / (Σ_i w_i)²`

따라서 radiance는 `w_i`로 누적하지만 variance channel은 `w_i²`로 누적한다. NVIDIA Falcor의 공개 SVGF shader도 illumination RGB에는 weight를 한 번, alpha에 저장된 variance에는 weight를 제곱해 적용하고 분모 역시 제곱해 정규화한다.

여러 spatial pass를 거치면 샘플 간 상관관계가 생기므로 이 식은 완전한 통계 모델은 아니다. 그래도 filter strength와 후속 iteration을 제어하는 실용적인 variance estimate로 충분한 가치가 있다.

### 4.5 GPU resource와 memory layout

실용적인 full-resolution resource 예시는 다음과 같다.

- Denoising signal + variance: `RGBA16_FLOAT`
  - RGB: illumination 또는 demodulated radiance
  - A: variance
- Linear depth: `R32_FLOAT` 또는 precision 요구에 따른 `R16_FLOAT`
- Octahedral normal: `RG16_SNORM` 또는 packed representation
- History length/confidence: `R8_UNORM`, `R16_UINT`
- Stable identity: `R16_UINT` 또는 `R32_UINT`
- À-trous output: 동일 format의 ping-pong texture 2개

`RGBA16_FLOAT`는 픽셀당 8바이트다.

- 1920×1080 한 장: 약 15.8 MiB
- 1920×1080 ping-pong 두 장: 약 31.6 MiB
- 3840×2160 한 장: 약 63.3 MiB
- 3840×2160 ping-pong 두 장: 약 126.6 MiB

여기에 depth, normal, history metadata를 반복해서 읽으므로 실제 병목은 ALU보다 memory bandwidth와 cache locality가 되기 쉽다.

AoS 형태로 radiance와 variance를 `RGBA16F`에 함께 두면 한 번의 fetch로 필요한 데이터를 얻을 수 있다. 반대로 variance를 일부 iteration에서만 사용하거나 RGB와 variance의 precision 요구가 다르면 SoA로 분리하는 편이 bandwidth와 format 선택에 유리할 수 있다.

### 4.6 Compute shader와 cache locality

5×5 full kernel을 5회 수행하면 출력 픽셀 하나당 최대 125개의 signal sample과 다수의 guidance sample을 읽는다. 초기 iteration은 인접 access라 texture cache hit가 높지만, `step = 8, 16`에서는 warp 또는 wave의 인접 thread가 서로 멀리 떨어진 주소를 요청해 locality가 악화된다.

Shared memory tile은 작은 step에서 효과적이다. 그러나 `T×T` output tile에 필요한 halo가 대략 `(T + 4·step)²`로 증가하므로 큰 step에서는 적재량과 shared memory 사용량이 급격히 커진다.

따라서 실무 최적화는 다음 관점을 함께 본다.

- 작은 step만 workgroup/shared memory에 staging
- 큰 step에서는 texture cache와 invocation ordering 최적화
- signal과 guidance의 fetch 수 축소 및 packing
- bounds check의 divergence 최소화
- iteration별 specialization constant 또는 shader variant 사용
- variance를 별도 texture로 기록할지 register에서 이어갈지 비교
- debug 가능성과 bandwidth 절감을 고려한 pass fusion 범위 결정

2024년의 *A Fast GPU Schedule for À-Trous Wavelet-Based Denoisers*는 global invocation index를 재배열해 dilation된 access를 메모리 관점에서 다시 local하게 만드는 scheduling을 제안했다. 해당 연구는 일반적인 wavelet 구성에서 1.3–3.8배의 speedup을 보고했다.

2026년 JCGT의 SVGF 최적화 연구도 현대 GPU의 memory access 특성에 맞춰 à-trous 단계를 재구성하고, base SVGF 대비 테스트 조건에서 1.3–2.5배의 성능 향상을 보고했다. 핵심은 필터 수식보다 **thread가 어떤 순서로 어떤 주소를 읽는가**가 frame time을 좌우할 수 있다는 점이다.

### 4.7 Graphics API와 render graph 관점

Vulkan, DirectX 12, Metal, WebGPU에서는 각 iteration이 이전 texture를 read하고 다음 texture를 write하도록 ping-pong resource를 구성하는 것이 명확하다. 같은 subresource를 동시에 sample과 storage write 대상으로 사용하지 않으면 hazard 관리와 portability가 쉬워진다.

C++ render graph에서는 다음 정보를 명시해야 한다.

- 각 pass의 input/output texture
- iteration index와 `stepSize`
- read-after-write dependency
- UAV/storage texture barrier 또는 resource transition
- 최종 iteration의 logical output alias
- dynamic resolution 변경 시 resource recreation과 history reset

여러 iteration은 데이터 의존성이 있어 순차 실행된다. 따라서 denoiser 내부의 pass 간 병렬성보다, 각 pass의 bandwidth를 줄이고 cache behavior를 개선하는 편이 더 직접적인 최적화가 된다.

### 4.8 실패 패턴을 weight 관점에서 읽기

- 경계 너머로 색이 번진다: depth/normal/identity weight가 너무 느슨함
- 평평한 영역의 noise가 남는다: luminance weight가 너무 강하거나 iteration support가 부족함
- 곡면이 조각난 것처럼 보인다: normal exponent가 너무 크거나 shading normal이 불안정함
- 후반 iteration이 효과가 없다: step 증가에 비해 depth threshold가 확장되지 않음
- shadow boundary가 두꺼워진다: variance가 실제 신호 변화를 noise로 해석하고 luminance gate가 느슨함
- disocclusion 부근이 얼룩진다: 낮은 history length를 고려한 variance 보완과 spatial fallback이 부족함
- specular가 고무처럼 번진다: diffuse/specular signal과 filter parameter가 분리되지 않음

필터 결과 하나만 보는 것보다 `W_depth`, `W_normal`, `W_luminance`, final weight sum, iteration별 output을 각각 보는 것이 원인 분석에 효과적이다.

## 5. 내 관심 분야와 연결

### Real-Time Rendering / Game Engine

Ray-traced GI, reflection, shadow denoising에서 à-trous filter는 temporal accumulation이 놓친 residual noise를 처리하는 전형적인 spatial stage다. 엔진 수준에서는 diffuse/specular 분리, material roughness, hit distance, motion vector quality와 함께 설계해야 한다.

Rendering pipeline 관점에서는 denoiser를 독립 post-process로 보기보다 G-buffer의 precision, stable identity, exposure, dynamic resolution, camera cut 정책에 의존하는 subsystem으로 보는 것이 맞다.

### CFD / Scientific Visualization

CFD scalar field를 stochastic volume rendering하거나 sparse sampling할 때도 variance-guided spatial filtering을 적용할 수 있다. 그러나 surface rendering의 depth와 normal만으로는 반투명 volume의 signal identity를 설명하기 어렵다.

Volume에서는 다음 guidance가 더 적합할 수 있다.

- opacity-weighted expected depth
- accumulated transmittance
- scalar value 또는 scalar gradient
- transfer-function region ID
- dominant material 또는 phase ID
- simulation timestep/revision

두 ray가 같은 depth에서 끝났더라도 서로 다른 density profile을 통과했다면 같은 이웃으로 취급하면 안 된다.

### Semiconductor 3D Visualization

공정 구조의 단면 heatmap이나 material interface를 필터링할 때는 geometry normal과 material ID가 강한 edge-stopping signal이 된다. Doping concentration처럼 큰 dynamic range를 가진 scalar는 linear difference보다 log-domain difference 또는 normalized concentration difference가 더 안정적인 guidance가 될 수 있다.

Marching Cubes mesh가 다시 생성될 때 triangle ID는 바뀔 수 있으므로 mesh primitive ID보다 source voxel region, material label, process revision 같은 stable identity를 사용하는 편이 안전하다.

### Sparse Voxel / Octree / NanoVDB

À-trous filtering 자체는 screen-space에서 수행되므로 underlying sparse structure와 직접 결합되지 않아도 된다. 다만 LOD 전환, brick streaming, topology revision으로 depth와 normal이 흔들리면 edge-stopping function이 잘못된 경계를 생성한다.

따라서 sparse volume renderer에서는 LOD transition mask, brick generation ID, stable world-space gradient를 temporal/spatial confidence에 연결할 필요가 있다.

### WebGPU / Vulkan / CUDA

WebGPU compute에서는 storage texture ping-pong과 workgroup memory를 이용해 초기 iteration의 neighborhood를 재사용할 수 있다. Vulkan과 DirectX 12에서는 iteration별 descriptor 변경을 줄이기 위해 descriptor array 또는 ping-pong index를 사용할 수 있다.

CUDA에서는 texture object의 2D locality와 shared memory tile을 비교할 수 있지만, 큰 dilation에서는 shared memory halo가 급격히 커진다는 동일한 문제가 존재한다. API보다 중요한 것은 dilation에 따라 변하는 memory access pattern이다.

## 6. 머릿속에 남길 질문 3개

1. Variance가 높을 때 spatial support는 넓혀야 하지만 luminance edge는 보존해야 한다면, filter radius와 luminance threshold를 어떤 신호로 분리해 제어해야 하는가?
2. `step = 16`인 후반 à-trous pass에서 shared memory tile이 비효율적이 되는 이유를 halo 크기와 cache locality 관점에서 설명할 수 있는가?
3. 반투명 volume rendering에서는 surface depth와 normal을 대신하거나 보완할 edge-stopping guidance로 무엇을 사용해야 하는가?

## 7. graphics engineer 면접 질문 1개와 답변

**질문: À-trous wavelet filter가 큰 Gaussian filter보다 실시간 denoising에 적합한 이유와, edge-stopping function이 필요한 이유를 GPU 관점에서 설명해보세요.**

**답변:**

À-trous filter는 작은 고정 커널의 탭 수를 유지하면서 iteration마다 sample spacing을 두 배씩 넓혀 큰 spatial support를 만든다. 따라서 큰 2D kernel을 직접 적용하는 것보다 연산량을 통제하면서 low-frequency noise까지 제거할 수 있고, full resolution을 유지하므로 downsample/upsample 과정에서 생기는 detail loss와 별도 pyramid resource를 줄일 수 있다.

하지만 spacing이 커질수록 서로 다른 표면의 픽셀을 쉽게 참조하므로 spatial kernel만 사용하면 depth edge, normal discontinuity, material boundary와 lighting detail이 섞인다. 이를 막기 위해 depth·normal·luminance·stable identity 기반 edge-stopping weight를 곱해 같은 표면과 같은 신호 분포로 판단되는 이웃만 누적한다.

GPU에서는 각 pass가 많은 texture fetch를 수행하며 후반 iteration의 dilation이 cache locality를 떨어뜨리므로 memory bandwidth가 주요 병목이 된다. 초기 pass는 shared memory tile이 효과적일 수 있지만, 큰 step에서는 halo가 커지므로 invocation scheduling, cache-aware access, resource packing, ping-pong bandwidth를 함께 최적화해야 한다.

## 8. 포트폴리오 / 커리어 연결

포트폴리오에서 denoising 결과 이미지 하나만 보여주는 것보다 다음 구조를 설명할 수 있으면 graphics engineer로서의 깊이가 드러난다.

- temporal accumulation과 spatial filtering의 책임 분리
- variance가 filter strength를 제어하는 방식
- depth·normal·luminance·identity weight의 역할
- diffuse/specular 또는 surface/volume signal별 parameter 분리
- `RGBA16F` ping-pong과 guidance buffer의 memory budget
- iteration별 GPU timing과 bandwidth 분석
- step 증가에 따른 cache miss와 shared memory halo 문제
- weight별 debug visualization을 이용한 artifact diagnosis

특히 RenderDoc, Nsight Graphics, PIX에서 iteration별 texture read, cache behavior, pass duration을 비교하고 “왜 후반 pass가 같은 탭 수인데 더 느린가”를 설명할 수 있다면 단순 shader 구현을 넘어 GPU architecture를 이해하는 엔지니어로 평가받을 수 있다.

게임 엔진·실시간 렌더링 직무에서는 SVGF 자체를 외우는 것보다, temporal validation에서 만들어진 confidence와 moments가 spatial filter의 radius·threshold·fallback으로 어떻게 이어지는지를 end-to-end pipeline으로 설명하는 능력이 중요하다.

## 9. 내일 이어서 볼 개념

**Albedo Demodulation and Signal-Space Denoising**

조명 신호를 surface albedo와 분리한 뒤 denoising하고 마지막에 다시 modulation하는 이유를 살펴본다. Texture detail이 edge-stopping luminance를 오염시키는 문제, diffuse/specular signal 분리, HDR precision, material boundary 처리까지 연결한다.

## 10. 참고 키워드

- À-Trous Wavelet Transform
- Edge-Avoiding Wavelet Filter
- SVGF
- Edge-Stopping Function
- Bilateral Filter
- Cross-Bilateral Filter
- B-Spline Kernel
- Dilated Kernel
- Luminance Variance
- Depth Discontinuity
- Normal Similarity
- Stable Surface Identity
- Ping-Pong Texture
- Memory Bandwidth
- Cache Locality
- Workgroup Shared Memory
- Invocation Permutation
- Diffuse / Specular Denoising
- Volume Denoising Guidance

### 참고 자료

- [NVIDIA Research — Spatiotemporal Variance-Guided Filtering](https://research.nvidia.com/labs/rtr/publication/schied2017spatiotemporal/)
- [NVIDIA Falcor — SVGFAtrous.ps.slang](https://github.com/NVIDIAGameWorks/Falcor/blob/master/Source/RenderPasses/SVGFPass/SVGFAtrous.ps.slang)
- [NVIDIA Falcor — SVGFCommon.slang](https://github.com/NVIDIAGameWorks/Falcor/blob/master/Source/RenderPasses/SVGFPass/SVGFCommon.slang)
- [JCGT 2026 — Optimizing Spatiotemporal Variance-Guided Filtering for Modern GPU Architectures](https://jcgt.org/published/0015/01/02/)
- [A Fast GPU Schedule for À-Trous Wavelet-Based Denoisers](https://reinerdolp.com/paper/fast-svgf/)
