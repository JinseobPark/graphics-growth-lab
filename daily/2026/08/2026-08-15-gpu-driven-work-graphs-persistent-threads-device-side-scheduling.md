---
title: "GPU-Driven Work Graphs and Persistent Threads: Device-Side Scheduling for Irregular Rendering Pipelines"
date: "2026-08-15"
category: Graphics
tags: [GPU, Work Graphs, Persistent Threads, Device-Side Scheduling, Wavefront Rendering, ReSTIR, SIMT, Work Queue, HLSL, Direct3D 12, Compute Shader, Memory Layout, C++]
level: intermediate
---

# [Daily Graphics Growth] 2026-08-15 - GPU-Driven Work Graphs and Persistent Threads: Device-Side Scheduling for Irregular Rendering Pipelines

## 1. 오늘의 개념

어제는 **wavefront queue scheduling**과 **path-state binning**으로 불규칙한 렌더링 작업을 여러 queue와 kernel로 분해하는 방법을 보았다. 오늘은 그 다음 단계인 **device-side scheduling**, 즉 GPU가 다음에 실행할 일을 스스로 생성하고 이어서 실행하는 구조를 본다.

전통적인 wavefront renderer는 GPU가 `SurfaceQueue`, `ShadowQueue`, `ReplayQueue` 같은 work queue를 채우더라도, 실제 다음 dispatch를 제출하는 주체는 여전히 CPU command stream인 경우가 많다. `ExecuteIndirect`를 사용하면 dispatch 크기 자체는 GPU가 결정할 수 있지만, 어떤 shader/pipeline이 다음에 실행될지에 대한 선택은 상대적으로 제한적이다.

이 문제를 해결하려는 대표적인 두 방향이 있다.

- **Persistent Threads / Persistent Kernel**: 일정 수의 GPU worker를 오래 resident시킨 뒤 software work queue에서 일을 꺼내 처리한다.
- **D3D12 Work Graphs**: shader node가 다른 node에 **record**를 출력하면 GPU runtime이 이후 node 실행과 record lifetime을 관리한다.

둘 다 목표는 비슷하다.

> CPU가 매 단계의 dispatch를 미세하게 지휘하는 대신, GPU가 데이터 의존적으로 다음 work를 생성하고 scheduling한다.

하지만 abstraction level과 memory ownership은 크게 다르다.

```text
CPU-driven wavefront
CPU -> Dispatch A -> Queue -> Dispatch B -> Queue -> Dispatch C

Persistent threads
CPU -> Persistent workers
          -> dequeue task
          -> execute
          -> enqueue next task
          -> repeat

D3D12 Work Graphs
CPU -> DispatchGraph
          Node A -> record -> Node B
                 -> record -> Node C
                           -> record -> Node D
```

Microsoft의 2026년 2월 Work Graphs specification은 이를 **GPU based work creation**으로 정의한다. producer node는 consumer node의 실행을 요청하는 record를 생성하고, runtime은 input이 준비되면 적절한 시점에 consumer를 scheduling한다.

## 2. 한 줄 핵심

> **Persistent threads는 애플리케이션이 GPU 내부 software scheduler와 queue를 직접 소유하는 방식이고, Work Graphs는 producer-consumer work creation과 record lifetime을 API/runtime에 맡겨 irregular GPU pipeline의 scheduling을 구조화하는 방식이다.**

## 3. 왜 중요한가

### CPU launch chain의 한계

Wavefront architecture가 divergence를 줄이더라도 stage 수가 늘어나면 다음 비용이 생긴다.

\[
T_{frame} \approx \sum_i
(T_{kernel,i} + T_{launch,i} + T_{barrier,i} + T_{queue,i})
\]

특히 queue의 실제 크기와 다음 stage가 GPU 실행 중에 결정되는 경우 CPU가 사전에 최적 dispatch sequence를 알기 어렵다.

ReSTIR PT를 예로 들면 한 candidate가 다음 중 어디로 갈지는 runtime data에 따라 달라진다.

```text
compatibility reject
short random replay
long random replay
reconnection
visibility revalidation
final shading
```

이런 workload는 **data-dependent work expansion**이다. GPU가 결과를 만든 직후 그 결과에 맞는 다음 work를 생성할 수 있다면 CPU round trip과 worst-case dispatch를 줄일 수 있다.

### Persistent threads가 해결하는 것

Persistent kernel은 일반적으로 GPU에 일정한 worker set을 resident시킨 뒤 global work queue를 반복해서 소비한다.

개념적으로는 다음과 같다.

\[
\text{worker} \rightarrow \text{dequeue} \rightarrow \text{process} \rightarrow \text{enqueue successors}
\]

장점은 매우 높은 scheduling 자유도다. 작업 priority, queue policy, task stealing, state machine까지 모두 application이 정의할 수 있다.

반면 application이 다음도 책임져야 한다.

- queue capacity와 overflow
- atomics와 contention
- termination detection
- fairness
- producer/consumer synchronization
- resident block 수와 occupancy
- 장시간 실행되는 worker 때문에 생기는 resource pinning

즉 persistent threads는 강력하지만 **scheduler 자체가 renderer의 일부**가 된다.

### Work Graphs가 바꾸는 ownership

D3D12 Work Graphs에서는 shader가 output record를 생성하고 target node를 선택한다. runtime은 record가 준비된 node를 hardware에 맞춰 scheduling할 자유를 가진다.

개발자는 다음을 기술한다.

```text
어떤 node가 존재하는가
어떤 record가 node 사이를 이동하는가
각 node가 어떤 launch mode를 사용하는가
최대 work expansion이 어느 정도인가
```

반면 실제 queue representation, 일부 scheduling detail, record spill은 implementation이 관리한다.

이 차이를 간단히 정리하면 다음과 같다.

| 관점 | Persistent Threads | D3D12 Work Graphs |
|---|---|---|
| Scheduler ownership | Application | Runtime / Driver / Hardware |
| Work representation | Custom queue entry | Node record |
| Kernel lifetime | Long-lived worker | Node invocation 단위 |
| Dynamic shader selection | 직접 구현 | Node 선택으로 표현 |
| Memory management | Queue/pool 직접 관리 | Record는 system-managed, bulk data는 UAV 가능 |
| Portability | CUDA/compute 환경별 별도 설계 | D3D12 표준 feature |
| Control | 매우 높음 | 추상화된 scheduling |
| Complexity | 높음 | 상대적으로 구조화됨 |

### Work Graphs는 "함수 호출 그래프"가 아니다

Node A가 Node B에 record를 출력한다고 해서 일반적인 함수 호출처럼 B가 끝난 후 A로 return하는 구조가 아니다.

Work Graphs의 mental model은 **continuation + dataflow**에 가깝다.

```text
Producer
  -> record
     -> Consumer
```

producer와 consumer의 실행은 분리되어 있으며 runtime은 가능한 시점에 consumer를 실행할 수 있다. 따라서 call stack보다는 **asynchronous producer-consumer pipeline**으로 이해하는 것이 중요하다.

### 2026년 기준 Work Graphs의 위치

Microsoft DirectX-Specs의 Work Graphs specification v1.012는 2026-02-04에 갱신되어 있다. Tier 1.0은 compute 기반 node를 중심으로 한 첫 정식 Work Graphs 기능이며, specification에는 graphics node와 mesh node 방향도 기술되어 있지만 **graphics nodes는 여전히 proposed / not supported yet**, Tier 1.1은 unreleased prototype으로 표시되어 있다.

따라서 현재 실무에서 Work Graphs를 생각할 때는 먼저 **GPU-driven compute dataflow scheduler**로 보는 것이 정확하다.

## 4. 구현 관점

### Node launch mode

Work Graphs에는 compute 기반 node를 실행하는 세 가지 중요한 방식이 있다.

#### Broadcasting Launch

하나의 input record가 dispatch grid 전체에 보인다.

```text
1 record -> many thread groups
```

전통적인 `Dispatch()`와 가장 비슷하며 tile, block, batch 단위 work에 적합하다.

#### Thread Launch

각 input record가 독립된 thread work가 된다.

```text
record 0 -> thread 0
record 1 -> thread 1
record 2 -> thread 2
```

작은 task가 많이 발생하는 producer-consumer pipeline에 자연스럽다.

#### Coalescing Launch

여러 input record를 하나의 thread group이 함께 소비할 수 있다.

```text
records {0..N} -> one thread group
```

작은 work item을 group 수준으로 묶어 처리하므로 explicit queue compaction과 유사한 역할을 일부 runtime에 맡길 수 있다.

### ReSTIR PT workload에 적용하는 사고방식

어제의 explicit shift-class binning을 Work Graph mental model로 바꾸면 다음처럼 볼 수 있다.

```text
Reuse Classifier Node
  -> Reject / Finish
  -> ShortReplay Record -> ShortReplay Node
  -> LongReplay Record  -> LongReplay Node
  -> Reconnect Record   -> Reconnection Node
  -> Visibility Record  -> Visibility Node
```

중요한 점은 node selection이 **estimator semantics를 결정하는 단계와 execution scheduling을 혼동하면 안 된다**는 것이다.

먼저 candidate의 probability, validity, shift mapping, bias correction이 수학적으로 결정되어야 하고, 이후 이미 결정된 작업을 어느 node에서 실행할지가 scheduling layer다.

\[
\text{Sampling Decision} \neq \text{Execution Node Selection Policy}
\]

실행 architecture를 바꾸더라도 estimator가 계산하는 distribution과 weight가 의도치 않게 달라져서는 안 된다.

### Record는 hot state로 생각한다

Work Graph record가 system-managed라고 해서 큰 path payload를 모두 record에 넣는 것이 좋은 것은 아니다.

record에는 다음처럼 **다음 node가 즉시 필요로 하는 hot state**를 두는 편이 자연스럽다.

```text
pathId
pixelId
reservoirHandle
subpathHandle
ray / hit summary
materialClass
shiftClass
flags
small scalar parameters
```

반대로 큰 payload는 SRV/UAV 또는 persistent pool에 둔다.

```text
full subpath vertices
large material state
historical reservoir payload
large visibility/reconnection metadata
```

따라서 어제와 그 전날에 정리한 구조가 그대로 이어진다.

```text
Logical path ID
   -> compact node record
      -> stable handle
         -> persistent GPU pool
```

### Backing memory

Work Graphs는 node input record와 graph execution context를 위해 **backing memory**를 사용할 수 있다.

Microsoft specification은 system이 record를 가능한 한 on-chip cache에 유지할 수 있지만 필요하면 backing memory에 spill할 수 있다고 설명한다. Application은 `GetWorkGraphMemoryRequirements()`를 통해 최소/최대/size granularity를 확인하고 allocation을 제공한다.

이를 performance 관점에서 보면 다음 관계를 생각할 수 있다.

\[
M_{backing}\propto
N_{inflight\ records}\times S_{record}\times F_{expansion}
\]

정확한 메모리 크기는 driver가 결정하지만 개발자가 control할 수 있는 중요한 변수는 다음이다.

- record size
- 최대 output record 수
- dispatch grid upper bound
- 동시에 살아 있을 수 있는 dataflow 폭

즉 **작은 record + 현실적인 expansion bound**가 memory footprint와 scheduler 부담을 줄이는 기본 방향이다.

### Record vs UAV

node 사이의 작은 control/data payload는 record가 자연스럽다.

```text
record = work description + compact hot data
```

하지만 큰 데이터는 UAV를 사용하는 것이 적합하다.

```text
record -> handle / offset
UAV    -> bulk payload
```

Work Graph specification도 record size에 제한이 있으므로 bulk transfer는 UAV를 사용하도록 설명한다. 다만 producer UAV write를 consumer가 즉시 읽는 구조는 visibility/synchronization 규칙을 따라야 하며, `globallycoherent` 또는 적절한 atomic/barrier semantics가 필요할 수 있다.

따라서 Work Graphs는 "intermediate buffer가 사라진다"가 아니라 다음처럼 보는 편이 정확하다.

> **control-flow queue의 일부가 runtime-managed record로 이동하고, bulk state는 여전히 application-owned GPU memory에 남는다.**

### Persistent threads의 memory layout

Persistent scheduler에서는 work entry 자체를 직접 설계한다.

예를 들어 logical layout은 다음과 같이 나눌 수 있다.

```text
Queue Header
- read cursor
- write cursor
- task count
- termination state

Task Entry
- type
- pathId
- payloadHandle
- compact parameters
```

Task payload를 entry 안에 크게 넣으면 global-memory bandwidth가 증가한다. 반대로 handle indirection이 너무 많으면 random access와 latency가 커진다.

따라서 persistent threads에서도 핵심은 동일하다.

\[
\text{Queue Locality} \quad vs. \quad \text{Payload Size}
\]

### C++ / D3D12 render graph 관점

Work Graphs를 renderer에 넣으면 C++ 쪽 abstraction도 기존 compute pass와 달라진다.

전통적인 render graph node는 보통 다음 contract를 가진다.

```text
Pass
- input resources
- output resources
- pipeline state
- dispatch dimensions
```

Work Graph pass는 여기에 추가로 다음 state가 필요하다.

```text
WorkGraph Program
- graph state object
- entrypoints
- backing memory allocation
- backing memory initialization state
- input record source
- root bindings / descriptor heaps
```

첫 backing-memory 사용 시 initialization semantics가 필요하며, 같은 backing memory를 서로 병렬 실행될 수 있는 device queue에서 동시에 공유하는 것은 허용되지 않는다. 따라서 backing memory도 render-graph resource lifetime과 queue ownership 관점에서 관리해야 한다.

### Persistent threads vs Work Graphs를 선택하는 기준

Persistent threads가 유리한 경우:

- scheduling policy를 매우 세밀하게 직접 제어해야 함
- CUDA 중심의 compute pipeline
- task stealing / priority / custom termination 같은 비표준 scheduler가 중요함
- hardware/API portability보다 절대적인 algorithm control이 중요함

Work Graphs가 유리한 경우:

- D3D12 renderer에서 GPU-generated work를 표준 abstraction으로 표현하고 싶음
- producer-consumer dataflow가 명확함
- work가 runtime data에 따라 branch/expand됨
- uber shader나 여러 ExecuteIndirect chain을 줄이고 싶음
- node specialization으로 register pressure와 divergence를 줄일 여지가 큼

둘은 완전한 대체 관계라기보다 **software scheduler와 runtime scheduler의 trade-off**다.

### Profiling에서 볼 지표

Work Graphs:

- node별 GPU time
- node invocation / record count
- graph expansion ratio
- record size
- backing memory requirement
- UAV traffic
- average active threads per warp
- specialized node의 register usage
- tiny-node 비율
- scheduler overhead가 compute보다 커지는 지점

Persistent threads:

- queue depth
- enqueue/dequeue contention
- atomic throughput
- worker idle ratio
- active blocks / occupancy
- task latency distribution
- termination tail
- queue overflow
- payload cache hit rate

NVIDIA의 D3D12 Work Graph deferred-shading case study도 graph execution 자체에 overhead가 있으므로, node shader가 너무 작은 일을 하면 scheduling 비용이 지배적일 수 있다고 지적한다. 즉 Work Graphs도 node를 무한히 잘게 쪼개는 것이 정답은 아니다.

## 5. 내 관심 분야와 연결

### ReSTIR / Path Tracing

ReSTIR PT의 temporal/spatial reuse, shift mapping, replay, reconnection, visibility는 비용이 크게 다른 irregular work를 만든다. Work Graphs는 이런 work를 explicit CPU dispatch chain보다 더 자연스럽게 dataflow graph로 표현할 가능성이 있다.

특히 어제 다룬 shift-class binning은 Work Graphs에서 **node specialization**으로 연결할 수 있다.

```text
explicit queue binning
        ↓
record-driven specialized node
```

### CFD / Streamline

GPU streamline integration에서도 particle마다 다음 상태가 달라진다.

```text
continue integration
adaptive-step retry
block migration
boundary handling
terminate
```

현재 particle state가 다음 work를 동적으로 생성하므로 Work Graph/persistent scheduler와 매우 유사한 형태다.

### Level-set / Sparse Voxel

Sparse volume에서는 active brick이 refinement, neighbor update, surface extraction, compaction 같은 후속 task를 만들 수 있다.

```text
Active Brick
 -> refine
 -> update neighbors
 -> extract surface
 -> retire
```

이 구조는 CPU가 모든 단계의 worst-case dispatch를 미리 넣는 것보다 device-generated work가 잘 맞는 대표적인 irregular compute problem이다.

### Game Engine / GPU-Driven Rendering

현대 GPU-driven renderer의 큰 흐름은 다음처럼 볼 수 있다.

```text
CPU command generation 감소
        ↓
GPU culling / classification
        ↓
indirect execution
        ↓
wavefront queues
        ↓
device-generated work
        ↓
work graphs / runtime scheduling
```

즉 Work Graphs는 단일 API feature라기보다 **renderer control plane을 CPU에서 GPU 쪽으로 이동시키는 장기적인 architecture 변화**로 이해하는 것이 중요하다.

### WebGPU / Vulkan 관점

Work Graphs는 현재 D3D12의 명시적인 programming model이지만, 핵심 사고방식은 API-independent하다.

- data-dependent work generation
- producer-consumer scheduling
- compact task records
- persistent payload pool
- specialized execution classes

WebGPU나 Vulkan에서 동일 abstraction이 직접 제공되지 않더라도 indirect dispatch, compaction, persistent queue, multi-pass compute를 설계할 때 같은 trade-off를 적용할 수 있다.

## 6. 머릿속에 남길 질문 3개

1. **Explicit wavefront queue를 D3D12 Work Graph node로 바꿨을 때, 줄어드는 UAV queue traffic과 늘어나는 graph scheduling/backing-memory 비용을 어떤 profiler 지표로 비교해야 할까?**
2. **ReSTIR PT처럼 task cost 분산이 큰 workload에서 node를 어느 granularity까지 분리해야 specialization 이득이 scheduling overhead보다 커질까?**
3. **Persistent threads가 제공하는 custom priority와 work-stealing 자유도를 포기하고 runtime-managed Work Graphs를 선택해도 되는 경계는 어디일까?**

## 7. graphics engineer 면접 질문 1개와 답변

### 질문

**GPU-driven renderer에서 persistent threads와 D3D12 Work Graphs는 어떤 문제를 해결하며, 둘의 가장 중요한 차이는 무엇인가요?**

### 답변

둘 다 CPU가 모든 fine-grained dispatch를 직접 제출해야 하는 구조를 줄이고, GPU가 데이터에 따라 다음 work를 생성하도록 만드는 방법이다.

Persistent threads는 일정한 GPU worker를 장시간 resident시키고 application-owned queue에서 task를 꺼내 처리한다. 따라서 scheduling policy, priority, task stealing, termination 등을 매우 세밀하게 제어할 수 있지만 queue synchronization, atomics, fairness, occupancy, overflow와 termination detection을 application이 직접 관리해야 한다.

D3D12 Work Graphs는 shader node가 다른 node로 record를 출력하는 producer-consumer model을 제공한다. runtime이 record storage와 scheduling의 상당 부분을 관리하며, node마다 specialized shader를 사용할 수 있어 uber shader의 divergence와 register pressure를 줄일 가능성이 있다. 대신 graph execution overhead와 backing-memory requirement가 있으며, scheduling detail에 대한 직접 제어는 persistent scheduler보다 적다.

핵심 차이는 다음 한 문장으로 정리할 수 있다.

> **Persistent threads는 application-owned GPU scheduler이고, Work Graphs는 API/runtime-owned GPU work scheduling abstraction이다.**

따라서 선택 기준은 단순 성능 비교가 아니라 scheduling control, portability, memory ownership, debugging complexity, workload granularity를 함께 봐야 한다.

## 8. 포트폴리오 / 커리어 연결

Graphics engineer 포트폴리오에서 이 주제를 다룬다면 단순히 “Work Graphs API를 사용했다”보다 **동일 irregular workload를 세 가지 execution architecture로 비교한 사고 과정**이 더 가치 있다.

```text
1. CPU-dispatched wavefront
2. Persistent-thread scheduler
3. Work Graph-based producer-consumer pipeline
```

비교 기준은 다음처럼 잡을 수 있다.

- GPU time
- CPU submission cost
- register pressure
- active lanes per warp
- global-memory queue traffic
- backing memory
- node/task granularity
- scheduler overhead
- worst-case expansion
- debugging complexity

이런 비교는 “API를 사용할 줄 안다”보다 한 단계 높은 역량을 보여준다.

> **Workload의 불규칙성을 보고 execution architecture와 scheduler ownership을 선택할 수 있는가?**

이는 rendering engineer, engine programmer, GPU compute engineer 면접에서 모두 강한 주제다. 특히 ray tracing, GPU-driven rendering, Nanite-like pipelines, particles, simulation, scientific visualization처럼 heterogeneous work가 많은 영역과 직접 연결된다.

## 9. 내일 이어서 볼 개념

**Shader Execution Reordering and Dynamic Coherence Recovery: Ray/Path Reordering Beyond Explicit Binning**

오늘까지는 queue, node, persistent worker처럼 **software-visible scheduling 구조**를 다뤘다. 내일은 한 단계 더 내려가, ray/path가 이미 shader 실행 중인 상황에서 hardware/runtime가 execution을 재정렬해 coherence를 회복하는 **Shader Execution Reordering (SER)** 관점을 본다.

핵심 연결 질문은 다음이다.

> **Explicit queue binning이나 Work Graph node specialization으로도 잡히지 않는 runtime divergence를 hardware-assisted reordering은 어디까지 줄일 수 있을까?**

## 10. 참고 키워드

- **D3D12 Work Graphs**
- **GPU Based Work Creation**
- **Persistent Threads / Persistent Kernel**
- **Device-Side Scheduling**
- **Producer-Consumer Dataflow**
- **Node Record**
- **Broadcasting Launch**
- **Thread Launch**
- **Coalescing Launch**
- **Backing Memory**
- **Work Expansion**
- **ExecuteIndirect**
- **GPU-Driven Rendering**
- **Wavefront Rendering**
- **Uber Shader**
- **Register Pressure**
- **SIMT Divergence**
- **Work Queue**
- **Persistent GPU Pool**
- **UAV Producer-Consumer Visibility**
- **HLSL Shader Model 6.8**
- **D3D12_WORK_GRAPHS_TIER_1_0**
- **Shader Execution Reordering (SER)**

공식 참고 자료:

- Microsoft DirectX-Specs — D3D12 Work Graphs, v1.012 (2026-02-04): https://microsoft.github.io/DirectX-Specs/d3d/WorkGraphs.html
- Microsoft DirectX Developer Blog — D3D12 Work Graphs: https://devblogs.microsoft.com/directx/d3d12-work-graphs/
- NVIDIA Technical Blog — Advancing GPU-Driven Rendering with Work Graphs in Direct3D 12: https://developer.nvidia.com/blog/advancing-gpu-driven-rendering-with-work-graphs-in-direct3d-12/
- NVIDIA Technical Blog — Work Graphs in Direct3D 12: A Case Study of Deferred Shading: https://developer.nvidia.com/blog/work-graphs-in-direct3d-12-a-case-study-of-deferred-shading/
