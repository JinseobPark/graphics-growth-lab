---
title: "GPU Traversal Continuations: Stack Compression, Suspend/Resume Work, and Tail-Phase Load Balancing"
date: "2026-09-27"
category: Graphics
tags: [GPU, Rendering, GPU-Driven Rendering, Hierarchy Traversal, Continuation, Stack Compression, Persistent Kernel, Work Queue, Tail Load Balancing, BVH, LOD, Meshlet, CUDA, Vulkan, C++]
level: intermediate
---

# [Daily Graphics Growth] 2026-09-27 - GPU Traversal Continuations: Stack Compression, Suspend/Resume Work, and Tail-Phase Load Balancing

## 1. 오늘의 개념

어제는 **GPU Traversal Prefetch and Latency Hiding**에서 future node address를 stack/frontier/treelet state에서 일찍 노출하고, current-node 계산과 next-node fetch를 겹쳐 memory latency를 숨기는 방법을 봤다.

오늘은 그 다음 문제인 **continuation**을 본다. Hierarchy traversal은 subtree마다 work size가 크게 다르다. 작은 subtree는 수십 node로 끝나지만 특정 dense region은 수천 node를 방문할 수 있다. Persistent worker가 큰 subtree를 끝까지 독점하면 queue는 비었는데 일부 worker만 오래 남는 tail phase가 발생한다.

반대로 모든 child를 즉시 global queue에 내보내면 load balance는 좋아지지만 queue atomic traffic이 커지고 parent-child cache locality가 무너진다. 중간 해법이 traversal state를 일정 budget마다 **suspend/resume**하는 것이다.

Local traversal은 treelet locality를 유지하며 일정 node budget을 처리한다. Budget이 끝나면 current node, compressed stack, snapshot epoch 등 최소 상태만 continuation record로 저장해 global queue에 다시 publish한다. Idle worker는 이 continuation을 가져와 resume한다.

핵심 질문은 다음이다.

> **Subtree locality를 유지할 만큼 오래 일하면서도, monster subtree가 frame tail을 독점하지 않도록 어느 시점에 state를 저장하고 work를 양보해야 하는가?**

최근 NVIDIA의 Blackwell Cluster Launch Control 문서도 persistent scheduling이 실제 available resource와 workload imbalance에 취약할 수 있음을 설명하고, 남은 work를 runtime에 동적으로 재분배하는 방향을 제시한다. 오늘 노트는 특정 hardware feature보다 이 dynamic scheduling principle을 hierarchy traversal에 연결한다.

## 2. 한 줄 핵심

> GPU traversal continuation의 핵심은 **worker가 locality가 좋은 subtree를 일정 budget 동안 처리하고, budget을 넘으면 최소한의 stack/state만 compact continuation으로 저장해 다른 worker가 resume하게 함으로써 locality와 tail load balancing을 동시에 확보하는 것**이다.

## 3. 왜 중요한가

Persistent queue는 task 간 imbalance에는 강하지만 task 내부 imbalance까지 자동으로 해결하지는 않는다. Global queue에 task가 하나만 남았는데 그 task가 거대한 subtree라면 다른 worker는 idle 상태가 된다.

따라서 두 극단을 피해야 한다.

- 너무 coarse한 task: locality는 좋지만 p95/p99 tail latency가 나쁘다.
- 너무 fine한 task: fairness는 좋지만 queue atomic, cache miss, scheduling overhead가 커진다.

Continuation은 traversal을 완료할 때까지 소유하는 task가 아니라 **일정 budget만 처리하고 양보할 수 있는 task**로 바꾼다.

Interactive renderer에서는 평균 traversal time보다 tail이 중요하다. 한 worker가 늦게 끝나면 visible-cluster list 생성, indirect work generation, subsequent render pass까지 밀릴 수 있기 때문이다.

또 continuation은 어제의 prefetch와도 연결된다. 너무 자주 suspend하면 software prefetch pipeline이 steady state에 도달하기 전에 다시 warm-up해야 한다. 따라서 continuation budget은 단순 fairness 값이 아니라 locality, prefetch, queue pressure를 함께 결정하는 scheduler parameter다.

## 4. 구현 관점

### 4.1 Explicit Traversal State

CPU recursive traversal에서는 call stack이 state를 숨겨준다. GPU suspend/resume에서는 state를 명시해야 한다.

대표 상태는 root/treelet ID, current node, pending stack, stack top, snapshot epoch, LOD/refinement policy다. Continuation에는 resume에 필요한 최소 subset만 저장하는 것이 중요하다.

### 4.2 Continuation Record

Conceptual record는 다음 semantic field를 가진다.

- TreeletId 또는 RootId
- CurrentNode
- CompressedStackOffset / Count
- SnapshotEpoch
- Priority
- ResumeCount
- YieldReason

Continuation record가 지나치게 크면 traversal metadata traffic을 줄이려다 scheduler state traffic을 새로 만들게 된다.

### 4.3 Stack Compression

DFS stack depth가 32이고 entry가 32-bit global NodeID라면 128B다. Suspend가 빈번하면 이 state spill/read가 반복된다.

Treelet node가 contiguous하면 global NodeID 대신 8-bit 또는 16-bit local index를 사용할 수 있다.

예를 들어 16-bit local index를 사용하면 depth-32 stack은 128B에서 64B로 줄어든다. Treelet을 256 nodes 이하로 제한할 수 있다면 8-bit local index도 가능하다.

중요한 invariant는 continuation이 가리키는 treelet base와 local-index namespace가 snapshot lifetime 동안 안정적이어야 한다는 것이다.

### 4.4 Recompute vs Save

Bounds, error, residency result 같은 derived state까지 continuation에 저장할 필요는 없다. Resume 시 node table에서 다시 읽고 계산할 수 있다.

Trade-off는 다음과 같다.

- state를 많이 저장: resume는 빠르지만 continuation bytes 증가
- state를 적게 저장: traffic은 작지만 resume refetch/recompute 증가

Memory-bound traversal에서는 compact state가 더 유리한 경우가 많다.

### 4.5 Local Work Budget

Yield 기준은 maxNodes, maxOutputClusters, maxChildrenGenerated 같은 deterministic count가 실용적이다. GPU cycle budget은 hardware/frequency 차이 때문에 portability가 떨어진다.

Budget이 너무 작으면 continuation round trip, queue atomic, cache warm-up 손실이 증가한다. 너무 크면 monster subtree가 tail을 지배한다.

### 4.6 Adaptive Budget

Queue depth와 active worker 수를 scheduler feedback으로 사용할 수 있다.

- queue가 깊고 active worker가 충분함: larger local budget, locality 우선
- queue가 얕고 idle worker가 늘어남: smaller budget, yield/split 적극화

즉 tail phase에서만 더 공격적으로 subtree를 분산할 수 있다.

### 4.7 Continuation Splitting

Suspend할 때 pending stack 전체를 continuation 하나에 넣지 않고 일부 stack entry를 별도 task로 떼어낼 수 있다.

예를 들어 pending subtree A, B, C, D 중 A/B는 current worker가 계속 처리하고 C/D는 global queue로 export한다. 이 순간 local DFS state가 parallel frontier로 변환된다.

좋은 split point는 다른 treelet로 넘어가는 entry, Morton distance가 큰 entry, 큰 subtree로 추정되는 entry다.

### 4.8 Tail-Phase Detection

Tail signal은 queue depth 감소, idle worker 증가, active continuation 수 감소, 일부 task duration 증가로 잡을 수 있다.

이때 목표는 **work conservation**이다.

> 처리 가능한 work가 남아 있는데 worker가 idle하지 않게 한다.

Continuation은 private local work를 필요할 때 다시 globally visible work로 바꾸는 메커니즘이다.

### 4.9 Workgroup vs Subgroup Continuation

Subgroup-level continuation은 fine-grained balance를 제공하지만 state/queue 수가 늘어난다. Workgroup-level continuation은 cooperative shared stack과 node batch를 유지하기 쉽지만 granularity가 크다.

Wide-node evaluation과 treelet staging을 workgroup 단위로 하고 있다면 workgroup continuation이 더 자연스러울 수 있다.

### 4.10 Shared Stack Spill

Treelet-local traversal stack을 shared memory에 유지한다면 suspend 시 compact global record로 spill하고 resume 시 reconstruction한다.

이 spill/reload 비용이 크면 yield frequency를 낮춰야 한다. Fixed-size hot-path continuation + rare overflow buffer 구조가 allocator와 queue를 단순하게 유지하기 좋다.

### 4.11 Snapshot Epoch

Continuation은 생성된 immutable hierarchy/residency snapshot epoch를 포함해야 한다.

Resume 시 같은 epoch가 아직 valid해야 하며, interactive frame에서 old refinement continuation이 의미 없어졌다면 stale continuation을 discard하거나 root/treelet부터 다시 평가할 수 있다.

Prefetch와 마찬가지로 continuation state는 correctness ownership과 분리해야 한다.

### 4.12 Termination Detection

Continuation을 도입하면 queue-empty만으로 kernel 종료를 결정하면 안 된다.

Worker가 local stack을 처리하고 있다가 queue-empty 판정 직후 새 continuation을 publish할 수 있기 때문이다.

Conceptual quiescence 조건은 다음과 같다.

- global queue empty
- active worker 없음
- unpublished local work 없음
- outstanding work count가 0

이전에 다룬 GPU termination detection의 active-work counter/epoch protocol이 다시 필요하다.

### 4.13 Tail-Termination Race

위험한 sequence는 queue가 비어 보임 → 다른 worker 종료 → 마지막 worker가 continuation publish → 소비할 worker 없음이다.

Persistent loop가 outstanding-work state를 확인하고, task ownership과 child/continuation publication을 하나의 lifecycle로 다뤄야 한다.

### 4.14 Cluster Launch Control에서 얻는 교훈

NVIDIA Blackwell Cluster Launch Control은 persistent scheduling에서 실제 available SM과 남은 work의 imbalance를 다루는 dynamic scheduling 방향을 보여준다.

Hierarchy traversal에 같은 mechanism을 반드시 사용해야 한다는 뜻은 아니다. 시스템적 교훈은 **static ownership보다 runtime-observed remaining work를 기반으로 work를 재분배하는 것이 tail efficiency에 중요하다**는 것이다.

### 4.15 CUDA Dynamic Parallelism과 비교

CUDA Dynamic Parallelism은 device code에서 새 kernel work를 만들 수 있다. 그러나 수많은 작은 subtree마다 child kernel을 launch하는 방식은 fine-grained traversal에는 무거울 수 있다.

따라서 hierarchy traversal에서는 persistent workers + continuation queue가 더 작은 scheduling unit을 제공하는 경우가 많다.

### 4.16 Local Queue + Global Queue

Queue contention을 줄이기 위해 two-level scheduler를 만들 수 있다.

- local workgroup queue: same treelet/locality work
- global continuation queue: fairness와 load balance가 필요한 oversized work

작은 child는 local에서 처리하고 tail risk가 생긴 subtree만 globalize한다.

### 4.17 Work Stealing과의 연결

Global FIFO 대신 owner-local deque + thief stealing을 사용할 수도 있다.

Owner는 DFS locality를 유지하며 local end에서 pop하고, idle worker는 큰-granularity end에서 steal한다. 다만 GPU에서는 deque synchronization, ABA/versioning, termination이 복잡해진다.

그래서 오늘은 continuation을 먼저 확립하고 내일 work stealing으로 이어간다.

### 4.18 Determinism

Dynamic resume order는 visible cluster list 순서를 바꿀 수 있다.

후속 rendering이 order-independent라면 문제가 없지만 reproducible debugging이 필요하면 deterministic scheduler mode, stable compaction, logical-ID ordering을 별도로 둘 수 있다.

### 4.19 Profiler에서 볼 지표

- traversal GPU time
- p50/p95/p99 task duration
- active workers over time
- queue depth
- tail duration
- continuation create/resume count
- average resume count
- continuation bytes written/read
- compressed stack bytes
- stack overflow count
- useful node visits / continuation round trip
- global queue atomic count
- cache hit after resume
- stale continuation drop count
- worker idle cycles
- outstanding-work count
- termination-check overhead
- prefetch warm-up loss after resume

핵심 derived metric은 두 가지다.

**TailFraction = active worker가 낮은 상태로 소비한 시간 / 전체 traversal time**

**ContinuationEfficiency = useful node visits / continuation round trips**

TailFraction이 줄어도 ContinuationEfficiency가 지나치게 낮다면 over-yielding일 수 있다.

## 5. 내 관심 분야와 연결

### Semiconductor Process Visualization

반도체 구조는 region별 complexity 차이가 크다.

Flat substrate는 traversal이 빨리 끝나는 반면 gate/trench가 조밀하거나 cross-section plane 근처인 region은 더 깊은 LOD/refinement traversal을 요구한다.

Static XY tile assignment에서는 dense tile을 잡은 worker가 tail을 만들기 쉽다. Treelet-local continuation을 사용하면 dense tile도 일정 budget으로 나눠 idle worker가 이어받을 수 있다.

### Dynamic SDF

Dirty brick 역시 topology complexity 편차가 크다. 어떤 brick은 surface가 거의 없고 어떤 brick은 많은 triangles/meshlets를 생성한다.

Brick 또는 surface-tree traversal을 oversized task로 보고 continuation으로 분할하면 dynamic meshing scheduler에도 같은 패턴을 적용할 수 있다.

### CUDA / Warp

Persistent kernel + global queue + local stack + compressed continuation + outstanding-work termination은 CUDA에서 자연스러운 architecture다.

어제의 async prefetch pipeline과 결합할 때 continuation budget은 prefetch steady-state를 유지할 만큼 충분히 길어야 한다.

### Vulkan Compute

Vulkan compute에서도 long-running queue-driven dispatch를 설계할 수 있지만 platform/watchdog와 dispatch lifetime을 고려해야 한다. Multi-dispatch frontier path와 persistent-like path를 모두 두고 workload에 따라 고르는 방식도 실용적이다.

### CFD

AMR, sparse octree, iso-surface extraction은 subtree 크기 편차가 크다. Continuation + tail splitting은 spatial locality와 global utilization을 균형 있게 유지하는 데 적합하다.

## 6. 머릿속에 남길 질문 3개

1. **Continuation budget이 너무 작으면 queue/state overhead가 늘고 너무 크면 tail latency가 늘어난다. Queue depth와 active worker count를 이용해 adaptive budget을 어떻게 설계할 수 있을까?**
2. **Treelet-local DFS stack을 32-bit global NodeID 대신 8/16-bit local index로 압축할 때 hierarchy build와 snapshot lifetime에 어떤 invariant가 필요할까?**
3. **Continuation resume로 worker가 바뀌며 cache locality가 깨질 때 strict affinity control이 어려운 GPU에서 queue ordering과 treelet grouping으로 이를 어떻게 완화할 수 있을까?**

## 7. graphics engineer 면접 질문 1개와 답변

### 질문

**“Persistent GPU work queue를 쓰면 irregular hierarchy traversal의 load balancing 문제는 해결된 것 아닌가요?”**

### 답변

완전히 해결되지는 않는다.

Persistent queue는 task 간 imbalance에는 강하다. Worker가 짧은 task를 끝내면 다음 task를 가져올 수 있기 때문이다. 하지만 task 하나가 거대한 subtree라면 그 task 내부의 work는 global scheduler에서 보이지 않는다.

즉 queue가 비어도 한 worker 안에는 수천 node의 hidden local work가 남을 수 있다.

그래서 일정 node budget마다 current node와 compressed stack을 continuation으로 저장하고 global queue에 다시 노출하면 idle worker가 해당 subtree의 나머지 work를 이어받을 수 있다.

다만 너무 자주 yield하면 queue atomic, state spill/reload, prefetch warm-up 손실이 커진다. 따라서 핵심 trade-off는 **locality vs fairness/tail latency**다.

평균 traversal time만 보지 않고 active-worker timeline, p95/p99 task duration, TailFraction, useful nodes per continuation round trip을 함께 봐야 한다.

## 8. 포트폴리오 / 커리어 연결

이 주제는 graphics engineer가 단순 work queue가 아니라 **irregular GPU scheduling과 tail latency**까지 이해한다는 것을 보여주기 좋다.

포트폴리오 연결 포인트는 DFS/treelet traversal, persistent workers, continuation, stack compression, outstanding-work quiescence, queue contention, p95/p99 profiling이다.

포트폴리오에서는 다음처럼 설명할 수 있다.

> **“Persistent traversal queue만으로 남는 large-subtree tail을 줄이기 위해 per-worker local node budget을 도입했습니다. Budget 소진 시 DFS stack을 treelet-local 16-bit indices로 압축한 continuation record로 global queue에 publish하고, queue depth와 idle-worker 수에 따라 yield budget을 조절했습니다. Snapshot epoch를 continuation에 포함해 stale work를 제거하고 outstanding-work counter로 queue-empty race 없이 termination을 판정했습니다.”**

## 9. 내일 이어서 볼 개념

**GPU Work Stealing for Irregular Rendering: Local Deques, Victim Selection, and Locality-Preserving Task Migration**

오늘은 continuation으로 traversal state를 suspend/resume해 tail을 줄이는 방법을 봤다.

다음 질문은:

> **Global queue 하나의 contention을 줄이면서도 idle worker가 다른 worker의 남는 subtree를 가져오게 하려면 어떤 work-stealing 구조가 필요한가?**

다음 노트에서는 local deque, owner-pop/thief-steal, victim selection, steal granularity, locality loss, ABA/versioning, termination detection, deterministic fallback을 연결한다.

## 10. 참고 키워드

- GPU Traversal Continuation
- Suspend / Resume
- Stack Compression
- DFS Stack
- Treelet Local Index
- Persistent Kernel
- Persistent Work Queue
- Tail Latency
- Tail-Phase Load Balancing
- Work Conservation
- Adaptive Work Budget
- Queue Depth
- Worker Utilization
- Continuation Splitting
- Work Stealing
- Outstanding Work Counter
- Quiescence Detection
- Dynamic Scheduling
- Cluster Launch Control
- CUDA Dynamic Parallelism
- Fixed-Size Continuation Record
- Snapshot Epoch
- Stale Work
- Sparse BVH / Octree
- Continuous LOD
- Dynamic SDF
- NVIDIA CUTLASS — **Blackwell Cluster Launch Control**
  - https://docs.nvidia.com/cutlass/latest/media/docs/cpp/blackwell_cluster_launch_control.html
- NVIDIA CUDA Programming Guide — **CUDA Dynamic Parallelism**
  - https://docs.nvidia.com/cuda/cuda-programming-guide/04-special-topics/dynamic-parallelism.html
- Y. S. Tozlu, A. Naithani, H. Zhou — **TTP: A Hardware-Efficient Design for Precise Prefetching in Ray Tracing**, 2026
  - https://arxiv.org/abs/2605.16253
