---
title: "Reservoir Weight Stabilization and Boiling Suppression in ReSTIR"
date: "2026-08-09"
category: Graphics
tags: ["GPU", "ReSTIR", "Reservoir Sampling", "Ray Tracing", "Temporal Reuse", "Spatial Reuse", "Boiling", "Monte Carlo", "Compute Shader", "Memory Layout", "Real-Time Rendering"]
level: intermediate
---

# [Daily Graphics Growth] 2026-08-09 - Reservoir Weight Stabilization and Boiling Suppression in ReSTIR

## 1. 오늘의 개념

어제는 **BRDF-Lobe-Aware Firefly Rejection and Energy-Preserving Specular Clamping**을 통해 한 번의 극단적인 Monte Carlo sample이 temporal moment와 denoiser history를 오염시키는 과정을 다뤘다. 오늘은 문제를 한 단계 upstream으로 옮겨, denoiser에 들어오기 전 **ReSTIR(Reservoir-based Spatiotemporal Importance Resampling)** 자체에서 나타나는 시간적 불안정성을 본다.

ReSTIR의 핵심은 많은 후보를 모두 저장하지 않고, 중요도에 따라 하나의 대표 sample과 통계량만 남기는 **weighted reservoir sampling**이다. Direct illumination을 단순화하면 candidate `x_i`의 resampling weight는

`w_i = p_hat(x_i) / p_i(x_i)`

처럼 생각할 수 있다.

- `p_hat(x_i)`: 현재 shading point에서 candidate가 얼마나 유용한지를 나타내는 target function
- `p_i(x_i)`: 해당 candidate가 생성된 proposal PDF

Reservoir는 대표 sample `y`, 누적 weight `W_sum`, 후보 수 또는 effective multiplicity `M` 등을 유지한다.

`W_sum = Σ_i w_i`

candidate `i`가 reservoir의 대표 sample로 선택될 확률은 개념적으로

`P(select i) = w_i / W_sum`

이다.

문제는 spatial/temporal reuse가 반복되면 reservoir가 단순한 "현재 frame의 한 sample"이 아니라 **과거 여러 frame과 주변 pixel의 선택 압력이 압축된 상태**가 된다는 점이다. 과거 reservoir의 `W_sum`이나 `M`이 지나치게 강하면 새로운 candidate가 거의 선택되지 않아 scene change에 늦게 반응한다. 반대로 dominant sample이 교체되는 순간에는 한 pixel 또는 작은 영역의 lighting state가 갑자기 바뀌어 화면이 끓는 듯한 **boiling / sparkle / temporal correlation artifact**가 나타날 수 있다.

Boiling은 firefly와 비슷하게 보일 수 있지만 원인은 다르다.

- **Firefly**: 개별 estimator contribution의 극단값이 문제다.
- **Boiling**: reservoir의 winner가 frame마다 불연속적으로 바뀌거나, 동일/유사 sample이 시공간적으로 과도하게 복제되어 생기는 correlation이 문제다.

따라서 오늘의 핵심은 단순한 luminance clamp가 아니다. **reservoir의 통계적 영향력, history age, spatial compatibility, sample duplication, visibility 변화가 서로 어떤 식으로 temporal stability를 결정하는가**를 이해하는 것이다.

## 2. 한 줄 핵심

**ReSTIR의 temporal stability는 밝은 sample을 자르는 문제가 아니라 과거 reservoir의 통계적 영향력이 현재 candidate를 압도하지 않도록 `M`, weight normalization, history validity와 sample correlation을 함께 제어하는 문제다.**

## 3. 왜 중요한가

### 3.1 Reservoir는 작은 메모리에 거대한 sample history를 압축한다

ReSTIR의 강점은 수많은 후보를 모두 유지하지 않아도 된다는 점이다. 하나의 reservoir가 대략 다음 의미를 가진다.

`R = {selected sample y, weight state, M, auxiliary metadata}`

이 구조는 GPU에 매우 잘 맞는다. Pixel당 수십~수백 candidate를 보존하는 대신 compact structured buffer를 유지하고, temporal pass와 spatial pass에서 reservoir끼리 다시 결합할 수 있다.

하지만 압축은 곧 **정보 손실**이기도 하다. Reservoir에 동일한 `W_sum`이 저장되어 있어도 그것이

- 다양한 independent candidate가 누적된 결과인지,
- 거의 같은 path가 여러 번 복제된 결과인지,
- 오래된 history 하나가 계속 살아남은 결과인지

만으로는 구분하기 어렵다.

즉 숫자상 큰 `M`이 실제 independent sample count와 같다고 볼 수 없다. ReSTIR에서는 sample count뿐 아니라 **correlation structure**가 품질을 결정한다.

### 3.2 오래된 reservoir가 현재 frame을 잠그는 현상

Temporal reuse에서 previous reservoir와 current reservoir를 결합한다고 생각해 보자.

과거 reservoir가 `M_prev = 30`, 현재 새 후보가 `M_curr = 1`에 해당하는 영향력을 가진다면, surface와 lighting이 거의 변하지 않는 동안에는 매우 효율적이다. 하지만 light가 이동하거나 visibility가 변한 순간에는 문제가 된다.

과거 state가 아직 target function에서 높은 평가를 받는다면 reservoir가 새 candidate를 쉽게 받아들이지 않는다. 결과적으로

`high reuse -> low variance -> slow response`

라는 trade-off가 생긴다.

이것은 TAA의 history length와 비슷해 보이지만 더 까다롭다. TAA는 보통 color를 연속적으로 blend하지만 reservoir는 **discrete representative sample**을 선택한다. 따라서 history가 교체되는 순간 lighting state가 더 크게 튈 수 있다.

그래서 RTXDI의 temporal ReSTIR API에는 `maxHistoryLength` 같은 제한이 존재한다. 오래된 reservoir의 influence가 무한히 누적되지 않게 하는 것은 단순한 메모리 관리가 아니라 responsiveness를 보장하는 통계적 제어다.

### 3.3 Boiling은 variance뿐 아니라 correlation 문제다

전통적인 Monte Carlo 관점에서는 variance를 줄이면 이미지가 좋아진다고 생각하기 쉽다. 하지만 실시간 animation에서는 frame 간 correlation도 중요하다.

Noise가 매 frame 독립적이면 grain 형태로 보일 수 있지만, 특정 sample이 넓은 영역에 복제되었다가 한꺼번에 다른 sample로 교체되면 사람이 훨씬 잘 인지하는 low-frequency flicker가 된다.

이를 개념적으로 두 축으로 볼 수 있다.

- **Estimator variance**: 정답 주변에서 값이 얼마나 흔들리는가
- **Temporal/spatial covariance**: 서로 다른 pixel/frame의 error가 얼마나 같이 움직이는가

2026년 NVIDIA 연구인 **Compatibility-Guided Neighbor Selection for ReSTIR**은 neighbor selection을 개선해 error뿐 아니라 temporal covariance도 줄이는 방향을 보여준다. 즉 modern ReSTIR 품질 평가는 단순 MSE보다 "오류가 시공간적으로 어떻게 묶여 보이는가"까지 본다.

### 3.4 Visibility 변화는 reservoir weight를 순간적으로 무효화한다

ReSTIR DI/GI에서 candidate가 과거 surface에서는 유효했지만 현재 surface에서는 occluded될 수 있다. 이때 stored sample 자체의 light/path 정보는 여전히 좋아 보여도 실제 contribution은 0에 가까워진다.

따라서 reuse 과정의 bias correction과 visibility validation은 temporal stability에 직접 연결된다.

- visibility 검사를 너무 적게 하면 stale visible sample이 남아 bright leakage가 생길 수 있다.
- visibility 검사를 공격적으로 하고 invalid reservoir를 바로 버리면 disocclusion 영역에서 sample count가 급락해 noise가 폭증할 수 있다.

RTXDI는 temporal/spatial resampling에서 bias correction mode, depth/normal threshold, reservoir age, fallback sampling 등을 별도 parameter로 둔다. 이것은 "reservoir weight"만으로 reuse validity를 결정할 수 없다는 뜻이다.

### 3.5 Denoiser는 ReSTIR boiling을 완전히 해결할 수 없다

Denoiser는 radiance signal을 안정화하지만 reservoir winner 자체가 frame마다 다른 light/path로 바뀌면 motion-compensated history가 불안정해진다.

특히 specular나 high-frequency visibility에서는

`reservoir instability -> radiance discontinuity -> denoiser history rejection -> low history length -> noise 증가`

의 feedback loop가 생길 수 있다.

따라서 robust real-time path tracer에서는 sampling과 denoising을 독립된 두 모듈로 보지 않고 다음처럼 연결해서 생각하는 것이 좋다.

`candidate generation -> reservoir reuse -> confidence/validity -> radiance reconstruction -> denoiser history`

Upstream sampling이 안정적일수록 downstream denoiser가 더 긴 history를 안전하게 사용할 수 있다.

## 4. 구현 관점

### 4.1 Reservoir state의 핵심 의미

Implementation에서 reservoir 구조체는 알고리즘마다 다르지만, 개념적으로 다음 정보가 필요하다.

- selected sample reference
- accumulated / normalized weight
- sample count `M`
- target PDF 또는 inverse PDF 관련 값
- sample age
- visibility reuse metadata
- GI라면 secondary surface position/normal/radiance 같은 path state

RTXDI의 `RTXDI_DIReservoir`는 sample reference, weight state, `M`, visibility 정보를 유지하며 packed representation을 `RWStructuredBuffer`에 저장한다. `FinalizeResampling` 이후 weight state는 shading에 사용할 inverse PDF 성격의 값으로 변환된다.

여기서 graphics engineer가 특히 주의해서 봐야 할 것은 **같은 float field가 resampling 중과 final shading 단계에서 서로 다른 semantic을 가질 수 있다는 점**이다. Reservoir struct는 단순 데이터 컨테이너가 아니라 pipeline stage 간 statistical contract다.

### 4.2 History length cap은 단순 `M = min(M, maxM)`이 아니다

Temporal reuse의 `M`을 제한하는 목적은 과거 reservoir가 무한히 강해지는 것을 막는 것이다. 다만 reservoir의 weight와 `M`은 normalization에서 연결되어 있으므로 `M`만 숫자로 줄이면 estimator 의미가 깨질 수 있다.

중요한 관점은 다음과 같다.

`history compression = influence compression`

즉 history cap은 과거 candidate의 multiplicity를 제한하면서, 이에 대응하는 weight normalization도 같은 의미로 유지되어야 한다.

실무에서는 이 부분이 자주 bug source가 된다.

- visual result는 그럴듯하지만 scene brightness가 미세하게 변함
- moving light에서만 energy가 튐
- static frame에서는 정상인데 camera motion에서 bias가 보임

이런 현상은 reservoir의 `M`, accumulated weight, target PDF, final normalization 중 하나의 semantic이 pass 사이에서 어긋났을 가능성을 의심할 수 있다.

### 4.3 Weighted radiance 기반 boiling detection

RTXDI의 ReSTIR GI에는 compute shader thread group 안에서 동작하는 **GI Boiling Filter**가 존재하며, 주변보다 weighted radiance가 비정상적으로 큰 reservoir를 제거해 boiling을 완화한다.

여기서 중요한 것은 raw radiance만 보는 것이 아니라 **reservoir weight가 반영된 최종 영향력**을 본다는 점이다.

개념적으로 안정성 metric을

`E_i = luminance(L_i) · W_i`

처럼 생각할 수 있다.

그리고 group 또는 neighborhood의 robust scale과 비교해

`E_i >> local expected energy`

인 reservoir를 unstable candidate로 분류할 수 있다.

이 방식은 어제의 firefly rejection과 닮았지만 의미는 다르다. 어제는 raw Monte Carlo contribution outlier를 찾았다면 오늘은 **reuse를 거치며 증폭된 reservoir state의 영향력 outlier**를 찾는다.

### 4.4 Thread-group 기반 filter의 GPU 장점과 한계

Boiling filter를 compute thread group 내부에서 처리하면 neighboring reservoir statistic을 group-shared memory 또는 subgroup/wave operation으로 빠르게 공유할 수 있다.

장점:

- global memory round-trip을 줄일 수 있다.
- group-local reduction으로 평균/최댓값/robust statistic을 계산하기 쉽다.
- reservoir storage와 같은 screen-space layout을 사용하기 좋다.

한계:

- thread-group boundary에서 neighborhood가 끊긴다.
- geometry edge를 모르는 단순 group statistic은 서로 다른 surface를 섞을 수 있다.
- very glossy reflection처럼 valid high-energy sample이 sparse한 경우 true signal을 제거할 수 있다.

따라서 production 관점에서는 boiling suppression을 단순 image filter라기보다 **reservoir validity + surface compatibility + statistical outlier control**로 보는 편이 안전하다.

### 4.5 Temporal reuse의 compatibility gate

Previous reservoir를 current pixel에 가져올 때 최소한 다음 guide들이 함께 고려된다.

`compatibility = depth × normal × material/path state × motion × age × visibility`

특히 GI에서는 secondary hit까지 포함되므로 primary G-buffer만 맞는다고 충분하지 않을 수 있다. Reprojection이 맞더라도 reused path가 현재 pixel footprint를 대표하지 못하면 temporal correlation artifact가 생길 수 있다.

최근 ReSTIR 연구가 footprint-aware reconnection, reservoir splatting, compatibility-guided neighbor selection을 다루는 이유도 여기에 있다. 단순 screen-space proximity보다 **path가 현재 integration domain과 얼마나 호환되는지**가 중요해지고 있다.

### 4.6 Sample duplication과 correlation

Spatial reuse는 한 reservoir의 좋은 sample을 여러 neighboring pixel에 전달한다. 이는 빠른 variance reduction을 주지만, 같은 sample ancestry가 화면 넓은 영역으로 퍼지면 error가 서로 강하게 correlated된다.

예를 들어 8x8 영역의 많은 pixel이 결국 동일한 light/path sample의 후손이라면 숫자상으로는 각 pixel마다 reservoir가 있지만 실제 independent information은 훨씬 적다.

이때 다음 frame에서 해당 sample이 invalid해지면 전체 영역이 동시에 다른 sample로 전환되어 큰 boiling이 보일 수 있다.

2024년의 **Decorrelating ReSTIR Samplers via MCMC Mutations**은 reservoir resampling 사이에 mutation을 넣어 이런 correlation과 sample impoverishment를 줄이는 방향을 제시했고, 2026년 **ReSTIR PT Enhanced**는 duplication map으로 spatiotemporal correlation을 줄이는 engineering improvement를 제시한다.

즉 reservoir stability의 고급 관점은 `large M = good`이 아니라

`useful reuse / independent diversity`

의 비율을 보는 것이다.

### 4.7 Memory layout

Full-HD 기준으로 pixel당 reservoir가 32 bytes라면 single buffer는 대략

`1920 × 1080 × 32 ≈ 63 MiB`

가 된다. Temporal ping-pong, DI/GI 분리, multiple reservoir layer까지 사용하면 storage가 빠르게 커진다.

그래서 reservoir packing은 중요한 GPU engineering 문제다.

관찰할 항목:

- sample index / light index를 32-bit integer로 유지할지
- sample UV를 half 또는 packed format으로 줄일지
- `M`, age, visibility flags를 bit field로 합칠지
- weight/inverse PDF를 FP16으로 내릴 수 있는 dynamic range인지
- secondary position을 world-space float3로 둘지 quantized/local coordinate로 저장할지

Weight 관련 값은 dynamic range가 크기 때문에 무조건 FP16으로 내리면 overflow/underflow가 temporal artifact로 나타날 수 있다. 반대로 모든 field를 FP32로 유지하면 reservoir bandwidth가 커져 temporal/spatial reuse pass가 memory-bound가 되기 쉽다.

따라서 reservoir format은 **precision budget과 bandwidth budget을 함께 설계하는 문제**다.

### 4.8 C++ / Render Graph 관점

C++ renderer에서는 ReSTIR을 한 shader 기능으로 보기보다 resource lifetime을 가진 여러 pass의 계약으로 보는 편이 좋다.

개념적 pipeline은 다음과 같다.

`Initial Candidate Generation`

`-> Temporal Reservoir Reuse`

`-> Boiling / Stability Control`

`-> Spatial Reservoir Reuse`

`-> Final Visibility / Shading`

`-> Denoiser`

여기서 render graph가 관리해야 할 핵심 resource는

- current/previous reservoir buffer
- current/previous G-buffer 또는 surface data
- neighbor offset / sampling tables
- motion vectors
- optional visibility metadata
- output radiance/confidence

이다.

특히 reservoir buffer는 current/previous frame의 ping-pong 관계가 있으므로 frame index와 buffer index가 어긋나는 bug가 치명적이다. 이런 bug는 crash보다 "lighting이 이상하게 흔들리는" 형태로 나타나기 때문에 RenderDoc/Nsight에서 resource version과 pass dependency를 추적하는 능력이 중요하다.

## 5. 내 관심 분야와 연결

### 실시간 렌더링 / 게임 엔진

ReSTIR은 단순 ray tracing 논문 주제가 아니라 real-time engine에서 "적은 ray budget으로 많은 light/path 후보를 어떻게 재사용할 것인가"라는 문제의 대표적인 해법이다. 특히 GPU ray tracing pipeline과 denoiser, TAA/upscaler 사이의 연결을 이해하는 데 좋다.

### GPU Compute

Reservoir resampling은 대부분 screen-space compute pass로 구성하기 좋고, thread-group local statistic, subgroup operation, structured buffer, packed state, random number generation이 모두 등장한다. 즉 compute shader의 ALU 최적화뿐 아니라 **memory traffic와 correlation-aware sampling**을 함께 보는 사례다.

### Scientific Visualization / Simulation

ReSTIR 자체는 light transport 기법이지만 reservoir sampling의 사고방식은 시뮬레이션 visualization에도 연결할 수 있다. 예를 들어 수많은 particle/voxel/cell 후보에서 화면 기여도가 높은 representative sample을 유지하거나, 시간축에서 중요 sample을 재사용하는 문제는 유사한 통계적 구조를 가진다.

특히 volume rendering에서 ReSTIR 계열 연구가 실제로 존재한다는 점은 CFD/scientific volume renderer에서도 sample reuse와 temporal stability가 연결될 수 있음을 보여준다.

### C++ / API 설계

Reservoir의 핵심 난점은 shader 수식만이 아니다. CPU side에서 `maxHistoryLength`, bias correction mode, depth/normal threshold, spatial sample count, sampling radius 같은 parameter가 shader-side statistical semantics와 정확히 맞아야 한다.

이런 시스템은 graphics engineer가 **C++ API contract, resource state, GPU memory layout, Monte Carlo estimator를 동시에 이해해야 하는 대표 사례**다.

## 6. 머릿속에 남길 질문 3개

1. **Reservoir의 `M`이 32라고 할 때, 그것을 정말 32개의 independent sample이 누적된 것으로 해석해도 되는가? 아니라면 spatial/temporal reuse가 만드는 correlation을 어떤 별도 metric으로 볼 수 있을까?**

2. **History length를 늘리면 variance는 감소하지만 moving light와 visibility change에 대한 response가 느려진다. Renderer가 이 trade-off를 surface/material/path type별로 다르게 가져가야 하는 이유는 무엇일까?**

3. **Boiling filter가 high-energy reservoir를 제거할 때 true caustic/glossy highlight와 unstable outlier를 어떻게 구분해야 하며, 이 판단에 roughness, path footprint, visibility age를 어떻게 연결할 수 있을까?**

## 7. graphics engineer 면접 질문 1개와 답변

### 질문

**ReSTIR에서 temporal reuse를 강하게 하면 noise가 줄어드는데도 오히려 animation 품질이 나빠질 수 있는 이유를 reservoir의 `M`, weight, correlation 관점에서 설명해보세요.**

### 답변

Temporal reuse는 이전 frame의 reservoir를 현재 frame에 결합해 effective candidate 수를 크게 늘리므로 static scene에서는 variance를 빠르게 줄인다. 하지만 과거 reservoir의 `M`과 statistical weight가 계속 누적되면 current candidate가 선택될 확률이 낮아져 moving light, visibility change, disocclusion에 늦게 반응할 수 있다.

또한 spatial/temporal reuse는 같은 sample의 descendants를 여러 pixel과 frame에 복제할 수 있다. 이 경우 reservoir 수나 `M`은 커져도 실제 independent information은 그만큼 늘지 않으며, errors가 서로 correlated된다. Dominant sample이 invalid해지거나 다른 sample로 교체되면 넓은 영역이 함께 변해 boiling이나 flicker로 보일 수 있다.

따라서 production ReSTIR에서는 단순히 history를 길게 유지하는 것이 아니라 history length/age 제한, surface compatibility, visibility/bias correction, spatial neighbor selection, correlation reduction을 함께 설계해야 한다. 핵심은 **variance reduction과 temporal responsiveness, sample diversity의 균형**이다.

## 8. 포트폴리오 / 커리어 연결

ReSTIR을 포트폴리오에서 설명할 때 "논문 구현"만 강조하면 알고리즘 재현에 머무를 수 있다. Graphics engineer 관점에서는 다음 연결이 더 강하다.

- reservoir tuple의 statistical semantic을 설명할 수 있는가
- temporal/spatial pass가 GPU memory bandwidth에 어떤 영향을 주는가
- history cap과 bias correction이 image quality에 어떤 trade-off를 만드는가
- screen-space neighbor reuse가 correlation artifact를 만드는 이유를 설명할 수 있는가
- reservoir storage precision과 packed layout을 어떻게 판단하는가
- sampling output이 NRD/SVGF류 denoiser history에 어떤 영향을 주는가
- RenderDoc/Nsight에서 reservoir buffer와 reprojection failure를 어떻게 진단할 것인가

특히 2026년 ReSTIR 연구 흐름은 단순한 "더 많은 reuse"보다 **reuse의 quality, compatibility, decorrelation, disocclusion robustness**에 집중하고 있다. 이 관점을 이해하면 최신 real-time path tracing 기술을 논문 수준뿐 아니라 production engineering 문제로 설명할 수 있다.

면접이나 포트폴리오 설명에서 좋은 핵심 문장은 다음과 같다.

> ReSTIR의 성능 이점은 sample을 많이 저장하는 데서 나오지 않고, 작은 reservoir state로 중요한 후보를 시공간에 재사용하는 데서 나온다. 그러나 reuse가 강할수록 correlation과 stale history가 커질 수 있으므로, 실제 엔진에서는 weight normalization, history validity, compatibility, memory layout을 하나의 시스템으로 설계해야 한다.

## 9. 내일 이어서 볼 개념

**Reservoir Compatibility and Bias Correction: Pairwise MIS, Visibility Reuse, and Neighbor Selection**

오늘은 reservoir가 "얼마나 오래/강하게 남는가"를 봤다. 다음에는 한 단계 더 나아가 **서로 다른 pixel/frame의 reservoir를 결합할 때 왜 단순 합치기가 biased해질 수 있는지**, target function mismatch, pairwise MIS, visibility reuse, compatibility-guided neighbor selection을 중심으로 본다.

특히 다음 연결이 핵심이다.

`reuse quantity -> reuse validity -> bias / correlation control`

이 흐름을 이해하면 ReSTIR DI/GI/PT 계열이 왜 다양한 bias correction과 neighbor selection 전략을 필요로 하는지 연결해서 볼 수 있다.

## 10. 참고 키워드

- ReSTIR (Reservoir-based Spatiotemporal Importance Resampling)
- Weighted Reservoir Sampling
- Resampled Importance Sampling (RIS)
- Reservoir `M`, weight sum, inverse PDF
- Target Function / Target PDF
- Temporal Reuse / Spatial Reuse
- History Length / Reservoir Age
- Boiling Artifact / Temporal Correlation
- Sample Impoverishment / Sample Duplication
- Bias Correction
- Visibility Reuse
- Pairwise MIS
- Reprojection / Disocclusion
- Compatibility-Guided Neighbor Selection
- Reservoir Splatting
- Duplication Map
- MCMC Mutation
- RTXDI
- ReSTIR DI / ReSTIR GI / ReSTIR PT
- `RWStructuredBuffer`
- Packed Reservoir
- GPU Memory Bandwidth
- FP16 vs FP32 Weight Precision
- Subgroup / Wave Operations
- Compute Shader Thread Group
- Render Graph / Temporal Ping-Pong Resource
- NVIDIA RTXDI Shader API
- Bitterli et al., **Spatiotemporal Reservoir Resampling for Real-time Ray Tracing with Dynamic Direct Lighting**, SIGGRAPH 2020
- Wyman & Panteleev, **Rearchitecting Spatiotemporal Resampling for Production**, HPG 2021
- Sawhney et al., **Decorrelating ReSTIR Samplers via MCMC Mutations**, 2024
- Lin, Kettunen & Wyman, **ReSTIR PT Enhanced: Algorithmic Advances for Faster and More Robust ReSTIR Path Tracing**, 2026
- Junkins et al., **Compatibility-Guided Neighbor Selection for ReSTIR**, 2026
- Hong et al., **Multi-Layer Reservoir Splatting for Temporal Reuse under Disocclusion**, 2026
