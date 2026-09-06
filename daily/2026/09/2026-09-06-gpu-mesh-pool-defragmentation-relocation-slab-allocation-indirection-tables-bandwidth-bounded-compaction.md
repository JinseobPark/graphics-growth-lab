---
title: "GPU Mesh Pool Defragmentation and Relocation: Slab Allocation, Indirection Tables, and Bandwidth-Bounded Compaction"
date: "2026-09-06"
category: Graphics
tags: [GPU, Rendering, Dynamic Mesh, Memory Pool, Defragmentation, Relocation, Slab Allocation, Indirection Table, Stream Compaction, CUDA, Vulkan, Buffer Device Address, Indirect Draw, Memory Layout, C++]
level: intermediate
---

# [Daily Graphics Growth] 2026-09-06 - GPU Mesh Pool Defragmentation and Relocation: Slab Allocation, Indirection Tables, and Bandwidth-Bounded Compaction

## 1. 오늘의 개념

어제는 **Incremental GPU Meshing on Dynamic Sparse SDFs**에서 changed brick만 다시 meshing하고, `SurfaceHandle -> HandleTable -> PhysicalVertexSlot` 형태로 logical identity와 physical storage를 분리하는 구조를 봤다.

오늘은 그 persistent mesh가 수백~수천 번 갱신된 이후에 생기는 문제로 넘어간다.

> **vertex/index/meshlet chunk가 계속 생성·삭제·크기 변경될 때, stable logical handle을 유지하면서 GPU memory fragmentation을 줄이고, relocation 비용을 frame budget 안에 제한하려면 mesh pool을 어떻게 설계해야 하는가?**

Dynamic mesh pool은 단순한 `malloc/free` 문제가 아니다. Rendering pipeline에서는 allocation이 다음과 연결되어 있다.

- vertex/index buffer offset
- meshlet descriptor
- indirect draw command
- shader-visible address 또는 descriptor
- picking/metrology handle
- in-flight command buffer
- CUDA/Vulkan producer-consumer synchronization

따라서 **defragmentation(조각 모음)** 은 메모리 공간을 예쁘게 정리하는 작업이 아니라, **live GPU object의 physical location을 바꾸면서 모든 consumer가 동일한 relocation epoch를 보도록 만드는 data-movement protocol**이다.

오늘은 특히 세 가지 축을 본다.

1. **Slab / size-class allocation**으로 fragmentation의 상한을 관리하는 방법
2. **Indirection table**로 logical identity와 physical offset/address를 분리하는 방법
3. **Bandwidth-bounded compaction**으로 relocation을 한 frame에 몰아넣지 않는 방법

---

## 2. 한 줄 핵심

> Dynamic GPU mesh pool의 핵심은 **“완벽하게 compact한 메모리”가 아니라, relocation-safe indirection을 유지한 채 fragmentation 이득이 copy bandwidth와 synchronization 비용보다 클 때만 제한된 byte budget으로 조금씩 이동하는 것**이다.

---

## 3. 왜 중요한가

Incremental meshing은 full rebuild를 피하지만 새로운 종류의 비용을 만든다.

예를 들어 brick별 mesh가 다음처럼 변한다고 하자.

- Brick A: 12 KB → 삭제
- Brick B: 20 KB → 28 KB로 증가
- Brick C: 8 KB → 그대로 유지
- Brick D: 새로 16 KB 생성

시간이 지나면 free space의 총량은 충분해도 작은 hole로 흩어져 큰 contiguous allocation을 만들지 못할 수 있다. 이를 **external fragmentation**이라고 한다.

반대로 size-class/slab을 사용하면 block 내부에 사용하지 않는 slack이 생길 수 있다. 이것은 **internal fragmentation**이다.

둘은 trade-off 관계다.

- exact-size free-list → internal waste는 작지만 external fragmentation/allocator metadata가 커질 수 있음
- slab/size-class → external fragmentation과 allocation latency를 줄이기 쉽지만 capacity slack이 생김

Graphics에서는 여기에 세 번째 비용이 추가된다.

**relocation bandwidth**다.

Mesh chunk 500 MB를 더 조밀하게 만들 수 있더라도, 그 500 MB를 한 frame에 `src -> dst` copy하면 frame time이 크게 흔들릴 수 있다. GPU memory bandwidth가 높아도 동시에 shading, simulation, sparse-volume update, meshing이 bandwidth를 사용하고 있기 때문이다.

CUDA의 최신 Best Practices Guide도 effective bandwidth를 핵심 성능 지표로 보고, global-memory access의 coalescing과 실제 bandwidth 사용을 최적화 중심에 둔다. Dynamic mesh compaction도 결국 많은 경우 compute-bound가 아니라 **memory-traffic-bound** workload다.

그래서 목표는 다음이 아니다.

> “fragmentation을 0으로 만든다.”

더 실용적인 목표는 다음과 같다.

> “필요한 allocation이 실패하지 않고, active mesh의 working set locality를 유지하며, relocation traffic을 일정 budget 이하로 제한한다.”

---

## 4. 구현 관점

### 4.1 Fragmentation을 하나의 숫자로 보지 않는다

먼저 어떤 fragmentation을 줄이려는지 구분해야 한다.

#### External fragmentation

Free bytes가 여러 hole로 흩어진 상태다.

간단한 diagnostic으로 다음과 같은 비율을 생각할 수 있다.

`F_external = 1 - LargestFreeBlock / TotalFreeBytes`

- `0`에 가까움: free space가 큰 덩어리로 모여 있음
- `1`에 가까움: free space가 잘게 흩어짐

이 식 자체가 절대적인 allocator quality metric은 아니지만, **“총 free memory는 충분한데 큰 allocation이 왜 실패하는가?”** 를 설명하는 데 유용하다.

#### Internal fragmentation

할당한 block capacity와 실제 payload size의 차이다.

`InternalWaste = Σ(capacity_i - used_i)`

Slab allocator에서는 이 값이 명시적으로 보인다.

#### Logical fragmentation

메모리는 충분히 compact하지만 함께 사용되는 meshlet/vertex chunk가 물리적으로 멀리 흩어져 cache locality가 나쁜 경우다.

즉 `memory compactness`와 `rendering locality`는 같은 목표가 아니다.

---

### 4.2 Slab / size-class allocation이 dynamic mesh에 잘 맞는 이유

Dynamic mesh chunk size를 완전히 arbitrary하게 두면 allocator가 복잡해진다.

대신 몇 개의 **size class**를 정의할 수 있다.

예:

- 4 KB
- 8 KB
- 16 KB
- 32 KB
- 64 KB
- 128 KB

요청 크기 `S`는 가장 작은 `class >= S`에 배치한다.

이 구조의 장점은 다음과 같다.

- allocation/free가 단순하다.
- free-list metadata가 작다.
- 같은 class 내부 relocation이 쉽다.
- GPU-side allocator를 만들 때 atomic contention을 제한하기 쉽다.
- compaction destination의 capacity가 예측 가능하다.

대신 33 KB mesh가 64 KB slab을 차지할 수 있으므로 internal waste가 생긴다.

Dynamic sparse SDF meshing처럼 brick별 mesh complexity가 어느 정도 범위 안에 있다면 size distribution을 profiling해서 **실제 histogram에 맞는 class boundary**를 정하는 것이 중요하다.

Power-of-two class가 구현은 쉽지만 항상 최적은 아니다.

---

### 4.3 Vertex와 index를 반드시 같은 allocation으로 묶을 필요는 없다

한 mesh chunk를 다음처럼 하나의 큰 AoS-like allocation으로 묶을 수 있다.

`[Vertex][Normal][Attribute][Index][Meshlet]`

관리하기는 쉽지만 일부 데이터만 갱신하거나 일부 consumer만 읽을 때 bandwidth locality가 나빠질 수 있다.

반대로 pool을 분리할 수 있다.

- Vertex position pool
- Normal/attribute pool
- Index pool
- Meshlet descriptor pool
- Cold debug/metrology pool

이때 relocation 단위도 달라진다.

예를 들어 index topology는 그대로이고 QEF vertex만 갱신되는 경우 vertex pool은 overwrite만 하고 index pool은 건드리지 않을 수 있다.

즉 allocator의 단위는 **“mesh object”** 보다 **“동일 lifetime과 access pattern을 공유하는 data stream”** 에 맞추는 것이 좋다.

---

### 4.4 Stable handle은 relocation의 핵심 contract다

Application이 다음을 직접 저장한다고 하자.

`vertexBase = 1,284,096`

Compaction 후 vertex chunk가 이동하면 이 값은 stale하다.

대신 logical handle을 둔다.

```text
SurfaceChunkHandle = {tableIndex, generation}
```

그리고 GPU-visible relocation table을 둔다.

```text
MeshChunkRecord {
    vertexOffset
    vertexCount
    indexOffset
    indexCount
    meshletOffset
    generation
}
```

Consumer는

`Handle -> MeshChunkRecord -> Physical Offset`

으로 해석한다.

이 구조의 장점은 physical relocation 시 **작은 table entry만 수정하면 logical identity는 그대로 유지**된다는 점이다.

이 패턴은 어제의 stable surface handle과 연결되며, 오늘은 이를 allocator-level relocation protocol로 확장한 것이다.

---

### 4.5 Indirection은 공짜가 아니다

Indirection table은 relocation을 쉽게 만들지만 매 draw/meshlet access에 추가 load가 생길 수 있다.

따라서 hot path에서는 다음을 고려한다.

- handle record를 compact하게 유지
- SoA 또는 cache-friendly AoSoA
- active chunk record만 작은 working set에 유지
- draw/meshlet preprocessing pass에서 physical offsets를 resolve
- shader가 매 vertex마다 handle table을 lookup하지 않도록 granularity를 coarse하게 유지

좋은 구조는 보통

`per-vertex indirection`

이 아니라

`per-mesh / per-brick / per-meshlet-range indirection`

이다.

즉 relocation flexibility와 lookup bandwidth 사이의 균형이 필요하다.

---

### 4.6 Buffer Device Address를 쓰면 relocation discipline이 더 중요해진다

Vulkan의 `vkGetBufferDeviceAddress()`는 buffer에 bind된 memory를 shader가 64-bit device address로 접근할 수 있게 한다.

이것은 pointer-like programming을 가능하게 하지만 relocation에는 위험이 있다.

두 경우를 구분해야 한다.

#### Case A: 같은 큰 VkBuffer 내부에서 subrange만 이동

큰 buffer object의 base device address는 유지될 수 있다.

하지만 cached pointer가

`base + oldOffset`

을 저장하고 있었다면 oldOffset이 stale하므로 여전히 잘못된 위치를 가리킨다.

#### Case B: resource 자체를 recreate/rebind

새 buffer 또는 새 binding은 address가 달라질 수 있다.

따라서 shader-visible absolute address를 persistent identity로 저장하면 defragmentation이 매우 어려워진다.

Relocation이 필요한 resource에서는

`logical handle -> current address/offset`

indirection을 유지하거나, absolute pointer의 lifetime을 relocation epoch 안으로 제한하는 편이 안전하다.

---

### 4.7 Indirect draw command도 relocation 대상의 consumer다

Indexed indirect draw는 `firstIndex`, `vertexOffset` 같은 physical placement 정보를 가진다.

Mesh chunk가 이동했는데 indirect command를 갱신하지 않으면 buffer 자체는 valid해도 잘못된 geometry를 읽는다.

따라서 relocation dependency graph에는 다음이 포함될 수 있다.

`allocation move -> handle table -> meshlet table -> indirect draw command -> renderer visibility`

두 가지 전략이 있다.

#### Patch strategy

이동한 chunk와 관련된 indirect command만 수정한다.

장점:
- 변경량이 적을 때 싸다.

단점:
- reverse mapping이 필요하다.

#### Regenerate strategy

GPU가 current handle table을 읽고 indirect command stream을 다시 생성한다.

장점:
- relocation logic이 단순해진다.
- GPU-driven rendering과 잘 맞는다.

단점:
- command regeneration bandwidth가 든다.

Dynamic scene에서는 **relocation rate와 draw-command count**에 따라 선택이 달라진다.

---

### 4.8 Defragmentation은 “move plan”과 “publish”를 분리해야 한다

Relocation은 다음 네 단계로 보는 편이 안전하다.

1. **Plan**
   - 어떤 allocation을 이동할지 선택
   - destination reservation
   - copy byte budget 계산

2. **Copy**
   - old range -> new range
   - vertex/index/meshlet payload 이동

3. **Publish**
   - handle table 또는 relocation table을 새 offset으로 전환

4. **Retire**
   - 이전 location을 더 이상 어떤 in-flight consumer도 읽지 않는 시점에 free

가장 위험한 것은 copy가 끝나기 전에 table을 publish하거나, old location을 너무 빨리 free하는 것이다.

따라서 physical memory의 lifetime과 logical handle의 visibility를 분리해야 한다.

---

### 4.9 Double-location window가 필요할 수 있다

Compaction 중에는 동일 logical object가 잠시 두 physical location을 가질 수 있다.

- old location: 현재 frame/in-flight draw가 읽고 있음
- new location: copy 완료 후 다음 epoch에서 publish 예정

이 상태를 허용하면 large global stall 없이 relocation을 진행할 수 있다.

메모리는 잠시 더 사용하지만 synchronization이 쉬워진다.

이를 **copy-on-relocate** 관점으로 볼 수 있다.

Peak memory pressure가 높으면 모든 allocation을 동시에 duplicate할 수 없으므로 relocation batch size를 제한해야 한다.

---

### 4.10 Relocation epoch가 fence보다 더 높은 의미를 가진다

GPU fence/timeline semaphore는 “copy가 완료되었다”를 알려준다.

하지만 rendering system이 알고 싶은 것은 보통 더 많다.

- 어느 handle table version이 valid한가
- 어떤 indirect command stream이 그 table을 기준으로 생성됐는가
- 어떤 frame이 old address를 아직 참조하는가

따라서 다음과 같은 semantic version이 유용하다.

`MeshPoolEpoch`

예를 들면:

- Epoch 41: old layout
- copy jobs 진행
- Epoch 42 relocation table 생성
- copy complete signal
- next frame이 Epoch 42 publish
- Epoch 41 consumer retire 후 old ranges free

즉 synchronization object는 timing primitive이고, epoch는 **resource interpretation contract**다.

---

### 4.11 Bandwidth-bounded compaction

Compaction을 한 번에 끝내려 하지 않는다.

Frame마다 copy budget을 둔다.

`MoveBytesThisFrame <= B_compact`

예를 들어 budget을 frame time이 아니라 **bytes**로 관리하는 이유는 compaction이 대체로 bandwidth-bound이기 때문이다.

더 나아가 dynamic budget을 사용할 수 있다.

- GPU frame time이 여유 있음 → budget 증가
- simulation/meshing bandwidth가 높음 → budget 감소
- allocation failure 위험 증가 → emergency budget 증가
- VRAM pressure 증가 → more aggressive compaction

핵심은 compaction을 background-like maintenance workload로 취급하는 것이다.

단, 실제 GPU command는 명시적으로 현재 frame/queue에 들어가며 별도의 “공짜 background bandwidth”가 존재하는 것은 아니다.

---

### 4.12 어떤 allocation을 먼저 옮길 것인가

단순히 address가 낮은 쪽부터 채우는 방식만이 정답은 아니다.

Priority score를 생각할 수 있다.

```text
Benefit =
    ReclaimedContiguousBytes
  + LocalityGain
  + AllocationFailureRiskReduction

Cost =
    CopyBytes
  + MetadataPatchBytes
  + SynchronizationRisk
  + InFlightLifetimeDelay
```

그리고 대략

`Priority = Benefit / Cost`

관점으로 볼 수 있다.

예:

- 매우 큰 hole을 하나로 합칠 수 있는 작은 chunk → 좋은 candidate
- 수백 MB를 옮겨야 하지만 reclaim이 적음 → 나쁜 candidate
- 자주 함께 렌더링되는 meshlets를 가까이 둘 수 있음 → locality benefit 추가
- 매 frame 수정되는 transient chunk → 이동 직후 다시 바뀔 가능성이 있어 낮은 priority

즉 defragmentation은 allocator problem이면서 scheduler problem이다.

---

### 4.13 “완전히 compact”보다 high-water mark가 더 유용할 수 있다

큰 linear backing buffer 안에서 active allocations이 앞쪽에 몰려 있다면 마지막 live byte를 **high-water mark**로 볼 수 있다.

Compaction 목표를

`used region = [0, highWaterMark)`

으로 줄이면 뒤쪽 large free range를 확보할 수 있다.

이 방식은 allocator 전체 entropy를 최소화하지 않아도

- resize
- buffer shrink
- large allocation
- sparse residency budget

에 유리하다.

즉 목표 metric을 “fragment count”보다 **largest usable free range** 또는 **high-water mark reduction**으로 잡을 수 있다.

---

### 4.14 Slab compaction은 class 내부와 class 간을 구분한다

Size-class allocator에서는 두 종류의 이동이 있다.

#### Intra-class relocation

같은 size class 내에서 hole을 채운다.

- destination capacity가 확실함
- metadata가 단순함
- move logic이 predictable함

#### Inter-class migration

payload size가 줄거나 증가하여 다른 class로 이동한다.

예:

`64 KB slab -> 32 KB slab`

이는 internal fragmentation을 줄일 수 있지만 allocation class와 handle record가 함께 바뀐다.

Dynamic mesh complexity가 자주 바뀐다면 **hysteresis**를 두는 편이 좋다.

예:

- 한 번 64 KB가 필요했다고 바로 128 KB로 permanently 승격하지 않음
- usage가 작아졌다고 매 frame 64 → 32 → 64 KB를 반복하지 않음

Allocator에도 temporal stability가 필요하다.

---

### 4.15 Mesh pool과 general-purpose GPU allocator는 계층이 다르다

Vulkan Memory Allocator(VMA) 같은 general-purpose allocator의 defragmentation과 application mesh suballocator를 구분해야 한다.

VMA는 `VkDeviceMemory` 수준 allocation을 관리하고 defragmentation move plan을 제공하지만, resource의 의미와 payload를 알지 못하므로 application이 buffer/image를 recreate하고 copy해야 한다. 현재 VMA 문서도 `maxBytesPerPass`, `maxAllocationsPerPass`로 incremental defragmentation budget을 제한할 수 있게 한다.

반면 mesh pool은 한 개 또는 몇 개의 큰 GPU buffer 안에서

- vertex range
- index range
- meshlet range

를 application이 suballocate하는 경우가 많다.

즉 두 계층이 동시에 존재할 수 있다.

`VkDeviceMemory allocator -> large VkBuffer -> application mesh pool -> mesh chunks`

아래 계층 fragmentation과 위 계층 fragmentation은 별개다.

---

### 4.16 CUDA stream-ordered memory pool과도 개념적으로 연결된다

CUDA의 `cudaMallocAsync` / stream-ordered memory allocator는 allocation/free를 stream ordering과 memory pool에 연결하여 frequent allocation의 global synchronization overhead를 줄이는 방향을 제공한다.

하지만 application mesh pool의 stable handle, vertex/index offsets, indirect draw dependency까지 자동으로 해결하는 것은 아니다.

즉 CUDA memory pool은 **physical allocation policy**이고, 오늘의 mesh relocation table은 **graphics object identity와 placement policy**다.

두 레벨을 섞지 않는 것이 중요하다.

---

### 4.17 Compute copy vs transfer/copy engine

Mesh relocation은 단순 byte copy라면 graphics API의 transfer operation이 자연스럽다.

하지만 relocation 과정에서 다음 작업을 함께 해야 한다면 compute가 유리할 수 있다.

- vertex format conversion
- index rebasing
- meshlet descriptor patch
- attribute repacking
- Morton/material reordering
- validation checksum

즉 선택은

`raw bytes 이동`

인지

`data transformation + relocation`

인지에 따라 달라진다.

Raw copy를 compute shader로 굳이 수행하면 copy engine이 할 일을 shader core와 memory pipeline이 대신 사용하게 될 수 있다.

---

### 4.18 Index rebasing을 피하는 layout이 유리할 수 있다

Mesh가 local index를 사용하고 draw command의 `vertexOffset`으로 base vertex를 제공한다면 vertex chunk relocation 때 index payload 자체를 수정하지 않아도 된다.

반대로 index buffer에 absolute global vertex index를 bake하면 vertex relocation 때 index 값까지 patch해야 할 수 있다.

따라서 relocation-friendly representation은 대체로

- local indices
- separate base vertex/offset
- handle-resolved physical base

처럼 **relative addressing**을 선호한다.

이것은 serialization과 compaction에서도 유리하다.

---

### 4.19 Meshlet도 같은 문제를 반복한다

Mesh shader pipeline에서는 traditional index draw 대신 meshlet descriptor가 다음을 가질 수 있다.

- vertex list offset
- primitive list offset
- bounds
- material/cluster metadata

Meshlet payload를 compact하면 descriptor의 offsets가 stale할 수 있다.

따라서 meshlet system도

`MeshletHandle -> MeshletRecord -> physical payload ranges`

로 설계할 수 있다.

특히 visibility culling output이 meshlet handle을 유지하면, physical compaction 후에도 culling identity를 바꾸지 않고 draw stream만 새 location으로 resolve할 수 있다.

---

### 4.20 C++ ownership은 세 종류를 분리해야 한다

C++ architecture에서는 다음을 다른 type으로 보는 것이 좋다.

#### Logical resource

`SurfaceChunkHandle`

- stable identity
- generation
- application/picking/metrology가 보유 가능

#### Allocation record

`MeshPoolAllocation`

- physical offset
- capacity
- size class
- pool epoch

#### GPU resource owner

`GpuMeshPoolBuffer`

- VkBuffer / CUDA external memory
- allocation lifetime
- queue synchronization
- recreation

이들을 하나의 struct에 섞으면 relocation 시 “object identity가 이동한 것인지, allocation만 이동한 것인지”가 모호해진다.

---

### 4.21 Relocation-safe debug metric

프로파일링에서는 다음을 함께 보는 것이 좋다.

- total pool capacity
- live bytes
- internal waste bytes
- total free bytes
- largest free block
- high-water mark
- allocation failure / retry count
- moved bytes/frame
- moved allocations/frame
- relocation copy time
- indirect-command patch bytes
- handle-table patch count
- stale-generation reject count
- old-epoch retained bytes
- average relocation lifetime
- L2 hit rate before/after compaction

중요한 점은 **defragmentation이 성공했는데 renderer가 느려질 수 있다**는 것이다.

예를 들어 memory utilization은 좋아졌지만 spatial/material locality를 무시하고 chunk를 재배치해서 L2 hit rate가 떨어질 수 있다.

그래서 allocator quality와 renderer locality를 함께 본다.

---

## 5. 내 관심 분야와 연결

현재 같은 **GPU-resident sparse volume → mesh extraction → rendering** 구조에서는 mesh pool이 자연스럽게 persistent system이 된다.

Semiconductor process visualization을 생각하면 process step마다 전체 geometry가 바뀌는 것이 아니라 특정 narrow-band region이 변한다. Incremental meshing으로 changed brick만 업데이트하면 성능은 좋아지지만, 장시간 실행할수록 다음 문제가 누적될 수 있다.

- trench 주변 mesh가 반복적으로 grow/shrink
- deposition layer 생성/삭제
- QEF refinement로 vertex count 변화
- material region별 mesh chunk lifetime 차이
- metrology/picking이 오래된 vertex index를 보유
- renderer가 indirect draw range를 캐시

오늘의 구조를 적용하면 다음처럼 생각할 수 있다.

`Sparse Brick`
→ `Stable SurfaceChunkHandle`
→ `Relocation Table`
→ `Vertex/Index/Meshlet Slab Pools`
→ `GPU-generated Draw Stream`

이때 CUDA가 mesh를 생산하고 Vulkan/WebGPU renderer가 소비한다면 relocation epoch는 cross-API ownership과도 연결된다.

- CUDA: new chunk 생성/compaction copy
- synchronization
- relocation table publish
- Vulkan: current epoch의 indirect draw/meshlet descriptor 소비
- old epoch retire

이 구조는 단순 mesh memory optimization이 아니라 **compute-render interop의 persistent data ownership architecture**다.

게임 엔진에서도 voxel destruction, procedural terrain, runtime marching cubes, particle/fluid surface, geometry streaming에 동일한 패턴이 적용된다.

---

## 6. 머릿속에 남길 질문 3개

1. **왜 “free memory가 충분하다”는 사실만으로 large mesh allocation이 성공한다고 보장할 수 없으며, internal fragmentation과 external fragmentation은 각각 어떤 allocator 선택에서 증가하는가?**

2. **Buffer Device Address나 indirect draw offset을 사용하는 renderer에서 mesh chunk relocation이 일어날 때, 어떤 consumer state가 stale해질 수 있고 logical handle indirection은 그 문제를 어떻게 줄이는가?**

3. **Compaction을 최대한 빨리 끝내는 것보다 `moved bytes/frame` budget을 두는 편이 왜 frame-time stability에 유리하며, 어떤 상황에서 emergency compaction을 허용해야 하는가?**

---

## 7. graphics engineer 면접 질문 1개와 답변

### 질문

**“Dynamic GPU mesh allocator에서 fragmentation이 심해졌습니다. 모든 live mesh를 한 번에 compact한 새 buffer로 복사하면 가장 깔끔하지 않나요?”**

### 답변

메모리 관점에서는 가장 깔끔하지만 real-time renderer에서는 항상 좋은 선택은 아니다.

첫째, full compaction은 live mesh 전체 크기만큼 device-memory traffic을 발생시키므로 보통 **bandwidth-bound spike**가 된다. 같은 frame에 shading, simulation, meshing도 memory bandwidth를 사용하고 있다면 큰 hitch가 생길 수 있다.

둘째, physical location이 바뀌면 vertex/index offset뿐 아니라 meshlet descriptor, indirect draw command, shader-visible device address, picking/metrology mapping 같은 **consumer metadata**도 함께 갱신해야 한다.

셋째, in-flight frame이 old layout을 아직 읽고 있다면 old storage를 즉시 free할 수 없다. 따라서 copy 완료와 publish, retire를 분리하고 fence/timeline과 mesh-pool epoch를 함께 관리해야 한다.

그래서 실무적인 구조는 대개 다음과 같다.

1. stable logical handle과 physical offset을 indirection table로 분리한다.
2. fragmentation과 largest-free-block/high-water-mark를 측정한다.
3. benefit이 큰 allocation부터 relocation candidate로 선택한다.
4. `max moved bytes/frame` 같은 bandwidth budget 안에서 incremental compaction한다.
5. copy 완료 후 새 relocation table epoch를 publish한다.
6. old epoch의 in-flight consumers가 끝난 뒤 source range를 reclaim한다.

즉 목표는 **perfect compaction**이 아니라 **relocation-safe, hitch-bounded memory maintenance**다.

---

## 8. 포트폴리오 / 커리어 연결

이 주제는 graphics engineer가 GPU algorithm뿐 아니라 **long-lived engine memory architecture**를 이해한다는 것을 보여주기 좋다.

포트폴리오에서 설명할 수 있는 포인트는 다음과 같다.

- **Allocator design:** slab/size-class, free-list, internal/external fragmentation
- **GPU memory:** bandwidth-bound compaction과 copy budget
- **Rendering pipeline:** vertex/index/meshlet/indirect command dependency
- **Shader architecture:** Buffer Device Address와 relocation-safe indirection
- **C++ design:** logical handle / allocation record / resource owner 분리
- **Synchronization:** copy → publish → retire, epoch와 fence의 역할 분리
- **GPU-driven rendering:** relocation 후 indirect command regeneration
- **Interop:** CUDA producer와 Vulkan consumer가 같은 mesh epoch를 공유
- **Profiling:** moved bytes, largest free block, L2 locality, retained old-epoch bytes

면접에서 “메모리 fragmentation이 생기면 defrag한다” 수준을 넘어,

> **“어떤 address가 stale해지고, 어떤 metadata를 patch해야 하며, 얼마나 옮길지 어떻게 budget하고, in-flight frame 때문에 old allocation을 언제 retire하는가”**

까지 설명할 수 있으면 engine/graphics systems 역량을 강하게 보여준다.

---

## 9. 내일 이어서 볼 개념

**Relocation-Safe GPU-Driven Rendering: Meshlet Tables, Indirect Commands, and Buffer Device Address Hygiene**

오늘은 physical mesh payload를 이동하면서 stable logical identity를 유지하는 allocator architecture를 봤다.

내일은 relocation이 끝난 뒤 renderer가 그 새로운 placement를 어떻게 안전하게 소비하는지로 이어간다.

학습 흐름은 다음과 같다.

`incremental meshing`
→ `persistent mesh pool`
→ `incremental defragmentation`
→ `relocation-safe meshlet/indirect rendering`

특히 다음을 본다.

- Buffer Device Address의 lifetime
- GPU pointer와 offset handle의 차이
- meshlet descriptor relocation
- indirect draw command regeneration
- descriptor/record epoch
- GPU-driven culling 결과의 stable identity
- relocation table caching과 L1/L2 locality

---

## 10. 참고 키워드

- GPU Memory Pool
- Dynamic Mesh Allocator
- Slab Allocation / Size Class
- Free List / Buddy Allocator
- Internal Fragmentation
- External Fragmentation
- Largest Free Block
- High-Water Mark
- Incremental Defragmentation
- Relocation Table
- Stable Logical Handle
- Generational Handle
- Bandwidth-Bounded Compaction
- Copy-on-Relocate
- Mesh Pool Epoch
- Vertex / Index Pool
- Meshlet Payload Pool
- Indirect Draw Command
- Base Vertex / Relative Indexing
- Vulkan Buffer Device Address
- `vkGetBufferDeviceAddress`
- `vkCmdDrawIndexedIndirect`
- CUDA Stream-Ordered Memory Allocator
- `cudaMallocAsync`
- Effective Memory Bandwidth
- L2 Locality
- GPU Stream Compaction
- VMA Defragmentation
- `maxBytesPerPass`
- `maxAllocationsPerPass`
- Vulkan Memory Allocator — Defragmentation
- Vulkan 1.4 Specification — Buffer Device Address / Indexed Draw
- CUDA C++ Best Practices Guide — Effective Bandwidth / Coalesced Global Memory Access
- CUDA Programming Guide — Stream-Ordered Memory Allocation
