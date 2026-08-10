---
title: "Reservoir Compatibility and Bias Correction: Pairwise MIS, Visibility Reuse, and Neighbor Selection"
date: "2026-08-10"
category: Graphics
tags: ["GPU", "ReSTIR", "GRIS", "Pairwise MIS", "Visibility Reuse", "Bias Correction", "Neighbor Selection", "Ray Tracing", "Compute Shader", "Memory Layout", "Real-Time Rendering"]
level: intermediate
---

# [Daily Graphics Growth] 2026-08-10 - Reservoir Compatibility and Bias Correction: Pairwise MIS, Visibility Reuse, and Neighbor Selection

## 1. 오늘의 개념

어제는 **Reservoir Weight Stabilization and Boiling Suppression in ReSTIR**을 통해 reservoir의 큰 `M`, 오래된 history, sample duplication과 correlation이 어떻게 temporal boiling으로 이어지는지 보았다. 오늘은 그 다음 단계인 **“이 reservoir를 다른 pixel/frame에서 가져와도 되는가?”**를 다룬다.

ReSTIR의 spatial/temporal reuse는 단순히 이웃 pixel의 sample을 복사하는 과정이 아니다. 서로 다른 shading domain에서 생성된 sample을 현재 pixel의 target domain으로 옮겨 다시 평가하고, 여러 proposal이 같은 적분값을 추정하도록 **bias correction**과 **Multiple Importance Sampling(MIS)** 의미를 유지해야 한다.

핵심 문제는 세 가지다.

1. **Reservoir compatibility**: 이웃 reservoir의 path/light sample이 현재 surface에서도 의미가 있는가?
2. **Bias correction / Pairwise MIS**: 서로 다른 domain의 sample을 섞을 때 contribution weight를 어떻게 보정하는가?
3. **Visibility reuse**: 이전 frame에서 계산한 shadow visibility를 재사용할 때 ray cost와 bias 사이를 어떻게 조절하는가?

ReSTIR에서 좋은 neighbor는 단순히 가까운 pixel이 아니다. 현재 pixel과 **path-space support가 충분히 겹치고**, shift/reconnection 후 target function이 안정적으로 평가되며, visibility와 geometry state가 지나치게 달라지지 않는 reservoir가 좋은 neighbor다.

이를 개념적으로 다음처럼 볼 수 있다.

`reuse quality ≈ compatibility × correct weighting × valid visibility × low correlation`

어제까지는 reservoir 내부의 weight가 얼마나 안정적인지를 봤다면, 오늘은 **reservoir와 reservoir 사이의 관계**를 본다.

---

## 2. 한 줄 핵심

**ReSTIR의 reuse 품질은 “가까운 이웃을 많이 쓰는 것”이 아니라, 현재 pixel과 호환되는 reservoir만 가져오고 Pairwise MIS·visibility validation으로 domain mismatch가 만드는 bias를 제어하는 데서 결정된다.**

---

## 3. 왜 중요한가

### 3.1 Reuse는 sample 복사가 아니라 domain transfer다

현재 pixel을 `c`, neighbor pixel을 `i`라고 하자. Neighbor reservoir가 가진 selected sample `X_i`는 원래 neighbor의 shading domain에서 생성되었다.

Spatial reuse에서는 이 sample을 현재 pixel에서 사용할 수 있는 형태로 옮긴다.

`Y_i = T_i(X_i)`

여기서 `T_i`는 **shift mapping / reconnection mapping**이다.

Direct illumination에서는 같은 light sample을 현재 shading point에서 다시 평가하는 비교적 단순한 형태가 가능하지만, GI나 path tracing에서는 path vertex를 reconnect하거나 path-space domain을 변경해야 한다. 이때 mapping이 invalid하거나 Jacobian, target function, visibility 의미가 달라지면 단순 weight reuse는 bias로 이어진다.

즉 reuse는

`neighbor sample -> current-domain sample -> target reevaluation -> MIS normalization`

이라는 변환 pipeline이다.

### 3.2 왜 MIS가 필요한가

한 pixel의 최종 candidate는 여러 source에서 올 수 있다.

- current-frame canonical sample
- previous-frame temporal reservoir
- spatial neighbor reservoir
- light sampler
- BRDF sampler
- GI path reservoir

같은 최종 path `y`가 여러 proposal에서 생성될 수 있다면, 한 proposal이 우연히 높은 probability를 가진다고 해서 그 contribution을 그대로 더하면 중복 계산 또는 variance 증가가 생길 수 있다.

MIS의 목적은 각 proposal의 상대적 설명력을 비교해 sample contribution을 나누는 것이다.

전통적인 balance heuristic을 단순화하면

`m_i(y) = p_i(y) / Σ_j p_j(y)`

처럼 이해할 수 있다.

하지만 ReSTIR/GRIS에서는 proposal PDF를 직접 알기 어렵고 sample reuse로 correlation까지 생긴다. 그래서 **target function과 unbiased contribution weight(UCW)**, confidence weight를 이용한 generalized resampling weight가 필요해진다.

### 3.3 Generalized balance heuristic의 비용 문제

여러 reservoir candidate `M`개를 모두 서로 비교하는 generalized balance heuristic은 각 candidate를 여러 domain에서 다시 평가해야 하므로 평가량이 빠르게 커진다.

개념적으로 모든 candidate `i`에 대해 모든 domain `j`에서 target을 확인하면

`O(M²)`

평가가 필요할 수 있다.

Real-time ray tracing에서는 target evaluation 자체가 BRDF, geometry term, visibility, path reconnection을 포함할 수 있으므로 `M²`은 매우 비싸다.

**Generalized Pairwise MIS**는 각 non-canonical candidate를 canonical domain과 pair로 비교해 비용을 `O(M)` 수준으로 줄인다.

개념적으로 neighbor `i`의 pairwise weight는 다음과 같은 두-domain balance로 생각할 수 있다.

`m_i(y) ∝ c_i p̂_i(y) / (c_i p̂_i(y) + c_c p̂_c(y))`

여기서

- `p̂_i(y)`: neighbor domain에서의 target-like quantity
- `p̂_c(y)`: canonical/current domain에서의 target-like quantity
- `c_i`, `c_c`: confidence weight

이다.

정확한 ReSTIR/GRIS 공식은 shift mapping과 UCW 정의를 포함하지만, graphics engineer 관점에서 기억할 핵심은 **“모든 source를 서로 비교하지 않고 canonical과 pair를 만들어 상대적 호환성을 계산한다”**는 것이다.

2026년의 **Stochastic Pairwise MIS**는 여기서 더 나아가, 큰 spatial neighborhood 전체를 비싸게 평가하지 않고 일부 candidate를 stochastic하게 선택하면서도 unbiased expectation을 유지하는 방향을 제시한다. 이는 disocclusion처럼 useful reservoir가 sparse한 영역에서 특히 효과적이다.

### 3.4 호환되지 않는 neighbor는 variance만 늘리는 것이 아니다

두 pixel이 screen-space로 가까워도 다음 조건이 다르면 sample reuse가 나쁘다.

- surface normal이 크게 다름
- object/material ID가 다름
- depth가 크게 다름
- roughness 또는 BRDF lobe가 다름
- secondary path vertex가 현재 pixel과 연결 불가능
- visibility 상태가 다름
- path support가 거의 겹치지 않음

이런 neighbor를 포함하면 target function이 매우 작아져 weight가 거의 0이 되거나, 반대로 특정 reconnection에서 큰 ratio가 생길 수 있다.

또한 잘못된 neighbor를 자주 고르면 GPU는 expensive target evaluation과 shadow ray를 수행하고도 실제로는 거의 기여하지 못한다.

따라서 neighbor selection은 품질 알고리즘이면서 동시에 **GPU work scheduling 문제**다.

### 3.5 2026 Compatibility-Guided Neighbor Selection의 의미

2026년 NVIDIA/University of Utah 연구인 **Compatibility-Guided Neighbor Selection for ReSTIR**은 spatial neighbor를 uniform random하게 선택하는 대신 path compatibility를 반영하는 방법을 제시했다.

보고된 결과는 단순 MSE 감소만이 아니다.

- SMAPE 6–29% 감소
- temporal covariance 22–49% 감소
- 추가 비용 2–5%

여기서 중요한 것은 **temporal covariance**다. ReSTIR artifacts는 pixel별 독립 noise보다 여러 pixel/frame이 함께 흔들릴 때 훨씬 눈에 띈다. 좋은 neighbor selection은 variance만 낮추는 것이 아니라 correlated error의 구조도 개선한다.

이것은 어제 다룬 boiling 문제와 직접 이어진다.

`bad neighbor reuse -> duplicated/correlated reservoir -> temporal covariance 증가 -> boiling`

반대로

`compatibility-aware neighbor -> useful reuse 비율 증가 -> lower covariance -> denoiser history 안정`

이라는 흐름을 만들 수 있다.

### 3.6 Visibility reuse는 성능 최적화이면서 bias knob이다

Direct illumination에서 reservoir의 light sample을 선택한 뒤 최종 shadow ray를 매번 쏘면 정확하지만 비싸다.

RTXDI는 reservoir에 이전 visibility를 저장하고, 일정 조건 아래에서 이를 재사용할 수 있다.

주요 조건은 다음과 같다.

- stored visibility age
- current 위치와 stored 위치의 distance

Threshold를 넓히면 shadow ray 수를 줄일 수 있지만 stale visibility를 재사용할 가능성이 커진다.

즉

`more visibility reuse -> fewer rays -> more bias risk`

이다.

특히 shadow boundary에서 과거에는 visible이었지만 현재는 occluded인 sample을 재사용하면 bright leakage가 생길 수 있다. 반대로 zero visibility sample을 지나치게 공격적으로 discard하면 darkening bias가 생길 수 있다.

RTXDI 문서가 final visibility가 zero인 reservoir를 무조건 discard하는 것을 경고하는 이유도 여기에 있다.

---

## 4. 구현 관점

### 4.1 ReSTIR pipeline에서 compatibility가 들어가는 위치

실무적인 render graph 관점에서 ReSTIR DI pipeline을 단순화하면 다음과 같다.

`G-buffer`
`-> Initial Candidate Sampling`
`-> Temporal Resampling`
`-> Spatial Resampling`
`-> Final Visibility / Shading`
`-> Denoiser`

Compatibility test는 한 곳에만 있지 않는다.

**Temporal pass**에서는

- motion/reprojection validity
- depth/normal consistency
- reservoir age
- object/material continuity

를 본다.

**Spatial pass**에서는

- depth/normal/object compatibility
- path/light support overlap
- roughness/material similarity
- neighbor reuse quality

를 본다.

**Final shading**에서는

- stored visibility reuse 가능 여부
- 최종 shadow ray 필요 여부

를 판단한다.

즉 compatibility metadata는 ReSTIR pass 전체를 관통한다.

### 4.2 Reservoir struct는 statistical state다

RTXDI의 `RTXDI_DIReservoir`는 개념적으로 다음 의미를 가진다.

- selected light/sample reference
- reservoir weight state
- sample count `M`
- visibility state

그리고 storage 단계에서는 `RTXDI_PackedDIReservoir` 형태로 압축된다.

중요한 점은 `weightSum` 같은 field의 semantic이 resampling 중과 finalize 이후 다를 수 있다는 점이다.

Resampling 중:

`weightSum ≈ accumulated resampling weights`

Finalize 이후:

`weightSum -> inverse-PDF-like shading weight`

이런 semantic transition을 C++/shader interface에서 명확히 관리하지 않으면 매우 찾기 어려운 brightness bias가 생긴다.

Graphics engine에서는 resource 이름을 단순히 `reservoirBuffer`로 끝내기보다 pass state를 contract로 분명하게 정의하는 것이 중요하다.

예를 들면 개념적으로

- `InitialReservoir`
- `TemporalReservoir`
- `SpatialReservoir`
- `FinalReservoir`

처럼 stage semantics를 구분하는 것이 reasoning에 유리하다.

### 4.3 GPU memory layout

Reservoir reuse는 화면 전체 pixel에 대해 수행되므로 구조체 크기가 bandwidth를 직접 결정한다.

예를 들어 4K 해상도는 약 830만 pixel이다. Pixel당 reservoir가 32 byte라면 한 buffer만으로도 약

`8.3M × 32 B ≈ 266 MB`

가 된다.

Temporal ping-pong, spatial intermediate, GI reservoir를 따로 두면 빠르게 수백 MB에 도달한다.

따라서 production 구현은 보통

- packed light/sample ID
- quantized UV
- compressed visibility
- compact `M` / age
- FP16 가능 field

를 검토한다.

하지만 weight normalization과 target ratio는 dynamic range가 크기 때문에 무조건 FP16로 줄이는 것은 위험하다. 특히 glossy/specular, tiny light, near-grazing geometry에서는 weight가 크게 변할 수 있다.

Memory 최적화에서 중요한 원칙은

**“storage precision을 줄이되 estimator semantic은 줄이지 않는다.”**

이다.

### 4.4 Neighbor read는 bandwidth/coherence 문제다

Spatial ReSTIR은 주변 reservoir를 random access한다. 따라서 thread가 서로 다른 neighbor를 읽으면 cache locality가 나빠질 수 있다.

GPU 관점에서 neighbor selection은 다음 trade-off를 가진다.

- 완전 random neighbor: correlation 감소 가능, cache locality 나쁨
- tile-local neighbor: cache locality 좋음, local correlation 증가 가능
- compatibility-cell neighbor: useful sample 비율 증가, 별도 classification 비용 발생

2026 Stochastic Pairwise MIS 연구는 8×8 tile과 normal/object ID를 이용한 reuse cell을 구성하고, 큰 neighborhood 중 일부 contributing candidate를 선택하는 방향을 보여준다.

이는 단순 statistical idea가 아니라 **memory-coherent candidate filtering**으로 해석할 수 있다.

즉 good ReSTIR design은

`statistical efficiency × cache efficiency × ray efficiency`

를 동시에 본다.

### 4.5 Pairwise MIS의 shader cost

Pairwise MIS가 `O(M)`이라고 해도 각 pair 평가가 싸다는 뜻은 아니다.

Candidate마다 다음 비용이 들어갈 수 있다.

- current surface에서 target function 평가
- shifted/reconnected sample 평가
- BRDF와 cosine term
- geometry term
- Jacobian
- confidence weight
- optional visibility

따라서 shader에서는 arithmetic count보다 **expensive path-space evaluation을 몇 번 호출하는가**가 중요하다.

또한 많은 candidate를 loop로 처리하면 register pressure가 커지고 occupancy가 떨어질 수 있다.

실무 profiling에서는 다음을 함께 본다.

- candidate count per pixel
- target evaluation count
- visibility ray count
- register usage
- occupancy
- L1/L2 hit rate
- reservoir buffer bandwidth

### 4.6 Visibility cache의 metadata contract

Stored visibility를 재사용하려면 단순 RGB visibility 값만 있어서는 부족하다.

최소한 다음 의미가 필요하다.

- 어떤 sample/light에 대한 visibility인가
- 어느 위치에서 계산했는가
- 얼마나 오래되었는가
- 현재 reservoir sample이 같은 sample identity를 유지하는가

RTXDI는 visibility age와 distance threshold를 사용해 stored visibility의 적용 가능성을 판단한다.

이것은 TAA history validity와 매우 비슷한 구조다.

`history value + validity metadata`

가 함께 있어야 한다.

Graphics pipeline 전반에서 같은 패턴이 반복된다.

- TAA: color + depth/normal/motion
- denoiser: radiance + moments + history length
- ReSTIR: sample + weight + M + visibility + age
- sparse simulation cache: field value + transform/version/validity

즉 **“data 자체보다 data의 유효 범위를 설명하는 metadata가 중요하다”**는 점은 rendering과 simulation 모두에서 공통된 설계 원칙이다.

### 4.7 C++ render graph 관점

C++ engine에서 ReSTIR pass를 구성할 때 중요한 것은 algorithm 이름보다 resource dependency다.

Temporal resampling은 previous-frame reservoir와 previous G-buffer를 읽고 current G-buffer와 current candidate를 결합한다.

Spatial resampling은 temporal output과 current guide buffer를 읽는다.

Final shading은 spatial reservoir와 visibility cache를 읽고 필요하면 ray tracing workload를 발생시킨다.

따라서 frame graph에서는

- previous/current reservoir lifetime
- G-buffer history lifetime
- UAV barrier
- ping-pong index
- resize/reset 시 history invalidation
- camera cut 시 reservoir clear

가 명확해야 한다.

특히 resolution change, dynamic resolution, camera cut에서 previous reservoir를 잘못 유지하면 통계적 bias 이전에 memory addressing 자체가 잘못될 수 있다.

---

## 5. 내 관심 분야와 연결

### 5.1 Real-time rendering / game engine

ReSTIR은 modern real-time ray tracing에서 단순 sampling technique을 넘어 **lighting subsystem architecture**에 가깝다.

게임 엔진에서는 수백만 light, emissive geometry, GI, reflections를 low-spp로 처리해야 하기 때문에 sample reuse가 중요하다. 그러나 reuse가 aggressive할수록 motion, disocclusion, material edge에서 temporal artifact가 생긴다.

따라서 graphics engineer는 ReSTIR을 볼 때

- sampling theory
- ray tracing pipeline
- temporal reconstruction
- denoiser
- render graph
- GPU memory

를 하나의 시스템으로 연결해서 이해해야 한다.

### 5.2 CFD / scientific visualization

ReSTIR 자체는 light transport용이지만 compatibility-aware reuse라는 사고방식은 scientific visualization에도 직접 연결된다.

예를 들어 time-varying CFD volume에서 이전 frame의 sampling result를 재사용할 때도

- 같은 spatial region인가
- scalar/vector field가 크게 변했는가
- topology가 변했는가
- transfer function이 바뀌었는가

를 확인해야 한다.

즉 temporal cache reuse의 본질은 같다.

`reuse = cached value + domain compatibility + correction`

Volume ray marching, streamline seed reuse, adaptive sampling에서도 같은 설계 패턴이 등장한다.

### 5.3 Voxel / level-set / semiconductor visualization

Level-set이나 voxel 구조가 공정 step마다 변할 때 이전 frame의 surface/visibility/sample cache를 재사용하고 싶다면 surface identity와 topology revision이 중요하다.

특히 etch/deposition처럼 geometry가 크게 변하는 경우 단순 world-space proximity만으로 history를 이어 붙이면 다른 material layer의 cache를 잘못 가져올 수 있다.

ReSTIR의 compatibility gate는 다음과 비슷한 시뮬레이션 metadata 설계로 확장할 수 있다.

- material ID
- surface normal
- signed-distance range
- topology revision
- process step ID

즉 오늘의 개념은 ray tracing 전용 기법이면서 동시에 **GPU temporal cache design의 좋은 사례**다.

### 5.4 WebGPU / compute

WebGPU에서도 reservoir-style algorithm은 `storage_buffer`와 compute pass로 표현 가능하다. 핵심 제약은 API보다 memory traffic과 synchronization이다.

- previous/current storage buffer
- G-buffer texture sampling
- workgroup-local candidate reduction
- packed reservoir layout

등은 Vulkan/DX12/WebGPU에서 매우 유사한 구조를 가진다.

그래서 ReSTIR은 특정 API보다 **modern explicit GPU programming model**을 이해하는 데 좋은 주제다.

---

## 6. 머릿속에 남길 질문 3개

1. **Screen-space로 가까운 두 pixel이 실제 path-space에서도 좋은 reuse pair라는 보장은 왜 없는가?**

2. **Pairwise MIS가 generalized balance heuristic보다 계산량을 크게 줄일 수 있는 이유와, 그 대신 canonical sample이 왜 중요한 기준점이 되는가?**

3. **Stored visibility의 age/distance threshold를 크게 했을 때 ray 수는 줄지만 shadow boundary에서 어떤 종류의 bias가 증가할 수 있는가?**

---

## 7. graphics engineer 면접 질문 1개와 답변

**Q. ReSTIR spatial reuse에서 단순히 depth와 normal이 비슷한 neighbor reservoir를 가져오면 충분하지 않은 이유를 설명하고, Pairwise MIS와 visibility validation이 각각 어떤 문제를 해결하는지 설명해보세요.**

**A.** Depth와 normal은 surface geometry compatibility를 확인하는 좋은 1차 guide지만, 두 pixel의 light/path sampling domain이 동일하다는 보장은 없다. 같은 평면 위에서도 BRDF, roughness, light visibility, secondary bounce geometry가 다르면 neighbor sample의 target contribution이 현재 pixel에서 크게 달라질 수 있다.

Pairwise MIS는 neighbor sample과 canonical/current-domain sample을 상대적으로 평가해 여러 proposal에서 들어온 sample의 contribution을 적절히 분배한다. ReSTIR/GRIS에서는 정확한 proposal PDF 대신 target-like quantity, shift mapping, confidence와 UCW를 이용해 domain transfer의 weight를 보정한다. Generalized all-pairs evaluation보다 pairwise 방식은 보통 훨씬 낮은 평가 비용으로 real-time reuse를 가능하게 한다.

Visibility validation은 별개의 문제를 해결한다. MIS weight가 통계적으로 맞아도 현재 shading point와 selected light 사이가 occluded라면 실제 contribution은 달라진다. 이전 visibility를 재사용하면 shadow ray를 줄일 수 있지만, 오래되거나 멀리 떨어진 위치의 visibility를 사용하면 shadow edge에서 bright/dark bias가 생길 수 있다. 따라서 production ReSTIR은 geometry compatibility, statistical weight correction, visibility validity를 각각 독립적으로 관리해야 한다.

---

## 8. 포트폴리오 / 커리어 연결

ReSTIR을 포트폴리오에서 단순히 “reservoir sampling 구현”이라고 설명하는 것보다 다음 구조로 설명하는 편이 graphics engineer 역량을 더 잘 보여준다.

**Algorithm**

- weighted reservoir sampling
- temporal/spatial reuse
- Pairwise MIS / bias correction
- visibility reuse

**GPU architecture**

- packed reservoir buffer
- compute-based resampling
- random neighbor access와 cache locality
- ray budget 관리
- register pressure / occupancy

**Temporal robustness**

- depth/normal/material compatibility
- history age
- disocclusion reset
- boiling / correlation control

**Engine integration**

- C++ render graph
- previous/current resource lifetime
- camera cut / resize invalidation
- denoiser와의 signal contract

면접에서도 ReSTIR 공식 자체를 외우는 것보다 다음 trade-off를 설명하는 능력이 중요하다.

`more reuse -> lower variance but higher correlation/bias risk`

`more validation rays -> higher correctness but higher cost`

`larger neighborhood -> better chance of useful samples but higher evaluation/bandwidth cost`

그리고 최근 연구 흐름까지 연결하면 단순 논문 독해가 아니라 production 관점의 이해를 보여줄 수 있다.

2026년 연구 흐름은 특히 두 방향이 분명하다.

- **Stochastic Pairwise MIS**: 큰 neighborhood에서 useful candidate를 더 효율적으로 찾으면서 unbiased reuse 유지
- **Compatibility-Guided Neighbor Selection**: path compatibility를 이용해 variance와 temporal covariance를 함께 개선

이는 차세대 real-time path tracing에서 “더 많은 sample”보다 **더 좋은 reuse topology**가 중요해지고 있음을 보여준다.

---

## 9. 내일 이어서 볼 개념

**Shift Mapping and Jacobian-Aware Path Reuse: Reconnection, Domain Change, and Path-Space Support**

오늘은 reservoir 간 compatibility와 MIS correction을 중심으로 보았다. 내일은 그 안에서 실제로 sample을 한 shading domain에서 다른 domain으로 옮기는 **shift mapping**을 더 깊게 본다.

이어질 핵심은 다음이다.

- ReSTIR GI/PT에서 path reconnection이 필요한 이유
- invertible / non-invertible mapping
- Jacobian determinant가 contribution weight에 들어가는 이유
- specular chain과 rough surface에서 mapping이 어려워지는 이유
- footprint-aware reconnection과 path-space support
- geometry/LoD 변화가 temporal reuse를 깨는 방식

즉 오늘의 질문인

**“이 reservoir를 가져와도 되는가?”**

에서 한 단계 더 들어가

**“가져온 path를 현재 pixel의 적분 domain으로 어떻게 올바르게 변환하는가?”**

를 다룬다.

---

## 10. 참고 키워드

- ReSTIR — Reservoir-based Spatiotemporal Importance Resampling
- GRIS — Generalized Resampled Importance Sampling
- Weighted Reservoir Sampling
- Unbiased Contribution Weight (UCW)
- Multiple Importance Sampling (MIS)
- Generalized Balance Heuristic
- Pairwise MIS
- Defensive Pairwise MIS
- Stochastic Pairwise MIS
- Canonical Sample / Canonical Domain
- Non-Canonical Sample
- Shift Mapping
- Reconnection Mapping
- Target Function
- Confidence Weight
- Bias Correction
- Visibility Reuse
- Visibility Age
- Shadow Ray Budget
- Reservoir Compatibility
- Path Compatibility
- Neighbor Selection
- Temporal Covariance
- Disocclusion
- Reservoir Correlation
- RTXDI
- `RTXDI_DIReservoir`
- `RTXDI_PackedDIReservoir`
- `RTXDI_FinalizeResampling`
- `RTXDI_GetDIReservoirVisibility`
- Packed GPU Buffer
- RWStructuredBuffer / Storage Buffer
- Compute Shader
- Render Graph
- Temporal Resource Lifetime
- Camera-Cut Invalidation

참고 흐름: **Generalized Resampled Importance Sampling: Foundations of ReSTIR (2022)**, **RTXDI Shader API**, **Stochastic Pairwise MIS for Unbiased Large-Kernel Reuse in Real Time (2026)**, **Compatibility-Guided Neighbor Selection for ReSTIR (2026)**.