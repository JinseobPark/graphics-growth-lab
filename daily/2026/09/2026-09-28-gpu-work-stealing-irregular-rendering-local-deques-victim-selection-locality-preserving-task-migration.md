---
title: "GPU Work Stealing for Irregular Rendering: Local Deques, Victim Selection, and Locality-Preserving Task Migration"
date: "2026-09-28"
category: Graphics
tags: [GPU, Rendering, Work Stealing, Persistent Kernel, BVH, LOD, Meshlet, CUDA, Vulkan, C++]
level: intermediate
---

# [Daily Graphics Growth] 2026-09-28 - GPU Work Stealing for Irregular Rendering: Local Deques, Victim Selection, and Locality-Preserving Task Migration

## 1. 오늘의 개념

어제의 **GPU Traversal Continuations**에서는 큰 subtree를 budget 단위로 suspend/resume해 tail latency를 줄였다. 오늘은 그 continuation을 여러 GPU worker 사이에서 어떻게 재분배할지 본다.

중앙 global queue는 단순하지만 모든 worker가 같은 atomic queue metadata를 경쟁한다. Local queue는 locality가 좋지만 busy/idle worker imbalance를 해결하지 못한다. Work stealing은 두 장점을 합친다.

```text
top ---------------- bottom
 ^                      ^
thief steals        owner push/pop
(old work)          (recent work)
```

Owner는 최신 task를 LIFO로 처리해 DFS/treelet locality를 유지하고, idle thief는 반대쪽에서 오래된 coarse task를 가져간다.

2026년 CUDA Programming Guide는 Blackwell **Cluster Launch Control (CLC)** 을 work stealing의 공식 사례로 설명한다. 오늘은 그보다 일반적인 application-level **local deque + victim selection + task migration** 구조를 다룬다.

## 2. 한 줄 핵심

> GPU work stealing은 **local fast path에서 locality를 유지하고, imbalance가 생겼을 때만 idle worker가 다른 worker의 오래된 coarse task를 가져가 global contention과 tail latency를 줄이는 scheduling policy**다.

## 3. 왜 중요한가

Irregular rendering/simulation에서는 task 크기 편차가 크다.

```text
flat tile → little work
dense gate/trench → much work
empty SDF brick → almost none
complex surface brick → many meshlets
```

Global queue만 쓰면 tiny task가 많을수록 atomic contention이 커지고, local queue만 쓰면 monster subtree가 특정 worker에 남는다. 따라서 이상적인 구조는:

```text
most work → local push/pop
imbalance → remote steal
tail → finer steal allowed
```

이다. Stealing은 normal path가 아니라 **imbalance 해결용 slow path**여야 한다.

## 4. 구현 관점

### 4.1 Local deque

Chase-Lev 계열의 핵심은 **single owner, multiple thieves**다.

```text
owner: pushBottom / popBottom
thief: stealTop
```

Owner와 thief가 반대쪽 끝을 사용하므로 common-path contention이 줄어든다.

### 4.2 Owner LIFO와 locality

최근 child는 parent와 같은 treelet, Morton range, SDF brick에 있을 가능성이 높다. Newest-first 실행은 DFS에 가까워 cache locality와 warm traversal state를 유지하기 쉽다.

### 4.3 Coarse-grained stealing

Thief는 owner hot side가 아니라 오래된 coarse continuation을 가져가는 편이 좋다. 한 task씩 가져가면 CAS 비용이 커지고, 너무 큰 batch는 locality/fairness를 깨뜨린다.

```text
small batch → low migration cost, high atomic/task
large batch → low atomic/task, higher locality risk
```

### 4.4 GPU worker granularity

Thread마다 deque를 두기보다 **warp/subgroup 또는 workgroup/CTA** 단위가 실용적이다. Workgroup은 shared-memory local queue와 cooperative traversal을 사용하기 좋다.

### 4.5 Local hot queue + global stealable queue

```text
shared/local hot queue
+
device-global stealable queue
```

최근 child는 local에 두고, local work가 high-water mark를 넘을 때 오래된 continuation만 global-visible하게 export한다. 모든 child publication에 device-wide atomic을 쓰지 않는 것이 핵심이다.

### 4.6 Victim selection

가능한 정책은 random, round-robin, sampled-heavy, locality-aware victim이다. Exact queue length를 계속 읽는 대신 `EMPTY / LOW / MEDIUM / HIGH` 같은 coarse load hint를 사용할 수 있다.

여러 thief가 같은 victim에 몰리는 **thundering herd**는 randomized victim, bounded retry, batch stealing으로 완화한다.

### 4.7 Generation / memory ordering

Ring slot 재사용 때문에 stale thief가 old task를 읽지 않도록 `slot index + generation`을 검증한다. Queue slot에는 큰 task struct보다 `ContinuationId` 같은 작은 handle을 두는 편이 좋다.

Owner의 payload write와 publish, thief의 claim과 payload read 사이에는 **release/acquire** 관계가 필요하다.

```text
local path → workgroup scope
steal path → device scope
```

### 4.8 Continuation과 tail-aware stealing

어제의 continuation이 자연스러운 steal unit이다.

```text
large subtree
→ continuation
→ local deque
→ stealable under pressure/tail
```

Steady state에서는 coarse steal만 허용하고, idle worker가 많아지는 tail에서는 더 작은 continuation도 허용할 수 있다.

### 4.9 Distributed termination

Local deque가 분산되면 `globalQueue.empty()`만으로 종료할 수 없다.

```text
all local queues empty
AND no active worker
AND no steal in flight
AND no pending child publication
```

을 race 없이 판정해야 한다. 이 문제가 내일 주제다.

### 4.10 프로파일링

볼 지표:

- steal attempts / success ratio
- stolen work size
- CAS retry count
- victim distribution
- remote queue traffic
- idle worker ratio
- tail fraction
- cache hit after steal
- continuation resume latency
- p95/p99 subtree time

```text
StealEfficiency = UsefulWorkAfterSteal / StealOverhead
```

## 5. 내 관심 분야와 연결

**Semiconductor visualization:** flat substrate와 dense gate/trench region의 work 편차가 크므로 XY tile/treelet local deque와 coarse continuation stealing이 잘 맞는다.

**Dynamic SDF:** dirty brick마다 topology complexity가 크게 달라 one-brick-per-workgroup이 쉽게 imbalance를 만든다.

**CFD:** AMR block, sparse octree, particle bucket도 shock/front 근처에서 work가 집중된다.

**CUDA/Vulkan:** CUDA에서는 persistent scheduler와 Blackwell CLC가 선택지다. Vulkan에서는 software deque가 가능하지만 persistent-kernel portability를 고려해 multi-dispatch frontier를 baseline으로 유지하는 hybrid가 실용적이다.

## 6. 머릿속에 남길 질문 3개

1. **Owner가 newest task를 LIFO로 처리하고 thief가 oldest task를 가져가는 구조가 locality와 contention을 동시에 개선하는 이유는 무엇인가?**
2. **Steal batch가 커질수록 atomic overhead는 줄지만 locality/fairness가 바뀌는데, steal success ratio·cache hit after steal·tail fraction으로 batch size를 어떻게 조절할까?**
3. **Dynamic SDF에서 shared-memory local queue와 device-global stealable queue를 함께 쓸 때 어떤 continuation만 global-visible하게 export해야 할까?**

## 7. graphics engineer 면접 질문 1개와 답변

### 질문
**Irregular GPU workload에서 global atomic queue 하나 대신 work stealing을 쓰면 왜 더 빠를 수 있나요?**

### 답변
Global queue 하나는 모든 worker가 같은 metadata를 경쟁한다. Work stealing은 대부분의 work를 local fast path에 두고, idle worker만 rare remote operation을 수행한다.

Owner는 최신 child를 LIFO로 처리해 DFS/treelet locality를 유지하고, thief는 오래된 coarse continuation을 가져가 독립적인 work chunk를 확보한다. 다만 remote atomic, CAS retry, cold-cache migration 비용이 있으므로 steady state에서는 local execution이 dominant하고 imbalance/tail에서만 적극적으로 사용하는 것이 좋다.

> **Work stealing은 locality를 owner에게 남기면서 idle GPU capacity만 busy region으로 이동시키는 policy다.**

## 8. 포트폴리오 / 커리어 연결

> **“Irregular hierarchy traversal에서 중앙 atomic queue contention을 줄이기 위해 per-workgroup local deques를 사용했습니다. Owner는 최신 treelet continuation을 LIFO로 처리하고, idle workgroup은 victim의 오래된 coarse continuation을 batch steal했습니다. Queue slot에는 continuation handle만 저장하고 generation/snapshot epoch로 stale work를 검증했으며, idle-worker ratio와 tail fraction으로 steal aggressiveness를 조절했습니다.”**

이 설명은 C++, GPU memory model, rendering traversal, sparse simulation, profiling을 하나의 시스템 이야기로 연결한다.

## 9. 내일 이어서 볼 개념

**GPU Distributed Termination Detection: Quiescence, Epoch Counters, and Race-Free Persistent Schedulers**

다음에는 queue-empty race, active/outstanding-work counters, quiescence, epoch/token termination, in-flight steal, child-publication race를 연결한다.

## 10. 참고 키워드

- GPU Work Stealing
- Chase-Lev Deque
- Owner LIFO / Thief Steal
- Victim Selection / Batch Stealing
- Persistent Kernel / Tail Latency
- CAS / ABA / Generation
- Device-Scope Atomics
- Continuation / Treelet / BVH / Meshlet
- Sparse SDF
- Cluster Launch Control
- NVIDIA CUDA Programming Guide — Work Stealing with Cluster Launch Control
  - https://docs.nvidia.com/cuda/cuda-programming-guide/04-special-topics/cluster-launch-control.html
- David Chase, Yossi Lev — Dynamic Circular Work-Stealing Deque
  - https://www.dre.vanderbilt.edu/~schmidt/PDF/work-stealing-dequeue.pdf
- Nhat Minh Lê et al. — Correct and Efficient Work-Stealing for Weak Memory Models
  - https://fzn.fr/readings/ppopp13.pdf
- Petr Korolev — What Irregularity Costs: CUDA C++, Rust, and Triton on a Hash-Blocked GPU Workload, 2026
  - https://arxiv.org/abs/2608.08287
