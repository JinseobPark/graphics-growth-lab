---
title: "Continuation State Compression and Path-State Virtualization: Hot/Cold Splitting, Tokenization, and GPU Working-Set Control"
date: "2026-08-19"
category: Graphics
tags: [GPU, Path Tracing, Ray Tracing, Continuation, State Compression, Path-State Virtualization, Working Set, Hot Cold Splitting, Tokenization, Wavefront, SER, DXR, Vulkan, Shader, Memory Layout, C++]
level: intermediate
---

# [Daily Graphics Growth] 2026-08-19 - Continuation State Compression and Path-State Virtualization: Hot/Cold Splitting, Tokenization, and GPU Working-Set Control

## 1. 오늘의 개념

어제는 **Continuation-Passing Style(CPS)**과 **stackless GPU state machine**을 통해, GPU path tracer가 논리적으로 재귀적인 알고리즘을 실제 실행에서는 `next stage + persistent state` 형태로 평탄화할 수 있다는 관점을 다뤘다.

오늘은 그 다음 질문으로 넘어간다.

> continuation을 명시적으로 저장하거나 runtime이 보존해야 한다면, **그 상태를 얼마나 작게 만들고, 얼마나 자주 쓰는 데이터만 GPU의 빠른 working set에 남길 것인가?**

오늘의 핵심은 **Continuation State Compression**과 **Path-State Virtualization**이다.

이 둘은 단순히 구조체 크기를 줄이는 최적화가 아니다.

- **Continuation State Compression**은 reorder point, `TraceRay`, callable transition, wavefront stage boundary를 가로질러 보존되는 상태의 크기를 줄이는 문제다.
- **Path-State Virtualization**은 path 전체 상태를 매 stage마다 직접 들고 다니지 않고, 작은 **token / handle / path index**로 persistent backing store를 간접 참조하는 문제다.

개념적으로 다음처럼 볼 수 있다.

```text
logical path state
    = hot continuation state
    + warm shading state
    + cold rarely-used state

queue / continuation record
    = compact token
    + minimal hot payload

persistent backing store
    = virtualized full path state
```

즉 CPU의 virtual memory가 큰 address space를 작은 physical working set으로 다루듯, GPU renderer도 전체 path state를 항상 register나 queue entry 안에 보관하지 않고 **필요한 시점에 필요한 subset만 active working set으로 끌어오는 구조**를 만들 수 있다.

여기서 `virtualization`은 운영체제 수준의 page fault를 뜻하지 않는다. renderer가 논리적 path 상태와 물리적 GPU storage 위치를 분리하고, **stable logical identity**와 **compact active representation**을 따로 유지한다는 의미다.

---

## 2. 한 줄 핵심

> **GPU path tracer의 성능은 “path state 전체 크기”보다 각 execution boundary에서 실제로 살아 있어야 하는 working set의 크기에 더 민감하며, hot/cold splitting과 tokenized indirection은 continuation traffic·register pressure·cache footprint를 동시에 줄이는 핵심 설계다.**

---

## 3. 왜 중요한가

### 3.1 상태가 커질수록 비용은 한 곳이 아니라 여러 계층에서 증폭된다

Path state가 커지면 단순히 global memory 사용량만 늘어나는 것이 아니다.

예를 들어 한 path가 다음 정보를 가진다고 하자.

```text
ray origin / direction
throughput
accumulated radiance
RNG state
bounce depth
BSDF / material context
MIS state
medium state
reservoir state
debug metadata
```

이 모든 값을 매 bounce마다 local variable로 유지하면 일부 값은 `TraceRay`나 `MaybeReorderThread` 같은 boundary를 가로질러 **RT live state**가 될 수 있다. 반대로 wavefront renderer에서 이 모든 값을 queue entry에 넣으면 queue bandwidth와 cache footprint가 커진다.

즉 동일한 logical state가 architecture에 따라 서로 다른 병목으로 나타난다.

```text
Megakernel / SER
    large local state
    -> register pressure
    -> live-state spill
    -> continuation movement cost

Wavefront
    large explicit state
    -> global memory traffic
    -> queue footprint
    -> cache pressure
    -> compaction bandwidth
```

따라서 핵심 최적화 대상은 다음과 같다.

\[
W_{active} = \sum_i S_i \cdot A_i
\]

- \(S_i\): 상태 항목의 byte 크기
- \(A_i\): 해당 stage에서 실제로 active하게 필요한 비율

전체 path state 크기 \(S_{total}\)보다 **현재 stage의 active working set \(W_{active}\)**가 더 중요한 이유다.

### 3.2 SER에서는 live state가 execution coherence의 이득을 먹어버릴 수 있다

Shader Execution Reordering(SER)의 목적은 divergent ray/path를 다시 묶어 execution과 memory coherence를 높이는 것이다.

하지만 reorder point를 가로질러 살아 있는 값은 invocation과 함께 보존되어야 한다. Microsoft의 현재 DXR functional specification은 `MaybeReorderThread`를 가로지르는 live state가 일부 architecture에서 더 비쌀 수 있음을 명시하고, 아예 **compress → reorder → uncompress** 형태의 live-state optimization 예시를 제공한다.

NVIDIA의 2025년 Indiana Jones path-tracing optimization 사례에서도 SER 적용 자체보다 **RT live state reduction**이 성능을 실질적으로 끌어올리는 중요한 단계였다. 해당 사례에서는 RayGen shader의 live-state spill이 222 bytes/thread 수준이었고, 상태 크기를 크게 줄인 뒤 SER의 효율이 더 좋아졌다.

즉 SER 성능은 다음처럼 보는 편이 정확하다.

\[
Gain_{SER}
\approx Gain_{coherence}
- Cost_{state\ movement}
- Cost_{spill/reload}
\]

coherence hint가 아무리 좋아도 continuation state가 과도하게 크면 순이익이 작아질 수 있다.

### 3.3 Wavefront renderer에서는 queue entry가 사실상 continuation ABI다

Wavefront path tracer에서는 queue가 stage 사이의 함수 호출을 대신한다.

```text
TraceQueue
   ↓
ShadeQueue
   ↓
ShadowQueue / NextBounceQueue
```

여기서 queue entry가 다음처럼 크다고 가정하자.

```text
QueueEntry = full Ray + throughput + RNG + BSDF + MIS + flags + ...
```

수백만 path가 여러 stage를 이동하면 단순 stage transition만으로도 큰 memory traffic이 발생한다.

반대로 queue를 다음처럼 축소할 수 있다.

```text
QueueEntry
    pathToken
    compact stage-local metadata
```

실제 persistent state는 별도 storage에 둔다.

```text
pathToken -> PathState backing store
```

이때 `pathToken`은 단순 index일 수도 있고, generation을 포함한 handle일 수도 있으며, virtualized pool의 logical slot일 수도 있다.

핵심은 다음과 같다.

> **queue는 데이터를 운반하는 컨테이너가 아니라 continuation ABI이며, ABI가 작을수록 scheduler와 memory system의 부담이 줄어든다.**

### 3.4 Working-set control은 cache locality와 occupancy를 동시에 건드린다

GPU path tracing의 state optimization은 항상 trade-off를 가진다.

큰 struct를 한 번 읽으면 코드가 단순하지만 cache line에 불필요한 field가 같이 들어온다. 반대로 너무 잘게 분리하면 indirection과 load instruction이 늘어난다.

이를 개념적으로 다음처럼 볼 수 있다.

\[
T \approx T_{compute}
+ T_{state\ load/store}
+ T_{cache\ miss}
+ T_{indirection}
+ T_{schedule}
\]

State compression은 `T_state load/store`와 `T_cache miss`를 줄일 수 있지만 decode 비용을 늘린다. Path-state virtualization은 active footprint를 줄일 수 있지만 `T_indirection`을 늘린다.

따라서 목표는 **가장 작은 구조체**가 아니라 **가장 작은 유효 working set**이다.

---

## 4. 구현 관점

### 4.1 Hot / Warm / Cold state 분리는 lifetime이 아니라 access frequency 기준으로 본다

Path state를 크게 세 계층으로 나눌 수 있다.

#### Hot state

거의 모든 bounce와 대부분의 stage에서 사용된다.

```text
throughput
RNG state
ray direction / compact origin representation
bounce depth
path flags
path token
```

Hot state는 register 또는 cache-friendly SoA에 가까이 있을수록 유리하다.

#### Warm state

특정 shading stage에서 반복적으로 사용된다.

```text
surface normal
material ID
roughness / lobe ID
PDF / MIS terms
light sample metadata
```

Warm state는 shading queue가 활성화될 때만 working set에 들어오도록 분리할 수 있다.

#### Cold state

드문 branch나 특수 path에서만 사용된다.

```text
nested medium stack
subsurface state
rare material extension
large debug payload
long-path diagnostics
optional reservoir provenance
```

Cold state까지 모든 path에 동일하게 배치하면 평균 path가 사용하지 않는 데이터 때문에 cache와 bandwidth를 소비한다.

이 분리는 단순한 SoA/AoS 논쟁보다 중요하다.

```text
AoS vs SoA
    = field layout 문제

Hot/Warm/Cold split
    = lifetime + access domain 문제
```

둘은 함께 사용될 수 있다.

### 4.2 Tokenization은 pointer 축소가 아니라 state ownership 분리다

Path-state virtualization의 중심에는 작은 **token**이 있다.

개념적으로 다음 형태다.

```text
PathToken
    logical path ID
    generation / epoch
    optional state-class bits
```

그리고 backing store는 token을 실제 물리 위치로 해석한다.

```text
PathToken
    ↓
Handle / indirection table
    ↓
Physical path-state slot
```

이 구조의 의미는 단순 주소 압축보다 크다.

- queue compaction 중에도 logical path identity를 유지할 수 있다.
- physical pool을 재배치해도 continuation reference를 안정적으로 유지할 수 있다.
- inactive/cold state를 별도 pool로 옮길 수 있다.
- stage별 queue entry를 작게 유지할 수 있다.

특히 temporal reuse나 persistent path pool이 존재하면 raw array index만으로는 stale reference 문제가 생길 수 있다. generation/epoch을 포함한 token은 **lifetime correctness**까지 표현할 수 있다.

### 4.3 Continuation record와 path state를 같은 구조체로 묶지 않는 이유

Continuation이 필요로 하는 것은 보통 전체 path가 아니다.

논리적으로 continuation state를 다음처럼 나눌 수 있다.

\[
C = (PC_{next}, H, T)
\]

- \(PC_{next}\): 다음 stage / resume point
- \(H\): hot state
- \(T\): backing store를 가리키는 token

반면 전체 persistent path는

\[
P = H \cup W \cup C_{cold}
\]

이다.

즉 continuation record는 전체 path state의 **view**이지 복사본이 아니다.

```text
ContinuationRecord
    stageTag
    pathToken
    compact hot state

PersistentPathStore
    warm state
    cold state
    optional large payload
```

이 설계는 explicit wavefront뿐 아니라 driver-managed continuation을 이해할 때도 유용하다. compiler가 어떤 local variable을 boundary 너머로 live하게 남기는지 보는 것은 사실상 **implicit continuation record를 추적하는 일**이기 때문이다.

### 4.4 압축 후보는 값의 의미와 오차 전파를 함께 본다

Graphics shader에서 흔히 줄일 수 있는 값은 다음과 같다.

```text
float3 normal        -> octahedral encoding
float3 direction     -> compact direction encoding
float4 radiance      -> half / shared exponent candidate
uint32 flags         -> bit field
material state       -> material ID + late reconstruction
full transform data  -> instance ID / transform index
surface attributes   -> primitive ID + barycentrics
```

하지만 모든 값이 같은 위험도를 갖지는 않는다.

예를 들어 다음 값은 precision error가 path 전체에 누적될 수 있다.

- throughput
- PDF
- MIS weight
- accumulated radiance
- world-space origin for very large scenes

반면 다음 값은 재구성 오차에 상대적으로 강할 수 있다.

- normalized normal
- compact enum/flag
- bounded roughness
- material ID

따라서 compression 판단은 단순 byte 절감이 아니라

\[
Benefit
= \Delta Bandwidth + \Delta Cache + \Delta LiveState
- DecodeCost
- NumericalRisk
\]

로 보는 편이 좋다.

### 4.5 Recomputation은 storage compression의 한 형태다

가장 강한 압축은 값을 저장하지 않는 것이다.

예를 들어 surface hit 이후 다음 데이터를 모두 carry할 수 있다.

```text
position
normal
UV
material parameters
```

또는 더 작은 provenance만 유지할 수 있다.

```text
instance ID
primitive ID
barycentrics
```

그리고 shading 시 geometry/material data에서 필요한 값을 다시 reconstruct한다.

이것은 전형적인 **recompute vs carry** trade-off다.

```text
Carry
    + 계산 감소
    - state 증가
    - cache / continuation 부담

Recompute
    + state 감소
    + continuation 축소
    - instruction 증가
    - geometry/material fetch 증가 가능
```

Ray tracing에서는 texture/material fetch가 이미 memory-bound일 수 있으므로 recomputation이 항상 이득은 아니다. 반대로 reorder point를 가로지르는 state가 특히 비싼 architecture에서는 몇 개의 ALU 연산으로 복구 가능한 값을 저장하지 않는 것이 유리할 수 있다.

### 4.6 SoA와 virtualization은 서로 보완적이다

PBRT v4의 GPU wavefront path tracer도 `PixelSampleState`를 SoA 형태로 유지하고 sample index를 통해 해당 상태를 찾아간다. 이것은 이미 path-state virtualization의 기본 아이디어와 가깝다.

```text
path/sample index
   ↓
SOA state arrays
```

여기서 한 단계 더 발전시키면 각 state domain을 서로 다른 pool로 나눌 수 있다.

```text
PathCoreSoA[token]
SurfaceStateSoA[surfaceToken]
MediumStatePool[mediumToken]
ReservoirState[reservoirToken]
```

이 방식은 stage마다 실제 사용하는 array만 접근하게 하므로 working set을 줄인다.

하지만 indirection이 지나치게 많아지면 pointer chasing과 scatter access가 늘어날 수 있다. 따라서 **한 path의 모든 상태를 하나로 묶는 것**과 **모든 field를 별도 buffer로 찢는 것** 사이의 중간점이 필요하다.

실무적으로는 접근 패턴이 비슷한 field를 하나의 state class로 묶는 **semantic SoA / clustered SoA**가 이해하기 좋은 모델이다.

### 4.7 C++ render-graph 관점에서는 state class마다 lifetime contract가 달라진다

C++ renderer에서 path-state virtualization을 사용하면 GPU buffer를 단순 리소스가 아니라 lifetime domain으로 봐야 한다.

```text
PathCoreBuffer
    path lifetime 전체

SurfaceStateBuffer
    hit -> shade 구간

ShadowStateBuffer
    direct-light visibility 구간

ColdExtensionPool
    rare feature가 필요한 path만
```

이렇게 보면 transient resource aliasing과 persistent resource ownership의 경계도 선명해진다.

- **persistent state**: path identity를 유지해야 함
- **transient stage state**: 해당 dispatch/wavefront phase 이후 폐기 가능
- **reconstructible state**: 저장하지 않고 provenance만 유지 가능
- **cold state**: sparse pool 또는 optional allocation 가능

Render graph에서 resource lifetime을 잘못 짧게 잡으면 단순 graphics corruption이 아니라 continuation이 stale state를 참조하는 correctness bug가 된다.

### 4.8 프로파일링에서는 “bytes”와 “dependency cost”를 함께 본다

최근 Nsight Graphics의 Shader Profiler는 **Ray Tracing Live State**에서 callsite별 live-state bytes뿐 아니라 어떤 variable이 `TraceRay`/`traceRayEXT`를 가로질러 보존되는지 보여준다. 최신 문서에서는 live-state 크기뿐 아니라 dependency-attributed cost를 함께 보는 것이 유용하다고 설명한다.

따라서 다음 두 shader가 있다고 해도

```text
Shader A: 120 B live state
Shader B: 80 B live state
```

단순히 B가 무조건 빠르다고 결론내리면 안 된다.

실제 성능은

- spill 여부
- reuse frequency
- L1/L2 hit rate
- occupancy
- reorder frequency
- dependency stall
- decode/reconstruction cost

에 함께 좌우된다.

State compression의 목표는 **프로파일러 숫자를 가장 작게 만드는 것**이 아니라 **전체 path-tracing frame cost를 줄이는 것**이다.

---

## 5. 내 관심 분야와 연결

### C++ / GPU renderer

C++ renderer의 resource abstraction을 단순 `Buffer<T>`가 아니라 **state lifetime + ownership + access domain**으로 보면 구조가 훨씬 명확해진다.

예를 들어 path tracing뿐 아니라 particle simulation, sparse voxel traversal, level-set processing에서도 동일한 패턴이 나타난다.

```text
logical entity ID
    ↓
compact active queue
    ↓
persistent backing store
```

이 구조는 GPU-driven renderer와 simulation 모두에 적용된다.

### Rendering pipeline / shader

Raster pipeline에서는 vertex/fragment invocation이 비교적 짧고 stage가 명시적이다. 반면 ray/path tracing은 한 path가 여러 traversal, material, visibility stage를 거치므로 **state movement 자체가 pipeline architecture**가 된다.

따라서 shader 최적화의 관점이

```text
ALU 줄이기
texture fetch 줄이기
```

에서 끝나지 않고

```text
이 값은 어느 boundary를 가로질러 살아 있는가?
이 값은 queue에 직접 들어가야 하는가?
ID만 저장하고 재구성할 수 있는가?
이 state는 모든 path에 필요한가?
```

까지 확장된다.

### CUDA / GPGPU

CUDA 기반 simulation에서도 비슷한 문제가 있다.

예를 들어 sparse particle/voxel workflow에서 active element가 전체 state를 매 queue에 복사하는 대신 compact index를 운반하고 SoA pool을 참조하면 working set을 줄일 수 있다.

```text
active index list
   ↓
compact dispatch
   ↓
large persistent field arrays
```

이는 path-state virtualization과 동일한 구조적 아이디어다.

### Scientific visualization / semiconductor emulation

Level-set, sparse volume, marching cubes, multi-material emulation에서는 모든 voxel/element가 동일한 metadata를 필요로 하지 않는다.

예를 들어 대부분의 cell은

```text
phi
material
```

만 필요하지만 일부 active narrow-band cell만 추가 state를 필요로 할 수 있다.

이때 hot/cold split과 compact tokenized indirection은 graphics path tracing을 넘어 **GPU simulation working-set 설계**로 자연스럽게 연결된다.

즉 오늘의 개념은 ray tracing 전용 트릭이라기보다 다음의 일반 원칙이다.

> **큰 logical state를 그대로 실행 단위에 싣지 말고, active computation이 실제로 필요로 하는 최소 working set만 전면에 둔다.**

---

## 6. 머릿속에 남길 질문 3개

1. **한 path의 전체 state가 256 bytes라고 해도, 특정 continuation boundary에서 실제로 필요한 값이 48 bytes뿐이라면 나머지 208 bytes는 어디에 존재해야 가장 효율적인가?**

2. **FP16 packing, bit packing, octahedral encoding처럼 “값을 압축하는 방식”과 primitive/material ID만 저장하고 나중에 재구성하는 “의미를 압축하는 방식”은 어떤 조건에서 각각 유리한가?**

3. **Path-state virtualization으로 queue entry를 작게 만들었지만 backing store 접근이 random해져 L2 miss가 증가한다면, 이 최적화는 성공한 것인가? 어떤 지표로 판단해야 하는가?**

---

## 7. graphics engineer 면접 질문 1개와 답변

### 질문

**Wavefront path tracer의 queue entry를 줄이기 위해 full path state 대신 `pathIndex`만 저장하는 설계의 장점과 단점은 무엇인가요?**

### 답변

가장 큰 장점은 **stage transition bandwidth와 queue working set을 줄일 수 있다는 점**이다. Queue entry가 `pathIndex` 또는 compact token만 가지면 compaction·scan·dispatch 과정에서 이동하는 데이터가 작아지고, active queue가 cache에 더 잘 들어갈 수 있다. 또한 전체 path state를 persistent SoA에 두면 stage별로 필요한 field만 접근할 수 있어 hot/cold splitting이 쉬워진다.

반면 단점은 **indirection 비용과 random access 가능성**이다. Queue는 작아졌지만 `pathIndex`가 가리키는 backing store가 공간적으로 흩어져 있으면 L1/L2 locality가 나빠질 수 있고, 여러 state buffer를 따라가는 pointer chasing이 늘어날 수 있다. 또한 token lifetime 관리가 잘못되면 stale index나 ABA-style reuse 같은 correctness 문제가 생긴다.

따라서 좋은 설계는 단순히 queue entry를 최소화하는 것이 아니라 다음을 함께 맞춰야 한다.

- queue entry 크기
- backing store의 SoA/AoSoA layout
- token의 안정성
- compaction 이후 locality
- stage별 실제 field access pattern
- cache miss와 global-memory transaction

즉 핵심 trade-off는 **“state copy cost를 줄이는 대신 indirection cost를 얼마나 감당할 것인가”**이다.

---

## 8. 포트폴리오 / 커리어 연결

Graphics engineer 포트폴리오에서 path tracer의 기능만 보여주는 것보다, **state movement와 working set을 설계한 이유를 설명할 수 있는 것**이 훨씬 강하다.

예를 들어 다음과 같은 설명은 시스템 수준의 이해를 보여준다.

```text
Before
- full path payload가 여러 stage를 통과
- live state / queue footprint 증가
- stage마다 불필요한 field 접근

After
- hot continuation state와 persistent path state 분리
- queue entry는 compact token 중심
- surface/material/cold state를 access domain별 분리
- recomputable attribute는 provenance만 유지
```

그리고 성능 분석은 다음 지표로 연결할 수 있다.

```text
RT live-state bytes / callsite
register count
occupancy
L1/L2 hit rate
queue bytes moved per bounce
active paths per stage
global load/store throughput
SER active-lane improvement
state spill/reload traffic
```

이런 설명은 단순히 “GPU 최적화를 했다”보다 훨씬 구체적이다.

면접에서도 다음 관점을 설명할 수 있으면 강하다.

> **GPU renderer의 성능 문제를 shader instruction count만으로 보지 않고, control-flow boundary를 가로지르는 state lifetime과 memory working set 문제로 모델링할 수 있다.**

이는 게임 엔진, real-time path tracing, GPU simulation, WebGPU compute, scientific visualization에서 모두 통하는 역량이다.

---

## 9. 내일 이어서 볼 개념

**State Locality Restoration After Compaction: Morton/Material Binning, Handle Remapping, and Cache-Coherent Path Pools**

오늘은 path state를 작게 만들고 token으로 virtualize하는 방법을 봤다. 하지만 queue compaction과 path reuse가 반복되면 logical path ID와 physical storage가 점점 뒤섞여 **backing store의 spatial locality가 무너질 수 있다**.

내일은 다음 질문으로 이어간다.

> **작은 token을 사용해 state movement는 줄였는데, token이 가리키는 실제 state가 메모리 전체에 흩어져 있다면 locality를 어떻게 다시 회복할 것인가?**

다룰 흐름은 다음과 같다.

```text
path-state virtualization
        ↓
physical fragmentation
        ↓
cache-unfriendly gather
        ↓
state compaction / remapping
        ↓
Morton/material/depth binning
        ↓
cache-coherent path pool
```

즉 오늘의 **working-set size**에서 내일은 **working-set placement**로 이동한다.

---

## 10. 참고 키워드

### 핵심 용어

- Continuation State Compression
- Path-State Virtualization
- GPU Working Set
- Hot / Warm / Cold State
- Tokenization
- Handle Indirection
- Path Token
- Continuation ABI
- Persistent Path State
- RT Live State
- Live-State Spill
- Shader Execution Reordering (SER)
- `MaybeReorderThread`
- Wavefront Path Tracing
- Megakernel
- State Reconstruction
- Recompute vs Carry
- SoA / AoSoA
- Queue Compaction
- Cache Locality
- Register Pressure
- L1 / L2 Working Set
- Generational Handle
- Resource Lifetime

### 연결해서 읽을 자료

- Microsoft DirectX Raytracing Functional Spec — Shader Execution Reordering, reorder points, live-state optimization example
  - https://microsoft.github.io/DirectX-Specs/d3d/Raytracing.html
- NVIDIA Technical Blog — *Path Tracing Optimization in Indiana Jones™: Shader Execution Reordering and Live State Reductions* (2025-05-15)
  - https://developer.nvidia.com/blog/path-tracing-optimization-in-indiana-jones-shader-execution-reordering-and-live-state-reductions
- NVIDIA Nsight Graphics Shader Profiler — Ray Tracing Live State
  - https://docs.nvidia.com/nsight-graphics/UserGuide/shader-profiler.html
- Khronos Vulkan Documentation — Shader Execution Reordering / `VK_EXT_ray_tracing_invocation_reorder`
  - https://docs.vulkan.org/samples/latest/samples/extensions/ray_tracing_invocation_reorder/README.html
- Laine, Karras, Aila — *Megakernels Considered Harmful: Wavefront Path Tracing on GPUs*, HPG 2013
  - https://research.nvidia.com/publication/2013-07_megakernels-considered-harmful-wavefront-path-tracing-gpus
- PBRT v4 — *Wavefront Rendering on GPUs / Path Tracer Implementation*
  - https://www.pbr-book.org/4ed/Wavefront_Rendering_on_GPUs/Path_Tracer_Implementation
