---
title: "Cross-API Image Interop and Layout Contracts: Vulkan Images, CUDA Mipmapped Arrays, and Surface Access"
date: "2026-08-23"
category: Graphics
tags: [GPU, CUDA, Vulkan, Image Interop, External Memory, Mipmapped Array, Surface Object, Image Layout, Memory Layout, C++]
level: intermediate
---

# [Daily Graphics Growth] 2026-08-23 - Cross-API Image Interop and Layout Contracts: Vulkan Images, CUDA Mipmapped Arrays, and Surface Access

## 1. 오늘의 개념
어제는 CUDA와 Vulkan이 **같은 GPU allocation을 공유할 때의 ownership transfer**를 External Memory와 Timeline Semaphore 관점에서 봤다. 오늘은 그 공유 대상이 단순한 `VkBuffer`가 아니라 **`VkImage`**일 때 무엇이 달라지는지 본다.

Buffer는 비교적 단순하다. `offset + size`를 맞추면 CUDA에서는 device pointer로 접근할 수 있다. 반면 Image는 **format, extent, mip level, array layer, tiling, aspect, usage**를 함께 가진다. 특히 `VK_IMAGE_TILING_OPTIMAL`의 physical texel layout은 구현 의존적이므로, image를 단순한 `float*`나 `uint8_t*`로 해석하는 접근은 성립하지 않는다.

그래서 Vulkan image를 CUDA에서 공유할 때는 보통 다음 추상화가 연결된다.

`VkImage + VkDeviceMemory -> external memory handle -> cudaExternalMemory_t -> cudaMipmappedArray_t -> cudaArray_t -> cudaSurfaceObject_t / cudaTextureObject_t`

여기서 중요한 것은 **같은 bytes를 공유한다는 사실보다, 두 API가 같은 image semantics를 유지하는가**다.

이 문제는 크게 네 개의 contract로 나눌 수 있다.

1. **Storage Contract** — 어느 `VkDeviceMemory`의 어느 offset이 image를 backing하는가
2. **Shape/Format Contract** — width, height, depth, mip count, layer count, channel format을 두 API가 같은 의미로 보는가
3. **Access Contract** — CUDA에서 surface/texture로 어떤 방식으로 읽고 쓰는가
4. **Ownership/Synchronization Contract** — 어느 시점에 Vulkan과 CUDA 중 누가 image contents를 사용할 수 있는가

즉 **Image Interop은 pointer sharing이 아니라 image metadata와 access semantics의 cross-API ABI를 만드는 문제**다.

## 2. 한 줄 핵심
> Vulkan image를 CUDA와 zero-copy로 공유할 때 핵심은 raw memory address가 아니라, **opaque tiled image를 `cudaMipmappedArray`로 동일하게 재해석하고 format·mip·layer·surface access·ownership을 하나의 ABI로 맞추는 것**이다.

## 3. 왜 중요한가
Simulation/visualization pipeline에서는 buffer보다 image가 더 자연스러운 경우가 많다. 예를 들어 CUDA가 계산한 scalar field를 Vulkan이 바로 heatmap texture로 sampling하거나, volume density를 3D texture로 사용하거나, CUDA post-process 결과를 Vulkan composition pass가 그대로 읽는 구조다.

이때 중간에 `CUDA buffer -> staging -> Vulkan image` copy가 들어가면 data movement와 synchronization 비용이 커진다. 특히 2D/3D field가 매 frame 갱신되는 경우에는 계산 자체보다 copy가 pipeline latency를 지배할 수 있다. External Image Interop은 이 copy를 제거하면서 CUDA compute와 graphics sampling 사이의 경계를 직접 연결한다.

하지만 image interop은 buffer interop보다 correctness 조건이 많다. 가장 흔한 오해는 **VkImage의 backing memory가 있으니 CUDA pointer arithmetic으로 texel을 찾을 수 있다고 생각하는 것**이다. Vulkan의 `VK_IMAGE_TILING_OPTIMAL`은 texel 배치가 implementation-dependent다. GPU가 texture cache와 block layout에 유리하도록 내부적으로 배치하기 때문에 application이 row pitch를 가정할 수 없다. CUDA 쪽에서도 이런 image storage는 linear pointer가 아니라 **CUDA Array / Mipmapped Array** 추상화로 접근하는 것이 맞다.

또 하나는 **format bit pattern과 numerical semantics가 같다는 보장이 없다는 점**이다. 예를 들어 8-bit four-channel image라도 Vulkan에서 UNORM color인지, integer label인지, sRGB encoded color인지에 따라 같은 32bit texel의 의미가 달라진다. Surface write는 texture sampling과 달리 filtering이나 normalized conversion을 자동으로 해결해 주는 개념이 아니다. 따라서 interop format은 단순히 `32 bits per pixel`이 아니라 **data meaning까지 포함하는 ABI**여야 한다.

## 4. 구현 관점
### 4.1 `VkImage`는 linear buffer가 아니다
Vulkan에서 `VK_IMAGE_TILING_OPTIMAL`은 texel이 implementation-dependent arrangement로 저장됨을 뜻한다. 반대로 `VK_IMAGE_TILING_LINEAR`는 row-major 형태이지만 padding과 pitch가 존재할 수 있고, 일반적인 high-performance rendering에서는 optimal tiling이 더 적합하다.

중요한 구분은 다음과 같다.

- **Physical tiling**: 실제 VRAM에서 texel이 어떤 방식으로 배치되는가
- **Vulkan image layout**: `GENERAL`, `SHADER_READ_ONLY_OPTIMAL` 같은 resource usage/state 의미
- **Format layout**: 한 texel 안에서 R/G/B/A channel이 어떤 bit width와 numerical type을 가지는가

이 세 가지를 모두 "layout"이라고 부르기 때문에 혼동이 많다. `VK_IMAGE_LAYOUT_GENERAL`이라고 해서 image가 linear memory가 되는 것은 아니다. 반대로 optimal tiling image라고 해서 CUDA가 접근할 수 없는 것도 아니다. CUDA는 raw pointer 대신 CUDA Array abstraction을 통해 opaque image layout을 다룬다.

### 4.2 Vulkan image creation 단계부터 interop 가능성을 결정한다
External image는 나중에 임의의 `VkImage`를 export하는 문제가 아니다. Image creation 시점부터 **external memory handle type과 image format/usage가 해당 physical device에서 지원되는지**가 중요하다.

Vulkan에서는 external-memory image capability와 memory requirement를 확인해야 하며, 일부 image/handle 조합은 dedicated allocation을 요구하거나 선호할 수 있다. 이 경우 `VkImage`와 `VkDeviceMemory`의 관계는 사실상 1:1에 가까워지고 CUDA import에서도 dedicated memory 의미를 일치시켜야 한다.

즉 renderer의 resource descriptor에는 최소한 다음 정보가 interop ABI의 일부로 남아야 한다.

- `VkFormat`
- `VkExtent3D`
- mip level count
- array layer count
- `VkImageUsageFlags`
- tiling
- external handle type
- dedicated allocation 여부
- CUDA channel description과 array flags

이 metadata를 resource 생성 시점에 확정해 두면 이후 CUDA mapping과 Vulkan view 생성이 같은 source of truth를 사용할 수 있다.

### 4.3 CUDA에서는 `cudaMipmappedArray_t`로 image를 다시 본다
Vulkan memory를 `cudaImportExternalMemory()`로 import한 뒤 image 영역은 `cudaExternalMemoryGetMappedMipmappedArray()`를 사용해 CUDA mipmapped array로 mapping할 수 있다.

현재 CUDA Runtime API의 `cudaExternalMemoryMipmappedArrayDesc`는 핵심적으로 다음 정보를 가진다.

- `offset`
- `formatDesc`
- `extent`
- `flags`
- `numLevels`

이 값들은 Vulkan image를 만들 때 사용한 image semantics와 맞아야 한다. 특히 **offset, dimensions, format, mip count**가 어긋나면 같은 allocation을 바라보더라도 두 API가 서로 다른 image를 해석하게 된다.

`cudaMipmappedArray_t`는 mip chain 전체를 나타내고, 특정 mip level은 `cudaGetMipmappedArrayLevel()`을 통해 `cudaArray_t`로 얻는다. 이후 해당 level을 CUDA surface 또는 texture object의 resource로 사용할 수 있다.

이 구조가 중요한 이유는 **mip level을 pointer offset으로 직접 계산하지 않아도 된다는 것**이다. 내부 tiling과 per-level placement는 driver/runtime가 관리하고 application은 image-level abstraction을 유지한다.

### 4.4 Surface Access는 write 가능한 image view다
CUDA Array를 kernel에서 쓰려면 surface access가 핵심 도구다. Mapping descriptor의 array flags는 CUDA array의 사용 가능 기능을 나타낸다. Surface load/store가 필요하다면 `cudaArraySurfaceLoadStore` 의미를 맞춰야 하며, graphics API에서 color target으로 사용되는 mipmapped array라면 CUDA external-memory mapping 시 `cudaArrayColorAttachment`도 요구된다.

개념적으로 access chain은 다음과 같다.

`cudaMipmappedArray_t -> mip level cudaArray_t -> cudaResourceDesc -> cudaSurfaceObject_t`

Surface Object는 pointer보다 image access semantics에 가깝다. CUDA가 underlying row pitch와 tiled representation을 직접 노출하지 않고 array abstraction을 통해 read/write를 수행한다.

여기서 자주 놓치는 세부사항이 하나 있다. **CUDA surface의 x coordinate는 element index가 아니라 byte offset**이다. 예를 들어 32-bit float 2D surface에서 논리적 `(x, y)` texel에 접근할 때 surface API의 x 좌표는 개념적으로 `x * sizeof(float)`가 된다. 반면 y 좌표에 필요한 underlying line pitch는 CUDA array가 내부적으로 처리한다.

이 차이는 graphics engineer 면접에서도 좋은 함정 질문이다. Texture sampling coordinate와 Surface byte addressing을 같은 방식으로 생각하면 정상적으로 mapping된 image에서도 잘못된 texel을 읽을 수 있다.

### 4.5 Format translation은 bit width가 아니라 semantic translation이다
Vulkan `VkFormat`과 CUDA `cudaChannelFormatDesc`는 1:1 enum mapping이 아니다. Application이 두 API의 format을 공통 semantic으로 연결해야 한다.

예를 들어 개념적으로 다음은 비교적 직접적이다.

- `VK_FORMAT_R32_SFLOAT` -> 32-bit float single channel
- `VK_FORMAT_R16G16_SFLOAT` -> 16-bit float two channels
- `VK_FORMAT_R16G16B16A16_SFLOAT` -> 16-bit float four channels
- `VK_FORMAT_R32G32B32A32_SFLOAT` -> 32-bit float four channels

하지만 UNORM/SNORM/sRGB 계열은 더 조심해야 한다. 같은 8-bit unsigned channel이라도 Vulkan texture sampling은 normalized numerical semantics를 가질 수 있고, sRGB format은 color-space conversion이 개입할 수 있다. CUDA surface read/write는 "Vulkan shader에서 이 format을 읽을 때와 동일한 fixed-function conversion"을 자동으로 보장하는 개념이 아니다.

따라서 interop resource는 format 이름보다 **semantic type**을 먼저 정의하는 편이 안전하다.

- linear floating-point radiance
- normalized mask
- integer material/region ID
- encoded display color
- scalar simulation field

그 뒤 Vulkan format과 CUDA channel format을 그 semantic에 맞춰 선택해야 한다.

### 4.6 Mip, layer, 3D depth는 서로 다른 차원이다
Image interop에서는 `extent.depth`, array layer count, mip level을 혼동하기 쉽다.

CUDA Array 관점에서 3D volume과 2D layered image는 둘 다 3개의 extent 값을 사용할 수 있지만 의미가 다르다. Layered array에서는 depth-like 값이 layer count를 나타내고 `cudaArrayLayered` 계열 flag가 의미를 결정한다. 진짜 3D array에서는 그 값이 volume의 z dimension이다.

Vulkan에서도 `VK_IMAGE_TYPE_3D`와 `VK_IMAGE_TYPE_2D + arrayLayers > 1`은 sampling semantics와 view semantics가 다르다. 따라서 cross-API descriptor는 단순 `width/height/depth` 구조 하나만 들고 있기보다 **image dimensionality와 layering semantic을 별도로 보존**하는 것이 좋다.

Mip chain도 마찬가지다. `numLevels`는 mapping 전체 contract에 들어가고, CUDA는 각 level을 별도 `cudaArray_t`로 다룰 수 있다. Vulkan은 `VkImageView`의 subresource range로 mip/layer subset을 바라볼 수 있다. 두 API가 다른 view를 만들 수 있다는 사실과 **underlying shared resource의 ownership이 분리된다는 것**은 같은 이야기가 아니다. Subresource 단위 concurrency가 필요하다면 external-memory ownership과 synchronization 규칙이 실제로 그 granularity를 허용하는지 별도로 검증해야 한다.

### 4.7 Image layout transition과 external ownership은 별도 문제다
Vulkan의 image layout transition은 resource가 다음 사용에 적합한 internal state를 갖도록 하는 Vulkan-side contract다. External memory ownership transfer는 resource를 Vulkan instance 밖의 API가 사용하도록 handoff하는 cross-API contract다.

따라서 다음 두 질문을 분리해야 한다.

1. Vulkan 안에서 이 image는 어떤 image layout/state에 있어야 하는가?
2. CUDA가 사용할 수 있도록 external ownership과 completion을 어떻게 넘길 것인가?

Semaphore signal/wait는 producer completion과 consumer start의 execution dependency를 만든다. Vulkan의 image barrier는 layout, stage/access scope, queue-family ownership을 표현한다. CUDA 쪽에서는 imported external semaphore를 stream의 wait/signal로 연결한다.

핵심은 **"semaphore가 있으니 image layout 문제도 해결됐다"고 생각하지 않는 것**이다. 반대로 image layout transition만 했다고 CUDA가 안전하게 접근할 수 있는 것도 아니다. 어제 본 External Synchronization과 오늘의 Image Layout Contract가 함께 맞아야 한다.

### 4.8 Image vs Buffer를 언제 선택할 것인가
같은 simulation data라도 반드시 image interop이 정답은 아니다.

**Image/CUDA Array가 유리한 경우**
- 2D/3D spatial field
- texture filtering이 중요한 data
- Vulkan shader에서 sampled/storage image로 자연스럽게 소비하는 data
- regular grid와 mip hierarchy

**Buffer가 유리한 경우**
- variable-length mesh/particle list
- prefix sum, compaction, append/indirect argument 생성
- scatter-heavy data structure
- sparse node/brick metadata
- custom byte layout과 random indexing이 핵심인 data

예를 들어 Marching Cubes의 최종 vertex/index output은 buffer가 자연스럽지만, 그 입력이 되는 dense scalar/SDF field는 3D image/array가 자연스러울 수 있다. 즉 하나의 pipeline에서도 **field representation과 extracted geometry representation의 최적 interop primitive가 다를 수 있다.**

### 4.9 Memory lifetime과 C++ resource contract
C++ engine에서는 `VkImage`, external allocation, CUDA mipmapped array, surface object를 서로 독립적인 handle로 흩어 놓으면 destruction ordering과 stale-view bug가 발생하기 쉽다.

논리적으로는 다음 계층을 하나의 interop resource family로 묶어 생각할 수 있다.

- **Logical Image ID**: engine/render graph가 보는 stable identity
- **Vulkan Image View**: `VkImage`, `VkImageView`, format/layout state
- **External Allocation**: `VkDeviceMemory`, export handle, dedicated/suballocated metadata
- **CUDA Image View**: `cudaExternalMemory_t`, `cudaMipmappedArray_t`
- **Per-Level Access View**: `cudaArray_t`, `cudaSurfaceObject_t`, `cudaTextureObject_t`
- **Ownership Epoch**: timeline semaphore value와 current producer/consumer

이 구조의 핵심은 wrapper 수를 늘리는 것이 아니라 **storage lifetime과 view lifetime을 분리하는 것**이다. Surface object는 특정 CUDA array view에 의존하고, mipmapped array mapping은 imported external memory에 의존하며, external memory는 Vulkan allocation의 lifetime과 연결된다. C++ RAII는 이 dependency graph의 역순으로 안전하게 해제되도록 설계되어야 한다.

## 5. 내 관심 분야와 연결
GPU simulation과 real-time visualization을 연결할 때 이 주제는 매우 직접적이다. CUDA에서 계산한 field를 CPU로 내려보내지 않고 Vulkan이 바로 sampling할 수 있으면 **GPU-stay-GPU pipeline**을 유지할 수 있다.

### 2D scalar/heatmap
CUDA가 pressure, temperature, doping, density 같은 scalar field를 2D image에 기록하고 Vulkan fragment/compute shader가 colormap과 contour를 적용할 수 있다. 이때 storage를 `R16F` 또는 `R32F`처럼 정의하면 simulation semantic과 rendering semantic의 연결이 단순해진다.

### 3D volume
Dense SDF, level-set φ, density, velocity magnitude 같은 regular grid는 3D CUDA Array와 Vulkan 3D image의 conceptual model이 잘 맞는다. CUDA는 volume update를 수행하고 Vulkan은 ray marching 또는 slice sampling으로 visualization할 수 있다. 여기서는 pointer coalescing만 보는 것이 아니라 **3D texture cache와 spatial sampling locality까지 representation 선택의 일부**가 된다.

### Mesh extraction
Marching Cubes를 생각하면 입력 scalar field는 image/array로 유지하면서, active-cell scan과 생성된 vertex/index는 buffer로 분리할 수 있다. 즉 "interop resource 하나로 모든 데이터를 표현"하는 것이 아니라, **access pattern에 맞게 image와 buffer를 역할 분담하는 architecture**가 자연스럽다.

### C++ graphics engine
Render Graph 수준에서는 image interop pass가 단순 `VkImage` resource가 아니라 `ForeignWritableImage` 또는 `ExternalComputeImage`처럼 **foreign API ownership이 존재하는 resource class**가 될 수 있다. Pass dependency는 shader read/write뿐 아니라 `CUDA owns epoch N`, `Vulkan owns epoch N+1` 같은 cross-API state를 포함해야 한다.

이 관점은 Vulkan/CUDA뿐 아니라 D3D12/CUDA, native compute와 graphics interop, future GPU service architecture에서도 그대로 확장된다. API가 달라도 핵심은 **same storage + different views + explicit ownership**이다.

## 6. 머릿속에 남길 질문 3개
1. `VK_IMAGE_TILING_OPTIMAL` image의 backing `VkDeviceMemory`를 CUDA에서 import했는데도 왜 `float*` pointer arithmetic으로 texel을 읽으면 안 되는가?
2. Vulkan `VkFormat`과 CUDA `cudaChannelFormatDesc`의 bit width가 같더라도 UNORM, integer, sRGB 같은 numerical semantics가 달라질 수 있는 이유는 무엇인가?
3. 같은 external image를 Vulkan과 CUDA가 공유할 때 **image layout transition, external semaphore, queue-family/resource ownership**은 각각 어떤 문제를 해결하는가?

## 7. graphics engineer 면접 질문 1개와 답변
### 질문
Vulkan의 `VK_IMAGE_TILING_OPTIMAL` 3D image를 CUDA simulation kernel이 직접 갱신하고, 이후 Vulkan shader가 같은 image를 sampling하도록 zero-copy pipeline을 설계하려 한다. 왜 raw device pointer mapping만으로는 충분하지 않으며, 어떤 resource contract가 필요한가?

### 답변
`VK_IMAGE_TILING_OPTIMAL`은 texel의 physical memory arrangement가 implementation-dependent이므로 application이 linear row/depth pitch를 가정해서 pointer arithmetic을 할 수 없다. 따라서 Vulkan image의 external memory를 CUDA에 import한 뒤, image의 offset·extent·channel format·mip count와 일치하는 `cudaMipmappedArray_t`로 mapping하고 각 mip level을 `cudaArray_t`로 접근하는 방식이 적합하다. CUDA에서 write가 필요하면 surface-load/store capability를 가진 CUDA Array view와 Surface Object를 사용한다.

Correctness 측면에서는 네 contract가 필요하다. 첫째, Vulkan image와 CUDA array가 동일한 storage와 dimensions를 보는 **storage/shape contract**. 둘째, `VkFormat`과 CUDA channel representation이 같은 numerical meaning을 갖는 **format contract**. 셋째, Vulkan image layout과 CUDA surface/texture access가 각각 올바른 상태에 있는 **access contract**. 넷째, Vulkan과 CUDA가 동시에 conflicting access를 하지 않도록 external semaphore와 resource ownership handoff를 사용하는 **synchronization contract**다.

성능 측면에서는 image가 regular spatial field와 filtered sampling에 적합한 반면, variable-length geometry나 append/scan 중심 data는 buffer가 더 적합할 수 있다. 따라서 zero-copy라는 이유만으로 모든 shared resource를 image로 만들기보다 **consumer access pattern에 맞춰 Image와 Buffer를 선택하는 것**이 중요하다.

## 8. 포트폴리오 / 커리어 연결
이 주제는 graphics engineer 포트폴리오에서 단순 API 사용 경험보다 **GPU data architecture를 이해한다는 증거**가 될 수 있다.

강한 설명 포인트는 다음과 같다.

- `VkBuffer`와 `VkImage` interop의 차이를 설명할 수 있다.
- Optimal tiling을 raw pointer layout과 혼동하지 않는다.
- External Memory mapping에서 image metadata를 cross-API ABI로 관리한다.
- `cudaMipmappedArray_t`, per-mip `cudaArray_t`, Surface/Texture Object의 역할을 구분한다.
- Surface byte addressing과 format semantic 차이를 알고 있다.
- Timeline semaphore와 image layout/ownership transition을 별도 문제로 본다.
- 2D/3D field는 image, variable-length output은 buffer처럼 access pattern 기반 representation을 선택한다.
- C++에서 logical resource identity, allocation lifetime, API-specific view lifetime을 분리한다.

면접에서는 "CUDA와 Vulkan을 연결해 봤다"보다 다음과 같은 설명이 훨씬 강하다.

> Buffer interop은 주로 address/size contract지만, Image interop은 format·dimension·mip·layer·tiling abstraction과 access semantics까지 포함한다. 그래서 image를 linear pointer로 취급하지 않고 CUDA Array abstraction으로 유지하며, Vulkan layout state와 cross-API synchronization을 분리해 설계한다.

이 한 문장은 rendering pipeline, GPU memory model, compute interoperability를 동시에 이해하고 있다는 신호가 된다.

## 9. 내일 이어서 볼 개념
**Sparse 3D Volume Residency and Brick Streaming: Vulkan Sparse Images, CUDA Sparse Arrays, and Working-Set Control**

오늘은 dense image가 두 API에서 같은 storage와 semantics를 유지하는 방법을 봤다. 다음 단계에서는 3D field가 너무 커서 전체 volume을 항상 VRAM에 둘 수 없을 때로 확장한다.

핵심 질문은 다음이다.

- Sparse image/array에서 virtual extent와 physically resident tiles는 어떻게 분리되는가?
- 3D volume을 brick 단위로 관리하면 ray marching과 simulation working set을 어떻게 맞출 수 있는가?
- Residency map, brick pool, active-region compaction은 sparse voxel/NanoVDB 관점과 어떤 차이가 있는가?
- CUDA와 Vulkan이 sparse resource를 공유할 때 memory residency와 synchronization은 어떻게 분리해서 생각해야 하는가?

즉 오늘의 **Image Layout Contract**에서 내일은 **Image Residency Contract**로 이동한다.

## 10. 참고 키워드
- `cudaImportExternalMemory`
- `cudaExternalMemoryGetMappedMipmappedArray`
- `cudaExternalMemoryMipmappedArrayDesc`
- `cudaMipmappedArray_t`
- `cudaGetMipmappedArrayLevel`
- `cudaArray_t`
- `cudaArraySurfaceLoadStore`
- `cudaArrayColorAttachment`
- `cudaCreateSurfaceObject`
- `cudaSurfaceObject_t`
- Surface byte addressing
- `VkExternalMemoryImageCreateInfo`
- `VkPhysicalDeviceExternalImageFormatInfo`
- `VkMemoryDedicatedRequirements`
- `VK_IMAGE_TILING_OPTIMAL`
- `VK_IMAGE_TILING_LINEAR`
- `VkImageLayout`
- `VkImageMemoryBarrier2`
- `VK_QUEUE_FAMILY_EXTERNAL`
- mip level / array layer / image aspect
- 3D image vs 2D array image
- format semantic ABI
- image vs buffer access pattern
- zero-copy image interop
- GPU-stay-GPU visualization pipeline

공식 문서 흐름:
- NVIDIA CUDA Programming Guide — CUDA Interoperability with APIs: https://docs.nvidia.com/cuda/cuda-programming-guide/04-special-topics/graphics-interop.html
- NVIDIA CUDA Runtime API — External Resource Interoperability: https://docs.nvidia.com/cuda/cuda-runtime-api/group__CUDART__EXTRES__INTEROP.html
- NVIDIA CUDA Programming Guide — Surface Object API: https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html
- Vulkan Specification — Resource Creation / Image Tiling: https://docs.vulkan.org/spec/latest/chapters/resources.html
- Vulkan Specification — External Resource Sharing / Queue Family Ownership: https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html