---
title: "Sparse 3D Volume Residency and Brick Streaming: Vulkan Sparse Images, CUDA Sparse Arrays, and Working-Set Control"
date: "2026-08-24"
category: Graphics
tags: [GPU, Vulkan, CUDA, Sparse Residency, Sparse Image, Sparse Array, 3D Volume, Brick Streaming, Virtual Memory, Working Set, Memory Layout, C++]
level: intermediate
---

# [Daily Graphics Growth] 2026-08-24 - Sparse 3D Volume Residency and Brick Streaming: Vulkan Sparse Images, CUDA Sparse Arrays, and Working-Set Control

## 1. 오늘의 개념
어제는 Vulkan `VkImage`와 CUDA `cudaMipmappedArray_t`를 같은 physical allocation 위에 놓고, **format·mip·layout·ownership을 cross-API contract로 맞추는 방법**을 봤다. 오늘은 그 전제를 하나 더 제거한다.

> **3D volume 전체가 항상 VRAM에 resident할 필요가 있는가?**

고해상도 scalar field, SDF, level-set, density, temperature, doping field처럼 규칙적인 3D grid는 texture/image access와 잘 맞지만, 해상도가 커지면 dense allocation이 빠르게 비싸진다. 예를 들어 단일 `float` channel만 사용해도 `1024^3` volume은 약 4 GiB이며, material ID·doping·gradient·history까지 함께 들고 있으면 working set은 더 커진다.

Sparse resource의 핵심은 **virtual extent와 physical residency를 분리하는 것**이다.

- 논리적으로는 큰 3D image/array를 유지한다.
- 실제 GPU memory는 현재 필요한 **brick/tile/page**에만 붙인다.
- camera, simulation activity, surface band, LOD에 따라 resident working set을 이동시킨다.

Vulkan에서는 `VK_IMAGE_CREATE_SPARSE_BINDING_BIT`와 `VK_IMAGE_CREATE_SPARSE_RESIDENCY_BIT`, `vkQueueBindSparse()`가 이 모델을 제공한다. CUDA에서는 sparse CUDA array/mipmapped array와 tile pool, `cuMemMapArrayAsync()`가 유사한 역할을 한다.

다만 중요한 점이 있다. **두 API가 모두 sparse resource를 지원한다고 해서 sparse page table 자체가 자동으로 공유되는 것은 아니다.** Application 관점에서는 보통 `BrickKey -> ResidencyEntry -> PhysicalTile`이라는 상위 residency model을 두고, Vulkan sparse binding과 CUDA sparse-array mapping을 각각 backend contract로 다루는 편이 안전하다.

즉 오늘의 주제는 단순한 메모리 절약 기법이 아니라, **volume의 주소 공간(address space), physical backing, streaming priority, synchronization을 분리해 GPU working set을 제어하는 data-oriented architecture**다.

## 2. 한 줄 핵심
> Sparse 3D volume의 본질은 큰 virtual texture를 만드는 것이 아니라, **논리적 brick identity와 physical tile residency를 분리하고 현재 필요한 working set만 GPU memory에 매핑하는 것**이다.

## 3. 왜 중요한가
### 3.1 Dense grid는 해상도가 한 단계만 올라가도 비용이 8배가 된다
3D volume의 메모리는 `O(N^3)`으로 증가한다. 해상도를 축마다 2배 늘리면 voxel 수는 8배가 된다.

예를 들어 다음 필드를 한 voxel에 보관한다고 생각해 보자.

- `phi`: FP32 SDF/level-set — 4 bytes
- material: `uint8` — 1 byte
- doping: FP16 — 2 bytes
- padding/flags — 1 byte

총 8 bytes/voxel이면 `1024^3`에서 약 8 GiB다. 여기에 ping-pong simulation state, gradient, temporary classification, mesh extraction buffer까지 더해지면 dense representation은 quickly VRAM-bound가 된다.

하지만 실제로 의미 있는 데이터가 전체 volume에 균일하게 존재하는 것은 아니다. Level-set surface는 대개 **narrow band** 주변이 중요하고, visualization은 camera frustum과 current LOD에 들어오는 region만 높은 해상도가 필요하다. Semiconductor process geometry도 넓은 XY에 비해 Z 방향은 얇은 layer가 반복될 수 있어 active region이 매우 비균일하다.

Sparse residency는 이 비균일성을 물리 메모리 사용량에 반영한다.

### 3.2 Sparse data structure와 sparse residency는 다른 문제다
NanoVDB, octree, hash grid 같은 자료구조는 **logical sparsity**를 압축한다. 반면 Vulkan Sparse Image나 CUDA Sparse Array는 큰 정규 주소 공간을 유지한 채 **physical memory residency**를 부분적으로 제공한다.

두 접근은 경쟁 관계가 아니다.

- **Sparse data structure**: 어떤 voxel이 존재하는가를 구조적으로 압축
- **Sparse residency**: virtual image/array의 어느 page가 physical memory를 가지는가를 관리

예를 들어 active surface band를 NanoVDB로 계산한 뒤, visualization/texture sampling이 필요한 영역만 sparse 3D image에 brick 단위로 materialize할 수도 있다. 반대로 marching cubes나 texture filtering처럼 regular neighborhood access가 중요한 단계는 sparse image/array가 더 GPU-friendly할 수 있다.

### 3.3 진짜 병목은 capacity보다 working-set movement일 수 있다
Sparse resource가 VRAM capacity 문제를 해결해도 성능이 자동으로 좋아지는 것은 아니다. 매 frame resident brick이 크게 바뀌면 다음 비용이 생긴다.

- page/brick request generation
- physical tile allocation
- sparse bind/map operation
- brick data upload 또는 compute initialization
- mapping 변경과 consumer pass 사이 synchronization
- eviction bookkeeping

즉 sparse volume은 **memory capacity problem을 streaming/scheduling problem으로 바꾼다.**

좋은 설계는 "최소 메모리"보다 **안정적인 resident working set**을 목표로 한다. Camera가 조금 움직일 때마다 brick이 대량 swap된다면 thrashing 때문에 dense texture보다 느릴 수 있다.

## 4. 구현 관점
### 4.1 Virtual Volume과 Physical Tile Pool을 분리한다
Sparse volume을 생각할 때 가장 먼저 분리해야 하는 두 객체가 있다.

**Virtual Volume**
- 전체 logical extent
- format
- mip hierarchy
- logical voxel coordinate space
- brick coordinate space

**Physical Tile Pool**
- 실제 VRAM allocation
- hardware가 요구하는 sparse granularity에 맞춘 tile/page
- free/used state
- last-use epoch
- owner brick

Application-level metadata는 개념적으로 다음과 같이 볼 수 있다.

`BrickKey -> ResidencyEntry -> PhysicalTileHandle`

`BrickKey`는 `(mip, bx, by, bz)`와 같은 stable logical identity다. `PhysicalTileHandle`은 현재 어느 physical allocation/page가 backing하는지 나타낸다. Compaction이나 eviction이 발생해도 logical brick identity는 유지된다.

이 구조는 최근 학습한 **generational handle / logical handle != physical slot** 원칙과 동일하다. 차이는 이번에는 path state가 아니라 **3D virtual address space**에 적용된다는 점이다.

### 4.2 Hardware sparse granularity와 application brick size는 반드시 같을 필요가 없다
Vulkan에서 sparse image block dimension은 `vkGetImageSparseMemoryRequirements()` 계열로 확인한다. CUDA sparse array도 `cuArrayGetSparseProperties()` / `cuMipmappedArrayGetSparseProperties()`를 통해 tile extent와 mip-tail 정보를 얻는다.

따라서 `32^3`, `64^3`처럼 application이 임의로 정한 brick size를 hardware sparse tile과 동일하다고 가정하면 안 된다.

오히려 두 층을 나누는 편이 좋다.

- **Hardware Tile**: API/driver가 요구하는 최소 mapping granularity
- **Logical Brick**: streaming, simulation, compression 관점에서 사용하는 상위 단위

Logical brick 하나가 여러 hardware tile을 묶을 수 있다. 이 방법은 page-table update 수를 줄이고, 서로 다른 API의 sparse granularity 차이도 상위 abstraction에서 흡수하기 쉽다.

### 4.3 Vulkan Sparse Image의 핵심 contract
Vulkan sparse image에서는 일반 image처럼 전체 memory를 한 번에 `vkBindImageMemory()`로 붙이지 않는다.

핵심 개념은 다음과 같다.

- `VK_IMAGE_CREATE_SPARSE_BINDING_BIT`: sparse binding 허용
- `VK_IMAGE_CREATE_SPARSE_RESIDENCY_BIT`: 일부 region만 resident해도 device access 허용
- `sparseResidencyImage3D`: 3D partially resident image 지원 여부
- `VkSparseImageMemoryRequirements`: image aspect별 sparse block/mip-tail 요구사항
- `VkSparseImageMemoryBind`: image offset/extent를 특정 `VkDeviceMemory` 범위에 binding
- `vkQueueBindSparse()`: sparse mapping 변경 자체가 queue operation

특히 `vkQueueBindSparse()`가 queue operation이라는 점이 중요하다. Sparse bind는 단순 CPU-side metadata edit가 아니라 GPU page-table mapping을 변경하는 작업이므로, rendering/compute가 같은 region을 접근하는 시점과 synchronization 관계가 필요하다.

Vulkan specification에서 sparse residency가 활성화된 image는 일부 block이 unbound여도 사용할 수 있다. 다만 `residencyNonResidentStrict`가 없으면 unbound region read 값은 undefined일 수 있다. 따라서 "unmapped voxel은 항상 0이다"라는 shader 가정은 device property를 확인하지 않고 일반화하면 안 된다.

### 4.4 Mip Tail은 일반 brick과 다른 residency 단위다
Sparse mipmapped resource에서는 mip level이 작아지면 한 level이 sparse tile보다 작아지는 시점이 온다. 이 작은 mip들은 개별 brick mapping보다 **mip tail**이라는 별도 연속 영역으로 묶이는 경우가 있다.

Vulkan과 CUDA 모두 sparse mipmapped resource에서 mip-tail 개념을 노출한다.

이것은 streaming architecture에 중요한 의미가 있다.

- 고해상도 mip: spatial brick 단위 residency
- 저해상도 mip tail: coarse fallback 전체를 작은 비용으로 유지 가능

그래서 sparse volume renderer는 "보이지 않는 brick은 완전히 없는 상태"보다 **coarse mip는 항상 resident, fine brick은 demand-resident** 구조를 만들 수 있다. 이렇게 하면 camera 이동이나 sudden zoom에서 missing data가 검은 hole로 나타나는 대신 coarse representation으로 fallback할 수 있다.

### 4.5 CUDA Sparse Array는 texture/surface-friendly sparse address space다
CUDA에서는 sparse flag로 생성한 CUDA Array/Mipmapped Array의 subregion을 physical tile pool에 map/unmap할 수 있다.

Driver API 관점의 주요 요소는 다음과 같다.

- `CUDA_ARRAY3D_SPARSE`
- `CUDA_ARRAY_SPARSE_PROPERTIES`
- `cuArrayGetSparseProperties()` / `cuMipmappedArrayGetSparseProperties()`
- `CU_MEM_CREATE_USAGE_TILE_POOL`
- `CUarrayMapInfo`
- `cuMemMapArrayAsync()`

`cuMemMapArrayAsync()`의 sparse-level mapping은 mip level, layer, XYZ offset과 extent를 지정하며, 해당 offset/extent는 sparse tile dimension에 맞아야 한다. 즉 CUDA에서도 sparse array는 "아무 byte range나 pointer처럼 붙이는 것"이 아니라 **array-coordinate-aligned page mapping**이다.

이 장점은 resident brick이 CUDA texture/surface access model 안에 그대로 남는다는 것이다. Regular 3D neighborhood sampling, trilinear filtering, surface load/store처럼 image-oriented access가 필요한 simulation/visualization stage에 적합하다.

### 4.6 Cross-API에서는 page table보다 Residency Model을 공유한다
어제의 dense image interop에서는 하나의 exportable Vulkan memory allocation을 CUDA가 `cudaExternalMemoryGetMappedMipmappedArray()`로 같은 image semantics로 바라보는 구조를 다뤘다.

Sparse resource에서는 한 단계 더 복잡하다. Vulkan sparse image는 `vkQueueBindSparse()`로 Vulkan resource의 virtual region과 memory range를 연결하고, CUDA sparse array는 `cuMemMapArrayAsync()`로 CUDA array tile과 tile-pool allocation을 연결한다. 두 API의 sparse-resource page table과 tile metadata는 각각의 API contract를 따른다.

그래서 portable architecture의 중심은 다음과 같은 application-level residency table이다.

- `BrickKey`
- residency state: nonresident / requested / uploading / resident / evicting
- physical tile handle
- mip level
- dirty bit
- last-read / last-write epoch
- producer API
- consumer API
- ready synchronization value

즉 **공유해야 하는 것은 "이 brick이 필요하고 어느 데이터 version이 유효한가"라는 의미이며, API-specific sparse mapping command는 backend가 수행한다.**

External memory가 sparse Vulkan image의 backing으로 사용될 수는 있지만, cross-API interop 설계에서 Vulkan sparse bind semantics와 CUDA sparse-array mapping semantics가 자동으로 동일한 tile map을 공유한다고 가정해서는 안 된다. Capability query와 handle/format/memory requirements가 함께 맞는지 확인해야 한다.

### 4.7 Residency는 Boolean이 아니라 State Machine이다
실제 GPU pipeline에서 brick은 단순히 resident/nonresident 두 상태만 가지지 않는다.

개념적으로는 다음 state가 유용하다.

`Unmapped -> Requested -> Allocated -> Populating -> Ready -> Resident -> EvictPending -> Unmapped`

왜 이렇게 세분화할까?

예를 들어 physical tile이 이미 할당됐더라도 CUDA가 아직 field를 계산 중이면 Vulkan shader가 읽어서는 안 된다. 반대로 Vulkan이 마지막으로 sampling한 brick을 fence/timeline completion 전에 physical pool로 반환하면 use-after-free에 가까운 GPU memory hazard가 생긴다.

따라서 ResidencyEntry에는 **memory ownership과 content validity를 분리**해서 기록해야 한다.

- backing page가 존재하는가?
- 최신 data version이 채워졌는가?
- 어느 queue/API가 마지막 write를 했는가?
- consumer가 접근 가능한 synchronization epoch는 무엇인가?

이 관점은 sparse streaming을 단순 allocator가 아니라 **resource state machine**으로 만든다.

### 4.8 Brick Priority는 camera distance 하나로 결정하면 부족하다
좋은 working-set controller는 여러 signal을 합쳐야 한다.

**Rendering 중요도**
- view frustum 안에 있는가
- projected screen size
- ray/volume traversal에서 접근될 가능성
- 현재 mip requirement

**Simulation 중요도**
- narrow-band SDF에 포함되는가
- material interface 근처인가
- field gradient가 큰가
- 이번 step에서 write될 가능성이 있는가

**Temporal 중요도**
- 최근 몇 frame 연속 사용됐는가
- 곧 다시 사용할 가능성이 높은가
- camera velocity가 어느 방향인가

즉 residency heuristic은 `visibility + simulation activity + temporal persistence`의 결합 문제다.

이 구조는 graphics와 simulation을 동시에 다루는 엔진에서 특히 중요하다. Renderer만 보고 brick을 evict하면 compute가 다음 step에 바로 다시 요구할 수 있고, simulation만 보고 유지하면 화면에 필요 없는 high-resolution data가 VRAM을 차지한다.

### 4.9 Eviction에는 Hysteresis가 필요하다
Sparse streaming에서 가장 나쁜 현상 중 하나가 **thrashing**이다.

Frame N에서 brick A를 evict하고, N+1에서 다시 request하고, N+2에서 또 evict하면 physical memory는 절약되지만 page mapping·upload·sync 비용이 폭발한다.

그래서 residency 정책에는 보통 hysteresis 개념이 필요하다.

- 최근 사용된 brick에 grace period 제공
- high-priority와 low-priority threshold를 다르게 설정
- visible region보다 한두 brick 넓은 prefetch halo 유지
- camera velocity 방향으로 앞쪽 brick에 우선순위 부여
- memory pressure가 높을 때만 aggressive eviction

이것은 virtual texturing, clipmap, streaming cache와 같은 문제 구조다. Sparse resource API는 mechanism을 제공할 뿐, **좋은 residency policy는 engine이 결정한다.**

### 4.10 Memory Layout은 voxel layout보다 metadata traffic이 중요해질 수 있다
Dense grid에서는 voxel payload layout이 핵심이었다. Sparse architecture에서는 metadata access가 새로운 hot path가 된다.

예를 들어 resident lookup이 매 voxel마다 큰 AoS 구조를 random access하면 sparse로 줄인 payload memory보다 residency metadata latency가 병목이 될 수 있다.

그래서 GPU-side metadata는 보통 compact한 형태가 유리하다.

- compact residency bit/flag
- physical tile index
- version/epoch
- mip/brick coordinate와 별도 SoA
- hot metadata와 cold debug/statistics 분리

특히 shader가 필요로 하는 것은 대개 "resident인가? 어느 physical page인가?"에 가까운 hot data다. CPU streaming system이 필요한 last-use timestamp, debug owner, compression state까지 같은 structure에 넣으면 cache line 낭비가 커진다.

이전에 다룬 **Hot/Warm/Cold state splitting**이 여기서도 그대로 적용된다.

### 4.11 C++ Engine Architecture에서는 Residency Manager가 API 경계를 소유한다
C++ 관점에서는 volume 자체와 sparse mapping backend를 분리하는 편이 좋다.

개념적으로 다음 책임이 나뉜다.

**VirtualVolume**
- logical dimensions
- voxel semantic/format
- mip hierarchy
- BrickKey 계산

**ResidencyManager**
- request priority
- physical tile budget
- allocation/eviction
- lifetime epoch
- mapping state machine

**VulkanSparseBackend**
- sparse image requirements
- `VkDeviceMemory` tile pool
- sparse bind submission
- Vulkan synchronization

**CudaSparseBackend**
- CUDA sparse properties
- tile-pool allocation
- sparse array map/unmap
- CUDA stream synchronization

이렇게 나누면 renderer와 simulation은 "brick이 resident하고 version이 준비됐는가"라는 공통 contract를 보고, 실제 API mapping details는 backend 안에 남는다.

### 4.12 Profiling은 VRAM 사용량만 보면 안 된다
Sparse volume이 성공했는지는 다음 지표를 함께 봐야 한다.

- resident brick 수 / virtual brick 수
- physical pool occupancy
- brick request rate
- map/unmap operations per frame
- eviction/re-request ratio
- average residency lifetime
- missing/fallback sample ratio
- bytes uploaded/generated per frame
- sparse bind/map GPU time
- synchronization stall
- L2/cache hit behavior

특히 **eviction/re-request ratio가 높으면 capacity는 잘 줄였지만 working-set prediction은 실패하고 있다는 뜻**이다.

Graphics engineer 관점에서는 "VRAM을 70% 줄였다"보다 **frame-time variance, streaming bandwidth, residency churn을 함께 설명할 수 있어야 한다.**

## 5. 내 관심 분야와 연결
### Semiconductor Process Emulation / Level-Set
Level-set 전체 공간을 dense FP32 grid로 유지하면 대부분의 voxel이 surface와 멀리 떨어져 있어도 같은 메모리를 사용한다. Sparse volume에서는 **interface 주변 narrow band만 fine brick으로 resident**하게 하고, 멀리 떨어진 region은 coarse representation 또는 implicit background state로 둘 수 있다.

Process step이 진행되면서 surface가 움직일 때 active brick set도 이동한다. 이때 중요한 문제는 단순 allocation보다 **surface propagation 속도보다 조금 앞선 prefetch halo**를 유지해 next-step residency miss를 줄이는 것이다.

### Doping / Scalar Field Visualization
Doping field가 전체 domain에 정의돼 있어도 현재 cross-section이나 surface overlay에서 필요한 region은 제한적일 수 있다. Renderer는 camera/cross-section plane 기준으로 high-resolution brick을 요구하고, simulation은 compute activity 기준으로 brick을 요구한다.

두 request stream을 하나의 residency priority system에서 합치면 graphics와 compute가 별도의 duplicate cache를 갖는 것을 줄일 수 있다.

### Marching Cubes / Mesh Extraction
Marching Cubes는 active cell 주변의 neighborhood access가 필요하다. Sparse volume에서는 전체 virtual grid를 scan하기보다 **resident/active brick list**가 dispatch domain 역할을 할 수 있다.

이때 sparse residency가 mesh extraction algorithm 자체를 자동으로 sparse하게 만드는 것은 아니다. 중요한 것은 application이 brick metadata를 이용해 **work generation domain을 resident active set으로 제한**할 수 있다는 점이다.

### Vulkan + CUDA GPU-Stay-GPU Pipeline
Cross-API zero-copy에서 다음 단계는 "같은 allocation을 공유"하는 것보다 **어느 region이 실제로 필요한지를 공유**하는 것이다.

즉 GPU-stay-GPU pipeline의 성숙 단계는 다음과 같이 볼 수 있다.

`No CPU copy -> Shared GPU resource -> Explicit ownership -> Sparse residency -> Shared working-set policy`

이 단계까지 오면 data movement 최적화가 단순 copy 제거를 넘어 **VRAM footprint와 compute/render scheduling을 함께 다루는 engine architecture**가 된다.

## 6. 머릿속에 남길 질문 3개
1. **Sparse data structure(NanoVDB/octree)와 sparse residency(Vulkan Sparse Image/CUDA Sparse Array)는 각각 어떤 종류의 sparsity를 해결하며, 언제 함께 사용하는 것이 유리한가?**
2. **Logical brick size를 hardware sparse tile size와 분리하면 어떤 장점이 있고, 그 대신 어떤 metadata/scheduling 비용이 생기는가?**
3. **VRAM 사용량은 줄었는데 frame time이 오히려 나빠졌다면, residency churn·mapping cost·synchronization·cache locality 중 무엇부터 의심해야 하는가?**

## 7. graphics engineer 면접 질문 1개와 답변
### 질문
**"Vulkan sparse 3D image를 사용하면 큰 volume texture의 메모리 문제는 자동으로 해결되나요? Sparse residency를 실제 renderer/simulation engine에 넣을 때 가장 중요한 추가 설계는 무엇인가요?"**

### 답변
자동으로 해결되지 않는다. Sparse Image는 **virtual resource의 일부 region에만 physical memory를 binding할 수 있는 mechanism**을 제공하지만, 어떤 brick을 resident시킬지는 application이 결정한다.

실제 엔진에서는 최소한 네 가지가 필요하다.

첫째, `BrickKey -> PhysicalTile` mapping과 상태를 관리하는 **residency table**이 필요하다. Logical identity와 physical placement를 분리해야 eviction과 remapping이 안전하다.

둘째, camera visibility, projected size, simulation activity, temporal reuse를 이용한 **working-set priority policy**가 필요하다. 그렇지 않으면 page thrashing 때문에 sparse mapping 비용이 더 커질 수 있다.

셋째, sparse bind/map과 rendering/compute access 사이의 **synchronization 및 lifetime 관리**가 필요하다. Vulkan의 `vkQueueBindSparse()`와 CUDA의 `cuMemMapArrayAsync()`는 mapping change 자체가 GPU-visible operation이므로 consumer가 mapping/data population completion 전에 접근하지 않도록 해야 한다.

넷째, hardware sparse tile granularity와 application-level logical brick size를 분리해야 한다. API가 보고하는 tile/mip-tail requirement를 존중하면서, engine은 여러 hardware tile을 하나의 streaming brick으로 묶어 request overhead와 portability를 관리할 수 있다.

따라서 핵심은 "sparse texture API를 썼다"가 아니라 **virtual address space, physical tile pool, residency policy, synchronization을 하나의 resource-state architecture로 만들었다**는 점이다.

## 8. 포트폴리오 / 커리어 연결
이 주제는 단순한 Vulkan API 사용 경험보다 훨씬 강한 graphics-engine signal을 만든다.

포트폴리오에서 가치가 높은 설명은 다음과 같다.

- 왜 dense 3D volume이 VRAM bottleneck이 되었는지 수치로 설명
- virtual extent와 physical working set을 분리한 이유
- sparse data structure와 sparse residency를 어떻게 구분했는지
- logical brick/hardware tile abstraction을 왜 나눴는지
- CUDA compute와 Vulkan rendering의 residency request를 어떻게 통합했는지
- eviction hysteresis와 prefetch가 frame-time stability에 어떤 영향을 주는지
- VRAM 절감뿐 아니라 mapping churn, bandwidth, synchronization stall까지 어떻게 측정했는지

게임 엔진에서는 virtual texturing, terrain, voxel GI, large-world streaming과 직접 연결되고, scientific visualization에서는 large volume rendering, adaptive field visualization, out-of-core GPU data management와 연결된다.

면접에서도 이 주제는 **GPU virtual memory, sparse resource API, cache locality, streaming, render graph synchronization, data-oriented C++ architecture**를 한 번에 묶어 설명할 수 있어 intermediate에서 advanced graphics engineer로 넘어가는 좋은 지점이다.

## 9. 내일 이어서 볼 개념
**GPU Feedback-Driven Sparse Residency: Residency Maps, Page Requests, and Visibility-Guided Brick Prioritization**

오늘은 CPU/application이 어떤 brick을 resident하게 만들지 결정하는 architecture를 봤다. 다음에는 shader/ray/volume traversal이 실제로 어떤 virtual region을 요구했는지를 **GPU feedback**으로 수집하고, request compaction·deduplication·priority scoring을 통해 다음 frame의 sparse working set을 결정하는 구조를 본다.

연결 흐름은 다음과 같다.

`Sparse Virtual Volume -> Brick Residency -> GPU Access Feedback -> Request Compaction -> Priority Queue -> Prefetch/Eviction -> Stable Working Set`

이 단계는 virtual texturing의 feedback map, mesh/texture streaming, sparse voxel/volume rendering뿐 아니라 GPU-driven simulation scheduling에도 직접 연결된다.

## 10. 참고 키워드
- Vulkan Sparse Resources
- `VK_IMAGE_CREATE_SPARSE_BINDING_BIT`
- `VK_IMAGE_CREATE_SPARSE_RESIDENCY_BIT`
- `sparseResidencyImage3D`
- `vkGetImageSparseMemoryRequirements`
- `VkSparseImageMemoryRequirements`
- `VkSparseImageMemoryBind`
- `vkQueueBindSparse`
- `residencyNonResidentStrict`
- `shaderResourceResidency`
- Sparse Image Block
- Mip Tail
- CUDA Sparse Array
- `CUDA_ARRAY3D_SPARSE`
- `CUDA_ARRAY_SPARSE_PROPERTIES`
- `cuArrayGetSparseProperties`
- `cuMipmappedArrayGetSparseProperties`
- `CU_MEM_CREATE_USAGE_TILE_POOL`
- `CUarrayMapInfo`
- `cuMemMapArrayAsync`
- Virtual Texture / Virtual Volume
- Brick Streaming
- Physical Tile Pool
- Working-Set Control
- Residency Hysteresis
- Prefetch Halo
- Page Thrashing
- Narrow-Band Level Set
- GPU Virtual Memory
- Sparse Voxel / Sparse Grid
- NanoVDB
- Out-of-Core Rendering
- C++ Resource State Machine

### 공식 참고 문서
- Khronos Vulkan Specification — Sparse Resources: https://docs.vulkan.org/spec/latest/chapters/sparsemem.html
- Khronos Vulkan Guide — Sparse Resources: https://docs.vulkan.org/guide/latest/sparse_resources.html
- Khronos Vulkan Sample — Sparse Image: https://docs.vulkan.org/samples/latest/samples/extensions/sparse_image/README.html
- NVIDIA CUDA Driver API — Virtual Memory / `cuMemMapArrayAsync`: https://docs.nvidia.com/cuda/cuda-driver-api/group__CUDA__VA.html
- NVIDIA CUDA Driver API — Sparse Array Properties: https://docs.nvidia.com/cuda/cuda-driver-api/group__CUDA__MEM.html
- NVIDIA CUDA Programming Guide — External Resource Interoperability: https://docs.nvidia.com/cuda/cuda-programming-guide/04-special-topics/graphics-interop.html
