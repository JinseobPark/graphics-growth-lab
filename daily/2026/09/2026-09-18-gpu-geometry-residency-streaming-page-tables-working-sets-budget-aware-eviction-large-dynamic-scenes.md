---
title: "GPU Geometry Residency and Streaming: Page Tables, Working Sets, and Budget-Aware Eviction for Large Dynamic Scenes"
date: "2026-09-18"
category: Graphics
tags: [GPU, Rendering, GPU-Driven Rendering, Geometry Streaming, Residency, Sparse Resources, Working Set, Eviction, Memory Budget, Vulkan, CUDA, Meshlet, Dynamic Geometry, Memory Layout, C++]
level: intermediate
---

# [Daily Graphics Growth] 2026-09-18 - GPU Geometry Residency and Streaming: Page Tables, Working Sets, and Budget-Aware Eviction for Large Dynamic Scenes

## 1. 오늘의 개념

어제는 **Incremental GPU Snapshot Publication**에서 큰 dynamic geometry를 full double/triple buffer하지 않고, `logical chunk → physical allocation` mapping을 immutable snapshot으로 versioning하고 changed chunk만 Copy-on-Write(COW)하는 구조를 봤다.

오늘은 그 다음 한계로 간다.

> **Persistent payload pool 자체가 GPU memory budget보다 커지면, 어떤 geometry를 resident하게 유지하고 어떤 geometry를 evict/stream해야 하는가?**

큰 scene에서는 모든 geometry가 동시에 GPU-local memory에 있을 필요가 없다.

```text
Virtual Geometry Space
        ↓
Residency Table
        ↓
Resident Working Set
        ↓
GPU-local Payload Pool
```

이 구조의 핵심은 geometry를 두 가지 관점으로 분리하는 것이다.

```text
Logical existence
≠
Physical residency
```

어떤 chunk가 scene graph에 존재한다고 해서 그 chunk의 vertex/index/meshlet payload가 지금 VRAM에 있어야 하는 것은 아니다.

반대로 resident memory에 있다고 해서 이번 frame에 반드시 render된다는 뜻도 아니다.

그래서 오늘은 다음 네 층을 분리한다.

1. **Visibility / Importance**
   - 앞으로 필요한 geometry가 무엇인가.

2. **Residency**
   - 그 geometry가 현재 GPU-accessible physical memory에 올라와 있는가.

3. **Streaming**
   - missing geometry를 언제 어떤 budget으로 올릴 것인가.

4. **Eviction**
   - memory pressure가 생겼을 때 무엇을 안전하게 내릴 것인가.

학습 흐름은 다음과 같다.

```text
Immutable Snapshot
    ↓
Logical Geometry ID
    ↓
Residency Mapping
    ↓
Resident Payload
    ↓
Visibility / Rendering
```

오늘의 핵심 질문은:

> **Visibility가 낮은 geometry를 evict하면 되는가, 아니면 in-flight snapshot·future visibility·streaming latency·LOD fallback까지 함께 고려해야 하는가?**

정답은 후자다.

---

## 2. 한 줄 핵심

> GPU geometry residency의 핵심은 **“현재 안 보이는 geometry”를 지우는 것이 아니라, logical geometry와 physical residency를 분리하고, working-set priority·in-flight pinning·streaming latency·memory budget을 함께 사용해 resident set을 안정적으로 유지하는 것**이다.

---

## 3. 왜 중요한가

Dynamic scientific scene이나 large-world renderer는 geometry가 VRAM보다 커질 수 있다.

예:

```text
Geometry payload total = 24 GB
GPU-local budget       = 12 GB
Current visible set    = 4 GB
Likely-near-future set = 3 GB
```

이 경우 full residency는 불가능하지만 frame에 필요한 실제 working set은 충분히 작을 수 있다.

문제는 단순 LRU cache처럼 구현하면 rendering workload와 잘 맞지 않는다는 점이다.

### 3.1 Visibility와 residency는 시간축이 다르다

현재 frame에서 invisible인 geometry도 다음 frame에서 필요할 수 있다.

예:

- camera turn
- section plane 이동
- clipping mode 변경
- LOD refinement
- simulation front 이동
- occluder가 사라져 disocclusion 발생

즉:

```text
not visible now
≠
safe to evict
```

다.

### 3.2 Residency miss는 일반 cache miss보다 비싸다

CPU cache miss는 보통 lower cache/DRAM에서 가져온다.

Geometry residency miss는:

```text
host/system memory
→ transfer queue
→ device memory
→ metadata patch
→ visibility/render dependency
```

처럼 여러 단계가 필요할 수 있다.

Streaming latency가 수 frame에 걸릴 수도 있다.

따라서 residency policy는 단순 recency가 아니라 **future demand prediction**을 포함해야 한다.

### 3.3 Memory budget은 고정 VRAM size가 아니다

Vulkan의 `VK_EXT_memory_budget`은 heap별 `heapBudget`과 `heapUsage`를 제공한다.

중요한 점은 `heapBudget`이 단순 physical heap size가 아니라 **현재 process가 성능 저하나 allocation 실패 전에 사용할 수 있는 양에 대한 implementation-dependent estimate**라는 것이다.

다른 process나 OS activity에 따라 값이 바뀔 수 있다.

그래서:

```text
VRAM size = 24 GB
```

라고 해도 engine이 23.5 GB를 목표로 쓰는 것은 좋은 전략이 아니다.

Residency manager는:

```text
dynamic budget
```

을 기준으로 동작해야 한다.

---

## 4. 구현 관점

### 4.1 먼저 세 가지 주소 공간을 분리한다

#### Logical Geometry Space

예:

```text
ChunkID
MeshHandle
BrickID
```

Scene에서 geometry의 identity다.

#### Virtual / Snapshot Mapping

```text
ChunkID
→ ResidencyEntry
```

Renderer가 logical ID를 physical location으로 resolve하는 table이다.

#### Physical Memory

```text
VkDeviceMemory
buffer range
CUDA allocation
```

실제 resident payload다.

이 세 층을 분리해야 eviction이 logical deletion으로 번지지 않는다.

### 4.2 Residency Entry

개념적 metadata:

```text
ResidencyEntry {
    state
    physicalPage
    payloadVersion
    residentBytes
    lastUsedEpoch
    priority
    pinCountOrFence
    fallbackLOD
}
```

대표 state:

```text
NON_RESIDENT
REQUESTED
UPLOADING
RESIDENT
EVICT_PENDING
PINNED
```

Renderer가 `RESIDENT`가 아닌 entry를 어떻게 처리할지도 contract에 포함되어야 한다.

### 4.3 Residency Miss의 세 가지 대응

Geometry가 필요한데 non-resident라면 다음 선택지가 있다.

#### Stall

업로드가 끝날 때까지 기다린다.

장점:
- 정확한 geometry

단점:
- frame hitch
- latency spike

#### Fallback LOD

낮은 LOD/proxy가 resident하면 그것을 사용한다.

장점:
- frame continuity 유지

단점:
- quality degradation

#### Skip / Deferred Visibility

이번 frame에는 render하지 않고 request만 생성한다.

장점:
- 가장 단순

단점:
- geometry pop-in
- correctness requirement에 따라 불가능할 수 있음

실시간 renderer에서는 보통 **resident fallback**이 가장 강력하다.

### 4.4 “항상 resident한 Root Representation”

Large geometry streaming에서 매우 중요한 패턴은 최소 representation을 항상 resident하게 두는 것이다.

예:

```text
LOD 3 coarse proxy → always resident
LOD 2             → streamed
LOD 1             → streamed
LOD 0 full detail → streamed
```

이렇게 하면 miss가:

```text
missing geometry
```

가 아니라:

```text
temporarily coarser geometry
```

가 된다.

Virtualized geometry 시스템이 강한 이유 중 하나다.

### 4.5 Working Set

Resident set 전체보다 중요한 것은 **working set**이다.

Working set은 가까운 미래에 실제 접근될 가능성이 높은 geometry 집합이다.

입력 signal:

- current visibility
- camera velocity
- projected size
- previous visibility
- occlusion confidence
- screen-space error
- edit/simulation dirty region
- section plane proximity
- user interaction direction

즉 working set은 `lastUsedFrame` 하나로 설명하기 어렵다.

### 4.6 LRU가 부족한 이유

단순 LRU:

```text
oldest unused
→ evict
```

는 다음 상황에 약하다.

예:

- camera가 잠깐 벽을 보고 돌아옴
- 중요 geometry가 5 frame 동안 occluded
- 다음 frame에 다시 대규모 visible

LRU는 이 geometry를 evict할 수 있다.

Streaming latency가 3 frame이면 camera를 돌리는 순간 pop-in/stall이 발생한다.

따라서 GPU geometry eviction은 보통 **recency + expected future value**를 같이 본다.

### 4.7 Residency Priority

개념적 priority:

```text
priority
≈
visibilityProbability
× projectedImportance
× reloadCost
× latencySensitivity
× updateCost
```

Eviction candidate는 inverse score를 사용할 수 있다.

예:

```text
low visibility
low projected size
cheap reload
coarse fallback exists
not pinned
```

인 chunk가 좋은 eviction 후보가 된다.

### 4.8 Screen-Space Error와 Residency

LOD 시스템과 residency는 분리하면 비효율적이다.

예:

```text
desiredLOD = L0
L0 non-resident
L1 resident
```

이면 renderer는 L1을 그리고 L0 request를 만들 수 있다.

즉 residency manager는 다음 정보를 공유한다.

```text
desiredLOD
residentLOD
streamingLOD
fallbackLOD
```

이렇게 하면 geometry streaming이 render-quality policy와 직접 연결된다.

### 4.9 Visibility Request Queue

Culling pass는 non-resident geometry를 발견하면 render work 대신 residency request를 생성할 수 있다.

```text
Chunk visible
→ desired LOD non-resident
→ RequestQueue append
```

Request item:

```text
ResidencyRequest {
    logicalChunk
    desiredLOD
    priority
    geometryEpoch
}
```

중복 요청은 coalesce해야 한다.

같은 chunk가 수백 meshlet에서 request되면 global queue가 폭발할 수 있다.

### 4.10 Request Deduplication

대표 방법:

```text
requestBit[chunk]
```

또는:

```text
requestedEpoch[chunk]
```

를 사용한다.

첫 requester만 실제 queue append:

```text
if atomicCAS(requestedEpoch, old, current) succeeds:
    append
```

식으로 구성할 수 있다.

이전 dirty-brick coalescing과 같은 원리다.

### 4.11 Streaming Budget은 bytes/frame으로 본다

“한 frame에 100개 chunk upload”보다:

```text
max upload bytes / frame
max transfer time
max allocation bytes
```

가 더 의미 있다.

Chunk size가 크게 다르기 때문이다.

Streaming manager는:

```text
priority queue
→ budget cutoff
```

로 frame별 request 일부만 처리할 수 있다.

### 4.12 Upload Budget과 Rendering Bandwidth 경쟁

Transfer engine이 독립적이어도 physical memory bandwidth는 완전히 무료가 아니다.

Large upload가 동시에 발생하면:

- rendering bandwidth
- compute meshing
- texture streaming
- PCIe/system memory traffic

과 경쟁할 수 있다.

따라서 streaming budget은 단순 VRAM budget이 아니라 **bandwidth budget**도 포함한다.

### 4.13 Prefetch가 중요하다

현재 visible해진 뒤 request하면 이미 늦을 수 있다.

Prefetch signal:

- camera forward velocity
- camera angular velocity
- near-frustum margin
- predicted section-plane movement
- simulation front propagation
- neighbor LOD demand

예:

```text
currently not visible
but inside expanded predictive frustum
→ prefetch
```

이렇게 latency를 숨긴다.

### 4.14 Visibility Frustum과 Residency Frustum은 다를 수 있다

Render frustum:

```text
exact current visible region
```

Residency prefetch frustum:

```text
expanded / predicted region
```

즉 residency system은 render culling보다 더 넓은 spatial horizon을 본다.

```text
render set ⊂ residency working set
```

인 것이 일반적이다.

### 4.15 Occlusion Culling 결과를 Eviction에 바로 사용하면 위험하다

Hi-Z에서 occluded됐다고 즉시 residency priority를 크게 낮추면 다음 문제가 생긴다.

- temporary occluder
- camera movement
- disocclusion
- previous-frame HZB uncertainty

그래서 occlusion은:

```text
importance signal
```

이지:

```text
evict permission
```

이 아니다.

Temporal hysteresis를 residency에도 적용할 수 있다.

### 4.16 Residency Hysteresis

예:

```text
visible → priority immediate up
invisible → priority slowly down
```

로 만들 수 있다.

즉 load는 공격적으로, eviction은 보수적으로 한다.

이런 비대칭 hysteresis는 thrashing을 줄인다.

### 4.17 Thrashing

GPU memory가 거의 꽉 차고 working set이 budget보다 약간 크면:

```text
A load
→ B evict
→ camera moves
→ B load
→ A evict
```

가 반복될 수 있다.

이것이 residency thrashing이다.

증상:

- transfer traffic 급증
- allocation churn
- frame hitch
- LOD pop
- heapUsage는 비슷한데 upload bytes가 매우 큼

### 4.18 High-Water / Low-Water Memory Budget

Thrashing을 줄이기 위해 memory budget에 hysteresis를 둔다.

예:

```text
heapUsage > 90% budget
→ eviction mode

heapUsage < 80% budget
→ normal mode
```

숫자는 platform별 tuning 대상이다.

핵심은 budget 경계에서 policy가 frame마다 on/off되지 않게 하는 것이다.

### 4.19 Budget과 Physical Heap Size는 분리한다

`VK_EXT_memory_budget`의 `heapBudget`은 rough estimate이고 동적으로 변할 수 있다.

그래서 target:

```text
targetResidentBytes
=
min(engineConfiguredBudget,
    currentHeapBudget × safetyFactor)
```

처럼 생각할 수 있다.

OS나 다른 application의 memory pressure가 커지면 engine도 resident set을 줄여야 한다.

### 4.20 `VK_EXT_memory_priority`

Vulkan의 `VK_EXT_memory_priority`는 allocation에 priority를 부여해 implementation이 device-local memory를 유지할 때 참고할 수 있게 한다.

이것은 explicit eviction manager의 대체물이 아니다.

예:

```text
critical root LOD / frame tables
→ high priority

streaming fine LOD
→ medium

temporary scratch
→ lower
```

처럼 OS/driver memory management와 cooperate하는 hint다.

### 4.21 Sparse Resource를 쓰는 두 가지 이유

Vulkan sparse resources는 resource와 physical memory binding을 분리한다.

Sparse resource는:

- non-contiguous memory binding
- lifetime 중 re-binding
- partially resident resource

를 지원할 수 있다.

큰 virtual geometry buffer를 만들고 필요한 page만 physical memory에 bind하는 구조를 만들 수 있다.

### 4.22 `sparseBinding`과 `sparseResidency`의 차이

이 차이를 정확히 알아야 한다.

#### Sparse Binding

Resource를 여러 physical allocation에 non-contiguous하게 bind하고 rebind할 수 있다.

하지만 sparse residency feature가 없으면 resource 전체가 사용 전에 결국 bound되어야 한다.

#### Sparse Residency

일부 region이 unbound인 상태에서도 resource를 사용할 수 있다.

즉 **partially resident resource**가 가능하다.

Vulkan에서 `sparseResidencyBuffer`는 partially resident buffer 지원 여부를 나타낸다.

### 4.23 Sparse Buffer Granularity

Sparse buffer의 binding granularity는 `VkMemoryRequirements::alignment`에서 보고된다.

즉 application이 원하는 4 KB page가 아니라 hardware/implementation이 요구하는 sparse block 크기를 따라야 한다.

그래서 logical geometry chunk size가 sparse page granularity와 크게 맞지 않으면 internal fragmentation이 커질 수 있다.

### 4.24 Sparse Resource가 자동 Streaming System은 아니다

Sparse buffer를 만들었다고:

- visibility prediction
- page request
- eviction
- synchronization
- fallback LOD

가 자동으로 생기지 않는다.

Sparse API는 **virtual-to-physical binding primitive**일 뿐이다.

Application이 residency manager를 설계해야 한다.

### 4.25 Manual Page Table vs Vulkan Sparse Binding

두 architecture를 비교할 수 있다.

#### Manual Indirection

```text
LogicalChunk
→ PageTableBuffer
→ Normal VkBuffer allocation
```

장점:
- portability
- app-defined page size
- shader-visible metadata 자유도

단점:
- shader indirection
- address table 관리

#### Vulkan Sparse Resource

```text
large sparse VkBuffer
→ vkQueueBindSparse
→ physical memory pages
```

장점:
- 하나의 virtual resource address range
- binding만 교체 가능

단점:
- sparse feature/support 제약
- hardware binding granularity
- queue bind synchronization
- portability/complexity

실무적으로 둘을 혼합할 수도 있다.

### 4.26 Sparse Binding 변경은 Synchronization Event다

Vulkan specification은 sparse binding change가 실행되는 동안 다른 queue가 같은 memory range에 접근하지 않도록 synchronization해야 한다고 명시한다.

즉 page eviction/rebind는 단순 allocator bookkeeping이 아니다.

```text
render access
↔ sparse bind operation
```

사이에 queue synchronization이 필요하다.

### 4.27 Unbound Access Semantics를 가정하지 않는다

Sparse residency에서 non-resident region read behavior는 device property에 따라 defined zero 또는 undefined value일 수 있다.

따라서 geometry renderer는:

```text
“page 없으면 shader가 0을 읽겠지”
```

를 portable contract로 만들지 않는 편이 좋다.

Page table/residency bit를 먼저 확인하고 missing resource를 접근하지 않는 구조가 명확하다.

### 4.28 Page Table Entry

Manual virtual geometry:

```text
PageEntry {
    resident
    physicalPage
    generation
    lod
    lastUsedEpoch
}
```

Shader flow:

```text
Logical page
→ entry
→ if resident: fetch payload
→ else: fallback / request
```

이 indirection이 GPU-driven residency의 중심이다.

### 4.29 Device Address를 Page Table에 저장할 때

Vulkan Buffer Device Address나 CUDA pointer를 physical entry에 저장할 수 있다.

하지만 allocation relocation/eviction이 발생하면 address가 stale해진다.

그래서:

```text
logical page ID
→ current address table
```

을 late resolve하는 편이 persistent command/worklist에 raw address를 bake하는 것보다 안전하다.

이전 relocation-safe note의 원칙이 다시 등장한다.

### 4.30 Page Generation

Physical page slot이 재사용되면:

```text
page index 42
generation 7
```

같은 generation을 붙일 수 있다.

Stale worklist:

```text
page 42 generation 6
```

는 reject된다.

Object generation, snapshot epoch, page generation은 각각 다른 lifetime domain이다.

### 4.31 In-Flight Snapshot은 Eviction을 막는다

Snapshot E가 allocation A를 가리키고 Vulkan이 아직 E를 렌더 중이라면 A는 visibility가 없어도 evict/reuse할 수 없다.

즉 residency entry에는 conceptually:

```text
pinUntilTimeline
```

같은 상태가 필요하다.

Eviction candidate 조건:

```text
not needed soon
AND
not pinned by in-flight consumer
```

이다.

### 4.32 Pinning은 Reference Count가 아닐 수도 있다

Graphics pipeline에서는 exact refcount 대신:

```text
safeAfterTimelineValue
```

가 충분할 수 있다.

Allocation을 마지막으로 참조할 submission value를 저장한다.

```text
completedTimeline >= safeAfter
→ evictable/reclaimable
```

이전 epoch reclamation과 같은 구조다.

### 4.33 Resident와 Reclaimable은 다르다

Allocation A가 renderer에서 더 이상 참조되지 않아도 residency manager가 계속 resident하게 유지할 수 있다.

```text
Reclaimable
≠
Must Evict
```

Reclaimability는 correctness 상태,
Residency priority는 performance policy다.

이 둘을 분리해야 한다.

### 4.34 Evict Pending

Eviction은 다음 state를 가질 수 있다.

```text
RESIDENT
→ EVICT_CANDIDATE
→ EVICT_PENDING
→ NON_RESIDENT
```

Pending 동안:

- 새 snapshot이 reference하지 않도록 함
- old snapshot consumer completion 기다림
- sparse unbind/free 수행

즉 eviction도 작은 lifetime state machine이다.

### 4.35 COW Snapshot과 Residency의 상호작용

Snapshot E+1을 만들 때 target allocation이 non-resident라면 두 선택이 있다.

```text
1. resident request 후 publish
2. fallback mapping으로 publish
```

즉 snapshot publication과 streaming policy가 결합될 수 있다.

Renderer-facing snapshot은 항상 **access-safe mapping**을 가져야 한다.

### 4.36 Resident Fallback Mapping

예:

```text
desired:
Chunk 7 LOD0 → not resident

snapshot mapping:
Chunk 7 → resident LOD2
```

그리고 background request:

```text
load LOD0
```

완료 후 다음 snapshot에서:

```text
Chunk 7 → LOD0
```

로 COW patch한다.

이 구조는 streaming을 snapshot system 안으로 자연스럽게 통합한다.

### 4.37 Geometry Streaming과 LOD Streaming은 사실상 같은 문제로 수렴한다

Large scene에서 fine LOD만 non-resident하고 coarse LOD가 resident하다면:

```text
Residency
+
LOD selection
```

이 하나의 quality/resource optimization이 된다.

Priority는:

```text
screen-space error reduction per resident byte
```

로 볼 수 있다.

개념적 metric:

```text
Benefit/Byte
=
Projected Quality Gain
----------------------
Additional Resident Bytes
```

### 4.38 Fine LOD의 Eviction Value

Eviction은 chunk 전체를 날리는 것보다 fine LOD부터 제거할 수 있다.

```text
keep coarse base
evict high detail
```

이렇게 하면 pop-in이 아니라 gradual quality reduction이 된다.

Virtualized geometry systems의 핵심적인 안정성 패턴이다.

### 4.39 Working Set Prediction for Scientific Viewer

게임 camera 예측뿐 아니라 scientific viewer는 domain-specific signal을 활용할 수 있다.

예:

- 현재 slice plane 위치와 이동 방향
- material isolation mode
- selected device region
- simulation front velocity
- user orbit velocity
- cross-section box

이런 정보는 generic LRU보다 훨씬 좋은 prefetch signal이 될 수 있다.

### 4.40 Dynamic SDF에서 Simulation Front를 Prefetch Signal로 사용

Etch/deposition front가 이동하면 다음 timestep에 dirty가 될 brick을 어느 정도 예측할 수 있다.

따라서:

```text
future simulation work
→ future geometry residency
```

로 연결할 수 있다.

Simulation과 rendering streaming manager가 완전히 독립적일 필요가 없다.

### 4.41 Host Memory Tier

Evicted geometry를 완전히 버릴지 host memory에 cache할지 선택할 수 있다.

Tier:

```text
GPU Local
↓
Host/Pinned Cache
↓
Disk / Recompute
```

Dynamic simulation에서는 geometry를 저장하기보다 SDF에서 재생성하는 편이 더 쌀 수도 있다.

즉 reload source가:

```text
transfer
or
recompute
```

중 하나일 수 있다.

### 4.42 Transfer vs Recompute

Chunk payload 8 MB를 PCIe로 가져오는 것과 GPU에서 0.2 ms에 remesh하는 것을 비교할 수 있다.

Residency policy는 단순 memory cache가 아니라:

```text
Reload Cost =
min(Transfer Cost,
    Recompute Cost)
```

를 사용할 수 있다.

Procedural/dynamic geometry에서 매우 중요한 관점이다.

### 4.43 Recompute Cache

SDF가 resident하고 mesh payload만 evicted될 수 있다.

```text
SDF brick resident
mesh non-resident
```

필요할 때:

```text
remesh on GPU
```

로 geometry를 복구한다.

이 경우 geometry residency manager는 transfer manager가 아니라 **derived-data cache manager**가 된다.

### 4.44 Primary Data와 Derived Data를 구분한다

예:

```text
Primary:
SDF / process state / scalar field

Derived:
mesh / meshlet / bounds / acceleration data
```

Derived geometry는 evict 후 recompute 가능하다.

Primary simulation state는 훨씬 높은 residency priority가 필요할 수 있다.

Memory pressure에서 무엇을 먼저 evict할지 결정하는 핵심 기준이다.

### 4.45 Memory Priority Class

개념적 priority tier:

```text
Tier 0: frame-critical tables / coarse fallback
Tier 1: primary simulation state
Tier 2: visible geometry
Tier 3: predicted geometry
Tier 4: fine LOD cache
Tier 5: temporary scratch
```

`VK_EXT_memory_priority`를 지원하면 allocation priority hint와 engine-level tier를 어느 정도 대응시킬 수 있다.

하지만 actual eviction policy는 application이 유지한다.

### 4.46 Budget-Aware Allocator와 Residency Manager의 차이

Allocator:

```text
where to place bytes
```

Residency manager:

```text
which bytes deserve to exist here now
```

둘은 다른 문제다.

Allocator fragmentation이 낮아도 wrong working set이면 streaming thrash가 발생한다.

### 4.47 Eviction Candidate Heap

CPU 또는 GPU-side에서 eviction candidate를 priority queue로 관리할 수 있다.

Key:

```text
low priority first
```

하지만 exact heap update가 비싸면 bucket class로 근사할 수 있다.

```text
HOT
WARM
COLD
EVICTABLE
```

Frame마다 state transition만 수행하면 overhead가 작다.

### 4.48 GPU-Side Feedback, CPU-Side Policy

실무에서 좋은 split:

GPU generates:

```text
used pages
visibility score
request queue
LOD demand
```

CPU/host residency manager:

```text
budget query
allocation
upload scheduling
sparse bind
eviction policy
```

이 구조는 OS/device memory API와의 interaction을 host에 두면서 hot feedback만 GPU에서 수집한다.

반대로 fully GPU-driven residency가 필요한 경우 CUDA/Vulkan shared allocator까지 더 복잡해진다.

### 4.49 Readback를 최소화하는 방법

매 meshlet 사용 정보를 CPU에 readback할 필요는 없다.

GPU에서:

```text
per-page max priority
used bitset
request compaction
```

을 먼저 수행해 compact request list만 readback하거나, GPU→transfer path로 직접 연결할 수 있다.

Residency feedback 자체도 reduction problem이다.

### 4.50 Page Use Bitset

각 resident page가 이번 frame에 사용되었는지:

```text
usedBits[page]
```

를 GPU가 설정할 수 있다.

Frame end에 bitset을 compact/aggregate해 residency score를 업데이트한다.

Per-meshlet atomic 대신 subgroup/block aggregation을 적용할 수 있다.

### 4.51 Temporal Usage Score

단순 lastUsedFrame 대신 exponential decay score를 쓸 수 있다.

```text
score =
score * decay
+ currentImportance
```

장점:

- 순간적인 occlusion에 덜 민감
- persistent hot geometry가 높은 priority 유지

이 역시 residency hysteresis의 한 형태다.

### 4.52 Memory Budget Poll Frequency

`heapBudget/heapUsage`는 동적으로 변할 수 있지만 매 draw마다 query할 필요는 없다.

예:

- frame 단위
- 몇 frame마다
- large allocation 직전
- OS pressure event에 대응

정책을 둘 수 있다.

중요한 것은 budget 변화가 asynchronous external condition이라는 점이다.

### 4.53 Safety Margin

목표 budget을 100%로 두지 않는 이유:

- transient allocation
- command/driver overhead
- external process activity
- render target growth
- temporary COW duplication

등이다.

따라서:

```text
target < heapBudget
```

인 headroom을 둔다.

### 4.54 Snapshot Build의 Temporary Memory도 Budget에 포함한다

COW update 중:

```text
old resident
+
new allocation building
```

이 동시에 존재한다.

따라서 memory budget 계산은 steady-state만 보면 안 된다.

Worst-case:

```text
resident
+ building
+ retired pending
+ upload staging
```

을 봐야 한다.

### 4.55 Residency와 Reclamation의 연결

어제의:

```text
LiveReferenced
RetiredPending
Reclaimable
```

상태에 오늘은:

```text
ResidentNeeded
ResidentCold
NonResident
```

축이 추가된다.

즉 allocation state는 두 축이다.

#### Lifetime Axis

```text
live / retired / reclaimable
```

#### Residency Axis

```text
hot / cold / evicted
```

두 축을 하나의 enum으로 합치면 state explosion이 생길 수 있다.

### 4.56 Multi-Queue Consumer Pinning

Geometry를 graphics queue뿐 아니라 compute culling, ray query, async processing이 읽을 수 있다면 eviction safe point는 모든 consumer를 고려해야 한다.

```text
safeAfter =
max(lastUseGraphics,
    lastUseCompute,
    lastUseOther)
```

단, timeline domain이 다르면 단순 numeric max가 아니라 각 queue completion condition을 별도로 관리해야 한다.

### 4.57 Ray Tracing / Acceleration Structure

Geometry page를 evict해도 BLAS가 그 memory를 reference한다면 unsafe하다.

즉 residency dependency graph:

```text
Geometry Payload
↑
Meshlet Table
↑
BLAS / Render Command / Visibility Data
```

같은 derived resource가 있다면 eviction 전에 dependency invalidation/rebuild가 필요하다.

### 4.58 Residency Fault Debugging

유용한 reason code:

```text
RESIDENT_HIT
PREFETCH_HIT
FALLBACK_LOD
MISS_REQUESTED
MISS_STALLED
EVICTED_COLD
PINNED_IN_FLIGHT
OVER_BUDGET_EVICTION
RECOMPUTE_SELECTED
```

화면 overlay:

- resident LOD
- desired LOD
- request priority
- page age
- pin state

를 보여주면 streaming 문제를 빠르게 찾을 수 있다.

### 4.59 프로파일링에서 볼 지표

- heapBudget
- heapUsage
- target resident bytes
- live resident bytes
- pinned resident bytes
- cold resident bytes
- retired pending bytes
- staging/upload bytes
- upload bytes/frame
- upload time/frame
- eviction bytes/frame
- request count
- deduplicated request count
- residency hit ratio
- prefetch hit ratio
- fallback LOD ratio
- hard miss/stall count
- average/p95 streaming latency
- page lifetime
- thrash reload count
- COW temporary high-water
- sparse bind operations/frame
- sparse bind latency
- recompute vs transfer count
- resident quality benefit/byte
- oldest pinned epoch

중요한 derived metric:

```text
Useful Visible Bytes
--------------------
Resident Bytes
```

와:

```text
Reloaded Within K Frames
------------------------
Evicted Bytes
```

두 번째가 높으면 thrashing이다.

---

## 5. 내 관심 분야와 연결

### Semiconductor process visualization

반도체 scene은 residency policy에 유리한 domain signal이 많다.

예:

- 현재 cut plane
- selected process layer
- current device region
- camera orbit target
- process front
- visible material subset

따라서 generic LRU보다 훨씬 좋은 working-set prediction이 가능하다.

```text
near current cross-section
→ HOT

far wafer region
→ COLD
```

처럼 볼 수 있다.

### Sparse SDF + Mesh Cache

사용자의 pipeline에서는 primary SDF와 derived mesh를 분리할 수 있다.

```text
SDF brick
→ primary state

Mesh / Meshlet
→ derived cache
```

VRAM pressure가 생기면 mesh를 evict하고 SDF만 유지한 뒤 필요 시 remesh할 수 있다.

이것은 일반 asset streaming보다 dynamic simulation에 더 적합한 전략일 수 있다.

### CUDA / Vulkan

가능한 architecture:

```text
CUDA:
visibility/meshing feedback
page used bits
mesh recompute
        ↓
Shared Residency Table
        ↓
Vulkan:
culling
LOD fallback
render
```

Host는 `VK_EXT_memory_budget`으로 budget을 관측하고 allocation/sparse binding policy를 조절할 수 있다.

### ColumnStack / Sparse Geometry

ColumnStack 또는 brick 단위를 residency page로 활용할 수 있다.

단, logical chunk 크기와 Vulkan sparse page granularity가 다르면 internal fragmentation이 생길 수 있으므로:

```text
logical update granularity
≠
physical sparse bind granularity
```

를 구분해야 한다.

### Game engine / graphics career

이 개념은 다음 역할과 직접 연결된다.

- virtualized geometry
- open-world streaming
- procedural terrain
- large CAD/scientific scene
- destructible mesh cache
- ray-tracing geometry residency
- GPU-driven asset system

Engine graphics 면접에서 좋은 질문은:

> **“안 보이는 mesh를 그냥 VRAM에서 빼면 되는 것 아닌가요?”**

에 대해 visibility, in-flight lifetime, fallback LOD, prefetch latency, memory budget을 함께 설명할 수 있는가다.

---

## 6. 머릿속에 남길 질문 3개

1. **현재 frame에서 occluded된 geometry를 즉시 eviction candidate로 낮추면 어떤 camera/disocclusion 상황에서 residency thrashing이 발생할 수 있으며, visibility hysteresis와 residency hysteresis를 어떻게 다르게 설계해야 할까?**
2. **Dynamic SDF pipeline에서 mesh payload를 host memory에서 다시 upload하는 것과 resident SDF에서 GPU remesh하는 것 중 어느 쪽이 더 유리한지 판단하려면 어떤 cost를 비교해야 할까?**
3. **Vulkan sparse buffer를 사용해 virtual geometry address space를 만들 때 logical chunk granularity와 sparse binding granularity가 다르면 fragmentation·streaming priority·snapshot mapping에 어떤 영향이 생길까?**

---

## 7. graphics engineer 면접 질문 1개와 답변

### 질문

**“GPU memory가 부족하면 가장 오래 안 보인 mesh부터 LRU로 evict하면 충분하지 않나요?”**

### 답변

단순한 scene에서는 가능하지만 large dynamic renderer에서는 LRU만으로 부족한 경우가 많다.

첫째, visibility와 future demand는 다르다. Mesh가 현재 occluded되어 몇 frame 동안 사용되지 않았더라도 camera가 조금만 움직이면 바로 다시 필요할 수 있다.

둘째, residency miss 비용이 일반 cache miss보다 크다. Geometry는 host→GPU transfer, allocation, sparse bind 또는 remesh, snapshot patch가 필요할 수 있으므로 reload latency가 여러 frame에 걸릴 수 있다.

셋째, 모든 geometry의 reload cost가 같지 않다.

```text
asset geometry
→ transfer가 필요

derived SDF mesh
→ GPU recompute 가능

coarse LOD resident
→ fine LOD miss를 fallback으로 숨길 수 있음
```

넷째, in-flight Vulkan snapshot이 old allocation을 참조하고 있다면 invisible이어도 즉시 free/evict할 수 없다.

그래서 production residency score는 보통 다음을 함께 본다.

```text
recency
visibility probability
projected importance
reload/recompute cost
fallback availability
in-flight pinning
memory budget pressure
```

또 `VK_EXT_memory_budget`의 `heapBudget`은 heap의 물리적 size가 아니라 현재 process가 allocation 실패나 performance degradation 전에 사용할 수 있는 양에 대한 동적 estimate이므로 residency target 자체도 runtime에 변할 수 있다.

핵심은:

> **Eviction은 “가장 오래 안 쓴 것”을 버리는 문제가 아니라, 미래 frame quality와 reload latency를 최소 비용으로 유지하면서 dynamic memory budget 안에 working set을 맞추는 문제다.**

---

## 8. 포트폴리오 / 커리어 연결

이 주제는 graphics engineer가 단순 allocator를 넘어 **virtualization + streaming + GPU-driven feedback**을 이해한다는 것을 보여주기 좋다.

### GPU Memory

- `VK_EXT_memory_budget`
- heapBudget / heapUsage
- safety margin
- resident working set
- high/low-water hysteresis

### Vulkan

- sparse binding
- sparse residency
- `vkQueueBindSparse`
- sparse block granularity
- memory priority
- queue synchronization

### Rendering

- fallback LOD
- desired vs resident LOD
- predictive frustum
- occlusion-aware priority
- residency miss behavior

### GPU Compute

- request queue
- request deduplication
- page-used bitset
- priority reduction
- remesh vs transfer

### Dynamic Geometry

- SDF as primary state
- mesh as derived cache
- incremental remeshing
- process-front prediction

### Memory Layout

- virtual page table
- generation-checked physical page
- logical handle indirection
- snapshot-compatible residency mapping

### C++

- `GeometryPageId`
- `ResidencyState`
- `MemoryBudget`
- `PublishEpoch`
- `SafeAfterTimeline`
- `StreamingPriority`

포트폴리오에서는 다음처럼 설명할 수 있다.

> **“Large dynamic geometry를 persistent virtual page space로 관리하고 current visibility, projected error, camera prediction을 이용해 working set을 만들었습니다. Coarse LOD는 항상 resident하게 유지하고 fine LOD miss는 fallback으로 처리했습니다. `VK_EXT_memory_budget` 기반 target budget과 high/low-water hysteresis로 eviction을 조절하고, in-flight snapshot이 참조하는 allocation은 timeline value까지 pin했습니다. SDF-derived mesh는 transfer와 remesh cost를 비교해 reload source를 선택했습니다.”**

이 설명은 graphics, GPU memory, streaming, Vulkan, simulation을 강하게 연결한다.

---

## 9. 내일 이어서 볼 개념

**GPU Residency Feedback Loops: Prefetch Prediction, Thrash Control, and Quality-Per-Byte Scheduling**

오늘은 geometry가 GPU-local memory보다 클 때 virtual residency와 eviction 구조를 봤다.

다음에는 residency manager를 **closed-loop control system**처럼 본다.

```text
Visibility / Camera Motion
    ↓
Residency Demand Prediction
    ↓
Streaming
    ↓
Memory Pressure
    ↓
Eviction
    ↓
Miss / Thrash Feedback
    ↺
```

다음 노트에서는:

- prefetch confidence
- miss latency feedback
- hysteresis
- thrashing detector
- quality-per-byte metric
- load/evict asymmetry
- adaptive streaming budget
- camera prediction error
- simulation-front prediction
- working-set controller

를 중심으로 이어간다.

---

## 10. 참고 키워드

- GPU Geometry Residency
- Geometry Streaming
- Working Set
- Virtual Geometry
- Residency Table
- Page Table
- Sparse Resource
- Sparse Binding
- Sparse Residency
- `VK_BUFFER_CREATE_SPARSE_BINDING_BIT`
- `VK_BUFFER_CREATE_SPARSE_RESIDENCY_BIT`
- `vkQueueBindSparse`
- Sparse Block Granularity
- `VkMemoryRequirements::alignment`
- `VK_EXT_memory_budget`
- `VkPhysicalDeviceMemoryBudgetPropertiesEXT`
- `heapBudget`
- `heapUsage`
- `VK_EXT_memory_priority`
- Resident Set
- Eviction
- Residency Miss
- Prefetch
- Predictive Frustum
- Residency Hysteresis
- Thrashing
- High-Water / Low-Water Mark
- Fallback LOD
- Desired LOD / Resident LOD
- Streaming Budget
- Bytes per Frame
- Page Request Queue
- Request Deduplication
- Residency Bitset
- Generation-Checked Page
- In-Flight Pinning
- Timeline Semaphore
- Deferred Reclamation
- Transfer vs Recompute
- Derived Geometry Cache
- Dynamic SDF
- Meshlet Streaming
- Khronos Vulkan Specification — **Sparse Resources**
  - https://docs.vulkan.org/spec/latest/chapters/sparsemem.html
- Khronos Vulkan Guide — **Sparse Resources**
  - https://docs.vulkan.org/guide/latest/sparse_resources.html
- Khronos Vulkan Specification — **Memory Allocation / VK_EXT_memory_budget**
  - https://docs.vulkan.org/spec/latest/chapters/memory.html
- Khronos Vulkan Reference — **VK_EXT_memory_budget**
  - https://docs.vulkan.org/refpages/latest/refpages/source/VK_EXT_memory_budget.html
- Khronos Vulkan Reference — **VK_EXT_memory_priority**
  - https://docs.vulkan.org/refpages/latest/refpages/source/VK_EXT_memory_priority.html
