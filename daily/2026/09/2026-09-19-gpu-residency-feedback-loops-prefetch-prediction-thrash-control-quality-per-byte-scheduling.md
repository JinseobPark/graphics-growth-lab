---
title: "GPU Residency Feedback Loops: Prefetch Prediction, Thrash Control, and Quality-Per-Byte Scheduling"
date: "2026-09-19"
category: Graphics
tags: [GPU, Rendering, GPU-Driven Rendering, Residency, Geometry Streaming, Prefetch, Working Set, Thrashing, Quality Per Byte, Memory Budget, Vulkan, DirectX 12, CUDA, Meshlet, Dynamic Geometry, Memory Layout, C++]
level: intermediate
---

# [Daily Graphics Growth] 2026-09-19 - GPU Residency Feedback Loops: Prefetch Prediction, Thrash Control, and Quality-Per-Byte Scheduling

## 1. 오늘의 개념

어제는 **GPU Geometry Residency and Streaming**에서 logical geometry와 physical residency를 분리하고, working set·fallback LOD·prefetch·eviction·in-flight pinning을 이용해 GPU-local memory보다 큰 scene을 다루는 구조를 봤다.

오늘은 residency manager를 단순한 cache가 아니라 **closed-loop controller**로 본다.

기본 흐름은 다음과 같다.

```text
Visibility / Camera / Simulation Signals
            ↓
      Demand Prediction
            ↓
         Prefetch
            ↓
      Resident Working Set
            ↓
          Render
            ↓
 Hit / Miss / Fallback / Stall Feedback
            ↓
 Streaming Budget / Eviction Policy Update
            ↺
```

즉 residency manager는 매 frame 같은 고정 규칙을 적용하는 것이 아니라 다음 질문에 반복적으로 답한다.

- Prefetch가 너무 늦어 miss가 생기는가?
- Prefetch가 너무 공격적이라 사용하지 않는 page를 많이 올리는가?
- Eviction이 너무 빠르기 때문에 몇 frame 뒤 같은 page를 다시 읽는가?
- Streaming budget이 너무 커서 rendering bandwidth를 방해하는가?
- Memory budget이 줄었을 때 quality degradation을 어떻게 최소화할 것인가?

오늘의 핵심은 **quality-per-byte**다.

GPU memory가 1 GB 부족하다고 하자.

모든 1 GB의 geometry가 같은 가치가 있는 것은 아니다.

```text
1 GB of near-camera fine geometry
!=
1 GB of barely visible distant detail
!=
1 GB of derived geometry that can be cheaply remeshed
```

따라서 residency scheduler의 질문은:

> **“무엇을 가장 오래 안 썼는가?”보다 “이 byte를 resident하게 유지했을 때 앞으로 얻는 visual/compute benefit이 얼마인가?”**

에 가깝다.

---

## 2. 한 줄 핵심

> GPU residency feedback loop의 핵심은 **prefetch hit/miss, reload latency, eviction regret, memory pressure를 지속적으로 측정하고, geometry의 visual importance와 reload cost를 resident byte로 나눈 quality-per-byte 관점으로 streaming·eviction aggressiveness를 조절하는 것**이다.

---

## 3. 왜 중요한가

Residency policy를 고정 heuristic으로 만들면 특정 workload에서만 잘 동작하기 쉽다.

예:

```text
Evict if invisible for 30 frames.
Prefetch if inside expanded frustum.
Upload 256 MB per frame.
```

이 값들은 camera speed, GPU memory pressure, PCIe bandwidth, scene topology에 따라 최적점이 크게 달라진다.

### 3.1 Working set은 움직인다

Camera가 천천히 움직일 때:

```text
future demand ≈ current demand
```

이므로 작은 prefetch horizon으로 충분하다.

빠른 orbit/camera cut에서는:

```text
future demand far from current demand
```

이므로 같은 horizon이면 miss가 급증한다.

### 3.2 Budget도 움직인다

현재 Vulkan specification의 `VK_EXT_memory_budget`은 `heapBudget`과 `heapUsage`를 **동적 추정치**로 정의한다. 다른 process나 OS activity 때문에 budget은 실행 중 변할 수 있다.

Direct3D 12/DXGI도 `QueryVideoMemoryInfo`의 `Budget`과 `CurrentUsage`를 제공하며, 문서는 application이 budget을 넘으면 OS background memory management 때문에 stutter 또는 performance penalty가 생길 수 있음을 설명한다.

즉 residency controller는 scene만 보는 것이 아니라 **system memory pressure**도 feedback signal로 받아야 한다.

### 3.3 Thrashing은 단순 miss보다 위험하다

Miss 한 번은 unavoidable할 수 있다.

하지만 같은 page를 반복해서:

```text
load → evict → load → evict
```

한다면 policy 자체가 틀린 것이다.

Thrash는:

- transfer bandwidth
- sparse bind / allocation work
- snapshot patch
- CPU/GPU synchronization
- pop-in/fallback time

을 동시에 낭비한다.

그래서 단순 hit ratio보다 **eviction regret**를 측정해야 한다.

---

## 4. 구현 관점

### 4.1 Residency Manager를 Controller로 본다

입력:

```text
Memory Budget
Camera Motion
Visibility
LOD Demand
Simulation Front
Upload Latency
Frame Time
```

상태:

```text
Resident Pages
Pending Requests
Cold Pages
Pinned Pages
Streaming Backlog
```

출력:

```text
Prefetch Set
Upload Budget
Eviction Set
Resident LOD Target
```

Feedback:

```text
Hit
Miss
Fallback
Stall
Reload-after-eviction
Unused-prefetch
```

이 구조를 명확히 하면 heuristic tuning이 “감”이 아니라 measured control problem이 된다.

### 4.2 Prefetch Hit Ratio

Prefetch request가 실제 demand 전에 완료됐는지를 측정한다.

개념적으로:

```text
Prefetch Hit =
needed page was resident
because it had been prefetched
```

Metric:

```text
prefetchHits / futureDemandedPages
```

Hit ratio가 낮은 원인:

- horizon이 짧음
- upload latency 증가
- priority ranking 오류
- camera prediction 오류

하지만 hit ratio만 높이면 안 된다.

모든 page를 미리 load하면 hit ratio는 높지만 memory/bandwidth를 낭비한다.

### 4.3 Prefetch Precision

그래서 precision도 함께 본다.

```text
Prefetch Precision =
prefetched pages actually used soon
-------------------------------
all prefetched pages
```

좋은 controller는:

```text
Recall: 필요한 page를 놓치지 않는다
Precision: 쓸 page만 가져온다
```

를 동시에 맞춘다.

이것은 머신러닝 분류와 비슷한 trade-off다.

### 4.4 Prediction Horizon

Prefetch horizon은 시간 또는 거리로 표현할 수 있다.

예:

```text
predictionTime = 150 ms
```

또는:

```text
cameraDistanceAhead = speed × horizon
```

Streaming latency가 100 ms인데 horizon이 20 ms면 늦다.

반대로 camera가 느린데 horizon이 2초면 많은 irrelevant page를 load한다.

따라서 개념적으로:

```text
Prefetch Horizon
≈
Observed Streaming Latency
+
Safety Margin
```

에서 시작할 수 있다.

### 4.5 Latency-Aware Horizon

Streaming latency 자체도 고정이 아니다.

```text
host cache hit  → low latency
disk/file load  → high latency
GPU remesh      → compute-dependent
```

따라서 resource마다:

```text
estimatedReadyTime
```

이 다를 수 있다.

Prefetch priority에 다음을 포함할 수 있다.

```text
timeToNeed
-
estimatedLoadTime
```

값이 작거나 음수에 가까우면 urgent다.

### 4.6 Time-to-Need

Geometry page가 언제 필요할지 rough estimate를 만든다.

예:

```text
distance to predicted frustum
camera velocity
angular velocity
slice-plane velocity
simulation front velocity
```

를 이용한다.

Priority mental model:

```text
Urgency
≈
1 / max(epsilon, timeToNeed - loadLatency)
```

정확한 물리식이 아니라 ranking signal이다.

### 4.7 Camera Prediction Error를 측정한다

예측 camera:

```text
predictedPose(t + Δ)
```

실제:

```text
actualPose(t + Δ)
```

차이를 측정할 수 있다.

Prediction error가 커지면:

- horizon 축소
- spatial margin 확대
- expensive fine LOD prefetch 억제

등으로 policy를 바꿀 수 있다.

Camera cut에서는 history 기반 prediction을 reset한다.

### 4.8 Fast Motion과 Slow Motion에서 Strategy가 다르다

#### Slow Camera

- high-confidence spatial prefetch
- fine LOD prefetch 가능
- cold eviction relatively aggressive

#### Fast Camera

- coarse LOD 넓게 유지
- fine LOD prefetch conservative
- resident base representation 중요

빠른 이동에서 fine detail을 넓게 prefetch하면 bandwidth가 폭발한다.

### 4.9 Quality-per-Byte

가장 중요한 scheduling score다.

개념적으로:

```text
QualityPerByte
=
Expected Benefit
----------------
Resident Bytes
```

Expected benefit은:

- projected screen area
- screen-space error reduction
- shading/geometry quality
- selection importance
- user focus
- fallback penalty

등으로 구성할 수 있다.

같은 32 MB라도:

```text
near high-detail geometry
```

가:

```text
far subpixel detail
```

보다 높은 priority를 가진다.

### 4.10 Benefit는 반드시 Visual Quality만은 아니다

Scientific visualization에서는 benefit이 다음일 수도 있다.

- selected region accuracy
- cross-section topology fidelity
- measurement interaction
- picking precision
- scalar overlay quality

즉 Quality는 application-specific utility다.

```text
UtilityPerByte
```

라고 부르는 것이 더 일반적일 수 있다.

### 4.11 Reload Cost를 Score에 포함한다

쉽게 다시 만들 수 있는 geometry는 eviction cost가 낮다.

예:

```text
derived mesh from resident SDF
→ cheap recompute

compressed asset page on disk
→ expensive I/O
```

Retention score:

```text
KeepScore
≈
Future Utility
× Reload Cost
----------------
Resident Bytes
```

Reload cost가 높을수록 같은 utility라도 오래 유지할 가치가 커진다.

### 4.12 Transfer vs Recompute를 Runtime Feedback으로 선택한다

어제의:

```text
Reload Cost = min(Transfer, Recompute)
```

를 실제 측정값으로 업데이트할 수 있다.

예:

```text
avg transfer latency[pageClass]
avg remesh latency[brickClass]
```

를 EWMA(Exponential Weighted Moving Average)로 유지한다.

그 결과:

```text
if remeshCost < transferCost:
    recompute
else:
    stream
```

정책이 hardware와 workload에 적응한다.

### 4.13 Eviction Regret

Page를 evict한 뒤 짧은 시간 안에 다시 load했다면 eviction decision이 좋지 않았을 가능성이 크다.

Metric:

```text
EvictionRegret(K)
=
pages reloaded within K frames after eviction
--------------------------------------------
evicted pages
```

이 값이 높다면:

- eviction threshold가 너무 aggressive
- hysteresis window가 짧음
- working set prediction이 부족

할 수 있다.

### 4.14 Reuse Distance

Page가 eviction된 뒤 몇 frame 만에 다시 필요했는지 측정한다.

```text
reuseDistance = demandFrame - evictionFrame
```

분포를 보면 residency hysteresis를 정할 수 있다.

예:

```text
p80 reuse distance = 12 frames
```

이라면 3-frame invisible rule은 지나치게 aggressive할 수 있다.

### 4.15 Residency Hysteresis

Load와 evict의 조건을 비대칭으로 만든다.

```text
Load:
priority > LOAD_THRESHOLD

Evict:
priority < EVICT_THRESHOLD
```

그리고:

```text
LOAD_THRESHOLD > EVICT_THRESHOLD
```

처럼 gap을 둘 수 있다.

이 hysteresis가 page가 경계에서 계속 들어왔다 나가는 것을 막는다.

### 4.16 Minimum Residency Age

새로 load한 page는 일정 기간 eviction 후보에서 제외할 수 있다.

```text
residentAge < minAge
→ not evictable
```

이것도 thrashing 방지다.

단 memory emergency에서는 override가 필요할 수 있다.

### 4.17 Memory Pressure Mode

Normal mode:

```text
quality first
```

Pressure mode:

```text
budget survival first
```

예:

```text
heapUsage / heapBudget
```

에 따라 mode를 나눈다.

```text
< 75% → COMFORT
75~85% → NORMAL
85~92% → PRESSURE
> 92% → EMERGENCY
```

숫자는 platform별 tuning 대상이다.

Mode마다:

- prefetch horizon
- fine LOD admission
- eviction aggressiveness
- scratch allocation allowance

를 바꿀 수 있다.

### 4.18 High/Low Water는 Controller State를 안정화한다

이전 노트처럼:

```text
enter PRESSURE at 90%
leave PRESSURE at 80%
```

식으로 hysteresis를 둔다.

Memory pressure가 89~91% 사이에서 움직인다고 매 frame mode를 뒤집지 않는다.

### 4.19 Budget Headroom은 “낭비”가 아니다

VRAM을 100% 쓰지 않는 이유:

- COW building allocation
- transient render target
- pipeline scratch
- acceleration structure rebuild
- OS budget reduction

이 있기 때문이다.

따라서:

```text
ResidentTarget
<
CurrentBudget
```

의 headroom은 stability budget이다.

### 4.20 Adaptive Streaming Bandwidth

Upload budget도 고정하지 않는다.

Feedback:

```text
Frame GPU Time
Transfer Queue Time
Miss Backlog
Memory Pressure
```

를 본다.

예:

```text
miss backlog high
AND frame has bandwidth headroom
→ upload budget ↑

render bandwidth saturated
→ upload budget ↓
```

Streaming scheduler가 rendering과 bandwidth를 공유한다면 매우 중요하다.

### 4.21 Upload Queue Backlog

Request가 생성되는 속도보다 upload 처리 속도가 느리면 backlog가 쌓인다.

Metrics:

```text
pendingRequestBytes
oldestRequestAge
```

Oldest request age가 계속 증가하면 controller가 demand를 따라가지 못하는 것이다.

이때:

- quality target 낮추기
- fine LOD request drop/coalesce
- upload budget 증가

중 선택이 필요하다.

### 4.22 Request Coalescing

같은 page에:

```text
LOD1 request
LOD0 request
LOD2 request
```

가 들어오면 priority와 target LOD를 합칠 수 있다.

예:

```text
desired = finest needed
priority = max(priority)
```

로 coalesce한다.

Queue pressure를 줄인다.

### 4.23 Request Obsolescence

Streaming request가 처리되기 전에 camera가 크게 움직여 page가 더 이상 필요하지 않을 수 있다.

따라서 queued request도:

```text
requestEpoch
predictionEpoch
```

을 가진다.

Processing 시:

```text
still useful?
```

를 다시 검사해 stale request를 drop할 수 있다.

### 4.24 Cancellation Value

Large transfer를 이미 시작했다면 cancel 비용이 더 클 수 있다.

상태를 나눈다.

```text
QUEUED
ALLOCATED
TRANSFER_STARTED
FINALIZING
RESIDENT
```

Cancellation은 early state에서만 허용하는 것이 단순하다.

### 4.25 Prefetch Waste

Load했지만 실제로 사용되기 전에 다시 evict된 bytes를 측정한다.

```text
PrefetchWasteBytes
```

높으면:

- horizon 너무 넓음
- prediction quality 낮음
- memory pressure와 prefetch policy 불일치

다.

### 4.26 Thrash Detector

간단한 detector:

```text
if reloadWithinKFramesRatio > threshold:
    thrashing = true
```

더 강한 signal:

```text
high upload bytes
high eviction bytes
stable heapUsage
low net resident gain
```

이면:

> memory가 이동만 하고 useful working set은 늘지 않는다.

라는 뜻이다.

### 4.27 Thrash Response

Thrashing이 감지되면:

- eviction hysteresis 증가
- minimum residency age 증가
- fine LOD admission 감소
- prefetch precision 우선
- working-set target 축소/확대 재평가

중 하나를 적용할 수 있다.

중요한 것은 load와 evict를 동시에 더 공격적으로 하지 않는 것이다.

### 4.28 Admission Control

모든 request를 load한 뒤 eviction하는 것보다 load 전에 질문할 수 있다.

```text
Is this page worth entering residency?
```

Admission score:

```text
Expected UtilityPerByte
>
Current Cold-Set UtilityPerByte
```

이면 cold page 하나를 밀어내고 admit한다.

이것은 cache admission control과 비슷하다.

### 4.29 Eviction을 “가장 나쁜 page 선택”으로 본다

Memory가 `X MB` 부족하면:

```text
lowest KeepScore pages
```

부터 제거한다.

정확한 global sort가 비싸면 bucket을 둔다.

```text
HOT
WARM
COLD
DISCARDABLE
```

GPU/CPU overhead를 줄인다.

### 4.30 Quality Budget

Memory budget이 작아지면 모든 region의 quality를 균등하게 낮출 필요는 없다.

예:

```text
selected device region → keep high LOD
peripheral region      → coarse LOD
offscreen predicted    → base LOD
```

즉 quality allocation도 priority scheduling이다.

### 4.31 Quality-Per-Byte와 Screen-Space Error

LOD0 → LOD1로 올렸을 때:

```text
additional bytes
```

와:

```text
screen-space error reduction
```

을 비교한다.

Priority:

```text
ΔQuality / ΔBytes
```

로 만들 수 있다.

Virtualized geometry의 residency와 자연스럽게 연결된다.

### 4.32 Quality-Per-Millisecond도 볼 수 있다

Memory가 충분하지만 streaming bandwidth가 부족할 수도 있다.

그때는:

```text
ΔQuality / LoadMilliseconds
```

가 더 중요한 score가 된다.

즉 resource constraint마다 denominator가 다르다.

```text
memory-limited → quality/byte
bandwidth-limited → quality/transfer-ms
compute-limited → quality/recompute-ms
```

이 관점이 매우 중요하다.

### 4.33 Multi-Resource Knapsack 관점

실제 시스템은 동시에:

- memory
- transfer bandwidth
- compute time

제약을 가진다.

따라서 완전한 최적화 문제는 multi-dimensional knapsack에 가깝다.

실무에서는 exact solver 대신 heuristic score와 class budget을 사용한다.

예:

```text
Geometry upload budget
Texture upload budget
Remesh compute budget
Scratch memory budget
```

를 분리한다.

### 4.34 Primary Data는 높은 Retention Priority

SDF/process field는 mesh를 재생성할 수 있는 primary state다.

Mesh는 derived data다.

Memory pressure가 심하면:

```text
evict derived mesh
keep primary SDF
```

가 자연스럽다.

반대로 SDF가 host에도 있고 GPU mesh가 매우 비싼 경우에는 cost model이 달라질 수 있다.

### 4.35 Scientific Viewer의 User Intent Signal

게임 camera 외에도 다음 signal이 중요하다.

- selected structure
- measurement cursor
- clipping plane
- layer isolation
- material filter
- probe region

사용자가 상호작용하는 영역은 screen area가 작아도 높은 utility를 가질 수 있다.

따라서 utility function은 단순 projected pixels가 아니다.

### 4.36 Focus-Aware Residency

예:

```text
selectedRegionBonus
hoveredRegionBonus
measurementRegionBonus
```

를 score에 추가한다.

Scientific/engineering tool에서 매우 실용적이다.

### 4.37 Simulation Front Prediction

Etch/deposition/flow front가 다음 timestep에 어디로 이동할지 알 수 있다면:

```text
future mesh demand
```

를 camera와 독립적으로 예측할 수 있다.

즉 working set은 두 source를 합친다.

```text
Render Demand Prediction
+
Simulation Demand Prediction
```

### 4.38 Rendering과 Simulation Priority가 충돌할 수 있다

현재 화면에 보이는 region은 render priority가 높다.

다음 process step에서 바뀔 region은 simulation priority가 높다.

Memory가 부족하면 same pool에서 경쟁할 수 있다.

따라서 primary state, derived mesh, render cache를 별도 budget domain으로 나누는 것이 유리할 수 있다.

### 4.39 Budget Partition

예:

```text
Primary Simulation  40%
Visible Geometry    35%
Prefetch Geometry   15%
Scratch/Transient   10%
```

고정 비율이 정답은 아니다.

Feedback loop가 class별 pressure를 보고 조절할 수 있다.

### 4.40 Soft Budget vs Hard Safety Limit

Soft target:

```text
normal optimization target
```

Hard limit:

```text
must not exceed
```

를 분리한다.

Soft target을 넘었다고 즉시 aggressive eviction할 필요는 없다.

Hard limit 접근에서는 fine LOD admission을 막는 등 emergency policy가 필요하다.

### 4.41 D3D12의 Cross-API 관점

DXGI의 `QueryVideoMemoryInfo`도:

```text
Budget
CurrentUsage
AvailableForReservation
CurrentReservation
```

을 제공한다.

D3D12 residency 문서는 budget이 background process나 foreground 전환에 따라 변할 수 있음을 강조한다.

또 `SetResidencyPriority`는 broad priority bucket을 제공하며, `EnqueueMakeResident`는 object residency를 async하게 요청할 수 있다.

즉 Vulkan이든 DirectX 12든 high-level 교훈은 같다.

> **Modern explicit graphics API에서도 application은 dynamic memory budget과 residency priority를 스스로 관리해야 한다.**

### 4.42 Priority Hint와 App Policy를 분리한다

Vulkan `VK_EXT_memory_priority`나 D3D12 `SetResidencyPriority`는 driver/OS에 주는 hint다.

App-level decision:

```text
which logical geometry to keep
```

를 대신하지 않는다.

즉 두 층:

```text
Engine Residency Policy
↓
API/OS Residency Priority Hint
```

로 본다.

### 4.43 GPU Feedback Compression

GPU가 모든 page의 full telemetry를 CPU에 보내면 readback 비용이 커진다.

GPU-side reduction:

```text
used bitset
miss bitset
priority max per page
request compaction
```

으로 compact feedback만 전달한다.

### 4.44 Per-Page Score는 SoA가 유리하다

Hot fields:

```text
residentState[]
importance[]
lastUsedEpoch[]
reuseDistanceClass[]
requestPriority[]
pinUntil[]
```

를 SoA로 두면 update/reduction pass가 필요한 field만 읽을 수 있다.

Cold debug fields는 분리한다.

### 4.45 Score Decay

과거 visibility가 영원히 priority를 유지하면 안 된다.

```text
score_t
=
decay × score_(t-1)
+ currentSignal
```

형태로 temporal decay를 사용한다.

Decay가 빠르면 reactive하지만 thrash 가능성이 커지고,
느리면 stable하지만 working set 변화에 늦다.

### 4.46 Separate Rise/Fall Rate

더 안정적인 방식:

```text
if signal rising:
    fast update
else:
    slow decay
```

즉 필요한 page는 빨리 올리고,
필요 없어 보이는 page는 천천히 내린다.

Residency hysteresis를 score dynamics로 구현한 것이다.

### 4.47 Camera Cut

Camera cut에서는 prediction history를 reset한다.

하지만 resident set 전체를 버리는 것은 좋지 않다.

Policy:

```text
old prediction confidence ↓
coarse fallback 유지
new frustum urgent requests ↑
fine prefetch temporarily reduced
```

처럼 transition mode를 둘 수 있다.

### 4.48 Cold Start

Scene load 직후 telemetry가 없다.

Cold-start policy:

- coarse root LOD resident
- current frustum first
- near-distance priority
- conservative bandwidth
- feedback 수집 후 adaptive mode

로 시작할 수 있다.

### 4.49 Controller Oscillation

Residency controller도 tuning을 잘못하면 oscillate한다.

예:

```text
miss ↑
→ upload budget ↑
→ bandwidth pressure ↑
→ frame time ↑
→ upload budget ↓
→ miss ↑
```

이런 feedback loop는 delayed system이라 쉽게 진동한다.

그래서:

- smoothing
- hysteresis
- bounded rate of change
- mode transition cooldown

이 중요하다.

### 4.50 Rate Limiting

Policy parameter를 frame마다 크게 바꾸지 않는다.

예:

```text
uploadBudget change ≤ 10% / second
```

처럼 변화율 제한을 둘 수 있다.

Exact 값은 profiler 기반이다.

### 4.51 EWMA

Noisy metric을 평활화한다.

```text
smoothedMissRate
=
α × currentMissRate
+
(1-α) × previous
```

Prefetch latency, reuse distance, upload throughput에도 사용할 수 있다.

### 4.52 Tail Latency를 봐야 한다

평균 streaming latency만 보면 rare long miss를 놓친다.

```text
p50
p95
p99
```

를 본다.

Interactive renderer에서는 p99 miss가 frame hitch를 만들 수 있다.

Prediction horizon은 평균보다 p95/p99에 맞춰야 할 수도 있다.

### 4.53 Residency Miss Reason Code

Miss를 하나로 합치지 않는다.

```text
MISS_NOT_REQUESTED
MISS_REQUEST_LATE
MISS_UPLOAD_BACKLOG
MISS_BUDGET_REJECT
MISS_EVICTED_TOO_EARLY
MISS_CAMERA_CUT
MISS_GENERATION_STALE
```

Reason별로 controller fix가 다르다.

### 4.54 Eviction Reason Code

```text
EVICT_MEMORY_PRESSURE
EVICT_FINE_LOD
EVICT_COLD
EVICT_RECOMPUTABLE
EVICT_STALE_EPOCH
```

를 기록하면 regret 분석이 쉬워진다.

### 4.55 Profiler에서 봐야 할 지표

- heapBudget / heapUsage
- resident target
- live resident bytes
- pinned bytes
- cold bytes
- upload bytes/frame
- upload bandwidth
- request backlog bytes
- oldest request age
- residency hit ratio
- prefetch hit ratio
- prefetch precision
- fallback LOD ratio
- hard miss count
- p50/p95/p99 streaming latency
- eviction bytes/frame
- reload-within-K-frames ratio
- eviction regret
- prefetch waste bytes
- thrash episodes
- average reuse distance
- quality-per-byte distribution
- quality-per-transfer-ms
- remesh vs transfer decision count
- memory-pressure mode time
- budget oscillation count
- camera prediction error
- simulation prediction hit rate

특히 다음 네 개는 dashboard의 핵심이 될 수 있다.

```text
Residency Hit Ratio
Eviction Regret
Prefetch Precision
Useful Quality / Resident Byte
```

---

## 5. 내 관심 분야와 연결

### Semiconductor Process Visualization

반도체 구조에서는 generic open-world camera보다 강한 domain signal을 사용할 수 있다.

예:

```text
Current cut plane
Selected process layer
Measurement region
Etch/deposition front
Material isolation mode
Camera orbit target
```

따라서 utility score를 단순 screen area가 아니라 **engineering relevance**까지 포함해 만들 수 있다.

```text
selected trench
→ small screen area
→ high utility
```

가 가능하다.

### Sparse SDF / Mesh Cache

사용자의 pipeline에서는 SDF를 primary state, mesh/meshlet을 derived cache로 분리할 수 있다.

```text
Primary SDF
    ↓
Derived Mesh
```

Mesh residency miss가 생기면:

```text
host transfer
vs
GPU remesh
```

를 runtime 측정값으로 선택할 수 있다.

Geometry cache가 일반 asset cache와 다른 핵심이다.

### CUDA → Vulkan GPU-Stay-GPU

GPU side:

```text
visibility feedback
page use bits
request compaction
remesh cost statistics
```

Host/Vulkan side:

```text
heap budget
allocation
sparse bind
transfer scheduling
```

처럼 hot feedback과 OS/API policy를 분리할 수 있다.

### CFD

CFD에서는 simulation front나 refinement indicator가 future visualization demand의 강한 predictor가 된다.

Camera prediction만으로 residency를 관리하는 것보다:

```text
physics-driven prediction
+
view-driven prediction
```

을 합치는 것이 좋다.

### Game Engine / Graphics Career

이 개념은:

- open-world streaming
- Nanite-like virtualized geometry
- procedural world
- dynamic destruction
- ray-tracing geometry cache
- GPU-driven asset manager

와 직접 연결된다.

면접에서 강한 설명은:

> **“LRU보다 어떤 metric이 더 좋은가?”**

에 대해 quality-per-byte, reload cost, eviction regret, latency-aware prefetch까지 설명하는 것이다.

---

## 6. 머릿속에 남길 질문 3개

1. **Prefetch hit ratio가 높지만 prefetch waste bytes도 크다면 prediction horizon, precision, admission policy 중 무엇을 먼저 조정해야 하며 왜 그런가?**
2. **Dynamic SDF mesh가 transfer보다 GPU remesh가 빠른 구간이 존재한다면 residency manager의 ‘reload cost’는 단순 I/O latency가 아니라 어떤 runtime measurements를 포함해야 할까?**
3. **Eviction regret가 계속 높은데 memory budget도 빠듯하다면 hysteresis를 늘리는 것, quality target을 낮추는 것, resident working-set prediction을 바꾸는 것 중 어떤 순서로 접근해야 할까?**

---

## 7. graphics engineer 면접 질문 1개와 답변

### 질문

**“Geometry streaming에서 cache hit ratio를 최대화하면 좋은 residency manager라고 볼 수 있나요?”**

### 답변

Hit ratio만으로는 충분하지 않다.

첫째, 모든 geometry를 오래 유지하면 hit ratio는 높아지지만 memory budget을 넘을 수 있다.

둘째, 중요하지 않은 1 MB page의 hit와 camera 근처 128 MB high-detail geometry miss를 동일하게 세면 실제 visual impact를 반영하지 못한다.

셋째, prefetch를 과도하게 하면 hit ratio는 좋아질 수 있지만 실제 사용되지 않는 page를 많이 load해 bandwidth와 memory를 낭비할 수 있다.

넷째, eviction 직후 다시 load하는 thrashing은 평균 hit ratio에 충분히 드러나지 않을 수 있다.

그래서 production residency manager는 다음을 함께 본다.

```text
Residency Hit Ratio
Prefetch Precision
Eviction Regret
Streaming Latency
Memory Budget Pressure
Utility / Resident Byte
```

특히 resource 간 value를 비교하려면:

```text
KeepScore
≈
Expected Future Utility
× Reload Cost
------------------------
Resident Bytes
```

같은 cost/benefit 관점이 유용하다.

Vulkan의 `heapBudget`은 실행 중 변할 수 있는 rough estimate이고, D3D12/DXGI의 video-memory `Budget`도 OS 상황에 따라 변한다. 따라서 fixed resident target보다 current budget과 feedback을 이용한 controller가 더 robust하다.

핵심은:

> **좋은 residency manager는 hit를 많이 만드는 시스템이 아니라, 제한된 memory와 bandwidth에서 중요한 geometry가 필요할 때 준비되어 있도록 만드는 시스템이다.**

---

## 8. 포트폴리오 / 커리어 연결

이 주제는 graphics engineer가 단순 streaming queue가 아니라 **feedback-controlled resource management**를 이해한다는 것을 보여주기 좋다.

### Rendering

- desired / resident LOD
- fallback geometry
- visibility-driven demand
- camera prediction
- screen-space quality

### GPU Memory

- dynamic budget
- working set
- admission control
- eviction hysteresis
- thrash prevention

### GPU Compute

- page-use feedback
- request compaction
- per-page priority reduction
- remesh cost feedback

### Simulation

- SDF primary state
- derived mesh cache
- simulation-front prediction
- recompute vs transfer

### Vulkan / DirectX 12

- `VK_EXT_memory_budget`
- `VK_EXT_memory_priority`
- sparse residency
- DXGI `QueryVideoMemoryInfo`
- D3D12 residency priority
- async make-resident concepts

### C++ / Engine Architecture

- `ResidencyScore`
- `StreamingBudget`
- `MemoryPressureMode`
- `RequestEpoch`
- `EvictionReason`
- `ReloadCostModel`

포트폴리오에서는 다음처럼 설명할 수 있다.

> **“GPU geometry residency를 LRU가 아닌 feedback controller로 구성했습니다. Camera/simulation prediction으로 prefetch request를 만들고, per-page utility-per-byte와 measured reload cost를 기반으로 admission/eviction을 결정했습니다. Prefetch precision, eviction regret, p99 streaming latency를 추적해 horizon과 hysteresis를 조절했고, memory budget pressure가 높아지면 fine LOD admission을 줄이고 coarse fallback을 유지했습니다. SDF-derived mesh는 runtime transfer/remesh 비용을 비교해 reload path를 선택했습니다.”**

이 설명은 GPU memory, rendering quality, streaming, profiling, simulation architecture를 하나로 연결한다.

---

## 9. 내일 이어서 볼 개념

**GPU Geometry Page Fault Avoidance: Request Latency Hiding, Fallback Hierarchies, and Stall-Free Rendering**

오늘은 residency system을 feedback loop로 만들고, quality-per-byte와 thrash control로 policy를 조절하는 방법을 봤다.

다음 질문은:

> **아무리 prediction을 잘해도 residency miss가 0이 될 수 없다면, miss가 발생했을 때 frame을 멈추지 않는 renderer를 어떻게 설계할 것인가?**

학습 흐름:

```text
Geometry Residency
→ Feedback-Controlled Streaming
→ Miss Handling
→ Fallback Hierarchy
→ Stall-Free Rendering
```

다음 노트에서는:

- hard miss vs soft miss
- resident root representation
- missing-page indirection
- late request generation
- draw suppression
- parent/ancestor fallback
- temporal continuity
- page arrival synchronization
- partial geometry availability
- frame-hitch avoidance

를 중심으로 이어간다.

---

## 10. 참고 키워드

- GPU Residency Feedback Loop
- Closed-Loop Residency Control
- Geometry Streaming
- Working Set
- Prefetch
- Prefetch Horizon
- Prefetch Precision
- Prefetch Recall
- Streaming Latency
- p95 / p99 Latency
- Time-to-Need
- Camera Prediction
- Camera Cut
- Simulation-Front Prediction
- Quality-per-Byte
- Utility-per-Byte
- Quality-per-Millisecond
- Reload Cost
- Transfer vs Recompute
- Admission Control
- Eviction Regret
- Reuse Distance
- Thrashing
- Residency Hysteresis
- Minimum Residency Age
- High-Water / Low-Water Mark
- Memory Pressure Mode
- Adaptive Streaming Budget
- Request Backlog
- Request Coalescing
- Request Obsolescence
- Fallback LOD
- EWMA
- Score Decay
- `VK_EXT_memory_budget`
- `VkPhysicalDeviceMemoryBudgetPropertiesEXT`
- `heapBudget`
- `heapUsage`
- `VK_EXT_memory_priority`
- DXGI `QueryVideoMemoryInfo`
- `DXGI_QUERY_VIDEO_MEMORY_INFO`
- D3D12 Residency
- `D3D12_RESIDENCY_PRIORITY`
- `ID3D12Device3::EnqueueMakeResident`
- NVIDIA RTX Mega Geometry
- Dynamic SDF
- Meshlet Streaming
- Derived Geometry Cache
- Khronos Vulkan Specification — **Memory Allocation / Memory Budget**
  - https://docs.vulkan.org/spec/latest/chapters/memory.html
- Khronos Vulkan Reference — **VK_EXT_memory_budget**
  - https://docs.vulkan.org/refpages/latest/refpages/source/VK_EXT_memory_budget.html
- Khronos Vulkan Specification — **Sparse Resources**
  - https://docs.vulkan.org/spec/latest/chapters/sparsemem.html
- Khronos Vulkan Guide — **Sparse Resources**
  - https://docs.vulkan.org/guide/latest/sparse_resources.html
- Microsoft Learn — **D3D12 Residency**
  - https://learn.microsoft.com/en-us/windows/win32/direct3d12/residency
- Microsoft Learn — **DXGI_QUERY_VIDEO_MEMORY_INFO**
  - https://learn.microsoft.com/en-us/windows/win32/api/dxgi1_4/ns-dxgi1_4-dxgi_query_video_memory_info
- Microsoft Learn — **D3D12_RESIDENCY_PRIORITY**
  - https://learn.microsoft.com/en-us/windows/win32/api/d3d12/ne-d3d12-d3d12_residency_priority
- Microsoft Learn — **ID3D12Device3::EnqueueMakeResident**
  - https://learn.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12device3-enqueuemakeresident
- NVIDIA Technical Blog — **RTX Mega Geometry**
  - https://developer.nvidia.com/blog/nvidia-rtx-mega-geometry-now-available-with-new-vulkan-samples/
