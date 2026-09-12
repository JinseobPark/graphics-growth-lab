---
title: "GPU Visibility Work Graphs: Persistent Queues, Device-Generated Commands, and Multi-Stage Work Amplification"
date: "2026-09-12"
category: Graphics
tags: [GPU, Rendering, GPU-Driven Rendering, Work Graphs, Persistent Queues, Device Generated Commands, Vulkan, DirectX 12, Meshlet, Mesh Shader, Indirect Dispatch, Work Amplification, Memory Layout, C++]
level: intermediate
---

# [Daily Graphics Growth] 2026-09-12 - GPU Visibility Work Graphs: Persistent Queues, Device-Generated Commands, and Multi-Stage Work Amplification

## 1. 오늘의 개념

최근 흐름은 다음처럼 이어졌다.

```text
Incremental GPU Meshing
  → Relocation-Safe GPU Rendering
  → Hierarchical Meshlet Visibility
  → Temporal Hi-Z
  → Occluder Selection
  → Front-to-Back Scheduling
```

어제까지는 이 과정을 비교적 명확한 **고정 pass chain(fixed pass chain)** 으로 생각했다.

```text
Cull
  → Compact
  → Depth Bin
  → Build Indirect Commands
  → Draw
```

오늘은 한 단계 더 올라가서 질문한다.

> **GPU가 어떤 work를 발견할 때마다 다음 단계의 work를 GPU 안에서 직접 생산하고, CPU가 중간 단계의 개수와 분기 구조를 몰라도 되는 실행 모델은 어떻게 설계해야 하는가?**

이 문제를 이해하기 위한 세 가지 축이 있다.

1. **Persistent GPU Queue**  
   Application이 직접 queue/counter를 GPU buffer에 만들고 compute shader가 work item을 소비·생산하는 방식.

2. **Indirect / Device-Generated Commands (DGC)**  
   GPU가 draw/dispatch와 일부 state를 표현하는 command stream을 만들고 API가 이를 실행하는 방식.

3. **Native Work Graphs**  
   Shader node가 다른 node의 work를 생성하며 runtime/device scheduler가 graph execution과 queueing을 관리하는 방식.

이 셋은 모두 CPU round-trip을 줄이지만 abstraction level이 다르다.

오늘의 핵심은 특정 API 하나를 외우는 것이 아니라 다음을 이해하는 것이다.

> **GPU-driven visibility가 커질수록 “command 생성”보다 “work 생성과 scheduling의 lifetime”이 더 중요한 문제로 바뀐다.**

Visibility pipeline을 work graph 관점으로 보면 다음처럼 표현할 수 있다.

```text
Scene Candidates
      ↓
Coarse Cull Node
      ↓ emits visible chunks
Meshlet Expand Node
      ↓ emits meshlets
Hi-Z Cull Node
      ↓ emits visible meshlets
LOD / Material Classify Node
      ↓ emits specialized work
Depth / Material Bin Node
      ↓ emits render packets
Draw / Mesh Task Node
```

어떤 chunk는 0개의 meshlet을 생산하고, 어떤 chunk는 수십 개를 생산한다. 어떤 meshlet은 culling에서 끝나고, 어떤 meshlet은 depth pass와 main pass 두 경로로 갈 수 있다.

이런 **data-dependent work amplification**이 오늘의 주제다.

---

## 2. 한 줄 핵심

> GPU visibility work graph의 핵심은 **CPU가 모든 pass의 개수와 work량을 미리 결정하지 않고, GPU가 logical work item을 단계적으로 consume/emit하도록 만들되 queue capacity, forward progress, synchronization, state lifetime을 명시적인 contract로 관리하는 것**이다.

---

## 3. 왜 중요한가

GPU-driven renderer가 단순할 때는 indirect draw만으로 충분하다.

```text
CPU
  → Dispatch Cull
GPU
  → VisibleCount
  → IndirectDrawBuffer
GPU
  → DrawIndirectCount
```

그러나 실제 renderer가 발전하면 중간 단계가 늘어난다.

```text
Instance Cull
  → Chunk Expand
  → Meshlet Cull
  → LOD Select
  → Material Classify
  → Occluder Pass
  → HZB Re-Cull
  → Front-to-Back Bin
  → Main Draw
```

여기서 fixed pass chain의 문제가 생긴다.

### 문제 1 — 대부분의 pass가 빈 work를 처리할 수 있다

예를 들어 90% object가 coarse culling에서 사라졌는데도 이후 pass가 전체 capacity buffer를 훑으면 GPU bandwidth가 낭비된다.

### 문제 2 — 각 stage마다 global buffer + counter + barrier가 생긴다

```text
Stage A write
  ↓ global barrier
Stage B read/write
  ↓ global barrier
Stage C read/write
```

Algorithm은 단순해도 synchronization과 memory traffic이 길어진다.

### 문제 3 — Work amplification이 동적이다

한 instance가 몇 개 meshlet을 만들지, 한 meshlet이 어느 LOD/material path로 갈지 CPU가 미리 알기 어렵다.

### 문제 4 — Shader specialization이 늘어난다

일부 geometry는:

- opaque
- alpha-tested
- depth-only
- wireframe
- selected/highlight
- heatmap
- clipping-enabled

같이 서로 다른 pipeline/state가 필요할 수 있다.

Classic indirect draw는 draw parameters를 GPU에서 만들기 쉽지만 **draw 사이의 state 변화**가 많아지면 표현력이 부족해질 수 있다.

이 점이 Vulkan의 `VK_EXT_device_generated_commands` 같은 기능이 등장한 이유 중 하나다. Vulkan 문서는 DGC가 GPU에서 culling, sorting, classification을 처리한 뒤 host readback 없이 state 변화와 draw/dispatch를 포함한 command stream을 실행하는 용도를 설명한다.

또 Direct3D 12 Work Graphs는 shader node가 다른 node의 invocation을 request할 수 있게 해, application이 GPU-side work graph의 queueing을 직접 전부 구현하지 않아도 되는 더 높은 abstraction을 제공한다.

즉 현대 GPU rendering의 방향은 다음처럼 볼 수 있다.

```text
CPU decides every draw
        ↓
GPU writes draw parameters
        ↓
GPU generates commands
        ↓
GPU dynamically generates work
```

---

## 4. 구현 관점

### 4.1 먼저 “command graph”와 “work graph”를 구분한다

둘은 비슷해 보이지만 다르다.

#### Command Graph 관점

```text
Draw A
Draw B
Dispatch C
Bind Shader D
Draw E
```

주 관심사는 **어떤 command를 실행할 것인가**다.

#### Work Graph 관점

```text
ChunkWork
  → emits MeshletWork
MeshletWork
  → emits VisibleWork
VisibleWork
  → emits DepthWork / MainWork
```

주 관심사는 **어떤 logical work item이 어떤 다음 work item을 생산하는가**다.

DGC는 command graph에 가깝고,
Persistent queue와 D3D12 Work Graphs는 work graph 사고방식에 더 가깝다.

둘을 함께 사용할 수도 있다.

```text
Work Graph
  → final render packet stream
  → DGC / indirect execution
```

---

### 4.2 Classic indirect pipeline은 “flat queue”다

가장 기본적인 GPU-generated work는 다음과 같다.

```text
Compute Cull
  → VkDrawIndexedIndirectCommand[]
  → count
  → vkCmdDrawIndexedIndirectCount
```

이 모델은 매우 강력하다.

장점:

- 구현이 단순
- 디버깅 쉬움
- GPU count 기반 실행
- CPU readback 없음

하지만 모든 work item이 같은 command structure를 사용하고, draw마다 shader/state를 다양하게 바꾸기 어려운 경우가 있다.

즉 classic indirect는 **homogeneous flat work stream**에 특히 잘 맞는다.

---

### 4.3 Vulkan DGC는 command stream의 표현력을 확장한다

현재 Vulkan의 ratified `VK_EXT_device_generated_commands`는 다음 개념을 제공한다.

```text
VkIndirectCommandsLayoutEXT
VkIndirectExecutionSetEXT
VkGeneratedCommandsInfoEXT
vkCmdPreprocessGeneratedCommandsEXT
vkCmdExecuteGeneratedCommandsEXT
```

`VkIndirectCommandsLayoutEXT`는 sequence 안에서 어떤 token을 실행할지 정의한다.

예를 들어 logical sequence는 다음처럼 생각할 수 있다.

```text
Bind shader/pipeline variant
Set push constants
Bind vertex/index state
Execute draw
```

중요한 제약은 **하나의 sequence가 고정된 homogeneous layout을 가진다**는 것이다.

즉 DGC는 arbitrary CPU command buffer를 GPU가 자유롭게 만드는 모델이 아니라, 미리 정의된 command grammar 안에서 GPU가 sequence data를 채우는 모델이다.

---

### 4.4 DGC에서 preprocessing은 별도 scheduling stage다

Vulkan DGC는 implementation이 input stream을 device-specific executable form으로 바꾸기 위해 preprocess buffer를 요구할 수 있다.

두 방식이 있다.

#### Implicit preprocess

```text
vkCmdExecuteGeneratedCommandsEXT(
    isPreprocessed = VK_FALSE
)
```

Execution 시 필요한 preprocessing을 함께 수행한다.

#### Explicit preprocess

```text
vkCmdPreprocessGeneratedCommandsEXT
    ↓ explicit synchronization
vkCmdExecuteGeneratedCommandsEXT(
    isPreprocessed = VK_TRUE
)
```

장점:

- preprocess timing을 scheduler가 조절 가능
- async compute와 overlap 가능성
- reusable command data에 유리할 수 있음

비용:

- preprocess buffer
- additional synchronization
- stale state snapshot 관리

중요한 점은 Vulkan spec이 preprocessing을 graphics/compute와 다른 logical pipeline stage로 취급하며, separate preprocess를 사용하면 execute와 명시적으로 synchronize해야 한다는 것이다.

즉 DGC도 “GPU가 command를 만들었으니 barrier가 사라진다”가 아니다.

---

### 4.5 DGC가 Work Graph 자체는 아니다

DGC는 GPU-generated command execution을 강하게 확장하지만, shader node가 실행 중 arbitrary child node를 재귀적으로 emit하는 execution model은 아니다.

Conceptually:

```text
DGC:
GPU prepares command sequences
      ↓
Execute sequences
```

반면 Work Graph는:

```text
Node A executes
  → emits B records
  → emits C records
Node B executes
  → emits D records
```

처럼 **execution 중 work topology가 전개**될 수 있다.

이 차이가 중요하다.

Visibility renderer에서 DGC는 final render packet execution에 매우 좋지만,
instance→meshlet→visibility→material path 전개 자체는 compute queue/work graph로 처리하는 편이 자연스러울 수 있다.

---

### 4.6 Persistent Queue는 application-managed Work Graph다

Native Work Graph API가 없어도 GPU buffer와 atomic counter로 비슷한 구조를 만들 수 있다.

개념적으로:

```text
QueueHeader {
    readHead
    writeTail
    capacity
}

WorkItem {
    type
    payload
    generation
}
```

Persistent compute workers가:

```text
while (work available) {
    pop();
    process();
    emit child work();
}
```

같은 구조를 사용할 수 있다.

하지만 이 단순 pseudo-code에는 큰 함정이 있다.

**GPU forward progress를 CPU thread처럼 가정하면 안 된다.**

---

### 4.7 Persistent kernel의 핵심 위험: forward progress

GPU는 모든 workgroup이 동시에 resident하다는 보장이 없다.

다음 구조를 생각하자.

```text
Producer group:
queue full이 풀릴 때까지 wait

Consumer group:
queue item을 소비해야 함
```

만약 모든 resident slot을 producer가 차지하고 producer가 consumer 실행을 기다린다면 deadlock-like 상황이 생길 수 있다.

즉 GPU persistent queue에서는:

- blocking wait 최소화
- bounded retry
- cooperative scheduling assumption 주의
- kernel boundary를 global synchronization으로 활용
- hardware/native work graph scheduler 활용 가능성

이 중요하다.

CPU lock-free algorithm을 GPU에 그대로 복사하는 것은 위험하다.

---

### 4.8 Queue capacity는 correctness 문제다

Work amplification이 있다면 output 개수는 input보다 클 수 있다.

예:

```text
1 Chunk
  → 64 Meshlets
64 Meshlets
  → 40 Visible Meshlets
40 Visible Meshlets
  → Depth/Main 두 queue로 분기 가능
```

Queue overflow를 단순히 “성능 issue”로 보면 안 된다.

Overflow 시 work item이 사라지면 geometry가 렌더되지 않는다.

즉 queue capacity는 **rendering correctness contract**다.

대응 방식:

- worst-case capacity
- hierarchical capacity budget
- overflow queue
- spill buffer
- multi-pass processing
- prefix-count 후 exact allocation
- bounded expansion

---

### 4.9 Append Queue vs Ring Queue

#### Append-only queue

```text
writeTail++
```

장점:

- 매우 단순
- stream compaction과 유사
- frame-local work에 좋음

단점:

- capacity를 한 번 쓰면 재사용 어려움
- multi-generation work에는 불편

#### Ring buffer

```text
head/tail modulo capacity
```

장점:

- persistent producer/consumer
- memory 재사용

단점:

- wraparound
- ABA-like generation 문제
- full/empty 판정
- contention
- forward progress complexity

Rendering의 frame-bounded pipeline에서는 append-only streams + stage boundary가 오히려 더 단순하고 빠를 수 있다.

즉 persistent queue가 항상 더 advanced한 정답은 아니다.

---

### 4.10 GPU queue는 payload를 작게 유지한다

Work queue에 full object data를 복사하면 bandwidth가 급격히 증가한다.

좋은 work item은 대체로:

```text
LogicalHandle
LocalIndex
Flags
Small classification key
```

정도만 들고,
실제 payload는 persistent table에서 late resolve한다.

예:

```text
MeshletWorkItem {
    MeshHandle mesh;
    uint meshletLocalIndex;
    uint generation;
}
```

이전 노트의 relocation-safe design과 동일하다.

> **Queue는 identity를 운반하고, physical payload는 table에서 resolve한다.**

---

### 4.11 Work type을 하나의 giant union으로 만들지 않는다

초기 구현에서는 다음처럼 만들기 쉽다.

```text
WorkItem {
    type;
    giant union payload;
}
```

하지만 type마다 payload 크기가 다르면 queue element가 가장 큰 type에 맞춰져 bandwidth가 낭비된다.

대안:

```text
ChunkQueue
MeshletQueue
DepthQueue
MainQueue
```

처럼 **typed queues**를 분리할 수 있다.

장점:

- compact layout
- coalesced access
- shader specialization
- type branch 감소

단점:

- queue 수 증가
- cross-queue synchronization 증가

Native Work Graph API가 이런 typed node/record abstraction을 runtime level에서 제공한다는 점이 중요한 이유다.

---

### 4.12 D3D12 Work Graphs의 핵심 아이디어

Direct3D 12 Work Graphs는 node shader가 다른 node를 위한 work records를 생성하고, runtime이 work graph 실행을 관리하는 model이다.

Conceptually:

```text
DispatchGraph
   ↓
Node: InstanceCull
   → output records
Node: MeshletExpand
   → output records
Node: MaterialClassify
   → output records
Node: RenderPreparation
```

Application이 직접 global queue의 head/tail, capacity scheduling, worker lifetime을 모두 구현하는 것보다 higher-level abstraction이다.

중요한 포인트는 Work Graph가 단순 function call graph가 아니라 **GPU work scheduling graph**라는 점이다.

---

### 4.13 Work amplification은 mesh shader의 task→mesh보다 더 일반적이다

Mesh shader pipeline에서도 이미 limited work amplification이 있다.

```text
Task Shader
  → emits Mesh Shader Workgroups
```

그러나 generic work graph는 rendering 외에도 다음을 표현할 수 있다.

```text
Brick Changed
  → Meshing
  → Meshlet Build
  → Bounds
  → Visibility
  → Render Work
```

즉 simulation/geometry/rendering을 하나의 broad GPU work DAG로 생각할 수 있다.

물론 API와 synchronization domain이 다르므로 실제로 하나의 native graph로 만들 수 있다는 뜻은 아니다.

중요한 것은 **system decomposition 방식**이다.

---

### 4.14 Multi-stage amplification에서는 fan-out 통계를 봐야 한다

Work graph는 node count보다 **fan-out distribution**이 중요하다.

예:

```text
average meshlets / chunk
visible ratio
LOD split ratio
material class distribution
occluder/main split ratio
```

이 값들이 queue memory와 occupancy를 결정한다.

평균만 보면 위험하다.

Queue capacity는 tail distribution, 즉 worst-case burst를 봐야 한다.

```text
P50 fanout
P95 fanout
P99 fanout
max burst
```

같은 관점이 필요하다.

---

### 4.15 Backpressure가 필요하다

Consumer보다 producer가 빠르면 queue가 계속 커진다.

CPU systems에서는 blocking/backpressure를 사용하지만 GPU에서는 blocking이 위험할 수 있다.

가능한 전략:

- producer emission cap
- node별 token budget
- multi-wave processing
- overflow list
- queue watermark
- stage break / relaunch

즉 GPU backpressure는 “wait until queue empty”보다 **work production을 제한하거나 stage를 분할하는 방향**이 안전한 경우가 많다.

---

### 4.16 Work-stealing 사고방식은 load imbalance에 유용하지만 비싸다

Meshlet cost는 균일하지 않다.

- triangle count 다름
- material cost 다름
- clipping mode 다름
- primitive culling 비율 다름

고정 chunk assignment는 일부 workgroup이 일찍 끝날 수 있다.

Global queue에서 atomic pop을 사용하면 dynamic load balancing이 가능하다.

하지만:

- atomic contention
- cache locality 감소
- deterministic order 감소

가 생긴다.

즉 persistent queue는 load balance와 locality 사이의 trade-off다.

---

### 4.17 Queue sharding으로 contention을 줄인다

하나의 global queue 대신:

```text
Queue per workgroup cluster
Queue per screen tile
Queue per material class
Queue per spatial brick region
```

처럼 shard할 수 있다.

장점:

- atomic contention 감소
- locality 증가

단점:

- shard imbalance
- cross-shard stealing 필요 가능

Visibility에서는 screen/spatial partition이 이미 존재하므로 자연스러운 shard key가 될 수 있다.

---

### 4.18 Persistent queue와 prefix-sum pipeline은 대체 관계가 아니다

두 방식은 상황에 따라 섞을 수 있다.

#### Prefix/scan

좋은 경우:

- large dense batch
- deterministic output
- exact capacity 필요
- stage boundary가 자연스러움

#### Persistent queue

좋은 경우:

- irregular work
- variable fan-out
- dynamic load balance
- low-latency multi-stage processing

Hybrid:

```text
Persistent coarse queue
  → batch accumulation
  → scan/compact
  → dense render packet stream
```

이런 구조도 가능하다.

---

### 4.19 Global barrier를 줄이되 semantic barrier는 남는다

Work graph의 장점 중 하나는 fixed pass 사이의 global barrier를 줄일 수 있다는 것이다.

하지만 다음 dependency가 사라지는 것은 아니다.

- geometry generation 완료
- bounds version ready
- HZB ready
- relocation table publish
- indirect/DGC input ready

즉:

```text
fewer API/global barriers
≠
no dependency
```

Dependency를 node-level record flow와 runtime scheduler가 더 세밀하게 표현하는 것이다.

---

### 4.20 DGC state lifetime과 work queue lifetime은 다르다

DGC input stream은 command sequence를 의미하고,
work queue는 logical work를 의미한다.

Example:

```text
MeshletWorkItem
  → classification
  → RenderPacket
  → DGC sequence
```

RenderPacket이 만들어지는 순간:

- shader/pipeline variant
- push constants
- index/vertex state
- draw count

같은 execution state가 resolve될 수 있다.

따라서 architecture에서 **logical work resolve boundary**를 정하는 것이 중요하다.

너무 일찍 resolve하면 relocation/material changes에 취약하고,
너무 늦게 resolve하면 rendering hot path의 indirection이 늘어난다.

---

### 4.21 DGC preprocessing과 relocation epoch

DGC input이 physical buffer address, shader state, index/vertex binding을 포함할 수 있으므로 explicit preprocess 이후 관련 state가 바뀌면 stale preprocess가 될 수 있다.

Dynamic mesh pool에서는:

```text
relocationEpoch
commandInputEpoch
preprocessEpoch
executeEpoch
```

관계가 중요하다.

Previous 노트의 relocation-safe rule이 command-generation layer까지 확장된다.

---

### 4.22 Device address는 work identity가 아니다

Persistent queue item에 다음처럼 raw address를 저장하는 것은 빠를 수 있다.

```text
vertexAddress
meshletAddress
```

하지만 defragmentation/relocation이 있는 renderer에서는 stale pointer가 된다.

Work graph에서도 동일한 원칙을 유지한다.

```text
Persistent Work Record
  → logical handle

Execution packet
  → physical address resolve
```

즉 dynamic work scheduling이 도입되어도 identity/placement separation은 그대로 중요하다.

---

### 4.23 Epoch를 queue item에 직접 넣을 수 있다

Dynamic geometry에서는 work item이 queue에서 기다리는 동안 source data가 갱신될 수 있다.

Work record에:

```text
generation
geometryEpoch
visibilityEpoch
```

등을 포함하면 consumer가 stale work를 reject할 수 있다.

이것은 queue가 길어질수록 중요하다.

Low-latency work graph는 performance뿐 아니라 stale work window도 줄인다.

---

### 4.24 GPU queue의 ABA 문제

Ring queue slot이 빠르게 재사용되면 같은 index가 전혀 다른 work를 의미할 수 있다.

그래서 slot state에 generation/ticket을 사용할 수 있다.

```text
slotIndex
slotGeneration
```

이전의 mesh handle generation과 같은 패턴이다.

GPU systems에서 **index reuse와 identity reuse를 구분하는 원칙**은 여러 계층에서 반복된다.

---

### 4.25 Work graph debugger는 node count만 보면 안 된다

필요한 관측값:

- input records / node
- emitted records / node
- fan-out histogram
- queue high-water mark
- queue overflow count
- average queue residency time
- stale-generation reject count
- atomic operations / work item
- memory bytes / work item
- idle worker ratio
- active wave occupancy
- DGC sequence count
- preprocess time
- execute time
- node transition count

특히:

```text
Useful GPU Work
----------------
Queue/Dispatch/Scheduling Overhead
```

를 봐야 한다.

---

### 4.26 “Pass 수 감소”가 목표가 아니다

Work graph로 8개 pass를 1개의 giant persistent kernel로 합쳤다고 하자.

겉으로는 pass 수가 줄었다.

하지만:

- register pressure 증가
- shader specialization 감소
- occupancy 감소
- instruction cache pressure
- queue atomics 증가
- debug 복잡성 증가

로 더 느려질 수 있다.

따라서 목표는 pass count가 아니라:

> **불필요한 global synchronization과 empty work traversal을 줄이는 것**이다.

---

### 4.27 Specialized nodes가 giant kernel보다 나을 수 있다

Work graph의 장점은 여러 specialized node를 유지하면서 GPU-side chaining을 할 수 있다는 것이다.

예:

```text
OpaqueMeshletNode
AlphaTestMeshletNode
DepthOnlyNode
HeatmapNode
```

각 node는:

- 작은 register footprint
- coherent control flow
- specialized memory access

를 가질 수 있다.

Dynamic classification이 strong한 renderer일수록 이 구조의 가치가 커진다.

---

### 4.28 Visibility pipeline에 적용한 concrete graph

사용자의 현재 학습 흐름을 하나의 graph로 묶으면:

```text
Node 0: Dirty/Active Chunk Input
          ↓
Node 1: Chunk Frustum + Generation Check
          ↓ visible chunks
Node 2: Meshlet Expansion
          ↓ candidate meshlets
Node 3: Frustum/Cone/Temporal Hi-Z
          ↓ visible candidates
Node 4: Occluder Score
          ├→ Depth Work
          └→ Deferred Candidate Work
Node 5: Partial HZB-dependent Re-Cull
          ↓ final visible
Node 6: Depth / Material Classification
          ↓ render packets
Node 7: DGC / Indirect Execution
```

이 graph에서 모든 node를 반드시 native Work Graph API로 구현해야 하는 것은 아니다.

중요한 것은 architecture reasoning이다.

- 어떤 edge가 logical record flow인가?
- 어디서 global snapshot barrier가 필요한가?
- 어디까지 persistent queue로 묶을 것인가?
- 어디부터 indirect/DGC execution packet으로 resolve할 것인가?

---

### 4.29 C++에서는 WorkType과 ResourceEpoch를 strong type으로 둔다

Host-side graph description이 커질수록 다음 값들이 혼동되기 쉽다.

```text
QueueIndex
NodeID
MeshHandle
Generation
GeometryEpoch
CommandEpoch
```

C++에서 strong type을 쓰면 work graph의 logical identity와 GPU physical offset을 분리하기 쉽다.

특히 debug build에서는:

```text
WorkRecordHeader {
    WorkType
    Generation
    SourceNode
    Epoch
}
```

같은 header가 GPU crash/debug capture에 큰 도움이 된다.

---

### 4.30 Vulkan DGC와 D3D12 Work Graphs를 경쟁 기능으로만 보지 않는다

둘은 abstraction level이 다르다.

| 관점 | Vulkan DGC | D3D12 Work Graphs |
|---|---|---|
| 중심 abstraction | Generated command sequence | Shader work node graph |
| GPU가 생성하는 것 | Draw/dispatch/state command inputs | 다음 node의 work records |
| 좋은 사용처 | GPU-side rendering/dispatch command stream | Irregular multi-stage GPU algorithms |
| state change | Execution set/tokens로 표현 | Node shader/resource model 중심 |
| queue scheduling | Application + driver preprocess/execute | Work graph runtime가 더 많이 담당 |

Renderer는 conceptual하게:

```text
Work Graph-like compute
  → Render Packets
  → Vulkan DGC
```

형태로도 설계할 수 있다.

---

## 5. 내 관심 분야와 연결

### 5.1 Semiconductor / CFD pipeline

Dynamic simulation geometry는 work graph 사고방식과 매우 잘 맞는다.

```text
Changed SDF Brick
    ↓
Surface Extraction
    ↓
Meshlet Build
    ↓
Bounds / QEF Metadata
    ↓
Visibility
    ↓
Render Classification
```

모든 brick이 같은 양의 geometry를 만들지 않는다.

- empty brick → 0 meshlets
- flat surface brick → few meshlets
- high-curvature region → many meshlets
- topology event → rebuild work 증가

즉 **variable fan-out**이 기본이다.

### 5.2 NVIDIA Warp / CUDA compute와 연결

Warp/CUDA에서 field processing을 하고 Vulkan에서 rendering한다면 native cross-API work graph 하나로 합치기는 어렵더라도 logical graph는 유지할 수 있다.

```text
CUDA / Warp
  → Dirty Brick Queue
  → Meshing Queue
  → Published Mesh Snapshot

Vulkan Compute
  → Visibility Work Graph
  → Render Packet Stream

Vulkan Graphics
  → DGC / Indirect Execution
```

이렇게 보면 external semaphore는 graph edge의 **execution dependency**이고,
queue/epoch metadata는 **semantic dependency**다.

### 5.3 Sparse field와 persistent queue

Sparse simulation은 active cell 수가 frame마다 바뀐다.

Dense dispatch:

```text
dispatch entire volume
```

보다 active brick queue를 사용할 수 있다.

이 사고방식은 rendering visibility queue와 동일하다.

그래서 compute/simulation engineer가 graphics work graph를 이해하면 sparse solver와 renderer scheduling을 공통 언어로 볼 수 있다.

### 5.4 Game engine 커리어 관점

Game engine에서 중요한 문제는:

- destruction
- procedural geometry
- particles
- virtualized geometry
- meshlet culling
- GPU scene update
- ray tracing work generation

처럼 irregular work가 많다는 것이다.

Work graph/persistent queue/DGC를 이해하면 단순 shader coding보다 **GPU execution architecture**를 설명할 수 있다.

Nintendo/Unity류 engine role에서도 강한 질문은:

> “GPU에서 work를 생성하면 왜 CPU overhead는 줄지만 queue overflow, lifetime, forward progress라는 새로운 문제가 생기는가?”

에 답할 수 있는가이다.

---

## 6. 머릿속에 남길 질문 3개

1. **Persistent GPU queue가 fixed pass + prefix-sum pipeline보다 유리해지는 경계는 variable fan-out, queue contention, global barrier cost 관점에서 어떻게 판단할 수 있을까?**
2. **Vulkan DGC는 GPU-generated rendering command를 강화하지만 왜 generic Work Graph와 동일한 abstraction이 아니며, visibility pipeline에서 두 모델을 어디에서 연결하는 것이 자연스러울까?**
3. **Dynamic SDF meshing에서 work item이 queue에 오래 머무는 동안 source geometry epoch가 바뀔 수 있다면 stale work를 어떤 generation/epoch contract로 검출해야 할까?**

---

## 7. graphics engineer 면접 질문 1개와 답변

### 질문

**“GPU-driven renderer의 여러 compute pass를 하나의 persistent kernel과 global work queue로 합치면 barrier와 dispatch overhead가 줄어드니 항상 더 빠른 것 아닌가요?”**

### 답변

항상 그렇지는 않다.

Persistent kernel은 irregular multi-stage work에서 장점이 있다.

- CPU dispatch 감소
- stage 간 global buffer scan 감소 가능
- dynamic load balance
- producer가 바로 child work를 emit 가능

하지만 비용도 크다.

1. **Forward progress 위험**  
   GPU workgroup은 CPU thread처럼 모두 동시에 실행된다는 보장이 없다. Blocking producer/consumer 구조는 resident workgroup 배치에 따라 deadlock-like 상황을 만들 수 있다.

2. **Queue contention**  
   Global head/tail atomic이 hotspot이 될 수 있다.

3. **Capacity / overflow**  
   Work amplification이 발생하면 queue overflow가 rendering correctness failure로 이어질 수 있다.

4. **Register / instruction footprint**  
   여러 stage를 giant kernel로 합치면 occupancy와 instruction cache 효율이 떨어질 수 있다.

5. **Specialization loss**  
   서로 다른 material/geometry path가 하나의 branch-heavy kernel에 들어가면 divergence가 커질 수 있다.

6. **Memory locality**  
   Dynamic work stealing이 load balance는 개선하지만 spatial/material locality를 깨뜨릴 수 있다.

그래서 practical renderer에서는 다음처럼 hybrid가 좋은 경우가 많다.

```text
Irregular coarse stages
  → persistent / work-graph style queue

Dense batch stage
  → scan/compaction

Final render packets
  → indirect draw or Vulkan DGC
```

핵심은 pass 수를 줄이는 것이 아니라:

> **global synchronization과 empty work를 줄이면서도 occupancy, locality, queue safety를 유지하는 것**

이다.

---

## 8. 포트폴리오 / 커리어 연결

이 주제는 graphics engineer가 **GPU algorithm을 넘어 execution architecture까지 이해한다는 것**을 보여주기 좋다.

### GPU Compute

- persistent kernel
- append/ring queue
- atomic producer/consumer
- work amplification
- backpressure
- subgroup aggregation
- load balancing

### Rendering

- GPU-driven visibility
- meshlet culling
- render packet generation
- indirect count
- front-to-back classification

### Vulkan

- `VK_EXT_device_generated_commands`
- `VkIndirectCommandsLayoutEXT`
- `VkIndirectExecutionSetEXT`
- `vkCmdPreprocessGeneratedCommandsEXT`
- `vkCmdExecuteGeneratedCommandsEXT`
- preprocess synchronization
- Buffer Device Address

### DirectX 12

- Work Graphs
- `DispatchGraph`
- node records
- GPU-side work scheduling

### Memory Layout

- typed queues
- ID-only work records
- hot/cold payload
- queue sharding
- generation ticket
- queue high-water mark

### Dynamic Simulation

- dirty SDF brick
- adaptive meshing fan-out
- geometry epoch
- CUDA/Warp producer → Vulkan consumer

면접에서 다음처럼 설명할 수 있으면 강하다.

> **“Flat indirect rendering은 homogeneous render stream에는 매우 효율적이지만, visibility/LOD/material 분기가 많아지면 GPU-side logical work를 여러 단계로 생성해야 합니다. 이때 persistent queues나 Work Graph 모델을 사용할 수 있지만 queue capacity와 forward progress가 correctness 문제가 됩니다. Final render packet 단계에서는 Vulkan DGC를 사용해 shader/state/draw sequence를 device-side에서 실행할 수 있고, dynamic mesh relocation이 있다면 preprocess와 execute가 같은 relocation epoch를 보도록 해야 합니다.”**

이 설명은 C++, GPU compute, rendering pipeline, synchronization, memory lifetime을 한 번에 연결한다.

---

## 9. 내일 이어서 볼 개념

**GPU Queue Backpressure and Forward Progress: Ring Buffers, Generation Tickets, and Deadlock-Free Persistent Scheduling**

오늘은 work graph의 전체 architecture를 봤다.

다음은 그중 가장 systems-level이고 위험한 부분을 깊게 본다.

```text
GPU Work Graph
    ↓
Persistent Queue
    ↓
Queue Saturation
    ↓
Backpressure / Spill
    ↓
Forward Progress
```

다음 노트에서는:

- ring buffer sequence/ticket
- MPMC queue의 GPU 특성
- atomic contention
- queue overflow semantics
- non-blocking producer policy
- occupancy와 resident workgroup
- persistent kernel forward progress
- stage break / relaunch
- spill queue
- high-water mark telemetry
- generation/ABA 문제

를 중심으로 이어간다.

---

## 10. 참고 키워드

- GPU Work Graph
- Persistent Kernel
- Persistent Threads
- Persistent Work Queue
- Producer / Consumer Queue
- Work Amplification
- Dynamic Fan-Out
- Backpressure
- Forward Progress
- GPU Deadlock
- Ring Buffer
- Append Queue
- Queue Sharding
- Atomic Contention
- Work Stealing
- Stream Compaction
- Prefix Sum / Scan
- Indirect Dispatch
- Indirect Draw Count
- Device-Generated Commands
- `VK_EXT_device_generated_commands`
- `VkIndirectCommandsLayoutEXT`
- `VkIndirectExecutionSetEXT`
- `VkGeneratedCommandsInfoEXT`
- `vkCmdPreprocessGeneratedCommandsEXT`
- `vkCmdExecuteGeneratedCommandsEXT`
- `VK_PIPELINE_STAGE_COMMAND_PREPROCESS_BIT_EXT`
- D3D12 Work Graphs
- `DispatchGraph`
- Mesh Shader / Task Shader
- Render Packet
- Logical Handle
- Generation Ticket
- Geometry Epoch
- Relocation Epoch
- CUDA-Vulkan Interop
- Vulkan Documentation Project — **VK_EXT_device_generated_commands**
- Vulkan Specification — **Device-Generated Commands**
- Vulkan Extension Proposal — **VK_EXT_device_generated_commands**
- Microsoft DirectX Developer Blog — **D3D12 Work Graphs**
- Khronos Vulkan Documentation Project — **GPU-Side Command Generation**
