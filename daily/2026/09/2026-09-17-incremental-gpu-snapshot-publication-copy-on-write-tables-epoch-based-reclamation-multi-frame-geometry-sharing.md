---
title: "Incremental GPU Snapshot Publication: Copy-on-Write Tables, Epoch-Based Reclamation, and Multi-Frame Geometry Sharing"
date: "2026-09-17"
category: Graphics
tags: [GPU, Rendering, GPU-Driven Rendering, Snapshot, Copy-on-Write, Epoch Reclamation, Deferred Destruction, Timeline Semaphore, CUDA, Vulkan, Meshlet, Dynamic Geometry, Memory Layout, C++]
level: intermediate
---

# [Daily Graphics Growth] 2026-09-17 - Incremental GPU Snapshot Publication: Copy-on-Write Tables, Epoch-Based Reclamation, and Multi-Frame Geometry Sharing

## 1. 오늘의 개념

어제는 **GPU Termination Detection and Quiescence**에서 distributed GPU runtime이 정말 종료되었는지 확인하고, 그 뒤 geometry snapshot을 CUDA에서 publish해 Vulkan이 소비하는 lifecycle을 봤다.

오늘은 그 snapshot을 **매 frame 통째로 복사하지 않고** 유지하는 방법으로 이어간다.

Dynamic SDF / CFD geometry에서 매 frame 바뀌는 영역은 전체 geometry의 일부일 가능성이 높다.

예를 들어:

```text
Frame N geometry
- substrate chunk A: unchanged
- oxide chunk B: unchanged
- trench chunk C: changed
- gate chunk D: unchanged
- spacer chunk E: changed
```

가 있다면 다음 frame에 A/B/D까지 다시 복사하는 것은 낭비다.

원하는 구조는:

```text
Persistent Payload Pool
          ↑
Snapshot Table N
          ↑
     unchanged entries
          ↓ shared
Snapshot Table N+1
          ↓
 changed entries only → new payload
```

즉 snapshot 자체는 immutable하게 유지하면서, 변경된 mapping만 새 version에 반영하는 **Copy-on-Write(COW) table**을 만든다.

하지만 여기서 두 번째 문제가 생긴다.

Snapshot N+1이 새로운 chunk를 가리킨다고 해서 Snapshot N이 가리키던 old chunk를 바로 free할 수는 없다.

Vulkan은 여전히:

```text
Frame N
Frame N-1
```

같은 old snapshot을 in-flight로 읽고 있을 수 있기 때문이다.

따라서 필요한 것이 **Epoch-Based Reclamation(EBR) / fence-based retirement** 사고방식이다.

오늘은 다음 네 가지를 연결한다.

1. **Immutable Snapshot Table**
   - renderer가 읽는 mapping은 frame 동안 변하지 않는다.

2. **Copy-on-Write Table Update**
   - changed chunk만 새 mapping/payload로 교체한다.

3. **Multi-Frame Payload Sharing**
   - unchanged geometry는 여러 snapshot에서 동일 allocation을 공유한다.

4. **Epoch-Based Reclamation**
   - old snapshot을 읽는 모든 consumer가 끝난 뒤에만 old allocation을 재사용한다.

핵심 질문은 다음이다.

> **Dynamic geometry를 매 frame 전체 double-buffer하지 않으면서도 CUDA producer와 여러 in-flight Vulkan frame이 같은 payload pool을 안전하게 공유하려면, snapshot version과 allocation lifetime을 어떻게 분리해야 하는가?**

---

## 2. 한 줄 핵심

> Incremental GPU snapshot의 핵심은 **payload 자체보다 “logical handle → physical allocation” mapping을 immutable epoch로 versioning하고, changed entry만 Copy-on-Write하며, allocation은 그 allocation을 참조하는 가장 늦은 in-flight epoch가 끝날 때까지 reclaim하지 않는 것**이다.

---

## 3. 왜 중요한가

Dynamic geometry에서 가장 단순한 안전한 설계는 전체 double buffering이다.

```text
Buffer A → render
Buffer B → build
swap
```

작은 scene에서는 매우 좋은 방법이다.

하지만 geometry가 커지면 문제가 된다.

예:

```text
Vertex / Index / Meshlet payload: 4 GB
Double buffer:                  8 GB
Triple buffer:                12 GB
```

Sparse SDF/CFD처럼 전체 geometry의 5~10%만 변경되어도 전체 payload를 여러 벌 유지하면 memory amplification이 크다.

또 copy bandwidth도 커진다.

```text
4 GB snapshot copy × 60 FPS
= 240 GB/s logical copy traffic
```

실제 GPU memory bandwidth가 높더라도 이 traffic은 rendering, simulation, mesh generation과 경쟁한다.

따라서 목표는:

```text
Snapshot consistency
without
Full snapshot duplication
```

이다.

### 3.1 Payload와 Mapping을 분리하면 문제의 크기가 바뀐다

Renderer가 실제로 필요한 것은:

```text
LogicalChunkID
→ current physical data
```

이다.

Payload가 변경되지 않았다면 그대로 재사용할 수 있다.

변경된 것은 mapping entry뿐이다.

예:

```text
Snapshot N:
Chunk 42 → Allocation A

Snapshot N+1:
Chunk 42 → Allocation B
```

Chunk 17은 unchanged:

```text
Snapshot N:
Chunk 17 → Allocation X

Snapshot N+1:
Chunk 17 → Allocation X
```

즉 immutable snapshot을 만들기 위해 반드시 **payload 전체를 복사할 필요는 없다.**

### 3.2 Snapshot lifetime과 allocation lifetime은 다르다

Snapshot N table을 retire해도 그 table이 가리키는 Allocation X는 Snapshot N+1도 사용할 수 있다.

반대로 Snapshot N+1에서 Chunk 42가 Allocation B로 바뀌었어도 Allocation A는 Snapshot N renderer가 끝날 때까지 유지해야 한다.

따라서 다음 lifetime은 서로 다르다.

```text
Snapshot Lifetime
Allocation Lifetime
Logical Object Lifetime
```

이 세 개를 하나로 묶으면 unnecessary copy 또는 use-after-free가 생긴다.

### 3.3 Timeline semaphore는 reclamation epoch와 잘 맞는다

현재 Vulkan specification에서 timeline semaphore는 **strictly increasing 64-bit payload**를 가진다.

CUDA의 Vulkan interoperability도 timeline semaphore를 외부 synchronization object로 import하고, 특정 value를 signal/wait하는 흐름을 지원한다.

따라서:

```text
Snapshot Epoch 100
→ Vulkan consumes
→ RenderComplete timeline = 100
```

처럼 GPU consumer progress를 monotonic epoch로 표현하기 좋다.

중요한 점은 semaphore value 그 자체가 memory reclamation algorithm은 아니라는 것이다.

> **Timeline value는 “어디까지 consumer가 끝났는가”를 알려주는 progress oracle이고, 어떤 allocation을 free할지는 allocator/reclamation policy가 결정한다.**

---

## 4. 구현 관점

### 4.1 가장 먼저 세 lifetime을 분리한다

#### Logical Object Lifetime

예:

```text
MeshHandle { index, generation }
```

Object가 생성/삭제되는 lifetime이다.

#### Snapshot Mapping Lifetime

특정 epoch에서:

```text
MeshHandle
→ AllocationRecord
```

mapping이 어떤 상태였는지 나타낸다.

#### Allocation Lifetime

실제:

```text
vertex range
index range
meshlet range
bounds
```

가 존재하는 physical storage lifetime이다.

세 lifetime을 분리하면 다음 상태가 자연스럽다.

```text
same logical object
different snapshot mapping
old + new allocation coexist
```

### 4.2 Snapshot Table은 immutable read model로 본다

Renderer가 frame 중에 읽는 table entry가 바뀌면 shader invocation마다 서로 다른 mapping을 볼 가능성이 생긴다.

그래서 renderer-facing table은 다음 contract를 가지는 편이 좋다.

```text
Snapshot E
= immutable for its consumer lifetime
```

Producer는 Snapshot E를 수정하지 않는다.

대신:

```text
Snapshot E+1
```

을 만든다.

이것은 CPU의 RCU(Read-Copy-Update)와 비슷한 mental model이다.

```text
Reader:
immutable old version

Writer:
new version build

Publish:
pointer/table epoch switch

Reclaim:
old readers finish later
```

### 4.3 Copy-on-Write의 대상은 보통 metadata다

“Copy-on-Write geometry”라고 하면 vertex buffer 전체를 copy하는 것으로 오해하기 쉽다.

실제로 COW의 핵심 대상은 작은 mapping entry다.

예:

```text
ChunkRecord {
    vertexAllocation
    indexAllocation
    meshletAllocation
    boundsIndex
    generation
}
```

Changed chunk만:

```text
old record
→ clone/update
→ new snapshot entry
```

한다.

Unchanged chunk는 같은 allocation record를 공유한다.

### 4.4 Flat Table Copy도 생각보다 쌀 수 있다

예를 들어 chunk가 100,000개이고 entry가 32 bytes라면:

```text
3.2 MB
```

다.

Payload가 수 GB인 것에 비하면 table 전체를 frame마다 복사하는 것이 충분히 쌀 수 있다.

따라서 COW를 너무 일찍 복잡한 tree/page 구조로 만들 필요는 없다.

첫 설계 후보:

```text
copy full small table
patch changed entries
```

장점:

- indexing 단순
- shader lookup 1회
- contiguous memory
- debug 쉬움

COW의 목표는 “모든 copy 제거”가 아니라 **큰 payload duplication을 제거하는 것**이다.

### 4.5 Table까지 커지면 Page-Level COW

Table 자체가 매우 크다면 page 단위 COW를 사용할 수 있다.

예:

```text
Root Table
  ↓
Page 0
Page 1
Page 2
...
```

Snapshot E+1은 unchanged page를 공유하고 changed page만 새로 만든다.

```text
Snapshot E:
Root → P0 P1 P2

Snapshot E+1:
Root → P0 P1' P2
```

장점:

- update bandwidth 감소
- large table에 scalable

단점:

- shader indirection 증가
- page-table fetch
- memory fragmentation
- allocator metadata 증가

따라서 table 크기와 update ratio를 profiler로 보고 선택해야 한다.

### 4.6 Chunk-Level Indirection이 좋은 경계다

Meshlet 하나마다 COW allocation을 만들면 metadata가 너무 많아질 수 있다.

반대로 entire scene을 한 allocation으로 유지하면 작은 update가 큰 relocation을 유발한다.

좋은 granularity 후보:

```text
brick
mesh chunk
surface patch
meshlet page
```

즉 update locality와 allocation overhead 사이의 중간점이다.

사용자의 sparse SDF pipeline에서는 brick/chunk가 이미 natural update boundary이므로 snapshot COW boundary로 재사용하기 좋다.

### 4.7 Changed Set을 먼저 만든다

Incremental publication에는:

```text
ChangedChunkIDs[]
```

가 핵심 input이 된다.

이 list는 다음에서 올 수 있다.

- dirty SDF brick bitset
- meshing result
- topology repair result
- material change
- LOD rebuild

Snapshot build:

```text
Base Snapshot E
    ↓ copy/reference
Patch Changed IDs
    ↓
Snapshot E+1
```

이렇게 하면 geometry algorithm의 dirty tracking과 renderer snapshot update가 직접 연결된다.

### 4.8 Dirty Bit와 Changed Mapping은 같은 것이 아니다

Brick이 dirty라고 해서 physical allocation이 반드시 바뀌는 것은 아니다.

예:

```text
old capacity >= new payload
```

라면 같은 allocation 안에서 overwrite할 수도 있다.

하지만 in-flight old snapshot이 그 allocation을 읽고 있으면 in-place overwrite는 unsafe하다.

따라서 changed chunk마다 다음 질문이 필요하다.

```text
Can update in place?
```

조건:

- old allocation을 어떤 consumer도 읽지 않음
- snapshot visibility contract가 깨지지 않음
- capacity 충분

보통 multi-frame in-flight renderer에서는 **in-place update보다 new allocation + mapping switch**가 reasoning하기 쉽다.

### 4.9 In-Place Update가 가능한 Safe Window

특정 allocation이:

```text
latest snapshot only
AND
no Vulkan consumer currently references it
```

라면 in-place update가 가능할 수 있다.

하지만 이 optimization을 위해 reference/lifetime tracking이 복잡해진다면 new allocation이 더 나을 수 있다.

Graphics systems에서는:

> **memory bandwidth 절약보다 lifetime reasoning 단순성이 더 큰 가치가 될 때가 많다.**

### 4.10 COW Update State Machine

Changed chunk 하나를 다음처럼 본다.

```text
OLD_ACTIVE
  ↓ allocate
NEW_BUILDING
  ↓ geometry written
NEW_FINALIZED
  ↓ snapshot E+1 references new
NEW_PUBLISHED
  ↓ old readers eventually finish
OLD_RECLAIMABLE
```

중요한 것은 new allocation을 만들었다고 old allocation이 바로 reclaimable하지 않다는 점이다.

### 4.11 Reclamation의 질문은 “누가 아직 읽을 수 있는가?”

Allocation A를 free하기 위한 조건:

```text
No in-flight snapshot can resolve to A
```

이다.

Reference count 방식이면:

```text
refCount(A) == 0
```

으로 표현할 수 있다.

Epoch 방식이면:

```text
lastPossibleReaderEpoch(A)
<
completedConsumerEpoch
```

으로 표현할 수 있다.

둘은 같은 문제를 다른 비용 구조로 푼다.

### 4.12 Reference Counting

각 allocation이 snapshot reference count를 가진다.

```text
Snapshot E references A
→ ref++

Snapshot E retired
→ ref--

ref == 0
→ reclaim
```

장점:

- 정확한 lifetime
- sharing pattern이 irregular해도 동작

단점:

- snapshot build/retire마다 atomic updates
- 많은 unchanged chunk에 ref count traffic
- GPU-side global contention 가능
- cycles는 geometry allocation에서는 보통 없지만 bookkeeping은 큼

Immutable snapshot table이 수십만 entry라면 per-entry refcount는 비쌀 수 있다.

### 4.13 Epoch-Based Reclamation

Epoch 방식은 allocation마다 exact reader count를 세지 않는다.

Allocation이 old mapping에서 제거될 때:

```text
retireEpoch = currentPublishEpoch
```

를 기록한다.

Consumer progress가:

```text
completedEpoch >= retireEpoch
```

보다 충분히 앞섰을 때 reclaim한다.

더 정확히는 allocation이 참조될 수 있는 **마지막 snapshot epoch**가 모두 끝났는지를 확인한다.

장점:

- common read path에 ref count 없음
- batch reclamation
- timeline semaphore와 자연스럽게 연결

단점:

- 느린 consumer가 reclamation을 지연
- 정확한 per-object lifetime보다 보수적
- 여러 consumer timeline이 있으면 min-progress 계산 필요

### 4.14 Fence-Based Reclamation은 Graphics EBR의 실용적 형태다

Graphics engine에서는 consumer가 이미 fence/timeline progress를 갖고 있다.

예:

```text
Frame E submitted
→ timeline value E

Frame E completed
→ timeline >= E
```

Allocation을 retire list에 넣을 때:

```text
safeAfterValue = lastRenderValueThatMayReferenceIt
```

를 저장한다.

Reclaimer:

```text
if completedTimeline >= safeAfterValue:
    free allocation
```

이 방식은 generic EBR보다 graphics pipeline에 직접 맞는다.

### 4.15 Timeline Value와 Geometry Epoch를 같은 숫자로 쓸 수도 있다

예:

```text
GeometryEpoch E
VulkanConsumeValue E
VulkanReleaseValue E
```

장점:

- debug 쉬움
- mapping 명확

하지만 반드시 같아야 하는 것은 아니다.

한 geometry snapshot을 여러 render frame이 소비할 수 있다면:

```text
GeometryEpoch 100
→ RenderFrame 400
→ RenderFrame 401
```

같은 관계가 가능하다.

따라서 semantic type은 분리하고 필요할 때 mapping한다.

### 4.16 Multiple In-Flight Frames

Triple-buffered rendering이라고 해도 geometry snapshot이 정확히 3개만 필요하다는 보장은 없다.

예:

```text
Frame 100 uses Geometry E
Frame 101 uses Geometry E
Frame 102 uses Geometry E+1
Frame 103 uses Geometry E+2
```

그리고 GPU queue가 밀리면 E가 오래 살아 있을 수 있다.

그래서:

```text
snapshot count = swapchain image count
```

처럼 고정 lifetime을 가정하기보다 **actual GPU completion timeline**을 기준으로 reclaim하는 것이 안전하다.

### 4.17 “Triple Buffer면 안전하다”는 일반 법칙이 아니다

CPU/GPU latency, async compute, multiple queues, capture/debug, long-running pass 때문에 resource lifetime이 fixed N frames를 넘을 수 있다.

`frameIndex - 3`만 보고 free하는 방식은 위험하다.

더 안전한 조건:

```text
GPU has signaled completion of every submission that can reference allocation
```

이다.

### 4.18 Retire List

Old allocation을 바로 allocator free list에 넣지 않고 retire queue에 넣는다.

```text
RetiredAllocation {
    allocation
    safeAfterTimeline
    bytes
    debugOwner
}
```

Periodic reclamation:

```text
completed = query/waited timeline progress

for retired:
    if retired.safeAfter <= completed:
        allocator.free()
```

GPU-side/CPU-side 어느 쪽에서 관리할지는 architecture에 따라 다르다.

### 4.19 Batch Reclamation

Allocation마다 즉시 free 처리하면 allocator contention이 생길 수 있다.

Timeline progress가 advance할 때:

```text
retire bucket per epoch
```

을 한 번에 reclaim할 수 있다.

예:

```text
Bucket 100
Bucket 101
Bucket 102
```

`completed = 101`이면 100/101 bucket을 batch free한다.

이것은 allocator lock/atomic traffic을 줄이고 debugging도 쉽게 한다.

### 4.20 Memory High-Water 때문에 Reclamation이 중요하다

COW는 write contention을 줄이지만 순간적으로 old + new allocation이 같이 존재한다.

Update burst가 크면:

```text
live bytes
+
retired-but-not-safe bytes
+
new building bytes
```

가 동시에 증가한다.

따라서 memory budget에서:

```text
Allocated
RetiredPending
Reclaimable
```

을 분리해야 한다.

“free하지 못한 old geometry”가 memory leak처럼 보일 수 있지만 실제로는 정상적인 in-flight lifetime일 수 있다.

### 4.21 Retired Bytes는 중요한 Profiler Metric이다

다음 값이 계속 커진다면:

```text
retiredPendingBytes
```

원인은:

- Vulkan consumer가 느림
- timeline release가 늦음
- snapshot이 너무 자주 생성됨
- changed chunk가 너무 큼
- reclamation polling이 느림

일 수 있다.

Allocator fragmentation만 보기 전에 **deferred lifetime pressure**를 봐야 한다.

### 4.22 Snapshot Table의 Hash/Generation Debug

각 snapshot entry에 debug용:

```text
generation
allocationID
payloadVersion
```

을 기록할 수 있다.

GPU crash/corruption에서:

```text
renderer epoch
logical chunk
physical allocation
retire state
```

를 역추적하기 쉬워진다.

Release build에서는 compact packing을 사용하더라도 debug build에서는 provenance가 매우 가치 있다.

### 4.23 Handle Generation은 COW와 별개다

Logical object가 같은데 allocation만 바뀌는 경우:

```text
handle generation = same
snapshot mapping = changed
```

이다.

Object가 삭제/recreated되면:

```text
handle generation = changed
```

따라서 다음 세 값을 혼동하지 않는다.

```text
ObjectGeneration
SnapshotEpoch
AllocationVersion
```

### 4.24 Allocation Version이 필요한 경우

동일 pool slot이 retire 후 재사용될 수 있다.

Debug/runtime validation에서:

```text
AllocationHandle {
    index
    generation
}
```

을 쓰면 stale physical reference를 잡기 쉽다.

이는 mesh handle generation과 다른 domain이다.

### 4.25 Bounds/Visibility Metadata도 Snapshot에 포함해야 한다

Vertex/index만 COW하고 bounds를 in-place update하면 renderer가:

```text
old mesh
+ new bound
```

또는:

```text
new mesh
+ old bound
```

를 볼 수 있다.

Occlusion culling correctness에 영향을 줄 수 있다.

따라서 snapshot version에는 최소한:

- geometry payload mapping
- meshlet range
- bounds
- material/state needed for interpretation

이 같은 semantic version으로 묶여야 한다.

### 4.26 Doping/Scalar Field Overlay는 Separate Epoch가 가능하다

Geometry와 visualization scalar가 서로 다른 update rate를 가질 수 있다.

예:

```text
geometryEpoch = 120
dopingEpoch   = 340
```

Renderer가 둘을 독립적으로 조합할 수 있다면 separate snapshot domain이 유리하다.

하지만 scalar coordinate mapping이 geometry topology에 의존하면 compatible epoch relation이 필요하다.

즉 모든 데이터를 하나의 global epoch로 묶는 것도 과도한 synchronization이 될 수 있다.

### 4.27 Snapshot Domain을 잘 나누면 Parallelism이 늘어난다

가능한 domain:

```text
Geometry Snapshot
Material Snapshot
Scalar-Field Snapshot
Visibility History
```

서로 독립적으로 update 가능한 state는 separate epoch로 유지할 수 있다.

그러나 shader가 함께 해석할 때 compatibility contract를 가져야 한다.

Architecture 목표는:

```text
minimum synchronization domain
that preserves semantic consistency
```

다.

### 4.28 Copy-on-Write와 Relocation의 관계

COW update에서 new allocation을 만들면 자연스럽게 relocation이 발생한다.

```text
Chunk 42:
Allocation A
→ Allocation B
```

이전의 relocation-safe renderer 원칙을 그대로 적용한다.

Persistent user:

```text
logical ChunkHandle
```

Snapshot table:

```text
ChunkHandle → Allocation B
```

Old snapshot:

```text
ChunkHandle → Allocation A
```

즉 COW snapshot은 relocation을 **versioned mapping**으로 안전하게 만드는 방법이기도 하다.

### 4.29 Defragmentation도 Snapshot Update로 표현할 수 있다

Geometry content가 바뀌지 않아도 allocator compaction으로 physical placement가 바뀔 수 있다.

```text
same logical geometry
same payload content
different allocation
```

이것도:

```text
Snapshot E   → A
Snapshot E+1 → B
```

로 처리할 수 있다.

따라서 simulation update와 allocator defrag가 같은 publication mechanism을 공유할 수 있다.

### 4.30 Memory Copy Completion과 Snapshot Publish

Defrag/COW copy가 destination allocation으로 끝났다고 mapping을 즉시 publish하면 안 된다.

순서:

```text
allocate B
→ copy/write payload
→ copy completion visible
→ finalize metadata
→ snapshot mapping points to B
→ publish snapshot
```

Snapshot table이 new allocation을 가리키는 순간 consumer는 B가 완전히 valid하다고 기대한다.

### 4.31 CUDA Stream Ordering과 Snapshot Build

CUDA 내에서는 같은 stream의 ordered operations를 이용해:

```text
mesh generation
→ table patch
→ finalize
→ external semaphore signal
```

의 순서를 표현할 수 있다.

Multiple streams를 사용한다면 event/semaphore dependency로 all producer streams가 finalization 전에 join되어야 한다.

즉 snapshot publication stream이 실제 producer completion을 dominate해야 한다.

### 4.32 Vulkan Queue Wait와 Resource Visibility

Vulkan timeline semaphore wait는 queue operation ordering을 표현한다.

CUDA external semaphore signal과 결합하면:

```text
CUDA writes
→ signal E
→ Vulkan waits E
→ rendering
```

흐름을 만든다.

현재 CUDA 문서는 imported timeline semaphore에 특정 value를 signal/wait하는 interop을 지원하고, Vulkan specification은 timeline semaphore가 monotonically increasing 64-bit payload를 가진다고 정의한다.

### 4.33 Vulkan Consumer Release

Rendering이 끝난 뒤 allocator가 old resource를 reuse하려면 consumer completion을 관측해야 한다.

예:

```text
Vulkan signal renderDone = R
```

Retire record:

```text
safeAfterRenderValue = R
```

이렇게 producer publish timeline과 consumer release timeline을 분리할 수 있다.

한 timeline에 encode할 수도 있지만 semantic clarity가 더 중요하다.

### 4.34 Multiple Vulkan Queues

Geometry snapshot을:

- graphics queue
- async compute culling queue
- transfer queue

여러 consumer가 읽을 수 있다면 reclaim 조건은 하나의 graphics timeline만으로 충분하지 않을 수 있다.

필요한 조건:

```text
all consumer queues that may reference allocation have passed safe point
```

대안:

- shared global completion timeline
- per-queue retire value + min condition
- ownership graph에서 final consumer가 aggregate signal

Renderer architecture에 맞춰 선택한다.

### 4.35 Min-Completed Epoch

여러 reader가 있다면 generic EBR처럼:

```text
safeEpoch = min(completedReaderEpochs)
```

이후의 allocation만 유지한다.

하지만 reader timeline이 서로 다른 semantic sequence를 가진다면 단순 numeric min은 의미가 없을 수 있다.

공통 geometry epoch 기준 acknowledgement를 만들어야 한다.

### 4.36 Hazard-Pointer식 정밀 추적은 언제 필요한가

Epoch reclamation은 느린 reader 하나가 많은 old allocation reclamation을 지연시킬 수 있다.

매우 long-lived reader가 존재하면 per-allocation/per-reader reference tracking이 더 memory-efficient할 수 있다.

CPU concurrent algorithms에서는 hazard pointer/epoch reclamation trade-off가 이런 문제를 다룬다.

GPU renderer에서는 보통 frame-bound reader가 짧고 timeline progress가 명확해 **epoch/fence reclamation이 더 단순**한 경우가 많다.

### 4.37 Renderer에서 Reference Count가 비싼 이유

매 snapshot마다 unchanged chunk 수십만 개에 ref++/ref--를 수행하면:

- atomic traffic
- cache-line contention
- snapshot retirement cost

가 커질 수 있다.

반면 fence-based retirement는 changed/retired allocation에만 metadata를 추가할 수 있다.

Dynamic sparse geometry에서는 **mutation-proportional bookkeeping**이 중요하다.

### 4.38 Snapshot Metadata 자체의 Retirement

Old snapshot table도 GPU가 더 이상 읽지 않을 때만 재사용할 수 있다.

따라서 table slot도:

```text
SnapshotSlot {
    epoch
    state
    safeAfter
}
```

를 가진다.

Payload allocation뿐 아니라 snapshot metadata buffer 자체도 lifecycle 관리 대상이다.

### 4.39 Ring of Snapshot Tables

작은 metadata table이라면:

```text
Table 0
Table 1
Table 2
Table 3
```

ring을 두고 consumer completion에 따라 재사용할 수 있다.

하지만 fixed ring이 모두 in-flight이면 producer가 기다려야 한다.

정책:

- grow ring
- wait/backpressure
- skip snapshot
- reuse latest compatible snapshot

중 workload semantics에 맞는 것을 선택한다.

### 4.40 Snapshot Backpressure

Producer가 renderer보다 빠르게 snapshot을 만들면 metadata/payload version이 계속 쌓인다.

예:

```text
Simulation 120 Hz
Rendering 60 Hz
```

모든 intermediate geometry snapshot을 render할 필요가 없을 수 있다.

그렇다면:

```text
E
E+1
E+2
```

중 아직 consumer가 잡지 않은 intermediate snapshot은 **coalesce/drop**할 수 있다.

단, simulation correctness와 render snapshot semantics가 허용할 때만 가능하다.

이것은 queue backpressure가 snapshot versioning으로 확장된 형태다.

### 4.41 “Latest Snapshot Wins” 정책

Visualization에서는 renderer가 모든 simulation step을 표시할 필요가 없다면:

```text
not-yet-consumed E+1
replaced by E+2
```

가 가능하다.

하지만 이미 Vulkan consumer가 E+1을 reference했다면 overwrite하면 안 된다.

즉 snapshot state:

```text
BUILDING
PUBLISHED_UNCLAIMED
CLAIMED
RETIRED
```

를 나누면 정책을 명확히 할 수 있다.

### 4.42 Snapshot Coalescing은 Latency를 줄일 수 있다

Renderer backlog가 있을 때 intermediate state를 모두 queueing하면 latency가 누적된다.

Interactive visualization에서는 latest state가 중요하므로:

```text
drop unclaimed intermediate snapshots
```

가 latency를 줄일 수 있다.

게임 logic처럼 every simulation state가 render semantics에 필요하지 않다면 유용하다.

### 4.43 C++ Ownership Model

Host-side type 예:

```text
GeometrySnapshotId
GeometryEpoch
AllocationHandle
AllocationGeneration
PublishValue
ReleaseValue
RetireRecord
```

API도 ownership을 표현하는 편이 좋다.

개념적 interface:

```text
beginSnapshot(baseEpoch)
patchChunk(handle, newAllocation)
finalizeSnapshot()
publishSnapshot()
retireCompleted()
```

Raw pointer를 곳곳에서 수정하는 architecture보다 lifetime review가 쉽다.

### 4.44 Strong Type으로 Timeline과 Epoch를 분리한다

모두 `uint64_t`여도:

```text
GeometryEpoch
RenderCompletionValue
CudaPublishValue
AllocationGeneration
```

은 다른 의미다.

Implicit conversion을 줄이면:

- wrong semaphore value
- wrong epoch compare
- generation/epoch 혼동

을 code review에서 잡기 쉽다.

### 4.45 Debug Invariants

유용한 invariant:

```text
PUBLISHED snapshot never mutates
```

```text
allocation reclaimed
⇒ no live snapshot references it
```

```text
snapshot E entry points to allocation A
⇒ A finalized before publish(E)
```

```text
renderer consumes E
⇒ publishTimeline >= required(E)
```

```text
allocator reuses A
⇒ releaseTimeline >= safeAfter(A)
```

Runtime validation이 어렵더라도 debug metadata로 capture할 수 있다.

### 4.46 Memory Budget Accounting

Allocator metric을 세 category로 분리한다.

```text
LiveReferencedBytes
RetiredPendingBytes
FreeBytes
```

추가로:

```text
BuildingBytes
FragmentationBytes
```

를 두면 COW가 실제 memory pressure에 미치는 영향을 파악하기 쉽다.

### 4.47 COW Update Ratio

매 frame:

```text
changed entries / total entries
```

를 측정한다.

Update ratio가 80~90%라면 incremental COW의 이득이 작아질 수 있다.

그때는:

- full rebuild
- new arena
- bulk table copy
- alternate snapshot strategy

가 더 단순하고 빠를 수 있다.

즉 **incremental architecture도 workload-dependent**다.

### 4.48 Full Rebuild Threshold

개념적 policy:

```text
if changedRatio < threshold:
    incremental patch
else:
    bulk rebuild
```

Threshold는:

- allocation cost
- fragmentation
- copy bandwidth
- table patch cost

를 profiler로 결정한다.

Sparse change에만 incremental을 적용하는 것이 중요하다.

### 4.49 Fragmentation과 COW

COW는 changed chunk마다 새 allocation을 만들기 때문에 allocator fragmentation을 증가시킬 수 있다.

따라서:

```text
COW update
→ old allocation retire
→ reclaim
→ periodic compaction
```

cycle이 필요하다.

이전의 mesh-pool defragmentation이 오늘 snapshot system 안에서 자연스럽게 돌아온다.

### 4.50 Snapshot + Defrag를 하나의 Transaction으로 본다

Defrag도 changed mapping list를 만든다.

```text
MovedAllocation {
    logicalChunk
    oldAllocation
    newAllocation
}
```

이를 다음 snapshot patch에 합칠 수 있다.

즉 simulation update와 allocator maintenance를 별도의 renderer update protocol로 만들 필요가 없다.

```text
Changed Mapping Set
= geometry changes
+ defrag changes
+ LOD residency changes
```

로 통합할 수 있다.

### 4.51 GPU-Driven Visibility와의 연결

Visibility worklist는 snapshot E의 logical handle을 해석한다.

```text
Visibility E
→ Snapshot Table E
→ Allocation
```

Visibility E work를 Snapshot E+1 table로 해석하면 다른 geometry를 볼 수 있다.

따라서:

```text
VisibilityEpoch
SnapshotEpoch
```

compatibility가 필요하다.

Frame-local worklist에 snapshot ID/epoch를 함께 묶는 이유다.

### 4.52 Mesh Shader와 COW Table

Mesh shader path에서는:

```text
VisibleMeshletID
→ Snapshot Meshlet Table
→ Chunk Allocation
→ vertex/primitive fetch
```

가 자연스럽다.

Persistent payload sharing과 logical ID indirection이 이미 필요하므로 COW snapshot mapping과 잘 맞는다.

Classic indexed draw에서는 `firstIndex/vertexOffset`이 command에 bake되므로 indirect command도 snapshot-specific resource가 될 수 있다.

### 4.53 Indirect Command도 Snapshot State다

Geometry mapping이 바뀌었는데 old `firstIndex/vertexOffset` command를 재사용하면 stale work다.

따라서 classic indirect pipeline에서는:

```text
Snapshot E
↔ Indirect Commands E
```

relation을 유지해야 한다.

Mesh shader처럼 late resolve가 많을수록 command state의 physical coupling은 줄어든다.

### 4.54 프로파일링에서 볼 지표

Incremental snapshot system:

- snapshot count in flight
- snapshot table bytes
- changed entries/frame
- changed ratio
- table-copy bytes
- payload new allocations/frame
- payload reused bytes
- COW allocation bytes
- live referenced bytes
- retired pending bytes
- reclaim bytes/frame
- retire latency in frames/ms
- oldest live epoch
- completed consumer timeline
- fragmentation ratio
- defrag moved bytes
- full rebuild frequency
- snapshot publish latency
- snapshot backlog
- dropped/coalesced unclaimed snapshots
- Vulkan wait time
- CUDA build time
- memory high-water mark

중요한 derived metric:

```text
Incremental Update Bytes
------------------------
Equivalent Full Snapshot Bytes
```

와:

```text
RetiredPendingBytes
-------------------
LiveReferencedBytes
```

다.

첫 번째는 incremental efficiency,
두 번째는 reclamation pressure를 보여준다.

---

## 5. 내 관심 분야와 연결

### Semiconductor process visualization

Process step마다 전체 wafer geometry가 바뀌는 것이 아니라 localized region이 바뀌는 경우가 많다.

예:

```text
substrate            unchanged
deep bulk oxide      unchanged
active etch front    changed
local deposition     changed
metal far region     unchanged
```

따라서:

```text
Dirty SDF Brick
→ Changed Mesh Chunk
→ New Allocation
→ Patch Snapshot Table
```

흐름이 매우 자연스럽다.

특히 얇은 layer가 많고 grid XY가 넓은 구조에서는 전체 geometry duplication보다 changed-column/chunk 중심 COW가 memory 측면에서 유리할 수 있다.

### GPU-stay-GPU pipeline

```text
CUDA / Warp
    ↓
Incremental Geometry Build
    ↓
Quiescence
    ↓
COW Snapshot Table E+1
    ↓
External Timeline Signal
    ↓
Vulkan Visibility / Rendering
    ↓
Release Timeline
    ↓
Epoch Reclamation
```

CPU가 geometry를 복사하거나 ref count를 순회할 필요가 없다.

### ColumnStack / Sparse Representation

ColumnStack이나 sparse brick representation의 natural chunk boundary를 snapshot entry boundary로 사용할 수 있다.

```text
Column / Brick Handle
→ payload allocation
```

Changed column만 mapping을 바꾸면 된다.

Representation 설계가 renderer lifetime 관리까지 직접 영향을 준다.

### CFD

CFD timestep에서도 전체 domain 중 일부 refinement region/iso-surface만 바뀔 수 있다.

Adaptive mesh refinement와 visualization geometry를 COW snapshot으로 publish하면 simulation과 rendering을 overlap시키기 좋다.

### Game engine / virtualized geometry

이 개념은 다음과 직접 연결된다.

- procedural terrain
- destruction
- streaming geometry
- virtualized mesh pages
- BLAS/input geometry update
- runtime asset hot reload
- GPU particle mesh output

특히 engine graphics role에서 중요한 질문은:

> **왜 double buffer가 무조건 안전한 해법이 아니며, immutable snapshot + deferred reclamation이 memory와 concurrency를 어떻게 분리하는가?**

다.

---

## 6. 머릿속에 남길 질문 3개

1. **Snapshot N과 N+1이 동일 allocation X를 공유하는 상태에서 N+1의 다른 chunk가 COW로 바뀌었을 때, snapshot lifetime과 allocation lifetime을 같은 refcount 하나로 관리하면 어떤 불필요한 traffic이나 lifetime coupling이 생길까?**
2. **Timeline semaphore value를 geometry epoch와 1:1로 대응시키는 설계와 render-frame completion value를 별도로 두는 설계는 multi-frame reuse와 debugging 측면에서 각각 어떤 장단점이 있을까?**
3. **Dynamic SDF scene에서 changed ratio가 높아질수록 incremental COW가 full rebuild보다 불리해지는 이유를 allocation churn, fragmentation, table patch bandwidth, reclamation pressure 관점에서 어떻게 설명할 수 있을까?**

---

## 7. graphics engineer 면접 질문 1개와 답변

### 질문

**“Dynamic GPU geometry를 안전하게 렌더링하려면 그냥 vertex/index buffer를 triple-buffer하면 충분하지 않나요?”**

### 답변

작은 geometry에서는 충분하고 좋은 설계일 수 있지만, 큰 dynamic geometry에서는 triple buffering이 memory와 lifetime 문제를 필요 이상으로 결합할 수 있다.

예를 들어 geometry payload가 4 GB라면 triple buffer만으로 12 GB가 필요하다. 그런데 frame마다 실제 변경되는 영역이 5%라면 대부분의 geometry를 중복 보관하는 셈이다.

더 scalable한 구조는 payload와 snapshot mapping을 분리하는 것이다.

```text
Persistent Payload Pool

Snapshot E:
Chunk A → Allocation 1
Chunk B → Allocation 2

Snapshot E+1:
Chunk A → Allocation 1   // shared
Chunk B → Allocation 3   // changed
```

이 구조에서 renderer-facing snapshot table은 immutable하고, changed chunk만 Copy-on-Write로 새 allocation을 가리킨다.

중요한 것은 old Allocation 2를 Snapshot E+1 publish 직후 free하면 안 된다는 점이다. Snapshot E를 사용하는 Vulkan frame이 아직 in-flight일 수 있다.

그래서 retire record에:

```text
safeAfterTimelineValue
```

같은 consumer completion condition을 저장하고, Vulkan timeline이 그 value를 지난 뒤에만 allocator가 old allocation을 재사용한다.

Timeline semaphore는 strictly increasing 64-bit progress value를 제공하므로 이런 fence-based epoch reclamation과 잘 맞는다.

핵심은:

> **Triple buffering은 “N개 full copies”로 concurrency를 해결하고, COW + epoch reclamation은 “immutable mappings + shared payload + actual consumer progress”로 concurrency를 해결한다.**

변경률이 낮고 geometry가 클수록 후자의 memory efficiency가 커질 수 있다.

---

## 8. 포트폴리오 / 커리어 연결

이 주제는 graphics engineer가 단순 buffer management가 아니라 **multi-frame concurrency와 allocator lifetime을 시스템적으로 이해한다는 것**을 보여주기 좋다.

### Rendering / GPU

- immutable frame snapshot
- meshlet table
- indirect command lifetime
- multi-frame in flight
- CUDA→Vulkan handoff

### Memory Management

- persistent payload pool
- Copy-on-Write
- retire list
- deferred reclamation
- fragmentation
- compaction

### Concurrency

- RCU-like read model
- epoch-based reclamation
- reference counting trade-off
- consumer progress tracking

### Vulkan / CUDA

- external timeline semaphore
- monotonic 64-bit synchronization value
- producer publish / consumer release
- multiple queue consumers

### Dynamic Geometry

- dirty SDF brick
- incremental meshing
- changed-chunk patch
- defrag as mapping update
- sparse snapshot update

### C++

- strong handle/epoch types
- snapshot descriptor
- allocation generation
- retire record
- invariant-based debugging

포트폴리오에서는 다음처럼 설명할 수 있다.

> **“Large dynamic geometry를 full triple-buffer하지 않고 persistent payload pool + immutable snapshot table로 분리했습니다. Changed chunk만 Copy-on-Write allocation을 만들고 new snapshot mapping을 patch하며, unchanged chunk는 여러 in-flight frame이 동일 payload를 공유합니다. Old allocation은 Vulkan completion timeline을 기준으로 retire-list에서 epoch reclamation하고, changed ratio가 높을 때는 incremental path 대신 bulk rebuild를 선택하도록 profiling했습니다.”**

이 설명은 rendering pipeline, GPU memory allocator, synchronization, data-oriented design을 하나로 연결한다.

---

## 9. 내일 이어서 볼 개념

**GPU Geometry Residency and Streaming: Page Tables, Working Sets, and Budget-Aware Eviction for Large Dynamic Scenes**

오늘은 여러 in-flight frame이 geometry payload를 공유하면서 changed chunk만 COW로 version-up하는 구조를 봤다.

다음 질문은:

> **Persistent payload pool 자체가 GPU memory budget보다 커지면 어떤 geometry를 resident하게 유지하고 어떤 geometry를 evict/stream해야 하는가?**

학습 흐름:

```text
Distributed Quiescence
→ Snapshot Publication
→ COW + Epoch Reclamation
→ Geometry Residency
→ Budget-Aware Streaming
```

다음 노트에서는:

- resident vs virtual geometry
- page/table indirection
- working set
- residency bit
- LRU가 GPU scene에서 부족한 이유
- visibility/LOD-driven residency priority
- eviction safety
- in-flight snapshot pinning
- staging/upload budget
- sparse/virtual buffer 관점
- dynamic SDF/meshlet streaming

을 중심으로 이어간다.

---

## 10. 참고 키워드

- Incremental GPU Snapshot
- Copy-on-Write (COW)
- Immutable Snapshot
- Persistent Payload Pool
- Snapshot Table
- Chunk-Level Indirection
- Page-Level COW
- Read-Copy-Update (RCU)
- Epoch-Based Reclamation (EBR)
- Deferred Reclamation
- Fence-Based Retirement
- Retire List
- Reference Counting
- Allocation Generation
- Object Generation
- Geometry Epoch
- Multi-Frame In Flight
- Timeline Semaphore
- Vulkan Timeline Semaphore
- CUDA External Semaphore
- CUDA-Vulkan Interoperability
- 64-bit Monotonic Timeline Value
- Persistent Geometry Pool
- Allocation Lifetime
- Snapshot Lifetime
- Deferred Destruction
- Memory High-Water Mark
- Retired Pending Bytes
- Fragmentation
- Defragmentation
- Incremental Meshing
- Dirty SDF Brick
- Meshlet Table
- Visibility Epoch
- Snapshot Backpressure
- Snapshot Coalescing
- Latest-Snapshot-Wins
- NVIDIA — **CUDA Programming Guide: CUDA Interoperability with APIs**
  - https://docs.nvidia.com/cuda/cuda-programming-guide/04-special-topics/graphics-interop.html
- NVIDIA — **CUDA Driver API: External Resource Interoperability**
  - https://docs.nvidia.com/cuda/cuda-driver-api/cuda_driver_api/group__CUDA__EXTRES__INTEROP.html
- Khronos — **Vulkan Specification: Synchronization and Cache Control**
  - https://docs.vulkan.org/spec/latest/chapters/synchronization.html
- Khronos — **Vulkan Tutorial: Timeline Semaphores**
  - https://docs.vulkan.org/tutorial/latest/Synchronization/Timeline_Semaphores/README.html
- Ajay Singh, Trevor Brown — **“Publish on Ping: A Better Way to Publish Reservations in Memory Reclamation for Concurrent Data Structures,” 2025**
  - https://arxiv.org/abs/2501.04250
