---
title: "Relocation-Safe GPU-Driven Rendering: Meshlet Tables, Device-Address Indirect Commands, and Generation-Checked Handles"
date: "2026-09-07"
category: Graphics
tags: [GPU, Rendering, GPU-Driven Rendering, Meshlet, Mesh Shader, Indirect Draw, Buffer Device Address, Device Address Commands, Relocation, Generation Handle, CUDA, Vulkan, Memory Layout, C++]
level: intermediate
---

# [Daily Graphics Growth] 2026-09-07 - Relocation-Safe GPU-Driven Rendering: Meshlet Tables, Device-Address Indirect Commands, and Generation-Checked Handles

## 1. 오늘의 개념

어제는 **GPU Mesh Pool Defragmentation and Relocation**에서 dynamic mesh pool의 물리적 배치를 계속 바꾸면서도 logical identity를 유지하는 방법을 봤다.

오늘은 그 다음 소비자(consumer) 쪽 문제다.

> **mesh chunk가 relocation된 뒤에도 GPU-driven rendering이 오래된 offset/device address/indirect command를 참조하지 않게 하려면, render-side state를 어떤 형태로 설계해야 하는가?**

전통적인 CPU-driven renderer에서는 CPU가 draw 직전에 vertex/index buffer와 offset을 다시 바인딩하므로 relocation 후 reference 갱신 지점이 비교적 명확했다.

GPU-driven renderer에서는 다르다.

- compute shader가 visibility를 판단한다.
- meshlet worklist를 만든다.
- indirect draw/mesh-task command를 생성한다.
- shader가 table 또는 device address로 geometry를 fetch한다.
- CPU는 개별 draw의 physical location을 모를 수도 있다.
- 여러 frame이 동시에 in-flight일 수 있다.

즉 relocation 문제는 단순히 `offset` 하나를 고치는 일이 아니라 **GPU가 스스로 생성한 command와 shader-visible pointer까지 포함하는 snapshot-consistency 문제**가 된다.

특히 2026년에 ratify된 `VK_KHR_device_address_commands`는 기존의 일부 buffer-object 기반 명령을 **device-address range 기반 명령**으로 확장한다. 최신 Vulkan 문서에서는 `vkCmdDrawMeshTasksIndirectCountEXT`가 `vkCmdDrawMeshTasksIndirectCount2EXT`에 의해 superseded되었다고 명시하며, 새로운 형태는 draw parameter와 draw count를 buffer handle/offset 대신 address range에서 읽을 수 있다.

이 변화는 GPU-driven renderer를 더 pointer-oriented하게 만들 수 있지만, 동시에 **address lifetime과 relocation discipline을 더 명확히 요구**한다.

오늘은 다음 네 가지를 하나의 contract로 묶어서 본다.

1. **Meshlet table**로 logical work identity와 physical mesh storage를 분리
2. **Generation-checked handle**로 stale reference를 탐지
3. **Indirect command**에 무엇을 bake하고 무엇을 늦게 resolve할지 결정
4. **Buffer Device Address(BDA)** 를 persistent identity처럼 사용하지 않는 address hygiene

---

## 2. 한 줄 핵심

> Relocation-safe GPU-driven rendering의 핵심은 **persistent state에는 logical handle을 저장하고, physical offset/device address는 가능한 늦게 현재 generation의 table에서 resolve하며, indirect command와 meshlet worklist까지 동일한 relocation epoch에 속하도록 만드는 것**이다.

---

## 3. 왜 중요한가

GPU-driven rendering의 장점은 CPU submission overhead를 줄이고 visibility, LOD, compaction, draw generation을 GPU에 남기는 것이다.

하지만 dynamic geometry와 결합하면 CPU가 해결해주던 reference update 지점이 사라진다.

예를 들어 frame N에서 compute pass가 다음 command를 만들었다고 하자.

`draw.indexOffset = 1,200,000`

그 뒤 mesh pool compaction이 해당 index chunk를

`1,200,000 -> 820,000`

으로 이동했다.

만약 frame N의 indirect command가 아직 실행 전인데 handle table만 새 offset으로 publish하거나, 반대로 command만 새 주소를 보고 vertex table은 이전 주소를 본다면 renderer는 다음 중 하나를 겪을 수 있다.

- 잘못된 index range 접근
- unrelated mesh 출력
- out-of-bounds access
- device loss 가능성이 있는 invalid access
- picking/object ID와 화면 geometry 불일치
- 한 frame만 나타나는 temporal corruption

이런 bug는 CPU-side dangling pointer와 비슷하지만 GPU에서는 더 찾기 어렵다.

왜냐하면 reference가 여러 형태로 복제되기 때문이다.

- meshlet descriptor 내부 offset
- material/instance record의 mesh handle
- compacted visible worklist
- indirect draw command
- task/mesh shader payload
- push constant
- descriptor buffer
- raw `VkDeviceAddress`
- secondary command buffer 또는 in-flight primary command buffer

따라서 relocation-safe renderer는 **“어떤 데이터가 physical address의 copy를 보유하는가?”** 를 architecture 수준에서 추적해야 한다.

또 하나 중요한 이유는 최신 Vulkan의 흐름이다.

`VK_KHR_device_address_commands`는 2026-03-10 revision 1로 공개된 ratified extension이며, `vkCmdDrawIndexedIndirect2KHR`, `vkCmdDrawIndirectCount2KHR`, `vkCmdDrawMeshTasksIndirect2EXT`, `vkCmdDrawMeshTasksIndirectCount2EXT` 같은 명령을 통해 device address range를 API command input으로 직접 사용한다.

즉 앞으로는 shader만 BDA를 사용하는 것이 아니라 **command generation 자체에서도 address-oriented design이 더 중요해질 수 있다.**

---

## 4. 구현 관점

### 4.1 먼저 identity, placement, execution을 분리한다

Dynamic mesh object를 한 구조체로 모두 표현하면 relocation dependency가 퍼진다.

개념적으로 세 층으로 나누는 편이 좋다.

#### Identity layer

장기간 유지되는 logical identity다.

```text
MeshHandle {
    uint32 index;
    uint32 generation;
}
```

이 값은 selection, instance, scene graph, metrology, persistent simulation object가 보유할 수 있다.

#### Placement layer

현재 physical storage를 가리킨다.

```text
MeshRecord {
    uint vertexOffset;
    uint vertexCount;
    uint indexOffset;
    uint indexCount;
    uint meshletOffset;
    uint meshletCount;
    uint generation;
    uint relocationEpoch;
}
```

이 table은 relocation 때 갱신될 수 있다.

#### Execution layer

해당 frame에서 실제 실행할 work를 담는다.

```text
VisibleMeshlet {
    uint meshRecordIndex;
    uint meshletLocalIndex;
    uint expectedGeneration;
}
```

또는 이미 resolve된 physical work item일 수 있다.

핵심은 **persistent identity와 frame-local execution reference를 같은 lifetime으로 취급하지 않는 것**이다.

---

### 4.2 Meshlet table은 relocation barrier 역할을 할 수 있다

Mesh shader pipeline에서 meshlet은 보통 작은 vertex/primitive cluster다.

Meshlet descriptor는 다음처럼 생각할 수 있다.

```text
MeshletRecord {
    uint vertexDataOffset;
    uint primitiveDataOffset;
    uint vertexCount;
    uint primitiveCount;
    BoundingSphere bounds;
    Cone cullCone;
}
```

문제는 `vertexDataOffset`과 `primitiveDataOffset`가 relocation에 취약하다는 것이다.

두 가지 design이 있다.

#### Design A: Physical meshlet descriptor

meshlet record가 물리 offset/address를 직접 가진다.

장점:
- shader fetch가 빠르다.
- indirection이 적다.

단점:
- relocation 시 meshlet descriptor까지 rewrite해야 한다.
- visible worklist가 descriptor를 copy했다면 stale copy가 생긴다.

#### Design B: Logical meshlet descriptor

```text
MeshletRecord {
    uint meshHandle;
    uint localVertexOffset;
    uint localPrimitiveOffset;
    ...
}
```

shader가

`meshHandle -> MeshRecord -> physical base`

를 resolve한다.

장점:
- parent chunk relocation 시 meshlet table 전체를 고칠 필요가 없다.
- persistent meshlet identity가 안정적이다.

단점:
- 추가 table read가 필요하다.

실무에서는 흔히 완전한 logical/physical 양극단보다 **chunk-level indirection + local relative offset**이 좋은 균형이다.

즉:

`PhysicalAddress = ChunkBase(handle) + MeshletLocalOffset`

이 구조는 relocation update 범위를 chunk record 하나로 줄인다.

---

### 4.3 Absolute address보다 relative offset이 relocation에 강하다

다음 두 표현을 비교하자.

```text
Absolute:
meshlet.vertexAddress = 0x00007F1234560000
```

```text
Relative:
meshlet.vertexOffset = 16384
chunk.vertexBase = currentPhysicalBase
```

같은 chunk 내부에서 relocation이 일어나면 relative offset은 그대로 유지될 수 있다.

그래서 GPU data structure에서 pointer-rich graph보다

- stable base
- compact relative offset
- generation

조합이 relocation-friendly하다.

또 relative offset은 64-bit device address 대신 32-bit offset으로 표현 가능한 경우 metadata bandwidth도 줄일 수 있다.

단, pool 하나가 4 GiB를 넘거나 여러 address space를 묶으면 range design을 다시 봐야 한다.

---

### 4.4 Generation counter는 stale handle을 탐지한다

Free-list allocator에서 slot 42가 삭제됐다가 다른 mesh가 같은 slot을 재사용할 수 있다.

단순히

`handle.index = 42`

만 저장하면 stale reference가 새 object를 정상 object처럼 읽을 수 있다.

그래서 generation을 둔다.

```text
Handle = { index=42, generation=17 }

Table[42].generation = 17
```

삭제/재사용 후:

```text
Table[42].generation = 18
```

consumer는

`expectedGeneration == currentGeneration`

인지 확인할 수 있다.

GPU release build에서 모든 fetch마다 branch하는 것은 비쌀 수 있으므로 전략은 나눌 수 있다.

- debug build: shader에서 generation check
- culling/compaction pass: 한 번 validate 후 compact
- trusted hot path: validated worklist만 소비
- invalid handle counter: atomic diagnostic

즉 generation은 매 vertex validation보다 **work-granularity validation**에 더 잘 맞는다.

---

### 4.5 Relocation epoch와 object generation은 다른 값이다

두 개를 혼동하기 쉽다.

#### Object generation

logical slot이 free/reuse됐는지를 나타낸다.

`MeshHandle.index`가 같은 다른 object로 바뀌는 것을 검출한다.

#### Relocation epoch

같은 logical object의 physical placement가 바뀌었는지를 나타낸다.

object identity는 그대로지만 offset/address가 달라진다.

따라서 다음 상태가 가능하다.

```text
same generation
different relocationEpoch
```

GPU-driven renderer에서는 visibility worklist가 object generation은 맞지만 relocation 이전 physical location을 bake했을 수 있다.

그래서 **stale-object detection과 stale-placement detection은 별도의 문제**다.

---

### 4.6 Indirect command에는 가능한 한 identity를 덜 bake한다

`VkDrawIndexedIndirectCommand`에는 다음 값이 들어간다.

- `indexCount`
- `instanceCount`
- `firstIndex`
- `vertexOffset`
- `firstInstance`

여기서 `firstIndex`와 `vertexOffset`는 physical buffer placement와 강하게 결합될 수 있다.

따라서 dynamic mesh pool에서는 두 가지 흐름을 비교해야 한다.

#### Early resolve

Culling pass에서 physical offset을 읽고 완성된 indirect command를 만든다.

```text
Handle
 -> MeshRecord
 -> VkDrawIndexedIndirectCommand
```

장점:
- draw 단계가 단순하다.
- classic vertex/index pipeline에 잘 맞는다.

단점:
- command 생성 뒤 relocation이 일어나면 stale.

#### Late resolve

Visible handle/meshlet list만 만들고, draw command를 relocation publish 이후의 별도 pass에서 생성한다.

```text
Visibility
 -> Logical Visible List
 -> Relocation Publish
 -> Physical Command Build
 -> Draw
```

장점:
- stale window가 짧다.
- compaction과 render scheduling을 분리할 수 있다.

단점:
- pass와 synchronization이 하나 늘어날 수 있다.

핵심은 **“physical value가 생성되는 시점을 가능한 consumer 가까이 미룬다”** 는 것이다.

---

### 4.7 Mesh shader는 relocation-friendly data model을 만들기 쉽다

Classic indexed draw는 index/vertex buffer binding과 `firstIndex`, `vertexOffset`에 많이 의존한다.

Mesh shader는 workgroup이 필요한 geometry를 storage buffer/device address를 통해 명시적으로 fetch하고 직접 primitives를 emit한다.

그래서 다음 구조가 자연스럽다.

`VisibleMeshletID -> MeshletTable -> ChunkTable -> vertex/primitive data`

즉 draw command는 단순히 “몇 개 mesh task workgroup을 launch할 것인가”를 결정하고, 실제 geometry 위치는 mesh shader가 table에서 resolve할 수 있다.

`VkDrawMeshTasksIndirectCommandEXT`도 command 자체는 `groupCountX/Y/Z`만 담는다.

이 특성은 relocation에 유리하다.

왜냐하면 command buffer에 geometry offset을 직접 bake하지 않아도 되기 때문이다.

다만 shader-side table lookup이 stale하면 문제는 그대로이므로 **mesh shader를 사용한다고 자동으로 relocation-safe가 되는 것은 아니다.**

---

### 4.8 2026 `VK_KHR_device_address_commands`가 바꾸는 점

기존 indirect draw command는 보통 `VkBuffer + offset` 조합으로 parameter buffer를 지정한다.

`VK_KHR_device_address_commands`는 device address range를 직접 사용하는 명령군을 추가한다.

대표적으로:

- `vkCmdDrawIndirect2KHR`
- `vkCmdDrawIndexedIndirect2KHR`
- `vkCmdDrawIndirectCount2KHR`
- `vkCmdDrawIndexedIndirectCount2KHR`
- `vkCmdDrawMeshTasksIndirect2EXT`
- `vkCmdDrawMeshTasksIndirectCount2EXT`
- `vkCmdDispatchIndirect2KHR`

`VkDrawIndirectCount2InfoKHR`는 draw parameter를 위한 `VkStridedDeviceAddressRangeKHR`와 draw count를 위한 `VkDeviceAddressRangeKHR`를 가진다.

이 방식은 여러 buffer object를 API-level에서 계속 들고 다니지 않고 device address 중심으로 command data를 표현할 수 있게 한다.

하지만 relocation 관점에서는 다음 질문이 더 중요해진다.

> **그 address range는 언제 resolve됐고, 실행 시점까지 그 backing range가 같은 의미를 유지하는가?**

address 기반 API는 handle lookup 비용을 줄일 수 있지만, stale pointer 문제를 없애지는 않는다.

오히려 ownership/lifetime이 명시적이지 않으면 physical placement가 더 넓게 복제될 수 있다.

---

### 4.9 Buffer Device Address는 object ID가 아니다

Vulkan `vkGetBufferDeviceAddress`가 반환하는 값은 buffer 시작점의 64-bit device address이며, 해당 buffer의 address range를 통해 bound memory를 접근할 수 있다.

이 값을 다음처럼 취급하면 위험하다.

```text
PersistentMeshID = VkDeviceAddress
```

왜냐하면 mesh data relocation과 logical object identity는 같은 개념이 아니기 때문이다.

특히 큰 buffer 내부 subrange relocation에서는 buffer base address가 같아도

`base + oldOffset`

이 stale하다.

resource를 recreate/rebind하는 architecture에서는 새 device address를 다시 resolve해야 할 수도 있다.

따라서 권장 mental model은 다음에 가깝다.

> `VkDeviceAddress`는 **현재 physical snapshot을 위한 access token**이지, application object의 장기 identity가 아니다.

---

### 4.10 Pointer chain은 relocation blast radius를 키운다

GPU에서 다음처럼 pointer chain을 만들 수 있다.

```text
Instance -> Mesh*
Mesh -> Meshlet*
Meshlet -> Vertex*
```

읽기는 편하지만 Mesh 또는 Meshlet storage가 이동하면 그 주소를 참조하는 모든 upstream node를 고쳐야 한다.

이를 **relocation blast radius**라고 생각할 수 있다.

대안은:

```text
Instance -> MeshHandle
MeshHandle -> MeshRecord
Meshlet -> localOffset
```

이 경우 relocation 시 수정 지점이 central table에 모인다.

Pointer-rich layout이 항상 나쁜 것은 아니다.

- static BLAS-like structures
- immutable asset heap
- long-lived arena
- capture/replay-oriented stable address design

에서는 유리할 수 있다.

하지만 high-churn dynamic mesh에서는 **centralized indirection이 update locality를 개선**한다.

---

### 4.11 Descriptor도 relocation dependency가 될 수 있다

Buffer range를 descriptor로 표현한다면 relocation 뒤 descriptor가 old buffer/range를 가리킬 수 있다.

특히 다음 경우를 구분해야 한다.

- descriptor가 whole pool을 가리키고 shader가 dynamic offset/table lookup 사용
- descriptor가 mesh chunk별 subrange를 직접 가리킴
- descriptor buffer에 resource descriptor가 저장됨
- shader가 raw BDA를 table에서 읽음

relocation-friendly한 방향은 보통 **descriptor granularity를 allocation granularity보다 크게 유지**하는 것이다.

예:

`descriptor -> large persistent pool`

`handle table -> per-chunk offset`

그러면 suballocation relocation마다 descriptor를 rewrite하지 않아도 된다.

즉 descriptor layout에서도 같은 원칙이 반복된다.

> 자주 움직이는 state는 작은 table entry에 모으고, heavyweight binding state는 가능한 안정적으로 유지한다.

---

### 4.12 Visibility 결과와 physical placement를 분리한다

GPU culling 결과가 다음처럼 physical meshlet address를 저장한다고 하자.

```text
VisibleItem {
    uint64 meshletAddress;
}
```

relocation 후 이 list는 invalid하다.

대신:

```text
VisibleItem {
    uint meshHandle;
    uint meshletLocalIndex;
    uint expectedGeneration;
}
```

처럼 logical result를 저장하면 relocation 이후에도 다시 resolve할 수 있다.

이 구조는 visibility와 memory placement의 lifetime을 decouple한다.

중요한 시스템적 장점은:

- camera가 같으면 visibility logical result 재사용 가능성
- relocation과 culling을 독립 scheduling 가능
- debug capture가 deterministic해짐
- physical allocator 변경이 culling algorithm에 덜 전파됨

이다.

---

### 4.13 하지만 late indirection도 bandwidth cost가 있다

모든 것을 logical handle로 만들면 relocation에는 강하지만 rendering hot path의 table lookup이 늘어난다.

예를 들어 mesh shader workgroup마다 다음을 읽는다고 하자.

1. visible item
2. mesh record
3. meshlet descriptor
4. vertex data
5. primitive data
6. material record

이때 front-end가 pointer chasing으로 latency-bound가 될 수 있다.

그래서 production renderer에서는 **resolve boundary**를 profiling해야 한다.

가능한 구조:

#### Option A: fully logical

relocation flexibility 최우선.

#### Option B: frame-local physical cache

frame begin에 logical handle을 physical descriptor로 resolve하고 그 frame 동안 immutable하게 사용.

#### Option C: hybrid

mesh-level base는 indirection, meshlet-local data는 relative offset.

대부분의 dynamic geometry에서는 Option C가 좋은 출발점이다.

---

### 4.14 Frame-local physical cache는 relocation-safe fast path다

매 shader invocation마다 full handle table을 따라가는 대신 frame-local resolved table을 만들 수 있다.

예:

```text
FrameMeshRecord {
    uint64 vertexBase;
    uint64 primitiveBase;
    uint meshletBase;
    uint materialIndex;
}
```

이 table은 특정 **render snapshot epoch**에 대해 immutable하다.

흐름:

```text
Persistent Handle Table
        ↓ resolve
Frame N Resolved Table
        ↓
Cull / Meshlet Worklist / Draw
```

동시에 allocator는 다음 epoch의 relocation을 준비할 수 있다.

즉 CPU의 frame resource ring과 비슷하게 GPU에서도

`persistent logical state`

와

`frame-local resolved physical state`

를 분리할 수 있다.

---

### 4.15 Publish는 copy 완료와 같은 의미가 아니다

Relocation sequence를 생각하자.

1. destination allocation
2. old -> new data copy
3. table entry의 new address 작성
4. renderer가 new table 읽음
5. old allocation free

여기서 가장 위험한 것은 3번을 너무 일찍 publish하는 것이다.

copy가 완료되지 않았거나 visibility/indirect command가 old/new snapshot을 섞어 읽으면 semantic corruption이 생긴다.

따라서 relocation protocol은 최소한 다음 상태를 구분해야 한다.

```text
Allocated
CopyScheduled
CopyComplete
Published
OldRetired
```

Vulkan synchronization barrier/semaphore는 memory visibility를 보장하지만, **어느 table version이 어느 command/worklist와 논리적으로 묶이는가**는 application protocol이 결정해야 한다.

---

### 4.16 In-flight frame 때문에 old address를 즉시 free할 수 없다

Frame N이 old mesh record를 참조하고 있고 frame N+1이 new record를 참조할 수 있다.

따라서 publish 직후 old range를 allocator free-list에 반환하면 안 된다.

필요한 것은 **deferred retirement**다.

예:

```text
RetireQueue {
    allocation
    safeAfterTimelineValue
}
```

Timeline semaphore 또는 engine frame fence가 특정 값에 도달한 뒤에만 old allocation을 재사용한다.

이것은 CPU RCU(Read-Copy-Update)와 비슷한 사고방식이다.

- reader는 immutable snapshot을 본다.
- writer는 new version을 준비한다.
- publish한다.
- old reader가 끝난 뒤 old version을 reclaim한다.

GPU persistent geometry에서도 매우 유용한 mental model이다.

---

### 4.17 CUDA -> Vulkan interop에서도 logical epoch가 필요하다

CUDA가 mesh data를 생성/compact하고 Vulkan이 render한다고 하자.

단순히 external semaphore 하나로

`CUDA done -> Vulkan start`

를 만들 수 있다.

하지만 다음 resource가 같은 version인지 별도로 확인해야 한다.

- vertex/index/meshlet payload
- handle/relocation table
- visible worklist
- indirect draw count
- indirect command buffer
- bounds/culling metadata

즉 synchronization object는 **execution order**를 전달하고, epoch/generation은 **semantic version**을 전달한다.

둘을 같은 문제로 보면 debug가 어렵다.

---

### 4.18 Indirect Count는 work generation과 잘 맞지만 snapshot rule이 필요하다

`vkCmdDrawIndexedIndirectCount`는 count buffer의 32-bit 값을 GPU에서 읽어 실제 draw 수를 결정한다.

Mesh shader의 `vkCmdDrawMeshTasksIndirectCountEXT`도 같은 아이디어로 mesh task draw count를 device buffer에서 읽는다.

최신 refpage에서는 이 mesh-task count command가 address-range 기반 `vkCmdDrawMeshTasksIndirectCount2EXT`에 의해 superseded되었다고 명시한다.

GPU pipeline 관점에서는 매우 자연스럽다.

```text
Cull
 -> Compact Visible Work
 -> Write Indirect Commands
 -> Write Draw Count
 -> Barrier
 -> Indirect Count Draw
```

relocation-safe 조건은 다음이다.

> visible work, command parameters, count, geometry table이 **같은 render snapshot**을 설명해야 한다.

count만 최신이고 command array가 이전 epoch인 경우도 logically invalid하다.

---

### 4.19 Debug build에서는 address provenance를 추적할 가치가 있다

Raw BDA bug는 CPU debugger에서 object name을 잃기 쉽다.

그래서 debug metadata를 둘 수 있다.

```text
AddressRecord {
    uint64 begin;
    uint64 end;
    uint handle;
    uint generation;
    uint relocationEpoch;
    uint allocationClass;
}
```

GPU fault/crash dump의 address를 이 range table과 대조하면

- 어떤 logical mesh였는가
- relocation 전/후 어느 epoch였는가
- 이미 retired된 allocation인가
- out-of-range offset인가

를 추적하기 쉬워진다.

Production에서는 table 전체를 유지하지 않더라도 development build에서 매우 가치가 있다.

---

### 4.20 Physical pointer를 저장해도 되는 기준

모든 raw address를 금지할 필요는 없다.

다음 조건을 만족하면 physical pointer가 합리적일 수 있다.

- allocation lifetime이 consumer보다 확실히 길다.
- 해당 allocation은 relocation 대상이 아니다.
- frame snapshot 안에서 immutable하다.
- ownership이 명확하다.
- address를 보유한 consumer 집합이 제한적이다.

예를 들면 frame-local resolved meshlet buffer는 raw address를 저장해도 좋을 수 있다.

반대로 다음은 위험 신호다.

- persistent scene state가 raw BDA 저장
- resize/defrag 가능한 pool의 suballocation address 저장
- 여러 in-flight frame이 같은 mutable address table 공유
- generation 없이 slot index만 저장
- indirect command 생성과 relocation이 같은 frame에서 unordered하게 진행

---

### 4.21 Meshlet table의 SoA/AoS는 access stage에 맞춘다

Culling pass가 필요한 정보:

- bounding sphere/cone
- LOD error
- mesh handle
- generation

Mesh shader가 필요한 정보:

- vertex offset
- primitive offset
- count

두 consumer가 읽는 데이터가 다르다.

따라서 하나의 큰 AoS:

```text
Meshlet {
    Bounds
    Cone
    LOD
    MeshHandle
    VertexOffset
    PrimitiveOffset
    Counts
    Debug...
}
```

보다 hot path를 분리할 수 있다.

```text
CullMeshletSOA
RenderMeshletSOA
ColdMeshletMetadata
```

Relocation 관련 field도 render table에만 두면 culling cache footprint가 작아진다.

즉 **relocation-safe architecture는 pointer semantics뿐 아니라 memory-layout optimization과 연결**된다.

---

### 4.22 C++ API도 raw offset의 확산을 막아야 한다

C++ renderer에서 다음 type이 모두 `uint64_t`면 semantic confusion이 생긴다.

```cpp
uint64_t handle;
uint64_t deviceAddress;
uint64_t byteOffset;
uint64_t epoch;
```

strong type을 두는 편이 좋다.

```text
MeshHandle
MeshGeneration
RelocationEpoch
DeviceAddress
ByteOffset
```

C++ type system으로 logical/physical 값을 구분하면

- persistent object에 `DeviceAddress`를 저장하는 코드
- generation 없는 handle copy
- old epoch indirect buffer 재사용

같은 실수를 code review에서 잡기 쉬워진다.

Graphics engine의 correctness는 shader뿐 아니라 **host-side semantic typing**에서도 나온다.

---

### 4.23 Profiling에서 봐야 할 것

Relocation-safe indirection은 correctness만 보고 끝나면 안 된다.

다음 metric을 같이 본다.

- handle-table bytes/frame
- meshlet-table L1/L2 hit rate
- average indirection depth
- visible meshlet count
- physical resolve pass time
- indirect command build time
- relocation bytes/frame
- retired-but-not-yet-reclaimable bytes
- stale-generation reject count
- relocation-epoch mismatch count
- meshlet payload locality
- draw/mesh-task indirect count
- task/mesh shader occupancy
- descriptor/table cache pressure

특히 중요한 질문은:

> relocation을 쉽게 만들기 위해 추가한 indirection의 bandwidth가, compaction으로 얻은 locality 이득보다 큰가?

이다.

---

## 5. 내 관심 분야와 연결

Dynamic sparse SDF에서 만들어지는 surface mesh는 static game asset보다 relocation churn이 훨씬 크다.

Semiconductor process visualization을 예로 들면 다음 step마다 geometry가 달라질 수 있다.

- deposition
- etch
- CMP
- oxidation
- lithography mask change
- level-set evolution

Incremental meshing으로 changed brick만 업데이트하면 mesh chunk의 크기도 계속 달라진다. 따라서 어제 다룬 mesh-pool compaction이 실제로 발생하고, 오늘의 relocation-safe render contract가 필요해진다.

가능한 pipeline은 다음과 같다.

```text
CUDA / GPU Compute
    ↓
Changed SDF Bricks
    ↓
Incremental Mesh / Meshlet Generation
    ↓
Persistent Mesh Pool
    ↓
Logical Mesh/Surface Handle Table
    ↓
Frame-Local Resolved Meshlet Table
    ↓
Visibility / LOD / Compaction
    ↓
Indirect Mesh Task Commands
    ↓
Vulkan Rendering
```

이 구조에서 중요한 점은 **SDF brick ID, mesh logical ID, physical mesh-pool slot, Vulkan device address를 서로 다른 identity domain으로 유지하는 것**이다.

또 Babylon.js/WebGPU 같은 higher-level renderer로 결과를 전달하는 경우에도 같은 원리가 적용된다.

WebGPU에서는 raw BDA가 직접 노출되지 않더라도 buffer suballocation offset과 bind-group/resource lifetime이 physical placement 역할을 한다. 즉 API가 pointer를 숨긴다고 relocation identity 문제가 사라지는 것은 아니다.

게임 엔진 관점에서는 다음과 연결된다.

- runtime destructible geometry
- voxel terrain
- fluid surface extraction
- procedural mesh
- GPU particle ribbon/trail mesh
- virtualized geometry
- meshlet streaming
- scene compaction

특히 엔진 팀에서 중요한 것은 특정 Vulkan extension 이름을 외우는 것보다 **GPU-generated work와 dynamic memory manager 사이의 lifetime contract를 설명할 수 있는가**다.

---

## 6. 머릿속에 남길 질문 3개

1. **왜 `VkDeviceAddress`를 persistent object identity처럼 저장하면 위험하며, logical handle + generation + relative offset 구조가 relocation blast radius를 줄이는가?**
2. **GPU visibility pass가 physical meshlet address를 출력하는 구조와 logical meshlet handle을 출력하는 구조는 relocation, bandwidth, latency 측면에서 어떤 trade-off가 있는가?**
3. **indirect command, draw count, meshlet table, geometry payload가 각각 다른 시점에 생성될 때 “같은 frame”이라는 사실만으로 semantic consistency가 보장되지 않는 이유는 무엇인가?**

---

## 7. graphics engineer 면접 질문 1개와 답변

### 질문

**“GPU-driven renderer에서 mesh buffer를 defragment한 뒤 handle table의 device address만 새 값으로 바꾸면 충분한가요?”**

### 답변

충분하지 않을 수 있다.

Handle table이 geometry의 physical location을 유일하게 resolve하는 지점이고, 모든 downstream consumer가 draw 직전에 그 table을 읽는다면 table entry update만으로 충분할 수 있다.

하지만 실제 GPU-driven renderer에서는 physical state가 여러 곳에 bake될 수 있다.

예를 들면:

- 이전 culling pass가 만든 visible meshlet list
- `VkDrawIndexedIndirectCommand`의 `firstIndex`/`vertexOffset`
- raw `VkDeviceAddress`가 들어간 frame-local descriptor
- task/mesh shader용 work record
- 이미 recording된/in-flight command가 참조하는 indirect buffer

이들이 relocation 이전 physical placement를 포함하면 handle table만 새 주소로 바꿔도 old/new snapshot이 섞인다.

그래서 robust한 설계는:

1. persistent state에는 logical handle을 저장하고,
2. generation으로 slot reuse를 검증하고,
3. physical address/offset은 가능한 late resolve하며,
4. relocation epoch와 render snapshot epoch를 맞추고,
5. old allocation은 모든 in-flight reader가 끝난 뒤 retire한다.

즉 문제는 단순 pointer update가 아니라 **reference replication과 lifetime synchronization** 문제다.

---

## 8. 포트폴리오 / 커리어 연결

이 주제는 graphics engineer 포트폴리오에서 **GPU algorithm과 systems design을 동시에 보여주기 좋다.**

설명할 수 있는 포인트는 다음과 같다.

- **GPU-driven rendering:** visibility → compaction → indirect draw/mesh-task generation
- **Mesh shader architecture:** meshlet 단위 fetch와 workgroup-based primitive emission
- **Memory manager:** dynamic pool relocation과 rendering consumer의 coupling
- **Data design:** logical handle, generation, relative offset, frame-local physical cache
- **Vulkan:** indirect count, mesh shader, Buffer Device Address, 2026 `VK_KHR_device_address_commands`
- **Synchronization:** copy/publish/retire와 timeline-based deferred reclamation
- **C++ architecture:** strong type으로 logical/physical reference 분리
- **Debugging:** stale generation, relocation epoch, address provenance 추적
- **Performance:** extra indirection bandwidth와 better compaction/locality의 trade-off
- **Interop:** CUDA producer → Vulkan GPU-driven consumer snapshot contract

면접에서 좋은 답변은 단순히

> “BDA는 빠르지만 위험하다.”

가 아니다.

더 강한 설명은:

> “Pointer 자체가 문제가 아니라 **physical placement의 lifetime보다 pointer copy의 lifetime이 길어지는 순간**이 문제이며, 그래서 dynamic renderer에서는 address를 resolve하는 boundary와 reclamation epoch를 architecture로 관리해야 한다.”

이다.

이 사고방식은 Vulkan뿐 아니라 DirectX 12 GPU virtual address, CUDA device pointer, Metal buffer+offset, WebGPU suballocation에도 확장된다.

---

## 9. 내일 이어서 볼 개념

**GPU-Driven Meshlet Visibility Pipelines: Hierarchical Culling, Worklist Compaction, and Indirect Count Synchronization**

오늘은 relocation 이후에도 renderer가 올바른 physical geometry snapshot을 읽는 방법을 봤다.

내일은 그 안정적인 meshlet table을 입력으로 사용해 **어떤 meshlet을 실제로 실행할 것인지**를 줄이는 GPU-driven visibility pipeline으로 이어간다.

연결 흐름은 다음과 같다.

`incremental meshing → mesh-pool relocation → relocation-safe render tables → hierarchical meshlet culling → compacted indirect work`

다음 노트에서는:

- instance/meshlet two-stage culling
- frustum / cone / Hi-Z occlusion
- prefix-sum vs atomic worklist append
- task shader와 compute culling의 역할 분리
- indirect count buffer
- temporal visibility coherence
- false-negative를 피하는 conservative bounds
- work amplification과 occupancy

를 중심으로 본다.

---

## 10. 참고 키워드

- GPU-Driven Rendering
- Meshlet
- Mesh Shader / Task Shader
- `VK_EXT_mesh_shader`
- `VkDrawMeshTasksIndirectCommandEXT`
- `vkCmdDrawMeshTasksIndirectCountEXT`
- `vkCmdDrawMeshTasksIndirectCount2EXT`
- `VK_KHR_device_address_commands`
- `VkDrawIndirectCount2InfoKHR`
- `VkStridedDeviceAddressRangeKHR`
- `VK_KHR_buffer_device_address`
- `vkGetBufferDeviceAddress`
- Buffer Device Address (BDA)
- Device Address Range
- Indirect Draw / Indirect Count
- Logical Handle
- Generation Counter
- Relocation Epoch
- Frame Snapshot
- Relative Offset
- Pointer Chasing
- Address Provenance
- Deferred Retirement
- Timeline Semaphore
- Read-Copy-Update (RCU) mental model
- Meshlet Table
- Visibility Worklist
- GPU Stream Compaction
- CUDA-Vulkan Interop
- Descriptor Buffer
- Hot/Cold Metadata
- Strong Types in C++
- Vulkan Documentation Project, **VK_KHR_device_address_commands** — ratified extension, Revision 1, 2026-03-10
- Vulkan Documentation Project, **vkCmdDrawMeshTasksIndirectCount2EXT**
- Vulkan Documentation Project, **vkGetBufferDeviceAddress**
- Vulkan Documentation Project, **VK_EXT_mesh_shader / Mesh Shading**
- Khronos Vulkan Samples, **Mesh Shader Culling**
