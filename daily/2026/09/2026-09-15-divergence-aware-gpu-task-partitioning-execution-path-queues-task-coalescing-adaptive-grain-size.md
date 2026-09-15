---
title: "Divergence-Aware GPU Task Partitioning: Execution-Path Queues, Task Coalescing, and Adaptive Grain Size"
date: "2026-09-15"
category: Graphics
tags: [GPU, Rendering, GPU-Driven Rendering, Persistent Threads, Task Scheduling, Warp Divergence, EPAQ, Task Coalescing, Adaptive Grain Size, CUDA, SDF, Meshing, Memory Layout, C++]
level: intermediate
---

# [Daily Graphics Growth] 2026-09-15 - Divergence-Aware GPU Task Partitioning: Execution-Path Queues, Task Coalescing, and Adaptive Grain Size

## 1. 오늘의 개념

어제는 **Local GPU Queues and Work Stealing**에서 global queue contention을 줄이고, idle worker가 busy worker의 local deque에서 work를 가져가 long tail을 줄이는 구조를 봤다.

하지만 load balance가 좋아졌다고 해서 GPU utilization이 자동으로 좋아지는 것은 아니다.

예를 들어 thread-level worker가 다음 task를 한 warp 안에 섞어 받았다고 하자.

```text
lane 0  → simple SDF crossing
lane 1  → complex topology repair
lane 2  → meshlet packing
lane 3  → no-op
lane 4  → coarse/fine transition
...
```

모든 lane이 “바쁜 상태”여도 실제로는 서로 다른 execution path를 따라가면서 warp가 각 path를 순차적으로 실행할 수 있다. CUDA의 현재 SIMT execution model에서도 warp는 하나의 common instruction stream을 실행하며, data-dependent branch로 lane들이 갈라지면 각 branch path를 따로 실행하고 다른 lane은 비활성화된다.

즉 어제의 문제는:

```text
worker-level load imbalance
```

였고, 오늘의 문제는:

```text
warp-level execution imbalance
```

다.

오늘은 이 문제를 세 가지 축으로 연결한다.

1. **Execution-Path Queueing**  
   비슷한 control flow를 가진 task끼리 queue/class를 나눠 같은 warp에 배치한다.

2. **Task Coalescing**  
   너무 작은 task 여러 개를 하나의 scheduling unit으로 묶어 queue/steal/dispatch overhead를 amortize한다.

3. **Adaptive Grain Size**  
   task가 너무 작아 scheduler overhead가 지배하거나 너무 커서 load imbalance가 생기는 두 극단 사이에서 runtime work unit 크기를 조절한다.

2026년의 **GTaP**은 GPU-resident fork-join runtime에서 thread-level worker의 heterogeneous task가 한 warp에 섞이면서 생기는 divergence를 줄이기 위해 **Execution-Path-Aware Queueing(EPAQ)** 을 제안한다. EPAQ는 user-defined 기준으로 task queue를 나누며, 효과는 workload dependent하지만 일부 benchmark에서는 유의미한 개선을 보였다.

오늘의 핵심 질문은 다음이다.

> **GPU task scheduler에서 “모든 worker를 바쁘게 만드는 것”보다 “같은 warp가 비슷한 일을 하게 만드는 것”이 더 중요해지는 지점은 어디이며, 그 과정에서 queue fragmentation과 parallelism 손실을 어떻게 제어해야 하는가?**

---

## 2. 한 줄 핵심

> Divergence-aware scheduling의 핵심은 **task를 무조건 잘게 쪼개고 많이 훔치는 것이 아니라, 비슷한 execution path를 가진 work를 충분한 grain으로 묶어 warp coherence를 높이면서도 queue depth와 parallel slack을 잃지 않는 균형점을 찾는 것**이다.

---

## 3. 왜 중요한가

GPU는 thread 수가 많다고 자동으로 효율적인 것이 아니다.

CUDA Best Practices Guide는 warp 내부 thread가 서로 다른 branch path를 따라가면 각 path를 따로 실행해야 하므로 effective instruction throughput이 감소한다고 설명한다. 반대로 warp 전체가 같은 path를 따르면 full efficiency에 가까워진다.

Irregular graphics/geometry workload에서는 이 문제가 자주 발생한다.

예를 들어 sparse SDF meshing에서 active cell마다 상황이 다르다.

```text
Case A: no surface crossing
Case B: simple single-surface crossing
Case C: multiple components
Case D: coarse/fine transition
Case E: topology ambiguity
```

이를 하나의 generic kernel/queue에서 무작위로 섞으면 한 warp가 여러 code path를 순차 실행할 수 있다.

하지만 task class를 지나치게 많이 나누면 또 다른 문제가 생긴다.

```text
Queue A: 6 tasks
Queue B: 12 tasks
Queue C: 2 tasks
Queue D: 3 tasks
```

각 queue가 너무 얕아져 GPU를 채울 parallelism이 사라질 수 있다.

즉 divergence-aware partitioning에는 항상 두 힘이 반대 방향으로 작용한다.

```text
More classes
→ better execution coherence
→ worse queue occupancy / parallel slack

Fewer classes
→ better load availability
→ worse warp divergence
```

여기에 task grain까지 들어오면 세 번째 축이 생긴다.

```text
Tiny tasks
→ excellent load balance
→ large scheduler overhead

Large tasks
→ small scheduler overhead
→ long tail / divergence inside task
```

그래서 좋은 GPU task runtime은 **queue count, task type, grain size를 독립적으로 결정하지 않는다.**

---

## 4. 구현 관점

### 4.1 Divergence에는 두 종류가 있다

GPU task scheduling에서 divergence를 단순 branch divergence 하나로만 보면 부족하다.

#### Control-Flow Divergence

같은 warp lane들이 서로 다른 branch path를 실행한다.

```text
if (simple)
    fast path
else
    topology path
```

#### Work-Length Divergence

모두 같은 path를 시작해도 lane마다 loop iteration이나 traversal depth가 다르다.

```text
lane 0 → 4 iterations
lane 1 → 80 iterations
lane 2 → 7 iterations
```

실무에서는 둘이 함께 나타나는 경우가 많다.

따라서 task partition criterion은 단순 `taskType`뿐 아니라:

- branch class
- expected iteration count
- topology complexity
- refinement level
- projected work size

같은 **execution signature**로 확장할 수 있다.

### 4.2 Execution-Path Queue는 “shader variant queue”와 비슷한 사고방식이다

렌더러에서 opaque, alpha-test, transparent를 같은 pipeline으로 억지로 처리하지 않는 것처럼 GPU task runtime도 execution path가 크게 다른 work를 분리할 수 있다.

예:

```text
QUEUE_SIMPLE_SURFACE
QUEUE_COMPLEX_TOPOLOGY
QUEUE_TRANSITION
QUEUE_VISIBILITY
```

중요한 것은 queue를 domain label마다 만드는 것이 아니라 **실제 instruction path 차이가 큰 경계**에서 나누는 것이다.

좋은 class boundary:

- 서로 다른 알고리즘 단계
- heavy loop vs short path
- topology repair 필요 여부
- data footprint가 크게 다른 path

나쁜 class boundary:

- 단순 material ID
- task마다 거의 동일한 code인데 semantic label만 다른 경우

즉 queue class는 “업무 분류”가 아니라 **SIMT execution similarity**를 표현해야 한다.

### 4.3 GTaP의 EPAQ에서 배울 점

2026년 GTaP은 thread-level workers에서 **Execution-Path-Aware Queueing(EPAQ)** 을 제시한다.

핵심 아이디어는:

```text
heterogeneous tasks
    ↓ user-defined classifier
multiple queues
    ↓
similar-path tasks grouped
```

이다.

중요한 점은 EPAQ가 자동으로 항상 이득을 내지 않는다는 것이다.

논문에서도 효과가 workload-dependent하다고 보고한다.

이유는 명확하다.

- divergence가 원래 작다면 queue partition 비용만 증가
- class별 task 수가 적으면 parallelism이 줄어듦
- classification 자체가 비싸면 scheduler overhead 증가
- 같은 path라도 memory access pattern이 다르면 bandwidth bottleneck이 남음

그래서 EPAQ의 진짜 교훈은:

> **Divergence를 줄이기 위해 queue를 나누는 것이 목적이 아니라, queue partition의 이득을 runtime distribution과 profiler로 확인해야 한다.**

### 4.4 Queue Count는 “많을수록 좋다”가 아니다

Task class가 `K`개라면 local worker마다 K개의 queue를 갖는 구조를 상상할 수 있다.

```text
Worker 0:
  Q0 Q1 Q2 Q3

Worker 1:
  Q0 Q1 Q2 Q3
```

K가 커지면:

- metadata 증가
- queue probing 증가
- steal victim × task-class 탐색 증가
- queue당 평균 depth 감소
- termination detection 복잡도 증가

가 생긴다.

따라서 class count는 **divergence reduction gain / queue fragmentation cost**의 trade-off다.

실무적으로는 실행 경로 차이가 큰 소수의 coarse class가 수십 개의 micro-class보다 합리적인 출발점이 되는 경우가 많다.

### 4.5 Classification은 cheap 해야 한다

Task를 실행하기 전에 어느 queue로 보낼지 판단하는 classifier 자체가 비싸면 이득이 사라진다.

좋은 classifier는 이미 upstream에서 알고 있는 metadata를 재사용한다.

예:

```text
topologyFlags
activeCellCount
refinementLevel
materialTransitionMask
estimatedPrimitiveCount
```

추가 geometry traversal 없이 몇 비트로 class를 결정할 수 있다면 scheduler overhead가 작다.

즉 **execution-path metadata를 algorithm output에 함께 저장하는 것**이 유리하다.

### 4.6 Task Coalescing은 scheduler overhead를 amortize한다

작은 task 여러 개를 하나의 batch로 묶는다.

```text
Before:
T0 T1 T2 T3 T4 T5 T6 T7

After:
Batch A = T0..T3
Batch B = T4..T7
```

이득:

- queue operation 수 감소
- steal 횟수 감소
- atomic reservation 감소
- task descriptor fetch 감소
- scheduling state transition 감소

하지만 무작위 task를 묶으면 divergence가 악화될 수 있다.

따라서 좋은 coalescing은 보통:

```text
same execution class
+ nearby spatial region
+ similar estimated cost
```

를 선호한다.

### 4.7 Coalescing은 Memory Coalescing과도 연결된다

Control-flow만 비슷한 task를 묶어도 memory access가 랜덤이면 warp efficiency가 충분히 좋아지지 않을 수 있다.

예:

```text
task class = same
but SDF brick addresses are far apart
```

이면 memory transaction이 흩어진다.

그래서 task batch key는 두 축을 가질 수 있다.

```text
Execution Coherence
+
Memory Locality
```

예를 들어:

```text
class = SIMPLE_SURFACE
spatialGroup = Morton prefix
```

를 함께 사용하면 비슷한 code path와 neighboring SDF pages를 동시에 맞출 수 있다.

GPU irregular workload 연구에서도 thread/data 재배열이 control divergence뿐 아니라 memory coalescing과 cache locality를 개선할 수 있다는 관점이 반복된다.

### 4.8 Coalescing이 너무 크면 다시 load imbalance가 생긴다

Batch size가 너무 커지면 worker 하나가 긴 batch를 붙잡는다.

```text
small batch
→ scheduling overhead high
→ balance good

large batch
→ scheduling overhead low
→ balance poor
```

따라서 batch size는 static constant보다 runtime workload의 task-cost distribution과 연동할 수 있다.

### 4.9 Grain Size는 “한 task가 몇 element를 처리하느냐”다

예를 들어 brick meshing task 하나가:

```text
1 brick
4 bricks
16 cells
64 cells
one spatial tile
```

중 무엇을 처리할지 결정하는 것이 grain size다.

좋은 grain size는 다음 조건을 만족해야 한다.

```text
Useful Work per Task
>>
Scheduler Overhead

Task Count
>>
Active Worker Count
```

두 조건이 동시에 필요하다.

첫 번째만 만족하면 task가 너무 커질 수 있고,
두 번째만 만족하면 task가 너무 작아 scheduler가 병목이 된다.

### 4.10 Parallel Slack이라는 관점

Worker가 120개라면 task가 120개만 있는 것보다 1,000개가 있는 편이 dynamic load balance에 유리하다.

이를 개념적으로 **parallel slack**으로 볼 수 있다.

```text
Parallel Slack
≈
Ready Tasks / Active Workers
```

너무 낮으면:

- steal할 work 부족
- long tail 증가

너무 높으면:

- queue memory 증가
- scheduling overhead 증가
- task granularity가 지나치게 작을 가능성

즉 “task 수가 많다” 자체가 목표는 아니다.

### 4.11 Adaptive Grain Size는 scheduler state를 feedback으로 쓸 수 있다

Task grain을 고정하지 않고 runtime pressure에 따라 바꿀 수 있다.

개념적 정책:

```text
ready tasks 부족
→ finer split

queue pressure 높음
→ coalesce / suppress split

worker idle 많음
→ split more

scheduler overhead 높음
→ merge more
```

중요한 것은 exact threshold가 아니라 **scheduler telemetry가 geometry work decomposition에 feedback을 준다**는 구조다.

### 4.12 Queue Depth는 Grain Feedback Signal이 될 수 있다

Worker 수보다 ready task가 훨씬 적으면 더 세분화할 이유가 있다.

반대로 queue가 이미 충분히 깊다면 additional child split은 queue pressure만 키운다.

따라서:

```text
QueueDepth / ActiveWorkers
```

는 유용한 grain-size signal이다.

숫자 자체는 workload와 hardware에 따라 달라진다.

### 4.13 Task Cost Prediction은 정확할 필요가 없다

Scheduler는 exact execution time을 알 필요가 없다.

다음과 같은 coarse class만 있어도 충분할 수 있다.

```text
CHEAP
MEDIUM
EXPENSIVE
```

Cost proxy:

- active voxel count
- number of crossings
- estimated triangles
- topology flags
- iteration count upper bound
- previous frame execution bucket

이 coarse predictor로:

- heavy task는 split
- tiny task는 coalesce
- similar cost끼리 warp 배치

를 할 수 있다.

### 4.14 Previous Execution Time은 유용하지만 stale할 수 있다

동일 brick/meshlet의 previous cost를 history로 저장할 수 있다.

장점:

- runtime-measured signal
- complex static cost model 불필요

문제:

- SDF topology가 바뀜
- LOD가 달라짐
- camera-dependent path가 바뀜
- hardware/cache condition이 다름

따라서 history는 hard contract가 아니라 **cost hint**다.

Generation/geometry epoch가 바뀌면 cost history를 reset하거나 confidence를 낮추는 편이 맞다.

### 4.15 Tiny Task는 여러 개 묶어 Warp Unit으로 만들 수 있다

Thread-level task가 너무 작다면 lane마다 하나씩 무작위로 가져오는 대신 동일 class의 32개 task를 warp batch로 만들 수 있다.

```text
Warp Batch:
32 SIMPLE tasks
```

이렇게 하면:

- 같은 code path
- queue pop amortization
- predictable warp occupancy

를 얻을 수 있다.

단, task 32개가 모두 충분히 비슷한 cost라는 보장은 없으므로 work-length divergence는 여전히 존재할 수 있다.

### 4.16 Block-Level Batch는 Shared Memory Reuse와 연결된다

Neighboring SDF brick task를 block batch로 묶으면:

- material table
- common field page
- marching table
- allocator metadata

를 shared/L1에 재사용할 수 있다.

즉 grain size는 단순 scheduler 단위가 아니라 **memory hierarchy reuse unit**이 될 수 있다.

Graphics/GPGPU에서는 좋은 grain이 “compute가 많은 task”라기보다:

> **협력 thread들이 함께 재사용할 data가 있는 task**

일 수 있다.

### 4.17 Warp Divergence와 Memory Divergence는 서로 독립적이다

같은 path의 task를 warp에 모았다고 하자.

```text
all lanes → simple meshing path
```

하지만 각 lane이 서로 다른 sparse page를 읽으면 global memory accesses가 흩어질 수 있다.

반대로 neighboring brick만 묶었는데 일부는 topology repair, 일부는 simple path면 control divergence가 커질 수 있다.

따라서 task packing의 두 목표는:

```text
Execution similarity
Spatial/memory similarity
```

다.

둘이 충돌할 때 profiler로 어느 병목이 더 큰지 봐야 한다.

### 4.18 SoA Metadata는 빠른 Classification에 유리하다

Task descriptor 전체가 큰 AoS라면 queue classification 단계에서 필요 없는 field까지 cache line에 들어온다.

Hot metadata를 분리할 수 있다.

```text
TaskClass[]
EstimatedCost[]
SpatialKey[]
Generation[]
Epoch[]
PayloadHandle[]
```

Classifier/queue builder는 앞의 몇 배열만 읽고,
실제 worker가 task를 실행할 때 payload를 resolve한다.

이전 노트의 hot/cold metadata split이 scheduler에도 그대로 적용된다.

### 4.19 Class Queue마다 다른 Worker Granularity를 둘 수 있다

모든 task class를 같은 worker shape로 실행할 필요는 없다.

예:

```text
Simple classification
→ thread/warp worker

Heavy topology repair
→ block worker

Large reduction
→ cooperative block
```

이렇게 task class가 execution path뿐 아니라 **execution shape**까지 encode할 수 있다.

다만 persistent runtime 하나 안에서 worker shape가 너무 다양하면 complexity가 커진다.

Subsystem boundary에서 별도 kernel/dispatch로 분리하는 것이 더 나은 경우도 있다.

### 4.20 Kernel Boundary는 Divergence 제거 도구이기도 하다

모든 irregular work를 하나의 persistent kernel에서 처리하면 queueing은 유연하지만 code footprint와 branch structure가 커질 수 있다.

반대로 큰 task class 경계에서 kernel을 분리하면:

- instruction path 완전 분리
- register footprint class별 최적화
- shared-memory usage 분리
- compiler optimization 단순화

가 가능하다.

비용은:

- launch/synchronization
- global memory intermediate
- scheduling latency

다.

즉 execution-path queue와 multi-kernel decomposition은 같은 문제의 서로 다른 해법이다.

### 4.21 Register Pressure도 Task Class의 근거가 된다

두 path가 control flow는 비슷해 보여도 한 path가 많은 register를 사용하면 whole kernel의 register allocation/occupancy에 영향을 줄 수 있다.

예:

```text
simple path: 20 registers
complex path: 80 registers
```

같은 mega-kernel에 묶이면 simple task도 높은 register footprint의 영향을 받을 수 있다.

이 경우 queue만 분리하는 것보다 kernel specialization 자체가 더 유리할 수 있다.

즉 class partition의 근거는:

- divergence
- register pressure
- shared-memory requirement
- memory footprint

까지 포함한다.

### 4.22 Branch Predication이 작은 Branch의 비용을 줄일 수 있다

CUDA Best Practices Guide는 짧은 branch는 compiler가 predication으로 바꿀 수 있다고 설명한다.

따라서 매우 작은 `if` 하나를 제거하려고 task queue를 추가하는 것은 오히려 손해일 수 있다.

Queue partition이 가치 있는 상황은 일반적으로:

- 긴 divergent path
- 반복 loop 차이 큼
- memory access pattern 차이 큼
- register/shared-memory footprint 차이 큼

같은 경우다.

즉 **모든 branch divergence를 scheduler로 해결하려 하지 않는다.**

### 4.23 Independent Thread Scheduling이 Divergence 비용을 없애는 것은 아니다

Volta 이후 Independent Thread Scheduling은 warp 내 thread execution의 유연성을 높였지만, CUDA 문서가 여전히 divergence 최소화를 높은 우선순위로 권장하는 이유는 명확하다.

같은 warp가 여러 path를 실행해야 하는 사실 자체는 남는다.

따라서:

```text
Independent scheduling
!=
free divergence
```

다.

### 4.24 Queue Fragmentation을 측정해야 한다

Task class가 많아지면 aggregate ready work는 충분한데 각 queue가 얕을 수 있다.

예:

```text
Total ready = 1024 tasks

Queue A = 950
Queue B = 30
Queue C = 20
Queue D = 24
```

특정 class 전용 worker가 있다면 B/C/D worker가 idle할 수 있다.

유용한 질문은 “전체 work는 충분한데 class partition 때문에 worker가 놀고 있는가?”다.

### 4.25 Stealing은 같은 Class부터 시작할 수 있다

어제의 local work stealing과 오늘의 execution class를 결합하면 thief policy를 계층화할 수 있다.

```text
1. local same-class
2. remote same-class
3. local compatible-class
4. remote compatible-class
5. fallback any-class
```

이렇게 하면 load balance를 위해 완전히 heterogeneous task를 섞기 전에 execution coherence를 최대한 유지할 수 있다.

### 4.26 Compatible Class라는 개념

Task class를 strict하게 분리하면 parallelism이 부족할 수 있다.

그래서 class compatibility graph를 둘 수 있다.

예:

```text
SIMPLE_SURFACE
 ↔ TRANSITION_LIGHT

TOPOLOGY_HEAVY
 ↔ MANIFOLD_REPAIR
```

비슷한 register/path/memory profile을 가진 class끼리는 같은 warp에 섞어도 divergence penalty가 상대적으로 작을 수 있다.

이는 hard partition과 no partition 사이의 중간 해법이다.

### 4.27 Task Coalescing에서 Duplicate Work 제거까지 할 수 있다

같은 logical object에 여러 update request가 들어온 경우 단순 batch가 아니라 coalescing으로 중복 자체를 제거할 수 있다.

예:

```text
Brick 42 dirty geometry
Brick 42 dirty material
Brick 42 neighbor invalidation
```

를:

```text
Brick 42 combined flags
```

로 합칠 수 있다.

Queue bandwidth 감소와 task execution 감소를 동시에 얻는다.

이전의 dirty epoch/bitset과 자연스럽게 연결된다.

### 4.28 Dynamic SDF에서 좋은 Execution Signature

Semiconductor process geometry를 생각하면 다음 metadata가 유용하다.

```text
hasSurfaceCrossing
crossingCountClass
topologyAmbiguous
needsTransitionRepair
activeMaterialCount
estimatedPrimitiveClass
neighborDirtyMask
```

이 값을 기반으로 coarse task class를 만들 수 있다.

예:

```text
EMPTY
SIMPLE_SURFACE
COMPLEX_SURFACE
TOPOLOGY_REPAIR
TRANSITION
```

중요한 점은 classification이 실제 geometry processing보다 훨씬 싸야 한다는 것이다.

### 4.29 Visibility Work에도 같은 원리가 적용된다

Rendering visibility에서도 meshlet마다 workload가 다르다.

예:

```text
frustum-only
cone test
Hi-Z test
LOD transition
alpha-tested special handling
```

모든 candidate가 같은 compute path를 타게 하면 divergence가 생길 수 있다.

하지만 visibility pass는 task가 매우 작기 때문에 queue partition overhead가 쉽게 더 커질 수 있다.

따라서 geometry/simulation 쪽보다 **queue partition threshold가 더 높아야 할 가능성**이 크다.

즉 같은 scheduler idea라도 workload grain에 따라 적용 가치가 달라진다.

### 4.30 Adaptive Grain은 Frame Budget과도 연결할 수 있다

Real-time renderer/visualizer에서는 throughput뿐 아니라 frame latency가 중요하다.

다음과 같은 policy를 생각할 수 있다.

```text
frame headroom 많음
→ finer tasks / better balance

frame budget tight
→ coarser batching / scheduler overhead 축소
```

다만 grain change가 결과 quality를 바꾸면 안 된다.

Scheduler grain은 **execution grouping**만 바꾸고 simulation/rendering semantics는 유지해야 한다.

### 4.31 GPU Runtime과 Vulkan Renderer의 경계

사용자의 GPU-stay-GPU 구조에서는 divergence-aware dynamic scheduling을 CUDA/Warp geometry domain 내부에 두고, Vulkan에는 compact한 stable snapshot을 넘기는 것이 유리하다.

```text
CUDA/Warp:
task classify
coalesce
adaptive grain
meshing
        ↓
Published meshlet snapshot
        ↓
Vulkan:
visibility
indirect rendering
```

이렇게 하면 renderer가 compute scheduler의 irregularity까지 직접 알 필요가 없다.

Subsystem boundary에서:

- geometry epoch
- meshlet handle table
- count/indirect metadata

만 publish하면 된다.

### 4.32 C++ Host-Side Scheduler Contract

Host-side graph description에서 task class를 plain integer로만 두기보다 의미를 분리할 수 있다.

```text
TaskClass
CostClass
SpatialGroup
GrainPolicy
ExecutionShape
```

이들은 서로 다른 축이다.

예:

```text
TaskClass = TOPOLOGY
CostClass = HEAVY
SpatialGroup = BrickRegion17
ExecutionShape = BLOCK
```

이렇게 분리하면 “task type = cost = worker shape”로 과도하게 결합되는 것을 막을 수 있다.

### 4.33 프로파일링에서 봐야 할 지표

Divergence-aware scheduler는 단순 warp efficiency 하나만 보면 안 된다.

핵심 metrics:

- branch efficiency
- warp execution efficiency
- active lanes / warp
- average divergent paths / warp
- task-class distribution
- queue depth per class
- queue fragmentation
- local same-class hit ratio
- cross-class fallback ratio
- steal success by class
- tasks per batch
- scheduler operations / task
- task descriptor bytes
- average/p95 task runtime
- p95/p99 worker idle time
- task split count
- task merge/coalesce count
- duplicate work eliminated
- L1/L2 hit rate
- memory transactions / warp
- register usage
- occupancy
- useful work / scheduler overhead
- frame tail duration

특히 다음 네 값을 함께 봐야 한다.

```text
Warp Execution Efficiency
Queue Depth per Class
Scheduler Overhead
Long-Tail Duration
```

Warp efficiency는 올랐는데 class queue가 얕아 long tail이 커졌다면 partition이 과도한 것이다.

---

## 5. 내 관심 분야와 연결

### 5.1 Semiconductor SDF Meshing

사용자의 dynamic geometry pipeline에서는 오늘 개념이 매우 직접적이다.

```text
Dense/Sparse SDF
    ↓
Active Brick Detection
    ↓
Task Classification
    ↓
EMPTY / SIMPLE / COMPLEX / TOPOLOGY
    ↓
Class-Aware Local Queues
    ↓
Task Coalescing
    ↓
Adaptive Grain
    ↓
Mesh / Meshlet Output
```

같은 brick count라도 topology complexity가 다르므로 static partition보다 task classification이 중요할 수 있다.

### 5.2 ColumnStack / Thin-Layer Geometry

Thin film과 stacked material 구조에서는 대부분 영역이 규칙적이지만 특정 trench/gate/spacer 주변에서만 complexity가 급격히 증가할 수 있다.

이런 workload에서는:

- regular layer update → coarse batch
- high-curvature / transition zone → finer grain
- topology repair → dedicated class

가 자연스러운 decomposition이다.

즉 geometry representation의 locality가 scheduler policy까지 내려온다.

### 5.3 Marching Cubes / Dual Contouring

Marching Cubes에서는 active-cell count나 case complexity를 cost proxy로 사용할 수 있다.

Dual Contouring에서는:

- Hermite sample 수
- QEF complexity
- multi-component cell
- adaptive transition

같은 값이 execution signature가 될 수 있다.

Task classifier가 geometry algorithm 내부 의미를 이해할수록 좋은 scheduling이 가능하다.

### 5.4 CFD / Scientific Visualization

CFD timestep에서도 다음 작업은 서로 다르다.

- regular cell update
- boundary condition
- refinement/coarsening
- iso-surface extraction
- high-gradient region

GPU runtime이 task class를 coarse하게 분리하면 regular bulk와 irregular boundary가 같은 warp에 섞이는 문제를 줄일 수 있다.

다만 CFD는 bandwidth-bound인 경우가 많으므로 execution coherence보다 spatial locality가 더 중요한 workload도 있다.

### 5.5 GPU-Driven Rendering

Rendering 쪽에서는 task grain이 작기 때문에 오늘의 아이디어를 더 보수적으로 적용해야 한다.

Meshlet 하나당 일이 매우 적으면 queue 분류 비용이 meshlet culling보다 더 비쌀 수 있다.

따라서 rendering에서는:

- coarse chunk-level classification
- material/depth binning
- existing worklist metadata 재사용

처럼 이미 필요한 classification과 결합하는 편이 좋다.

### 5.6 Graphics Engineer Career

이 주제는 다음 영역을 한 번에 연결한다.

- SIMT execution
- warp divergence
- persistent task runtime
- dynamic geometry
- scheduler cost model
- memory locality
- queue design
- profiler-driven optimization

Graphics/game engine 면접에서 강한 설명은 단순히:

> “Divergence를 줄이기 위해 branch를 없앴다.”

가 아니다.

더 강한 설명은:

> **“Irregular work에서 branch 자체보다 heterogeneous task placement가 원인이면, execution-path queueing으로 같은 path를 warp에 모을 수 있습니다. 다만 class를 너무 많이 만들면 queue fragmentation이 생기므로 adaptive grain과 same-class-first stealing을 함께 보고, warp efficiency와 queue depth를 동시에 프로파일링해야 합니다.”**

이다.

---

## 6. 머릿속에 남길 질문 3개

1. **Warp execution efficiency는 높아졌지만 frame tail이 길어졌다면 execution-path queue partition이 parallel slack을 너무 줄였는지 어떤 지표로 확인할 수 있을까?**
2. **Dynamic SDF meshing에서 task class를 `SIMPLE / COMPLEX / TOPOLOGY`로 나누는 것과 spatial Morton group으로 나누는 것은 각각 control-flow locality와 memory locality 중 무엇을 최적화하며, 둘이 충돌하면 무엇을 우선해야 할까?**
3. **Task coalescing으로 scheduler overhead를 줄일 때 batch가 너무 커져 load imbalance가 다시 생기지 않도록 queue depth, worker count, task-cost distribution을 어떤 관계로 봐야 할까?**

---

## 7. graphics engineer 면접 질문 1개와 답변

### 질문

**“GPU task runtime에서 warp divergence를 줄이기 위해 task type별로 queue를 하나씩 만들면 항상 더 빠르지 않나요?”**

### 답변

항상 그렇지는 않다.

Task queue partition은 같은 warp에 비슷한 execution path의 task를 모아 control-flow divergence를 줄일 수 있다. 특히 path가 길고 loop count나 memory access pattern이 크게 다른 irregular task에서는 효과가 클 수 있다.

하지만 queue를 너무 많이 나누면 각 queue의 depth가 얕아지고 ready work가 여러 작은 pool로 fragmentation된다. 그러면 GPU 전체에는 충분한 work가 있어도 특정 class worker가 idle해질 수 있다.

또 다음 비용도 추가된다.

- task classification
- queue metadata
- queue probing
- class-aware stealing
- termination detection
- smaller batching opportunity

그래서 production runtime에서는 보통:

1. 실행 경로 차이가 큰 coarse class만 분리하고,
2. 같은 class의 tiny task는 coalesce하며,
3. queue depth가 부족하면 compatible class 또는 broader stealing으로 fallback하고,
4. warp efficiency와 queue depth, scheduler overhead, p99 tail을 함께 측정한다.

짧은 branch라면 compiler predication이 더 싸게 해결할 수도 있기 때문에 queue partition 자체가 과잉 최적화일 수 있다.

핵심은:

> **Divergence-aware queueing은 branch를 제거하는 기술이 아니라, task placement를 조절해 SIMT execution coherence를 높이는 scheduler optimization이며, parallelism fragmentation이라는 반대 비용을 항상 함께 가진다.**

---

## 8. 포트폴리오 / 커리어 연결

이 주제는 graphics engineer 포트폴리오에서 **GPU kernel optimization을 runtime scheduling 문제까지 확장해서 설명하기** 좋다.

### SIMT / CUDA

- warp divergence
- branch predication
- independent thread scheduling
- active lanes
- register pressure

### GPU Runtime

- execution-path queue
- EPAQ
- local queue / work stealing
- same-class-first steal
- compatible class fallback
- queue fragmentation

### Adaptive Scheduling

- task coalescing
- split / merge
- grain-size control
- cost class
- parallel slack

### Geometry / Simulation

- sparse SDF bricks
- Marching Cubes
- Dual Contouring
- topology repair
- adaptive transition
- CFD boundary/refinement work

### Memory Layout

- SoA scheduler metadata
- spatial key
- execution signature
- compact logical task handle
- late payload resolve

### Profiling

- warp execution efficiency
- branch efficiency
- queue depth per class
- L1/L2 hit rate
- scheduler overhead
- p99 tail

포트폴리오에서는 다음처럼 설명할 수 있다.

> **“Irregular SDF meshing task를 execution signature로 coarse partition해 같은 warp에 비슷한 code path를 배치하고, 동일 class의 tiny task는 spatial key 기준으로 coalesce했습니다. Queue depth가 줄어 parallel slack이 부족해지는 경우에는 grain을 다시 줄이고 compatible-class stealing을 허용했습니다. 최적화 평가는 branch efficiency만 보지 않고 class별 queue depth, scheduler overhead, L2 locality, p99 tail time을 함께 사용했습니다.”**

이 설명은 GPU architecture와 geometry algorithm, runtime scheduler, memory layout을 연결한다.

---

## 9. 내일 이어서 볼 개념

**GPU Termination Detection and Quiescence: Distributed Counters, In-Flight Work, and Safe Snapshot Publication**

오늘은 task를 잘 분류하고 적절한 grain으로 배치해 warp efficiency와 load balance를 맞추는 문제를 봤다.

다음 질문은:

> **Local queue와 work stealing이 있는 distributed GPU runtime에서 “정말 모든 work가 끝났다”는 것을 어떻게 판단하는가?**

이다.

학습 흐름은 다음과 같다.

```text
Global Queue
    ↓
Backpressure / Forward Progress
    ↓
Local Queues + Work Stealing
    ↓
Divergence-Aware Task Partition
    ↓
Distributed Termination Detection
    ↓
Safe Geometry Snapshot Publish
```

다음 노트에서는:

- local queue empty와 global termination의 차이
- in-flight producer
- outstanding-work counter
- publication race
- epoch barrier
- distributed quiescence
- snapshot publish
- CUDA → Vulkan handoff
- geometry generation complete의 정확한 의미

를 중심으로 본다.

---

## 10. 참고 키워드

- Warp Divergence
- Control-Flow Divergence
- Work-Length Divergence
- SIMT Execution
- Active Lanes
- Branch Efficiency
- Warp Execution Efficiency
- Execution-Path-Aware Queueing (EPAQ)
- Divergence-Aware Scheduling
- Task Classification
- Execution Signature
- Task Coalescing
- Adaptive Grain Size
- Parallel Slack
- Task Split / Merge
- Queue Fragmentation
- Same-Class-First Stealing
- Compatible Task Class
- Spatial Locality
- Memory Coalescing
- Morton / Spatial Key
- Cost Prediction
- Cost History
- SoA Scheduler Metadata
- Persistent Threads
- GPU Work Stealing
- Dynamic SDF
- Marching Cubes
- Dual Contouring
- Topology Repair
- CFD Adaptive Work
- Yuki Maeda, Kenjiro Taura, **“GTaP: A GPU-Resident Fork-Join Task-Parallel Runtime with a Pragma-Based Interface,” 2026**
  - https://arxiv.org/abs/2604.05982
- NVIDIA, **CUDA C++ Best Practices Guide — Control Flow / Branching and Divergence**
  - https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/
- NVIDIA, **CUDA Programming Guide — SIMT Execution Model**
  - https://docs.nvidia.com/cuda/cuda-programming-guide/03-advanced/advanced-kernel-programming.html
- NVIDIA GTC, **“The Rocky Road to Tasking: Task Queues Reloaded”**
  - https://developer.nvidia.com/gtc/2020/video/s21189-vid
- Albert Segura, Jose-Maria Arnau, Antonio Gonzalez, **“Irregular Accesses Reorder Unit: Improving GPGPU Memory Coalescing for Graph-Based Workloads”**
  - https://arxiv.org/abs/2007.07131
