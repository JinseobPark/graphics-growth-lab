---
title: "Temporal Residency Prediction and Eviction Hysteresis: Prefetch Windows, Confidence Decay, and Thrash Control"
date: "2026-08-26"
category: Graphics
tags: [GPU, Sparse Residency, Temporal Prediction, Eviction Hysteresis, Prefetching, Working Set, Vulkan, CUDA, Sparse Volume, Brick Streaming, Compute Shader, Memory Layout, C++]
level: intermediate
---

# [Daily Graphics Growth] 2026-08-26 - Temporal Residency Prediction and Eviction Hysteresis: Prefetch Windows, Confidence Decay, and Thrash Control

## 1. 오늘의 개념
어제는 **GPU Feedback-Driven Sparse Residency**를 통해 shader·ray traversal·volume traversal이 실제로 요구한 logical brick을 feedback map으로 기록하고, 이를 compact request로 바꿔 다음 resident working set을 결정하는 구조를 봤다.

오늘은 그 다음 문제를 다룬다.

> **GPU feedback은 과거의 관측인데, sparse bind와 data population에는 latency가 있다. 그렇다면 필요한 brick을 늦지 않게 가져오면서도 VRAM을 불필요하게 소모하지 않으려면 어떻게 해야 하는가?**

이를 위해 필요한 두 축이 **Temporal Residency Prediction**과 **Eviction Hysteresis**다.

- **Temporal residency prediction**은 현재 frame의 request만 보는 것이 아니라 camera motion, 최근 request history, projected footprint, ray/volume traversal 추세, simulation activity를 이용해 가까운 미래에 필요한 brick을 미리 resident하게 만드는 정책이다.
- **Eviction hysteresis**는 한 번 resident해진 brick을 즉시 내보내지 않고, confidence와 age가 충분히 낮아질 때까지 유지해 `map -> unmap -> map`이 반복되는 **residency thrashing**을 억제하는 정책이다.

Sparse residency를 잘못 설계하면 메모리를 아끼는 대신 다음 비용을 반복해서 지불한다.

`request -> compact -> bind/map -> upload/decompress -> synchronize -> consume -> evict -> request again`

따라서 좋은 residency manager는 단순한 LRU cache가 아니라 **latency-aware, confidence-aware, cost-aware temporal controller**에 가깝다.

개념적 흐름은 다음과 같다.

`GPU Feedback -> Temporal Confidence -> Future-Demand Prediction -> Prefetch Set -> Budget Arbitration -> Resident Set -> Cooling/Hysteresis -> Eviction`

오늘의 핵심은 **"현재 필요한가?"보다 "가까운 미래에 필요할 확률과 다시 가져오는 비용이 얼마인가?"를 기준으로 residency를 결정하는 것**이다.

## 2. 한 줄 핵심
> Sparse residency의 안정성은 단순 LRU가 아니라 **prefetch latency를 포함한 미래 demand 예측 + 느리게 감소하는 confidence + 서로 다른 입·출력 threshold를 쓰는 hysteresis**로 residency churn과 late miss를 동시에 줄이는 데서 나온다.

## 3. 왜 중요한가

### 3.1 Feedback는 항상 한 박자 늦다
GPU feedback은 이미 발생한 access를 기록한다. 하지만 sparse page가 실제 사용 가능해지기까지는 여러 단계가 있다.

- feedback generation
- dedup / compaction
- request processing
- physical tile allocation
- sparse bind / map submission
- optional upload 또는 decompression
- queue synchronization
- 다음 graphics/compute pass에서 consumption

즉 `Frame N`에서 처음 관찰한 demand가 `Frame N+1`이나 `N+2`에서야 실제 residency로 반영될 수 있다.

카메라가 빠르게 움직이거나 volume traversal front가 이동하는 상황에서 **reactive-only residency**는 구조적으로 늦는다. 그래서 prediction은 품질을 위한 부가 기능이 아니라 latency를 숨기기 위한 필수 요소가 된다.

### 3.2 LRU는 카메라 경계에서 쉽게 thrash한다
가장 단순한 eviction 정책은 "오래 사용하지 않은 brick부터 제거"하는 LRU다. 하지만 sparse rendering에서는 사용 여부가 매우 불연속적으로 바뀔 수 있다.

예를 들어 camera frustum 경계에 걸친 brick은 다음처럼 움직일 수 있다.

`visible -> invisible -> visible -> invisible`

단순 LRU가 빠르게 eviction하면 같은 brick을 몇 frame 뒤 다시 map해야 한다. 이때 실제 손실은 단순한 page-table update 비용만이 아니다.

- tile allocation churn
- upload / decompression 반복
- sparse-bind submission 증가
- cache locality 손실
- synchronization 증가
- visible hole 또는 coarse fallback 증가

따라서 **recently resident였다는 사실 자체가 미래 reuse의 evidence**가 된다.

### 3.3 Prediction을 너무 공격적으로 하면 sparse의 의미가 사라진다
반대로 "곧 필요할 것 같다"는 이유로 넓은 영역을 모두 prefetch하면 resident set이 dense volume에 가까워진다.

Prediction에는 항상 precision/recall trade-off가 있다.

- 높은 recall: late miss 감소, VRAM 사용 증가
- 높은 precision: VRAM 효율 증가, miss 위험 증가

그래서 prediction은 단순히 camera velocity를 따라 주변 brick을 몇 겹 더 로드하는 기능이 아니라, **memory budget 안에서 어떤 future demand를 신뢰할지 결정하는 ranking 문제**다.

### 3.4 Simulation workload는 rendering보다 eviction 조건이 더 복잡하다
Rendering은 지금 화면에 보이지 않는 brick을 제거해도 coarse LOD나 fallback으로 품질을 낮출 수 있는 경우가 많다. 하지만 simulation은 그렇지 않을 수 있다.

예를 들어 level-set, SDF, CFD, stencil 기반 field update에는 다음 dependency가 존재할 수 있다.

- narrow band 주변 halo
- ghost cell
- boundary condition region
- 다음 compute step에서 확장될 active front
- 아직 화면에 보이지 않지만 downstream calculation이 읽을 region

따라서 `not visible = evictable`은 simulation/visualization 통합 파이프라인에서 위험하다. Residency policy는 **visual importance와 computational correctness dependency를 분리한 뒤 합쳐야 한다.**

### 3.5 Residency churn은 bandwidth보다 submission/synchronization 병목이 될 수 있다
Sparse binding은 일반 shader load/store처럼 자동으로 발생하는 memory access가 아니다. Vulkan에서는 sparse bind가 `vkQueueBindSparse()`로 제출되는 별도의 queue operation이며, binding 변경과 graphics/compute access의 순서는 semaphore 등으로 명시적으로 관리해야 한다.

즉 page가 너무 자주 들어오고 나가면 병목이 단순 VRAM bandwidth가 아니라 **mapping operation count, host submission, driver work, queue synchronization**으로 이동할 수 있다.

그래서 residency manager는 resident byte만 보지 않고 **bind/unbind churn 자체를 비용으로 모델링해야 한다.**

## 4. 구현 관점

### 4.1 Request history를 binary state가 아닌 temporal signal로 본다
Brick `i`에 대해 단순히 `requested = true/false`만 저장하면 temporal stability를 표현할 수 없다. 대신 최근 request를 confidence로 누적할 수 있다.

개념적으로는 다음과 같은 exponentially decayed signal을 생각할 수 있다.

`C_t(i) = alpha * R_t(i) + (1 - alpha) * C_{t-1}(i)`

여기서

- `R_t(i)`는 현재 frame의 request strength
- `C_t(i)`는 temporal confidence
- `alpha`는 새로운 observation을 얼마나 빠르게 반영할지 결정한다.

하지만 sparse residency에서는 **상승과 하강의 속도를 다르게 두는 asymmetric decay**가 더 자연스러운 경우가 많다.

- 새 request가 들어오면 confidence를 빠르게 올린다.
- request가 사라져도 confidence는 천천히 낮춘다.

이 구조가 곧 hysteresis의 시간축 버전이다.

### 4.2 Prefetch window는 고정 frame 수보다 latency budget으로 보는 편이 좋다
`N frame ahead`를 무조건 prefetch하는 것은 frame time이 변하거나 upload 비용이 달라질 때 의미가 흔들린다.

더 좋은 관점은 다음이다.

`Prefetch Horizon >= Detection Latency + Mapping Latency + Population Latency + Synchronization Margin`

예를 들어 mapping은 빠르지만 brick decompression과 upload가 느리다면 더 긴 horizon이 필요하다. 반대로 data가 이미 tile pool에 있고 page-table remap만 필요한 경우에는 짧은 horizon으로 충분할 수 있다.

즉 **prefetch distance는 공간 거리보다 end-to-end residency latency에서 도출되는 값**으로 보는 것이 좋다.

### 4.3 미래 demand는 여러 predictor를 합치는 문제다
예측 가능한 signal은 workload마다 다르다.

**Camera / rasterization 기반**

- camera linear/angular velocity
- frustum expansion
- projected screen footprint
- near-future occluder 변화
- mip/LOD trajectory

**Ray tracing / volume traversal 기반**

- ray cone / footprint expansion
- 최근 traversal-hit density
- neighboring brick request propagation
- secondary ray direction distribution

**Simulation 기반**

- active narrow-band velocity
- field gradient / front propagation
- dirty-region expansion
- stencil halo requirement
- next-step dependency

결국 predictor는 하나의 정확한 미래 모델이라기보다 **여러 약한 signal을 합친 priority estimator**에 가깝다.

개념적인 priority는 다음처럼 생각할 수 있다.

`Priority = w_v * Visibility + w_p * PredictedDemand + w_t * TemporalConfidence + w_s * SimulationImportance - w_c * ResidencyCost`

여기서 중요한 점은 score 자체보다 **각 항목의 의미를 분리해서 관측 가능하게 유지하는 것**이다.

### 4.4 Eviction hysteresis는 서로 다른 threshold를 사용한다
Oscillation을 줄이는 가장 대표적인 방식은 **admission threshold와 eviction threshold를 다르게 두는 것**이다.

- `T_enter`: 이 값 이상이면 resident 후보가 된다.
- `T_exit`: 이 값 이하이면 eviction 후보가 된다.
- 일반적으로 `T_exit < T_enter`다.

따라서 score가 경계 근처에서 흔들리더라도 바로 state가 뒤집히지 않는다.

개념적인 state machine은 다음과 같다.

`NonResident -> Requested/Prefetch -> Resident -> Cooling -> Evictable`

여기에 simulation correctness를 위한 `Pinned` 상태를 별도로 둘 수도 있다.

**Cooling**은 단순한 기다림이 아니다. "최근까지 중요했고 다시 중요해질 확률이 아직 충분히 높다"는 temporal evidence를 명시적으로 보존하는 상태다.

### 4.5 Minimum residency age와 eviction grace period를 분리한다
두 시간 상수는 비슷해 보이지만 역할이 다르다.

- **Minimum residency age**: 새로 들어온 brick이 최소한 얼마 동안은 쫓겨나지 않도록 보호한다.
- **Eviction grace period**: 마지막 meaningful request 이후 얼마 동안 더 유지할지 결정한다.

이 둘을 분리하면 방금 비싸게 upload한 brick이 budget pressure 때문에 즉시 eviction되는 pathological cycle을 줄일 수 있다.

특히 decompression cost가 큰 volume brick에서는 `residency byte`보다 **reacquisition cost**가 eviction policy에 더 중요한 경우도 있다.

### 4.6 Eviction은 LRU보다 cost-aware ranking으로 본다
실무적인 eviction score는 단순 age 하나보다 다음 요소를 결합하는 편이 유용하다.

- `lastUsedAge`
- temporal confidence
- predicted reuse probability
- physical bytes
- reload / decompression cost
- bind/unbind cost
- fallback availability
- simulation dependency / pin count

예를 들어 같은 64 KiB tile이라도 하나는 backing data가 CPU RAM에 이미 있고 즉시 upload 가능하지만, 다른 하나는 expensive procedural reconstruction이나 decompression이 필요할 수 있다. 이 둘을 동일한 eviction candidate로 취급할 이유는 없다.

즉 **eviction target은 "가장 오래 안 쓴 brick"이 아니라 "evict했을 때 미래 기대 비용이 가장 낮은 brick"**에 가깝다.

### 4.7 Hysteresis는 frame 단위뿐 아니라 공간 단위에도 적용할 수 있다
Camera가 경계를 오갈 때 하나의 brick만 단독으로 residency를 판단하면 checkerboard 형태의 churn이 생길 수 있다.

이를 줄이기 위해 공간적으로 다음을 고려할 수 있다.

- direct request brick은 높은 confidence
- immediate neighbor는 낮은 confidence의 prefetch candidate
- coarse parent brick은 persistent fallback
- simulation halo는 temporary pin

즉 resident set의 경계를 sharp binary frontier로 만들기보다 **confidence가 점진적으로 감소하는 spatial margin**으로 보는 것이다.

이 방식은 prefetch와 hysteresis를 공간축에서도 연결한다.

### 4.8 Metadata는 page table보다 더 자주 읽히므로 hot SoA로 분리한다
Residency metadata가 커지면 실제 volume data보다 metadata traversal이 병목이 될 수 있다.

GPU policy pass가 매 frame 자주 읽는 값은 대략 다음과 같다.

- state
- confidence
- lastRequestedEpoch
- lastResidentEpoch
- physical tile handle
- priority / importance
- pin/dependency flags

이 값들을 큰 AoS struct에 모두 넣으면 eviction scan에서 필요하지 않은 field까지 cache line에 끌려올 수 있다. 그래서 policy hot field를 SoA 또는 compact AoSoA로 분리하는 것이 자연스럽다.

개념적으로는 다음처럼 나눌 수 있다.

**Hot policy metadata**

- confidence
- age/epoch
- state/flags
- priority

**Cold mapping metadata**

- physical allocation handle
- upload/decompression provenance
- backend-specific sparse bind information

최근 다룬 path-state virtualization과 마찬가지로 **frequently scanned control state와 rarely touched payload를 분리하는 data-oriented layout**이 중요하다.

### 4.9 Epoch 기반 timestamp는 wrap-around contract가 필요하다
`lastRequestedFrame`을 32-bit frame index로 두면 사실상 오랜 기간 충분하지만, metadata를 더 줄이기 위해 16-bit epoch를 사용할 수도 있다.

이 경우 단순 integer 비교를 하면 wrap-around 이후 age 계산이 깨질 수 있다. 따라서 age는 modular difference로 정의해야 하고, 최대 유효 history window가 epoch half-range보다 작다는 contract가 필요하다.

이런 작은 detail은 graphics engineer 관점에서 중요하다. Memory packing은 byte 절약으로 끝나는 문제가 아니라 **시간 semantics까지 포함한 ABI contract**이기 때문이다.

### 4.10 Vulkan sparse bind는 residency epoch와 동기화해야 한다
Vulkan에서 sparse binding은 queue operation이다. `vkQueueBindSparse()`는 sparse resource의 physical binding을 바꾸며, 다른 graphics/compute submission이 같은 region을 접근하는 시점과 명시적인 ordering이 필요하다.

Timeline semaphore를 사용하면 residency manager가 다음과 같은 monotonic epoch를 가질 수 있다.

- value `N`: Brick set A의 bind/unbind 완료
- value `N+1`: Upload/initialization 완료
- graphics/compute pass: 필요한 residency epoch 이상을 wait

중요한 점은 **logical residency state를 `Resident`로 바꾸는 시점과 실제 GPU가 안전하게 접근 가능한 시점을 구분하는 것**이다.

즉 metadata에는 필요에 따라 다음을 구분할 수 있다.

- requested
- mapped
- populated
- visible-to-consumer

이 단계들을 하나의 boolean `resident`로 합치면 race와 stale-read bug를 추적하기 어려워진다.

### 4.11 CUDA sparse array도 mapping latency와 tile-pool reuse를 정책에 포함한다
CUDA Driver API의 `cuMemMapArrayAsync()`는 sparse CUDA array/mipmapped array의 subregion을 tile pool backing에 map/unmap할 수 있다. Sparse tile extent와 mip tail property도 API로 조회할 수 있다.

여기서도 residency policy는 API 호출 자체보다 **tile pool reuse와 mapping sequence를 안정적으로 유지하는 것**이 핵심이다.

- 자주 다시 쓰이는 region은 pool에서 오래 유지
- expensive-to-populate tile은 더 높은 retention
- short-lived transient tile은 aggressive eviction
- mapping stream과 consuming compute stream의 dependency를 명확히 관리

즉 Vulkan과 CUDA는 API 모양은 다르지만, temporal residency controller가 풀어야 할 본질적인 문제는 동일하다.

### 4.12 Non-resident access의 fallback contract를 명확히 한다
Prediction은 완벽하지 않으므로 miss는 반드시 발생한다. 따라서 residency architecture는 miss를 예외가 아니라 정상 상태로 취급해야 한다.

가능한 fallback은 다음과 같다.

- coarse mip / parent brick
- lower-resolution dense field
- conservative default material/value
- deferred shading/update
- request를 기록하고 이번 frame에서는 contribution을 제한

특히 Vulkan sparse image는 `residencyNonResidentStrict` 지원 여부에 따라 unbound region의 read 의미가 달라질 수 있으므로, cross-vendor renderer는 "unbound tile을 그냥 읽어도 된다"는 가정을 두지 않는 편이 안전하다.

좋은 system은 **late miss가 발생해도 frame correctness가 붕괴하지 않고 품질만 점진적으로 낮아지는 구조**를 갖는다.

### 4.13 Residency controller의 profiler 지표
Sparse residency는 시각적 결과만 보고 튜닝하기 어렵다. 다음 지표를 함께 봐야 한다.

- **Residency hit rate**: access 시점에 이미 resident였던 비율
- **Late miss rate**: request는 있었지만 consumption 전에 준비되지 못한 비율
- **Prefetch precision**: prefetch한 brick 중 실제 사용된 비율
- **Prefetch recall**: 실제 필요했던 brick 중 미리 준비된 비율
- **Average residency lifetime**
- **Re-residency interval**: eviction 후 다시 요청될 때까지의 시간
- **Churn bytes/frame**
- **Map/unmap operations/frame**
- **Upload/decompression bytes/frame**
- **Fallback usage rate**
- **Budget pressure / oversubscription count**

특히 `high hit rate`만 보면 지나치게 많은 brick을 resident하게 만든 정책도 좋아 보일 수 있다. 그래서 **hit rate + VRAM occupancy + prefetch precision + churn**을 같이 봐야 한다.

### 4.14 C++ architecture에서는 predictor와 backend를 분리한다
C++ 설계 관점에서는 residency policy와 API backend를 분리하는 편이 좋다.

개념적인 책임은 다음처럼 나눌 수 있다.

- **FeedbackCollector**: GPU request/usage signal 수집
- **ResidencyPredictor**: temporal confidence와 future demand 계산
- **ResidencyPolicy**: budget, hysteresis, pin/dependency를 반영한 admission/eviction 결정
- **TilePool**: physical backing allocation 관리
- **SparseBackend**: Vulkan `vkQueueBindSparse()`, CUDA sparse mapping 등 backend-specific 실행
- **ResidencyTelemetry**: miss/churn/prefetch precision 측정

이 구조의 장점은 "prediction 알고리즘"과 "Vulkan/CUDA resource API"를 섞지 않는 것이다. Graphics engineer 실무에서는 이런 분리가 테스트와 cross-API portability를 크게 개선한다.

## 5. 내 관심 분야와 연결
Sparse residency의 temporal controller는 **대규모 3D simulation/visualization field**와 특히 잘 연결된다.

반도체 공정 3D 구조처럼 `x/y` 방향은 넓고 `z` 방향에 매우 얇은 layer가 많이 존재하는 데이터에서는 모든 voxel을 dense하게 유지하는 것이 비효율적일 수 있다. Level-set `phi`, material ID, doping scalar를 brick 또는 sparse volume 형태로 관리한다면 rendering과 emulation이 요구하는 working set도 서로 다르게 움직일 수 있다.

예를 들어 다음과 같은 두 종류의 demand가 동시에 존재할 수 있다.

- **Rendering demand**: camera/cross-section plane 주변, visible surface, heatmap sampling region
- **Simulation demand**: active level-set narrow band, etch/deposition front, stencil halo, material-boundary neighborhood

이때 camera에서 멀어졌다는 이유로 simulation active brick을 eviction하면 compute correctness가 깨질 수 있고, 반대로 simulation active set 전체를 high-resolution rendering residency로 유지하면 VRAM이 낭비된다.

그래서 하나의 `BrickKey`에 대해 최소한 다음 importance를 분리해 보는 것이 유용하다.

- render confidence
- simulation confidence
- dependency/pin state
- projected LOD demand
- reacquisition cost

또한 CUDA compute와 Vulkan visualization이 같은 sparse backing 또는 서로 연결된 tile pool을 사용하는 구조라면 temporal residency policy는 **cross-API ownership transfer와 함께 동작해야 한다.** Brick이 "필요하다"는 정책적 판단, physical tile이 "mapped"됐다는 상태, CUDA가 "population을 끝냈다"는 상태, Vulkan이 "읽어도 된다"는 상태를 분리하면 GPU-stay-GPU 파이프라인의 correctness가 훨씬 명확해진다.

NanoVDB나 octree 같은 **sparse data structure**와 오늘의 **sparse physical residency**를 함께 쓰는 경우도 생각할 수 있다. 전자는 logical empty space를 압축하고, 후자는 실제 GPU memory working set을 제한한다. 두 sparse layer가 겹치면 효율은 커질 수 있지만 metadata indirection과 residency prediction이 복잡해지므로, `logical sparsity`와 `physical residency`를 서로 다른 계층으로 설계하는 것이 중요하다.

## 6. 머릿속에 남길 질문 3개
1. GPU feedback가 `Frame N`의 과거 demand라면, camera motion·ray footprint·simulation front 중 무엇을 이용해 **Frame N+1/N+2의 prefetch horizon**을 가장 안정적으로 예측할 수 있을까?
2. Eviction policy에서 `last used age`와 `reload cost`, `temporal confidence`, `simulation dependency`가 충돌할 때 어떤 기준으로 우선순위를 정해야 thrashing과 VRAM oversubscription을 동시에 줄일 수 있을까?
3. `requested -> mapped -> populated -> consumer-visible`을 하나의 `resident` flag로 압축했을 때 어떤 synchronization bug가 숨어들 수 있으며, 이를 residency epoch/state machine으로 어떻게 분리할 수 있을까?

## 7. graphics engineer 면접 질문 1개와 답변
**질문:** GPU feedback 기반 sparse residency 시스템에서 단순 LRU eviction만 사용하면 왜 문제가 생기며, prefetch와 hysteresis를 어떻게 설계하겠습니까?

**답변:**
단순 LRU는 과거 access recency만 보기 때문에 sparse binding과 upload/decompression latency를 숨기지 못하고, camera frustum 경계나 반복 traversal처럼 demand가 oscillation하는 영역에서 같은 brick을 계속 evict/reload하는 thrashing을 만들 수 있습니다. 따라서 먼저 GPU feedback, camera/traversal motion, simulation activity를 이용해 future demand confidence를 만들고, end-to-end mapping/population latency보다 긴 prefetch horizon을 둡니다.

Eviction에서는 admission threshold와 eviction threshold를 다르게 두는 hysteresis를 사용하고, minimum residency age와 grace period를 두어 score가 경계 근처에서 흔들려도 상태가 즉시 바뀌지 않게 합니다. 또한 LRU age뿐 아니라 reload cost, physical bytes, fallback availability, dependency/pin state를 포함한 cost-aware ranking을 사용하는 것이 좋습니다.

마지막으로 `requested`, `mapped`, `populated`, `consumer-visible` 상태와 residency epoch를 분리해 sparse bind/upload와 graphics/compute consumer 사이의 synchronization을 명확하게 관리해야 합니다. 성능 평가는 hit rate 하나가 아니라 prefetch precision/recall, late miss, churn bytes, map/unmap count, VRAM occupancy를 함께 봐야 합니다.

## 8. 포트폴리오 / 커리어 연결
이 주제를 포트폴리오에서 잘 설명하면 단순히 "sparse texture를 써봤다"보다 훨씬 높은 수준의 GPU systems thinking을 보여줄 수 있다.

좋은 설명 포인트는 다음과 같다.

- **Problem framing**: sparse resource API가 아니라 working-set prediction 문제로 정의
- **Temporal model**: reactive feedback의 latency를 prefetch horizon과 confidence decay로 보완
- **Stability**: dual threshold, grace period, minimum residency age로 thrashing 억제
- **Memory architecture**: hot policy metadata와 cold mapping payload 분리
- **Synchronization**: logical residency와 GPU-consumer visibility를 epoch로 구분
- **Cross-API design**: policy와 Vulkan/CUDA backend 분리
- **Profiling**: hit rate뿐 아니라 prefetch precision, late miss, churn, mapping operations, VRAM budget까지 측정

Graphics engineer 면접에서는 이런 설명을 통해 다음 역량을 동시에 보여줄 수 있다.

1. GPU memory hierarchy와 virtual/sparse residency 이해
2. rendering/compute pipeline latency 이해
3. temporal prediction과 cache policy trade-off 분석
4. C++ data-oriented architecture 설계
5. Vulkan/CUDA synchronization contract 이해
6. profiler metric을 시스템 설계와 연결하는 능력

특히 game engine의 virtual texturing/geometry streaming, scientific visualization의 out-of-core volume, GPU simulation의 adaptive working set은 구현 형태가 달라도 **future demand를 예측하고 memory pressure 아래에서 안정적으로 residency를 유지한다**는 동일한 시스템 문제로 연결된다.

## 9. 내일 이어서 볼 개념
**Residency-Aware Multi-Resolution Fallback: Mip Tails, Coarse Bricks, and Error-Bounded Rendering Under Memory Pressure**

오늘은 "필요한 brick을 미리 가져오고, 너무 빨리 버리지 않는 방법"을 봤다. 하지만 prediction은 항상 실패할 수 있고 VRAM budget도 항상 충분하지는 않다.

내일은 다음 질문으로 이어간다.

> **High-resolution brick이 아직 resident하지 않거나 eviction됐다면, frame을 깨뜨리지 않고 어떤 lower-resolution representation으로 대체할 것인가?**

이를 통해 mip tail, parent/coarse brick, hierarchical residency, LOD fallback, error metric, seamless transition, simulation-vs-rendering accuracy contract를 연결한다.

## 10. 참고 키워드
- Temporal Residency Prediction
- Sparse Residency
- Sparse Resource
- Working-Set Prediction
- Prefetch Horizon / Prefetch Window
- Residency Latency
- Eviction Hysteresis
- Dual Threshold
- Minimum Residency Age
- Eviction Grace Period
- Confidence Decay / Exponential Moving Average
- Asymmetric Decay
- LRU / Cost-Aware Eviction
- Residency Thrashing / Churn
- Re-residency Interval
- Prefetch Precision / Recall
- Late Miss Rate
- Tile Pool
- Sparse Binding
- `vkQueueBindSparse()`
- `VK_QUEUE_SPARSE_BINDING_BIT`
- Vulkan Timeline Semaphore
- `residencyNonResidentStrict`
- CUDA Sparse Array
- `cuMemMapArrayAsync()`
- `CUDA_ARRAY_SPARSE_PROPERTIES`
- Mip Tail
- GPU Feedback
- Sampler Feedback
- Brick Streaming
- Volume Rendering
- Level Set / SDF Narrow Band
- Ghost Cell / Halo Region
- Data-Oriented Design
- SoA / AoSoA
- Residency Epoch
- Resource State Machine
- Cross-API GPU Synchronization
- Vulkan Sparse Resources specification
- NVIDIA CUDA Driver API Virtual Memory Management