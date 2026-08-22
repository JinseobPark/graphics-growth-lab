---
title: "Cross-API GPU Ownership Transfer: Timeline Semaphores, External Memory, and Zero-Copy CUDA/Vulkan Interop"
date: "2026-08-22"
category: Graphics
tags: [GPU, CUDA, Vulkan, External Memory, Timeline Semaphore, Synchronization, Zero-Copy, Interop, Render Graph, C++]
level: intermediate
---

# [Daily Graphics Growth] 2026-08-22 - Cross-API GPU Ownership Transfer: Timeline Semaphores, External Memory, and Zero-Copy CUDA/Vulkan Interop

## 1. 오늘의 개념
어제는 GPU scene state에서 **identity, physical placement, temporal ownership**을 분리했다. 오늘은 그 state가 Vulkan rendering과 CUDA simulation 사이에서 실제로 같은 GPU allocation을 공유할 때 필요한 **Cross-API GPU Ownership Transfer**를 본다.

핵심은 **External Memory**와 **External Synchronization**을 별개의 contract로 보는 것이다. External Memory는 두 API가 같은 underlying allocation을 바라보게 한다. External Semaphore는 그 메모리를 어느 시점에 누가 읽고 쓸 수 있는지 실행 순서를 정의한다. 따라서 zero-copy interop은 "copy가 없다"는 뜻이지 "동기화가 없다"는 뜻이 아니다.

Vulkan-CUDA interop의 개념적 경로는 다음처럼 볼 수 있다.

`VkDeviceMemory -> OS external handle -> cudaExternalMemory_t -> CUDA mapping`

동기화 경로는 별도로 유지된다.

`VkSemaphore(timeline) -> OS external handle -> cudaExternalSemaphore_t -> wait/signal value`

이 두 경로가 함께 맞아야 동일한 GPU memory를 안전하게 공유할 수 있다.

## 2. 한 줄 핵심
> Zero-copy CUDA/Vulkan interop의 본질은 메모리를 공유하는 것이 아니라, **shared allocation의 lifetime·layout·ownership·visibility를 timeline 값으로 명시하는 것**이다.

## 3. 왜 중요한가
Simulation과 rendering을 같은 GPU에서 수행할 때 가장 비싼 경로 중 하나는 `CUDA -> host/staging -> Vulkan` 같은 불필요한 data movement다. External Memory를 사용하면 같은 allocation을 CUDA kernel과 Vulkan shader/vertex input/storage buffer가 직접 공유할 수 있어 대규모 particle, mesh, voxel, CFD field 같은 데이터를 매 frame 복사하지 않아도 된다.

하지만 single shared buffer를 만들었다고 pipeline이 자동으로 빨라지지는 않는다. CUDA가 쓰는 동안 Vulkan이 읽으면 data race이고, Vulkan이 이전 frame을 읽는 동안 CUDA가 같은 region을 overwrite하면 temporal ownership이 깨진다. Zero-copy는 bandwidth 문제를 줄이지만 **hazard, visibility, queue/API ordering** 문제는 더 명시적으로 만든다.

또 하나의 중요한 점은 **zero-copy와 overlap은 같은 개념이 아니라는 것**이다. 하나의 buffer를 `Vulkan read -> CUDA write -> Vulkan read` 순서로 엄격히 handoff하면 copy는 없지만 두 workload가 직렬화될 수 있다. 실제 overlap이 필요하면 여러 version의 external allocation 또는 region을 사용해 `Render[N]`과 `Simulate[N+1]`이 동시에 진행되는 구조가 필요하다.

## 4. 구현 관점
### 4.1 External Memory는 shared address contract다
Vulkan에서 export 가능한 `VkDeviceMemory`를 만들고 platform handle을 통해 CUDA에 import하면 두 API는 동일한 underlying allocation을 공유할 수 있다. CUDA에서는 `cudaImportExternalMemory()`로 external object를 만들고, buffer는 `cudaExternalMemoryGetMappedBuffer()`, image 계열은 mipmapped-array mapping을 통해 접근한다.

중요한 점은 **CUDA pointer 자체가 interop identity가 아니라는 것**이다. 장기 identity는 Vulkan allocation/resource와 export/import lifetime에 있고, CUDA pointer는 그 external memory에 대한 특정 mapping이다. 따라서 renderer architecture에서 persistent logical resource ID와 API별 view/mapping을 분리해서 보는 편이 안전하다.

### 4.2 같은 physical GPU인지 확인해야 한다
External object는 Vulkan에서 생성한 physical device와 대응하는 CUDA device에서 import되어야 한다. NVIDIA의 Vulkan-CUDA interop 가이드는 Vulkan `deviceUUID`와 CUDA device UUID를 비교해 같은 GPU를 선택하는 흐름을 사용한다.

멀티-GPU 시스템에서 단순히 `CUDA device 0 == Vulkan adapter 0`으로 가정하면 잘못된 GPU에 import를 시도할 수 있다. 따라서 **device identity도 interop ABI의 일부**다.

### 4.3 Timeline Semaphore는 ownership epoch로 볼 수 있다
Binary semaphore는 한 번의 signal/wait 관계를 표현하기 쉽지만, 지속적인 simulation-rendering handoff에는 많은 객체와 상태 관리가 필요하다. Timeline Semaphore는 단조 증가하는 64-bit counter로 진행 상태를 표현한다.

예를 들어 하나의 shared buffer에 대해 개념적으로 다음 epoch를 생각할 수 있다.

- `2N + 1`: Vulkan이 frame N에서 읽기를 끝냈다. CUDA가 다음 write를 시작해도 된다.
- `2N + 2`: CUDA가 simulation/write를 끝냈다. Vulkan이 새로운 결과를 읽어도 된다.

CUDA에서는 imported timeline semaphore의 wait/signal parameter에 counter value를 넣어 `cudaWaitExternalSemaphoresAsync()`와 `cudaSignalExternalSemaphoresAsync()`를 stream에 연결할 수 있다. 이 구조는 CPU polling보다 GPU-side dependency를 유지하기 좋다.

### 4.4 Semaphore ordering과 resource ownership을 혼동하지 않는다
Timeline semaphore는 "언제 다음 work가 시작할 수 있는가"라는 execution ordering을 표현한다. Vulkan resource가 `VK_SHARING_MODE_EXCLUSIVE`이고 external queue family semantics가 적용되는 경우에는 `VK_QUEUE_FAMILY_EXTERNAL` 같은 special queue family와 release/acquire barrier가 resource ownership contract의 일부가 될 수 있다.

즉 다음은 다른 층이다.

- **Execution dependency**: signal/wait가 producer와 consumer의 순서를 연결한다.
- **Memory/resource ownership**: Vulkan barrier의 stage/access scope와 queue-family ownership transition이 resource의 사용 권한과 visibility를 표현한다.
- **Data layout contract**: buffer offset/size, image format/layout, alignment가 두 API에서 동일한 storage를 같은 의미로 해석하게 한다.

Interop bug는 이 셋 중 하나만 맞고 나머지가 틀릴 때 자주 발생한다.

### 4.5 Dedicated allocation과 external handle semantics
External memory handle type에 따라 dedicated allocation이 필요하거나 유리할 수 있다. CUDA가 Vulkan dedicated allocation을 import할 때는 `cudaExternalMemoryDedicated` 의미를 맞춰야 한다. 반대로 suballocation을 공유한다면 offset, size, alignment를 두 API가 정확히 같은 기준으로 해석해야 한다.

OS handle lifetime도 platform마다 다르다. NVIDIA 문서 기준으로 Vulkan의 opaque FD를 CUDA에 성공적으로 import한 Linux 경로에서는 CUDA가 FD ownership을 인수한다. Windows NT handle 경로에서는 CUDA가 handle ownership을 인수하지 않으므로 application-side lifetime 관리가 남는다. 이 차이는 RAII wrapper 설계와 destructor ordering에 직접 영향을 준다.

### 4.6 Memory layout은 cross-API ABI다
Shared buffer 내부에 C++ pointer나 API 전용 object를 저장하는 구조는 interop에 부적합하다. GPU 양쪽에서 공통으로 해석 가능한 **POD-style layout, explicit offset, fixed-width type, alignment contract**가 중요하다.

예를 들어 simulation/rendering 공유 state는 논리적으로 다음처럼 나눌 수 있다.

- header / version / count
- position or vertex SoA
- velocity / scalar field SoA
- material or region ID
- indirect draw/dispatch arguments
- optional dirty range / active index list

CUDA는 compute-friendly SoA로 갱신하고 Vulkan은 storage buffer, vertex buffer, indirect argument buffer 등으로 같은 allocation의 region을 읽을 수 있다. 이때 가장 중요한 것은 "같은 bytes"가 아니라 **같은 semantic layout**이다.

### 4.7 Zero-copy와 multi-buffering
한 개 allocation의 ownership을 매 frame 왕복시키면 synchronization chain이 길어질 수 있다. 어제 다룬 multi-version scene state와 결합하면 다음 구조가 가능하다.

- `Shared[N]`: Vulkan graphics가 읽는 version
- `Shared[N+1]`: CUDA simulation이 쓰는 version
- timeline value: 각 version의 producer completion을 나타내는 epoch

이 구조에서는 allocation copy 없이도 graphics와 simulation이 다른 version에서 overlap할 수 있다. 메모리 사용량은 증가하지만 pipeline serialization을 줄일 수 있으므로 **bandwidth vs residency vs latency** trade-off로 봐야 한다.

### 4.8 C++ / Render Graph contract
C++ engine에서는 interop resource를 단순 `void*`로 노출하기보다 역할을 분리한 contract가 유리하다.

- `SharedGpuAllocation`: Vulkan allocation, exportability, size/alignment, external-memory lifetime
- `CudaExternalView`: imported external memory와 mapped pointer/array lifetime
- `InteropTimeline`: Vulkan semaphore와 CUDA external semaphore가 공유하는 timeline state
- `InteropEpoch`: 어느 producer가 어느 value까지 완료했는지 나타내는 logical version
- `RenderGraphExternalEdge`: Vulkan pass와 foreign CUDA work 사이의 release/wait/acquire dependency

이 구조의 목적은 wrapper를 많이 만드는 것이 아니라 **memory ownership과 synchronization ownership을 타입 수준에서 분리하는 것**이다.

## 5. 내 관심 분야와 연결
CUDA 기반 simulation 결과를 Vulkan/OpenGL/WebGPU 계열 renderer로 시각화하는 작업에서는 host readback을 제거하는 것이 큰 성능 이점이 될 수 있다. 특히 particle, dense/sparse voxel, marching-cubes vertex/index output, scalar field, CFD volume처럼 매 frame 수십~수백 MB가 움직일 수 있는 데이터는 external memory의 가치가 크다.

Marching Cubes 관점에서는 CUDA가 active cell scan과 vertex/index generation을 수행하고 Vulkan이 같은 shared buffer를 vertex/index/indirect input으로 읽는 구조를 생각할 수 있다. 이때 핵심은 mesh extraction 알고리즘보다 `CUDA write completion -> graphics visibility -> draw completion -> next CUDA overwrite`의 ownership cycle이다.

Sparse volume/NanoVDB 계열 pipeline에서도 external buffer를 사용하면 simulation/compute 결과를 rendering path가 별도 copy 없이 소비할 수 있다. 다만 sparse metadata, node pool, active list가 각각 다른 update cadence를 가지므로 하나의 거대한 semaphore dependency보다 resource/version 단위 epoch가 더 잘 맞을 수 있다.

C++ graphics engine 관점에서는 이 주제가 Render Graph의 경계를 API 내부 pass에서 **foreign GPU work**까지 확장하게 만든다. 좋은 engine architecture는 CUDA를 "render graph 밖의 black box"로 두지 않고, 외부 producer/consumer edge와 timeline value를 dependency graph의 일부로 표현한다.

## 6. 머릿속에 남길 질문 3개
1. External Memory로 CUDA와 Vulkan이 같은 allocation을 공유해도 왜 semaphore와 ownership/visibility contract가 여전히 필요한가?
2. 하나의 shared buffer를 timeline semaphore로 완벽하게 동기화했는데도 CUDA와 Vulkan이 overlap하지 못할 수 있는 이유는 무엇인가?
3. Cross-API shared buffer에서 raw CUDA pointer 대신 logical resource ID + API별 mapping을 분리하면 relocation과 lifetime 관리가 왜 쉬워지는가?

## 7. graphics engineer 면접 질문 1개와 답변
### 질문
CUDA simulation이 Vulkan에서 렌더링할 vertex buffer를 직접 갱신하도록 zero-copy interop을 설계하려 한다. "같은 GPU memory를 공유한다"는 것만으로 충분하지 않은 이유와 안전한 ownership handoff 구조를 설명해 보라.

### 답변
같은 allocation을 공유하는 것은 address/storage 문제만 해결한다. CUDA write와 Vulkan read가 동시에 발생하지 않도록 execution dependency가 필요하고, Vulkan 측에서는 resource의 memory visibility와 필요한 경우 external queue-family ownership transition까지 올바르게 표현해야 한다. 또한 두 API가 같은 offset, size, format, alignment를 해석하도록 layout contract가 일치해야 한다.

일반적으로 Vulkan에서 export 가능한 memory와 semaphore를 만들고 CUDA가 이를 import한다. Timeline Semaphore를 사용하면 frame/version별 handoff를 64-bit epoch로 표현할 수 있다. 예를 들어 Vulkan이 현재 version 사용을 끝낸 value를 signal하면 CUDA가 그 value를 wait하고 simulation을 수행한 뒤 다음 value를 signal한다. Vulkan은 그 value를 wait한 뒤 결과를 소비한다.

성능 관점에서는 single-buffer handoff가 workload를 직렬화할 수 있으므로 zero-copy 자체가 overlap을 보장하지 않는다. 필요하다면 여러 shared version을 두어 `Render[N]`과 `Simulate[N+1]`을 겹치게 하고, 각 version의 timeline/lifetime을 독립적으로 추적한다. 핵심은 **shared memory, synchronization, ownership, layout, versioning을 별도 contract로 관리하는 것**이다.

## 8. 포트폴리오 / 커리어 연결
이 주제는 "CUDA와 Vulkan을 둘 다 사용해 봤다"보다 훨씬 강한 systems-level 이야기를 만들 수 있다. 포트폴리오에서는 다음 연결이 중요하다.

- External Memory: host/staging copy 제거
- Device UUID matching: multi-GPU correctness
- Timeline Semaphore: cross-API progress를 monotonic epoch로 표현
- External Queue Ownership: Vulkan resource의 release/acquire semantics
- Explicit Layout: shared buffer를 cross-API ABI로 설계
- Multi-Version Buffering: zero-copy와 pipeline overlap의 분리
- Render Graph: CUDA work를 foreign producer/consumer edge로 모델링
- RAII Lifetime: OS handle, Vulkan object, CUDA external object의 destruction order 관리

면접에서는 API 이름을 나열하는 것보다 **왜 zero-copy가 synchronization-free가 아닌지**, **왜 single shared buffer가 오히려 serialization을 만들 수 있는지**, **왜 pointer가 identity가 아닌지**를 설명하는 것이 중요하다. 이 세 질문은 graphics/compute engine에서 data movement와 ownership을 시스템적으로 이해하는지를 잘 보여준다.

## 9. 내일 이어서 볼 개념
**Cross-API Image Interop and Layout Contracts: Vulkan Images, CUDA Mipmapped Arrays, and Surface Access**

오늘은 buffer 중심의 external memory와 timeline ownership을 봤다. 내일은 image로 확장해 linear buffer보다 복잡한 **format, extent, mip level, tiling, image layout, color-attachment semantics**가 CUDA mipmapped array/surface access와 어떻게 연결되는지 본다.

## 10. 참고 키워드
External Memory, External Semaphore, Timeline Semaphore, Vulkan-CUDA Interop, Zero-Copy, `cudaImportExternalMemory`, `cudaExternalMemoryGetMappedBuffer`, `cudaImportExternalSemaphore`, `cudaWaitExternalSemaphoresAsync`, `cudaSignalExternalSemaphoresAsync`, `VkExportMemoryAllocateInfo`, `VkExportSemaphoreCreateInfo`, `VK_QUEUE_FAMILY_EXTERNAL`, Queue Family Ownership Transfer, Memory Visibility, Device UUID, Dedicated Allocation, Opaque FD, Win32 NT Handle, SoA, POD ABI, Render Graph, Multi-Buffering, GPU Residency, Foreign Queue, Resource Epoch.

참고 자료:
- NVIDIA CUDA Programming Guide — CUDA Interoperability with APIs: https://docs.nvidia.com/cuda/cuda-programming-guide/04-special-topics/graphics-interop.html
- Vulkan Documentation Project — External Memory and Synchronization: https://docs.vulkan.org/guide/latest/extensions/external.html
- Vulkan Specification — Synchronization and Cache Control: https://docs.vulkan.org/spec/latest/chapters/synchronization.html
- Vulkan Documentation Project — Timeline Semaphores: https://docs.vulkan.org/tutorial/latest/Advanced_Vulkan_Compute/08_Asynchronous_Compute/03_timeline_semaphores.html
