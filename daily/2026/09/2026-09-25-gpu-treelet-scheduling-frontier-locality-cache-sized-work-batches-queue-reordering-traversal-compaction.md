---
title: "GPU Treelet Scheduling and Frontier Locality: Cache-Sized Work Batches, Queue Reordering, and Traversal Compaction"
date: "2026-09-25"
category: Graphics
tags: [GPU, Rendering, GPU-Driven Rendering, Treelet Scheduling, Frontier Locality, Traversal Compaction, Work Queue, Cache Locality, BVH, LOD, Meshlet, Vulkan, CUDA, C++]
level: intermediate
---

# [Daily Graphics Growth] 2026-09-25 - GPU Treelet Scheduling and Frontier Locality: Cache-Sized Work Batches, Queue Reordering, and Traversal Compaction

## 1. 오늘의 개념

어제는 **GPU Hierarchy Traversal Memory Layout**에서 wide node, quantized metadata, cache-coherent ordering으로 hierarchy 자체의 memory traffic을 줄이는 방법을 봤다.

오늘은 execution order로 내려간다.

> **Node가 memory에서 잘 배치되어 있어도 traversal frontier가 서로 먼 subtree를 섞어 처리하면 cache locality는 다시 깨진다.**

**Treelet Scheduling**은 큰 hierarchy를 cache-friendly한 작은 subtree 단위로 묶고, frontier task를 treelet 기준으로 재정렬해 비슷한 work를 가까운 시간에 처리하는 방법이다.

```text
Global Frontier
  ↓ bucket/compact
Treelet A batch
Treelet A batch
Treelet B batch
  ↓
cache-local traversal
```

여기서 treelet은 ray-tracing BVH 전용 개념이 아니다. LOD hierarchy, sparse SDF hierarchy, visibility tree, AMR/CFD spatial tree처럼 작은 subtree 내부에서 반복 접근이 있는 GPU traversal에 같은 원리를 적용할 수 있다.

오늘은 **cache-sized work batch**, **frontier reordering**, **subgroup traversal compaction**을 하나의 scheduling 문제로 본다.

## 2. 한 줄 핵심

> 좋은 hierarchy layout은 locality의 가능성을 만들고, 좋은 treelet scheduler는 frontier를 cache-local batch로 묶고 survivor를 subgroup 단위로 compact해 그 locality를 실제 실행 시간에 보존한다.

## 3. 왜 중요한가

Hierarchy traversal은 연산보다 node fetch가 지배하기 쉽다. 2026년 **TTP** 연구도 ray-tracing BVH traversal을 memory-bound 문제로 설명하고, traversal stack에 이미 있는 future node address를 prefetch해 평균 1.48× speedup을 보고했다.

즉 traversal state에는 다음 access에 대한 정보가 이미 있다.

```text
current work
→ likely next subtree
→ likely next node addresses
```

이 정보를 scheduling에 쓰지 않고 global queue를 무작위로 소비하면 physical Morton/DFS ordering의 이득을 잃을 수 있다.

Aila와 Karras의 treelet 연구는 hierarchy를 작은 treelet로 나누고 ray를 queue/reorder해 cache pressure를 낮추는 것이 memory bandwidth를 크게 줄일 수 있음을 보였다. 2026년 **JZ-Tree** 역시 GPU tree traversal의 핵심 문제를 divergence와 irregular memory access로 보고 Morton ordering과 collaborative execution으로 locality를 높인다.

따라서 traversal 최적화는:

```text
Memory Layout Locality
+
Execution Order Locality
```

를 함께 봐야 한다.

## 4. 구현 관점

### 4.1 Cache-Sized Treelet

Treelet은 conceptually:

```text
small connected subtree
whose metadata/workset is cache-friendly
```

한 단위다.

크기를 정할 때는 node bytes, fanout, leaf metadata, L1/L2 behavior, worker 수를 함께 본다. 너무 작으면 scheduling overhead가 커지고, 너무 크면 local working set이라는 의미가 사라진다.

### 4.2 Frontier Reordering

Frontier가:

```text
A0 F2 B1 A1 Q3
```

처럼 섞여 있다면 다음 key로 bucket/partition할 수 있다.

- treelet ID
- Morton prefix
- subtree root
- spatial tile
- LOD group

항상 full sort가 필요한 것은 아니다. Bucket, radix bin, per-treelet queue, chunked stable partition처럼 cheaper reordering이 실용적이다.

### 4.3 Reorder Frequency

매 node expansion마다 reorder하면 locality는 좋아질 수 있지만 sort/queue 비용이 커진다. 반대로 너무 드물면 frontier가 다시 섞인다.

좋은 trigger 후보:

- frontier size threshold
- treelet switch rate 증가
- L2 hit rate 하락
- 특정 hierarchy depth 도달

즉 **reorder overhead와 locality gain을 함께 측정**해야 한다.

### 4.4 Frontier Entropy

Execution-order locality를 다음처럼 볼 수 있다.

```text
A A A A B B B B  → low entropy
A B C D A C B D  → high entropy
```

실용적인 metric은:

```text
treelet switches / processed tasks
```

이다.

### 4.5 Subgroup Compaction

Node test 후 살아남은 child만 queue에 넣는다.

Naive한 lane별 atomic append 대신:

```text
ballot
→ active mask
→ prefix offset
→ leader reserves N slots
→ lanes scatter
```

형태를 사용한다.

이는 Vulkan subgroup, CUDA warp, DirectX Wave intrinsics에서 모두 대응되는 패턴이다.

### 4.6 Aggregated Queue Reservation

32 lanes에서 9개 child만 살아남았다면 global queue reservation은 9번이 아니라 한 번으로 줄일 수 있다.

```text
atomicAdd(queueTail, 9)
```

후 lane별 prefix offset으로 write한다.

Queue contention과 atomic serialization을 줄인다.

### 4.7 Compaction은 다음 stage의 Layout이다

Survivor를 단순히 빈칸 없이 붙이는 것보다:

```text
same treelet
same spatial key
same task class
```

순으로 compact하면 다음 traversal 단계의 locality까지 개선할 수 있다.

즉 compaction은 **future scheduling order를 만드는 단계**다.

### 4.8 Two-Level Queue

```text
Global Treelet Queue
        ↓
Per-Treelet Local Queue
```

구조를 생각할 수 있다.

Global queue는 load balance를 담당하고 local queue는 subtree locality를 유지한다. Idle worker는 locality를 단계적으로 완화하며 steal한다.

```text
same treelet
→ nearby treelet
→ same spatial parent
→ any work
```

### 4.9 Treelet Affinity

Worker가 방금 Treelet A를 처리했다면 A의 다음 batch를 우선하는 정책은 L1/shared metadata reuse 가능성을 높인다.

단 affinity가 너무 강하면 hot treelet monopolization과 tail latency가 생길 수 있으므로 fairness와 함께 본다.

### 4.10 Hybrid BFS/DFS

Pure DFS는 locality가 좋지만 parallelism이 작고, BFS는 parallelism이 크지만 working set이 넓다.

GPU에서는:

```text
BFS across treelets
+
local DFS inside a treelet
```

같은 hybrid가 자연스럽다.

### 4.11 Persistent Worker + Treelet Batch

Persistent worker가 global queue에서 node 하나씩 꺼내는 대신:

```text
treelet batch dequeue
→ local traversal
→ child-treelet emission
```

으로 동작하면 load balance와 locality를 결합할 수 있다.

### 4.12 Adaptive Batch Size

큰 batch는 cache reuse가 좋지만 long tail을 만들 수 있다. 작은 batch는 fairness가 좋지만 queue overhead가 커진다.

Runtime signal:

- queue depth
- worker idle ratio
- task-cost variance
- L2 hit rate

를 이용해 batch size를 조절할 수 있다.

### 4.13 Shared-Memory Staging

Treelet의 root/upper metadata가 작다면 block shared memory에 stage해 여러 local task가 재사용할 수 있다.

다만 shared memory 사용량이 커지면 occupancy가 내려가므로 **cache-sized**와 **shared-memory-sized**는 다른 기준이다.

### 4.14 Treelet Prefetch

다음 batch가 Treelet B라는 사실을 이미 알고 있다면 B의 upper node를 일찍 prefetch할 수 있다.

2026년 TTP의 핵심 교훈도 같다.

> **Traversal stack/queue는 future address oracle이 될 수 있다.**

Software scheduler에서도 다음 batch의 metadata load를 현재 batch compute와 overlap하는 구조를 생각할 수 있다.

### 4.15 Treelet + Execution Class

Memory locality만 맞추면 warp divergence가 남을 수 있다.

Scheduling key를:

```text
(treeletId, taskClass)
```

로 만들면 locality와 control-flow coherence를 함께 노릴 수 있다.

하지만 queue fragmentation이 커질 수 있으므로 모든 축을 hard partition하지 않고 coarse hint로 사용한다.

### 4.16 Residency와 Treelet

Treelet 내부 일부 page가 non-resident일 수 있다. 이 경우 이전 노트의 원칙을 유지한다.

- resident subset은 진행
- missing child는 fallback
- request 발행
- committed snapshot만 consume

Treelet ID와 physical address를 동일시하지 않는다.

```text
TreeletId
→ residency mapping
→ physical page
```

처럼 late resolve하는 편이 relocation과 streaming에 안전하다.

### 4.17 LOD Traversal 적용

LOD DAG에서는:

```text
global root evaluation
→ treelet-local cut evaluation
→ survivor compaction
→ child-treelet frontier
→ render cluster list
```

로 만들 수 있다.

현재 NVIDIA `vk_lod_clusters`도 GPU에서 LOD hierarchy를 traverse해 renderable cluster list를 만들고 streaming과 같은 traversal path를 공유한다.

### 4.18 SDF / Semiconductor 적용

Sparse SDF에서는 XY tile 또는 Morton-prefix region을 treelet로 잡을 수 있다.

같은 region의 brick은:

- SDF pages
- material tables
- neighbor metadata
- thin-layer/column metadata

를 공유할 가능성이 높다.

반도체 geometry처럼 XY는 넓고 Z 방향 layer가 얇은 구조에서는 **XY tile treelet + local Z metadata**가 자연스러운 grouping이 될 수 있다.

### 4.19 C++ / Memory Layout

Persistent task에는 raw node pointer보다 logical ID를 보관한다.

```text
TreeletId
NodeId
TaskClass
TraversalEpoch
```

Hot scheduler metadata는 SoA로 분리한다.

```text
treeletId[]
nodeId[]
taskClass[]
priority[]
```

### 4.20 프로파일링

핵심 metric:

- hierarchy node visits
- treelet switches / 1K tasks
- tasks per batch
- L1/L2 hit rate
- hierarchy DRAM bytes
- memory-dependency stalls
- queue atomic count
- subgroup compaction ratio
- worker idle ratio
- same-treelet steal ratio
- reorder/compaction pass time
- p95/p99 batch runtime

가장 중요한 식은:

```text
Net Locality Gain
=
Traversal Time Saved
-
Reorder/Compaction Cost
```

이다.

## 5. 내 관심 분야와 연결

사용자의 dynamic SDF/CFD pipeline에서는 dirty work가 spatially clustered되는 경우가 많다.

```text
process front
→ dirty bricks
→ spatial compact
→ treelet-local remesh / LOD update
```

로 구성하면 같은 field page와 neighbor table을 재사용할 수 있다.

CFD AMR에서도 같은 block group을 연속 처리하면 field/neighbor metadata locality가 좋아진다.

GPU-driven rendering에서는:

```text
Hierarchy
→ treelet-local traversal
→ compact render clusters
→ mesh/task shader
```

로 연결할 수 있다.

이 주제는 GPU memory hierarchy, queue design, subgroup programming, persistent scheduling, BVH/LOD/meshlet 시스템을 한꺼번에 설명할 수 있어 graphics engineer 포트폴리오와 면접에도 강하다.

## 6. 머릿속에 남길 질문 3개

1. **Hierarchy node가 이미 Morton/DFS 순서로 잘 배치되어 있어도 runtime frontier reordering이 추가 이득을 낼 수 있는 이유는 무엇이며, 어떤 telemetry로 execution-order locality가 깨졌는지 확인할 수 있을까?**
2. **Treelet batch를 크게 하면 cache reuse는 좋아지지만 tail latency가 커질 수 있는데, queue depth·worker idle ratio·L2 hit rate를 이용해 adaptive batch size를 어떻게 결정할 수 있을까?**
3. **Dynamic SDF에서 XY treelet과 execution-path class를 동시에 사용하면 locality와 warp coherence를 함께 높일 수 있지만 queue fragmentation이 생긴다. 어떤 축을 primary key로 둘지 무엇을 기준으로 정해야 할까?**

## 7. graphics engineer 면접 질문 1개와 답변

### 질문

**“Hierarchy가 이미 spatially sorted되어 있으면 runtime traversal queue도 자동으로 cache-friendly한 것 아닌가요?”**

### 답변

아니다.

Physical layout은 **주소 공간의 locality**를 만들고 scheduler는 **시간 순서의 locality**를 결정한다.

Treelet A의 node가 memory에서 연속 배치되어 있어도 실행 순서가:

```text
A → F → Q → B → A
```

라면 A를 다시 방문할 때 관련 cache line이 이미 eviction되었을 수 있다.

그래서 production traversal에서는 두 층을 분리한다.

```text
Layout locality:
Morton/DFS order, compact wide nodes

Execution locality:
treelet-aware frontier, subgroup compaction,
per-treelet batching, locality-preserving stealing
```

다만 reordering 비용도 있으므로 full sort를 매 level 수행하는 것은 보통 과하다. Bucket/radix partition이나 local compaction처럼 cheaper operation을 사용하고 L2 hit rate, hierarchy DRAM bytes, treelet-switch rate, reorder cost, total traversal time을 함께 측정한다.

핵심은:

> **좋은 hierarchy layout은 locality의 가능성을 만들고, 좋은 scheduler가 그 locality를 실제 실행 시간에 보존한다.**

## 8. 포트폴리오 / 커리어 연결

포트폴리오에서는 다음처럼 설명할 수 있다.

> **“Spatially compact hierarchy layout만으로 runtime locality가 유지되지 않아 traversal frontier를 treelet key로 bucketed compaction했습니다. Same-treelet task를 cache-sized batch로 persistent worker에 공급하고 subgroup ballot/prefix-sum으로 child queue append를 집계했습니다. Scheduling overhead를 포함한 net gain을 treelet-switch rate, L2 hit ratio, hierarchy DRAM bytes, p99 batch tail로 검증했습니다.”**

연결 키워드:

- GPU memory hierarchy
- persistent worker
- work queue / work stealing
- Vulkan subgroup / CUDA warp
- LOD/BVH traversal
- meshlet worklist
- sparse SDF / AMR
- cache-aware data layout

## 9. 내일 이어서 볼 개념

**GPU Traversal Prefetch and Latency Hiding: Software Pipelines, Async Copies, and Future-Node Prediction**

오늘은 frontier를 treelet 기준으로 모아 execution locality를 높였다.

다음 질문은:

> **다음 방문 node가 어느 정도 예측 가능하다면 실제 load가 필요해지기 전에 metadata를 가져와 memory latency를 숨길 수 있는가?**

다음 노트에서는 traversal stack/queue as future-address oracle, software prefetch distance, shared-memory staging, double buffering, prefetch accuracy vs cache pollution, DFS/BFS 차이, 2026 TTP 연구를 CUDA/Vulkan compute 관점에서 이어간다.

## 10. 참고 키워드

- Treelet Scheduling
- Frontier Locality
- Cache-Sized Work Batch
- Traversal Frontier
- Queue Reordering
- Traversal Compaction
- Treelet Affinity
- Locality-Preserving Work Stealing
- Subgroup Ballot
- Warp Prefix Sum
- Aggregated Queue Reservation
- Persistent Workers
- Hybrid BFS / DFS
- Treelet Switch Rate
- L1 / L2 Cache Locality
- Memory Dependency Stall
- Morton Ordering
- Traversal Prefetch
- TTP / Tree Traversal Prefetcher
- JZ-Tree
- BVH
- LOD Hierarchy
- Meshlet
- Sparse SDF
- AMR
- NVIDIA nvpro-samples — **vk_lod_clusters**
  - https://github.com/nvpro-samples/vk_lod_clusters
- Timo Aila, Tero Karras — **Architecture Considerations for Tracing Incoherent Rays**, HPG 2010
  - https://research.nvidia.com/publication/2010-06_architecture-considerations-tracing-incoherent-rays
- Yavuz Selim Tozlu, Anshul Naithani, Huiyang Zhou — **TTP: A Hardware-Efficient Design for Precise Prefetching in Ray Tracing**, 2026
  - https://arxiv.org/abs/2605.16253
- Jens Stücker et al. — **JZ-Tree: GPU friendly neighbour search and friends-of-friends with dual tree walks in JAX plus CUDA**, 2026
  - https://arxiv.org/abs/2604.05885
- Kirill Garanzha, Jacopo Pantaleoni, David McAllister — **Simpler and Faster HLBVH with Work Queues**
  - https://research.nvidia.com/publication/simpler-and-faster-hlbvh-work-queues
