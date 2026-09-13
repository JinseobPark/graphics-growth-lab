---
title: "GPU Queue Backpressure and Forward Progress: Ring Buffers, Generation Tickets, and Deadlock-Free Persistent Scheduling"
date: "2026-09-13"
category: Graphics
tags: [GPU, Rendering, GPU-Driven Rendering, Persistent Threads, Work Queue, Ring Buffer, Backpressure, Forward Progress, Lock-Free Queue, Generation Ticket, CUDA, Vulkan, Work Graph, Memory Ordering, C++]
level: intermediate
---

# [Daily Graphics Growth] 2026-09-13 - GPU Queue Backpressure and Forward Progress: Ring Buffers, Generation Tickets, and Deadlock-Free Persistent Scheduling

## 1. 오늘의 개념

어제는 **GPU Visibility Work Graphs**에서 visibility, meshlet expansion, Hi-Z, LOD/material classification, render-packet generation을 고정된 pass chain이 아니라 **data-dependent work graph**로 바라봤다.

그 순간 다음 문제가 생긴다.

```text
Producer Node
   ↓ emits variable work
GPU Work Queue
   ↓
Consumer Node
```

어떤 frame에서는 하나의 input이 0개의 child work를 만들고, 어떤 frame에서는 수십 개를 만든다. Queue capacity가 무한하지 않기 때문에 producer가 consumer보다 빨라지면 결국 queue가 찬다.

CPU thread pool이라면 producer를 sleep시키거나 OS scheduler가 다른 consumer thread를 실행하게 만들 수 있다. GPU에서는 그렇게 단순하지 않다.

특히 **persistent kernel / persistent threads** 구조에서 resident worker들이 모두 다음과 같은 상태가 되면 위험하다.

```text
Queue = FULL

Resident workers:
    producer work를 처리 중
    → child enqueue 필요
    → 빈 slot을 기다리며 spin

Consumer-capable workers:
    아직 resident하지 못함
```

이 경우 “조금 기다리면 consumer가 실행되겠지”라는 가정 자체가 틀릴 수 있다. GPU scheduling과 residency는 CPU thread scheduling과 동일한 forward-progress contract를 제공하지 않기 때문이다.

오늘의 주제는 세 가지를 연결한다.

1. **Backpressure**  
   Producer가 queue capacity보다 빠르게 work를 생성할 때 시스템이 그 압력을 어떻게 흡수하는가.

2. **Generation / Sequence Ticket**  
   Ring-buffer slot이 반복 사용될 때 같은 index의 서로 다른 세대를 어떻게 구별하는가.

3. **Forward Progress**  
   일부 worker가 기다리는 동안 그 상태를 풀어줄 다른 worker가 실제로 실행될 수 있다는 보장을 어떻게 확보하는가.

핵심 질문은 다음이다.

> **GPU queue가 꽉 찼을 때 worker가 “기다리는 것”이 왜 deadlock이 될 수 있으며, bounded memory 안에서 work loss 없이 진행성을 유지하려면 queue protocol과 scheduling policy를 어떻게 함께 설계해야 하는가?**

---

## 2. 한 줄 핵심

> GPU persistent scheduling에서 안전한 backpressure는 **“FULL이면 spin”이 아니라, queue slot의 세대를 ticket으로 구분하고, enqueue 실패를 정상적인 control-flow state로 취급하며, producer가 consumer의 실행을 막지 않는 non-blocking fallback을 갖는 것**이다.

---

## 3. 왜 중요한가

GPU work graph가 커질수록 irregular fan-out이 커진다.

예를 들어 visibility pipeline:

```text
Chunk
  ↓ 0~N
Meshlets
  ↓ 0~M
Visible Meshlets
  ↓ 1~K
Depth / Main / Overlay Work
```

여기서 평균 fan-out만 보고 queue capacity를 잡으면 안 된다.

평균이 4여도 순간적으로 특정 frame 또는 특정 spatial region에서 fan-out 64가 몰릴 수 있다.

Semiconductor / CFD geometry에서는 특히 그렇다.

- 대부분 empty/flat brick
- 일부 high-curvature brick
- topology-changing region
- cut plane과 겹치는 surface
- meshlet rebuild가 집중되는 dirty island

같은 workload가 존재한다.

Queue overflow를 단순한 performance warning으로 처리하면 안 되는 이유는 work item이 다음 의미를 가질 수 있기 때문이다.

- visible meshlet
- remesh request
- topology repair task
- render packet
- indirect command source

이 중 하나가 사라지면 geometry가 누락되거나 rendering correctness가 깨질 수 있다.

따라서 bounded GPU queue에는 두 종류의 correctness가 있다.

### Data-structure correctness

- 같은 slot을 producer 두 개가 동시에 소유하지 않는가
- consumer가 publish 전 payload를 읽지 않는가
- wraparound 후 stale slot을 현재 slot로 착각하지 않는가

### Scheduling correctness

- queue가 full일 때 모든 resident worker가 기다리면서 시스템 전체가 멈추지 않는가
- consumer가 실제로 run할 기회를 갖는가
- overflow 상황에서도 work가 유실되지 않는가

Lock-free queue라고 해서 자동으로 deadlock-free GPU scheduler가 되는 것은 아니다.

> **Queue algorithm의 progress property와 GPU execution model의 progress property는 별개의 층이다.**

---

## 4. 구현 관점

### 4.1 Append Queue와 Persistent Ring Queue는 다른 문제를 푼다

Frame-local rendering pipeline에서는 append-only stream이 가장 단순할 수 있다.

```text
tail = atomicAdd(count, N)
write output[tail ... tail+N)
```

Stage boundary 뒤에 buffer를 reset한다.

장점:

- wraparound 없음
- producer/consumer 동시성이 단순
- prefix-sum/compaction과 자연스럽게 연결
- generation 문제가 적음

반면 persistent work graph에서는 producer와 consumer가 같은 queue를 장시간 재사용할 수 있다.

```text
head → dequeue
tail → enqueue
index = ticket mod Capacity
```

이때 필요한 것이 **ring-buffer lifetime protocol**이다.

Persistent queue가 더 “고급”인 것이 아니라, stage boundary를 없앤 대가로 더 복잡한 lifetime correctness를 직접 관리하는 것이다.

---

### 4.2 `head % capacity == tail % capacity`만으로 empty/full을 구분할 수 없다

Ring buffer가 한 바퀴 돌면 같은 physical index가 반복된다.

예를 들어 capacity가 1024라고 하자.

```text
ticket 5
ticket 1029
ticket 2053
```

모두:

```text
index = 5
```

를 사용한다.

Physical slot index만 보면 서로 다른 세대를 구분할 수 없다.

그래서 queue state는 보통 다음처럼 생각하는 편이 좋다.

```text
Logical Ticket = monotonically increasing sequence
Physical Slot  = ticket % capacity
Generation     = ticket / capacity
```

핵심은 physical index와 logical ownership을 분리하는 것이다.

이 패턴은 이전 노트에서 계속 등장한 원칙과 같다.

```text
Mesh Handle != Physical Vertex Slot
Work Ticket != Ring Slot Index
```

---

### 4.3 Sequence Ticket은 slot의 “현재 세대”를 표현한다

각 slot에 작은 state를 둘 수 있다.

개념적 구조:

```text
Slot {
    sequence
    payload
}
```

Producer ticket이 `t`라면:

```text
index = t % Capacity
expectedFreeSequence = t
```

라는 식으로 이 slot이 현재 producer generation에서 free인지 판정할 수 있다.

Publish 이후 consumer가 기대하는 sequence는 다른 phase가 되고,
consume 완료 후에는 다음 wrap generation이 사용할 sequence로 전환된다.

구체적인 숫자 encoding보다 중요한 것은 다음 state machine이다.

```text
FREE for generation G
        ↓ producer owns
WRITING
        ↓ publish
READY for generation G
        ↓ consumer owns
READING
        ↓ release
FREE for generation G+1
```

즉 generation ticket은 ABA-like reuse 문제를 줄이는 **slot lifecycle version**이다.

---

### 4.4 ABA 문제는 “값이 같아 보이지만 의미가 다르다”는 문제다

Queue slot 7이:

```text
A work
→ consumed
→ B work
→ consumed
→ C work
```

로 빠르게 재사용될 수 있다.

Consumer가 old observation을 들고 있다가 slot 7을 다시 보면 physical index와 일부 state가 동일하게 보일 수 있다.

하지만 현재 slot은 전혀 다른 work generation이다.

Generation/sequence를 함께 비교하면:

```text
slot 7, generation 10
!=
slot 7, generation 11
```

로 분리할 수 있다.

GPU systems에서 ABA-like 문제는 queue뿐 아니라 다음에 반복된다.

- mesh handle reuse
- visibility history slot
- relocation table entry
- descriptor pool slot
- frame resource ring
- indirect-command packet

따라서 **index + generation**은 graphics systems에서 매우 일반적인 lifetime pattern이다.

---

### 4.5 Reservation과 Publication을 분리한다

Producer가 slot을 얻었다고 바로 consumer가 읽어도 되는 것은 아니다.

두 단계가 있다.

```text
1. Reserve ownership
2. Write payload
3. Publish READY state
```

Consumer도:

```text
1. Observe READY
2. Read payload
3. Release slot
```

로 본다.

여기서 `tail` 증가만으로 publish를 의미하게 만들면, 다른 producer가 늦게 payload를 쓰는 동안 consumer가 아직 완성되지 않은 slot을 볼 수 있다.

따라서 **queue position reservation과 payload readiness는 서로 다른 state**다.

이것은 GPU indirect command buffer에서도 동일하다.

```text
command slot allocated
!=
command contents visible and executable
```

---

### 4.6 Acquire/Release는 payload와 state의 의미를 연결한다

Producer-consumer queue에서는 대략 다음 memory-order relation이 필요하다.

```text
Producer:
payload writes
    ↓
READY publish (release)

Consumer:
READY observe (acquire)
    ↓
payload reads
```

Release/acquire가 필요한 이유는 atomic flag 자체만 안전하게 읽기 위해서가 아니다.

> **READY를 봤다면 그 READY보다 앞선 payload write도 보여야 한다는 happens-before relation**을 만드는 것이다.

CUDA의 현재 programming guide도 producer가 data를 쓴 뒤 release-store로 flag를 publish하고, consumer가 acquire-load로 flag를 확인한 뒤 data를 읽는 message-passing pattern을 설명한다.

단순 counter 증가에는 relaxed atomic이 충분할 수 있지만,
**payload readiness를 전달하는 flag/sequence에는 별도의 ordering 의미가 필요하다.**

---

### 4.7 Atomic scope도 queue topology의 일부다

CUDA에서는 atomic operation의 **thread scope**가 중요하다.

예:

- block scope
- cluster scope
- device scope
- system scope

Queue producer/consumer가 서로 다른 thread block이라면 block-scoped atomic으로 global queue ownership을 표현하면 의미가 맞지 않는다.

반대로 GPU 내부 queue에 무조건 system scope를 사용하면 필요 이상으로 강한 synchronization과 memory traffic을 유발할 수 있다.

그래서 queue 설계의 질문은:

```text
Who can produce?
Who can consume?
Where can they run?
```

이다.

그 답이 atomic scope를 결정한다.

---

### 4.8 “Queue Full이면 spin”이 위험한 이유

다음 persistent-kernel 구조를 생각한다.

```text
while (true) {
    work = dequeue()

    children = process(work)

    for child:
        while (!enqueue(child)) {
            spin
        }
}
```

CPU에서는 직관적으로 보일 수 있다.

GPU에서는 queue가 full이 된 순간 resident worker 전체가 child enqueue에서 spin할 수 있다.

그 queue를 비울 consumer work가:

- 다른 block에 있는데 아직 scheduling되지 않았거나
- 같은 workers가 다음 loop iteration에서 수행해야 하거나
- 다른 kernel/queue에 있지만 resource starvation 상태

라면 full 상태를 해소할 주체가 실행되지 못한다.

즉:

```text
Producer waits for Consumer
Consumer needs GPU residency
Producer occupies residency
```

라는 circular wait가 된다.

이것이 **occupancy-induced deadlock** 또는 forward-progress hazard다.

---

### 4.9 GPU residency는 thread-pool fairness가 아니다

CUDA의 최신 execution-model 문서는 forward progress를 명시적으로 다룬다.

중요한 mental model:

- cooperative grid라면 grid thread progress에 더 강한 보장이 있다.
- 일반 execution에서는 모든 다른 thread-block cluster가 반드시 곧 실행된다고 가정할 수 없다.
- host-side API progress와 device-thread progress도 별도 contract다.

즉 “다른 block이 언젠가 실행될 테니 spin해도 된다”는 사고는 안전한 기본 가정이 아니다.

특히 persistent block 수를 hardware capacity까지 꽉 채우면 대기 중인 block이 들어올 자리가 사라질 수 있다.

---

### 4.10 Occupancy는 성능 지표이면서 scheduling capacity다

Occupancy를 latency-hiding metric으로만 보면 persistent scheduler의 중요한 측면을 놓친다.

Persistent kernel이:

- 많은 registers
- 많은 shared memory
- 큰 thread block

을 사용하면 SM당 resident block 수가 줄어든다.

이때 producer/consumer role을 서로 다른 block에 의존시키면 **실제로 동시에 resident 가능한 actor 수**가 queue progress에 영향을 준다.

즉 persistent system에서 occupancy는:

```text
Performance Capacity
+
Progress Capacity
```

다.

100% occupancy 자체가 목표는 아니지만, queue protocol이 기대하는 producer/consumer concurrency가 실제 residency에서 가능한지 확인해야 한다.

---

### 4.11 Backpressure는 “기다리기”보다 “정상 상태 전환”이어야 한다

Queue full을 exceptional failure가 아니라 scheduler state로 본다.

가능한 의미:

```text
ENQUEUE_OK
QUEUE_FULL
SPILL_REQUIRED
DEFER_REQUIRED
INLINE_EXECUTION
RETRY_LATER
```

핵심은 worker가 무한 spin하며 진행성을 다른 worker에게 맡기지 않는 것이다.

Backpressure policy는 queue capacity를 넘는 순간 **다른 execution path로 전환**할 수 있어야 한다.

---

### 4.12 Spill Queue는 bounded fast path를 보호한다

Hot queue는 작고 cache-friendly하게 유지하고 overflow는 별도 append-only spill buffer에 기록할 수 있다.

```text
Fast Ring Queue
      ↓ full
Spill Stream
      ↓
Later Drain Pass
```

장점:

- hot queue capacity를 worst-case까지 과도하게 키우지 않아도 됨
- full 상태에서 worker가 block되지 않음
- overflow가 work loss로 이어지지 않음

비용:

- later pass 필요
- latency 증가
- spill metadata/readback 또는 dispatch scheduling 필요

Rendering에서는 한 frame 내 처리해야 하는 work와 다음 stage/frame으로 미룰 수 있는 work를 구분해야 한다.

Visible geometry처럼 drop할 수 없는 work는 spill되더라도 반드시 drain되어야 한다.

---

### 4.13 Multi-pass fallback은 “패배”가 아니다

Persistent graph의 목적을 “kernel launch가 1번이어야 한다”로 잡으면 queue full에 위험한 blocking logic을 넣기 쉽다.

더 안전한 구조는:

```text
Persistent fast path
    ↓ pressure threshold
End / publish remaining work
    ↓
Next kernel or pass drains overflow
```

가 될 수 있다.

Kernel boundary는 비용이 있지만 동시에:

- global progress point
- resource residency reset
- simple synchronization boundary
- exact queue sizing opportunity

를 제공한다.

따라서 **가끔 발생하는 overflow에서 한 번의 relaunch를 허용하는 것**이 giant persistent kernel의 deadlock risk보다 훨씬 좋은 trade-off일 수 있다.

---

### 4.14 Producer가 Consumer 역할도 수행할 수 있다

Work type이 동일하거나 재귀적으로 처리 가능하다면 queue가 full일 때 producer가 child를 계속 enqueue하기보다 일부 work를 직접 처리하는 depth-first fallback을 생각할 수 있다.

개념적으로:

```text
breadth-first:
enqueue children

pressure high:
process one child locally
```

이 방식은 queue footprint를 줄일 수 있다.

하지만 단점도 있다.

- control-flow divergence
- recursion/state complexity
- one work item의 latency 증가
- specialized node architecture 약화

그래서 universal solution이 아니라 **fan-out이 큰 recursive work graph의 pressure valve**로 볼 수 있다.

---

### 4.15 Credit-Based Admission Control

Producer가 work를 시작하기 전에 “이 작업이 최대 몇 개 child slot을 필요로 하는가”를 알고 있다면 queue credit를 먼저 확보할 수 있다.

```text
availableCredits >= worstCaseFanOut
```

일 때만 expand하고,
아니면 defer한다.

장점:

- expansion 중간에 queue가 찰 가능성 감소
- partial child emission을 피하기 쉬움

단점:

- worst-case fan-out이 크면 utilization 저하
- variable fan-out에서 credit 낭비
- 여러 downstream queue가 있으면 bookkeeping 증가

이것은 networking의 backpressure와 비슷하다.

> **Capacity를 다 쓴 뒤 막는 것이 아니라, downstream capacity를 admission rule로 사용한다.**

---

### 4.16 High-Water / Low-Water Mark는 queue oscillation을 줄인다

Queue가 99%일 때 producer를 줄이고 98%가 되자마자 다시 모든 producer가 fan-out을 시작하면 occupancy가 계속 출렁일 수 있다.

Hysteresis를 둘 수 있다.

```text
occupancy > HIGH_WATER
    → expansion/deep fan-out 억제

occupancy < LOW_WATER
    → normal mode 복귀
```

Visibility hysteresis와 동일한 시스템 패턴이다.

작은 state 변화에 scheduler policy가 매번 뒤집히지 않게 한다.

---

### 4.17 Queue capacity는 node별 fan-out distribution에서 결정한다

단순 공식:

```text
capacity = inputCount × averageFanOut
```

은 burst를 놓친다.

더 유용한 관측값:

- fan-out histogram
- p95 / p99 fan-out
- producer batch size
- dequeue throughput
- maximum simultaneous producer count
- queue residency time
- burst duration

즉 capacity planning은 평균보다 **burst + drain rate** 문제다.

Render work graph에서는 scene별 worst case가 매우 다르므로 runtime telemetry가 중요하다.

---

### 4.18 Queue item은 payload보다 identity를 운반한다

어제 work graph note의 원칙을 그대로 유지한다.

```text
WorkItem {
    LogicalHandle
    LocalIndex
    Generation
    Epoch
    SmallFlags
}
```

Full meshlet payload, material data, physical address를 queue에 복사하면:

- bandwidth 증가
- ring capacity 감소
- relocation에 취약
- cache line utilization 악화

가 생긴다.

그래서:

> **Queue는 “무엇을 처리할지”를 운반하고, “현재 어디에 있는지”는 persistent table에서 late resolve한다.**

---

### 4.19 Queue item의 generation과 slot sequence는 다른 generation이다

둘을 구분해야 한다.

#### Slot Sequence

Ring-buffer physical slot의 reuse generation.

```text
slot 17 generation 42
```

#### Object Generation

Work item이 가리키는 logical object의 lifetime generation.

```text
MeshHandle { index=9, generation=103 }
```

둘은 독립적이다.

Queue slot은 정상인데 payload가 가리키는 mesh object가 이미 삭제/reused되었을 수 있다.

그래서 consumer는 다음 두 correctness를 별도로 본다.

```text
Queue ownership valid?
Object identity valid?
```

---

### 4.20 Epoch는 “old but valid memory” 문제를 잡는다

Object generation이 동일해도 geometry snapshot은 바뀔 수 있다.

예:

```text
same mesh handle
geometryEpoch 55 → 56
```

Queue에 오래 머문 work가 epoch 55 기준 visibility result라면 physical memory가 유효해도 semantic하게 stale하다.

Work record에 필요한 경우:

- geometryEpoch
- visibilityEpoch
- relocationEpoch
- commandEpoch

를 넣는 이유다.

Queue residency time이 길어질수록 stale-work probability가 커진다.

---

### 4.21 Reservation Counter overflow도 생각해야 한다

Monotonic ticket은 wrap ambiguity를 줄이지만 integer 자체도 유한하다.

32-bit ticket이 매우 빠른 persistent queue에서 장시간 실행되면 wrap될 수 있다.

64-bit ticket은 현실적인 runtime에서 훨씬 긴 safety margin을 제공한다.

중요한 것은 “64-bit니까 영원하다”가 아니라:

> **ticket width와 최대 enqueue rate를 이용해 wrap interval을 계산할 수 있어야 한다.**

또 64-bit atomic cost와 platform support도 함께 고려한다.

---

### 4.22 Warp-Centric enqueue는 contention을 줄인다

32 lanes가 각각 하나의 work를 emit한다고 하자.

Naive:

```text
32 global atomics
```

Subgroup/warp aggregation:

```text
ballot active lanes
leader reserves N slots once
laneRank → reservedBase + rank
```

로 바꿀 수 있다.

장점:

- global atomic 감소
- contiguous queue writes
- better coalescing

이는 이전의 meshlet compaction과 같은 패턴이다.

Queue algorithm에서는 contention과 memory-order complexity를 동시에 줄여준다.

---

### 4.23 Global queue 하나는 scalability bottleneck이 될 수 있다

모든 SM/warp가 같은 head/tail을 갱신하면:

- atomic serialization
- cache-line ping-pong
- contention retry
- queue metadata hot spot

이 생긴다.

그래서 다음 단계의 architecture는:

```text
Per-worker / Per-block / Per-SM local queue
        ↓
Occasional global balancing
```

으로 갈 수 있다.

오늘은 global bounded queue의 correctness가 중심이고,
내일은 여기서 **local queues + work stealing**으로 확장한다.

---

### 4.24 Lock-Free와 Wait-Free를 구분한다

Concurrency terminology를 GPU에 적용할 때 구분이 필요하다.

#### Lock-Free

System 전체로 보면 누군가는 계속 progress한다.

특정 worker가 starvation될 수 있다.

#### Wait-Free

각 operation이 bounded step 안에 완료된다.

더 강한 조건이다.

GPU queue 논문에서 lock-free라고 해도 GPU kernel 전체의 scheduling deadlock이 자동으로 사라지는 것은 아니다.

왜냐하면:

```text
queue operation progress property
+
GPU residency / scheduler progress property
```

가 함께 필요하기 때문이다.

그래서 “lock-free queue를 썼으니 deadlock-free”라는 결론은 성립하지 않는다.

---

### 4.25 Spin-wait은 memory bandwidth와 scheduler capacity를 동시에 소모한다

Busy loop:

```text
while (state != READY) { load state; }
```

는 아무 일도 하지 않는 것처럼 보여도:

- repeated atomic/load traffic
- cache-line contention
- warp issue slots
- power
- residency

를 소비한다.

Spin이 짧을 것이 확실한 block-local protocol은 괜찮을 수 있지만,
cross-block / cross-queue condition을 indefinite spin으로 기다리는 구조는 위험하다.

특히 기다림을 해소할 producer가 아직 resident하지 않은 경우 spin은 self-defeating이다.

---

### 4.26 Watchdog/Progress Counter는 silent hang 디버깅에 유용하다

Persistent kernel hang은 GPU fault가 아니라 “아무 것도 진행되지 않음”으로 나타날 수 있다.

Debug metadata:

```text
globalProgressCounter
lastDequeueTicket
lastEnqueueTicket
queueHighWater
overflowCount
workerStateHistogram
```

를 두면:

```text
GPU time passes
but progressCounter unchanged
```

상태를 탐지할 수 있다.

Worker state reason code:

```text
ACTIVE
QUEUE_EMPTY
QUEUE_FULL
SPILL
STALE_WORK
WAITING_DEPENDENCY
```

도 유용하다.

이것은 performance profiler이면서 liveness debugger다.

---

### 4.27 CUDA forward-progress semantics를 architecture input으로 본다

현재 CUDA execution model은 device-thread forward progress를 구체적으로 설명한다.

특히 중요한 점은 일반 grid의 모든 block이 서로 arbitrary spin dependency를 가져도 안전하다고 약속하는 모델이 아니라는 것이다.

Persistent scheduler가 grid-wide producer/consumer protocol을 필요로 한다면 다음 질문을 해야 한다.

- cooperative launch semantics가 필요한가?
- 동시에 resident 가능한 blocks로 protocol이 닫혀 있는가?
- cluster scope 안에서만 dependency를 만들 수 있는가?
- kernel boundary로 global progress point를 만드는 것이 더 단순한가?

즉 hardware scheduler의 “보통 이렇게 되더라”가 아니라 **documented progress domain**을 기준으로 architecture를 잡는 것이 중요하다.

---

### 4.28 Cooperative Grid는 강력하지만 capacity constraint가 생긴다

Grid-wide cooperative synchronization을 사용하는 persistent kernel은 모든 participating blocks가 cooperative residency 조건을 만족해야 한다.

그 대가로 stronger coordination을 얻는다.

하지만:

- launch size 제한
- resource usage와 resident block 수 관계
- register/shared-memory pressure
- 다른 GPU work와 concurrency 영향

을 같이 봐야 한다.

따라서 cooperative execution은 queue correctness를 쉽게 만들 수 있는 도구지만, 일반 rendering pipeline에서 항상 가장 좋은 선택은 아니다.

---

### 4.29 Native Work Graph runtime은 queue complexity를 숨기지만 capacity 문제를 없애지는 않는다

D3D12 Work Graphs 같은 native work-graph model에서는 application이 직접 ring queue를 구현하지 않을 수 있다.

Runtime/device가 node record scheduling과 backing memory를 관리한다.

하지만 시스템 관점의 질문은 여전히 남는다.

- fan-out이 얼마나 큰가
- backing memory 요구량이 얼마나 되는가
- node별 work imbalance가 어떤가
- record가 오래 머무는가
- state specialization이 잘 되어 있는가

즉 explicit queue bookkeeping은 줄어도 **backpressure라는 물리적 현실**은 사라지지 않는다.

---

### 4.30 Vulkan DGC는 work queue 자체가 아니다

`VK_EXT_device_generated_commands`는 GPU가 command sequence input을 만들고 execute하도록 하는 기능이다.

이것은 application-level irregular producer/consumer ring queue의 forward progress를 대신 관리해주는 scheduler가 아니다.

개념적으로:

```text
Work Scheduler
    ↓ render packets
DGC Input Stream
    ↓
Generated Draw / Dispatch Execution
```

이다.

Queue pressure는 DGC 이전 logical-work stage에서 발생할 수 있다.

이 구분은 어제의:

```text
Indirect != DGC != Work Graph
```

을 오늘의 liveness 관점으로 이어준다.

---

### 4.31 Queue backpressure policy는 work semantics를 알아야 한다

Overflow 시 모든 work를 동일하게 처리할 필요는 없다.

예:

#### Must-Preserve

- visible geometry
- topology repair
- mesh update required for snapshot correctness

→ spill/multi-pass로 반드시 보존

#### Deferrable

- expensive refinement
- non-critical diagnostic
- next-frame-safe maintenance

→ later queue로 이동 가능

#### Replaceable / Coalescible

- 동일 object의 중복 update request
- 같은 brick의 여러 dirty event

→ handle/epoch 기준 coalescing 가능

Backpressure는 단순 allocator 정책이 아니라 **domain semantics 기반 scheduler policy**다.

---

### 4.32 Dynamic SDF에서는 duplicate work coalescing이 매우 중요하다

같은 brick이 한 frame에서 여러 원인으로 dirty가 될 수 있다.

```text
geometry change
neighbor topology halo
LOD change
material update
```

각 event가 별도 queue item을 만들면 queue pressure가 불필요하게 증가한다.

Bitset/epoch 방식:

```text
brickDirtyEpoch[brick] = currentEpoch
```

로 “이미 enqueue됨”을 표시하면 duplicate work를 coalesce할 수 있다.

즉 backpressure를 해결하는 가장 좋은 방법 중 하나는:

> **queue를 더 크게 만드는 것이 아니라 애초에 redundant work를 넣지 않는 것**이다.

---

### 4.33 C++ host architecture에서는 queue contract를 타입으로 표현한다

Host-side graph description에서 다음이 모두 `uint32_t`면 잘못 연결하기 쉽다.

- QueueID
- WorkType
- SlotTicket
- ObjectGeneration
- GeometryEpoch
- Capacity

Strong type은 conceptual safety에 도움이 된다.

예:

```text
QueueTicket
MeshGeneration
GeometryEpoch
QueueCapacity
```

또 queue descriptor가 다음 policy를 명시할 수 있다.

```text
OverflowPolicy
ProgressDomain
ProducerScope
ConsumerScope
WorkPreservationClass
```

이렇게 하면 queue가 단순 buffer allocation이 아니라 **scheduler contract**로 보인다.

---

### 4.34 프로파일링에서 봐야 할 지표

Queue system은 평균 queue length만 보면 안 된다.

핵심 metrics:

- queue capacity
- queue occupancy histogram
- high-water mark
- enqueue rate
- dequeue rate
- p95/p99 queue residency time
- enqueue failure/full count
- spill count / bytes
- retry/spin iterations
- atomic operations per work item
- CAS retry count
- warp-aggregated reservation rate
- producer/consumer active ratio
- resident blocks / SM
- register/shared-memory pressure
- stale object-generation reject count
- stale epoch reject count
- duplicate-work coalescing rate
- progress counter delta / millisecond
- time in `QUEUE_FULL` worker state
- useful work / scheduler overhead

특히 다음 두 그래프를 함께 보는 것이 중요하다.

```text
Queue Occupancy over Time
GPU Resident Worker State over Time
```

Queue가 100%에서 오래 유지되면서 worker 대부분이 producer-full 상태라면 성능 문제가 아니라 liveness 위험 신호다.

---

## 5. 내 관심 분야와 연결

### 5.1 Semiconductor process emulation / visualization

Dynamic process simulation에서는 work가 균일하지 않다.

예:

```text
Flat substrate brick
→ 거의 no-op

Etch front brick
→ field update
→ surface extraction
→ topology update
→ many meshlets

Thin film / trench intersection
→ high-curvature / topology-heavy work
```

따라서 global queue capacity를 평균 brick workload만으로 잡으면 특정 process step에서 burst overflow가 발생할 수 있다.

사용자의 관심 pipeline을 연결하면:

```text
Warp / CUDA
  ↓
Dirty Brick Queue
  ↓
SDF / Level-Set Update
  ↓
Meshing Work Queue
  ↓
Meshlet Build Queue
  ↓
Published Geometry Snapshot
  ↓
Vulkan Visibility Queue
  ↓
Render Packet / DGC
```

여기서 queue마다 work semantics가 다르다.

- dirty brick update는 duplicate coalescing 가능
- topology repair는 must-preserve
- optional high-detail refinement는 deferrable할 수 있음
- visible meshlet packet은 current render snapshot에서 preserve되어야 함

즉 하나의 generic queue policy보다 **node별 preservation/backpressure policy**가 더 자연스럽다.

---

### 5.2 GPU-stay-GPU에서도 “CPU가 없으니 기다린다”가 답이 아니다

CUDA → Vulkan pipeline에서 CPU round-trip을 줄이는 것이 목표라도 queue pressure를 GPU spin으로 해결하면 오히려 GPU 전체 progress를 막을 수 있다.

GPU-stay-GPU의 좋은 의미는:

```text
CPU가 intermediate data를 처리하지 않는다
```

이지:

```text
모든 dependency를 하나의 persistent kernel 안에서 spin으로 해결한다
```

가 아니다.

Kernel boundary, external semaphore, indirect dispatch, DGC 같은 GPU-side chaining을 적절히 사용하면 CPU data processing 없이도 안전한 global progress point를 만들 수 있다.

---

### 5.3 Mesh extraction과 queue pressure

Marching Cubes / Dual Contouring 계열에서 한 brick이 생성하는 triangle/meshlet 수는 크게 달라질 수 있다.

따라서:

```text
input brick count
```

만 알고 output queue size를 정하기 어렵다.

이전 노트에서 다룬:

- curvature
- topology component count
- active-cell count
- QEF ambiguity

같은 값이 **fan-out predictor**가 될 수 있다.

즉 geometry metadata가 memory/backpressure budget에도 사용될 수 있다.

---

### 5.4 Rendering visibility graph와 연결

어제의 visibility work graph:

```text
Chunk Cull
  → Meshlet Expand
  → Hi-Z
  → Classification
  → Render Packet
```

에서 가장 fan-out이 큰 단계는 반드시 가장 비싼 단계와 같지 않다.

예를 들어:

- chunk → meshlet: fan-out 큼
- Hi-Z → visible: fan-in/rejection 큼
- material classification: fan-out 작지만 queue type이 다양

따라서 queue capacity와 worker allocation은 node별 통계를 봐야 한다.

---

### 5.5 Game engine 관점

이 개념은 다음과 직접 연결된다.

- GPU-driven scene traversal
- meshlet expansion
- virtualized geometry
- GPU particle spawning
- destructible geometry
- BVH traversal
- ray work queues
- work graphs
- persistent task runtimes

게임 엔진 graphics role에서 중요한 것은 “lock-free queue를 구현할 수 있다”보다 다음 설명이다.

> **GPU에서는 queue operation의 lock-freedom만으로 scheduler progress가 보장되지 않으며, residency와 producer/consumer dependency를 함께 설계해야 한다.**

---

## 6. 머릿속에 남길 질문 3개

1. **Bounded GPU queue가 full일 때 모든 producer가 retry-spin하는 구조가 CPU thread pool보다 훨씬 위험한 이유를 GPU residency와 forward-progress domain 관점에서 어떻게 설명할 수 있는가?**
2. **Ring-buffer slot의 sequence generation, payload가 가리키는 object generation, geometry epoch은 각각 어떤 다른 stale-state 문제를 막는가?**
3. **Dynamic SDF work graph에서 queue capacity를 키우는 것, duplicate dirty work를 coalesce하는 것, spill/multi-pass fallback을 두는 것 중 어느 선택이 memory bandwidth와 worst-case correctness에 가장 큰 영향을 주는가?**

---

## 7. graphics engineer 면접 질문 1개와 답변

### 질문

**“Persistent GPU kernel에서 bounded work queue가 full이면 enqueue가 성공할 때까지 atomic retry loop로 기다리면 되지 않나요? Queue는 lock-free라고 가정하겠습니다.”**

### 답변

그 구조는 GPU에서는 deadlock 또는 forward-progress failure를 만들 수 있다.

Lock-free라는 것은 queue data structure의 concurrent operation에 대한 progress property이지, GPU scheduler가 필요한 producer/consumer block을 반드시 동시에 resident하게 만든다는 뜻이 아니다.

예를 들어 모든 resident block이 work를 처리한 뒤 child work를 enqueue하려고 하고 queue가 full이라고 하자.

각 block이:

```text
while (queue_full)
    retry
```

에서 spin하면 resident resource를 계속 점유한다.

Queue를 비울 consumer 역할이 다른 non-resident block에 있거나, 현재 worker가 enqueue를 끝낸 뒤에야 consumer loop로 돌아갈 수 있다면 full 상태를 해소할 실행 주체가 없다.

따라서 robust persistent scheduler는 queue full을 blocking wait가 아니라 **explicit backpressure state**로 다룬다.

가능한 정책은:

- spill buffer
- later drain pass
- bounded admission / credit
- duplicate work coalescing
- local inline processing
- high-water mark에서 fan-out 억제

등이다.

또 ring slot reuse에는 sequence/generation ticket이 필요하고, payload publication에는 release/acquire memory ordering이 필요하다. Queue slot generation과 work payload의 object generation/geometry epoch도 별도 검증 대상이다.

핵심은:

> **Lock-free queue correctness + memory ordering + GPU residency-aware progress policy가 모두 있어야 persistent GPU scheduler가 안전하다.**

---

## 8. 포트폴리오 / 커리어 연결

이 주제는 graphics engineer가 GPU concurrency를 “atomic을 쓸 줄 안다” 수준이 아니라 **execution model과 memory model을 함께 이해한다**는 것을 보여주기 좋다.

### GPU Systems

- persistent threads
- bounded work queue
- backpressure
- forward progress
- occupancy/residency
- spill/multi-pass fallback

### Concurrency / C++

- logical ticket vs physical index
- generation / ABA
- reservation vs publication
- acquire/release
- atomic scope
- lock-free vs wait-free

### Rendering

- visibility work queue
- meshlet expansion
- render-packet generation
- must-preserve vs deferrable work
- indirect/DGC boundary

### Simulation / Geometry

- dirty brick coalescing
- variable fan-out
- topology-heavy burst
- incremental meshing
- geometry epoch

### Memory Layout

- compact queue records
- identity-only payload
- warp-aggregated reservation
- hot queue metadata
- spill stream

### Profiling / Debug

- high-water mark
- queue-residency histogram
- atomic retry count
- worker-state histogram
- progress watchdog
- overflow/coalescing counters

포트폴리오에서 강한 설명은 다음과 같다.

> **“GPU work graph에서 bounded queue가 full일 때 spin하지 않고 overflow를 scheduler state로 처리했습니다. Ring slot에는 monotonic sequence ticket을 사용해 reuse generation을 구분하고, queue payload는 logical handle + object generation + geometry epoch만 운반했습니다. Warp-level reservation으로 atomic contention을 줄였고, high-water mark·spill rate·progress counter를 profiling해 throughput뿐 아니라 liveness까지 관측했습니다.”**

이 설명은 rendering algorithm보다 한 단계 아래의 **GPU runtime/system architecture** 이해를 보여준다.

---

## 9. 내일 이어서 볼 개념

**Local GPU Queues and Work Stealing: Per-Block Deques, Load Balancing, and Divergence-Aware Task Scheduling**

오늘은 하나의 bounded queue가 pressure를 받을 때 correctness와 forward progress를 어떻게 유지하는지 봤다.

다음 단계는 global queue 자체의 contention을 줄이는 것이다.

```text
Global Queue
    ↓ contention / imbalance
Local Queue per worker group
    ↓
Work Stealing
    ↓
Load-balanced persistent GPU runtime
```

다음 노트에서는:

- global queue bottleneck
- per-block / local work queue
- owner-pop / thief-steal
- local LIFO vs global FIFO
- work stealing
- locality vs load balance
- warp divergence와 task type partition
- heterogeneous work queues
- 2026 GTaP의 work stealing / Execution-Path-Aware Queueing 관점
- graphics visibility/meshing graph에 적용할 때의 trade-off

를 연결한다.

---

## 10. 참고 키워드

- GPU Backpressure
- Forward Progress
- Persistent Threads / Persistent Kernel
- GPU Work Queue
- Bounded MPMC Queue
- Ring Buffer
- Monotonic Ticket
- Sequence Number
- ABA Problem
- Generation Counter
- Reservation vs Publication
- Acquire / Release
- `cuda::atomic`
- `cuda::thread_scope_device`
- Lock-Free / Wait-Free
- Occupancy / Residency
- Spill Queue
- Overflow Queue
- High-Water / Low-Water Mark
- Credit-Based Admission
- Warp-Aggregated Atomics
- Work Coalescing
- Duplicate Dirty Suppression
- Work Preservation Class
- Progress Watchdog
- Cooperative Grid
- CUDA Forward Progress
- Work Graph
- Device-Generated Commands
- GPU-Driven Rendering
- Dynamic SDF / Level Set
- Meshlet Expansion
- Yuki Maeda, Kenjiro Taura, **“GTaP: A GPU-Resident Fork-Join Task-Parallel Runtime with a Pragma-Based Interface,” 2026**
- Xiangyu Zhang, Yangdong Deng, Shuai Mu, **“Toward Concurrent Lock-Free Queues on GPUs,” 2014**
- David Troendle et al., **“A Specialized Concurrent Queue for Scheduling Irregular Workloads on GPUs,” 2019**
- Kshitij Gupta, Jeff A. Stuart, John D. Owens, **“A Study of Persistent Threads Style GPU Programming for GPGPU Workloads,” 2012**
- NVIDIA, **CUDA Programming Guide — CUDA C++ Execution Model / Forward Progress**
- NVIDIA, **CUDA Programming Guide — Scoped Atomics and Memory Ordering**
- NVIDIA, **CUDA C++ Best Practices Guide — Occupancy**
