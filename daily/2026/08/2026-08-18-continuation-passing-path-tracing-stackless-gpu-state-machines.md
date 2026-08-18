---
title: "Continuation-Passing Path Tracing and Stackless GPU State Machines: Explicit State Serialization vs Driver-Managed Continuations"
date: "2026-08-18"
category: Graphics
tags: [GPU, Path Tracing, Ray Tracing, Continuation, Continuation Passing Style, CPS, State Machine, Wavefront, Megakernel, SER, DXR, Vulkan, Shader, Memory Layout, Register Pressure, C++]
level: intermediate
---

# [Daily Graphics Growth] 2026-08-18 - Continuation-Passing Path Tracing and Stackless GPU State Machines: Explicit State Serialization vs Driver-Managed Continuations

## 1. 오늘의 개념

어제는 **reorder point를 가로질러 살아 있는 상태(live state)**를 줄이는 문제를 다뤘다. 오늘은 그 상태가 실제로 **어디에 존재하고, 누가 소유하며, 어떤 형태로 다음 실행 단계에 전달되는가**를 한 단계 더 구조적으로 본다.

핵심 개념은 **Continuation-Passing Style (CPS)**과 **stackless GPU state machine**이다.

CPU 코드에서는 함수 호출이 자연스럽게 call stack을 만든다.

```text
PathTrace()
  -> TraceRay()
      -> ClosestHit()
          -> SampleBSDF()
              -> TraceRay()
                  ...
```

논리적으로는 매우 직관적이다. 하지만 GPU ray tracing에서는 이 호출 구조를 그대로 하드웨어 call stack으로 유지한다고 생각하면 실제 실행 모델을 잘못 이해하기 쉽다. `TraceRay`, hit shader transition, callable shader, SER reorder point 같은 경계에서는 shader invocation이 중단되고 다른 work가 실행될 수 있다. 이후 다시 이어지려면 **"어디까지 실행했고, 돌아왔을 때 무엇을 해야 하는가"**라는 continuation 정보가 필요하다.

Continuation을 가장 단순하게 정의하면 다음과 같다.

> **Continuation = 현재 계산이 중단된 뒤, 나중에 다시 실행을 이어가기 위해 필요한 다음 실행 위치와 최소 상태.**

이를 GPU renderer 관점에서 보면 두 가지 큰 설계가 있다.

### A. Driver-managed continuation

DXR/Vulkan ray tracing pipeline의 `TraceRay()`처럼 application은 함수 호출처럼 코드를 작성하고, shader compiler + driver + runtime이 내부적으로 필요한 state를 보존한다.

```text
RayGen
  -> TraceRay
       driver/runtime manages suspended state
  -> ClosestHit / Miss
  -> resume RayGen
```

장점은 programming model이 자연스럽다는 것이다. 반면 application이 continuation storage의 정확한 크기, spill 위치, migration cost를 직접 통제하기 어렵다.

### B. Explicit serialized continuation

Wavefront renderer나 persistent-thread renderer에서는 application이 path state를 직접 global memory에 저장하고, 다음 단계가 `pathIndex` 혹은 compact token을 받아 실행을 이어간다.

```text
Queue entry
  = { pathIndex, nextStage }

PathState[pathIndex]
  = throughput, radiance, rng, ray, depth, material context ...
```

이 경우 call stack은 사실상 사라지고 다음 상태를 명시하는 **stackless state machine**이 된다.

```text
TRACE -> SHADE -> SHADOW -> RESUME -> TRACE -> ...
```

둘 중 하나가 항상 더 좋은 것은 아니다. 중요한 것은 renderer가 결국 같은 문제를 서로 다른 위치에서 해결한다는 점이다.

> **Driver-managed pipeline은 continuation serialization을 runtime 내부로 숨기고, wavefront pipeline은 continuation serialization을 application-visible memory layout으로 끌어낸다.**

## 2. 한 줄 핵심

> **GPU path tracer는 논리적으로 재귀적인 알고리즘이어도 실행 관점에서는 “다음 stage + 최소 path state”를 전달하는 continuation state machine으로 볼 수 있으며, megakernel/SER은 이 상태를 driver가 관리하고 wavefront는 application이 명시적으로 직렬화한다.**

## 3. 왜 중요한가

### 재귀적 알고리즘과 실제 GPU 실행 구조는 다르다

Path tracing은 수식과 pseudocode만 보면 자연스럽게 recursive하다.

\[
L_o(x, \omega_o)
= L_e(x, \omega_o)
+ \int_\Omega f_r(x,\omega_i,\omega_o)L_i(x,\omega_i)(n\cdot\omega_i)d\omega_i
\]

한 bounce에서 새로운 방향을 샘플하고 다시 radiance를 평가하므로 논리적으로는 다음과 같다.

```text
Li(ray)
  -> intersect
  -> shade
  -> Li(nextRay)
```

하지만 수천~수백만 path를 동시에 처리하는 GPU에서 각 path가 서로 다른 bounce depth, material, visibility path를 갖기 시작하면 전통적인 CPU식 stack recursion은 scheduling과 memory 관점에서 비효율적이다.

그래서 modern GPU path tracer는 대체로 다음 두 극단 사이에 위치한다.

```text
Megakernel / recursive-looking shader
<---------------------------------->
Wavefront / explicit state machine
```

SER, HitObject, Work Graphs, persistent threads 같은 기술은 이 두 극단 사이에서 **coherence와 state ownership을 조정하는 도구**로 볼 수 있다.

### Continuation은 단순 return address가 아니다

CPU의 continuation을 단순 return address로 생각하면 GPU에서는 너무 좁은 정의가 된다.

한 path가 ray traversal 이전에 다음 상태를 가지고 있다고 하자.

```text
throughput
accumulated radiance
RNG state
bounce index
medium / IOR state
MIS bookkeeping
reservoir or light sample state
ray payload
```

`TraceRay()` 이후에도 이 값이 필요하면 runtime은 그 의미를 보존해야 한다. 즉 continuation은 사실상 다음과 같은 tuple로 볼 수 있다.

\[
K = (PC_{resume}, S_{live})
\]

- \(PC_{resume}\): resume point / next program location
- \(S_{live}\): resume 시 필요한 live state

GPU에서는 이 상태가 register에 계속 남아 있을 수도 있고, hidden stack이나 local/global memory에 spill될 수도 있으며, SER에서는 다른 wave/processor context로 이동할 가능성도 있다.

따라서 continuation cost를 개념적으로 다음처럼 볼 수 있다.

\[
C_{continuation}
\approx C_{capture}(S_{live})
+ C_{store/load}(S_{live})
+ C_{schedule}
+ C_{resume}
\]

어제 다룬 **live-state minimization**이 곧 continuation 최적화인 이유다.

### Wavefront renderer는 continuation을 데이터 구조로 바꾼다

Explicit wavefront renderer에서는 continuation이 숨겨진 runtime state가 아니라 직접 볼 수 있는 데이터가 된다.

예를 들어 path state가 다음처럼 분리될 수 있다.

```text
PathCoreSoA
  throughput[]
  radiance[]
  rng[]
  depth[]
  flags[]

RaySoA
  origin[]
  direction[]
  tMinMax[]

QueueEntry
  pathIndex
  continuationTag
```

이 구조에서는 다음 실행 위치가 call stack이 아니라 `continuationTag`가 된다.

```text
CONT_TRACE
CONT_SHADE
CONT_SHADOW
CONT_RESUME_AFTER_NEE
CONT_TERMINATE
```

이것이 **stackless state machine**이다.

흥미로운 점은 이 방식이 register/live-state 문제를 없애는 것이 아니라 위치를 바꾼다는 것이다.

```text
Driver-managed continuation
  낮은 explicit queue traffic
  높은 hidden live-state / continuation pressure 가능

Explicit continuation
  낮은 hidden call-state pressure
  높은 global-memory state traffic 가능
```

### SER은 continuation 자체를 제거하지 않는다

Shader Execution Reordering(SER)은 invocation을 더 coherent하게 재배치하지만, reorder point 이후 실행을 이어가기 위한 state는 여전히 필요하다.

현재 DXR 1.2 specification은 `TraceRay`, `HitObject::TraceRay`, `HitObject::Invoke`, `MaybeReorderThread` 등을 **reorder point**로 정의하며 runtime이 thread context를 다른 wave나 processor로 이동시킬 수 있음을 명시한다. 또한 Microsoft spec은 `MaybeReorderThread`를 가로지르는 live state가 일부 architecture에서 더 비쌀 수 있으므로 이를 tooling/profiling 대상으로 봐야 한다고 설명한다.

즉 SER을 continuation 관점에서 보면 다음과 같다.

```text
Continuation capture
      +
Dynamic regrouping
      +
Continuation resume
```

coherence가 좋아져도 continuation state가 너무 크면 state movement 비용이 커진다.

NVIDIA의 실제 path tracing 최적화 사례에서도 SER 적용 후 ray-tracing live-state spill로 L2 traffic이 증가했고, live state를 줄인 후 SER 효과가 더 커졌다. 이 사례는 **reordering과 continuation serialization을 따로 볼 수 없다는 점**을 잘 보여준다.

## 4. 구현 관점

### 4.1 Driver-managed continuation: 코드 구조는 단순하지만 비용은 간접적이다

전통적인 ray tracing pipeline에서는 다음처럼 작성할 수 있다.

```text
RayGen
  prepare ray
  TraceRay
  consume payload
  continue loop
```

application 코드에서는 `TraceRay`가 함수 호출처럼 보인다. 그러나 runtime은 내부적으로 caller state를 보존하고 hit/miss shader를 실행한 뒤 caller를 resume해야 한다.

이 모델의 장점은 다음과 같다.

- shader code가 algorithm structure와 가깝다.
- path-local 값이 자연스럽게 local variable로 표현된다.
- global path buffer와 queue traffic을 줄일 수 있다.
- compiler가 live range와 state placement를 최적화할 여지가 있다.

반대로 단점은 다음과 같다.

- 어떤 값이 RT live state가 되는지 source만 보고 판단하기 어렵다.
- driver/compiler backend에 따라 spill behavior가 달라질 수 있다.
- shader permutation 증가가 code size와 register pressure를 키울 수 있다.
- 여러 continuation boundary를 가로지르는 large local state가 숨은 비용이 된다.

즉 C++ 관점에서 API는 단순하지만, GPU execution contract는 오히려 더 implicit하다.

### 4.2 Explicit continuation: queue + state buffer가 call stack 역할을 한다

Wavefront path tracer에서는 각 stage가 work item을 소비하고 다음 queue를 생성한다.

개념적으로 다음 pipeline을 생각할 수 있다.

```text
GeneratePrimary
   ↓
TraceQueue
   ↓
ShadeQueue
   ├─> ShadowQueue
   └─> NextBounceQueue
          ↓
        TraceQueue
```

각 queue entry가 전체 path state를 복사하면 bandwidth가 폭증한다. 그래서 일반적으로 queue에는 **indirection token**만 두고 실제 state는 persistent buffer에 둔다.

```text
struct QueueEntry {
    uint pathIndex;
};
```

logical continuation은 다음처럼 표현된다.

```text
pathIndex -> persistent state
queue identity -> continuation PC
```

즉 queue 자체가 다음 프로그램 위치를 암시한다.

```text
ShadeQueue에 들어있다
= 다음 continuation은 Shade kernel이다
```

이 때문에 queue-per-stage design은 `continuationTag`를 별도로 저장하지 않아도 된다.

반대로 persistent-thread 또는 unified state-machine kernel에서는 하나의 queue에서 여러 continuation type을 다룰 수 있으므로 tag가 필요하다.

```text
{ pathIndex, stageTag }
```

### 4.3 Continuation state를 Hot / Warm / Cold로 나누면 memory layout이 명확해진다

모든 path state가 매 stage에서 필요하지는 않다.

이를 접근 빈도로 나누면 다음처럼 볼 수 있다.

#### Hot state
거의 모든 bounce에서 사용한다.

```text
throughput
rng
ray origin / direction
depth
flags
```

#### Warm state
특정 shading stage에서만 자주 사용한다.

```text
surface normal
material ID
BSDF lobe state
light sample state
PDF / MIS state
```

#### Cold state
드물게 필요하다.

```text
medium stack
debug/profiling fields
long-path metadata
special material payload
```

wavefront에서는 이를 서로 다른 SoA buffer로 분리하면 hot stage가 cold state까지 읽는 것을 피할 수 있다.

```text
PathCore[]
SurfaceState[]
MediumState[]
DebugState[]
```

반대로 megakernel에서는 local variable scope와 live range를 통해 같은 목적을 달성하려 한다.

결국 두 구조 모두 같은 최적화 원칙을 가진다.

> **다음 continuation에서 반드시 필요한 데이터만 가까이 유지하고, 나머지는 늦게 생성하거나 간접 참조한다.**

### 4.4 Stackless는 “stack이 전혀 없다”는 뜻보다 “control stack을 application state로 평탄화한다”에 가깝다

Path tracing에서 stack이 필요한 대표 사례는 두 가지다.

1. **control stack** — 어디로 return할지
2. **semantic stack** — medium/IOR nesting, nested material state 등

Stackless state machine이 제거하는 것은 주로 첫 번째다.

```text
recursive control flow
-> finite state machine / continuation token
```

하지만 glass inside glass처럼 nested dielectric을 정확히 추적한다면 medium/IOR stack 같은 semantic state는 여전히 필요할 수 있다.

따라서 다음을 구분해야 한다.

```text
call stack elimination
!=
all stack-like data elimination
```

면접에서도 이 차이를 말할 수 있으면 좋다. “wavefront path tracing은 stackless니까 stack이 필요 없다”는 답은 너무 거칠다.

### 4.5 Explicit state serialization의 비용은 bandwidth + latency + compaction이다

Wavefront 구조는 continuation을 명시적으로 제어할 수 있지만 그 대가는 memory system에 나타난다.

각 bounce에서 path state 일부를 global memory에 기록하고 다음 kernel이 다시 읽는다면 대략적인 traffic은 다음처럼 생각할 수 있다.

\[
B_{wavefront}
\approx N_{active}\cdot B_{state}\cdot N_{stage\ transitions}
\]

여기서

- \(N_{active}\): active path 수
- \(B_{state}\): stage 간 이동하는 state byte 수
- \(N_{stage\ transitions}\): continuation 전환 횟수

따라서 explicit state machine에서 중요한 것은 단순히 struct를 작은 자료형으로 만드는 것만이 아니다.

```text
queue entry 최소화
hot/cold state 분리
SoA coalescing
inactive path compaction
stage fusion
recomputation
state lifetime shortening
```

이 모든 것이 continuation serialization 비용을 낮추는 방법이다.

### 4.6 Megakernel과 wavefront 사이에는 넓은 hybrid 영역이 있다

실제 engine은 둘 중 하나만 선택하지 않는다.

예를 들어 다음 hybrid가 가능하다.

```text
Megakernel inside one bounce
+ explicit queue between expensive stages
```

또는

```text
Persistent threads
+ software work queue
+ inline ray query
```

또는

```text
SER-enabled RayGen loop
+ HitObject
+ explicit external reservoir/path-state buffers
```

또는

```text
Work Graph node scheduling
+ compact records
+ persistent path-state pool
```

중요한 것은 이름이 아니라 **continuation ownership boundary**다.

```text
어디까지 compiler/driver에게 맡길 것인가?
어디부터 application이 state를 serialize할 것인가?
```

이 질문을 기준으로 architecture를 보면 서로 다른 renderer를 비교하기 쉬워진다.

### 4.7 C++ render architecture에서는 state ownership이 API boundary가 된다

C++ renderer에서 explicit continuation을 도입하면 shader 문제가 곧 resource-lifetime 문제가 된다.

예를 들어 다음 개념들이 생긴다.

```text
PathStateBuffer
TraceQueue
ShadeQueue
ShadowQueue
QueueCounter
DispatchArgs
CompactionScratch
```

render graph에서는 각 resource에 대해 다음을 명확히 해야 한다.

```text
producer pass
consumer pass
UAV/SRV transition
queue counter reset
frame lifetime
capacity / overflow policy
history persistence 여부
```

특히 persistent path buffer가 여러 dispatch를 살아남는다면, 단순 transient scratch가 아니라 **execution state store**가 된다.

이때 잘못된 barrier나 queue counter race는 단순 rendering artifact가 아니라 continuation corruption으로 이어진다.

```text
잘못된 pathIndex
-> 다른 path state를 resume
-> estimator state 오염
-> NaN / firefly / random crash / nondeterminism
```

그래서 explicit continuation architecture는 debugging 측면에서도 `pathIndex`, `stage`, `bounce`, `generation` 같은 metadata를 추적하기 쉽게 만드는 것이 중요하다.

### 4.8 SER / Vulkan invocation reorder는 explicit wavefront를 완전히 대체하지 않는다

현재 DXR 1.2의 SER과 Vulkan의 `VK_EXT_ray_tracing_invocation_reorder`는 ray-generation invocation을 runtime이 재배치하여 control/data coherence를 개선한다. Khronos의 현재 sample 역시 hit object로 traversal과 shader invocation을 분리하고 `reorderThreadEXT()`를 통해 invocation을 재그룹화하는 모델을 보여준다.

하지만 SER은 다음 문제를 자동으로 해결하지 않는다.

```text
path-state lifetime
very long path continuation
queue capacity
specialized stage scheduling
shadow-ray batching
reservoir/subpath pool ownership
cross-frame state persistence
```

따라서 SER은 **driver-managed continuation의 scheduling 품질을 높이는 기술**이지, 모든 explicit state-machine architecture를 제거하는 기술이라고 보는 것은 과하다.

오히려 modern renderer에서는 다음처럼 같이 존재할 수 있다.

```text
coarse scheduling: wavefront / work graph
fine execution coherence: SER
persistent estimator state: explicit GPU buffers
```

## 5. 내 관심 분야와 연결

### 실시간 렌더링 / 게임 엔진

게임 엔진에서 full path tracing, hybrid ray tracing, GI, reflection이 복잡해질수록 한 shader가 모든 material과 ray type을 처리하는 megakernel은 register pressure와 divergence 문제를 키울 수 있다.

반대로 pass를 지나치게 잘게 나누면 queue traffic, dispatch cost, synchronization이 커진다.

따라서 실제 engine architecture에서 중요한 능력은 “megakernel이냐 wavefront냐”를 종교적으로 고르는 것이 아니라 다음을 profiler 기반으로 판단하는 것이다.

```text
어떤 state가 boundary를 넘어가는가?
어떤 stage가 divergence를 만드는가?
어떤 state가 global memory로 내릴 가치가 있는가?
어디까지 stage fusion이 가능한가?
```

### GPU compute / simulation

Continuation-passing 구조는 rendering에만 해당하지 않는다.

불규칙한 simulation/geometry pipeline에서도 다음 패턴이 반복된다.

```text
candidate generation
classification
specialized processing
retry / refinement
completion
```

예를 들어 sparse voxel processing, adaptive meshing, level-set narrow-band update, marching-cubes active-cell pipeline에서도 한 thread가 한 번에 모든 단계를 수행하도록 만드는 대신 work queue와 compact state를 이용해 state machine으로 구성할 수 있다.

그래서 path tracing의 continuation architecture는 **irregular GPU compute architecture**를 이해하는 좋은 교재다.

### WebGPU / Metal / Vulkan / DirectX

API별로 high-level ray tracing 기능 차이는 있지만, continuation을 보는 관점은 공통적이다.

- **DXR / Vulkan RT pipeline**: runtime-managed shader-stage continuation
- **Ray Query / Inline Ray Tracing**: traversal은 inline이므로 control flow ownership이 application shader 쪽에 더 많이 남는다.
- **Compute-based wavefront**: state와 scheduling을 거의 전부 application이 관리한다.
- **Work Graphs**: node scheduling은 runtime이 맡되 application이 node record와 persistent payload architecture를 설계한다.

즉 API를 외우기보다 **control ownership / state ownership / scheduling ownership** 세 축으로 비교하는 편이 훨씬 오래 남는다.

## 6. 머릿속에 남길 질문 3개

1. **Megakernel에서 `TraceRay()` 전후로 살아 있는 local variable과, wavefront renderer의 `PathStateBuffer` field는 본질적으로 어떤 동일한 continuation 정보를 다른 위치에 저장한 것인가?**
2. **Explicit wavefront 구조가 register pressure를 줄이더라도 global-memory traffic 때문에 더 느려질 수 있는 경계는 어떤 profiler 지표로 찾을 수 있는가?**
3. **SER, Work Graphs, persistent threads를 함께 사용하는 renderer에서 scheduling ownership과 continuation-state ownership을 각각 어느 layer에 두는 것이 가장 자연스러운가?**

## 7. graphics engineer 면접 질문 1개와 답변

### 질문

**Megakernel path tracer와 wavefront path tracer를 continuation 관점에서 비교하고, 각각의 주요 GPU 성능 trade-off를 설명해보세요.**

### 답변

Megakernel path tracer는 path의 여러 bounce와 shading 단계를 하나의 긴 shader control flow 안에 유지한다. `TraceRay()` 같은 ray-tracing boundary가 있어도 application 코드에서는 local variable과 함수 호출 형태가 유지되며, compiler/driver/runtime이 suspended invocation의 continuation state를 내부적으로 관리한다. 이 구조는 global work queue와 path-state buffer traffic이 적다는 장점이 있지만, large live range, register pressure, RT live-state spill, material divergence가 커질 수 있다.

Wavefront path tracer는 control flow를 여러 stage로 분리하고 path state를 global memory에 명시적으로 직렬화한다. queue의 위치나 stage tag가 continuation의 다음 program counter 역할을 하고, `pathIndex`가 persistent state를 가리킨다. 이 방식은 stage별로 coherent한 work를 묶고 kernel별 register footprint를 줄이기 쉽지만, queue append/compaction, global-memory read/write, dispatch synchronization 비용이 증가한다.

따라서 핵심 trade-off는 단순히 divergence 대 dispatch cost가 아니라 다음처럼 정리할 수 있다.

```text
Megakernel:
less explicit state traffic
vs
more hidden continuation/live-state pressure

Wavefront:
more explicit state traffic
vs
better control over specialization, live range, and scheduling
```

SER은 megakernel 쪽의 실행 coherence를 개선할 수 있지만 continuation state 자체를 제거하지는 않으며, Work Graphs나 persistent threads는 explicit scheduling의 일부 overhead를 줄일 수 있다. 최종 선택은 register/live-state bytes, L1/L2 traffic, active lanes, queue occupancy, compaction cost, dispatch overhead, GPU time을 함께 프로파일링해서 결정해야 한다.

## 8. 포트폴리오 / 커리어 연결

이 개념을 포트폴리오에서 보여줄 때 가장 가치 있는 포인트는 단순히 “path tracer를 만들었다”가 아니다.

더 강한 설명은 다음과 같은 architecture-level reasoning이다.

```text
Problem
- secondary ray divergence 증가
- RayGen live state 증가
- material permutation 확대

Architecture alternatives
- megakernel 유지 + SER
- material/stage queue 기반 wavefront
- persistent-thread scheduler
- hybrid stage split

Measurements
- registers/thread
- RT live-state bytes
- local-memory spill
- active lanes
- L2 traffic
- queue occupancy
- kernel/dispatch time

Decision
- 어느 continuation boundary를 explicit하게 만들었는가
- 어떤 state를 persistent buffer로 옮겼는가
- 무엇을 recompute하고 무엇을 carry했는가
```

Graphics engineer 면접에서도 이 수준의 설명은 강하다. API 사용법보다 **GPU execution model → memory layout → scheduler → profiler → architecture decision**을 연결해서 설명할 수 있기 때문이다.

특히 C++/Vulkan/DXR 기반 포트폴리오라면 renderer architecture diagram에서 다음을 표시하면 설계 의도가 분명해진다.

```text
CPU render graph
     ↓
GPU scheduling layer
     ↓
continuation boundary
     ↓
path-state ownership
     ↓
ray traversal / shading
```

이 사고방식은 game engine rendering뿐 아니라 scientific visualization, sparse GPU geometry processing, simulation pipeline에도 그대로 확장할 수 있다.

## 9. 내일 이어서 볼 개념

**Continuation State Compression and Path-State Virtualization: Hot/Cold Splitting, Tokenization, and GPU Working-Set Control**

오늘은 continuation state를 누가 관리하는지를 보았다. 다음에는 explicit/implicit continuation 모두에서 실제 working set을 줄이는 방향으로 들어간다.

핵심 질문은 다음이다.

> **모든 path state를 동일한 비용으로 유지하지 않고, hot continuation state만 빠르게 유지하면서 cold state는 token/indirection으로 가상화할 수 있는가?**

다음 흐름에서는

- Hot/Cold path-state splitting
- compact continuation token
- pointer 대신 stable index/handle
- path-state virtualization
- cache working set
- state compaction / quantization
- long-path spill policy
- GPU memory residency

를 연결해 본다.

## 10. 참고 키워드

### 핵심 용어

- Continuation
- Continuation-Passing Style (CPS)
- Stackless State Machine
- Driver-Managed Continuation
- Explicit State Serialization
- Wavefront Path Tracing
- Megakernel Path Tracing
- Persistent Threads
- Path-State Buffer
- Work Queue
- Resume Point
- Continuation Token
- RT Live State
- Register Pressure
- State Spill / Restore
- Shader Execution Reordering (SER)
- HitObject
- Ray Query / Inline Ray Tracing
- GPU Work Graphs
- SoA / AoSoA
- Queue Compaction
- State Lifetime
- GPU Working Set

### 표준·문서에서 이어서 볼 키워드

- DXR 1.2 `MaybeReorderThread`
- DXR reorder points
- DXR `HitObject::TraceRay` / `HitObject::Invoke`
- Vulkan `VK_EXT_ray_tracing_invocation_reorder`
- GLSL `GL_EXT_shader_invocation_reorder`
- ray-tracing live-state spill
- wavefront vs megakernel path tracing

### 참고 자료

- Microsoft, **DirectX Raytracing (DXR) Functional Spec v1.45 (2026-07-10)** — Shader Execution Reordering, reorder points, live-state considerations: https://microsoft.github.io/DirectX-Specs/d3d/Raytracing.html
- Khronos, **VK_EXT_ray_tracing_invocation_reorder** / Vulkan SER sample — hit object, invocation reorder, live-state guidance: https://docs.vulkan.org/samples/latest/samples/extensions/ray_tracing_invocation_reorder/README.html
- NVIDIA Technical Blog, **Path Tracing Optimization in Indiana Jones: Shader Execution Reordering and Live State Reductions** — RT live-state spill과 SER의 실제 성능 관계: https://developer.nvidia.com/blog/path-tracing-optimization-in-indiana-jones-shader-execution-reordering-and-live-state-reductions
