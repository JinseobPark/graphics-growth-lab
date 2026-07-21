---
title: "Temporal Moments and Variance-Guided History Confidence"
date: "2026-07-21"
category: "Graphics"
tags: ["GPU", "Temporal Rendering", "SVGF", "Temporal Moments", "Variance", "History Confidence", "Compute Shader", "Denoising", "Scientific Visualization"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-07-21 - Temporal Moments and Variance-Guided History Confidence

## 1. 오늘의 개념

**Temporal Moments**는 한 픽셀에 누적된 시간적 신호의 분포를 평균값 하나가 아니라 1차 모멘트와 2차 모멘트로 요약하는 방식이다.

- 1차 raw moment: `m1 = E[L]`
- 2차 raw moment: `m2 = E[L²]`
- 분산: `variance = max(m2 - m1², 0)`

여기서 `L`은 보통 luminance, radiance의 특정 성분, 또는 scientific visualization의 scalar 값이다. 전날 다룬 depth·normal·stable ID 기반 검증이 **같은 표면인가**를 판단했다면, temporal moments는 같은 표면 위의 신호가 **통계적으로 계속 같은 상태인가**를 판단하는 근거가 된다.

**Variance-Guided History Confidence**는 이전 프레임의 평균과 분산에 비해 현재 샘플이 얼마나 예상 밖인지 계산하고, 그 결과로 temporal accumulation의 history weight, history length, spatial filter radius를 조절하는 설계다.

## 2. 한 줄 핵심

**Temporal moments는 history의 평균과 불확실성을 함께 저장하고, 현재 샘플의 통계적 이탈 정도를 이용해 과거 데이터를 얼마나 믿을지 결정한다.**

## 3. 왜 중요한가

Depth, normal, object ID가 모두 일치하더라도 조명, 반사, 그림자, volume density, transfer function, simulation scalar는 급격히 바뀔 수 있다. Geometry validation만으로는 이런 **signal change**를 검출하지 못하므로 과거 radiance가 남아 temporal lag나 ghosting으로 보일 수 있다.

반대로 현재 값이 이전 평균과 다르다는 이유만으로 history를 즉시 폐기하면 Monte Carlo noise, undersampling, stochastic volume sampling까지 실제 변화로 오인한다. Temporal moments는 현재 변화가 history의 평소 변동 범위 안에 있는지 판단할 수 있게 한다.

SVGF(Spatiotemporal Variance-Guided Filtering)는 temporal accumulation으로 effective sample count를 높이고, spatiotemporal luminance variance를 계층적 image-space wavelet filter의 제어 신호로 사용한다. 중요한 점은 variance가 단순한 진단 값이 아니라 temporal과 spatial reconstruction을 연결하는 핵심 metadata라는 것이다.

다만 높은 variance에는 두 가지 의미가 섞인다.

- **Noise가 크다:** 더 많은 temporal accumulation과 넓은 spatial filtering이 필요하다.
- **실제 신호가 변했다:** history를 빠르게 줄여 responsiveness를 높여야 한다.

따라서 variance 하나만으로 history를 늘리거나 줄이지 않고, geometry confidence, innovation residual, history age, revision signal을 함께 사용해야 한다.

## 4. 구현 관점

### 4.1 어떤 신호의 moments를 저장할 것인가

가장 일반적인 선택은 pre-exposed luminance다.

`L = dot(rgb, luminanceWeights)`

RGB 각각의 moments를 저장하면 색 변화까지 포착할 수 있지만 storage와 bandwidth가 크게 증가한다. Luminance moments는 비용이 낮고 대부분의 radiance 변화에 민감하지만, 동일 luminance를 가진 chroma 변화는 놓칠 수 있다.

실시간 ray tracing denoiser에서는 diffuse와 specular의 통계적 성질이 다르므로 moments 또는 confidence를 분리하는 편이 유리하다. Specular는 roughness, hit distance, reflection motion에 의해 훨씬 빠르게 변하고 heavy-tail outlier가 많다.

Scientific visualization에서는 화면의 최종 RGB보다 원본 scalar 또는 opacity-weighted signal의 moments를 관리하는 방식이 더 의미 있을 수 있다. Colormap만 바뀌었을 때 물리 데이터 변화와 표현 변화가 섞이지 않기 때문이다.

### 4.2 Temporal moment update

이전 프레임에서 재투영한 moments를 `m1Prev`, `m2Prev`, 현재 샘플을 `L`이라고 하자.

History가 완전히 유효할 때 누적 샘플 수 기반 weight는 다음처럼 생각할 수 있다.

`wBase = nPrev / (nPrev + 1)`

실제 pipeline에서는 최대 history length를 제한하고 전날 계산한 geometry confidence와 오늘 계산할 signal confidence를 결합한다.

`wHistory = wBase * Cgeometry * Csignal`

`m1New = wHistory * m1Prev + (1 - wHistory) * L`

`m2New = wHistory * m2Prev + (1 - wHistory) * L²`

`varianceNew = max(m2New - m1New², varianceFloor)`

`Cgeometry = 0`이면 radiance뿐 아니라 moments, sample count, lock status를 함께 초기화해야 한다. Color만 reset하고 높은 history length와 낮은 variance를 유지하면 새 표면을 이미 수렴한 신호로 오판한다.

고정 EMA(Exponential Moving Average)를 사용할 수도 있지만, 이 경우 moments가 표현하는 effective sample count가 실제 frame count와 다르다. Debugging과 adaptive filter 설계를 위해서는 history length 또는 effective sample count를 별도 metadata로 유지하는 편이 명확하다.

### 4.3 Innovation residual로 signal confidence 만들기

현재 샘플이 이전 history에서 얼마나 예상 밖인지 판단하는 값이 **innovation residual**이다.

`residual = abs(L - m1Prev)`

이를 history의 표준편차로 정규화하면 장면 밝기와 noise scale에 덜 민감한 값이 된다.

`z = residual / sqrt(variancePrev + varianceCurrentEstimate + epsilon)`

Signal confidence는 다음과 같은 연속 함수로 만들 수 있다.

`Csignal = exp(-0.5 * z²)`

또는 threshold를 명확히 제어하려면 다음 형태가 가능하다.

`Csignal = 1 - smoothstep(zAccept, zReject, z)`

중요한 구현 규칙은 **현재 샘플을 반영하기 전의 moments로 현재 샘플을 검증하는 것**이다. 먼저 moments를 갱신한 뒤 residual을 계산하면 outlier가 평균과 분산을 스스로 끌어올려 mismatch가 약하게 보인다.

또한 variance가 매우 크면 denominator가 커져 실제 조명 변화도 정상 범위처럼 보일 수 있다. 이를 막기 위해 다음 신호를 함께 사용한다.

- normalized residual과 absolute residual을 동시에 검사
- validation에 사용하는 variance의 상한 설정
- light/material/transfer-function revision을 hard gate로 적용
- reactive mask나 application-provided history confidence와 결합
- diffuse와 specular를 분리해 서로 다른 threshold 사용

NVIDIA NRD의 공개 설계도 application-provided history confidence를 history length 제한에 반영해 dynamic lighting의 temporal lag를 낮추는 구조를 설명한다.

### 4.4 Variance와 spatial filter 연결

Variance는 spatial filter의 반경과 강도를 조절하는 입력으로 사용할 수 있다.

- 낮은 variance: history가 안정적이므로 작은 filter radius로 detail 보존
- 높은 variance: noise 가능성이 높으므로 더 넓은 neighborhood 사용
- 낮은 history length: temporal 통계가 약하므로 spatially 보완된 variance 필요

그러나 높은 variance가 실제 edge나 빠른 lighting change에서 발생할 수도 있으므로 depth, normal, albedo, roughness, material ID 같은 **edge-stopping signal**이 필요하다. Variance는 filter를 얼마나 강하게 할지를 정하고, G-buffer guidance는 어디를 넘어가면 안 되는지를 정한다.

### 4.5 GPU memory layout

실용적인 resource 구성 예시는 다음과 같다.

- Accumulated radiance: `RGBA16_FLOAT`
- Luminance moments `(m1, m2)`: `RG16_FLOAT` 또는 `RG32_FLOAT`
- History length: `R8_UNORM`, `R16_UINT`, 또는 다른 metadata와 packed format
- History confidence: `R8_UNORM`
- Variance: shader에서 moments로 즉시 계산하거나 공유가 필요하면 `R16_FLOAT`

`m2`는 `L²`을 저장하므로 FP16 overflow와 precision loss에 특히 취약하다. Half float의 최대 유한값은 약 65504이므로 exposure-relative luminance가 약 256을 넘으면 제곱값이 표현 범위를 넘어갈 수 있다. HDR pipeline에서는 다음 중 하나가 필요하다.

- pre-exposure를 적용한 luminance 사용
- moments에 scale factor 적용
- `m2`를 FP32로 저장
- log-domain 통계 사용
- firefly를 moments update 전에 robust clamp

Diffuse와 specular moments를 모두 저장하면 두 개의 `RG16_FLOAT` 대신 하나의 `RGBA16_FLOAT`로 묶을 수 있다. 다만 pass별 접근 패턴이 다르면 AoS packing이 불필요한 channel fetch를 만들 수 있으므로 SoA와 bandwidth를 비교해야 한다.

Temporal resources는 일반적으로 ping-pong한다.

`Previous Moments/History → Temporal Accumulation → Current Moments/History`

Variance를 한 pass에서만 사용한다면 별도 texture로 기록하지 않고 register에서 계산해 intermediate bandwidth를 줄일 수 있다. 여러 spatial iteration이나 debug pass가 공유한다면 저장 비용과 재계산 비용을 비교한다.

### 4.6 Compute pass와 동기화

일반적인 GPU pass 흐름은 다음과 같다.

`Reprojection → Geometry Validation → Signal Confidence → Moment/History Update → Variance Estimation → Spatial Filtering`

Temporal pass는 current noisy signal, previous accumulated signal, previous moments, motion vector, depth, normal, identity metadata를 읽고 current history를 쓴다. 이 단계는 texture fetch와 history write가 많아 ALU보다 memory bandwidth와 cache locality가 병목이 되기 쉽다.

Pass fusion은 moments와 variance의 intermediate write를 줄이지만 shader register pressure와 디버깅 복잡도를 높인다. Separate pass는 각 confidence channel을 관찰하기 쉽지만 full-resolution read/write가 추가된다.

À-trous wavelet 단계는 iteration마다 넓어지는 neighborhood를 읽기 때문에 memory access 최적화의 영향이 크다. 2026년 JCGT 연구는 base SVGF의 wavelet 단계에 현대 GPU의 memory access 특성을 반영해 테스트 조건에서 1.3~2.5배 성능 향상을 보고했다. 알고리즘 수식이 같아도 tile reuse, access pattern, kernel organization이 최종 frame time을 크게 좌우한다는 의미다.

### 4.7 반드시 분리해서 볼 debug view

- Previous first moment `m1Prev`
- Standard deviation `sqrt(variancePrev)`
- Absolute residual
- Normalized residual 또는 z-score
- Geometry confidence
- Signal confidence
- Final history weight
- History length
- Variance-guided spatial radius
- Reset/revision reason

Ghosting과 temporal lag는 false acceptance 또는 지나치게 큰 history weight를 의심한다. Flicker와 noise 증가는 false rejection, variance floor 부족, history length reset 과다를 의심한다.

## 5. 내 관심 분야와 연결

### CFD / Scientific Visualization

CFD의 pressure, velocity magnitude, temperature는 시간에 따라 실제로 변하는 physical signal이다. Screen-space temporal accumulation을 사용하면 stochastic rendering noise를 줄일 수 있지만, simulation timestep의 실제 변화를 noise로 평활화해서는 안 된다.

따라서 `geometryRevision`, `simulationTimestep`, `fieldRevision`, `visualizationRevision`을 분리할 필요가 있다. Geometry는 그대로인데 scalar field만 갱신되었다면 geometry history는 유지하고 scalar/radiance moments의 history length만 줄일 수 있다.

### Volume Rendering

Stochastic ray marching, sparse volume sampling, low-spp volume path tracing에서는 temporal moments가 효과적이다. 하지만 transfer function 변경은 density가 같아도 최종 radiance와 opacity의 의미를 바꾸므로 hard reset 대상이다.

Volume에서는 luminance moments 외에 transmittance 또는 accumulated opacity의 moments를 보조 신호로 사용할 수 있다. 밝기는 비슷하지만 ray termination 구조가 달라진 경우를 검출하는 데 도움이 된다.

### SDF / Level-Set / Marching Cubes

Topology가 변하는 generated surface에서는 stable cell/brick identity와 generation counter로 geometry를 먼저 검증해야 한다. 그 다음 moments로 같은 cell에서 생성된 shading signal의 안정성을 판단한다.

Geometry mismatch를 variance 증가로만 처리하면 다른 표면의 history가 잠시 섞인 뒤 분산이 커지는 순서가 된다. Identity validation은 history contamination 이전의 hard gate이고, temporal moments는 동일 identity 내부의 signal change를 다루는 soft gate다.

### Game Engine / Ray Tracing

Diffuse GI, glossy reflection, shadow, AO는 noise와 temporal behavior가 서로 다르다. 하나의 공통 moments buffer보다 signal별 history policy와 threshold를 두는 설계가 품질 조정에 유리하다.

엔진 관점에서는 renderer가 moments를 계산하더라도 light update, material animation, particle revision, camera cut 같은 event를 history system에 전달해야 한다. 좋은 temporal pipeline은 shader 하나가 아니라 renderer state와 GPU metadata가 연결된 구조다.

## 6. 머릿속에 남길 질문 3개

1. 높은 variance가 Monte Carlo noise인지 실제 lighting 또는 simulation 변화인지 어떤 추가 신호로 구분할 수 있는가?
2. HDR radiance의 second moment를 FP16에 저장할 때 어떤 overflow와 precision 문제가 발생하며, pre-exposure가 왜 중요한가?
3. Geometry confidence와 signal confidence를 하나의 값으로 바로 곱하기보다 별도의 debug channel과 policy로 유지해야 하는 이유는 무엇인가?

## 7. Graphics Engineer 면접 질문 1개와 답변

**질문: Temporal denoiser에서 first/second moments를 어떻게 갱신하고, 이를 history confidence와 spatial filtering에 어떻게 활용하겠습니까?**

**답변:** 재투영된 이전 luminance의 first moment와 second raw moment를 유지하고 `variance = max(m2 - m1², floor)`로 분산을 계산합니다. 현재 샘플을 반영하기 전에 이전 평균과 분산으로 normalized innovation residual을 구해 signal confidence를 만들고, depth·normal·stable ID 기반 geometry confidence와 결합해 history weight와 history length를 조절합니다. History가 무효이면 accumulated color뿐 아니라 moments와 sample count도 함께 reset합니다. 높은 variance는 spatial filter를 강화하는 입력으로 사용하되 depth, normal, material 같은 edge-stopping signal로 실제 detail을 보호합니다. GPU 구현에서는 moments의 ping-pong lifetime, full-resolution bandwidth, pass fusion, diffuse/specular 분리, FP16에 `L²`을 저장할 때 발생하는 HDR overflow를 고려합니다.

## 8. 포트폴리오 / 커리어 연결

포트폴리오에서는 단순히 temporal average를 적용했다고 설명하기보다 다음 구조를 명확히 보여주는 편이 강하다.

`Surface Validation → Temporal Moments → Innovation Test → History Confidence → Adaptive Accumulation → Variance-Guided Spatial Filter`

특히 다음 비교는 graphics engineer로서의 설계 역량을 보여준다.

- 평균만 저장한 accumulation과 first/second moments 기반 accumulation
- Geometry-only rejection과 geometry+signal confidence
- FP16 moments와 pre-exposed/FP32 moments
- Luminance-only와 diffuse/specular 분리
- Separate pass와 fused pass의 bandwidth 및 register trade-off
- Static scene noise와 dynamic lighting responsiveness

Debug view로 variance, z-score, history length, final confidence, filter radius를 시각화하고 GPU profiler에서 temporal pass와 spatial iteration별 비용을 제시하면 알고리즘 이해와 engine integration 능력을 동시에 드러낼 수 있다.

이 개념은 TAA, temporal upscaling, real-time ray tracing denoising, stochastic transparency, volume rendering, CFD field visualization을 하나의 **statistical temporal reconstruction** 관점으로 연결한다.

## 9. 내일 이어서 볼 개념

**À-Trous Wavelet Filtering and Edge-Stopping Functions**

오늘 계산한 variance가 spatial filtering의 scale을 어떻게 제어하는지, hole을 두고 커지는 à-trous kernel이 multi-scale noise를 처리하는 방식, 그리고 depth·normal·luminance·roughness 기반 edge-stopping weight가 detail과 noise를 어떻게 분리하는지 살펴본다.

## 10. 참고 키워드

Temporal Moments, First Raw Moment, Second Raw Moment, Luminance Variance, Innovation Residual, Z-Score, History Confidence, Effective Sample Count, Exponential Moving Average, Temporal Accumulation, SVGF, NRD History Confidence, Pre-Exposure, FP16 Overflow, Firefly, Variance Floor, Diffuse/Specular Separation, Ping-Pong History, Pass Fusion, Memory Bandwidth, À-Trous Wavelet Filter, Edge-Stopping Function, Scientific Visualization, Volume Denoising

참고 자료:

- Christoph Schied et al., *Spatiotemporal Variance-Guided Filtering: Real-Time Reconstruction for Path-Traced Global Illumination*, HPG 2017, DOI: 10.1145/3105762.3105770
- NVIDIA RTX, *NVIDIA Real-Time Denoisers (NRD)*, History Confidence documentation
- Rostyslav Pikulsky, *Optimizing Spatiotemporal Variance-Guided Filtering for Modern GPU Architectures*, Journal of Computer Graphics Techniques, Vol. 15, No. 1, 2026
