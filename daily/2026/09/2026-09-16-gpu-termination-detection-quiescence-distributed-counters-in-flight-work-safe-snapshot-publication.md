---
title: "GPU Termination Detection and Quiescence: Distributed Counters, In-Flight Work, and Safe Snapshot Publication"
date: "2026-09-16"
category: Graphics
tags: [GPU, Rendering, GPU-Driven Rendering, Persistent Threads, Termination Detection, Quiescence, Work Queue, Snapshot Publication, CUDA, Vulkan, Timeline Semaphore, Memory Ordering, SDF, Meshing, C++]
level: intermediate
---

# [Daily Graphics Growth] 2026-09-16 - GPU Termination Detection and Quiescence: Distributed Counters, In-Flight Work, and Safe Snapshot Publication

## 1. 오늘의 개념

어제는 **Divergence-Aware GPU Task Partitioning**에서 local queue와 work stealing 위에 execution-path queueing, task coalescing, adaptive grain size를 얹어 worker-level load balance와 warp-level execution coherence를 함께 보는 방법을 다뤘다.

오늘은 그 runtime을 실제 rendering pipeline에 넘기기 위해 반드시 필요한 마지막 질문을 본다.

> **Local queue와 work stealing이 있는 GPU runtime에서 “정말 모든 geometry work가 끝났다”는 것을 어떻게 증명하고, 그 결과를 Vulkan renderer가 읽어도 되는 immutable snapshot으로 안전하게 publish할 것인가?**

단순히 모든 queue가 비었다고 끝난 것이 아니다.

```text
Q0 = empty
Q1 = empty
Q2 = empty

Worker 2 = task 실행 중
Worker 2 → 곧 child task 8개 생성
```

이 순간 queue는 모두 비었지만 computation은 끝나지 않았다. 따라서 termination은 적어도 다음 세 상태를 함께 봐야 한다.

```text
Queued Work
In-Flight Work
Pending Publication
```

이들이 모두 0이고, 새 work를 만들 수 있는 producer도 없다는 것이 같은 synchronization epoch에서 보장될 때 비로소 **quiescence(정지 상태)** 라고 말할 수 있다.

오늘은 여기서 한 단계 더 나아가:

```text
GPU Task Runtime
    ↓ quiescence
Geometry Snapshot Finalize
    ↓ publish epoch
CUDA External Semaphore Signal
    ↓
Vulkan Wait
    ↓
Immutable Snapshot Consume
```

까지 연결한다.

핵심은 **“더 이상 계산할 일이 없다”와 “renderer가 결과를 읽어도 된다”는 서로 다른 correctness 문제**라는 점이다.

---

## 2. 한 줄 핵심

> 안전한 GPU 종료 조건은 **ready queue가 비었는지만 보는 것이 아니라 queued·in-flight·publishing responsibility가 모두 사라졌음을 증명하고, 그 뒤 geometry epoch를 release-publication한 다음 CUDA→Vulkan semaphore로 consumer에게 전달하는 것**이다.

---

## 3. 왜 중요한가

GPU-stay-GPU pipeline에서는 CPU가 중간 결과를 읽고 “이제 다음 단계”라고 판단하지 않는다.

```text
SDF / Simulation
   ↓
Dynamic GPU Runtime
   ↓
Incremental Meshing
   ↓
Meshlet Build
   ↓
Vulkan Rendering
```

따라서 stage completion semantics를 GPU runtime이 직접 만들어야 한다.

잘못 설계하면 두 종류의 bug가 생긴다.

### Early termination

Worker가 아직 child work를 생성할 수 있는데 종료를 선언한다.

결과:

- 일부 brick remesh 누락
- topology repair 누락
- meshlet hole
- render packet 누락

### Early publication

Logical task는 끝났지만 geometry table/meshlet bounds/count/relocation mapping이 하나의 snapshot으로 finalize되기 전에 renderer가 읽는다.

결과:

```text
new meshlet table + old payload
new count + old index range
new relocation entry + old bounds
```

같은 **torn snapshot**이 생긴다.

Kernel completion이 강력한 이유는 이 문제를 암묵적인 global stage boundary로 단순화해 주기 때문이다. CUDA Cooperative Groups는 persistent blocks 안에서 grid-wide synchronization을 제공하지만, barrier는 known participants가 특정 지점에 도달했음을 보장할 뿐 **앞으로 새 child work가 생성되지 않는다는 distributed termination 자체를 자동으로 증명하지는 않는다.**

---

## 4. 구현 관점

### 4.1 `queue.empty()`는 termination이 아니다

잘못된 식:

```text
termination = queue.empty()
```

더 정확한 사고:

```text
Quiescent =
    ReadyWork == 0
AND InFlightWork == 0
AND PendingPublication == 0
AND NoProducerCanCreateMoreWork
```

문제는 이 상태들을 서로 다른 atomic counter로 단순히 읽는 것만으로도 race가 생길 수 있다는 점이다.

### 4.2 Work의 lifecycle을 책임(Responsibility)으로 본다

Task 하나가 system에 존재하는 동안 하나의 logical responsibility token이 있다고 생각하면 reasoning이 쉬워진다.

```text
READY task
  ↓ dequeue
IN_FLIGHT worker
  ↓ child 생성
child READY
  ↓
leaf completion
```

Queue에서 dequeue되었다고 responsibility가 사라지는 것이 아니다. Queue가 가지고 있던 책임을 worker가 이어받는다.

Parent가 child 3개를 만들면:

```text
parent responsibility
→ child A
→ child B
→ child C
```

로 전환된다.

이렇게 보면 queue와 worker 사이를 이동하는 순간에도 global responsibility가 갑자기 0이 되어서는 안 된다는 invariant가 선명해진다.

### 4.3 Outstanding Work Counter

가장 단순한 global invariant는 아직 시스템이 책임지는 logical work 수를 추적하는 것이다.

```text
outstandingWork
```

Child 생성:

```text
outstanding += childCount
```

Leaf/parent 완료:

```text
outstanding -= 1
```

중요한 것은 **child responsibility가 등록되기 전에 parent responsibility를 제거하지 않는 것**이다.

잘못된 순서:

```text
parent outstanding--
→ 순간적으로 0
→ child outstanding += N
```

Observer가 가운데를 보면 false termination이다.

따라서 implementation 세부와 상관없이 logical invariant는:

> **Child responsibility is established before parent responsibility is retired.**

가 되어야 한다.

### 4.4 Reservation과 Publication을 분리한다

Child queue slot을 예약했다고 child가 소비 가능한 것은 아니다.

```text
reserve
→ payload write
→ READY publish
```

Consumer는:

```text
observe READY
→ payload read
```

를 따른다.

이전 queue note와 동일하게 reservation과 publication은 다른 상태다. Termination detector도 **reserved-but-not-published work**를 잊으면 안 된다.

### 4.5 Acquire/Release는 publish와 payload를 연결한다

현재 CUDA memory model은 producer-consumer message passing에서:

```text
Producer:
payload write
ready.store(true, release)

Consumer:
ready.load(acquire)
payload read
```

형태의 acquire/release 관계를 설명한다.

Snapshot publication도 같은 mental model을 쓸 수 있다.

```text
geometry/table writes
→ publishedEpoch.store(E, release)

renderer-side logical consumer
→ observe epoch E
→ read finalized snapshot
```

핵심은 epoch 값 그 자체가 아니라 **epoch E를 봤다면 그 이전 payload writes도 함께 보인다는 ordering relation**이다.

### 4.6 Relaxed atomic과 publish atomic을 구분한다

모든 counter를 acquire/release로 만들 필요는 없다.

단순 통계:

```text
processedTaskCount
debugCounter
```

는 ordering을 전달하지 않는다면 relaxed atomic이 적합할 수 있다.

반면:

```text
queue READY
snapshotPublishedEpoch
ownership state
```

처럼 다른 memory payload의 readiness를 전달하는 값에는 더 강한 semantics가 필요하다.

### 4.7 Atomic scope는 participant domain과 맞춘다

CUDA scope:

```text
block
cluster
device
system
```

중 실제 participant 범위에 맞는 scope를 선택한다.

예:

```text
per-cluster scheduler → cluster scope
whole-GPU runtime     → device scope
GPU↔CPU shared state  → system scope
```

너무 좁으면 correctness가 깨지고, 너무 넓으면 unnecessary fence/cumulativity cost가 커질 수 있다.

### 4.8 Work stealing 중에는 “in-transit” 상태가 있다

Local deque에서 thief가 task를 가져갈 때:

```text
victim deque
→ steal reservation
→ thief ownership
```

사이의 순간이 존재한다.

Victim count에서는 task가 빠졌는데 thief responsibility가 아직 등록되지 않은 구조라면 false quiescence가 가능하다.

따라서 steal도:

```text
remove + later add
```

가 아니라 **ownership transfer**로 설계해야 한다.

### 4.9 Approximate queue length는 termination proof가 아니다

어제 victim selection에 사용한:

```text
approximateLoad[worker]
```

같은 값은 stale해도 성능만 손해봐야 하는 scheduling hint다.

Termination에는 사용할 수 없다.

Termination proof는:

- exact ownership
- exact responsibility
- synchronized generation/epoch

같은 correctness state를 사용해야 한다.

> **Scheduling hint는 틀려도 느려질 뿐이어야 하고, termination state는 틀리면 결과가 틀린다.**

### 4.10 Epoch-Based Quiescence

Graphics/simulation workload는 frame 또는 timestep이라는 자연스러운 epoch가 있다.

```text
Epoch E input
→ derived work
→ quiescence E
→ snapshot E
```

새 input을 E에 계속 주입하지 않고 E+1에만 넣도록 제한하면 termination reasoning이 매우 단순해진다.

정확한 조건은:

```text
No work remains for E
AND
No producer can still create work for E
```

다.

Fully general continuous runtime보다 **frame-bounded runtime을 활용하는 것 자체가 좋은 engine architecture**일 수 있다.

### 4.11 Barrier와 Termination은 다르다

Barrier:

> 모든 participant가 여기까지 왔다.

Termination:

> 모든 work가 끝났고 앞으로 새 work를 생성할 participant도 없다.

Dynamic task graph에서는 worker가 child work를 생성할 수 있기 때문에 두 조건은 동일하지 않다.

CUDA Cooperative Groups의 grid sync는 known participant의 stage synchronization에 강하지만 dynamic termination detector를 대신하지 않는다.

### 4.12 Coordinator / Finalizer

모든 worker가 동시에 snapshot finalization을 시도하기보다 하나의 designated coordinator 또는 atomic election winner가 마지막 transition을 담당하는 편이 단순하다.

Coordinator가 확인하는 것:

```text
outstanding responsibility == 0
local queue summary == empty
pending publication == 0
work generation stable
```

그 뒤에만 snapshot을 `FINALIZING → PUBLISHED`로 전환한다.

### 4.13 Stable-generation double check

Quiescence scan 중 새 work가 만들어지는 race를 줄이기 위해 mutation generation을 두 번 읽을 수 있다.

```text
g0 = workGeneration

scan queues / counters

g1 = workGeneration

if g0 == g1 and all-zero:
    quiescent candidate
else:
    retry
```

새 work가 publish될 때 generation이 바뀐다면 scan 도중 mutation이 있었음을 감지할 수 있다.

이는 renderer의 versioned snapshot과 같은 사고방식이다.

### 4.14 Task termination과 snapshot publication을 분리한다

Logical task가 모두 끝났어도 geometry resource set은 아직 다음 상태일 수 있다.

```text
vertex payload written
meshlet table written
bounds finalize pending
draw count pending
relocation table pending
```

따라서:

```text
Compute Quiescence
```

와:

```text
Snapshot Publication Quiescence
```

는 분리하는 것이 좋다.

Snapshot state machine:

```text
BUILDING
→ FINALIZING
→ PUBLISHED
→ CONSUMED
→ RETIRED
```

### 4.15 Snapshot은 여러 resource의 일관된 set이다

Renderer가 읽는 geometry snapshot은 하나의 pointer가 아니다.

예:

```text
vertex/index payload
meshlet table
bounds
material assignment
draw/mesh-task count
relocation mapping
geometry epoch
```

따라서 publication의 의미는:

> **이 resource set 전체가 epoch E라는 하나의 immutable interpretation을 가진다.**

이다.

### 4.16 Double/Triple Buffer보다 “immutable mapping”이 핵심이다

단순한 구현은 snapshot metadata를 double/triple-buffer하는 것이다.

```text
Snapshot 0 → Vulkan consuming
Snapshot 1 → CUDA building
Snapshot 2 → retired/available
```

하지만 dynamic geometry가 매우 크면 payload 전체를 복제하기 어렵다.

대안:

```text
Persistent Payload Pool
        ↑
Snapshot Handle Table E
Snapshot Handle Table E+1
```

Changed chunk만 새 allocation/record를 가리키고 unchanged geometry는 공유한다.

즉 snapshot의 본질은 **payload duplicate가 아니라 logical mapping의 immutability**다.

### 4.17 CUDA → Vulkan publish

CUDA의 현재 Vulkan interoperability는 external memory와 external synchronization object를 양쪽 API에서 공유하고 signal/wait로 execution order를 정의한다.

개념적으로:

```text
CUDA stream
  geometry work
  snapshot finalize
  signal semaphore(value = E)

Vulkan queue
  wait semaphore(value >= E)
  consume snapshot E
```

Timeline semaphore를 사용하면:

```text
geometry epoch 105
↔ timeline value 105
```

처럼 snapshot version과 synchronization value를 대응시키기 쉽다.

### 4.18 External semaphore는 termination detector가 아니다

Semaphore는:

```text
producer finished up to this synchronization point
```

를 consumer에게 전달한다.

하지만 **언제 signal해도 되는가**를 판단하는 것은 application runtime이다.

순서는:

```text
Quiescence Detection
→ Snapshot Finalize
→ External Semaphore Signal
→ Vulkan Wait
```

이다.

Semaphore가 local queue와 in-flight work를 자동으로 검사해 주는 것은 아니다.

### 4.19 Consumer 완료 뒤에만 resource를 retire한다

CUDA가 snapshot E를 publish했다고 E의 memory를 즉시 재사용할 수는 없다. Vulkan이 아직 읽고 있을 수 있다.

따라서 반대 방향의 lifetime transition도 필요하다.

```text
CUDA publishes E
→ Vulkan consumes E
→ Vulkan release/completion signal
→ allocator may retire E-only resources
```

이전 relocation note의 deferred retirement가 snapshot 단위로 확장된 것이다.

### 4.20 Publish와 Retire를 대칭으로 본다

전체 lifecycle:

```text
Build
→ Quiesce
→ Publish
→ Consume
→ Release
→ Retire
```

GPU-stay-GPU architecture는 ownership이 없는 구조가 아니라 **ownership transition을 GPU synchronization primitive와 epoch로 표현하는 구조**다.

### 4.21 Hierarchical completion aggregation

Worker 수가 많으면 모든 tiny task 완료마다 global `outstanding--`를 호출하는 것 자체가 hot spot이 될 수 있다.

Warp/block 단위로 completion을 aggregate할 수 있다.

```text
warp completed N tasks
→ leader atomicSub(outstanding, N)
```

또는:

```text
thread → block local count
block → global count
```

를 사용할 수 있다.

다만 local aggregation 중의 미반영 responsibility를 termination detector가 잊지 않도록 invariant를 설계해야 한다.

### 4.22 Quiescence detector는 hot path가 아니라 tail path다

매 task마다 global scan을 돌 필요는 없다.

Trigger 예:

- local queue가 비었을 때
- steal이 연속 실패할 때
- outstanding이 small threshold 아래일 때
- coordinator가 tail phase를 감지했을 때

중요한 metric은:

```text
last useful work finished
→ snapshot published
```

사이의 **quiescence detection latency**다.

Throughput은 좋아도 이 tail이 길면 frame latency가 늘어난다.

### 4.23 Invariant 기반 debugging

Debug build에서는 다음 관계를 검사할 가치가 있다.

```text
tasksCreated - tasksCompleted == outstanding
```

또:

```text
publishedEpoch <= finalizedEpoch
retiredEpoch <= rendererReleasedEpoch
```

그리고:

```text
renderer consumes E
⇒ snapshot[E].state == PUBLISHED
```

이런 invariant가 intermittent one-frame geometry hole을 찾는 데 매우 강하다.

### 4.24 C++에서는 epoch의 의미를 type으로 분리한다

다음 값이 전부 `uint64_t`면 잘못 연결하기 쉽다.

```text
frameIndex
geometryEpoch
relocationEpoch
publishTimelineValue
releaseTimelineValue
```

Host-side에서:

```text
RenderFrame
GeometryEpoch
RelocationEpoch
PublishFenceValue
ReleaseFenceValue
```

처럼 의미를 분리하면 synchronization review가 쉬워진다.

### 4.25 프로파일링에서 볼 지표

- tasks created/completed
- outstanding responsibility
- ready queue total
- in-flight worker count
- steal-in-transit count
- pending publication count
- quiescence checks/frame
- rejected false-quiescence candidates
- global atomic operations/task
- warp-aggregated completion ratio
- last work → quiescence latency
- quiescence → snapshot publish latency
- publish → Vulkan start latency
- Vulkan consume → release latency
- snapshot slots in use
- deferred-retire bytes
- stale epoch rejects
- external semaphore wait time

세 구간을 따로 보는 것이 좋다.

```text
Useful Work Tail
→ Runtime Quiescence Tail
→ CUDA/Vulkan Interop Tail
```

---

## 5. 내 관심 분야와 연결

### Semiconductor process emulation

Process step:

```text
SDF update
→ dirty brick expansion
→ surface extraction
→ topology repair
→ meshlet build
```

에서 topology-heavy worker가 마지막 순간 child work를 추가할 수 있다. 모든 local queue가 잠깐 비었다는 이유만으로 geometry를 publish하면 incomplete mesh가 renderer로 넘어간다.

따라서 정확한 **Geometry Complete**의 의미는:

```text
No ready work
No in-flight producer
No pending publication
```

이다.

### CUDA / Warp → Vulkan rendering

관심 pipeline을 가장 직접적으로 표현하면:

```text
CUDA / Warp
    ↓
Dynamic SDF / Meshing Runtime
    ↓
Quiescence for GeometryEpoch E
    ↓
Finalize Meshlet / Bounds / Counts
    ↓
Publish Snapshot E
    ↓
CUDA signals external timeline value E
    ↓
Vulkan waits E
    ↓
Render Snapshot E
    ↓
Vulkan release
    ↓
Old snapshot retire
```

CPU readback 없이도 ownership을 명확히 유지할 수 있다.

### Persistent sparse geometry

Vertex/mesh payload 전체를 frame마다 복사하는 대신 persistent pool을 공유하고 snapshot table만 versioning할 수 있다.

```text
Persistent Geometry Pool
      ↑
Table E
Table E+1
```

Changed brick만 새 allocation을 가리키고 unchanged chunk는 공유한다.

### CFD timestep

CFD는 timestep epoch가 명확하므로:

```text
step N compute
→ derived visualization work
→ quiescence
→ snapshot N
```

구조가 자연스럽다. Domain semantics가 termination detection을 단순화해 준다.

### Game engine / graphics career

이 개념은 procedural terrain, destructible geometry, GPU particle generation, virtualized geometry, async simulation/render handoff와 직접 연결된다.

면접에서 중요한 답은:

> **“queue empty는 work ownership이 끝났다는 뜻이 아니며, quiescence와 memory publication, consumer lifetime을 별도로 증명해야 한다.”**

이다.

---

## 6. 머릿속에 남길 질문 3개

1. **Per-worker queue가 모두 empty인데도 global termination이 아닐 수 있는 이유를 in-flight producer, steal-in-transit, pending publication 세 상태로 어떻게 설명할 수 있는가?**
2. **`outstandingWork == 0`을 termination proof로 사용할 때 child responsibility 등록과 parent completion 순서를 잘못 잡으면 어떤 false-zero race가 생기는가?**
3. **CUDA가 geometry epoch E를 완성한 뒤 Vulkan이 읽도록 만들 때 quiescence detection, memory publication, external semaphore signal, old snapshot retirement은 각각 어떤 서로 다른 correctness 문제를 해결하는가?**

---

## 7. graphics engineer 면접 질문 1개와 답변

### 질문

**“GPU work-stealing runtime에서 모든 local queue가 empty이고 global counter도 0이면 바로 Vulkan renderer에게 geometry를 넘겨도 되나요?”**

### 답변

Counter가 어떤 invariant를 표현하는지에 따라 충분하지 않을 수 있다.

Worker가 task를 dequeue한 순간 responsibility를 counter에서 빼고, child work를 publish하기 전에 parent를 완료 처리한다면 순간적으로 0이 될 수 있다. Work stealing 중 task가 victim queue에서는 빠졌지만 thief ownership으로 완전히 이전되지 않은 상태도 같은 문제를 만든다.

따라서 counter는 queue entry 수가 아니라 **아직 system이 책임지는 logical work**를 추적해야 한다.

그 뒤에도 한 단계가 더 필요하다. Logical work가 모두 끝났다고 해서 vertex/index payload, meshlet table, bounds, counts, relocation mapping이 자동으로 하나의 consumer-safe snapshot이 되는 것은 아니다.

Robust한 순서는 다음과 같다.

```text
1. Global quiescence 증명
2. Geometry metadata/table/count finalize
3. Snapshot epoch E publish
4. CUDA external semaphore/timeline value E signal
5. Vulkan waits E
6. Vulkan consumes immutable snapshot E
7. Vulkan release 이후 old resources retire
```

즉:

> **Termination은 “더 이상 계산할 일이 없음”, publication은 “결과 memory가 일관된 version으로 읽을 수 있음”, semaphore는 “그 version의 producer→consumer 실행 순서”를 해결한다.**

세 문제는 서로 대체할 수 없다.

---

## 8. 포트폴리오 / 커리어 연결

이 주제는 GPU compute algorithm뿐 아니라 **runtime → memory model → renderer handoff**를 이해한다는 것을 보여주기 좋다.

포트폴리오에서 연결할 포인트:

- **GPU Runtime:** distributed termination, quiescence, in-flight ownership
- **Concurrency:** responsibility counter, acquire/release, atomic scope
- **Dynamic Geometry:** SDF dirty work, topology repair, meshlet build
- **Rendering:** immutable snapshot, compute→graphics handoff
- **Vulkan/CUDA:** external semaphore, timeline value, deferred retirement
- **Memory Layout:** persistent payload pool, versioned handle table
- **C++:** strong epoch/fence types, invariant-based validation
- **Profiling:** useful-work tail / quiescence tail / interop tail 분리

좋은 설명은 다음과 같다.

> **“GPU work-stealing runtime completion을 local queue empty로 판단하지 않고 logical outstanding-responsibility invariant로 추적했습니다. Child publication과 parent retirement의 ordering으로 false-zero를 막고, quiescence 뒤 meshlet/bounds/count table을 geometry epoch로 finalize했습니다. CUDA가 timeline value E를 signal하면 Vulkan은 snapshot E만 읽고, renderer release 이후에만 old allocation을 retire하도록 lifetime을 분리했습니다.”**

---

## 9. 내일 이어서 볼 개념

**Incremental GPU Snapshot Publication: Copy-on-Write Tables, Epoch-Based Reclamation, and Multi-Frame Geometry Sharing**

오늘은 dynamic runtime이 **언제 끝났다고 말할 수 있는가**와 그 결과를 renderer에게 publish하는 protocol을 봤다.

다음 질문은:

> **매 frame 전체 geometry를 double-buffer하지 않고 변경된 chunk만 version-up하면서 여러 in-flight frame이 같은 geometry payload를 안전하게 공유하려면 어떻게 해야 하는가?**

학습 흐름:

```text
Local GPU Runtime
→ Distributed Quiescence
→ Safe Snapshot Publication
→ Copy-on-Write Snapshot Tables
→ Epoch-Based Reclamation
```

다음 노트에서는 COW geometry table, changed-chunk patch, fence-based retirement, reference counting과 epoch reclamation의 차이, multi-frame geometry sharing, sparse update bandwidth를 연결한다.

---

## 10. 참고 키워드

- GPU Termination Detection
- Distributed Quiescence
- Outstanding Work Counter
- Responsibility Token
- In-Flight Work
- Work Stealing
- In-Transit Task
- Pending Publication
- False-Zero Termination
- Epoch-Based Termination
- Snapshot Publication
- Immutable Snapshot
- Geometry Epoch
- Acquire / Release
- `cuda::atomic`
- Atomic Thread Scope
- Memory Synchronization Domain
- Cooperative Groups
- `grid_group::sync`
- Persistent Kernel
- Double / Triple Buffering
- Copy-on-Write
- Persistent Geometry Pool
- Deferred Retirement
- Timeline Semaphore
- CUDA External Semaphore
- CUDA-Vulkan Interoperability
- Snapshot ABI
- Dynamic SDF
- Incremental Meshing
- Meshlet Table
- NVIDIA — CUDA Programming Guide, **Cooperative Groups**
  - https://docs.nvidia.com/cuda/cuda-programming-guide/04-special-topics/cooperative-groups.html
- NVIDIA — CUDA Programming Guide, **Advanced Synchronization Primitives / Scoped Atomics**
  - https://docs.nvidia.com/cuda/cuda-programming-guide/03-advanced/advanced-kernel-programming.html
- NVIDIA — CUDA Programming Guide, **CUDA C++ Memory Model**
  - https://docs.nvidia.com/cuda/cuda-programming-guide/05-appendices/cuda-cpp-memory-model.html
- NVIDIA — CUDA Programming Guide, **Memory Synchronization Domains**
  - https://docs.nvidia.com/cuda/cuda-programming-guide/04-special-topics/memory-sync-domains.html
- NVIDIA — CUDA Programming Guide, **Vulkan Interoperability**
  - https://docs.nvidia.com/cuda/cuda-programming-guide/04-special-topics/graphics-interop.html
- Khronos — Vulkan Specification, **Synchronization and Cache Control / Semaphores**
  - https://docs.vulkan.org/spec/latest/chapters/synchronization.html
- Edsger W. Dijkstra, C. S. Scholten, **“Termination Detection for Diffusing Computations,” 1980**
