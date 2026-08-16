---
title: "Shader Execution Reordering and Dynamic Coherence Recovery: Ray/Path Reordering Beyond Explicit Binning"
date: "2026-08-16"
category: Graphics
tags: [GPU, Ray Tracing, Shader Execution Reordering, SER, DXR 1.2, Shader Model 6.9, Vulkan, HitObject, SIMT, Divergence, Path Tracing, Memory Coherence, Register Pressure, C++]
level: intermediate
---

# [Daily Graphics Growth] 2026-08-16 - Shader Execution Reordering and Dynamic Coherence Recovery: Ray/Path Reordering Beyond Explicit Binning

## 1. 오늘의 개념

어제는 **GPU-Driven Work Graphs**와 **Persistent Threads**를 통해 불규칙한 렌더링 workload를 GPU 내부에서 생성하고 scheduling하는 구조를 보았다. 오늘은 그보다 더 미세한 실행 단위로 내려가, 이미 실행 중인 ray/path invocation의 **동적 실행 coherence**를 회복하는 **Shader Execution Reordering (SER)**을 본다.

Path tracing에서는 같은 화면 tile에서 시작한 ray라도 몇 bounce만 지나면 서로 다른 geometry, material, texture, control flow를 만나게 된다. 결과적으로 하나의 warp/wave 안에서 각 lane이 서로 다른 코드를 수행하거나 서로 다른 memory region을 읽는다.

이 문제를 전통적으로 해결하는 방법은 explicit queue와 binning이다.

```text
trace -> material classify -> compact/sort -> shade -> next queue
```

하지만 이 방식은 추가 queue memory, prefix sum/compaction, indirect dispatch, synchronization을 요구한다. SER은 다른 방향을 취한다.

> **shader가 특정 지점에서 “이후 실행이 비슷한 invocation끼리 다시 묶어 달라”고 runtime/hardware에 힌트를 주고, 실제 재배치는 구현체에 맡긴다.**

DirectX에서는 DXR 1.2 / Shader Model 6.9의 `HitObject`와 `MaybeReorderThread()`가 핵심 abstraction이다. Vulkan에서는 `VK_EXT_ray_tracing_invocation_reorder`와 `GL_EXT_shader_invocation_reorder` 계열의 hit object 및 reorder operation으로 같은 개념을 표현한다.

SER의 중요한 구조적 변화는 **ray traversal과 hit shading의 분리**다.

```text
기존 TraceRay
Traversal + AnyHit/Intersection + ClosestHit/Miss

SER mental model
Traversal
   -> HitObject
   -> Reorder Point
   -> ClosestHit/Miss 또는 이후 custom work
```

즉 SER은 단순한 “thread sort instruction”이 아니라, ray tracing pipeline을 **traversal / reorder / shading**의 세 단계로 다시 볼 수 있게 하는 실행 모델이다.

## 2. 한 줄 핵심

> **SER은 explicit queue sorting 없이도 ray/path invocation을 reorder point에서 다시 묶어 control-flow와 memory coherence를 회복하지만, 실제 성능은 coherence hint의 품질과 reorder 이후까지 살아 있는 live state의 크기에 크게 좌우된다.**

## 3. 왜 중요한가

### SIMT divergence의 실제 비용

GPU의 warp/wave가 \(W\)개의 lane으로 구성되어 있다고 하자. branch \(k\)에서 실제 활성 lane 수가 \(A_k\)라면 간단한 실행 효율은 다음처럼 볼 수 있다.

\[
\eta_k = \frac{A_k}{W}
\]

서로 다른 material branch가 순차적으로 실행되면 총 비용은 단순 평균이 아니라 각 branch가 warp를 점유하는 시간의 합에 가까워진다.

```text
lane 0~7   : diffuse
lane 8~15  : glass
lane 16~23 : layered metal
lane 24~31 : emissive / miss
```

논리적으로 32개 ray가 동시에 존재해도 실제 shader 실행은 여러 branch mask를 순차적으로 처리할 수 있다. 여기에 texture/material data access까지 흩어지면 L1/L2 cache locality도 함께 악화된다.

Path tracing이 특히 어려운 이유는 divergence가 bounce마다 누적된다는 점이다.

\[
\text{coherent primary rays}
\rightarrow \text{secondary rays}
\rightarrow \text{material divergence}
\rightarrow \text{direction divergence}
\rightarrow \text{memory divergence}
\]

### Explicit binning과 SER의 차이

어제까지 본 wavefront / Work Graph architecture는 **work를 stage 또는 class 단위로 분리**한다.

```text
Material Queue A
Material Queue B
Visibility Queue
Reconnection Queue
```

SER은 stage를 새로 만드는 대신 **현재 ray-generation execution 안에서 invocation의 물리적 grouping을 바꿀 수 있다.**

| 관점 | Explicit Binning / Wavefront | SER |
|---|---|---|
| 실행 단위 | queue entry / kernel | shader invocation |
| grouping 제어 | application이 명시적 분류 | application hint + runtime |
| data movement | queue write/read 필요 | context migration / repacking |
| synchronization | dispatch/queue 경계 | reorder point |
| 비용 | compaction + memory traffic | live-state save/restore + reorder overhead |
| 제어 수준 | 높음 | 구현체 의존 |
| 적합한 문제 | 큰 stage 분리 | shader 내부의 fine-grained divergence |

둘은 경쟁 관계라기보다 계층이 다르다.

> **Wavefront scheduling은 coarse-grained coherence를 만들고, SER은 그 stage 내부에서 남아 있는 fine-grained incoherence를 다시 회복할 수 있다.**

### 2026년 기준 API 위치

DirectX의 Shader Execution Reordering은 2026년 2월 Shader Model 6.9와 함께 preview를 벗어나 정식으로 공개되었다. 2026년 7월 10일자 DirectX Raytracing functional spec v1.45에는 SER, `HitObject`, `MaybeReorderThread`, reorder point와 memory visibility 규칙이 포함되어 있다.

중요한 점은 **SER API 지원과 실제 hardware reordering이 동일한 의미가 아니라는 것**이다.

Shader Model 6.9 + ray tracing을 지원하는 장치는 SER shader code를 받아들여야 하지만, 실제 reorder가 일어나지 않는 구현도 허용된다. DirectX에서는 `ShaderExecutionReorderingActuallyReorders` capability로 이를 구분한다.

Vulkan의 현재 Khronos 문서는 `VK_EXT_ray_tracing_invocation_reorder`를 통해 같은 구조를 표준 EXT 형태로 제공한다. 역시 device property를 통해 구현이 실제 reorder를 수행하는지 확인할 수 있다.

이 설계는 graphics engineer 관점에서 중요하다.

> SER을 사용한다고 해서 algorithm correctness가 hardware sorting에 의존해서는 안 된다. Reordering은 결과를 바꾸는 기능이 아니라 **동일한 work를 더 coherent하게 실행하는 성능 layer**여야 한다.

## 4. 구현 관점

### HitObject: traversal 결과를 shading에서 분리한다

SER에서 가장 중요한 상태 객체는 **HitObject**다.

개념적으로 HitObject는 다음과 같은 정보를 가진 opaque traversal result로 볼 수 있다.

```text
HitObject
- hit / miss / nop state
- ray T / hit geometry state
- shader table record identity
- instance / primitive information
- traversal-derived state
```

기존 `TraceRay()`에서는 traversal이 끝나면 바로 closest-hit 또는 miss shader가 실행될 수 있다. HitObject 방식에서는 traversal 결과를 먼저 확보한 뒤, 그 정보를 기준으로 reorder하고 나서 shading을 실행할 수 있다.

```text
HitObject::TraceRay(...)
        ↓
MaybeReorderThread(hit, hint)
        ↓
HitObject::Invoke(...)
```

이 분리가 중요한 이유는 **shading coherence를 traversal 순서와 분리**하기 때문이다.

### Coherence hint의 의미

SER은 hit object 자체만으로도 grouping을 시도할 수 있지만, application-defined **coherence hint**를 추가할 수 있다.

좋은 hint는 reorder 이후 실행할 코드와 memory access를 잘 예측해야 한다.

예를 들어 다음 값들이 후보가 될 수 있다.

```text
material class
BSDF lobe class
decal / alpha state
texture-set family
path state class
reconnection / replay class
Russian-roulette state
```

반대로 단순히 공간적으로 가까운 pixel ID가 항상 좋은 hint인 것은 아니다. secondary ray 이후에는 screen-space proximity와 material/data coherence가 크게 분리될 수 있기 때문이다.

핵심은 다음이다.

\[
\text{Good Hint} \approx
\text{Predictor of future control flow + future memory access}
\]

즉 hint는 “과거에 어디서 왔는가”보다 **reorder 이후 무엇을 할 것인가**를 표현하는 편이 더 중요하다.

### Reorder point는 semantic boundary다

`MaybeReorderThread()`는 단순 compiler intrinsic 이상의 의미를 가진다. DirectX spec에서 reorder point는 invocation이 다른 processor, wave, lane 위치로 이동할 수 있는 지점이다.

따라서 reorder 전후에 다음 가정은 위험하다.

```text
wave lane index가 계속 동일하다
같은 wave에 있던 lane들이 계속 함께 있다
WaveActive* 결과의 participant set이 유지된다
subgroup-local ordering이 유지된다
```

SER 이후에는 **logical shader invocation identity는 유지되지만 physical execution grouping은 바뀔 수 있다.**

이 구분은 GPU debugging에서도 중요하다.

```text
Logical state: pixel/path/ray identity
Physical state: wave/lane/processor placement
```

SER은 logical state를 유지하면서 physical state를 재배치한다.

### Live state가 SER의 숨은 비용이다

SER의 가장 중요한 성능 비용 중 하나는 reorder point를 가로질러 살아 있는 **live state**다.

예를 들어 다음 값이 reorder 뒤에서도 필요하다면 runtime은 invocation을 이동시킬 때 해당 상태를 보존해야 한다.

```text
ray throughput
path depth
RNG state
BSDF temporary values
multiple float3 vectors
reservoir metadata
texture coordinates
large payload
```

이를 개념적으로 다음처럼 볼 수 있다.

\[
C_{SER} \approx C_{reorder} + C_{save/restore}(S_{live})
\]

여기서 \(S_{live}\)는 reorder point를 가로질러 살아 있는 register/local state의 크기다.

SER로 divergence를 줄여 얻는 이득보다 context migration 비용이 커지면 성능이 악화될 수 있다.

그래서 SER shader에서는 **reorder boundary가 register-lifetime boundary이기도 하다.**

```text
Before reorder
- traversal에 필요한 state
- hint 생성에 필요한 최소 state

Reorder point

After reorder
- material fetch
- BSDF evaluation
- expensive texture access
- divergent control flow
```

reorder 이전에 계산한 대량의 temporary를 이후까지 유지하는 구조는 SER의 장점을 약화시킨다.

### Register pressure와 occupancy

Live state가 크면 단순 save/restore 비용뿐 아니라 shader 전체 register pressure에도 영향을 줄 수 있다.

\[
\text{Occupancy} \propto
\frac{R_{SM}}{R_{thread} \times T_{resident}}
\]

정확한 occupancy 계산은 architecture마다 다르지만, thread당 register 사용량 \(R_{thread}\)가 증가하면 동시에 resident할 수 있는 wave 수가 감소할 수 있다.

따라서 SER의 성능 분석은 다음 두 지표를 함께 봐야 한다.

```text
reorder 후 active-lane ratio ↑
register/live-state pressure ↑ or ↓ ?
```

SER 적용 후 warp execution efficiency만 좋아졌는데 전체 GPU time이 개선되지 않는다면, live state와 occupancy가 첫 번째 의심 지점이 된다.

### Memory coherence는 자동으로 해결되지 않는다

SER은 비슷한 invocation을 묶어 **coherent access 가능성**을 높이지만, memory ordering/barrier를 대신하지 않는다.

DirectX의 reorder point는 물리적 thread placement가 바뀔 수 있는 semantic boundary이므로 UAV를 통한 communication에는 명시된 visibility 규칙을 고려해야 한다.

또한 non-uniform resource access를 reorder point 전후로 compiler가 임의 이동시키면 의도했던 coherence가 사라질 수 있다. DirectX spec이 reorder point 주변의 non-uniform access movement를 특별히 다루는 이유다.

즉 두 개념을 분리해야 한다.

```text
Execution coherence
= 비슷한 일을 하는 thread를 같이 실행

Memory synchronization
= write/read ordering과 visibility 보장
```

SER은 전자에 가깝다.

### C++ renderer 관점의 feature contract

Renderer의 C++ 쪽에서는 SER을 단순 shader macro 하나로 보기보다 capability와 pipeline contract로 다루는 편이 자연스럽다.

개념적 state는 다음과 같다.

```text
RayTracingCapabilities
- rayTracingTier
- shaderModel
- serShaderSupported
- serActuallyReorders
- hitObjectSupported
```

shader permutation도 최소한 다음 세 경로를 생각할 수 있다.

```text
Legacy TraceRay path
HitObject path without effective reorder
HitObject + SER path
```

실제 reorder가 no-op인 hardware에서도 HitObject는 traversal과 shading 분리 자체로 가치가 있다. 따라서 “SER off = 반드시 legacy TraceRay”라는 구조만 있는 것은 아니다.

### SER과 Work Graphs의 관계

어제의 Work Graphs와 오늘의 SER을 하나의 계층으로 정리하면 다음과 같다.

```text
Render Graph / CPU command level
        ↓
Work Graph / Persistent Queue
  dynamic stage scheduling
        ↓
Wavefront stage
  coarse work classification
        ↓
Shader Execution Reordering
  fine-grained invocation regrouping
        ↓
SIMT execution
```

Work Graphs는 **어떤 work node를 언제 실행할 것인가**를 다루고, SER은 **현재 shader work를 어떤 invocation grouping으로 실행할 것인가**를 다룬다.

ReSTIR PT처럼 workload class가 다양하고 각 class 내부에서도 material divergence가 심한 경우 두 구조는 함께 존재할 수 있다.

## 5. 내 관심 분야와 연결

### ReSTIR / path reuse

최근 이어서 본 ReSTIR PT 흐름에서는 candidate가 compatibility, replay length, reconnection, visibility 상태에 따라 서로 다른 실행 경로를 가진다.

큰 실행 class는 wavefront queue 또는 Work Graph node로 분리할 수 있고, 그 내부의 hit shading은 SER로 다시 묶을 수 있다.

```text
Reconnection Node
    -> trace shifted path
    -> HitObject
    -> reorder by material / lobe
    -> evaluate BSDF
```

이 구조의 핵심은 estimator와 scheduler를 분리하는 것이다.

- reservoir weight / MIS / Jacobian: **sampling correctness**
- queue / Work Graph / SER: **execution efficiency**

SER이 sample probability를 바꾸면 안 되며, reorder 여부에 따라 결과가 달라져서도 안 된다.

### GPU compute와 scientific visualization

SER 자체는 ray-generation 중심 기능이지만 그 underlying idea는 일반 GPU workload에도 연결된다.

> **dynamic data가 만든 divergence를 compile-time이 아니라 runtime information으로 다시 coherent하게 만든다.**

CFD/voxel/level-set pipeline에서도 비슷한 문제는 나타난다.

```text
active voxel classification
material / phase classification
surface / interior classification
adaptive branch
```

일반 compute에서는 SER을 그대로 사용할 수 없더라도, 같은 사고방식이 queue binning, subgroup specialization, persistent scheduler, work graphs 설계로 이어진다.

### Rendering pipeline 설계

SER을 이해하면 ray tracing pipeline을 “shader stage들의 고정 연결”로만 보지 않게 된다.

```text
Traversal
Hit representation
Scheduling / reordering
Material evaluation
Lighting
Continuation
```

이 분해는 C++ engine architecture에서도 ray payload, material system, SBT, render graph, profiler event를 더 명확하게 나누는 기준이 된다.

### Memory layout 관점

SER은 큰 global buffer를 직접 sort하지 않기 때문에 explicit binning보다 queue bandwidth를 줄일 가능성이 있다. 하지만 invocation context가 reorder point를 넘어 이동하므로 **live register/local state가 사실상 hidden payload**가 된다.

따라서 SER에서 memory layout을 볼 때는 visible GPU buffers뿐 아니라 다음도 함께 봐야 한다.

```text
visible state
- ray payload
- HitObject
- material buffers
- path/reservoir pools

hidden execution state
- live registers
- local variables
- compiler-generated continuation state
```

그래픽스 엔지니어가 SER을 잘 다룬다는 것은 API 호출을 아는 것보다 **이 hidden state의 비용을 추론할 수 있다는 의미**에 더 가깝다.

## 6. 머릿속에 남길 질문 3개

1. **Explicit material binning이 이미 존재하는 wavefront path tracer에서 SER을 추가하면 어떤 divergence가 여전히 남아 있으며, 어느 수준부터 두 기법의 overhead가 중복되기 시작하는가?**

2. **좋은 SER coherence hint는 현재 hit의 속성을 표현해야 하는가, 아니면 reorder 이후 실행할 material/BSDF/texture access pattern을 예측해야 하는가? 두 값이 다를 때 무엇을 우선해야 하는가?**

3. **SER 적용 후 active-lane ratio는 좋아졌지만 frame time이 거의 줄지 않았다면, live state, register pressure, memory latency, reorder frequency 중 무엇을 어떤 순서로 의심할 것인가?**

## 7. graphics engineer 면접 질문 1개와 답변

### 질문

**Shader Execution Reordering과 traditional wavefront material sorting의 차이를 설명하고, SER이 항상 더 빠르지 않은 이유를 말해보세요.**

### 답변

Wavefront material sorting은 ray/path를 explicit queue에 기록하고 material 또는 work class별로 compaction/sort한 뒤 별도 dispatch에서 처리하는 **coarse-grained software scheduling** 방식이다. 장점은 grouping policy를 애플리케이션이 완전히 제어할 수 있다는 것이고, 단점은 queue memory traffic, scan/compaction, indirect dispatch, synchronization 비용이다.

SER은 ray-generation shader의 reorder point에서 `HitObject`와 coherence hint를 runtime/hardware에 전달해 invocation을 다시 묶는 **fine-grained execution reordering** 방식이다. 별도의 material queue를 만들지 않아도 hit shading과 subsequent control flow의 coherence를 개선할 수 있지만 실제 reordering policy와 범위는 구현체에 의존한다.

SER이 항상 빠르지 않은 가장 큰 이유는 reorder 자체가 무료가 아니기 때문이다. reorder point를 넘어 살아 있는 register/local variable이 많으면 invocation context 보존 비용이 증가하고 register pressure/occupancy가 악화될 수 있다. 또한 이미 매우 coherent한 primary ray나 단순 shader에서는 얻을 수 있는 divergence 감소가 작다. 따라서 성능 판단은 **reorder 이후 coherence 개선량과 live-state/reorder overhead의 차이**로 봐야 한다.

## 8. 포트폴리오 / 커리어 연결

SER은 단순히 “RTX 기능을 사용했다”보다 **GPU execution architecture를 이해하는 근거**로 보여주는 것이 가치가 크다.

포트폴리오에서 좋은 설명 축은 다음과 같다.

```text
Problem
- secondary ray 이후 material divergence 증가

Baseline
- monolithic TraceRay 또는 explicit material queue

Architecture
- traversal / HitObject / reorder / shading 분리

Performance reasoning
- active lane ratio
- shader/material divergence
- live-state size
- register pressure
- cache behavior

Fallback
- no-op reorder hardware에서도 동일한 correctness 유지
```

특히 NVIDIA/AMD/Intel vendor feature를 외우는 수준보다, **coarse scheduling(Work Graphs/wavefront)과 fine scheduling(SER)의 역할을 분리해서 설명할 수 있는 능력**이 graphics/game-engine 면접에서 더 강한 신호가 된다.

C++ 관점에서도 capability query, shader permutation, pipeline fallback, profiler marker, feature abstraction까지 연결하면 renderer engineer 포트폴리오로서 완성도가 높아진다.

## 9. 내일 이어서 볼 개념

**Live-State Minimization Across Reorder Points: Register Pressure, Continuations, and Payload Compaction**

오늘 SER에서 가장 중요한 숨은 비용이 **reorder point를 가로질러 살아 있는 state**라는 점을 확인했다. 내일은 이를 더 깊게 파고들어 다음을 연결한다.

- live range와 register allocation
- ray payload와 local variable의 차이
- continuation state / stack pressure
- payload compaction과 recomputation trade-off
- FP16 / packed state의 의미
- Nsight/PIX에서 register·occupancy 병목을 읽는 관점
- megakernel, wavefront, SER architecture에서 state lifetime이 달라지는 이유

즉 오늘의 질문이 “thread를 어떻게 다시 묶을 것인가?”였다면, 내일은 **“다시 묶을 때 thread가 들고 이동해야 하는 짐을 어떻게 줄일 것인가?”**로 이어진다.

## 10. 참고 키워드

- Shader Execution Reordering (SER)
- DirectX Raytracing 1.2 (DXR 1.2)
- Shader Model 6.9
- `dx::MaybeReorderThread`
- `HitObject`
- Reorder Point
- Shader Invocation Reordering
- `VK_EXT_ray_tracing_invocation_reorder`
- `GL_EXT_shader_invocation_reorder`
- SIMT Divergence
- Warp / Wave Coherence
- Material Divergence
- Data Divergence
- Coherence Hint
- Live State
- Register Pressure
- Occupancy
- Ray Payload
- Shader Binding Table (SBT)
- Wavefront Path Tracing
- Explicit Binning
- Work Graphs
- Persistent Threads
- ReSTIR PT
- DirectX Raytracing Functional Spec v1.45, 2026-07-10
- Microsoft DirectX Developer Blog: D3D12 Shader Execution Reordering, 2026-02-26
- Khronos Vulkan Documentation: `VK_EXT_ray_tracing_invocation_reorder`
