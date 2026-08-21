---
title: "GPU Data-Oriented Scene State: Sparse Sets, Generational Handles, and Multi-Buffer Residency for Rendering/Simulation Interop"
date: "2026-08-21"
category: Graphics
tags: [GPU, Data-Oriented Design, Sparse Set, Generational Handle, Scene State, CUDA, Vulkan, DirectX12, C++]
level: intermediate
---

# [Daily Graphics Growth] 2026-08-21 - GPU Data-Oriented Scene State: Sparse Sets, Generational Handles, and Multi-Buffer Residency for Rendering/Simulation Interop

## 1. 오늘의 개념
어제는 compact한 `PathState`를 재정렬해 physical locality를 회복하는 문제를 봤다. 오늘은 그 원칙을 path tracer의 일시적 상태에서 scene-wide persistent state로 확장한다.

핵심은 scene object의 **stable logical identity**와 GPU의 **dense physical storage**를 분리하는 것이다. Sparse Set은 `sparse[entity] -> denseSlot`과 `denseEntity[denseSlot] -> entity`의 양방향 관계를 통해 빠른 membership lookup과 dense traversal을 함께 제공한다. Generational Handle은 대략 `[generation | index]`로 볼 수 있으며, slot이 재사용될 때 generation을 바꿔 stale reference가 새 object를 잘못 가리키는 것을 막는다.

마지막 축은 **Multi-Buffer Residency**다. CPU update, simulation compute, graphics가 서로 다른 시간의 scene state를 동시에 다룰 수 있도록 scene snapshot을 version별 buffer로 분리한다. 여기서 residency는 software-level scene version의 유효 위치를 뜻하며, D3D12의 physical memory residency와는 구분한다.

## 2. 한 줄 핵심
> GPU scene architecture에서는 identity는 stable handle로, placement는 dense slot으로, concurrency는 versioned buffer로 분리해야 compaction·streaming·simulation·rendering을 서로 깨뜨리지 않고 연결할 수 있다.

## 3. 왜 중요한가
렌더러와 simulation이 커지면 Transform, Material, Bounds, Geometry Reference, Visibility, Particle/Voxel State처럼 update frequency와 access pattern이 다른 데이터가 동시에 존재한다.

raw dense index를 identity로 쓰면 compaction 이후 의미가 바뀐다. raw pointer나 GPU device address를 영구 identity로 쓰면 buffer reallocation과 relocation이 어려워진다. 반대로 모든 데이터를 stable 위치에 고정하면 fragmentation과 poor locality가 생긴다.

Sparse Set + Generational Handle은 이 충돌을 분리한다. **handle은 object가 누구인지**, **dense slot은 지금 어디에 저장되는지**를 담당한다. 따라서 physical layout은 material, spatial locality, visibility에 따라 재배치하면서 외부 reference는 유지할 수 있다.

또한 한 mutable buffer를 CPU·compute·graphics가 동시에 사용하면 이전 frame을 읽는 GPU와 다음 frame을 쓰는 producer가 충돌한다. Frame-local snapshot 또는 versioned scene buffer는 temporal ownership을 분리하고, fence/semaphore가 version handoff를 명시한다.

## 4. 구현 관점
### Identity layer
개념적 handle은 `SceneHandle = { index, generation }`이다. `index`는 handle table entry를 찾고, `generation`은 그 entry가 여전히 같은 logical object인지 검증한다. object가 제거되고 slot이 recycle되면 generation이 증가한다.

핵심 경로는 다음과 같다.

`SceneHandle -> HandleTable -> DenseSlot -> Component SoA`

이 구조에서는 dense storage를 compact/reorder해도 handle table의 mapping만 갱신하면 logical identity가 유지된다.

### Storage layer
GPU hot path는 object graph보다 dense SoA(Structure of Arrays)가 유리한 경우가 많다. 예를 들면 `Transform[]`, `Bounds[]`, `MaterialId[]`, `GeometryHandle[]`, `SimulationStateRef[]`, `VisibilityMask[]`를 독립된 배열로 둘 수 있다.

**SoA는 field locality**, **Sparse Set은 logical-to-dense mapping**, **sorting/binning은 entity ordering**을 해결한다. 세 문제는 서로 다른 층이므로 하나로 대체되지 않는다.

### Device address는 identity가 아니다
Vulkan Buffer Device Address나 CUDA pointer는 shader/compute에서 빠른 접근에 유용하지만, 장기 logical identity로 쓰면 relocation이 어렵다. 장기 reference에는 `SceneHandle` 또는 compact token을 사용하고, 실행 시점에 current buffer address/offset으로 resolve하는 편이 relocation-safe하다.

### Multi-buffer residency
중요한 것은 buffer 개수가 아니라 **각 version의 ownership**이다. 개념적으로 다음 상태가 동시에 있을 수 있다.

- `Scene[N]`: graphics가 현재 frame에서 읽는 read-only snapshot
- `Scene[N+1]`: compute/simulation이 다음 state를 작성 중인 snapshot
- `Upload/Staging`: CPU가 dirty data를 준비하는 영역

version마다 fence/semaphore value를 연결하면 `producer completed -> consumer visible` 관계를 명확히 할 수 있다. Vulkan의 semaphore/fence와 D3D12의 fence-based resource management가 이 temporal lifetime 문제와 직접 연결된다.

### Dirty propagation
매 frame scene 전체를 복사하지 않고 `dirty bitset`, changed range, epoch를 추적하면 변경된 dense region만 다음 GPU snapshot에 반영할 수 있다. 대규모 scene에서는 `CPU authoritative -> upload pending -> GPU version resident -> retired` 같은 상태 머신으로 이해할 수 있다.

### Physical residency와 구분
D3D12의 `MakeResident`/`Evict`는 VRAM budget과 pageable resource의 physical memory residency를 다룬다. 오늘의 scene residency는 그보다 상위 계층의 logical version 문제다. resource가 VRAM에 resident하더라도 그 안의 scene entry가 current version이라는 보장은 없다.

### C++ / Render Graph contract
`SceneHandle`, `DenseSlot`, `Generation`, `SceneVersion`, `ResourceEpoch`를 다른 타입/개념으로 분리하면 architecture가 명확해진다. Render Graph는 어떤 pass가 어떤 SceneVersion을 read/write하는지 표현하고, 실제 API barrier/fence/semaphore가 queue ownership과 memory visibility를 보장한다.

## 5. 내 관심 분야와 연결
C++ graphics engine에서는 entity를 raw pointer graph로 묶기보다 handle + dense storage로 관리하면 relocation-safe architecture를 만들기 쉽다. visible entity list, material binning, indirect draw/dispatch generation과도 자연스럽게 연결된다.

CUDA/GPU compute에서는 active simulation set을 sparse-to-dense 구조로 유지하면 kernel은 dense range를 처리하면서 logical identity를 별도 mapping으로 보존할 수 있다. compaction 이후에도 simulation 결과와 rendering state를 stable handle로 연결할 수 있다.

Rendering + simulation interop에서는 simulation이 `N+1` version을 쓰고 renderer가 `N` version을 읽는 구조가 compute/graphics overlap의 기본 형태다. external memory로 copy를 줄여도 ownership과 synchronization은 별도로 해결해야 한다.

Sparse volume, active voxel, particle, mesh chunk처럼 생성·삭제·compaction이 빈번한 데이터도 logical ID와 physical slot을 분리하면 visibility나 LOD 기반 재배치와 stable reference를 동시에 만족시키기 쉽다.

## 6. 머릿속에 남길 질문 3개
1. 왜 GPU device address나 dense array index를 scene object의 영구 identity로 사용하면 compaction과 streaming이 어려워지는가?
2. Generational Handle의 generation은 multi-frame GPU workload에서 어떤 stale-reference 문제를 막아 주는가?
3. Triple buffering과 multi-version scene state의 차이를 queue ownership과 synchronization 관점에서 설명할 수 있는가?

## 7. graphics engineer 면접 질문 1개와 답변
### 질문
대규모 GPU scene에서 object 제거 후 dense array를 compact해야 하지만 이전 frame의 GPU command와 simulation queue가 그 object를 참조할 수 있다. locality와 lifetime correctness를 동시에 유지하려면 어떤 구조가 적절한가?

### 답변
Logical identity와 physical location을 분리한다. 외부 참조는 `[index, generation]` 형태의 Generational Handle을 사용하고, Handle Table이 현재 DenseSlot을 가리키게 한다. Dense storage는 SoA와 sparse-set mapping으로 compact/reorder할 수 있으며 relocation 후 mapping만 갱신한다.

이전 frame의 stale handle은 generation 검증으로 recycled object를 잘못 참조하지 않게 한다. 동시에 scene data를 frame/queue별 version으로 분리해 graphics가 읽는 snapshot을 next simulation update가 overwrite하지 않도록 하고, version handoff는 fence/semaphore와 resource barrier로 명시한다. 핵심은 **identity, placement, temporal ownership을 독립 contract로 분리하는 것**이다.

## 8. 포트폴리오 / 커리어 연결
이 주제는 단순 ECS 지식보다 GPU systems architecture를 설명하기 좋다. 포트폴리오에서는 다음 연결이 중요하다.

- Sparse Set: O(1) membership + dense traversal
- Generational Handle: stale reference와 slot recycling 분리
- SoA: GPU field locality와 coalesced access
- Handle Table: relocation/compaction-safe indirection
- Multi-buffer Scene State: CPU/compute/graphics의 temporal ownership 분리
- Fence/Semaphore: version handoff와 memory visibility 보장
- External Memory: copy는 줄여도 logical lifetime과 synchronization 문제는 남음

면접에서는 자료구조 하나보다 `scene mutation -> GPU compaction -> queue overlap -> render-graph synchronization -> resource lifetime`을 하나의 시스템 문제로 설명할 수 있다는 점이 더 중요하다.

## 9. 내일 이어서 볼 개념
**Cross-API GPU Ownership Transfer: Timeline Semaphores, External Memory, and Zero-Copy CUDA/Vulkan Interop**

오늘 identity·storage·version을 분리했으므로 내일은 simulation과 rendering이 실제로 같은 GPU memory를 공유할 때 `who owns this version now?`를 timeline semaphore, CUDA external semaphore, external memory lifetime 관점에서 이어서 본다.

## 10. 참고 키워드
Sparse Set, Dense Set, Generational Handle, Entity Version, Stable Handle, Handle Table, Indirection, Dense Slot, SoA, AoSoA, Scene Version, Frame Resources, Multi-Buffering, GPU Residency, Physical Residency, Buffer Device Address, Dirty Bitset, Compaction, Relocation, Resource Epoch, Fence, Timeline Semaphore, Queue Ownership, External Memory, CUDA-Vulkan Interop, Render Graph, EnTT, D3D12 MakeResident/Evict, Vulkan Synchronization.

참고 자료: EnTT Documentation, Vulkan 1.4 Specification, Microsoft Direct3D 12 Fence-Based Resource Management/Residency documentation, NVIDIA CUDA Programming Guide graphics interoperability section.
