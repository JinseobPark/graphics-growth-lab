---
title: "Local GPU Queues and Work Stealing: Per-Block Deques, Load Balancing, and Divergence-Aware Task Scheduling"
date: "2026-09-14"
category: Graphics
tags: [GPU, Rendering, GPU-Driven Rendering, Persistent Threads, Work Stealing, Local Queue, Deque, Load Balancing, Task Scheduling, Warp Divergence, CUDA, Work Graph, Dynamic Geometry, Memory Layout, C++]
level: intermediate
---

# [Daily Graphics Growth] 2026-09-14 - Local GPU Queues and Work Stealing: Per-Block Deques, Load Balancing, and Divergence-Aware Task Scheduling

## 1. 오늘의 개념

어제는 **GPU Queue Backpressure and Forward Progress**에서 bounded global queue가 가득 찼을 때 단순 spin retry가 GPU residency를 점유해 consumer의 실행을 막을 수 있고, queue correctness와 scheduler progress를 별도로 설계해야 한다는 점을 봤다.

오늘은 그 다음 병목으로 이동한다.

Global queue 하나가 correctness 면에서 안전하더라도 모든 worker가 같은 `head/tail`과 같은 cache line을 두드리면 다음 문제가 생긴다.

```text
Worker 0 ─┐
Worker 1 ─┼─> Global Queue
Worker 2 ─┼─> head / tail atomics
...       ─┘
```

Work가 많아질수록 queue 자체가 계산보다 비싼 중앙 병목이 될 수 있다.

그래서 irregular GPU runtime에서는 흔히 다음 구조를 생각한다.

```text
Worker / Block 0 → Local Deque 0
Worker / Block 1 → Local Deque 1
Worker / Block 2 → Local Deque 2
                         ↑
                  idle worker steals
```

각 worker는 자기 local queue에서 대부분의 task를 싸게 가져오고, local work가 떨어졌을 때만 다른 worker의 queue에 접근해 **work stealing**을 수행한다.

이 구조의 핵심은 단순히 queue를 여러 개 만드는 것이 아니다.

오늘은 다음 네 가지를 하나의 scheduling problem으로 본다.

1. **Locality**  
   Owner가 자기 queue에서 task를 처리할 때 atomic/contention과 data movement를 줄일 수 있는가.

2. **Load Balance**  
   특정 brick/meshlet/task group에 일이 몰렸을 때 idle worker가 실제로 그 work를 가져올 수 있는가.

3. **Deque Semantics**  
   Owner와 thief가 같은 end를 두드리지 않도록 ownership pattern을 설계할 수 있는가.

4. **Execution-Path Coherence**  
   서로 다른 control-flow의 task를 한 warp에 섞어서 load balance는 좋아졌지만 divergence가 커지는 역효과를 어떻게 막는가.

특히 2026년의 **GTaP**은 GPU-resident fork-join runtime에서 global queue보다 work stealing이 더 높은 scalability를 보이도록 설계했고, thread-level worker에서는 **Execution-Path-Aware Queueing(EPAQ)** 으로 서로 다른 execution path의 task를 분리해 warp divergence를 줄이는 방향을 제시한다.

또 현재 CUDA Programming Guide는 Blackwell(compute capability 10.0)의 **Cluster Launch Control**을 이용한 work stealing을 별도로 설명한다. 이것은 application-level deque에서 arbitrary task를 훔치는 것과 동일한 메커니즘은 아니지만, irregular workload의 tail을 줄이기 위해 idle block이 아직 시작되지 않은 block의 launch work를 가져온다는 점에서 중요한 최신 hardware-level 비교 대상이다.

---

## 2. 한 줄 핵심

> GPU work stealing의 핵심은 **대부분의 scheduling을 owner-local queue에서 처리해 global atomic hot spot을 피하고, idle worker만 드물게 remote steal을 수행하되, task type·execution path·memory locality를 무시한 과도한 stealing이 warp divergence와 cache miss로 다시 비용을 만들지 않도록 하는 것**이다.

---

## 3. 왜 중요한가

Irregular workload는 “총 work 양”보다 **work가 worker 사이에 얼마나 불균등하게 분포되는가**가 성능을 결정할 때가 많다.

예를 들어 80개의 worker block이 있고 총 80,000개의 task가 있다고 하자. 평균은 block당 1,000개다.

하지만 실제 분포가:

```text
10 blocks → 5,000 tasks
40 blocks →   700 tasks
30 blocks →   100 tasks
```

라면 뒤의 30 block은 일찍 idle 상태가 되고, 일부 block만 긴 tail을 처리하게 된다.

GPU throughput 관점에서는 다음과 같은 **long tail**이 생긴다.

```text
Active Workers
████████████████████
████████████████
████████
██
█
time →
```

Global queue 하나를 사용하면 load balance는 비교적 쉽다. 모든 worker가 같은 곳에서 work를 가져가면 되기 때문이다.

하지만 work item이 작고 worker 수가 많아질수록:

- global atomic contention
- cache-line serialization
- head/tail retry
- queue metadata traffic
- scheduler overhead

가 커진다.

반대로 완전히 local queue만 사용하면 contention은 낮지만 workload가 한 worker에 몰렸을 때 다른 worker가 도와줄 수 없다.

Work stealing은 이 둘의 중간점을 노린다.

```text
Common case:
local pop → cheap

Imbalance case:
remote steal → expensive but rare
```

즉 work stealing의 좋은 상태는 **steal을 많이 하는 시스템**이 아니다.

> **Local hit가 대부분이고, imbalance가 발생했을 때만 steal이 작동하는 시스템**이 좋은 상태에 가깝다.

### 3.1 최신 GPU에서도 load imbalance는 하드웨어 scheduler만으로 완전히 해결되지 않는다

CUDA의 최신 문서는 variable runtime의 block workload에서 fixed-work-per-block 방식은 scheduler가 block을 자연스럽게 분배해 load balance를 얻는 장점이 있지만, fixed number of persistent blocks는 launch overhead를 줄이는 대신 imbalance가 생길 수 있다고 설명한다.

Blackwell의 **Cluster Launch Control**은 이 간극을 줄이기 위해 block이 아직 실행되지 않은 다른 block/cluster의 launch를 취소하고 그 index의 work를 가져오는 형태의 work stealing을 제공한다.

이것은 다음 사실을 보여준다.

> **Irregular load balance는 software runtime만의 오래된 문제가 아니라 최신 GPU execution model에서도 중요한 scheduling 문제다.**

---

## 4. 구현 관점

### 4.1 Global Queue의 장점과 한계

가장 단순한 persistent scheduler:

```text
while (true) {
    task = globalQueue.pop()
    if task:
        process(task)
}
```

장점:

- load balance가 자연스럽다.
- task가 어디서 만들어졌는지 몰라도 된다.
- global ordering/debug가 단순하다.

단점:

- 모든 worker가 같은 metadata를 갱신
- atomic hot spot
- L2 line contention
- 작은 task에서 scheduling cost 비율 증가
- high fan-out에서 enqueue burst 집중

특히 task가 수 마이크로초 이하의 fine-grained work라면 queue operation 자체가 유의미한 비율을 차지할 수 있다.

### 4.2 Local Queue는 common path를 싸게 만든다

Local queue architecture:

```text
Worker i
    ↓
LocalQueue[i]
```

Task가 같은 worker에서 생성되고 소비될 확률이 높다면 local queue는 다음 이점을 가진다.

- global head/tail contention 감소
- ownership reasoning 단순화
- 최근 생성된 task와 관련 data의 cache locality 가능성
- local task ordering policy 적용 가능

하지만 “local”의 정확한 의미를 구분해야 한다.

- **Logical local queue:** 특정 worker만 owner로 push/pop하지만 storage는 global memory
- **Shared-memory local cache:** block 내부에서만 접근
- **Cluster-visible queue:** thread-block cluster 범위에서 Distributed Shared Memory 등을 이용할 수 있는 구조
- **Global stealable deque:** 다른 worker가 thief로 접근 가능하도록 global-visible storage 사용

Arbitrary block이 훔쳐야 하는 task를 block private shared memory에만 두면 steal할 수 없다.

### 4.3 Work-Stealing Deque의 기본 패턴

고전적인 work stealing은 **double-ended queue(deque)** 를 사용한다.

Owner:

```text
push bottom
pop bottom
```

Thief:

```text
steal top
```

즉 owner와 thief가 주로 서로 다른 end에 접근한다.

이 구조의 목적은 단순 API 편의가 아니다.

- owner의 common path는 tail/bottom 중심
- remote thief는 head/top 중심
- 같은 atomic location에 대한 contention을 줄일 수 있음

Task runtime에서는 흔히 owner 쪽이 **LIFO**에 가까운 동작을 하고 thief가 오래된 task를 가져가는 구조가 locality와 parallel slack을 동시에 얻는 데 유리하다.

### 4.4 왜 Owner는 LIFO, Thief는 오래된 work를 훔치는가

재귀적/분할형 work를 생각하자.

```text
Task A
 ├─ A1
 │   ├─ A1a
 │   └─ A1b
 └─ A2
```

Owner가 방금 생성한 A1b를 바로 처리하면:

- 관련 state가 register/cache에 남아 있을 가능성이 높고
- working set locality가 좋다.

반면 thief가 오래된 A2 같은 task를 훔치면:

- 상대적으로 큰 독립 subtree일 가능성이 높고
- thief가 충분한 work를 오래 처리할 가능성이 높다.

즉:

```text
Owner LIFO → locality
Thief old task → parallelism
```

이라는 직관이 있다.

GPU geometry workload에서도 동일하게 해석할 수 있다.

최근에 subdivision된 neighboring cell/brick work는 owner가 계속 처리하고,
idle worker는 비교적 큰 independent brick/chunk를 훔치는 편이 locality에 유리할 수 있다.

### 4.5 Per-Block Queue가 자연스러운 이유

GPU에서 worker granularity를 thread 하나로 잡으면 queue access와 task execution control이 매우 복잡해질 수 있다.

Per-block worker에서는:

```text
1 block = 1 scheduler worker
```

로 보고 block 내부 threads가 한 task를 협력해서 처리할 수 있다.

장점:

- task 단위가 mesh brick / BVH node batch처럼 block-sized work와 잘 맞음
- queue operation을 lane/thread 0 하나가 담당 가능
- block 내부 collaboration 가능
- per-block local metadata 관리가 단순

역사적인 GPU work-stealing runtime도 worker block마다 deque를 두고, local task가 없을 때 다른 worker의 deque에서 task를 steal하는 구조를 사용했다.

### 4.6 Thread-Level Worker는 load balance가 더 세밀하지만 divergence 위험이 커진다

Thread 단위 worker는 fine-grained task를 매우 유연하게 처리할 수 있다.

하지만 한 warp에 다음 task가 섞일 수 있다.

```text
lane 0 → BVH traversal
lane 1 → meshlet classification
lane 2 → topology repair
lane 3 → short no-op
...
```

SIMT에서는 서로 다른 execution path가 순차적으로 실행될 수 있어 utilization이 낮아진다.

따라서 thread-level scheduling에서는 단순 “idle lane에 아무 task나 준다”가 최적이 아니다.

이것이 **Execution-Path-Aware Queueing(EPAQ)** 같은 아이디어가 중요한 이유다.

### 4.7 Execution-Path-Aware Queueing

Task를 execution characteristic에 따라 분리한다.

예:

```text
Queue 0 → field update
Queue 1 → surface extraction
Queue 2 → topology repair
Queue 3 → visibility test
```

또는 같은 task type 안에서도 branch characteristic을 나눌 수 있다.

목적:

```text
같은 warp
→ 비슷한 control flow
→ branch divergence 감소
```

GTaP은 thread-level workers에서 user-defined criterion으로 task queue를 partition하는 EPAQ를 제시하며, 효과는 workload dependent지만 일부 benchmark에서 divergence 감소에 따른 성능 향상을 보여준다.

중요한 것은 queue가 많을수록 무조건 좋지 않다는 점이다.

너무 세밀하게 분리하면:

- 각 queue에 task가 부족해짐
- load balance가 악화
- stealing probe 증가
- metadata 증가

가 생긴다.

즉:

> **Execution coherence와 available parallelism 사이의 trade-off**다.

### 4.8 Locality Queue와 Execution-Path Queue는 서로 다른 축이다

다음 두 질문을 분리한다.

1. **어디에 있는 task인가?**
   - worker-local
   - neighboring spatial region
   - remote victim

2. **어떤 code path의 task인가?**
   - meshing
   - culling
   - topology
   - shading classification

고급 scheduler에서는 conceptually 2D structure가 가능하다.

```text
Worker 0:
  MeshingQueue
  TopologyQueue
  VisibilityQueue

Worker 1:
  MeshingQueue
  TopologyQueue
  VisibilityQueue
```

하지만 실제 queue 수가 폭발할 수 있으므로 full matrix를 만들기보다 small number of task classes + local ownership을 사용하는 편이 현실적이다.

### 4.9 Victim Selection은 Steal 비용을 결정한다

Idle worker가 누구에게서 steal할지 선택해야 한다.

가능한 policy:

- round-robin
- pseudo-random
- last-success victim 재시도
- queue-length hint 기반
- spatially nearby victim
- same task-class 우선

Random victim은 centralized directory 없이 단순하게 분산할 수 있다.

하지만 queue 대부분이 empty라면 random probing이 반복되어 비용이 커진다.

따라서 victim selection 자체도 profiler 대상이다.

### 4.10 Queue-Length Hint는 정확하지 않아도 된다

모든 victim queue length를 정확히 추적하면 그 metadata 갱신이 또 global bottleneck이 된다.

Scheduling hint는 stale해도 correctness에 영향을 주지 않아야 한다.

예:

```text
approximateLoad[worker]
```

를 낮은 빈도로 업데이트하고 steal target 우선순위에만 사용한다.

틀린 hint의 결과는:

```text
steal miss → 다른 victim 시도
```

여야 한다.

> **Correctness state와 scheduling hint를 분리하는 것**이 중요하다.

### 4.11 Single-Task Steal vs Batch Steal

Steal operation은 remote atomic과 metadata access가 필요하다.

Task 하나만 훔치면 곧 다시 steal해야 할 수 있다.

그래서 여러 task를 batch로 훔칠 수 있다.

```text
steal N tasks
```

장점:

- steal overhead amortization
- victim probing 감소
- thief가 한동안 local execution 가능

단점:

- victim의 locality를 과도하게 뺏을 수 있음
- load가 반대로 불균형해질 수 있음
- task type이 다양하면 thief warp divergence 증가

따라서 batch size는 workload granularity와 task duration에 의존한다.

### 4.12 Half-Steal과 Fixed-Batch는 다른 trade-off다

CPU work-stealing runtime에서 queue의 일부를 훔치는 정책을 생각할 수 있다.

GPU에서도 conceptually:

```text
steal = min(fixedBatch, victimCount / 2)
```

같은 방식이 가능하다.

큰 subtree/task set에서는 half-steal이 빠르게 균형을 맞출 수 있지만,
fine-grained queue에서는 과도한 metadata movement가 될 수 있다.

Graphics workload에서는 task payload를 복사하는 대신 **logical task ID range** 또는 compact descriptor를 steal하는 편이 유리하다.

### 4.13 Queue Payload는 작아야 한다

Queue item이 다음처럼 크다면:

```text
full meshlet descriptor
bounds
material
addresses
debug info
```

steal 자체가 memory bandwidth 작업이 된다.

더 좋은 방향:

```text
TaskRecord {
    logicalHandle
    localIndex
    taskClass
    generation
    epoch
}
```

실제 geometry/material data는 persistent table에서 late resolve한다.

이전 노트의 원칙이 다시 등장한다.

```text
Queue = identity / intent
Table = placement / payload
```

### 4.14 Owner-Only Tail과 Thief-Shared Head

Deque의 scalability는 access pattern에서 나온다.

개념적으로:

```text
tail/bottom:
mostly owner

head/top:
thieves + boundary case
```

Queue가 충분히 차 있을 때 owner local push/pop은 remote contention 없이 진행될 수 있다.

문제는 queue size가 1개 정도 남는 **last-item race**다.

Owner pop과 thief steal이 같은 마지막 task를 동시에 가져가지 않도록 atomic compare/exchange 같은 arbitration이 필요하다.

즉 common path는 싸게,
rare contention path에만 강한 atomic을 쓰는 것이 핵심이다.

### 4.15 ABA / Generation 문제는 Deque Head에서도 나타난다

Ring deque의 head index가 wrap되면 동일 physical value가 다시 나타날 수 있다.

어제의 sequence ticket 원칙을 그대로 적용할 수 있다.

```text
head = {index, generation}
```

Thief가 old head snapshot으로 CAS를 시도했는데 queue가 한 바퀴 돌아 같은 index가 된 경우 generation이 다르면 stale observation을 구별할 수 있다.

Work stealing도 결국 **lifetime/versioning problem**을 포함한다.

### 4.16 Local Queue Overflow는 Global Spill로 처리할 수 있다

Per-worker deque를 무한하게 만들 수는 없다.

Local queue가 full이면:

```text
Local Queue
   ↓ full
Global Spill / Overflow Stream
```

으로 일부 work를 이동할 수 있다.

이 global path는 common path가 아니므로 global atomic contention을 허용할 수 있다.

좋은 hierarchical design은:

```text
Fast common path → local
Rare imbalance   → steal
Rare overflow    → global spill
```

처럼 비용이 높은 mechanism을 rare path로 밀어낸다.

### 4.17 Global Ingress Queue는 여전히 유용하다

Host 또는 upstream pass에서 새 work를 시작할 때 처음부터 특정 worker를 고르기 어려울 수 있다.

그래서:

```text
Global Ingress
    ↓ distribute
Per-Worker Local Queues
```

구조를 사용할 수 있다.

Global queue를 완전히 제거하는 것이 목표가 아니라,

> **모든 task operation이 global queue를 통과하지 않게 만드는 것**

이 핵심이다.

### 4.18 Work Sharing과 Work Stealing을 혼합할 수 있다

두 개념:

**Work Sharing**
- busy producer가 다른 worker에게 task를 push

**Work Stealing**
- idle worker가 busy victim에서 task를 pull

Work sharing은 imbalance를 일찍 분산할 수 있지만 producer가 distribution cost를 지불한다.

Work stealing은 common path를 local하게 유지하지만 idle이 발생한 뒤 반응한다.

Irregular graphics runtime에서는:

```text
large initial fan-out → sharing
fine-grained dynamic imbalance → stealing
```

처럼 hybrid가 가능하다.

### 4.19 Task Granularity가 Scheduler 성능을 결정한다

Task가 너무 작으면:

```text
queue overhead
steal overhead
generation validation
task dispatch
>
useful computation
```

이 된다.

Task가 너무 크면:

```text
few tasks
→ steal opportunity 부족
→ long tail
```

이 된다.

즉 work stealing에는 적당한 **parallel slack**이 필요하다.

좋은 task size는:

- 충분히 오래 실행되어 scheduler overhead를 amortize하고
- worker 수보다 충분히 많은 task를 만들어 load balance 여지를 주는

중간 영역이다.

### 4.20 Adaptive Grain Size

Dynamic geometry에서는 task cost를 사전에 완벽하게 알기 어렵다.

하지만 다음 metadata는 cost proxy가 될 수 있다.

- active voxel/cell count
- surface crossings
- topology component count
- estimated meshlet count
- refinement level
- projected visibility
- previous execution time class

큰 task는 split하고,
작은 neighboring task는 coalesce하는 adaptive grain-size policy를 둘 수 있다.

Work stealing과 task granularity는 독립적이지 않다.

> **Scheduler가 훔칠 수 있는 좋은 단위를 geometry system이 만들어줘야 한다.**

### 4.21 Spatial Locality와 Steal의 충돌

Brick A가 처리한 직후 neighboring brick B를 처리하면 다음 data가 cache에 남아 있을 수 있다.

- SDF page
- material table
- boundary halo
- mesh allocator metadata

Remote worker가 B를 steal하면 locality를 잃을 수 있다.

그래서 steal target을 purely random하게 고르기보다:

```text
same spatial partition
same memory page group
same task class
```

를 우선하는 topology-aware policy를 고려할 수 있다.

하지만 너무 locality를 고집하면 load balance가 나빠질 수 있다.

### 4.22 “좋은 Steal”은 충분히 큰 Work와 낮은 Migration Cost를 가진다

개념적 steal value:

```text
Expected Idle Time Saved
------------------------
Remote Queue Cost
+ Data Locality Loss
+ Divergence Risk
```

Steal할 task가 100ns짜리 작업인데 remote metadata 접근이 더 비싸다면 훔치지 않는 편이 낫다.

즉 idle worker라고 해서 무조건 remote work를 가져오는 것이 최적은 아니다.

### 4.23 Backoff는 Empty-System에서 Probe Storm을 줄인다

Work가 거의 끝났을 때 worker 대부분이 idle이 되고 서로의 empty queue를 반복 probe할 수 있다.

```text
steal fail
→ immediate next victim
→ fail
→ repeat
```

이 상태는 useful work 없이 global memory traffic을 만든다.

가능한 policy:

- exponential / bounded backoff
- probe 횟수 제한
- global remaining-work counter
- epoch-based sleep/exit state
- new-work notification generation

Persistent runtime에서는 termination detection과 연결된다.

### 4.24 Termination Detection은 Local Empty보다 어렵다

내 local queue가 비었다고 전체 computation이 끝난 것은 아니다.

다른 worker가:

- 아직 task 실행 중
- child task 생성 예정
- spill queue 보유
- in-flight steal 중

일 수 있다.

따라서 종료 조건은 conceptually:

```text
all queues empty
AND
no worker executing producer-capable work
AND
no in-flight publication
```

같은 global quiescence를 의미해야 한다.

Work stealing runtime에서 load balance보다 종료 detection이 더 어려운 경우도 있다.

Graphics pipeline에서는 kernel boundary나 stage counter를 이용해 이 문제를 단순화하는 선택도 충분히 합리적이다.

### 4.25 Warp-Level Stealing과 Block-Level Stealing

Steal granularity도 선택할 수 있다.

#### Block-Level Worker
- 한 block이 victim 선택
- block 전체가 stolen task를 협력 처리
- scheduler metadata가 작음

#### Warp-Level Worker
- block 안 여러 warp가 독립 worker처럼 행동
- finer load balance
- shared state/queue 수 증가
- task shape가 warp-sized일 때 유리

#### Thread-Level Worker
- 가장 fine-grained
- maximum scheduling flexibility
- divergence와 queue overhead 위험 최대

GPU task runtime은 task execution shape와 scheduling granularity를 맞추는 것이 중요하다.

### 4.26 Divergence-Aware Queue Partition의 과분할 문제

Execution path별 queue:

```text
Type A
Type B
Type C
...
```

를 늘리면 warp coherence는 좋아질 수 있다.

하지만 각 queue의 평균 depth가 너무 낮아지면:

- parallelism fragment
- steal miss 증가
- worker idle 증가

가 생긴다.

따라서 queue partition 수는 다음을 함께 봐야 한다.

```text
branch coherence gain
vs
queue occupancy / available parallelism loss
```

GTaP의 EPAQ 성능이 workload-dependent라는 점도 이 trade-off를 보여준다.

### 4.27 Task Class는 Shader/Kernel 실행 경로와 맞춘다

Graphics work graph에서 유용한 task class 예:

```text
CLASS_FIELD
CLASS_MESH_SIMPLE
CLASS_MESH_TOPOLOGY
CLASS_CULL
CLASS_RENDER_PACKET
```

`CLASS_MESH_SIMPLE`과 `CLASS_MESH_TOPOLOGY`를 분리하면 같은 warp에서 simple surface extraction과 complex topology repair가 섞이는 것을 줄일 수 있다.

반대로 material ID마다 queue를 만드는 수준으로 지나치게 세분화하면 parallelism이 깨질 수 있다.

Task class는 **실제 instruction-path 차이가 큰 경계**에 두는 편이 좋다.

### 4.28 Hardware-Assisted Work Stealing: Cluster Launch Control

현재 CUDA의 **Cluster Launch Control**은 Blackwell GPU(compute capability 10.0)에서 thread block이 아직 시작되지 않은 다른 block 또는 cluster의 launch를 cancel하고 그 work index를 가져올 수 있게 한다.

이것은 software deque work stealing과 다르다.

#### Software Deque

```text
arbitrary application task record
local worker deque
remote thief
```

#### Cluster Launch Control

```text
kernel launch가 이미 정의한 block work
unstarted block/cluster index
running block이 cancellation 후 그 index를 처리
```

장점은 GPU scheduler에 남아 있는 unlaunched work를 이용해 low-tail imbalance를 줄일 수 있다는 점이다.

하지만 runtime-generated arbitrary child task queue를 대체하는 기능은 아니다.

두 mechanism은 **서로 다른 scheduling layer**다.

### 4.29 GTaP 2026의 의미

2026년 공개된 GTaP은 GPU-resident fork-join runtime을 persistent kernel 위에 구성하고, block-level과 thread-level worker를 지원한다.

핵심 포인트:

- fork-join을 continuation/state-machine 형태로 표현
- global queue보다 work stealing으로 scalability 개선
- thread-level worker에서 EPAQ로 execution path 분리
- EPAQ 효과는 workload-dependent
- irregular task parallelism을 CPU round-trip 없이 GPU 내부에서 처리

Graphics engineer 관점에서 중요한 점은 특정 runtime을 쓰는 것이 아니라:

> **global contention → local queue → steal → divergence-aware partition**

이라는 scheduler evolution을 실제 2026 연구에서도 확인할 수 있다는 것이다.

### 4.30 Work Stealing을 Vulkan Rendering에 직접 넣는다는 의미는 아니다

Vulkan graphics pipeline 자체가 per-block deque scheduler API를 제공하는 것은 아니다.

이 개념은 주로:

- CUDA compute
- custom GPU runtime
- compute shader persistent scheduling
- geometry preprocessing
- simulation
- visibility work generation

같은 영역에 적용된다.

결과는 이후:

```text
Render Packet
→ Indirect Draw / Mesh Tasks
→ Vulkan Graphics
```

로 넘어갈 수 있다.

즉 work stealing은 rendering API 기능이라기보다 **rendering work를 준비하는 GPU compute runtime architecture**다.

### 4.31 CUDA → Vulkan Pipeline에서 Local Queue의 경계

사용자의 관심 pipeline을 생각하면:

```text
CUDA / Warp:
SDF update
meshing
meshlet build
work stealing runtime
        ↓
Published Geometry Snapshot
        ↓ external synchronization
Vulkan:
visibility
indirect / DGC
render
```

처럼 compute-side irregular graph에서 work stealing을 사용하고, renderer에는 compact하고 deterministic한 snapshot을 publish하는 구조가 좋다.

Rendering consumer까지 같은 persistent task runtime으로 묶으면 synchronization과 debugging이 지나치게 복잡해질 수 있다.

즉 **irregular domain 내부에서는 dynamic scheduling, subsystem boundary에서는 snapshot publication**이라는 분리가 실무적으로 강하다.

### 4.32 Profiler에서 볼 지표

Work-stealing runtime의 핵심 metrics:

- local queue push/pop count
- local-hit ratio
- steal attempts
- successful steals
- steal success ratio
- tasks stolen per steal
- victim probe count
- victim distribution entropy
- deque occupancy histogram
- local queue overflow/spill count
- global ingress traffic
- global atomic operations / task
- thief-side CAS retry
- idle worker ratio
- p50/p95/p99 task time
- p95/p99 queue wait time
- task-class distribution
- branch efficiency / warp divergence
- warp execution efficiency
- L1/L2 hit rate
- task payload bytes
- scheduling overhead / useful work
- long-tail duration
- time from first worker idle → global completion

특히 다음 네 값을 함께 본다.

```text
Local-Hit Ratio
Steal Success Ratio
Warp Execution Efficiency
Long-Tail Duration
```

Local-hit이 높은데 long tail이 길면 steal policy가 약한 것이고,
steal success는 높은데 warp efficiency가 떨어지면 heterogeneous task를 너무 공격적으로 섞고 있을 가능성이 있다.

---

## 5. 내 관심 분야와 연결

### 5.1 Dynamic SDF / Level-Set Meshing

Sparse SDF brick workload는 work stealing과 매우 잘 맞는 irregular pattern이다.

```text
Brick A: no surface      → very cheap
Brick B: flat surface    → cheap
Brick C: sharp feature   → expensive
Brick D: topology event  → very expensive
```

Static partition으로 brick 수만 똑같이 나누면 실행 시간은 크게 달라진다.

Per-worker brick deque를 두고 idle worker가 busy worker에서 independent brick을 steal하면 tail을 줄일 수 있다.

### 5.2 Spatial Locality를 보존한 Stealing

Semiconductor structure는 neighboring brick이 같은 material/SDF page를 공유할 가능성이 높다.

따라서 owner는 local stack처럼 neighboring brick을 계속 처리하고,
thief는 상대적으로 오래된 독립 brick group을 steal하는 방향이 자연스럽다.

```text
owner:
recent child / neighboring brick

thief:
older independent brick subtree
```

이는 classic work-stealing locality intuition과 잘 맞는다.

### 5.3 Topology-Heavy Task를 별도 Class로 분리

Dual Contouring/Adaptive meshing에서 모든 active cell의 code path가 같지 않다.

예:

```text
simple crossing
complex multi-component cell
coarse/fine transition
manifold repair
```

이를 하나의 thread-level task queue에 무작위로 섞으면 warp divergence가 커질 수 있다.

EPAQ 관점으로 보면:

```text
SimpleMeshingQueue
TopologyQueue
TransitionQueue
```

정도의 큰 execution-path class를 두는 것이 의미가 있다.

### 5.4 CFD Simulation

Adaptive CFD에서도:

- active cell update
- boundary cell
- refinement/coarsening
- flux-heavy region

의 실행 비용이 다르다.

Work stealing은 mesh partition을 매 timestep 완전히 다시 만드는 것보다 더 fine-grained한 runtime load balance를 제공할 수 있다.

다만 task migration으로 field locality가 손상되면 bandwidth-bound CFD에서는 이득이 줄 수 있다.

### 5.5 GPU-Stay-GPU Architecture

최종적으로 사용자의 관심 pipeline은 다음처럼 볼 수 있다.

```text
Sparse Field
   ↓
Local Task Queues
   ↓
Work Stealing
   ↓
Divergence-Aware Task Classes
   ↓
Incremental Geometry
   ↓
Snapshot Publication
   ↓
GPU-Driven Vulkan Rendering
```

이 구조의 강점은 CPU가 work distribution을 매번 관리하지 않으면서도 GPU 내부의 irregularity를 흡수할 수 있다는 것이다.

### 5.6 Graphics / Game Engine Career

이 개념은 다음 역할과 직접 연결된다.

- rendering runtime
- virtualized geometry
- destructible geometry
- BVH traversal
- particle/simulation scheduler
- GPU-driven scene processing
- meshlet generation
- asynchronous geometry pipeline

면접에서는 “work stealing을 안다”보다 다음을 설명할 수 있는 것이 더 중요하다.

> **Global queue contention을 왜 local deque가 줄이는지, owner-LIFO/thief-steal 구조가 locality와 parallelism을 어떻게 나누는지, 그리고 load balance를 개선한 뒤 warp divergence가 새로운 병목이 될 수 있다는 점**이다.

---

## 6. 머릿속에 남길 질문 3개

1. **Global queue의 atomic contention을 줄이기 위해 per-block deque를 도입했는데 전체 성능이 오히려 떨어진다면, steal miss·task migration locality·warp divergence 중 어떤 지표를 어떤 순서로 확인해야 할까?**
2. **Dynamic SDF meshing에서 owner는 neighboring brick을 local LIFO로 처리하고 thief는 오래된 independent brick을 가져가는 정책이 cache locality와 load balance를 동시에 얻을 수 있는 이유는 무엇인가?**
3. **Execution-path별 queue를 늘릴수록 warp divergence는 줄 수 있지만 parallelism이 fragment될 수 있는데, task-class 수를 정할 때 어떤 profiler 지표를 함께 봐야 할까?**

---

## 7. graphics engineer 면접 질문 1개와 답변

### 질문

**“Persistent GPU scheduler에서 global concurrent queue가 이미 lock-free라면 왜 per-block queue와 work stealing을 추가해야 하나요?”**

### 답변

Lock-free global queue는 correctness와 system-wide progress 면에서 좋은 출발점이지만, 모든 worker가 같은 queue metadata에 접근하면 fine-grained workload에서는 queue 자체가 scalability bottleneck이 될 수 있다.

특히 worker 수가 늘수록:

- head/tail atomic contention
- cache-line serialization
- retry traffic
- global metadata bandwidth

가 증가한다.

Per-block 또는 per-worker deque를 사용하면 common path를 local ownership으로 바꿀 수 있다.

```text
Owner:
push/pop local bottom

Idle worker:
occasionally steal remote top
```

즉 대부분의 task operation은 global contention 없이 처리하고 imbalance가 생겼을 때만 비싼 remote operation을 수행한다.

하지만 work stealing도 자동으로 빠른 것은 아니다.

Remote steal은 data locality를 잃을 수 있고, 서로 다른 task type을 한 warp에 섞으면 divergence가 커질 수 있다. 그래서 victim policy, steal batch size, task granularity, execution-path-aware queue partition을 함께 설계해야 한다.

또 stealable task가 block-private shared memory에만 존재하면 다른 block이 접근할 수 없으므로, remote access가 필요한 queue metadata/payload는 thief가 볼 수 있는 memory scope에 있어야 한다.

핵심은:

> **Global queue는 load balance를 중앙화하고, work stealing은 scheduling common path를 분산한다. Work stealing의 목적은 steal을 많이 하는 것이 아니라 local execution을 최대화하면서 long tail만 분산하는 것이다.**

---

## 8. 포트폴리오 / 커리어 연결

이 주제는 graphics engineer 포트폴리오에서 GPU runtime 사고방식을 보여주기 좋다.

### GPU Runtime

- persistent workers
- per-block/local queues
- work stealing
- victim selection
- local-hit / steal ratio
- global spill

### Concurrency

- owner-only tail
- thief-shared head
- CAS boundary race
- ABA / generation
- approximate scheduling hints
- termination detection

### SIMT / Shader Execution

- warp divergence
- execution-path grouping
- task class
- EPAQ
- task granularity

### Geometry / Simulation

- sparse SDF bricks
- adaptive meshing
- topology-heavy task
- CFD adaptive region
- bursty workload

### Memory Layout

- compact logical task record
- payload late resolve
- persistent spatial order
- local queue occupancy
- remote-steal bandwidth

### Modern GPU

- Blackwell Cluster Launch Control
- hardware-assisted block work stealing
- persistent kernel
- GPU-resident task runtime

포트폴리오에서는 다음처럼 설명할 수 있다.

> **“Global queue contention을 줄이기 위해 worker-local deques를 사용하고, local queue가 비었을 때만 remote steal을 수행하는 hierarchical scheduler를 설계했습니다. Owner는 recent task를 local LIFO로 처리해 locality를 유지하고 thief는 older independent work를 가져가 long tail을 줄입니다. Task class를 execution path별로 제한적으로 분리해 load balance 개선이 warp divergence 악화로 이어지지 않게 했고, local-hit ratio·steal success·warp execution efficiency·p99 tail time을 함께 프로파일링했습니다.”**

이 설명은 C++/CUDA concurrency, GPU memory hierarchy, SIMT divergence, graphics workload 구조를 하나로 연결한다.

---

## 9. 내일 이어서 볼 개념

**Divergence-Aware GPU Task Partitioning: Execution-Path Queues, Task Coalescing, and Adaptive Grain Size**

오늘은 local queues + work stealing으로 **worker 간 load imbalance**를 줄였다.

그 다음 문제는 worker가 충분히 바빠졌는데도 warp lane들이 서로 다른 code path를 타면서 실제 SIMD/SIMT efficiency가 떨어지는 경우다.

학습 흐름:

```text
Global Queue
    ↓
Backpressure / Forward Progress
    ↓
Local Queues + Work Stealing
    ↓
Execution-Path-Aware Task Partitioning
    ↓
Adaptive Task Granularity
```

다음 노트에서는:

- task class와 execution-path signature
- EPAQ의 의미
- warp divergence vs queue fragmentation
- tiny-task coalescing
- task split threshold
- runtime cost prediction
- occupancy와 grain size
- spatial locality를 유지한 task batching
- SDF/CFD work를 homogeneous batch로 만드는 기준

을 중심으로 이어간다.

---

## 10. 참고 키워드

- GPU Work Stealing
- Work-Stealing Deque
- Per-Block Queue
- Per-Worker Local Queue
- Owner Push / Pop
- Thief Steal
- LIFO Locality
- Load Balancing
- Long-Tail Effect
- Persistent Threads
- Persistent Kernel
- Global Queue Contention
- Atomic Hot Spot
- Victim Selection
- Randomized Stealing
- Batch Steal
- Work Sharing
- Local Queue Overflow
- Global Spill Queue
- Task Granularity
- Parallel Slack
- Task Coalescing
- Warp Divergence
- Execution-Path-Aware Queueing (EPAQ)
- Task Class
- Branch Efficiency
- Warp Execution Efficiency
- Spatial Locality
- ABA / Generation
- Termination Detection
- Logical Task Handle
- Cluster Launch Control
- Blackwell / Compute Capability 10.0
- Dynamic SDF / Level Set
- Adaptive Meshing
- CFD Load Balancing
- GPU-Driven Rendering
- Yuki Maeda, Kenjiro Taura, **“GTaP: A GPU-Resident Fork-Join Task-Parallel Runtime with a Pragma-Based Interface,” arXiv:2604.05982, 2026**
  - https://arxiv.org/abs/2604.05982
- NVIDIA, **CUDA Programming Guide — Work Stealing with Cluster Launch Control**
  - https://docs.nvidia.com/cuda/cuda-programming-guide/04-special-topics/cluster-launch-control.html
- NVIDIA, **CUDA Programming Guide — CUDA C++ Execution Model / Forward Progress**
  - https://docs.nvidia.com/cuda/cuda-programming-guide/05-appendices/cuda-cpp-execution-model.html
- Sanjay Chatterjee, Max Grossman, Alina Sbirlea, Vivek Sarkar, **“Dynamic Task Parallelism with a GPU Work-Stealing Runtime System,” 2011**
- David Troendle, Tuan Ta, Byunghyun Jang, **“A Specialized Concurrent Queue for Scheduling Irregular Workloads on GPUs,” ICPP 2019**
- Xiangyu Zhang, Yangdong Deng, Shuai Mu, **“Toward Concurrent Lock-Free Queues on GPUs,” 2014**
