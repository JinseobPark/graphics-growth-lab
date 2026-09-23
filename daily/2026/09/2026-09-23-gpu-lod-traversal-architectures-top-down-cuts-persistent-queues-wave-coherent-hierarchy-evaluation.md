---
title: "GPU LOD Traversal Architectures: Top-Down Cuts, Persistent Queues, and Wave-Coherent Hierarchy Evaluation"
date: "2026-09-23"
category: Graphics
tags: [GPU, Rendering, GPU-Driven Rendering, LOD, Hierarchy Traversal, Persistent Kernel, Work Queue, Subgroup, Warp Coherence, Meshlet, Cluster LOD, Vulkan, CUDA, Memory Layout, C++]
level: intermediate
---

# [Daily Graphics Growth] 2026-09-23 - GPU LOD Traversal Architectures: Top-Down Cuts, Persistent Queues, and Wave-Coherent Hierarchy Evaluation

## 1. 오늘의 개념

어제는 **GPU LOD Error Metrics**에서 continuous LOD hierarchy/DAG의 각 node가 가진 object-space error를 screen-space error로 투영하고, semantic importance와 refine/coarsen hysteresis를 결합해 **어떤 region을 더 세밀하게 봐야 하는가**를 판단했다.

오늘은 그 판단을 수백만 node에 실제로 적용하는 **GPU LOD traversal architecture**를 본다.

핵심 질문은 다음이다.

> **수백만 개의 LOD node 중 현재 frame에서 필요한 cut만 빠르게 찾아내면서, traversal divergence·global queue contention·subgroup under-utilization을 어떻게 줄일 것인가?**

LOD traversal은 단순 tree walk가 아니다.

Runtime은 동시에 다음을 수행한다.

```text
LOD hierarchy / DAG
    ↓
view-dependent error test
    ↓
subtree prune or descend
    ↓
residency feasibility test
    ↓
renderable cluster output
    ↓
streaming/refinement request output
```

즉 하나의 compute pass가 사실상 다음 역할을 모두 수행하는 **GPU-side graph selection stage**다.

- spatial culling
- error-bounded LOD cut selection
- residency-aware fallback
- child expansion
- render worklist generation
- refinement request generation

현재 NVIDIA `vk_lod_clusters`의 runtime도 GPU에서 LOD hierarchy를 traverse해 `renderClusterInfos`를 만들며, traversal은 **multi-pass frontier 방식**과 **persistent producer/consumer queue 방식**을 모두 지원한다. 또한 hierarchy node traversal과 leaf/group 처리를 서로 다른 kernel로 분리해 divergence를 줄이는 구조를 사용한다.

이 점은 최근 며칠간 본 주제들을 그대로 연결한다.

```text
LOD Error Metric
    ↓
Hierarchy Traversal
    ↓
Persistent Queue / Frontier
    ↓
Wave-Coherent Evaluation
    ↓
Renderable Cut
```

---

## 2. 한 줄 핵심

> GPU LOD traversal의 핵심은 **top-down error pruning으로 search space를 줄이고, node expansion은 subgroup-friendly batch로 처리하며, workload가 작을 때는 multi-pass frontier를, irregular depth와 긴 tail이 클 때는 persistent queue를 선택해 traversal cost와 scheduling overhead를 균형 잡는 것**이다.

---

## 3. 왜 중요한가

Continuous LOD는 object마다 LOD index 하나를 고르는 방식이 아니다.

한 mesh 안에서도:

```text
near region  → fine clusters
far region   → coarse clusters
```

가 동시에 선택될 수 있다.

즉 runtime은 hierarchy/DAG를 탐색해 **현재 frame의 valid cut**을 생성해야 한다.

문제는 hierarchy 전체 node 수가 매우 클 수 있다는 점이다.

예:

```text
10 M source triangles
→ hundreds of thousands of clusters
→ many LOD groups
→ spatial hierarchy nodes
```

모든 node를 매 frame brute-force로 test하면 continuous LOD가 geometry cost를 줄이기도 전에 traversal 자체가 병목이 된다.

그래서 좋은 traversal은 다음 세 가지를 동시에 만족해야 한다.

### Search Efficiency

```text
불필요한 subtree를 가능한 빨리 제거한다.
```

### GPU Execution Efficiency

```text
같은 subgroup/warp가 비슷한 traversal path를 처리한다.
```

### Scheduling Efficiency

```text
irregular depth와 fan-out을 처리하면서 global atomic/queue overhead를 제어한다.
```

이 세 축 중 하나만 최적화하면 전체 성능이 오히려 나빠질 수 있다.

---

## 4. 구현 관점

### 4.1 Top-Down Traversal

가장 기본적인 구조는 root에서 시작한다.

```text
Root
 ↓ error too high?
Children
 ↓
Grandchildren
```

Node마다:

```text
bounds
max/cumulative error
child range
```

를 갖고 있다고 하자.

Runtime test:

```text
if projectedError <= threshold:
    stop descending
else:
    descend children
```

이 구조의 성능을 결정하는 핵심은 **subtree prune ratio**다.

많은 subtree가 상위 node에서 탈락하면 traversal cost는 geometry size보다 훨씬 작아진다.

### 4.2 Traversal은 “Node가 선택되는가?”보다 “더 내려갈 필요가 있는가?”를 묻는다

Hierarchy traversal에서 internal node test의 목적은 render 여부가 아니다.

질문은:

```text
이 subtree 안에서 더 fine한 representation이 필요할 가능성이 있는가?
```

이다.

그래서 internal node에 저장되는 error는 child들의 conservative worst-case를 포함해야 한다.

이전 노트의 monotonic cumulative error가 여기서 직접 사용된다.

### 4.3 Early Exit가 Traversal의 핵심이다

Node error가 threshold보다 이미 작다면:

```text
subtree 전체는 충분히 fine하다
```

고 판단해 더 내려가지 않는다.

반대로 너무 coarse한 hierarchy level을 parallel hierarchy처럼 구성한 경우에도 전체 level tree를 빨리 종료할 수 있다.

현재 `nv_cluster_lod_builder`/`vk_lod_clusters`도 spatial hierarchy node가 child들의 maximum error와 bounding volume을 보유하고, angular-error test를 이용해 search space를 줄인다.

### 4.4 Stack-Based Traversal

Thread 하나가 하나의 object/instance hierarchy를 탐색하는 가장 직관적인 방식이다.

```text
push root
while stack not empty:
    node = pop
    if descend:
        push children
    else:
        output
```

장점:

- global queue 불필요
- thread-local reasoning 쉬움
- shallow/small hierarchy에서 효율적

단점:

- thread마다 traversal depth가 달라 warp divergence 증가
- local stack 크기 필요
- 한 instance가 매우 heavy하면 한 thread가 long tail

즉 **instance-level work가 균일할 때** 적합하다.

### 4.5 Per-Warp Traversal

한 warp/subgroup가 하나 또는 여러 node batch를 협력해서 처리할 수 있다.

```text
warp loads node batch
→ lanes evaluate children
→ ballot active children
→ compact next frontier
```

장점:

- child test parallelism
- ballot/prefix sum으로 compact 가능
- coherent memory access

단점:

- fan-out이 작으면 lane 낭비
- 한 warp가 하나의 heavy object에 오래 묶일 수 있음

GPU hierarchy traversal은 **task granularity와 subgroup width가 얼마나 잘 맞는가**가 중요하다.

### 4.6 Wave-Coherent Evaluation

가장 중요한 optimization 중 하나는 같은 subgroup에서 가능한 한 동일한 stage의 work를 처리하는 것이다.

나쁜 예:

```text
lane 0 → internal node
lane 1 → leaf group
lane 2 → culling fail
lane 3 → streaming request
```

좋은 구조는 node traversal과 leaf/group processing을 분리한다.

현재 `vk_lod_clusters`의 `traversal_run.comp.glsl` 설명도 hierarchy node traversal과 leaf cluster-group 처리를 **두 kernel로 분리하며**, 이 구조가 divergence를 줄일 수 있다고 명시한다.

즉 이전에 배운 **Execution-Path-Aware Queueing**이 LOD traversal에도 그대로 적용된다.

### 4.7 Node Queue와 Group Queue를 분리한다

개념적으로:

```text
Node Frontier
→ internal traversal

Group Frontier
→ cluster/group evaluation

Render List
→ final clusters
```

이렇게 stage별 queue를 나누면:

- control flow coherence 증가
- descriptor/data layout specialization
- per-stage profiler 분리

가 가능하다.

### 4.8 Multi-Pass Frontier Traversal

가장 단순한 GPU breadth/depth hybrid 구조다.

```text
Pass N:
InputFrontier
→ evaluate
→ append children to OutputFrontier

Pass N+1:
swap frontiers
```

장점:

- kernel boundary가 명확한 global synchronization
- queue termination 단순
- per-pass count 정확
- overflow/diagnostics 쉬움

단점:

- hierarchy depth만큼 dispatch/passes 증가
- frontier가 작은 tail에서 GPU utilization 감소
- intermediate buffer ping-pong

### 4.9 Multi-Pass가 생각보다 강한 이유

Persistent kernel이 항상 더 빠른 것은 아니다.

LOD hierarchy는 많은 node가 early prune될 수 있기 때문에 실제 active depth가 작을 수 있다.

예:

```text
Depth 12 hierarchy
but most instances stop at depth 3~5
```

이라면 multi-pass dispatch 수가 많지 않고, kernel boundary가 주는 synchronization 단순성이 더 큰 이득일 수 있다.

### 4.10 Persistent Traversal Kernel

반대로 large irregular scene에서는 persistent worker가 queue를 계속 소비할 수 있다.

```text
fixed worker threads
    ↓
consume traversal task
    ↓
process children
    ↓
enqueue new tasks
    ↓
repeat until outstanding == 0
```

현재 NVIDIA sample의 persistent mode도 fixed amount of threads가 producer/consumer queue를 사용하고, `traversalTaskCounter`로 in-flight task 수를 추적해 0이 되면 종료한다.

이 구조는 이전의 **GPU Termination Detection**과 정확히 연결된다.

### 4.11 Persistent Kernel의 장점

- hierarchy depth와 무관한 single kernel residency
- active work가 있는 동안 worker 재사용
- irregular depth/load balance 개선
- dispatch overhead 감소

특히:

```text
많은 instance
+ 서로 다른 LOD depth
+ long tail
```

에서 강하다.

### 4.12 Persistent Kernel의 비용

- global queue atomic
- producer/consumer contention
- forward-progress reasoning
- termination counter
- queue capacity
- worker count tuning

현재 `vk_lod_clusters` README도 persistent traversal thread 수가 아직 crude heuristic 기반이며 optimal amount로 평가되지 않았다고 명시한다.

즉 persistent traversal은 **algorithmic win이 아니라 scheduling trade-off**다.

### 4.13 Frontier vs Persistent 선택 기준

#### Multi-Pass가 유리할 수 있는 경우

```text
small active hierarchy depth
large coherent frontiers
simple debugging 중요
queue atomic 비용이 큼
```

#### Persistent가 유리할 수 있는 경우

```text
irregular depth
long tail
many instances
frequent child expansion
launch/pass overhead가 큼
```

둘 다 구현해 profiler로 scene-dependent 선택하는 것도 합리적이다.

### 4.14 Persistent Queue는 Append-Only Linear Queue로도 가능하다

Traversal은 일반적인 cyclic MPMC queue가 아니라:

```text
readCounter
writeCounter
linear task array
```

형태로 구현할 수 있다.

한 frame traversal에서 최대 task bound를 잡을 수 있다면 ring wraparound를 피하고 queue logic을 단순화할 수 있다.

이전 queue note에서 다룬:

```text
Append Stream vs Persistent Ring Queue
```

의 선택이 다시 등장한다.

### 4.15 Outstanding Task Counter

Persistent mode의 중요한 invariant:

```text
outstandingTraversalTasks
```

은 아직 시스템이 책임지는 traversal work 수를 의미해야 한다.

Parent가 child를 생성한다면:

```text
child responsibility register
→ parent task retire
```

순서를 보장해야 false-zero termination을 막을 수 있다.

### 4.16 Subgroup Register Task Packing

현재 NVIDIA traversal shader는 input tasks를 subgroup register에 보관하고 필요한 field를 `subgroupShuffle`로 lane 간 공유하는 구조를 사용한다.

개념적으로:

```text
lane group holds task headers
child lanes fetch parent task fields via shuffle
```

이 방식은 같은 parent의 child를 여러 lane이 처리할 때 global/shared memory round-trip을 줄일 수 있다.

### 4.17 Child Expansion을 Lane에 Mapping한다

Parent node fan-out이 `N`이라면:

```text
lane 0 → child 0
lane 1 → child 1
...
```

처럼 mapping할 수 있다.

문제는 fan-out이 subgroup width보다 작으면 lane utilization이 떨어지는 것이다.

따라서 여러 parent의 child를 같은 subgroup에 pack하는 **work packing**이 중요하다.

### 4.18 Multi-Task Subgroup Packing

한 subgroup가 여러 parent의 children을 동시에 처리한다.

```text
Parent A: 3 children
Parent B: 5 children
Parent C: 8 children

→ 16 lanes에 compact
```

필요한 정보:

```text
which parent?
which child index?
```

ballot + prefix sum + shuffle 조합으로 구현할 수 있다.

이것은 irregular traversal에서 subgroup occupancy를 크게 개선할 수 있다.

### 4.19 Ballot + Prefix Sum

Traversal test 결과:

```text
lane active?
```

를 ballot으로 모은다.

```text
mask = ballot(descend)
rank = popcount(mask & lanesBeforeMe)
count = popcount(mask)
```

Leader가 한 번만 queue slots `count`개를 reserve하고 각 lane이:

```text
base + rank
```

에 child를 쓴다.

이전 note의 **warp-aggregated atomics**가 hierarchy traversal에도 그대로 적용된다.

### 4.20 Queue Reservation을 Subgroup 단위로 Aggregate

Naive:

```text
32 lanes
→ up to 32 atomicAdd(writeCounter)
```

Subgroup aggregate:

```text
1 atomicAdd(writeCounter, activeCount)
```

후 lane scatter.

Global atomic traffic을 크게 줄일 수 있다.

### 4.21 Traversal Divergence의 세 종류

#### Depth Divergence

lane마다 hierarchy depth가 다름.

#### Branch Divergence

culling/error/residency result가 다름.

#### Work-Type Divergence

node traversal / group evaluation / render output / request output이 섞임.

각각 다른 해법이 필요하다.

### 4.22 Depth Divergence → Dynamic Work Redistribution

Local stack thread가 너무 깊게 들어가는 대신 child를 global/local queue로 내보내 다른 worker가 가져가게 할 수 있다.

즉 traversal과 work stealing이 연결된다.

하지만 너무 작은 child까지 queue로 내보내면 scheduler overhead가 커진다.

### 4.23 Hybrid Local Stack + Global Queue

좋은 중간 구조:

```text
common path:
small local stack

stack pressure / large branch:
spill children to global queue
```

장점:

- local DFS locality
- global load balance
- queue traffic 감소

이전의 local queue + spill architecture와 동일한 패턴이다.

### 4.24 DFS vs BFS

#### DFS

```text
recent children immediately process
```

장점:

- hierarchy/local data locality
- stack small 가능

단점:

- parallel frontier 작아질 수 있음

#### BFS

```text
same level broad frontier
```

장점:

- high parallelism
- similar error scale/path

단점:

- queue memory 큼
- locality 감소 가능

GPU에서는 **pure DFS/BFS보다 hybrid frontier**가 현실적이다.

### 4.25 Level-Synchronous Traversal의 장점

같은 LOD hierarchy level을 batch 처리하면:

- node layout locality
- similar error range
- similar branch behavior

를 얻을 수 있다.

하지만 continuous LOD는 region마다 필요한 depth가 달라 active level이 다양해질 수 있다.

### 4.26 DAG Traversal에서는 Duplicate Work에 주의

Hierarchy가 strict tree가 아니라 DAG relation을 포함하면 동일 group/node가 여러 path에서 reachable할 수 있다.

Runtime spatial hierarchy가 tree로 별도 구성되어 있다면 traversal duplicate를 줄일 수 있다.

중요한 분리:

```text
LOD relation DAG
≠
Traversal acceleration hierarchy
```

현재 `nv_cluster_lod_builder`도 runtime search를 위해 별도의 spatial hierarchy를 만든다.

### 4.27 Spatial Hierarchy가 LOD DAG를 가속한다

LOD DAG 자체를 그대로 traversal하면 dependency relation은 잘 표현하지만 spatial locality/pruning이 좋지 않을 수 있다.

그래서:

```text
LOD data
+ spatial hierarchy over cluster groups
```

구조를 사용할 수 있다.

Spatial node는:

```text
bounds
max error
cluster/group range
```

를 가지고 early prune를 지원한다.

### 4.28 Traversal Acceleration Structure는 Runtime Query 전용이다

Offline LOD DAG:

```text
representation generation relation
```

Runtime traversal hierarchy:

```text
fast view-dependent search
```

역할이 다르다.

이 둘을 하나로 강제로 합치면 offline simplification relation 때문에 runtime memory layout이 나빠질 수 있다.

### 4.29 Wide Node

GPU hierarchy는 binary보다 4/8-way가 유리할 수 있다.

```text
one node fetch
→ multiple child tests in subgroup
```

장점:

- depth 감소
- lane parallelism

단점:

- node size 증가
- child test bandwidth 증가

Optimal fan-out은:

```text
subgroup width
cache line
node metadata size
branch selectivity
```

에 의존한다.

### 4.30 SoA vs AoS

Traversal hot data:

```text
bounds[]
error[]
childOffset[]
childCount[]
```

를 SoA로 두면 error-only/culling-only path에서 필요한 field만 읽을 수 있다.

반대로 모든 test가 항상 같은 metadata를 읽는다면 compact AoS node가 cache line 효율이 좋을 수 있다.

즉 layout은 shader access pattern으로 결정한다.

### 4.31 Node Compression

수백만 node에서 metadata bandwidth가 traversal bottleneck이 될 수 있다.

가능한 packing:

```text
quantized bounds
FP16/log error
packed child offset/count
compact flags
```

단 error/bounds는 conservative 방향으로 quantize해야 false-coarse selection을 막는다.

### 4.32 Traversal은 Memory-Latency Bound가 되기 쉽다

Node test 자체 arithmetic은 가볍다.

```text
load metadata
few matrix/math ops
branch
```

따라서 실제 병목은:

- random node fetch
- queue traffic
- atomic counter
- divergent loads

일 가능성이 높다.

GPU traversal optimization은 ALU보다 **memory locality + coherence**가 더 중요할 때가 많다.

### 4.33 Spatial Ordering

Nodes/cluster groups를 Morton/Hilbert-like spatial order로 배치하면 nearby camera region의 traversal이 유사한 memory region을 접근할 가능성이 높다.

특히 multiple nearby instances/regions를 batch 처리할 때 cache reuse가 좋아질 수 있다.

### 4.34 Instance Sorting

현재 `vk_lod_clusters`는 optional instance sorting에 `vulkan_radix_sort`를 사용한다.

General lesson:

```text
similar instance / spatial region / LOD state
```

를 가까이 배치하면 traversal coherence를 높일 수 있다.

단 sorting cost보다 traversal gain이 커야 한다.

### 4.35 Rasterization과 Ray Tracing Traversal은 동일하지 않다

Rasterization:

- frustum culling 강력
- Hi-Z occlusion culling 활용 가능
- render cluster list 직접 사용

Ray tracing:

- off-screen geometry가 reflection/shadow에 필요할 수 있음
- selected cluster set이 BLAS/CLAS build cost에 연결

현재 NVIDIA sample도 hierarchy traversal/streaming core는 공유하지만 ray tracing에서는 이후 BLAS/CLAS building 단계가 추가된다.

### 4.36 Traversal Output이 Downstream Cost를 결정한다

LOD traversal 결과의 cluster 수가:

```text
raster mesh shader work
or
ray tracing AS build work
```

를 결정한다.

따라서 traversal threshold optimization은 단순 compute pass timing만 보면 안 된다.

### 4.37 Desired Cut와 Resident Cut을 한 Traversal에서 계산할 수 있다

Node test에서:

```text
error says refine
but child non-resident
```

이면:

```text
DesiredCut → child
ResidentCut → parent/fallback
Request → child
```

세 결과를 동시에 만들 수 있다.

즉 traversal output:

```text
RenderList
StreamingRequestList
RefinementDebt
```

를 함께 생성할 수 있다.

### 4.38 Separate Traversal로 나누는 방법도 있다

#### Pass A

```text
error-only desired cut
```

#### Pass B

```text
residency resolve / fallback
```

장점:

- stage 단순
- debug 쉬움

단점:

- hierarchy metadata 재접근
- extra pass/worklist

통합/분리는 profiler 기반 선택이다.

### 4.39 Traversal Worklist는 Compact Logical IDs가 좋다

Queue item:

```text
instanceID
nodeID
small flags
```

정도로 유지한다.

Full bounds/error payload를 queue에 복사하기보다 persistent hierarchy table에서 resolve한다.

이전 queue note의:

```text
Queue = identity / intent
Table = payload
```

원칙을 그대로 적용한다.

### 4.40 Persistent Worklist와 Raw Pointer

Traversal queue에 physical node address를 bake하면 defrag/layout rebuild에 취약하다.

Stable logical node/index를 사용하면 hierarchy storage relocation과 traversal work를 분리할 수 있다.

### 4.41 Generation / Epoch

Dynamic hierarchy rebuild가 있다면:

```text
NodeHandle { index, generation }
HierarchyEpoch
```

를 사용할 수 있다.

Stale traversal task가 previous hierarchy를 참조하면 reject한다.

Dynamic SDF에서는 중요한 문제다.

### 4.42 Dynamic SDF에서는 Hierarchy Update와 Traversal이 Overlap할 수 있다

Geometry producer가 hierarchy E+1을 만들고 renderer가 E를 traversal 중일 수 있다.

따라서:

```text
Traversal Snapshot E
Hierarchy Build E+1
```

를 분리해야 한다.

어제까지의 immutable snapshot architecture가 여기서 다시 필요하다.

### 4.43 Traversal Snapshot에는 무엇이 들어가나

최소:

- hierarchy node table
- bounds/error table
- residency mapping
- cluster/group table
- generation/epoch

가 서로 compatible해야 한다.

### 4.44 Persistent Queue Capacity

Worst-case refine에서 많은 child가 동시에 expand될 수 있다.

Queue size를 average frontier만 보고 잡으면 overflow가 생긴다.

Metrics:

```text
frontier size histogram
max write-read distance
child fan-out distribution
```

을 봐야 한다.

### 4.45 Queue Overflow Policy

LOD traversal은 overflow 시 work를 drop하면 geometry hole이 생길 수 있다.

대안:

- spill buffer
- multi-pass fallback
- forced coarse fallback

특히 마지막 방법은 geometry hierarchy 특유의 장점이다.

```text
queue pressure high
→ stop refinement
→ retain coarse node
```

이면 correctness를 유지하면서 quality만 낮출 수 있다.

### 4.46 Traversal Backpressure를 Quality Policy로 연결한다

이것은 매우 중요한 system-level 아이디어다.

```text
Traversal Queue Pressure ↑
→ refine admission ↓
→ coarser cut
→ task count ↓
```

즉 queue pressure를 단순 overflow bug가 아니라 **quality control signal**로 사용할 수 있다.

### 4.47 Budget-Aware Traversal

Traversal 자체에 frame budget을 줄 수 있다.

```text
max nodes visited
max output clusters
max refinement requests
```

Budget을 넘으면 coarse representation을 유지한다.

이렇게 하면 worst-case camera cut에서도 traversal spike를 제한할 수 있다.

### 4.48 Node Visit Budget의 위험

단순히 node count에서 중단하면 중요한 near-camera region이 refine되지 않을 수 있다.

따라서 priority:

```text
projected error
semantic importance
```

가 높은 subtree부터 process해야 한다.

즉 traversal frontier도 priority queue 개념으로 확장될 수 있다.

### 4.49 Priority Queue의 비용

정확한 heap priority queue는 GPU에서 비쌀 수 있다.

대안:

```text
error buckets
HOT / WARM / COLD
quantized priority classes
```

같은 bucketed frontier를 사용할 수 있다.

### 4.50 Wave-Coherent Priority Buckets

비슷한 error/LOD level의 node를 같은 queue에 넣으면:

- similar branch behavior
- similar depth
- similar residency pattern

을 기대할 수 있다.

Execution-Path-Aware Queueing이 hierarchy traversal priority scheduling으로 확장된다.

### 4.51 Persistent Worker Count

너무 적으면:

```text
parallelism 부족
```

너무 많으면:

```text
queue atomic contention
register/shared memory pressure
other GPU work와 concurrency 감소
```

가 생긴다.

따라서 occupancy 최대화가 아니라 **throughput × contention × concurrency**를 봐야 한다.

### 4.52 Compute와 Graphics Overlap

Traversal compute가 graphics와 overlap한다면 persistent kernel이 많은 SM resource를 장시간 점유하는 것이 graphics latency에 악영향을 줄 수 있다.

Multi-pass short kernels가 scheduler에게 더 많은 preemption/interleaving opportunity를 줄 수 있다.

즉 persistent traversal은 async-compute 환경에서 특히 측정이 필요하다.

### 4.53 Profiling에서 볼 지표

핵심 traversal metrics:

- root seeds / frame
- nodes visited
- leaf/groups visited
- clusters emitted
- node prune ratio
- average traversal depth
- p95/p99 traversal depth
- frontier size histogram
- max frontier size
- node queue high-water
- subgroup active lane ratio
- subgroup ballot utilization
- queue atomics / visited node
- queue bytes / frame
- local stack hit / spill ratio
- persistent worker occupancy
- task counter tail latency
- desired cut size
- resident cut size
- fallback count
- refinement request count
- render list bytes
- hierarchy metadata bytes read
- L1/L2 hit rate
- traversal GPU ms
- downstream mesh shader/AS build ms

특히 다음 네 개를 함께 봐야 한다.

```text
Nodes Visited / Rendered Cluster
Subgroup Active Lane Ratio
Queue Atomics / Node
Traversal GPU Time
```

그리고 end-to-end로는:

```text
Traversal GPU Time
+
Downstream Render/AS Cost
```

를 봐야 한다.

---

## 5. 내 관심 분야와 연결

### Semiconductor Process Visualization

사용자의 dynamic semiconductor scene에서는 traversal priority가 generic game scene보다 domain-specific할 수 있다.

예:

```text
selected trench
cross-section plane
thin oxide layer
active process front
```

이 region의 hierarchy subtree를 먼저 traversal/refine할 수 있다.

즉 root seed나 frontier priority에 semantic importance를 반영할 수 있다.

### Dynamic SDF / Level Set

Pipeline:

```text
Sparse SDF hierarchy
    ↓
Geometry LOD hierarchy
    ↓
GPU traversal
    ↓
Resident cut
    ↓
Meshlet worklist
```

Fine geometry가 non-resident면 traversal이 parent/fallback을 출력하면서 remesh request를 동시에 생성할 수 있다.

### ColumnStack / Thin Layers

XY spatial tile 기준 hierarchy와 Z material/layer metadata를 분리하면:

- traversal은 XY spatial coherence
- semantic check는 thin-layer importance

로 역할을 나눌 수 있다.

### CFD

Adaptive mesh/refinement region에서도 hierarchy traversal은:

- projected error
- scalar gradient
- shock/front proximity

를 기준으로 subtree를 선택하는 방식으로 확장할 수 있다.

### CUDA / Vulkan

가능한 역할 분리:

```text
CUDA/Warp:
dynamic hierarchy build / mesh generation

Vulkan Compute:
LOD traversal / residency resolve / worklist build

Vulkan Graphics:
mesh/task shader rendering
```

또는 CUDA persistent kernel에서 traversal 자체를 수행하고 Vulkan에는 final render list만 publish할 수도 있다.

### Graphics / Game Engine Career

이 개념은 다음 역할과 직접 연결된다.

- Nanite-like virtualized geometry
- meshlet/cluster culling
- GPU-driven scene traversal
- hierarchical culling
- virtual geometry streaming
- ray tracing LOD
- persistent compute runtime

면접에서 강한 포인트는:

> **“Tree traversal을 GPU에서 병렬화한다”**

정도가 아니라:

> **“stack, frontier, persistent queue의 trade-off와 subgroup divergence, queue pressure, downstream work amplification까지 설명하는 것”**

이다.

---

## 6. 머릿속에 남길 질문 3개

1. **LOD hierarchy가 shallow하지만 instance 수가 매우 많을 때 persistent queue traversal보다 multi-pass frontier가 더 유리할 수 있는 이유를 queue atomic·kernel boundary·frontier parallelism 관점에서 어떻게 설명할 수 있을까?**
2. **Node traversal과 leaf/group evaluation을 서로 다른 queue/kernel로 분리하면 divergence는 줄지만 intermediate worklist traffic은 늘어나는데, subgroup active-lane ratio와 memory bandwidth 중 어느 병목이 우선인지 어떻게 판단할까?**
3. **Dynamic SDF hierarchy에서 traversal queue pressure가 급증할 때 work를 drop하는 대신 coarse fallback을 유지하도록 refine admission을 줄이는 것이 correctness와 frame stability 측면에서 왜 좋은가?**

---

## 7. graphics engineer 면접 질문 1개와 답변

### 질문

**“GPU LOD hierarchy traversal은 persistent kernel 하나로 돌리는 것이 multi-pass compute보다 항상 더 빠르지 않나요?”**

### 답변

항상 그렇지는 않다.

Persistent kernel은 fixed worker가 global work queue를 계속 소비하기 때문에 irregular hierarchy depth와 long tail을 잘 처리하고 kernel dispatch overhead를 줄일 수 있다.

하지만 그 대가로:

```text
queue atomics
producer/consumer contention
termination tracking
persistent resource residency
worker-count tuning
```

이 필요하다.

반면 multi-pass frontier는:

```text
frontier N
→ evaluate
→ frontier N+1
```

처럼 kernel boundary마다 global synchronization과 exact count를 얻을 수 있고, hierarchy active depth가 작거나 frontier가 넓고 coherent하면 매우 효율적일 수 있다.

또 persistent kernel이 많은 SM resource를 계속 점유하면 async graphics/compute overlap에서 다른 workload의 latency를 높일 수도 있다.

현재 NVIDIA `vk_lod_clusters`의 traversal shader도 두 방식을 모두 지원한다. Multi-pass에서는 pass마다 input traversal node를 읽어 다음 frontier를 append하고, persistent mode에서는 fixed thread가 producer/consumer queue와 in-flight task counter를 사용한다.

따라서 선택 기준은:

```text
Hierarchy depth distribution
Frontier size
Queue atomic cost
Long-tail severity
Async overlap requirement
```

이다.

그리고 node/leaf stage를 분리하거나 subgroup ballot/compaction을 이용해 warp coherence를 높이는 것이 traversal architecture 선택만큼 중요하다.

핵심은:

> **Persistent traversal은 “더 최신인 방식”이 아니라, irregular scheduling 비용을 queue overhead와 맞바꾸는 방식이다.**

---

## 8. 포트폴리오 / 커리어 연결

이 주제는 graphics engineer가 hierarchy algorithm뿐 아니라 **GPU execution model과 scheduler design**을 이해한다는 것을 보여주기 좋다.

### GPU-Driven Rendering

- hierarchy traversal
- render worklist generation
- meshlet/cluster selection
- culling + LOD fusion

### GPU Runtime

- persistent kernel
- producer/consumer queue
- frontier traversal
- termination counter
- queue backpressure

### SIMT / Shader

- subgroup ballot
- shuffle
- compact child expansion
- active lane utilization
- divergence-aware stages

### Memory Layout

- SoA hierarchy metadata
- node compression
- spatial ordering
- compact logical work item

### Dynamic Geometry

- hierarchy epoch
- resident cut
- fallback output
- remesh/refinement request

### C++ / Engine Architecture

- strong node/group IDs
- traversal policy
- multi-pass/persistent selectable backend
- profiler-driven scheduling mode

포트폴리오에서는 다음처럼 설명할 수 있다.

> **“GPU continuous-LOD traversal을 top-down error pruning으로 구성하고, node traversal과 leaf/group evaluation을 분리해 subgroup divergence를 줄였습니다. Scene에 따라 multi-pass frontier와 persistent producer/consumer traversal을 선택할 수 있게 했으며, subgroup ballot으로 child queue reservation을 aggregate했습니다. Queue pressure가 증가하면 fine refinement를 억제하고 coarse cut을 유지해 traversal overflow가 geometry correctness 문제로 전파되지 않도록 했습니다.”**

이 설명은 rendering algorithm, shader execution, queue design, memory hierarchy, dynamic geometry를 하나로 연결한다.

---

## 9. 내일 이어서 볼 개념

**GPU Hierarchy Traversal Memory Layout: Wide BVHs, Quantized Nodes, and Cache-Coherent Cluster Ordering**

오늘은 traversal scheduling architecture를 봤다.

다음 질문은:

> **같은 traversal algorithm이라도 node memory layout과 ordering을 바꾸면 왜 성능이 크게 달라지는가?**

학습 흐름:

```text
LOD Error Metric
→ GPU Hierarchy Traversal
→ Traversal Memory Layout
→ Cache-Coherent Worklist
```

다음 노트에서는:

- binary vs wide hierarchy
- AoS vs SoA
- quantized bounds/error
- child range packing
- Morton/spatial ordering
- cache line utilization
- pointerless hierarchy
- breadth/depth linearization
- subgroup-friendly node fetch
- traversal bandwidth model

을 중심으로 이어간다.

---

## 10. 참고 키워드

- GPU LOD Traversal
- Hierarchy Traversal
- Top-Down Traversal
- Continuous LOD
- Cluster LOD
- Meshlet Hierarchy
- DAG Cut
- Spatial Hierarchy
- Multi-Pass Frontier
- Persistent Traversal Kernel
- Producer / Consumer Queue
- Outstanding Task Counter
- Subgroup / Warp
- `subgroupBallot`
- `subgroupShuffle`
- Warp-Aggregated Atomics
- Child Compaction
- Traversal Divergence
- Depth Divergence
- Work-Type Divergence
- Local Stack
- Global Work Queue
- Hybrid DFS / BFS
- Frontier Compaction
- Queue Backpressure
- Desired Cut / Resident Cut
- Render Cluster List
- Streaming Request List
- Hierarchy Epoch
- SoA Node Metadata
- Wide Hierarchy
- Quantized Bounds
- Quantized Error
- RTX Mega Geometry
- `VK_NV_cluster_acceleration_structure`
- `VK_EXT_mesh_shader`
- Dynamic SDF
- GPU-Driven Rendering
- NVIDIA nvpro-samples — **vk_lod_clusters**
  - https://github.com/nvpro-samples/vk_lod_clusters
- NVIDIA `vk_lod_clusters` — **Runtime Rendering Operations**
  - https://github.com/nvpro-samples/vk_lod_clusters#runtime-rendering-operations
- NVIDIA `vk_lod_clusters` — **traversal_run.comp.glsl**
  - https://github.com/nvpro-samples/vk_lod_clusters/blob/main/shaders/traversal_run.comp.glsl
- NVIDIA nvpro-samples — **nv_cluster_lod_builder**
  - https://github.com/nvpro-samples/nv_cluster_lod_builder
- NVIDIA `vk_lod_clusters` — **LOD Generation Documentation**
  - https://github.com/nvpro-samples/vk_lod_clusters/blob/main/docs/lod_generation.md
- Brian Karis et al. — **A Deep Dive into Nanite Virtualized Geometry**, 2021
  - https://advances.realtimerendering.com/s2021/
