---
title: "GPU Geometry Page Fault Avoidance: Request Latency Hiding, Fallback Hierarchies, and Stall-Free Rendering"
date: "2026-09-20"
category: Graphics
tags: [GPU, Rendering, GPU-Driven Rendering, Geometry Streaming, Residency, Page Fault Avoidance, Fallback LOD, Sparse Resources, Vulkan, DirectX 12, DirectStorage, Meshlet, Dynamic Geometry, Memory Layout, C++]
level: intermediate
---

# [Daily Graphics Growth] 2026-09-20 - GPU Geometry Page Fault Avoidance: Request Latency Hiding, Fallback Hierarchies, and Stall-Free Rendering

## 1. 오늘의 개념

어제는 **GPU Residency Feedback Loops**에서 geometry residency를 단순 LRU가 아니라 `prefetch → hit/miss → eviction regret → policy update`의 closed-loop controller로 보았다.

오늘은 그 다음 문제다.

> **Prefetch를 아무리 잘해도 residency miss는 남는다. 그렇다면 missing geometry 때문에 frame을 멈추지 않는 renderer는 어떻게 설계해야 하는가?**

여기서 말하는 **page fault**는 CPU의 투명한 demand paging과 같은 모델이 아니다. Vulkan/Direct3D 12 같은 explicit graphics API에서는 application이 logical page의 residency를 직접 추적하고, missing page를 동기적으로 기다리기보다 다음처럼 **soft miss**로 바꾸는 편이 강하다.

```text
Logical Geometry
    ↓
Desired LOD/Page
    ↓ residency lookup
┌───────────────┬─────────────────┐
│ resident      │ non-resident    │
↓               ↓
full detail     resident fallback
render          + async request
                ↓
                later snapshot upgrade
```

즉 목표는:

```text
Residency Miss ≠ Frame Stall
```

이다.

오늘의 세 축은 **Request Latency Hiding**, **Fallback Hierarchy**, **Immutable Snapshot Promotion**이다.

---

## 2. 한 줄 핵심

> Stall-free geometry streaming은 **missing page를 기다리지 않고 항상 resident한 coarse/parent representation으로 즉시 대체하고, request를 비동기로 발행한 뒤 page가 완성되면 다음 immutable snapshot에서 detail을 승격하는 구조**다.

---

## 3. 왜 중요한가

Large dynamic scene에서는 다음 이유로 miss를 0으로 만들기 어렵다.

- camera cut / 빠른 orbit
- memory budget 축소
- eviction 직후 재접근
- transfer backlog
- simulation front 급변
- fine LOD request burst
- prediction error

동기식 구조는 작은 miss를 frame hitch로 증폭시킨다.

```text
visibility
→ page missing
→ upload/remesh wait
→ mapping update
→ rendering resume
```

반대로 stall-free 구조는 latency를 critical path 밖으로 뺀다.

```text
visibility
→ desired page missing
→ fallback draw
→ request append
→ async transfer/remesh
→ finalize
→ next snapshot publish
```

Explicit API의 non-resident access semantics도 이유가 된다. Vulkan sparse residency에서 unbound read는 `residencyNonResidentStrict`가 `VK_TRUE`이면 0처럼 동작하지만, 그렇지 않으면 read value가 undefined일 수 있다. D3D12 tiled resources도 Tier 2 이상과 Tier 1의 NULL-mapping semantics가 다르다.

따라서 portable geometry system은:

> **payload를 읽은 뒤 missing을 추론하지 말고, explicit residency metadata를 먼저 검사한 뒤에만 physical payload를 dereference한다.**

2026년 시점의 industry 방향도 이 흐름과 잘 맞는다. NVIDIA의 RTX Mega Geometry Vulkan sample은 cluster LOD와 RAM→VRAM on-demand geometry streaming을 보여주고, Microsoft는 2026년 3월 DirectStorage 1.4 public preview에서 Zstandard를 추가하며 content-rich streaming의 throughput과 compression 경로를 확장했다. 빠른 I/O 자체보다 **I/O latency를 rendering hierarchy로 숨기는 architecture**가 핵심이다.

---

## 4. 구현 관점

### 4.1 Hard Miss와 Soft Miss

**Hard Miss**는 정확한 data가 반드시 필요해 기다림이 허용되는 경우다.

- exact measurement
- export
- mandatory simulation state
- correctness-critical compute

**Soft Miss**는 낮은 detail로 current frame을 계속 진행할 수 있다.

- fine meshlet page
- distant high LOD
- secondary overlay
- high-resolution displacement

Interactive renderer에서는 가능한 많은 visual miss를 soft miss로 바꾸는 것이 좋다.

### 4.2 Always-Resident Root

각 logical geometry가 최소 하나의 resident ancestor를 가진다.

```text
Root / Coarse Proxy  ← always resident
        ↓
      LOD2
        ↓
      LOD1
        ↓
      LOD0
```

LOD0가 없으면 LOD1, LOD1도 없으면 LOD2처럼 ancestor를 찾는다.

이 구조의 의미는 miss가 **“geometry 없음”**이 아니라 **“temporarily coarser geometry”**가 된다는 것이다.

### 4.3 Fallback은 Rendering Mode에 따라 다르다

Scientific/engineering viewer에서는 모든 fallback이 같은 의미가 아니다.

```text
Orbit / overview     → coarse fallback 허용
Cross-section        → section 근처 detail priority
Measurement          → exact region hard requirement
Export               → stall 허용, exact snapshot 요구
```

따라서 `renderable fallback`과 `measurement-safe fallback`을 분리해야 한다.

### 4.4 Residency Lookup은 Payload Fetch보다 먼저

```text
LogicalPageID
→ ResidencyEntry
→ resident?
→ physical page/address resolve
→ payload fetch
```

이 순서를 지킨다.

Conceptual hot entry:

```text
ResidencyEntry {
    stateBits
    physicalPage
    generation
    fallbackPage
}
```

`lastUsed`, `requestPriority`, `debugReason` 같은 cold metadata는 별도 SoA로 분리할 수 있다.

### 4.5 Page Generation

Physical page slot은 재사용된다.

```text
page = 42
generation = 7
```

Old worklist가 `page 42 / generation 6`을 갖고 있다면 reject한다.

즉 system은 non-resident miss뿐 아니라 **stale residency hit**도 막아야 한다.

### 4.6 Request는 가능한 한 Coarse Stage에서 만든다

Mesh shader가 vertex fetch 직전에 miss를 발견하는 것보다:

```text
chunk visibility
→ LOD selection
→ residency check
→ request/fallback
→ meshlet expansion
```

순서가 낫다.

Missing detail의 meshlet expansion 자체를 피할 수 있고 fallback worklist를 바로 만들 수 있다.

### 4.7 Request Deduplication

같은 page가 여러 candidate에서 요청될 수 있다.

```text
requestedEpoch[page]
```

같은 field를 이용해 첫 requester만 queue에 append한다.

Request item 예:

```text
PageRequest {
    logicalPage
    desiredLOD
    priority
    requestEpoch
}
```

Miss storm에서 dedup은 단순 optimization이 아니라 queue stability에 중요하다.

### 4.8 Request Queue Overflow

Request queue가 full이어도 current frame fallback rendering은 계속 가능해야 한다.

Low-priority fine request는 drop/coalesce할 수 있지만:

```text
request loss
→ quality recovery delay
```

일 뿐:

```text
request loss
→ current-frame correctness loss
```

가 되어서는 안 된다.

### 4.9 Coverage-First Recovery

Camera cut에서는 수천 page가 동시에 필요해질 수 있다.

모든 fine LOD를 즉시 요청하지 않고:

```text
Phase 1: all visible regions get coarse coverage
Phase 2: important regions refine
Phase 3: fine detail recovery
```

순서로 복구한다.

이 방식은 일부 object만 완벽하고 나머지는 통째로 사라지는 것보다 temporal stability가 좋다.

### 4.10 Streaming Debt

현재 desired quality와 resident quality 차이를 **streaming debt**로 볼 수 있다.

```text
StreamingDebt
≈
Σ qualityGap(page)
```

Debt를 갚는 순서는 어제의 `quality-per-byte`와 연결된다.

```text
ΔQuality / UploadBytes
```

가 높은 page부터 refinement한다.

### 4.11 SDF는 Cross-Representation Fallback을 만들 수 있다

사용자의 domain에서는 fine mesh가 없어도 SDF가 resident할 수 있다.

가능한 fallback:

```text
fine mesh
→ coarse mesh
→ coarse SDF raymarch
```

그리고 recovery:

```text
host transfer
or
GPU remesh
```

를 runtime cost model로 선택할 수 있다.

### 4.12 Transfer vs Recompute

Derived mesh miss의 recovery cost:

```text
readyTimeTransfer
vs
readyTimeRemesh
```

를 비교한다.

단순 latency뿐 아니라:

- transfer bandwidth pressure
- compute occupancy
- source SDF residency
- allocation pressure

를 포함한다.

### 4.13 Current Snapshot을 In-Place Patch하지 않는다

Page가 도착했다고 현재 render 중인 snapshot을 수정하지 않는다.

```text
Page finalized
→ READY_FOR_SNAPSHOT
→ Snapshot E+1 patch
→ publish E+1
```

로 승격한다.

이렇게 해야 frame 안에서 일부 shader가 old mapping, 일부가 new mapping을 보는 torn state를 막을 수 있다.

### 4.14 Page Arrival State Machine

```text
NON_RESIDENT
→ REQUESTED
→ ALLOCATED
→ TRANSFER / RECOMPUTE
→ FINALIZED
→ READY_FOR_SNAPSHOT
→ PUBLISHED
→ RESIDENT
```

Renderer는 `PUBLISHED` 이전 payload를 소비하지 않는다.

### 4.15 Request Latency Hiding

두 경로를 동시에 사용한다.

```text
Prediction path:
prefetch → page ready before demand

Failure path:
unexpected miss → fallback → urgent request
```

즉 **prefetch는 latency hiding이고 fallback은 prediction failure insurance**다.

### 4.16 Miss Storm Priority Classes

Request queue를 class로 나눌 수 있다.

```text
COVERAGE
VISIBLE_REFINE
PREFETCH
BACKGROUND
```

일반 우선순위:

```text
Coverage > VisibleRefine > Prefetch > Background
```

Camera cut에서 fine prefetch가 coarse coverage를 막지 않게 한다.

### 4.17 Fallback Crack

인접 chunk가 서로 다른 LOD를 사용하면 crack이 생길 수 있다.

```text
Chunk A → LOD0
Chunk B → LOD2
```

대응:

- transition mesh
- skirt
- shared boundary constraints
- crack-free hierarchy

즉 residency hierarchy는 LOD topology와 분리해서 생각할 수 없다.

### 4.18 Residency Atomicity Unit

Page 하나가 resident하다는 의미는 해당 page의:

- vertex/index
- meshlet descriptor
- bounds
- interpretation metadata

가 모두 같은 version으로 valid하다는 뜻이어야 한다.

`resident bit`은 단순 allocation 존재 여부가 아니라 **payload validity contract**다.

### 4.19 Vulkan Sparse Resource

Vulkan sparse residency는 resource를 partially resident하게 만들 수 있지만 sparse API가 자동 streaming system을 제공하는 것은 아니다.

Application이 직접 관리해야 하는 것:

```text
request
allocation
binding
synchronization
fallback
eviction
```

또 `residencyNonResidentStrict`가 없는 device에서는 unbound read value가 undefined이므로 explicit residency lookup이 더욱 중요하다.

### 4.20 D3D12 Tiled Resource

D3D12 Tier 2 이상에서는 NULL-mapped tile read가 zero로 취급될 수 있지만 Tier 1에서는 undefined다.

그러나 geometry에서 zero vertex/index는 safe visual fallback이 아니다.

즉 API의 NULL tile semantics는 **memory safety primitive**이고, application의 parent/fallback geometry는 **rendering policy**다.

### 4.21 DirectStorage의 역할

DirectStorage 1.3은 request batching과 D3D12 fence synchronization을 강화했고, 1.4 public preview는 Zstd를 추가했다.

Architecture는 여전히:

```text
I/O
→ decompression
→ GPU-ready payload
→ mapping/snapshot publication
→ rendering
```

이다.

I/O 완료와 render-visible publication은 같은 사건이 아니다.

### 4.22 Mesh Shader 경로

```text
logical cluster
→ desired LOD
→ residency lookup
→ desired meshlets OR fallback meshlets
→ request if needed
```

처럼 late-binding하기 쉽다.

Classic indexed indirect draw는 `firstIndex/vertexOffset`이 physical placement에 더 강하게 묶여 fallback-specific command 생성이 필요할 수 있다.

### 4.23 Interactive Mode와 Exact Mode

**Interactive Mode**

```text
stall-free
fallback-first
latest published snapshot
```

**Exact Mode**

```text
required pages fully resident
wait allowed
deterministic snapshot
```

Engineering tool은 두 policy를 함께 갖는 것이 자연스럽다.

### 4.24 프로파일링

핵심 metrics:

```text
Residency-induced frame stall time
Soft miss count
Hard miss count
Fallback screen coverage
Average fallback depth
Urgent request bytes
Request dedup ratio
Request queue overflow
p50/p95/p99 page ready latency
Ready→published latency
Camera-cut coverage recovery time
Target-quality recovery time
Transfer vs remesh latency
Generation mismatch count
```

가장 중요한 결과 지표는:

```text
Residency-Induced Frame Stall Time
```

을 0에 가깝게 유지하면서:

```text
Fallback Screen Coverage
```

를 빠르게 줄이는 것이다.

---

## 5. 내 관심 분야와 연결

사용자의 SDF/CFD/반도체 visualization pipeline에서는 이 개념이 특히 잘 맞는다.

```text
Primary SDF
    ↓
Derived Mesh / Meshlet Cache
    ↓
Residency-Aware Visibility
```

Fine mesh가 없더라도 SDF가 resident하면:

```text
coarse mesh
or
raymarch fallback
```

으로 current frame을 유지하고, background에서 GPU remesh할 수 있다.

반도체 viewer에서는 generic screen-space importance 외에도 다음 signal을 priority에 넣을 수 있다.

- cross-section plane 근접도
- selected material/layer
- measurement region
- etch/deposition front
- camera orbit target

즉 게임 엔진의 visual LOD보다 **semantic importance**가 강한 residency policy를 만들 수 있다.

CUDA/Warp는 request compaction·remesh·payload finalization을 담당하고, Vulkan은 committed residency snapshot만 읽어 visibility와 rendering을 수행하는 구조가 안정적이다.

---

## 6. 머릿속에 남길 질문 3개

1. **Always-resident ancestor LOD를 유지할 때 root memory overhead와 camera-cut recovery latency 사이의 균형은 어떻게 잡아야 할까?**
2. **Dynamic SDF에서 fine mesh miss가 발생했을 때 host transfer, GPU remesh, coarse raymarch 중 어느 recovery path를 선택할지 어떤 runtime cost model이 가장 실용적일까?**
3. **Vulkan sparse resource에서 unbound access가 safe한 device에서도 payload fetch 전에 explicit residency bit를 확인해야 하는 이유를 portability와 geometry semantics 관점에서 어떻게 설명할 수 있을까?**

---

## 7. graphics engineer 면접 질문 1개와 답변

### 질문

**“GPU geometry streaming에서 required page가 없으면 upload가 끝날 때까지 기다리면 가장 정확하지 않나요?”**

### 답변

정확도만 보면 그렇지만 real-time renderer에서는 page miss를 frame critical path에 넣으면 작은 streaming delay가 전체 hitch로 확대된다.

그래서 visual miss 대부분을 **soft miss**로 바꾸는 편이 강하다.

```text
desired LOD missing
→ resident ancestor 선택
→ current frame 계속 render
→ async request
→ transfer/remesh finalize
→ next immutable snapshot에서 detail upgrade
```

이 구조에서는 streaming latency가 frame stall이 아니라 일시적인 quality loss가 된다.

또 sparse resource의 unbound read 결과에 의존해 missing page를 판단하면 안 된다. Vulkan은 `residencyNonResidentStrict` 여부에 따라 read semantics가 달라지고, D3D12 tiled resources도 tier에 따라 NULL mapping semantics가 다르다.

따라서 production 구조는:

- explicit residency metadata
- always-resident fallback
- request deduplication
- page generation validation
- coverage-first recovery
- immutable snapshot promotion

을 함께 사용한다.

핵심은:

> **Streaming system의 목표는 miss를 완전히 없애는 것이 아니라 miss가 발생해도 frame progress를 멈추지 않게 만드는 것이다.**

---

## 8. 포트폴리오 / 커리어 연결

이 주제는 단순 asset loader보다 한 단계 높은 **stall-free rendering architecture**를 보여주기 좋다.

연결할 포인트:

- **Rendering:** fallback LOD, coverage-first recovery, temporal stability
- **GPU-driven:** residency-aware visibility, request generation, meshlet fallback
- **Memory:** page state machine, generation, sparse binding
- **Simulation:** SDF primary state, mesh derived cache, recompute fallback
- **Vulkan/D3D12:** sparse residency, tiled resources, DirectStorage
- **C++:** `ResidencyEntry`, `PageGeneration`, `PageRequest`, `FallbackPolicy`
- **Profiling:** stall time, fallback coverage, miss latency, request backlog

포트폴리오에서는 다음처럼 설명할 수 있다.

> **“Geometry miss를 동기 upload로 해결하지 않고 always-resident ancestor hierarchy로 soft fault 처리했습니다. Visibility 단계에서 desired LOD residency를 확인하고 missing page는 deduplicated request queue에 넣은 뒤 parent meshlet을 current-frame worklist에 사용합니다. Transfer 또는 GPU remesh가 끝난 page는 current snapshot을 수정하지 않고 next immutable snapshot에서 승격하며, camera cut에서는 fine detail보다 coarse coverage recovery를 우선합니다.”**

---

## 9. 내일 이어서 볼 개념

**Hierarchical Geometry Streaming Trees: Parent Residency Invariants, Crack-Free LOD Transitions, and Refinement Scheduling**

오늘은 miss를 ancestor fallback으로 처리해 frame stall을 피하는 구조를 봤다.

다음에는 fallback hierarchy 자체를 깊게 본다.

```text
Resident Root
    ↓
Parent LOD
    ↓
Child LOD
    ↓
Fine Meshlet Pages
```

다음 노트에서는:

- parent residency invariant
- refine/coarsen state machine
- partial child residency
- crack-free transition
- neighbor dependency
- screen-space error propagation
- refinement budget
- parent retire condition
- cluster-tree memory layout

을 연결한다.

---

## 10. 참고 키워드

- GPU Geometry Page Fault Avoidance
- Residency Miss
- Soft Miss / Hard Miss
- Stall-Free Rendering
- Fallback Hierarchy
- Resident Root Representation
- Parent LOD / Ancestor Fallback
- Coverage-First Recovery
- Streaming Debt
- Request Latency Hiding
- Page Request Queue
- Request Deduplication
- Page Generation
- Immutable Residency Snapshot
- Vulkan Sparse Residency
- `residencyNonResidentStrict`
- `vkQueueBindSparse`
- D3D12 Tiled Resources
- NULL-Mapped Tiles
- DirectStorage 1.3 `EnqueueRequests`
- DirectStorage 1.4 Zstandard
- NVIDIA RTX Mega Geometry
- Cluster LOD
- Mesh Shader
- Dynamic SDF
- Transfer vs Recompute
- SDF Raymarch Fallback
- Khronos Vulkan Specification — **Sparse Resources**
  - https://docs.vulkan.org/spec/latest/chapters/sparsemem.html
- Microsoft Learn — **D3D12_TILED_RESOURCES_TIER**
  - https://learn.microsoft.com/en-us/windows/win32/api/d3d12/ne-d3d12-d3d12_tiled_resources_tier
- Microsoft Learn — **ID3D12CommandQueue::UpdateTileMappings**
  - https://learn.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12commandqueue-updatetilemappings
- Microsoft DirectX Developer Blog — **DirectStorage 1.3 is now available**, 2025-07-01
  - https://devblogs.microsoft.com/directx/directstorage-1-3-is-now-available/
- Microsoft DirectX Developer Blog — **DirectStorage 1.4 release adds support for Zstandard**, 2026-03-11
  - https://devblogs.microsoft.com/directx/directstorage-1-4-release-adds-support-for-zstandard/
- NVIDIA Technical Blog — **NVIDIA RTX Mega Geometry Now Available with New Vulkan Samples**, 2025-02-12
  - https://developer.nvidia.com/blog/nvidia-rtx-mega-geometry-now-available-with-new-vulkan-samples/
