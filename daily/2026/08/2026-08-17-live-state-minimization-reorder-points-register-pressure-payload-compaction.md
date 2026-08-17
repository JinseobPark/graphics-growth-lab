---
title: "Live-State Minimization Across Reorder Points: Register Pressure, Continuations, and Payload Compaction"
date: "2026-08-17"
category: Graphics
tags: [GPU, Ray Tracing, Shader Execution Reordering, SER, Live State, Register Pressure, Continuation, Payload, FP16, Occupancy, Path Tracing, DXR, Vulkan, C++]
level: intermediate
---

# [Daily Graphics Growth] 2026-08-17 - Live-State Minimization Across Reorder Points: Register Pressure, Continuations, and Payload Compaction

## 1. 오늘의 개념

어제는 **Shader Execution Reordering (SER)**을 통해 ray/path invocation의 실행 순서를 재배치하여 control-flow와 memory coherence를 회복하는 구조를 보았다. 오늘은 그 다음 질문으로 들어간다.

> **thread를 더 잘 묶는 것만으로 충분한가? 아니면 reorder point를 넘어 이동해야 하는 thread의 상태 자체를 작게 만들어야 하는가?**

Ray tracing이나 path tracing shader에서 `TraceRay`, `HitObject::TraceRay`, `MaybeReorderThread`, callable shader transition 같은 지점은 단순한 함수 호출이 아니다. GPU runtime 또는 driver가 invocation의 실행 위치를 바꾸거나, shader stage 사이를 이동하거나, 실행을 잠시 중단했다가 다시 이어야 하는 **continuation boundary**로 볼 수 있다.

이 경계에서 다음 값들이 이후에도 필요하다면 모두 **live state**가 된다.

```text
path throughput
accumulated radiance
RNG state
bounce index
ray origin / direction
material context
BSDF temporary data
reservoir metadata
surface stack / IOR stack
payload fields
```

컴파일러와 driver는 이 상태를 register에 계속 둘 수 없거나 reorder/trace boundary를 넘어 보존해야 할 경우 memory에 spill한 뒤 다시 restore할 수 있다. 특히 SER에서는 invocation이 다른 wave나 processor로 이동할 수 있으므로, live state가 클수록 coherence를 얻기 위해 지불하는 **state migration cost**도 커질 수 있다.

따라서 오늘의 주제는 SER 자체가 아니라 그 아래에 있는 더 일반적인 GPU 설계 원칙이다.

> **reorder point, traversal boundary, continuation boundary를 가로질러 살아 있는 상태를 최소화하고, 정말 필요한 상태만 compact representation으로 전달한다.**

이 관점은 DXR/OptiX/Vulkan ray tracing뿐 아니라 persistent-thread renderer, wavefront path tracer, coroutine-like GPU scheduler, Work Graphs에도 그대로 이어진다.

## 2. 한 줄 핵심

> **GPU ray/path workload의 성능은 “몇 개의 register를 쓰는가”보다 “비싼 execution boundary를 가로질러 몇 byte의 상태를 살아 있게 만드는가”에 크게 좌우되며, live range 축소·recomputation·payload compaction·precision demotion은 divergence 최적화만큼 중요한 설계 도구다.**

## 3. 왜 중요한가

### Register pressure와 RT live state는 비슷하지만 같은 개념이 아니다

일반 compute shader에서 **register pressure**는 thread가 동시에 필요로 하는 register 수가 많아지는 문제다. thread당 register 사용량이 증가하면 한 SM/CU에 동시에 resident할 수 있는 warp/wave 수가 줄어 occupancy가 낮아질 수 있다. register가 부족하면 일부 값은 **local memory spill**로 내려갈 수 있다.

Ray tracing에서는 여기에 한 층이 더 생긴다.

```text
ordinary register pressure
    +
state that survives TraceRay / reorder / callable boundaries
    =
ray-tracing live-state problem
```

즉 어떤 값이 shader 전체에서 register 하나를 차지하는 것보다 더 비싼 경우는, 그 값이 `TraceRay()` 전에도 필요하고 호출 이후에도 다시 필요해서 **continuation state**가 되는 경우다.

개념적으로 한 invocation의 boundary cost를 다음처럼 생각할 수 있다.

\[
C_{boundary}
\approx C_{base}
+ B_{live} \cdot C_{save/restore}
+ B_{migrate} \cdot C_{movement}
\]

- \(B_{live}\): boundary를 가로질러 보존해야 하는 live-state byte 수
- \(B_{migrate}\): reorder 시 실제로 이동해야 하는 state 크기
- \(C_{save/restore}\): memory spill/reload 비용
- \(C_{movement}\): processor/wave 간 state migration 비용

SER이 active-lane ratio를 높여도 \(B_{live}\)가 너무 크면 전체 성능이 생각보다 개선되지 않을 수 있다.

### 실제 사례: coherence가 좋아져도 memory traffic이 늘 수 있다

NVIDIA가 공개한 *Indiana Jones and the Great Circle* path tracing 최적화 사례에서는 SER을 적용하면서 active threads per warp가 크게 좋아졌지만, 동시에 L2 traffic이 증가했다. 원인은 RayGen shader의 **RT live-state spill**이었다.

해당 사례에서 profiler는 thread당 222 bytes의 ray-tracing live state를 보여주었고, loop 구조 단순화와 FP16 사용 등으로 이를 84 bytes까지 줄인 뒤 SER의 GPU-time 개선 폭이 더 커졌다.

여기서 중요한 교훈은 숫자 자체가 아니다.

> **coherence optimization과 state-size optimization은 서로 독립된 문제가 아니라 서로 곱해지는 문제다.**

SER로 thread를 더 자주/더 멀리 재배치할수록, 살아 있는 state가 크다면 그 이동 비용이 더 크게 드러난다.

### Live range가 진짜 최적화 단위다

register 최적화를 “자료형을 half로 줄이는 것”으로만 이해하면 중요한 부분을 놓친다.

다음 두 shader가 있다고 생각해보자.

```text
A:
largeMaterialData = LoadMaterial(...)
TraceRay(...)
Use(largeMaterialData)

B:
TraceRay(...)
largeMaterialData = LoadMaterial(...)
Use(largeMaterialData)
```

두 shader가 같은 계산을 하더라도 A에서는 `largeMaterialData`가 ray traversal boundary를 넘어 살아 있어야 한다. B에서는 traversal 후 처음 생성되므로 boundary live state가 아니다.

따라서 핵심은 값의 **크기(size)**뿐 아니라 **생존 구간(live range)**이다.

\[
\text{Optimization Priority}
\approx \text{State Size} \times \text{Boundary Crossings}
\]

64-byte struct 하나가 5개의 continuation/reorder boundary를 가로지르면, 16-byte 값 여러 개보다 더 위험한 상태가 될 수 있다.

### Occupancy는 목표가 아니라 결과 지표다

register pressure가 줄면 occupancy가 오를 수 있다. 하지만 높은 occupancy 자체를 절대 목표로 잡으면 안 된다.

낮은 occupancy에서도 충분한 instruction-level parallelism이 있다면 성능이 좋을 수 있고, register 수를 억지로 줄여 spill을 만들면 오히려 느려질 수 있다. CUDA Best Practices Guide도 occupancy와 성능이 단순 비례하지 않으며, register 사용량·spill·latency hiding을 함께 봐야 한다고 설명한다.

따라서 graphics engineer가 봐야 할 지표는 다음의 조합이다.

```text
registers / thread
RT live-state bytes / callsite
local-memory spill loads/stores
L1/L2 traffic
active lanes / warp
occupancy
GPU time
```

단일 지표가 아니라 **원인 → 실행 특성 → 실제 시간**의 연결을 보는 것이 중요하다.

## 4. 구현 관점

### 4.1 Continuation boundary를 먼저 식별한다

Ray/path renderer에서 가장 먼저 구분해야 할 것은 “일반 코드 경계”와 “state preservation이 필요한 경계”다.

대표적인 boundary는 다음과 같다.

```text
TraceRay / ray traversal
HitObject::TraceRay
MaybeReorderThread
HitObject::Invoke
CallShader / callable
wavefront queue write -> next kernel
persistent-thread yield
Work Graph node emission
```

이 경계를 기준으로 shader의 local variable을 세 종류로 나눌 수 있다.

```text
[Pre-boundary only]
ray setup, temporary address calculation, hint construction

[Cross-boundary live]
path throughput, RNG seed, bounce state, continuation token

[Post-boundary only]
material fetch result, BSDF evaluation temporaries, texture samples
```

성능 관점에서 가장 비싼 것은 가운데 그룹이다.

### 4.2 “carry”보다 “recompute”가 싸울 수 있다

GPU에서는 계산을 다시 하는 것이 memory에 상태를 저장하고 복원하는 것보다 싸게 끝나는 경우가 많다.

예를 들어 다음 값들은 상황에 따라 재계산 후보가 될 수 있다.

```text
normalized direction
simple geometric term
small basis reconstruction
hash-derived RNG component
material-table offset
packed normal decode
```

이를 단순한 비용 모델로 보면 다음과 같다.

\[
C_{carry} = C_{spill} + C_{reload} + C_{migration}
\]

\[
C_{recompute} = C_{ALU}
\]

만약 \(C_{ALU} < C_{carry}\)라면 재계산이 더 낫다.

특히 modern GPU에서 ALU는 상대적으로 풍부하고 off-chip memory traffic은 비싸기 때문에, **값을 유지하는 대신 의미를 유지하고 값은 다시 만드는 설계**가 자주 유리하다.

다만 texture lookup, random access table fetch처럼 cache miss 가능성이 큰 값은 무조건 재계산이 유리하다고 볼 수 없다. 핵심은 데이터의 성격을 구분하는 것이다.

### 4.3 Payload는 “편리한 struct”가 아니라 ABI다

DXR이나 Vulkan ray tracing에서 payload는 shader stage 사이의 통신 구조다. C++ 코드에서 일반 구조체를 설계하듯 필드를 계속 추가하면, 어느 순간 payload가 사실상 path state 전체를 운반하게 된다.

나쁜 mental model은 다음과 같다.

```text
Payload = 필요한 값은 일단 다 넣는 객체
```

더 좋은 mental model은 다음이다.

```text
Payload = continuation boundary를 통과해야 하는 최소 ABI
```

예를 들어 다음 두 구조를 비교할 수 있다.

```text
Large payload
float3 throughput
float3 radiance
float3 rayDir
float3 normal
float2 uv
uint  materialId
uint  bounce
uint  rng0
uint  rng1
bool  terminated
...
```

대신 의미를 분리하면 다음처럼 볼 수 있다.

```text
Compact continuation payload
uint packedThroughput
uint packedDirection
uint packedPathState
uint rngState
uint surfaceToken
```

물론 실제 압축 방식은 image-quality와 numerical stability 요구에 따라 달라진다. 중요한 것은 **payload field 하나를 추가할 때마다 pipeline ABI와 boundary state 비용이 커질 수 있다는 인식**이다.

### 4.4 FP16/16-bit integer는 bandwidth가 아니라 live state에도 의미가 있다

FP16은 흔히 memory bandwidth optimization으로만 생각한다. 하지만 live-state context에서는 register/state size optimization이기도 하다.

예를 들어 다음 값들은 full FP32가 반드시 필요하지 않을 수 있다.

```text
normalized hit distance
bounded roughness
throughput after exposure normalization
secondary radiance accumulator의 일부 경로
compact PDF/log-PDF
short bounce counters
flags / material classes
```

NVIDIA의 실제 path-tracing 사례에서는 accumulated radiance/hit-distance vector를 FP16으로 낮추고, throughput을 half precision으로 바꾸고, bounce counters를 16-bit로 줄이며, ray direction을 32-bit packed form으로 바꾸는 방식이 live-state 감소에 사용되었다.

여기서 주의할 점은 **precision demotion과 semantic compaction을 구분하는 것**이다.

```text
Precision demotion
float -> half
uint32 -> uint16

Semantic compaction
float3 direction -> octahedral / packed representation
multiple flags -> bit field
pointer-like state -> compact index/token
```

둘을 함께 사용하면 효과가 크지만, error propagation이 누적되는 path throughput이나 MIS weight는 정밀도 요구를 따로 검증해야 한다.

### 4.5 AoS/SoA보다 먼저 “boundary state / persistent state”를 나눈다

GPU memory layout 이야기에서는 AoS와 SoA가 자주 먼저 등장한다. 그러나 ray/path continuation에서는 그보다 상위 결정이 있다.

```text
Transient shader-local state
vs
Boundary-crossing continuation state
vs
Persistent path state in global memory
```

예를 들어 wavefront renderer에서는 모든 path state를 global SoA에 저장하고 kernel 간에는 `pathIndex`만 전달할 수 있다.

```text
queue entry: uint pathIndex

PathStateSoA
- throughput[]
- rng[]
- rayOrigin[]
- rayDirection[]
- bounce[]
- flags[]
```

이 방식은 register/live-state를 줄이는 대신 global memory traffic을 늘린다.

반대로 megakernel/SER renderer는 많은 상태를 shader local state로 유지해 global path buffer traffic을 줄일 수 있지만, boundary live state가 커질 위험이 있다.

그래서 두 구조는 다음 trade-off를 가진다.

| 관점 | Megakernel / SER | Wavefront / explicit state |
|---|---|---|
| state 위치 | register + RT continuation state | global path-state buffer |
| queue traffic | 적음 | 많음 |
| live-state pressure | 높아질 수 있음 | kernel 경계에서 작게 제어 가능 |
| divergence | SER/driver에 의존 | explicit binning 가능 |
| state visibility | compiler/driver 내부 비중 큼 | application이 명시적으로 관리 |
| debugging | callsite live-state 분석 중요 | buffer layout/queue 분석 중요 |

따라서 오늘의 개념은 특정 API 최적화가 아니라 **state placement architecture**의 문제다.

### 4.6 Continuation token이라는 관점

복잡한 path state를 그대로 옮기지 않고, 다음 실행을 복원할 수 있는 작은 **continuation token**만 전달하는 구조를 생각할 수 있다.

```text
continuation token
- path index
- stage / bounce state
- compact flags
- material/surface identifier
- RNG offset or seed
```

실제 큰 state는 persistent pool에 있고 token은 indirection만 제공한다.

이는 지난주에 다룬 **reservoir indirection / persistent subpath pool**과 직접 연결된다. 그때는 reservoir가 큰 subpath payload를 stable handle로 참조했다. 오늘은 같은 사고방식을 execution state에 적용한다.

```text
large state travels with thread
        ↓
small token travels with thread
large state stays in structured storage
```

단, tokenization은 free optimization이 아니다. indirection이 늘고 cache locality가 나빠질 수 있으므로, 자주 읽는 hot state와 드물게 읽는 cold state를 분리하는 것이 중요하다.

### 4.7 Hot / warm / cold path state 분리

Path state 전체를 하나의 struct로 두기보다 access frequency로 나누는 관점이 유용하다.

```text
Hot state
throughput, ray, RNG, bounce, flags

Warm state
material/surface metadata, previous PDF, medium state

Cold state
debug/provenance, long IOR stack, auxiliary denoiser metadata
```

Hot state는 가능한 한 compact하게 register 또는 cache-friendly memory에 두고, cold state는 필요할 때만 indirection으로 접근한다.

이렇게 하면 모든 invocation이 큰 struct를 carry하지 않아도 된다.

### 4.8 C++ render architecture에서의 contract

C++ 쪽에서도 shader payload를 단순 shader implementation detail로 두기보다 명시적인 contract로 관리하는 편이 좋다.

개념적으로는 다음 메타데이터가 유용하다.

```text
RayPipelineStateLayout
- PayloadBytes
- AttributeBytes
- ContinuationStateVersion
- UsesSER
- PackedDirectionFormat
- ThroughputPrecision
- PersistentPathStateStride
```

이 정보가 render pipeline creation과 shader permutation, profiler marker에 연결되면 “shader code 몇 줄 수정했더니 GPU time이 20% 증가했다” 같은 문제를 추적하기 쉬워진다.

특히 payload layout version을 C++과 HLSL/GLSL 양쪽에서 동일하게 관리하면 stale shader binary나 mismatched struct로 인한 오류도 줄일 수 있다.

### 4.9 Profiler에서 무엇을 봐야 하나

live-state 문제는 source code만 보고 정확히 추정하기 어렵다. compiler optimization과 driver lowering이 개입하기 때문이다.

그래픽스 엔지니어 관점에서 중요한 질문은 다음과 같다.

```text
어느 callsite에서 live-state bytes가 가장 큰가?
어떤 source variable이 boundary를 가로질러 살아 있는가?
SER ON/OFF에서 L2 traffic이 어떻게 변하는가?
register count와 local-memory spill이 어떻게 변하는가?
active-lane ratio 개선이 실제 GPU time 개선으로 이어지는가?
```

NVIDIA Nsight Graphics의 Ray Tracing Live State처럼 callsite별 live-state를 보여주는 도구는 이 문제를 직접 관찰하게 해준다.

CUDA 계열에서는 Nsight Compute의 register/local-memory 지표와 occupancy를 함께 볼 수 있다. DirectX 쪽에서는 PIX와 vendor profiler를 통해 shader execution 및 memory pressure를 교차 확인하는 구조가 현실적이다.

## 5. 내 관심 분야와 연결

### 실시간 렌더링 엔진

Deferred/Forward rendering에서는 register pressure가 주로 복잡한 material shader, large G-buffer decode, lighting loop에서 문제된다. Ray tracing/path tracing으로 가면 여기에 **continuation state**가 추가된다.

즉 renderer architecture가 발전하면서 최적화 관점도 바뀐다.

```text
Raster
instruction count / texture latency / register pressure

Compute-heavy rendering
occupancy / shared memory / register spill

Ray tracing
+ traversal boundary / payload / continuation live state

SER / irregular scheduling
+ state migration / reorder-boundary live range
```

이 계층을 설명할 수 있으면 단순 API 사용자가 아니라 GPU execution model을 이해하는 엔지니어로 보인다.

### GPU compute / simulation

CFD, particle, level-set 같은 simulation kernel에서도 비슷한 패턴이 있다.

예를 들어 긴 kernel에서 모든 intermediate를 유지하면 register pressure가 올라간다. kernel을 여러 stage로 분리하면 register pressure는 줄어들 수 있지만 global-memory round trip이 생긴다.

즉 simulation에서도 본질은 같다.

```text
keep state in registers
vs
materialize state to memory
vs
recompute state
```

Ray tracing의 continuation 문제는 이 trade-off가 더 명시적으로 드러난 사례라고 볼 수 있다.

### WebGPU / portable graphics

WebGPU에는 SER과 동일한 abstraction이 없지만, **live range minimization, compute pass granularity, storage-buffer state layout**이라는 핵심 원리는 동일하다.

하나의 giant compute shader가 많은 temporary와 branch를 가지면 register pressure와 divergence가 커질 수 있고, pass를 너무 잘게 나누면 storage-buffer traffic과 dispatch overhead가 커진다.

따라서 오늘의 개념은 vendor-specific ray tracing API를 넘어 portable GPU design principle로 이어진다.

## 6. 머릿속에 남길 질문 3개

1. **어떤 값이 “필요한 상태”인 것과 “boundary를 가로질러 살아 있어야 하는 상태”인 것은 왜 다른가?**
2. **FP16 packing, recomputation, global-state indirection 중 어떤 선택이 유리한지는 어떤 비용 모델로 비교해야 하는가?**
3. **SER로 active-lane ratio는 크게 좋아졌는데 GPU time이 거의 줄지 않았다면, live-state 관점에서 어떤 profiler 지표를 먼저 확인해야 하는가?**

## 7. graphics engineer 면접 질문 1개와 답변

### 질문

**“Ray tracing shader에서 register 사용량을 줄였는데도 성능이 거의 좋아지지 않았습니다. 반대로 SER을 켠 뒤 active lanes는 증가했지만 L2 traffic이 크게 늘었습니다. 어떤 관점으로 원인을 분석하겠습니까?”**

### 답변

먼저 일반적인 **register pressure**와 **ray-tracing live state**를 분리해서 보겠습니다. thread당 register 수가 줄었다고 해도 `TraceRay`, `MaybeReorderThread`, callable transition 같은 boundary를 가로질러 살아 있는 값이 많다면 driver가 continuation state를 memory에 spill하고 이후 restore할 수 있습니다.

SER을 사용하면 thread context가 reorder 과정에서 다른 wave/processor로 이동할 수 있기 때문에, boundary live-state byte 수가 크면 state migration과 L2 traffic이 증가할 수 있습니다.

따라서 다음 순서로 보겠습니다.

```text
1. callsite별 RT live-state bytes 확인
2. 어떤 변수/struct가 reorder 또는 trace boundary를 가로지르는지 확인
3. local-memory spill / L1-L2 traffic 확인
4. register count와 occupancy 변화 확인
5. active-lane ratio와 실제 GPU time을 함께 비교
```

개선 방향은 단순히 register cap을 강제하는 것이 아니라 **live range를 boundary 안쪽으로 이동**, 필요 없는 state 제거, 값의 **recomputation**, FP16/bit packing, large state를 persistent buffer로 옮기고 compact token만 carry하는 방법 등을 검토합니다.

핵심은 occupancy 하나를 최대화하는 것이 아니라 **execution coherence 이득보다 state save/restore와 memory traffic 비용이 작도록 만드는 것**입니다.

## 8. 포트폴리오 / 커리어 연결

이 개념은 포트폴리오에서 “SER을 사용했다”보다 훨씬 강한 이야기를 만들 수 있다.

좋은 graphics-engineering 설명은 기능 소개가 아니라 병목의 구조를 보여준다.

```text
Problem
Path tracing shader의 material/branch divergence가 높음

Observation
SER 적용 후 active-lane ratio는 개선됐지만 memory traffic 증가

Diagnosis
reorder point를 가로지르는 large live state 확인

Architecture change
- live range 축소
- payload ABI 축소
- hot/cold state 분리
- selected fields FP16/packed representation
- cheap values recompute

Validation
GPU time, RT live-state bytes, L2 traffic, active lanes를 함께 비교
```

이런 형태의 사례는 GPU 최적화 면접에서 매우 좋다. 왜냐하면 다음 능력을 동시에 보여주기 때문이다.

- SIMT execution 이해
- compiler/register lifetime 이해
- ray-tracing pipeline 이해
- memory hierarchy 이해
- C++/shader ABI 설계
- profiler 기반 성능 검증

특히 게임 엔진이나 실시간 renderer 팀에서는 “API feature를 알고 있다”보다 **실제 frame time을 지배하는 state/data movement를 추론할 수 있다**는 점이 더 중요하다.

또한 simulation/visualization 배경과 연결하면, 같은 사고방식을 compute shader의 state lifetime과 kernel fusion/fission 문제로 확장해서 설명할 수 있다.

## 9. 내일 이어서 볼 개념

**Continuation-Passing Path Tracing and Stackless GPU State Machines: Explicit State Serialization vs Driver-Managed Continuations**

오늘은 “boundary를 넘어가는 state를 작게 만드는 방법”을 보았다. 내일은 한 단계 더 나아가 **continuation 자체를 application이 명시적으로 표현하면 renderer architecture가 어떻게 바뀌는가**를 본다.

핵심 질문은 다음이다.

```text
Megakernel / driver-managed continuation
vs
Wavefront / explicit continuation token
vs
Persistent-thread state machine
```

그리고 recursive-looking path tracing을 stackless GPU state machine으로 바꾸었을 때 control flow, queue, path-state memory, cache locality, shader specialization이 어떻게 달라지는지 연결한다.

## 10. 참고 키워드

- Live State / Live Range
- Ray-Tracing Live-State Spill
- Register Pressure
- Register Spilling / Local Memory
- Occupancy
- Continuation / Continuation State
- Shader Execution Reordering (SER)
- Reorder Point
- `MaybeReorderThread`
- `HitObject`
- DXR 1.2 / Shader Model 6.9
- Payload ABI
- Payload Compaction
- Precision Demotion
- FP16 / `float16_t`
- 16-bit Integer
- Octahedral Direction Encoding
- Bit Packing
- Recomputation vs Carry
- Hot / Warm / Cold State
- Persistent Path State
- Continuation Token
- Wavefront Path Tracing
- Megakernel
- Local Memory Spill
- Nsight Graphics Ray Tracing Live State
- Nsight Compute Register / Occupancy Metrics

### 참고 문서

- Microsoft HLSL SER Proposal — https://microsoft.github.io/hlsl-specs/proposals/0027-shader-execution-reordering/
- DirectX Raytracing Functional Spec — https://microsoft.github.io/DirectX-Specs/d3d/Raytracing.html
- NVIDIA, *Path Tracing Optimization in Indiana Jones: Shader Execution Reordering and Live State Reductions* — https://developer.nvidia.com/blog/path-tracing-optimization-in-indiana-jones-shader-execution-reordering-and-live-state-reductions/
- NVIDIA CUDA C++ Best Practices Guide — https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/
