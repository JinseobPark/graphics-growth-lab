---
title: "Persistent Subpath Pools and Reservoir Indirection: GPU Lifetime, Compaction, and Memory Coherence"
date: "2026-08-13"
category: Graphics
tags:
  - ReSTIR
  - GRIS
  - Conditional ReSTIR
  - GPU Memory
  - Wavefront Path Tracing
  - Reservoir
  - Subpath Reuse
  - Compaction
  - Indirection
  - C++
  - Vulkan
  - DirectX 12
level: intermediate
---

# [Daily Graphics Growth] 2026-08-13 - Persistent Subpath Pools and Reservoir Indirection: GPU Lifetime, Compaction, and Memory Coherence

## 1. 오늘의 개념

어제는 **Conditional GRIS / Conditional ReSTIR**에서 하나의 full path 전체를 그대로 재사용하는 대신, 현재 pixel에서 새로운 prefix를 만들고 과거 sample의 suffix/subpath를 조건부로 재사용하는 구조를 보았다. 오늘은 그 확률적 개념을 실제 GPU renderer의 메모리 시스템으로 옮긴다.

핵심 문제는 단순하다.

> Reservoir는 작고 고정된 크기를 원하지만, 재사용할 subpath는 길이가 가변적(variable-length)이다.

예를 들어 reservoir 하나가 다음과 같은 estimator state를 가진다고 생각할 수 있다.

- selected sample / candidate identity
- accumulated weight 또는 UCW 관련 상태
- `M`, age, history metadata
- visibility / material 관련 metadata
- 재사용할 **subpath reference**

반면 subpath는 1개 vertex일 수도 있고 6개 vertex일 수도 있다. vertex마다 position, normal, throughput, PDF, material/BSDF state, random replay 정보 등을 보관하면 크기는 빠르게 커진다.

따라서 GPU에서는 흔히 개념적으로 다음처럼 분리하는 편이 자연스럽다.

```text
Reservoir Buffer
+-------------------------------+
| weight | M | age | subpathRef |
+-------------------------------+
             |
             v
Indirection / Handle Table
+-------------------------------+
| slot | generation | poolBase  |
+-------------------------------+
             |
             v
Persistent Subpath Pool
+--------------------------------------------------+
| vertex data | vertex data | ... | vertex data   |
+--------------------------------------------------+
```

즉, 오늘의 주제는 **persistent pool + stable handle + temporal lifetime + compaction**의 결합이다.

여기서 중요한 점은 이 문제가 단순한 allocator 문제가 아니라는 것이다. ReSTIR/GRIS에서 reservoir가 참조하는 sample은 확률적 provenance를 가진다. 잘못된 pool entry를 읽으면 단순히 색이 조금 틀리는 것이 아니라 **다른 확률 분포에서 생성된 sample을 현재 estimator의 sample이라고 착각하는 것**이 된다.

따라서 다음 두 provenance가 일치해야 한다.

\[
\text{Probability Provenance}
\quad \leftrightarrow \quad
\text{Memory Provenance}
\]

- Probability provenance: 이 sample이 어떤 domain, conditioning state, proposal/reuse 과정에서 왔는가?
- Memory provenance: 현재 handle이 실제로 그 sample의 subpath payload를 가리키고 있는가?

이 둘이 깨지면 memory safety가 유지되어도 estimator correctness는 깨질 수 있다.

---

## 2. 한 줄 핵심

> **Temporal reservoir가 variable-length subpath를 안전하게 재사용하려면 raw pool index가 아니라 lifetime이 검증되는 stable indirection을 사용하고, compaction·frame-in-flight·history invalidation을 하나의 ownership 문제로 다뤄야 한다.**

---

## 3. 왜 중요한가

### 3.1 Reservoir를 크게 만들면 GPU 효율이 빠르게 무너진다

Reservoir는 화면 해상도에 비례해 매우 많이 존재할 수 있다.

1920×1080만 해도 약 207만 pixel이다. 각 pixel에 reservoir 하나만 있어도 header가 32 bytes라면 약 63 MiB 수준이다.

\[
1920 \times 1080 \times 32\ \text{bytes}
\approx 63.3\ \text{MiB}
\]

여기에 variable-length path vertex를 reservoir 안에 inline으로 넣기 시작하면 메모리는 바로 폭증한다.

예를 들어 평균 4개 vertex × 48 bytes가 추가되면 reservoir 하나가 약 224 bytes가 된다.

\[
1920 \times 1080 \times 224\ \text{bytes}
\approx 443\ \text{MiB}
\]

Temporal double buffering, multiple reservoir passes, visibility/history buffers까지 생각하면 현실적인 real-time budget에서는 부담이 크다.

그래서 **작은 fixed-size reservoir header와 큰 variable-size path payload를 분리**하는 것이 중요하다.

### 3.2 Temporal reuse 때문에 pool은 한 dispatch의 임시 메모리가 아니다

일반 compute pass의 scratch buffer라면 dispatch가 끝난 뒤 내용을 버려도 된다. 하지만 temporal ReSTIR에서는 frame `N`의 reservoir가 frame `N-1`, `N-2`에서 생성된 sample provenance를 참조할 수 있다.

즉 subpath pool의 lifetime은 단순히

```text
kernel start -> kernel end
```

이 아니라

```text
sample creation
    -> temporal reuse
    -> possible spatial reuse
    -> final gather / shading
    -> no surviving reservoir references
    -> GPU completion
    -> reclaim
```

까지 이어진다.

CUDA의 global/device memory가 kernel launch 사이에서도 allocation이 해제되기 전까지 지속된다는 점은 이런 persistent state를 담는 기반이 된다. 하지만 **메모리가 지속된다는 것과 sample이 논리적으로 살아 있다는 것은 전혀 다른 문제**다.

### 3.3 Stale handle은 crash보다 더 위험할 수 있다

다음 상황을 생각해보자.

```text
Frame N
reservoir.subpathIndex = 1042
pool[1042] = path A

Frame N+1
path A retired
pool[1042] recycled
pool[1042] = path B

Frame N+2
old reservoir still references 1042
```

GPU 입장에서는 `pool[1042]`가 정상 주소다. access violation도 없고 값도 finite할 수 있다.

하지만 estimator 입장에서는 path A를 재사용한다고 믿으면서 path B를 읽고 있다.

이런 오류는 다음처럼 보일 수 있다.

- 드문 firefly
- 특정 카메라 각도에서만 나타나는 flicker
- temporal boiling
- 갑작스러운 luminance spike
- denoiser가 제거하지 못하는 structured noise
- scene update 뒤에만 나타나는 stochastic artifact

그래서 **generation counter, pool epoch, history version** 같은 검증 정보가 중요해진다.

### 3.4 Compaction은 성능 문제이면서 reference stability 문제다

Variable-length pool에는 시간이 지나면서 hole이 생긴다.

```text
[A][dead][C][dead][dead][F][G][dead]
```

이 상태를 그대로 두면 capacity 대비 live data 비율이 낮아진다.

간단한 fragmentation 지표는 다음처럼 생각할 수 있다.

\[
F = 1 - \frac{N_{live}}{N_{capacity}}
\]

`F`가 커지면 메모리를 많이 잡아두면서 실제로 사용하는 payload는 적다.

하지만 payload를 조밀하게 옮기는 compaction을 하면 index가 바뀔 수 있다.

```text
Before
A -> slot 0
C -> slot 2
F -> slot 5

After
A -> slot 0
C -> slot 1
F -> slot 2
```

reservoir가 raw index `5`를 들고 있었다면 이제 잘못된 reference다.

따라서 **payload location과 logical sample identity를 분리하는 indirection layer**가 중요하다.

### 3.5 최신 ReSTIR 연구도 알고리즘보다 engineering을 무시할 수 없음을 보여준다

NVIDIA의 2026년 **ReSTIR PT Enhanced**는 ReSTIR PT의 theoretical extension보다 실제 algorithmic/engineering optimization에 집중해 기존 방식보다 2–3× 빠른 실행과 더 나은 robustness를 보고한다. 이는 modern ReSTIR renderer에서 estimator 수식만큼 **memory layout, reuse cost, correlation control, execution structure**가 중요하다는 좋은 사례다.

Conditional ReSTIR의 original prototype도 Falcor 위에서 실제 rendering component로 구성되어 있으며, subpath reuse가 이론에서 끝나는 것이 아니라 DXR 기반의 상당히 복잡한 renderer state로 이어진다는 점을 보여준다.

---

## 4. 구현 관점

### 4.1 가장 먼저 분리해야 하는 두 종류의 상태

GPU data structure를 생각할 때 가장 중요한 것은 **logical state**와 **physical storage**를 분리하는 것이다.

#### Logical reservoir state

```cpp
struct ReservoirHeaderConcept {
    float weight;
    uint  M;
    uint  age;
    Handle subpath;
    uint  historyEpoch;
};
```

여기서 `Handle`은 "GPU address" 자체가 아니라 **논리적 sample reference**에 가깝다.

#### Physical subpath storage

```text
position[]
normal[]
throughput[]
pdf[]
materialID[]
flags[]
...
```

이 둘을 분리하면 reservoir header는 compact하게 유지하면서, variable-length payload는 별도 pool에서 관리할 수 있다.

핵심은 다음 식으로 표현할 수 있다.

\[
\text{Reservoir}
\rightarrow
\text{Logical Handle}
\rightarrow
\text{Physical Pool Location}
\]

직접

\[
\text{Reservoir} \rightarrow \text{Physical Array Index}
\]

로 연결하는 것보다 compaction과 lifetime 관리가 유연하다.

### 4.2 Stable handle: index 하나로는 부족하다

가장 단순한 handle은 pool index다.

```text
handle = 1042
```

하지만 slot reuse가 있으면 ABA 문제가 생긴다.

```text
1042 -> A
1042 freed
1042 -> B
```

old reference와 new reference가 같은 숫자를 갖기 때문이다.

그래서 일반적인 generational-handle 개념을 적용하면 다음처럼 볼 수 있다.

\[
H = (index, generation)
\]

slot metadata에는 현재 generation을 둔다.

\[
Valid(H)
=
(H.generation = Slot[H.index].generation)
\]

Temporal renderer에서는 여기에 frame/history epoch를 더할 수 있다.

\[
Valid(H)
=
G_{handle}=G_{slot}
\land
E_{handle}=E_{history}
\]

여기서 중요한 것은 bit packing 자체가 아니라 **logical identity와 physical slot reuse를 구분하는 계약(contract)**이다.

### 4.3 Handle table을 두면 compaction이 쉬워진다

한 단계 더 나아가 reservoir가 payload slot을 직접 가리키지 않고 handle table entry를 가리키게 할 수 있다.

```text
Reservoir
   |
   v
Handle ID 73
   |
   v
HandleTable[73] = { poolOffset = 9821, generation = 14 }
   |
   v
Subpath Pool
```

Compaction으로 subpath가 `9821 -> 4102`로 이동해도 handle table의 physical offset만 수정하면 reservoir는 그대로 유지된다.

장점:

- reservoir rewrite가 필요하지 않을 수 있음
- compaction과 logical identity 분리
- generation validation 위치가 명확함
- debugging이 쉬움

단점:

- 추가 global-memory indirection
- cache miss 가능성
- handle table 자체의 bandwidth
- random reservoir access 시 memory divergence

즉 stable indirection은 correctness를 쉽게 만들지만 공짜는 아니다.

### 4.4 Multi-frame lifetime: CPU의 free 시점과 GPU의 free 시점은 다르다

Explicit graphics API에서 가장 위험한 착각 중 하나는 CPU가 더 이상 resource를 사용하지 않는 순간 GPU도 사용을 끝냈다고 보는 것이다.

Frame-in-flight가 여러 개라면 다음과 같은 상황이 가능하다.

```text
CPU frame:  N+2
GPU frame:  N
```

CPU는 frame `N`의 reservoir가 끝났다고 생각해 pool slot을 recycle했지만 GPU가 아직 그 slot을 읽는 중일 수 있다.

따라서 reclaim 조건은 개념적으로 다음과 같아야 한다.

\[
\text{Reclaimable}
=
\text{NoLogicalReference}
\land
\text{GPUCompleted}
\]

Temporal renderer에서는 첫 항도 생각보다 어렵다. 단순히 current frame에서 reference count가 0이라는 것만으로는 부족할 수 있다. 이전 reservoir history가 다음 frame에 재사용될 가능성이 있기 때문이다.

그래서 resource lifetime을 다음 세 단계로 구분하면 사고가 명확하다.

1. **Live**: reservoir에서 참조 가능
2. **Retired**: 새 reference는 만들지 않지만 아직 GPU가 읽을 수 있음
3. **Reclaimable**: 모든 relevant queue/fence가 완료되어 재사용 가능

이 구조는 C++ resource manager의 deferred destruction과 매우 비슷하다.

### 4.5 Pool allocation 전략: append, slab, free-list, ring

Persistent subpath pool은 여러 방식으로 구성할 수 있다.

#### Append-only frame slab

frame마다 큰 contiguous region을 할당하고 끝에서부터 append한다.

장점:

- atomic allocation이 단순함
- sequential write가 가능
- locality가 좋음
- generation 관리가 쉬움

단점:

- 오래 살아남는 subpath가 있으면 slab 전체 회수가 늦어질 수 있음
- temporal history가 길면 memory peak가 커질 수 있음

#### Free-list pool

dead entry를 개별적으로 다시 사용한다.

장점:

- memory reuse가 빠름
- peak capacity를 줄일 수 있음

단점:

- allocation contention
- random slot reuse
- fragmentation
- stale-handle 검증 필수

#### Ring / epoch pool

여러 frame epoch를 ring 형태로 유지한다.

```text
Epoch N-2 | Epoch N-1 | Epoch N | writing N+1
```

장점:

- lifetime reasoning이 단순해질 수 있음
- frame history 제한이 명확함

단점:

- unusually long-lived sample을 처리하기 어려움
- history length 정책과 allocator 정책이 강하게 결합됨

실제 renderer에서는 하나의 방식보다 **frame slab + stable handle + occasional compaction** 같은 hybrid가 더 자연스러울 수 있다.

### 4.6 Variable-length subpath의 compact representation

Subpath payload를 생각할 때 가장 중요한 질문은 "vertex 하나가 무엇을 반드시 기억해야 하는가?"다.

Full shading state를 저장하면 비싸다.

```text
world position
geometric normal
shading normal
throughput RGB
forward PDF
reverse PDF
roughness
material ID
instance ID
primitive ID
RNG state
...
```

반대로 너무 적게 저장하면 replay 시 다시 geometry/material 정보를 읽거나 ray trace를 해야 한다.

그래서 memory와 recomputation 사이의 trade-off가 생긴다.

\[
T_{frame}
=
T_{memory\ fetch}
+
T_{recompute}
+
T_{trace}
+
T_{sync}
\]

GPU에서는 arithmetic을 줄이는 것보다 bandwidth를 줄이는 편이 더 유리한 경우도 많다. 특히 path state가 큰데 일부 field만 읽는 pass가 많다면 **Structure of Arrays (SoA)**가 강해진다.

예를 들어 visibility pass가 position과 flags만 필요하다면 AoS에서는 사용하지 않는 throughput, PDF, material state까지 같은 cache line에 끌고 올 수 있다.

### 4.7 AoS / SoA / AoSoA를 subpath pool 관점에서 보기

#### AoS

```text
Vertex0 {P, N, T, pdf, flags}
Vertex1 {P, N, T, pdf, flags}
Vertex2 {P, N, T, pdf, flags}
```

장점:

- vertex 단위 전체 접근이 편함
- CPU debugging과 serialization이 쉬움

단점:

- shader마다 필요한 field가 다르면 overfetch가 큼

#### SoA

```text
P[]
N[]
T[]
pdf[]
flags[]
```

장점:

- 같은 field를 wave 전체가 읽을 때 coalescing에 유리
- pass-specific bandwidth를 줄이기 쉬움

단점:

- 하나의 vertex 전체 state가 필요하면 load instruction이 많아질 수 있음
- variable-length path indexing이 복잡함

CUDA Programming Guide가 강조하듯 global-memory access는 warp 단위로 가능한 한 적은 transaction으로 coalesce될수록 좋다. Persistent subpath pool처럼 데이터가 큰 구조에서는 **어떤 layout이 cache line을 가장 잘 쓰는가**가 직접 frame time으로 이어질 수 있다.

#### AoSoA

여러 vertex를 작은 block 단위로 묶어 field별로 정렬한다.

```text
Block0:
  P[0..7]
  N[0..7]
  T[0..7]
Block1:
  ...
```

wave/subgroup 단위 locality와 vertex-local access 사이의 중간 지점이다.

### 4.8 Compaction: 언제 해야 하는가

Compaction은 공짜가 아니다.

일반적으로 다음 비용이 들어간다.

1. live/dead mark
2. prefix sum / scan
3. new offset 생성
4. payload scatter
5. handle table remap
6. synchronization

따라서 매 frame 무조건 compact하는 것은 좋지 않을 수 있다.

개념적으로는 다음 비교가 필요하다.

\[
C_{compact}
<
C_{fragmentation\ over\ future\ frames}
\]

fragmentation의 실제 비용은 단순 wasted bytes가 아니다.

- larger working set
- cache miss 증가
- 더 큰 allocation 유지
- random free-list reuse
- memory-pressure로 다른 render resource가 밀림

따라서 threshold는 `dead slot 비율` 하나보다 **working-set 크기와 future reuse horizon**을 함께 보는 편이 맞다.

### 4.9 Compaction에서 가장 중요한 규칙: identity를 옮기지 말고 storage를 옮긴다

좋은 conceptual model은 다음과 같다.

```text
Sample Identity = stable
Storage Location = movable
```

즉 compaction은 sample identity를 바꾸는 작업이 아니라 sample payload의 physical location만 바꾸는 작업이어야 한다.

이 원칙을 따르면 reservoir는 다음을 기억한다.

```text
logical sample handle
```

handle table은 다음을 기억한다.

```text
current physical storage location
```

payload pool은 다음을 담는다.

```text
actual path vertex data
```

이 세 층을 분리하면 memory manager와 estimator state의 경계가 훨씬 선명해진다.

### 4.10 History invalidation과 pool epoch

Temporal path state는 여러 이벤트에서 한꺼번에 무효화될 수 있다.

- resolution change
- render mode change
- incompatible material table rebuild
- scene topology replacement
- TLAS/BLAS 재구성으로 identity contract가 바뀐 경우
- camera cut로 temporal mapping policy를 reset하는 경우
- reservoir algorithm 설정 변경

이때 reservoir header만 clear하고 pool handle을 남겨두면 stale state가 섞일 수 있다.

그래서 전체 temporal system에 epoch를 둘 수 있다.

\[
E_{current} \leftarrow E_{current} + 1
\]

그리고

\[
Valid(H) \Rightarrow H.epoch = E_{current}
\]

처럼 현재 history generation과 맞는 reference만 인정한다.

여기서 epoch는 메모리를 즉시 zero-clear하기 위한 도구라기보다 **old history와 new history를 논리적으로 분리하는 cheap invalidation mechanism**이다.

### 4.11 Debugging에서 반드시 분리해 볼 지표

Stochastic renderer의 memory bug는 image만 보고 찾기 어렵다. 다음 지표를 독립적으로 보면 좋다.

#### Correctness counters

- invalid-generation handle count
- invalid-epoch reference count
- out-of-range pool offset count
- retired-slot access count
- duplicate logical handle count

#### Memory counters

- live payload bytes
- allocated payload bytes
- fragmentation ratio
- compaction bytes moved
- handle-table lookup count
- average subpath length

#### GPU counters

- global-memory throughput
- L2 hit rate
- wave occupancy
- register usage
- divergent branch rate
- active-ray count

#### Rendering-quality counters

- temporal luminance variance
- reservoir age distribution
- effective unique sample count
- rejected temporal reuse ratio

이 네 범주를 같이 보면 "noise가 algorithmic variance인지 memory-lifetime bug인지"를 분리하기 쉬워진다.

### 4.12 C++ Render Graph에서의 ownership

C++ engine에서는 subpath pool을 단순 transient buffer로 등록하기보다 **temporal persistent resource**로 보는 것이 맞다.

개념적으로 다음 ownership이 존재한다.

```text
Renderer
  ├── ReservoirHistory[N]
  ├── ReservoirHistory[N-1]
  ├── SubpathPool
  ├── HandleTable
  └── RetiredSlotQueue
```

Render graph는 pass dependency를 표현하지만, 여러 frame에 걸친 lifetime까지 자동으로 이해하지 못하는 경우가 많다. 따라서 엔진 레이어에는 별도로 다음 contract가 필요하다.

- 어떤 frame이 pool entry를 생성했는가
- 어떤 history buffer가 이를 참조할 수 있는가
- 어떤 GPU queue가 마지막으로 이를 읽는가
- 언제 retired 상태로 이동하는가
- 어떤 fence completion 이후 reclaim 가능한가

이것은 Vulkan/D3D12에서 descriptor/resource lifetime을 다루는 것과 동일한 사고방식이지만, ReSTIR에서는 한 단계 더 나아가 **sampling correctness까지 ownership에 묶인다.**

---

## 5. 내 관심 분야와 연결

### C++ / Vulkan / DirectX 12

이 주제는 explicit graphics API를 공부할 때 흔히 만나는 resource-lifetime 문제를 훨씬 더 깊게 만든다.

보통 stale descriptor나 freed buffer를 잘못 참조하면 GPU validation error나 crash를 기대한다. 그러나 reservoir indirection에서는 **유효한 memory address인데 의미상 틀린 sample**을 읽는 문제가 더 중요하다.

즉 C++ 레이어의 관심사가

```text
memory valid?
```

에서

```text
memory valid?
AND
sample identity valid?
AND
history provenance valid?
```

로 확장된다.

### GPU compute / GPGPU

Pool compaction은 GPU compute의 대표적인 building block과 직접 연결된다.

- predicate generation
- prefix sum / scan
- scatter
- active-list generation
- indirect dispatch arguments

이런 연산은 particle simulation, sparse voxel processing, stream compaction에서도 반복해서 등장한다.

따라서 ReSTIR에서 배운 pool-management 관점은 graphics-only 지식이 아니라 **general GPU data-oriented design**으로 확장된다.

### Simulation / Scientific Visualization

대규모 CFD/volume/particle 데이터에서도 비슷한 문제가 있다.

- sparse active cell
- particle death / birth
- chunk relocation
- LOD block pool
- sparse voxel brick pool

특히 sparse voxel이나 particle buffer에서 raw index를 외부에서 장기간 보관하면 compaction 후 reference가 깨진다. generation handle과 indirection은 이런 simulation data structure에서도 동일하게 등장한다.

### Rendering pipeline

Traditional raster renderer에서는 많은 resource가 frame-local이다. 하지만 modern path-traced renderer에서는 reservoir, denoiser history, visibility cache, path state가 모두 multi-frame temporal system을 만든다.

그래서 pipeline을 다음처럼 보는 편이 더 정확하다.

\[
\text{Sampling}
\rightarrow
\text{Persistent State}
\rightarrow
\text{Reuse}
\rightarrow
\text{Shading}
\rightarrow
\text{Denoising}
\]

Persistent state가 틀리면 뒤의 모든 stage가 정상이어도 결과가 흔들린다.

### Graphics Engineer 관점

이 주제의 가장 큰 의미는 "좋은 알고리즘을 GPU에 올린다"와 "production renderer로 만든다" 사이의 차이를 보여준다는 점이다.

Production 수준에서는 다음 질문을 피할 수 없다.

- sample은 어디에 저장되는가?
- 몇 frame 살아 있는가?
- 누가 ownership을 갖는가?
- compaction 후 identity는 어떻게 유지되는가?
- GPU가 아직 읽고 있을 때 누가 free를 막는가?
- memory layout이 wave access pattern과 맞는가?
- history reset이 estimator와 resource layer에서 동시에 일어나는가?

이 질문에 답할 수 있어야 algorithm paper를 engine architecture로 번역할 수 있다.

---

## 6. 머릿속에 남길 질문 3개

1. **Reservoir가 raw pool index를 직접 저장하는 방식과 stable handle table을 거치는 방식은 compaction correctness, cache locality, bandwidth에서 각각 어떤 trade-off를 만들까?**

2. **Frame `N-2`에서 만들어진 subpath를 frame `N`의 reservoir가 아직 참조할 수 있고 GPU에는 여러 frame이 in-flight라면, pool entry가 실제로 reclaimable하다고 판단하기 위해 어떤 logical lifetime과 fence state가 함께 필요할까?**

3. **Fragmentation이 높아졌을 때 무조건 compaction하는 것이 아니라, working-set size·future reuse horizon·bytes moved·cache behavior까지 고려해야 하는 이유는 무엇일까?**

---

## 7. graphics engineer 면접 질문 1개와 답변

### Q. ReSTIR/GRIS renderer에서 variable-length subpath를 reservoir 구조체 안에 직접 저장하지 않고 별도 persistent pool + indirection으로 분리하는 이유는 무엇이며, pool compaction 시 어떤 correctness 문제가 생길 수 있는가?

**A.** 가장 큰 이유는 **fixed-size hot state와 variable-size cold payload를 분리해 memory footprint와 access pattern을 제어하기 위해서**다.

Reservoir는 화면 해상도만큼 많이 존재하고 temporal/spatial reuse pass에서 반복적으로 읽힌다. 따라서 weight, `M`, age, sample metadata처럼 자주 접근하는 값은 작은 fixed-size header에 두는 편이 cache와 bandwidth에 유리하다. 반면 path vertex 전체를 inline 저장하면 reservoir stride가 커지고, 실제로 사용하지 않는 field까지 읽게 되어 메모리 비용이 크게 증가한다.

Subpath를 별도 pool에 두면 reservoir는 작은 logical handle만 보유할 수 있다. 하지만 이때 새로운 correctness 문제가 생긴다.

Pool entry를 compaction하거나 recycle하면 physical index가 바뀔 수 있다. Reservoir가 raw index를 들고 있으면 old index가 다른 sample을 가리키는 stale reference가 될 수 있다. 이 오류는 GPU address 자체는 valid일 수 있기 때문에 crash 없이 stochastic artifact로 나타날 수 있다.

따라서 production 설계에서는 보통 다음 개념이 필요하다.

- stable logical handle
- generation/version validation
- optional handle-to-physical-offset indirection
- history epoch
- retired state와 deferred reclaim
- GPU fence/queue completion 확인

핵심은 **sample identity와 storage location을 분리하는 것**이다.

좋은 답변은 여기서 한 단계 더 나아가 다음을 언급할 수 있다.

> ReSTIR에서는 stale handle이 단순 memory bug가 아니라 estimator bug다. reservoir의 weight와 provenance는 path A에 대한 것인데 pool에서 path B를 읽으면, 유효한 메모리를 읽더라도 확률적으로 잘못된 sample을 평가하게 된다.

그리고 performance 관점에서는 indirection이 추가 global-memory access를 만들기 때문에 SoA/AoSoA, handle-table locality, compaction frequency까지 함께 측정해야 한다.

---

## 8. 포트폴리오 / 커리어 연결

이 주제는 포트폴리오에서 **renderer algorithm을 systems architecture로 번역할 수 있는 능력**을 보여주기 좋다.

단순히 "ReSTIR를 이해한다"가 아니라 다음 계층을 하나의 설계로 설명할 수 있다.

### 1) Estimator layer

- reservoir provenance
- temporal reuse
- subpath reuse
- sample identity
- invalid-history rejection

### 2) Memory-system layer

- fixed-size reservoir header
- variable-size payload pool
- stable handle
- generation / epoch
- fragmentation / compaction

### 3) GPU execution layer

- coalesced access
- SoA / AoSoA
- scan / scatter
- active-list compaction
- cache locality
- occupancy와 register pressure

### 4) C++ engine layer

- frame-in-flight
- GPU fence
- deferred free
- render-graph persistent resource
- resize / scene-update invalidation

포트폴리오에서 특히 강한 설명은 "자료구조를 만들었다"보다 **왜 이 구조가 필요한지 trade-off를 수치로 설명하는 것**이다.

예를 들면 다음 metric이 의미 있다.

- bytes / reservoir
- bytes / live subpath
- average / p95 subpath length
- live-to-capacity ratio
- invalid-generation rejection count
- compaction bytes moved
- handle-table bandwidth
- L2 hit rate
- frame time before/after compaction
- temporal artifact rate 또는 variance 변화

그리고 architecture diagram에서는 다음 흐름이 명확하면 좋다.

```text
Temporal Reservoir
      |
      v
Stable Handle
      |
      v
Handle Table
      |
      v
Persistent Subpath Pool
      |
      +--> Compaction
      |
      +--> Deferred Reclaim
      |
      +--> Wavefront Queues
```

이 수준의 설명은 graphics programmer가 단순 shader author가 아니라 **C++ resource system, GPU memory architecture, Monte Carlo estimator correctness를 함께 다룰 수 있음**을 보여준다.

2026년 ReSTIR PT Enhanced가 theoretical novelty뿐 아니라 implementation optimization으로 2–3×의 성능 향상을 보여준 것도 이 관점과 잘 맞는다. 실제 graphics engineering에서는 알고리즘의 이름보다 **어떤 state가 hot한지, 어떤 work가 redundant한지, 어떤 data dependency가 cache와 correlation을 망치는지**를 찾아내는 능력이 중요하다.

---

## 9. 내일 이어서 볼 개념

**Wavefront Queue Scheduling and Path-State Binning: Divergence Control for Irregular ReSTIR Workloads**

오늘은 variable-length path state를 **어디에, 얼마나 오래, 어떤 identity로 저장할 것인가**를 다뤘다. 다음에는 그 state를 가진 path/ray를 **GPU에서 어떤 순서로 실행시킬 것인가**로 넘어간다.

이어질 핵심 연결점은 다음과 같다.

- megakernel path tracing vs wavefront path tracing
- active-ray queue
- material / path-depth binning
- queue compaction
- persistent threads
- indirect dispatch
- ray divergence
- shader execution reordering과 유사한 사고방식
- reservoir reuse pass와 visibility queue 분리
- path-state SoA와 queue index의 관계
- queue locality와 subpath pool locality
- queue 크기, occupancy, synchronization 사이 trade-off

즉 오늘이 **memory provenance**였다면 내일은 **execution coherence**다.

---

## 10. 참고 키워드

### 핵심 용어

- Persistent Subpath Pool
- Path Vertex Pool
- Reservoir Indirection
- Stable Handle
- Generational Handle
- Generation Counter
- Pool Epoch
- History Epoch
- Sample Provenance
- Memory Provenance
- Stale Handle
- ABA Problem
- Deferred Reclaim
- Deferred Destruction
- Frame-in-Flight
- GPU Fence
- Resource Lifetime
- Temporal Resource
- Variable-Length Payload
- Fixed-Size Header
- Structure of Arrays (SoA)
- Array of Structures (AoS)
- Array of Structures of Arrays (AoSoA)
- Global Memory Coalescing
- Working Set
- Fragmentation
- Stream Compaction
- Prefix Sum / Scan
- Scatter
- Free List
- Slab Allocator
- Ring Buffer
- Handle Table
- Physical Offset
- Logical Identity
- Wavefront Path Tracing
- Active-Ray Queue
- ReSTIR
- GRIS
- Conditional GRIS
- Conditional ReSTIR
- Joint UCW
- Final Gather

### 관련 자료

- [Conditional Resampled Importance Sampling and ReSTIR — NVIDIA Research, SIGGRAPH Asia 2023](https://research.nvidia.com/labs/rtr/publication/kettunen2023conditional/)
- [Generalized Resampled Importance Sampling: Foundations of ReSTIR — NVIDIA Research, SIGGRAPH 2022](https://research.nvidia.com/labs/rtr/publication/lin2022generalized/)
- [Conditional ReSTIR Prototype — NVlabs GitHub](https://github.com/NVlabs/conditional-restir-prototype)
- [ReSTIR PT Enhanced: Algorithmic Advances for Faster and More Robust ReSTIR Path Tracing — NVIDIA Research, 2026](https://research.nvidia.com/labs/rtr/publication/lin2026restirptenhanced/)
- [CUDA Programming Guide — GPU Memory and Coalesced Global Memory Access](https://docs.nvidia.com/cuda/cuda-programming-guide/)
