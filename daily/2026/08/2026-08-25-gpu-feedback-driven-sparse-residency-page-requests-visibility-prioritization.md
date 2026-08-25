---
title: "GPU Feedback-Driven Sparse Residency: Residency Maps, Page Requests, and Visibility-Guided Brick Prioritization"
date: "2026-08-25"
category: Graphics
tags: [GPU, Sparse Residency, GPU Feedback, Page Request, Brick Streaming, Vulkan, CUDA, Direct3D 12, Sampler Feedback, Compute Shader, Memory Layout, Compaction, Working Set, C++]
level: intermediate
---

# [Daily Graphics Growth] 2026-08-25 - GPU Feedback-Driven Sparse Residency: Residency Maps, Page Requests, and Visibility-Guided Brick Prioritization

## 1. 오늘의 개념
어제는 큰 3D virtual volume을 유지하면서 실제 VRAM에는 필요한 brick/tile만 매핑하는 **Sparse 3D Volume Residency**를 봤다. 오늘은 그 다음 질문으로 넘어간다.

> **어떤 brick이 실제로 필요한지 CPU가 미리 추측하는 대신, GPU가 실제 접근 패턴을 feedback으로 남기게 하면 무엇이 달라지는가?**

Sparse residency의 어려움은 physical page를 매핑하는 API 자체보다, 앞으로 필요한 working set을 얼마나 정확하고 안정적으로 예측하느냐에 있다. Camera frustum만 보고 brick을 고르면 occlusion, ray traversal, screen-space footprint, LOD, simulation dependency를 놓칠 수 있다. 반대로 너무 보수적으로 prefetch하면 sparse resource의 장점인 VRAM 절약이 줄어든다.

**GPU feedback-driven residency**는 rasterization, ray/volume traversal, compute simulation이 실제로 접근하거나 접근하려 했던 logical region을 GPU에서 기록하고, 이를 다음 residency decision의 입력으로 사용하는 구조다.

개념적 흐름은 다음과 같다.

`Shader/Traversal -> Feedback Map -> Dedup/Compaction -> Request Queue -> Priority/Budget -> Sparse Bind/Map -> Resident Working Set`

여기서 중요한 구분은 두 가지다.

1. **Feedback generation**은 GPU가 실제 workload를 관찰해 demand signal을 만드는 단계다.
2. **Sparse binding**은 Vulkan `vkQueueBindSparse()`, CUDA sparse-array mapping, D3D12 tiled-resource mapping처럼 API가 physical backing을 바꾸는 단계다.

즉 **GPU가 request를 만든다고 해서 page mapping까지 완전히 device-side로 수행된다는 뜻은 아니다.** 대부분의 graphics API에서는 feedback을 GPU가 만들고, compact한 request를 application/runtime이 해석한 뒤 host-side API submission으로 sparse mapping을 변경하는 구조가 현실적이다.

오늘의 핵심은 sparse volume을 단순한 memory allocator 문제가 아니라 **GPU 관측 기반 working-set control loop**로 보는 것이다.

## 2. 한 줄 핵심
> GPU feedback-driven sparse residency는 **실제 shader/traversal의 demand를 compact한 page request로 변환하고, visibility·LOD·simulation importance·budget을 결합해 다음 frame의 resident working set을 결정하는 폐루프(closed-loop) 메모리 시스템**이다.

## 3. 왜 중요한가
### 3.1 CPU visibility는 실제 GPU access와 다를 수 있다
CPU는 object bounds, frustum, distance를 기반으로 필요한 region을 예측할 수 있다. 하지만 GPU가 실제로 읽는 데이터는 훨씬 복잡하다.

- occlusion 뒤의 object는 rasterization에서 거의 기여하지 않을 수 있다.
- ray tracing은 화면 밖 geometry나 volume을 secondary ray가 요구할 수 있다.
- anisotropic/filtering footprint는 단순한 pixel 중심보다 넓은 texel region을 요구한다.
- volume ray marching은 front-to-back early termination 때문에 camera-facing brick만 실제 접근될 수 있다.
- simulation은 화면에는 보이지 않아도 stencil/halo dependency 때문에 이웃 brick을 필요로 할 수 있다.

따라서 CPU prediction만 사용하면 **over-residency** 또는 **under-residency**가 쉽게 발생한다. GPU feedback은 “실제로 필요했던 것”을 측정하기 때문에 working-set controller에 더 직접적인 신호를 준다.

### 3.2 Demand signal은 Boolean보다 풍부할수록 좋다
가장 단순한 feedback은 `requested/not requested` 한 bit다. 하지만 residency budget이 제한되면 모든 request를 만족시킬 수 없기 때문에 importance가 필요하다.

GPU는 request와 함께 다음과 같은 통계적 signal을 축적할 수 있다.

- request count
- minimum requested mip / maximum detail demand
- projected screen coverage
- nearest visible depth
- ray hit count
- traversal frequency
- simulation activity 또는 dirty state
- temporal persistence

이 정보가 있으면 “요청됐다”에서 끝나지 않고 **어떤 brick을 먼저 resident하게 만들 것인가**를 결정할 수 있다.

### 3.3 Feedback latency를 숨기지 못하면 sparse hole이 보인다
GPU feedback은 대개 현재 frame의 관측이다. 실제 sparse bind/map과 data population이 완료되는 시점은 다음 frame 또는 그 이후일 수 있다.

즉 시스템에는 자연스럽게 latency가 존재한다.

`Frame N demand -> Frame N feedback -> N/N+1 processing -> mapping/upload -> N+1 or N+2 consumption`

그래서 feedback-driven streaming은 미래를 정확히 아는 시스템이 아니라 **늦게 도착하는 evidence를 이용하는 시스템**이다. Coarse mip fallback, conservative neighborhood prefetch, temporal hysteresis가 중요한 이유가 여기 있다.

### 3.4 Feedback 처리 비용도 GPU bandwidth를 소비한다
Request map을 너무 세밀하게 만들면 feedback 자체가 bandwidth/atomic bottleneck이 될 수 있다. 반대로 너무 거칠면 unnecessary residency가 증가한다.

즉 sparse residency granularity와 feedback granularity는 함께 설계해야 한다.

- 작은 feedback cell: 높은 precision, 큰 metadata/atomic 비용
- 큰 feedback cell: 낮은 overhead, 높은 over-fetch 가능성

이 trade-off는 texture virtual memory, voxel/volume streaming, geometry page streaming 모두에서 반복된다.

## 4. 구현 관점
### 4.1 Feedback 표현: Bitset, Append Queue, Counter Map
GPU feedback의 대표적인 표현은 세 가지로 나눌 수 있다.

**Bitset / Residency Request Map**

Logical brick 하나당 1 bit를 둔다. 여러 shader invocation이 같은 brick을 요구하면 atomic OR로 같은 bit에 합쳐진다.

장점은 매우 compact하고 dedup이 즉시 된다는 점이다. 반면 인접 brick이 동일한 32/64-bit word에 몰리면 atomic contention이 생길 수 있고, request importance를 표현하기 어렵다.

**Append Request Queue**

각 invocation 또는 workgroup이 `BrickKey`를 append한다. 초기 기록은 단순하지만 동일 brick이 수천 번 들어갈 수 있어 뒤에서 sort/dedup이 필요하다. Request frequency 자체를 importance signal로 사용할 수 있다는 장점도 있다.

**Counter / Metadata Map**

Brick별 counter, minimum mip, nearest depth 같은 값을 atomic reduction으로 유지한다. 정보량은 많지만 metadata memory와 contention 비용이 커진다.

실무에서는 한 가지 방식만 쓰기보다 **coarse bitset + compact metadata**, 또는 **workgroup-local aggregation + global map**처럼 계층적으로 구성하는 경우가 자연스럽다.

### 4.2 `BrickKey`는 sparse mapping의 stable identity다
Feedback entry가 physical tile index를 직접 기록하면 compaction/eviction 이후 의미가 깨질 수 있다. Request는 physical backing이 아니라 **logical brick identity**를 가리키는 편이 안전하다.

예를 들면 다음 정보가 stable key가 될 수 있다.

- mip 또는 LOD
- brick coordinate `(bx, by, bz)`
- volume/field ID
- optional generation/version

개념적으로는 다음과 같은 mapping이 유지된다.

`BrickKey -> ResidencyTable -> PhysicalTileHandle`

이 구조는 최근 다룬 generational handle, logical identity와 physical slot 분리 원칙이 sparse volume에서도 반복되는 모습이다.

### 4.3 GPU compaction은 feedback을 “결정 가능한 크기”로 바꾼다
Full-resolution request map 전체를 CPU로 readback하면 feedback-driven system의 장점이 크게 줄어든다. 따라서 GPU에서 먼저 sparse request만 뽑아 **dense request list**로 압축하는 것이 중요하다.

개념적으로는 다음 단계가 있다.

- bit/counter map에서 active entry 식별
- prefix scan 또는 hierarchical scan
- compact request array 생성
- optional priority key 생성
- 필요하면 priority/brick locality 기준 정렬

중요한 점은 compaction의 목적이 단순히 transfer bytes를 줄이는 것이 아니라, residency manager가 처리할 work를 **active set에 비례하도록 만드는 것**이다.

### 4.4 Feedback buffer의 memory layout은 atomics와 scan 모두를 고려한다
Feedback 기록 단계와 compaction 단계는 선호하는 layout이 다르다.

- 기록 단계: atomic-friendly packed words, 낮은 random-write footprint
- scan 단계: sequential/coalesced access
- priority 단계: field별 SoA가 유리할 수 있음
- CPU readback 단계: compact AoS record가 편리할 수 있음

예를 들어 GPU 내부 request state는 `requested bits`, `importance`, `lastRequestedEpoch`를 별도 SoA buffer로 두고, compact 결과만 `BrickKey + priority + flags`의 고정 크기 record로 만드는 식의 분리가 자연스럽다.

이때 feedback map 전체를 매 frame clear하는 비용도 무시할 수 없다. Frame/epoch tagging을 사용하면 매 entry를 0으로 지우는 대신 “이 값이 현재 epoch에 속하는가”로 freshness를 구분할 수 있지만, epoch 폭과 wrap-around contract가 필요해진다.

### 4.5 Visibility-Guided Priority는 하나의 score가 아니라 policy다
Resident budget이 `B`개의 brick이라면 request 수가 `B`보다 많은 순간 selection policy가 필요하다.

개념적으로 priority는 다음 요소의 조합으로 볼 수 있다.

`Priority = Visibility + DetailDemand + TemporalPersistence + SimulationImportance - StreamingCost`

여기서 각 항목의 의미는 다르다.

- **Visibility**: 실제 화면 기여 또는 ray contribution 가능성
- **DetailDemand**: 현재 projected footprint에서 필요한 mip/LOD
- **TemporalPersistence**: 여러 frame 연속 요청되는 안정적인 region인지
- **SimulationImportance**: active narrow band, stencil halo, dirty region처럼 계산상 필요한지
- **StreamingCost**: 새로 population해야 하는 byte 수나 decompression 비용

중요한 점은 camera visibility만 높게 두면 simulation correctness에 필요한 off-screen brick을 eviction할 수 있다는 것이다. Rendering priority와 simulation dependency는 별도의 signal로 유지한 뒤 policy에서 결합하는 편이 더 명확하다.

### 4.6 Vulkan에서는 sparse residency 검출과 feedback 기록을 분리해서 본다
Vulkan의 `shaderResourceResidency` feature가 활성화되면 SPIR-V `OpImageSparse*` 계열 image operation이 residency code를 반환할 수 있고, `OpImageSparseTexelsResident`를 통해 sample이 non-resident texel을 포함했는지 알 수 있다.

하지만 이것은 **자동 residency request map 시스템**과 동일하지 않다. Vulkan은 sample이 resident였는지에 대한 shader-visible information을 제공하지만, 어떤 logical brick을 다음 frame에 매핑할지는 application이 별도의 buffer/image feedback 구조로 기록해야 한다.

3D sparse image 자체는 `sparseResidencyImage3D` capability에 의존하므로, volume feedback pipeline은 다음 두 capability를 별도로 본다.

- resource가 partially resident 3D image가 될 수 있는가?
- shader가 sparse access의 residency status를 관측할 수 있는가?

이 둘을 분리해서 보는 것이 cross-vendor Vulkan architecture에서 중요하다.

### 4.7 D3D12 Sampler Feedback은 hardware-assisted feedback의 좋은 기준점이다
Direct3D 12의 **Sampler Feedback**은 texture sampling이 어떤 mip region을 요구했는지를 GPU가 feedback map에 기록하도록 설계된 기능이다.

대표적인 두 표현은 다음과 같다.

- **MinMip feedback**: region별 가장 detailed하게 요청된 mip
- **MipRegionUsed feedback**: mip별 region이 사용됐는지 boolean 형태로 기록

Reserved/tiled texture streaming과 결합하면 “어떤 tile/mip이 실제 sample footprint에서 요구됐는가”를 CPU 추측보다 직접적으로 얻을 수 있다.

다만 현재 Sampler Feedback 모델은 주로 2D texture streaming을 위한 기능이다. 따라서 3D scientific volume에서 그대로 사용할 수 있는 범용 page-fault 시스템으로 이해하면 안 된다. 오늘의 관점에서는 **hardware sampler가 access footprint를 feedback으로 노출하는 대표 사례**로 보는 것이 적절하다.

### 4.8 CUDA compute에서는 access 자체가 request signal이 될 수 있다
CUDA sparse array에는 D3D12 Sampler Feedback과 같은 자동 feedback map abstraction이 없다. 대신 compute kernel이나 volume traversal은 어떤 logical brick coordinate를 접근하려는지 application-defined addressing 과정에서 이미 알고 있는 경우가 많다.

따라서 simulation/compute workload에서는 request bitset이나 counter map을 명시적으로 기록하기 쉽다. 중요한 점은 CUDA sparse-array mapping 자체와 feedback collection을 동일한 기능으로 보지 않는 것이다.

- feedback: 어떤 logical page가 필요했는가
- mapping: 어떤 physical tile이 그 logical page를 backing하는가

이 분리는 Vulkan/CUDA cross-API residency manager를 설계할 때 특히 중요하다.

### 4.9 Multi-frame pipeline은 feedback, mapping, consumption의 epoch를 분리한다
Sparse streaming에서 흔한 버그는 “request가 존재한다”와 “data가 읽을 준비가 됐다”를 동일시하는 것이다.

Residency metadata에는 최소한 다음 시점이 분리되어야 한다.

- requested epoch
- allocated/mapped epoch
- populated/written epoch
- visible-to-consumer epoch
- last-used epoch

Frame N에서 request된 brick이 N+1에 physical tile을 받더라도 upload/decompression/compute population이 끝나지 않았다면 consumer가 읽어서는 안 된다. 그래서 timeline semaphore/fence와 residency state machine이 연결된다.

또한 GPU feedback buffer 자체도 double/triple buffering 또는 epoch partitioning이 필요하다. Frame N shader가 feedback을 쓰는 동안 CPU가 같은 memory를 readback하면 ownership hazard가 생길 수 있기 때문이다.

### 4.10 GPU feedback의 성능 평가는 hit rate보다 “wasted residency”와 “fault latency”를 본다
단순 resident hit rate만 높이면 모든 것을 resident하게 만드는 정책이 이긴다. 좋은 sparse residency metric은 memory budget을 함께 본다.

실무적으로 의미 있는 지표는 다음과 같다.

- requested brick 수 / unique requested brick 수
- resident-hit ratio
- non-resident access count
- request-to-ready latency
- prefetch hit ratio
- unused resident bytes
- eviction 후 짧은 시간 안에 재요청된 brick 수
- mapping/upload bandwidth
- feedback recording/compaction GPU time
- priority queue depth와 dropped request 수

특히 **evict 후 즉시 다시 request되는 패턴**은 thrashing을 의미한다. 이것이 내일 다룰 temporal prediction과 hysteresis로 이어진다.

## 5. 내 관심 분야와 연결
### Real-time rendering
Virtual texturing, virtual shadow maps, sparse radiance cache, voxel GI, large-world terrain은 모두 “전체 virtual address space는 크지만 현재 화면에 필요한 physical working set은 작다”는 공통 구조를 가진다. GPU feedback은 camera/frustum 기반 heuristic보다 실제 sample footprint를 직접 반영할 수 있다는 점에서 강력하다.

### Ray tracing
Primary visibility만으로 residency를 결정하면 secondary ray가 요구하는 geometry/volume/texture page를 놓칠 수 있다. Ray hit count, path depth, contribution estimate 같은 signal을 feedback priority에 포함하면 raster-only visibility와 다른 working set을 만들 수 있다.

### Simulation / visualization
3D scalar/vector field, SDF, level-set, density volume에서는 화면에 보이는 brick과 계산상 필요한 brick이 다를 수 있다. Rendering feedback과 simulation dependency를 별도 channel로 유지하면 **visible working set**과 **computational working set**을 하나의 residency budget에서 조정할 수 있다.

### C++ engine architecture
C++ 측에서는 `FeedbackCollector`, `RequestCompactor`, `ResidencyPolicy`, `SparseBackend`, `TilePool`을 서로 분리하면 API-specific mapping과 policy를 분리할 수 있다. Vulkan/CUDA/D3D12 backend는 달라도 `BrickKey`, epoch, priority, residency state 같은 상위 contract는 공유할 수 있다.

## 6. 머릿속에 남길 질문 3개
1. **GPU feedback이 Frame N의 과거 demand라면, Frame N+1의 필요한 working set을 안정적으로 예측하기 위해 어떤 temporal signal을 추가해야 하는가?**
2. **Bitset, append queue, counter map 중 어떤 feedback representation이 request precision·atomic contention·compaction cost 사이에서 가장 좋은 균형을 만드는가?**
3. **Rendering visibility와 simulation dependency가 충돌할 때 한정된 VRAM budget을 어떤 policy로 배분해야 correctness와 visual quality를 함께 유지할 수 있는가?**

## 7. graphics engineer 면접 질문 1개와 답변
### 질문
**Sparse resource streaming에서 GPU feedback을 사용하면 CPU visibility 기반 streaming보다 항상 더 정확하고 빠른가?**

### 답변
아니다. GPU feedback은 실제 shader/traversal demand를 관측하기 때문에 visibility accuracy는 높일 수 있지만, **feedback latency와 processing overhead**가 추가된다. Current frame에서 생성한 request가 sparse mapping과 data population을 거쳐 usable해지기까지 한두 frame 이상 걸릴 수 있고, feedback map 기록은 atomic traffic과 bandwidth를 사용한다.

따라서 좋은 시스템은 GPU feedback만 맹목적으로 따르지 않는다. Coarse mip나 fallback representation을 유지하고, camera velocity나 neighborhood 기반 prefetch를 이용해 latency를 숨기며, temporal persistence와 hysteresis로 eviction thrashing을 줄인다. 또한 full feedback map을 CPU로 읽는 대신 GPU에서 dedup/compaction한 small request set만 policy layer로 넘기는 것이 일반적으로 더 scalable하다.

면접에서는 특히 **GPU feedback generation과 sparse binding은 다른 단계이며, GPU-generated request가 곧 device-side page mapping을 의미하지 않는다**는 점을 설명하면 architecture 이해도를 보여줄 수 있다.

## 8. 포트폴리오 / 커리어 연결
이 주제는 단순한 rendering effect보다 **GPU memory system과 execution pipeline을 함께 설계하는 능력**을 보여주기 좋다.

포트폴리오에서 설명 가치가 높은 관점은 다음과 같다.

- virtual address space와 physical residency 분리
- shader-generated feedback의 memory layout
- atomic contention과 request dedup/compaction
- frame-to-frame feedback latency
- VRAM budget 기반 priority policy
- Vulkan sparse residency와 shader resource residency의 차이
- D3D12 Sampler Feedback과 generic sparse streaming의 차이
- CUDA compute feedback과 sparse-array mapping 분리
- render graph에서 feedback -> compaction -> readback/policy -> sparse bind -> consumer의 dependency 표현
- profiler에서 request rate, feedback cost, page churn, wasted residency를 보는 방법

Graphics engineer 실무에서는 "몇 FPS가 나왔다"보다 **왜 이 metadata layout을 골랐고, 어떤 contention/latency를 줄였으며, sparse budget이 변할 때 quality가 어떻게 degrade되는가**를 설명할 수 있는지가 중요하다.

면접 관점에서도 이 개념은 shader, compute, GPU virtual memory, synchronization, data-oriented C++, streaming architecture를 한 번에 연결한다. 특히 Vulkan과 D3D12의 sparse/feedback model 차이를 구분해 설명하면 API 암기보다 abstraction 이해를 보여줄 수 있다.

## 9. 내일 이어서 볼 개념
**Temporal Residency Prediction and Eviction Hysteresis: Prefetch Windows, Confidence Decay, and Thrash Control**

오늘의 GPU feedback은 실제 demand를 관측하지만 본질적으로 과거 정보다. 다음에는 camera/ray/simulation motion을 이용한 **prefetch window**, 여러 frame의 request persistence를 이용한 **residency confidence**, recently evicted page의 재요청을 줄이는 **hysteresis/cooldown**, VRAM pressure가 높을 때 quality를 점진적으로 낮추는 **budget-aware degradation**을 연결한다.

핵심 질문은 다음과 같다.

> **늦게 도착하는 feedback을 어떻게 미래 working-set prediction으로 바꾸면서, 과도한 prefetch와 eviction thrashing을 동시에 억제할 것인가?**

## 10. 참고 키워드
- GPU Feedback-Driven Streaming
- Sparse Residency / Partially Resident Resource
- Residency Map / Request Map
- Page Request / Brick Request
- Working Set
- GPU Bitset / Atomic OR
- Append Buffer / Request Queue
- Prefix Sum / Stream Compaction
- Request Deduplication
- Visibility-Guided Priority
- Temporal Persistence / Residency Confidence
- Vulkan `shaderResourceResidency`
- SPIR-V `OpImageSparse*`
- SPIR-V `OpImageSparseTexelsResident`
- Vulkan `sparseResidencyImage3D`
- Vulkan `vkQueueBindSparse()`
- D3D12 Sampler Feedback
- `SAMPLER_FEEDBACK_MIN_MIP`
- `SAMPLER_FEEDBACK_MIP_REGION_USED`
- Reserved Resource / Tiled Resource
- CUDA Sparse Array
- `cuMemMapArrayAsync()`
- Tile Pool / Physical Page Pool
- Frame Epoch / Timeline Semaphore
- Prefetch / Hysteresis / Thrashing
- GPU Memory Budget
- C++ Render Graph
