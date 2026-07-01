---
title: "Page Table and Indirection in GPU Rendering"
date: "2026-07-01"
category: "Graphics"
tags: ["GPU", "Rendering Pipeline", "Page Table", "Indirection", "Virtual Texturing", "Sparse Voxel", "Bindless", "GPU Memory", "Scientific Visualization"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-07-01 - Page Table and Indirection in GPU Rendering

## 1. 오늘의 개념

**Page Table and Indirection in GPU Rendering**은 shader가 texture, voxel brick, material, field block, meshlet data를 직접 고정 주소로 접근하지 않고, 먼저 **index / virtual address / id**를 통해 lookup table을 조회한 뒤 실제 physical resource 위치를 찾아가는 구조다.

이 개념은 이전 노트의 Sparse Residency and Virtual Texturing과 직접 연결된다. Virtual Texturing에서는 shader가 UV로 virtual page를 계산하고, page table을 통해 physical page pool의 위치를 찾아 sample한다. Sparse voxel renderer에서는 ray가 world-space position으로 virtual brick id를 계산하고, page table 또는 sparse tree를 통해 실제 resident brick 위치를 찾는다.

핵심 변화는 다음이다.

> Shader가 data를 직접 읽는 구조에서, shader가 먼저 “어디에 있는지”를 찾고 그 결과로 data를 읽는 구조로 이동한다.

이 indirection 구조는 다음 시스템에서 반복적으로 등장한다.

- Virtual Texturing
- Sparse Residency
- Sparse Voxel / Voxel Brick Pool
- Bindless Material Table
- Visibility Buffer Shading
- Meshlet / Primitive Data Fetch
- GPU-driven Rendering
- CFD Field Block Streaming
- NanoVDB-style hierarchical lookup

즉 page table은 단순 texture streaming용 자료구조가 아니라, **large-scale GPU data를 virtual space와 physical memory로 분리하는 기본 패턴**이다.

## 2. 한 줄 핵심

**Page Table and Indirection은 shader가 virtual id를 physical resource 위치로 변환해, 거대한 texture/voxel/material/field data를 제한된 GPU memory 안에서 유연하게 접근하게 만드는 GPU data access pattern이다.**

## 3. 왜 중요한가

Large-scale renderer에서는 모든 data를 고정된 dense 배열로 들고 있기 어렵다. Texture, volume, voxel, material, meshlet, CFD field block은 너무 많거나, frame마다 필요한 subset이 달라질 수 있다.

Indirection이 중요한 이유는 다음과 같다.

- virtual resource space와 physical GPU memory를 분리할 수 있다.
- 필요한 page/brick/material만 resident하게 유지할 수 있다.
- bindless resource table처럼 shader가 index로 data를 선택할 수 있다.
- visibility buffer에서 primitive id를 통해 후속 shading data를 찾을 수 있다.
- streaming, LOD, sparse representation과 결합할 수 있다.
- data layout 변경을 shader interface와 분리할 수 있다.

Graphics engineer 관점에서 page table은 단순 자료구조가 아니라, **GPU memory capacity, cache locality, streaming latency, shader flexibility 사이의 trade-off를 만드는 중심 구조**다.

## 4. 구현 관점

### 4.1 Direct access와 indirect access의 차이

Direct access는 shader가 resource의 위치를 이미 알고 있다고 가정한다.

```glsl
vec4 value = texture(albedoTexture, uv);
```

Indirect access는 먼저 table을 조회한다.

```glsl
uint pageId = ComputeVirtualPage(uv);
PageEntry entry = pageTable[pageId];
vec2 physicalUV = RemapToPhysicalPage(entry, uv);
vec4 value = texture(physicalPagePool, physicalUV);
```

이 구조는 한 번 더 lookup이 필요하므로 비용이 든다. 하지만 대신 resource 전체를 고정된 physical memory에 둘 필요가 없고, page table mapping만 바꾸면 virtual resource의 physical 위치를 바꿀 수 있다.

### 4.2 Page table entry 설계

Page table entry는 virtual page가 실제 어디에 있는지 알려준다. 2D virtual texture라면 다음 정보를 가질 수 있다.

```cpp
struct PageEntry
{
    uint physicalPageX;
    uint physicalPageY;
    uint mipLevel;
    uint residencyFlags;
};
```

3D voxel brick이나 field block이라면 다음처럼 확장될 수 있다.

```cpp
struct BrickPageEntry
{
    uint brickPoolIndex;
    uint lodLevel;
    uint materialOrFieldId;
    uint flags;
};
```

중요한 설계 포인트는 다음이다.

- entry 크기를 compact하게 유지한다.
- shader에서 branch가 많아지지 않도록 한다.
- missing page fallback 정보를 포함한다.
- LOD level과 residency state를 빠르게 판단할 수 있게 한다.
- cache-friendly한 table layout을 선택한다.

### 4.3 Virtual Texturing에서의 indirection

Virtual Texturing에서는 UV와 mip level로 virtual page coordinate를 계산한다. Shader는 page table을 통해 physical tile을 찾고, local UV를 physical pool coordinate로 remap한다.

개념 흐름은 다음이다.

```text
UV → virtual page id → page table lookup → physical page id → physical texture coordinate → sample
```

이때 필요한 추가 구조는 다음이다.

- feedback buffer: 이번 frame에 필요한 page 기록
- physical page pool: resident tile storage
- page table update: streaming 결과 반영
- fallback mapping: missing page가 있을 때 parent mip 사용

즉 page table은 sampling path와 streaming path를 연결한다.

### 4.4 Sparse voxel / volume에서의 indirection

Sparse voxel renderer에서는 world position 또는 ray traversal 결과로 brick id를 찾는다.

```text
world position → virtual brick coordinate → page table / sparse tree lookup → physical brick pool index → density/SDF/field sample
```

3D volume에서는 page table이 3D texture일 수도 있고, buffer-based hash table일 수도 있고, octree / B-tree / NanoVDB-style hierarchy일 수도 있다.

선택 기준은 다음이다.

- lookup이 몇 번 필요한가?
- memory footprint는 얼마나 큰가?
- empty space를 얼마나 잘 압축하는가?
- ray marching 중 coherent access가 가능한가?
- brick streaming과 eviction이 쉬운가?

Sparse voxel에서는 lookup 비용이 ray marching step마다 반복될 수 있다. 따라서 indirection depth가 너무 깊으면 shading보다 lookup이 병목이 될 수 있다.

### 4.5 Bindless material table과의 연결

Bindless Resources도 indirection 구조다. Object나 material은 texture pointer를 직접 들고 있는 것이 아니라 texture index를 들고 있다.

```glsl
ObjectData obj = objectBuffer[objectId];
MaterialData mat = materialBuffer[obj.materialId];
vec4 albedo = texture(textures[mat.albedoTextureIndex], uv);
```

여기서 `materialId`와 `albedoTextureIndex`가 indirection이다. Object → Material → Texture resource로 이어지는 lookup chain이다.

이 구조는 renderer를 유연하게 만들지만 다음 비용도 만든다.

- random material access
- texture cache locality 저하 가능성
- descriptor indexing 제한
- non-uniform indexing 주의
- resource lifetime / index recycling 관리

즉 bindless도 page table과 같은 큰 패턴 안에서 볼 수 있다.

### 4.6 Visibility Buffer와의 연결

Visibility Buffer Rendering에서는 pixel에 primitive attribute를 직접 저장하지 않고 primitive id, meshlet id, material id 같은 reference를 저장한다.

후속 shading pass는 다음 indirection을 수행한다.

```text
pixel → visibility value → meshlet/primitive id → vertex/index buffer → material id → texture/resource table → shading
```

이 구조는 G-buffer bandwidth를 줄이지만 lookup chain이 길어진다. 따라서 performance는 다음에 의해 결정된다.

- visibility buffer compactness
- primitive data layout
- vertex fetch coherence
- material table cache locality
- texture sampling locality

Visibility Buffer는 “memory bandwidth를 줄이고 indirection과 recomputation 비용을 지불하는 구조”로 볼 수 있다.

### 4.7 Cache locality와 divergence

Indirection은 유연하지만 cache locality를 해칠 수 있다. 같은 warp/wave 안의 thread가 서로 다른 page, brick, material을 lookup하면 memory access가 분산된다.

문제가 되는 상황은 다음이다.

- pixel마다 다른 material table entry 접근
- ray마다 다른 voxel brick 접근
- virtual texture page가 화면에서 불연속적으로 샘플링됨
- page table lookup이 여러 단계로 깊음
- missing page fallback branch가 많음

완화 전략은 다음이다.

- screen-space tiling / clustering
- material sorting
- brick ordering / Morton order
- page table entry compact packing
- parent fallback을 branch-light하게 구성
- coherent ray marching order 유지
- visible list compaction 시 locality 고려

즉 indirection 구조를 설계할 때는 “찾을 수 있는가”보다 “coherent하게 찾을 수 있는가”가 중요하다.

### 4.8 CPU/GPU synchronization 문제

Page table은 streaming system과 연결될 때 CPU/GPU synchronization 문제를 만든다.

예를 들어 GPU feedback을 CPU가 읽고 page request를 만든다면 readback latency가 생긴다. GPU-only feedback과 GPU-driven page request 구조를 만들면 CPU stall은 줄일 수 있지만 구현 복잡도가 커진다.

일반적인 frame flow는 다음과 같다.

```text
Frame N: GPU feedback 기록
Frame N+1: CPU/GPU가 feedback 분석, page request 생성
Frame N+2: page upload와 page table update 반영
```

Streaming latency가 있으므로 missing page fallback이 필수다. Virtual texturing이나 sparse voxel streaming에서는 “바로 고해상도가 보이는 것”보다 “낮은 해상도라도 안정적으로 보이고, 시간이 지나며 refine되는 것”이 중요하다.

## 5. 내 관심 분야와 연결

### CFD / scientific visualization

CFD 대용량 field에서는 block id, field id, variable id, time step id가 모두 indirection의 대상이 될 수 있다.

예를 들어 다음 구조가 가능하다.

```text
screen pixel → visible cell/block id → field page table → pressure/velocity/temperature block → colormap/shading
```

이 구조는 전체 field를 G-buffer나 dense 3D texture에 저장하지 않고, 필요한 block만 fetch하게 해준다. 다만 scientific visualization에서는 missing page나 낮은 LOD가 해석 오류로 이어질 수 있으므로 fallback과 error 표시가 중요하다.

### Sparse voxel / octree / NanoVDB

Sparse voxel renderer는 indirection의 집합이다.

- world position → node index
- node index → child pointer
- brick id → physical brick pool index
- material id → transfer function index
- field id → scalar buffer index

NanoVDB도 hierarchy traversal을 통해 sparse volume data를 찾는다. 핵심은 empty space를 압축하면서도 GPU에서 lookup이 빠르게 이루어지도록 memory layout을 구성하는 것이다.

### Game engine architecture

Game engine에서는 indirection이 everywhere다.

- bindless material table
- virtual texture page table
- virtual shadow map page cache
- meshlet data table
- visibility buffer primitive lookup
- GPU scene object table
- animation / skinning buffer index

면접에서 이 개념을 설명할 때는 “포인터를 한 번 더 탄다”가 아니라, **virtual resource space와 physical memory layout을 분리하는 방식**이라고 말하는 것이 좋다.

## 6. 머릿속에 남길 질문 3개

1. Page table indirection이 GPU memory 사용량을 줄이는 대신 어떤 shader 비용과 cache 문제를 만들 수 있는가?
2. Virtual Texturing, Sparse Voxel, Bindless Material Table은 모두 어떤 공통된 lookup 구조를 공유하는가?
3. CFD field block streaming에서 page table 구조를 사용할 때 missing page fallback을 어떻게 설계해야 data 해석 오류를 줄일 수 있는가?

## 7. graphics engineer 면접 질문 1개와 답변

### Q. GPU rendering에서 page table 또는 indirection 구조는 왜 사용하며, 어떤 trade-off가 있나요?

**A.** Page table과 indirection 구조는 shader가 virtual id나 logical coordinate를 실제 physical resource 위치로 변환하기 위해 사용합니다. Virtual Texturing에서는 virtual page id를 physical texture tile로 매핑하고, Sparse Voxel renderer에서는 virtual brick coordinate를 physical brick pool index로 매핑합니다. Bindless material system에서도 object id에서 material id, texture index로 이어지는 lookup chain이 indirection입니다.

장점은 virtual resource space와 physical GPU memory를 분리할 수 있다는 점입니다. 필요한 page, brick, material만 resident하게 유지할 수 있고, streaming이나 sparse representation과 잘 맞습니다. 단점은 shader lookup이 추가되고, cache locality가 나빠질 수 있으며, wave 내부 thread가 서로 다른 page나 material을 접근하면 divergence와 random memory access가 증가한다는 점입니다. 따라서 indirection은 유연성을 주지만, table layout, access coherence, fallback, streaming latency를 함께 설계해야 합니다.

## 8. 포트폴리오 / 커리어 연결

Page Table and Indirection in GPU Rendering은 포트폴리오에서 다음 메시지를 만든다.

> “나는 GPU resource를 단순 buffer/texture로만 보지 않고, virtual id와 physical memory를 분리하는 lookup architecture로 설계할 수 있다.”

네 배경과 연결하면 다음 표현이 좋다.

- CFD / VTK 대용량 field를 block/page table 기반으로 관리하는 사고
- Sparse voxel / NanoVDB renderer에서 brick pool과 page table lookup 구조 이해
- Visibility Buffer에서 primitive id 기반 deferred shading 구조 이해
- Bindless material table과 virtual texturing을 같은 indirection pattern으로 설명 가능
- WebGPU / Vulkan renderer에서 storage buffer 기반 table-driven rendering으로 확장 가능

면접에서는 다음처럼 말할 수 있다.

> “Page table indirection은 shader가 logical id를 physical resource 위치로 변환하는 구조입니다. 이 구조는 virtual texturing, sparse voxel brick pool, bindless material table, visibility buffer shading에 공통으로 나타나며, memory flexibility를 얻는 대신 lookup cost와 cache locality 문제를 관리해야 합니다.”

## 9. 내일 이어서 볼 개념

**Morton Order and GPU Memory Locality**

Page table과 indirection을 이해한 다음에는 Morton order를 보는 것이 자연스럽다. Indirection이 많아질수록 lookup 자체보다 memory locality가 중요해지고, Morton/Z-order는 2D/3D spatial data를 cache-friendly하게 배치하는 대표적인 방식이다.

## 10. 참고 키워드

- Page Table
- Indirection
- Virtual Address
- Physical Resource
- Virtual Texturing
- Sparse Residency
- Sparse Voxel
- Brick Pool
- Bindless Material Table
- Visibility Buffer
- Meshlet Data Fetch
- GPU Scene Table
- Feedback Buffer
- Missing Page Fallback
- Cache Locality
- Memory Coherence
- Morton Order
- NanoVDB
- Scientific Visualization
