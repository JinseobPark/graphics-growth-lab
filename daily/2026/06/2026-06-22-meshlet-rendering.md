---
title: "Meshlet Rendering"
date: "2026-06-22"
category: "Graphics"
tags: ["GPU", "Rendering Pipeline", "Meshlet", "Cluster Culling", "Mesh Shader", "GPU-Driven Rendering", "Visibility Buffer", "LOD", "Real-time Rendering"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-06-22 - Meshlet Rendering

## 1. 오늘의 개념

**Meshlet Rendering**은 큰 mesh를 GPU가 처리하기 좋은 작은 geometry cluster, 즉 **meshlet** 단위로 나누고, 각 meshlet에 대해 culling, LOD, shading data fetch, indirect draw 또는 mesh shader dispatch를 수행하는 렌더링 구조다.

전통적인 renderer에서는 mesh 하나가 draw call의 기본 단위가 되는 경우가 많다. 하지만 큰 mesh 안에서도 실제 화면에 보이는 triangle은 일부일 수 있다. Object는 frustum 안에 있지만 object 내부의 상당 부분은 화면 밖이거나, 다른 geometry에 가려지거나, 너무 작아서 낮은 LOD로 처리해도 충분할 수 있다.

Meshlet은 이 문제를 해결하기 위해 mesh를 작은 단위로 나눈다.

일반적인 meshlet은 다음 데이터를 가진다.

- local vertex index list
- local primitive / triangle index list
- bounding sphere 또는 AABB
- cone culling 정보
- material 또는 submesh reference
- LOD / cluster hierarchy reference

핵심 변화는 다음이다.

> Mesh 전체를 렌더링 단위로 보는 것이 아니라, GPU가 빠르게 판정할 수 있는 작은 cluster를 visibility와 shading의 단위로 사용한다.

## 2. 한 줄 핵심

**Meshlet Rendering은 mesh를 작은 GPU-friendly cluster로 나누어, object 내부 geometry까지 culling하고 GPU-driven rendering / visibility buffer / mesh shader와 연결하는 modern geometry pipeline이다.**

## 3. 왜 중요한가

현대 real-time renderer는 단순히 object 수만 많은 것이 아니라, object 하나의 geometry density도 매우 높다. CAD model, photogrammetry mesh, scanned asset, Nanite-style virtual geometry, terrain, dense scientific mesh에서는 object 단위 culling만으로는 부족하다.

Meshlet Rendering이 중요한 이유는 다음과 같다.

- object 내부의 보이지 않는 triangle cluster를 제거할 수 있다.
- GPU-driven culling의 단위를 object보다 더 작게 만들 수 있다.
- mesh shader pipeline과 잘 맞는다.
- visibility buffer에서 primitive id / meshlet id 기반 shading과 연결된다.
- LOD, streaming, virtual geometry의 단위로 확장 가능하다.
- large CAD / CFD / semiconductor visualization mesh에 유리하다.

Graphics engineer 관점에서는 Meshlet이 단순 asset preprocessing 결과가 아니라, **GPU memory layout과 visibility pipeline을 바꾸는 geometry representation**이라는 점이 핵심이다.

## 4. 구현 관점

### 4.1 Meshlet의 기본 구조

Meshlet은 보통 제한된 수의 vertex와 triangle을 가진 작은 cluster다. 예를 들어 하나의 meshlet이 최대 64 vertices, 126 triangles를 가지도록 만들 수 있다.

```cpp
struct Meshlet
{
    uint vertexOffset;
    uint vertexCount;
    uint primitiveOffset;
    uint primitiveCount;
    vec4 boundingSphere;
    vec4 coneCullingData;
    uint materialIndex;
};
```

별도 buffer에는 meshlet별 local vertex index와 primitive index가 저장된다.

```cpp
StructuredBuffer<Meshlet> meshlets;
StructuredBuffer<uint> meshletVertexIndices;
StructuredBuffer<uint> meshletPrimitiveIndices;
StructuredBuffer<Vertex> vertices;
```

이 구조에서는 GPU가 meshlet 단위로 bounds를 읽고, 보이는 meshlet만 shading / rasterization 단계로 넘길 수 있다.

### 4.2 Object culling과 Meshlet culling의 차이

Object culling은 object의 bounding box나 bounding sphere가 camera frustum에 들어오는지만 판단한다. Object가 보이면 그 object의 전체 mesh를 draw할 가능성이 높다.

Meshlet culling은 object 내부를 더 작은 cluster로 나누어 판단한다.

- Object는 보이지만 일부 meshlet은 frustum 밖일 수 있다.
- Object는 보이지만 일부 meshlet은 backface cone culling으로 제거될 수 있다.
- Object는 보이지만 일부 meshlet은 Hi-Z occlusion으로 가려질 수 있다.
- Object는 보이지만 멀리 있는 meshlet은 낮은 LOD로 대체될 수 있다.

즉 Meshlet Rendering은 visibility granularity를 object에서 geometry cluster로 낮춘다.

### 4.3 Cone culling

Meshlet은 작은 triangle 집합이므로, 그 안의 triangle normal 방향이 어느 정도 비슷할 수 있다. 이 정보를 이용하면 backface culling을 meshlet 단위로 수행할 수 있다.

Cone culling은 meshlet의 normal cone을 저장하고, camera view direction과 비교해 전체 meshlet이 뒤를 보고 있으면 제거한다.

장점은 triangle 단위 rasterization 전에 cluster 전체를 제거할 수 있다는 점이다. 단점은 너무 다양한 normal을 가진 meshlet에서는 cone이 넓어져 culling 효율이 떨어진다.

좋은 meshlet 생성기는 단순히 triangle 수만 맞추는 것이 아니라, spatial locality와 normal coherence를 함께 고려해야 한다.

### 4.4 Mesh shader와의 연결

Meshlet은 modern GPU의 **Mesh Shader** pipeline과 잘 맞는다. Mesh shader에서는 vertex shader + primitive assembly 일부를 대체해, thread group이 하나의 meshlet을 처리하고 필요한 primitive만 emit할 수 있다.

개념적으로는 다음 흐름이다.

1. Task shader 또는 compute shader가 meshlet visibility를 판단한다.
2. 보이는 meshlet만 mesh shader로 전달한다.
3. Mesh shader가 meshlet vertex / primitive를 읽는다.
4. GPU 내부에서 primitive를 생성해 rasterization으로 넘긴다.

전통적인 vertex/index buffer draw에서는 GPU가 index buffer를 따라 vertex를 처리한다. Mesh shader 기반에서는 meshlet 단위로 geometry amplification / culling / LOD 선택을 더 명시적으로 제어할 수 있다.

Vulkan에서는 mesh shader extension, DirectX 12에서는 Mesh Shader pipeline, Metal은 object/mesh shader 개념과 유사한 구조를 플랫폼별로 고려해야 한다. WebGPU는 아직 native mesh shader를 직접 제공하지 않지만, compute shader + indirect draw + storage buffer 기반으로 meshlet-like pipeline을 학습할 수 있다.

### 4.5 GPU-driven rendering과 연결

Meshlet Rendering은 GPU-driven rendering과 자연스럽게 연결된다.

대표 pipeline은 다음과 같다.

1. CPU가 meshlet buffer와 object transform buffer를 준비한다.
2. Compute shader가 object culling을 수행한다.
3. Visible object의 meshlet에 대해 frustum / cone / Hi-Z culling을 수행한다.
4. Visible meshlet list를 compact한다.
5. Indirect draw 또는 mesh shader dispatch command를 생성한다.
6. Rasterization / visibility buffer pass를 수행한다.

이 구조에서 CPU는 meshlet별 draw를 직접 제출하지 않는다. GPU가 visible meshlet list를 만들고, 그 결과를 다음 pass가 소비한다.

### 4.6 Visibility Buffer와의 연결

이전 노트의 Visibility Buffer Rendering과 Meshlet은 잘 맞는다. Visibility Buffer에는 pixel별로 다음 정보를 저장할 수 있다.

- meshlet id
- local primitive id
- instance id
- material id
- depth

후속 shading pass에서는 meshlet id와 local primitive id를 이용해 vertex/index data를 fetch하고, barycentric interpolation으로 normal, uv, tangent, material attribute를 복원한다.

즉 Meshlet은 Visibility Buffer에서 “primitive attribute를 다시 찾기 위한 주소 체계”가 된다.

```text
Visibility Buffer value
= instance id + meshlet id + local primitive id
```

이 구조는 G-buffer bandwidth를 줄이는 대신, shading pass에서 meshlet data fetch와 attribute reconstruction 비용을 지불한다.

### 4.7 Memory layout 관점

Meshlet Rendering에서 성능은 meshlet buffer layout에 크게 좌우된다.

중요한 설계 요소는 다음이다.

- meshlet metadata는 compact하게 저장한다.
- vertex index / primitive index는 연속 memory에 배치한다.
- meshlet 내부 vertex reuse를 높인다.
- bounding volume은 culling shader가 빠르게 읽을 수 있어야 한다.
- material index와 resource index는 bindless table과 연결한다.
- meshlet list compaction 시 atomic contention을 관리한다.

Meshlet은 작을수록 culling granularity는 좋아지지만 metadata와 dispatch overhead가 증가한다. 반대로 meshlet이 너무 크면 culling 효율이 떨어진다. 따라서 meshlet 크기는 GPU wave size, cache locality, triangle density, asset 특성에 맞춰 조정해야 한다.

### 4.8 LOD와 virtual geometry

Meshlet은 LOD 시스템의 단위로도 사용될 수 있다. 멀리 있는 meshlet은 더 낮은 해상도 cluster로 대체하고, 가까운 meshlet은 더 세밀한 cluster를 사용할 수 있다.

Nanite-style virtualized geometry는 매우 고도화된 cluster hierarchy를 사용한다. 핵심 사고는 비슷하다.

- geometry를 cluster 단위로 나눈다.
- screen-space error로 LOD를 선택한다.
- visible cluster만 rasterize한다.
- 필요한 cluster data만 streaming한다.

Meshlet Rendering은 이러한 virtual geometry renderer를 이해하기 위한 기본 단위다.

## 5. 내 관심 분야와 연결

### CFD / scientific visualization

CFD visualization에서는 iso-surface, boundary mesh, cut surface, streamline tube, particle surface가 매우 많은 triangle을 가질 수 있다. Object 단위로만 culling하면 내부의 보이지 않는 geometry를 제거하기 어렵다.

Meshlet 개념을 적용하면 scientific mesh를 cluster 단위로 나누고, 각 cluster에 대해 bounds, scalar range, vector range, material field index를 저장할 수 있다.

예를 들어 다음 구조가 가능하다.

- iso-surface meshlet별 bounding box
- meshlet별 scalar value min/max
- meshlet별 material / field block reference
- Hi-Z 기반 occlusion culling
- visibility buffer에 meshlet id 저장

이렇게 하면 대용량 CFD surface visualization에서 보이는 cluster만 shading하거나, 특정 scalar range에 해당하는 meshlet만 강조할 수 있다.

### Sparse voxel / octree / NanoVDB

Sparse voxel이나 octree에서는 meshlet과 유사하게 brick 또는 node가 rendering 단위가 된다. Meshlet은 triangle cluster이고, voxel brick은 volume cluster다. 둘 다 GPU가 bounds를 보고 빠르게 visibility와 LOD를 판단한다는 공통점이 있다.

따라서 Meshlet Rendering을 이해하면 다음 구조를 더 쉽게 설계할 수 있다.

- voxel brick culling
- octree node LOD selection
- marching cubes 결과 mesh cluster화
- SDF surface patch 단위 visibility
- brick id 기반 visibility buffer

반도체 3D visualization에서는 layer surface를 meshlet화하고, 내부 volume field는 brick화하는 hybrid renderer로 확장할 수 있다.

### Game engine architecture

Game engine에서는 Meshlet이 modern geometry pipeline의 핵심 단위다.

- mesh shader
- GPU-driven culling
- cluster LOD
- visibility buffer
- virtual geometry
- many-object renderer
- dense asset rendering

면접에서 Meshlet을 설명할 때는 “작은 mesh 조각”보다 “GPU가 visibility와 geometry processing을 효율적으로 수행하기 위한 cluster representation”이라고 말하는 것이 좋다.

## 6. 머릿속에 남길 질문 3개

1. Meshlet Rendering이 object 단위 culling보다 더 세밀한 visibility optimization을 가능하게 하는 이유는 무엇인가?
2. Visibility Buffer에서 meshlet id와 local primitive id를 저장하면 후속 shading pass에서 어떤 정보를 복원할 수 있는가?
3. CFD iso-surface나 sparse voxel marching cubes 결과를 meshlet 단위로 나누면 어떤 culling / LOD 전략이 가능해지는가?

## 7. graphics engineer 면접 질문 1개와 답변

### Q. Meshlet Rendering이란 무엇이며, GPU-driven rendering과 어떤 관계가 있나요?

**A.** Meshlet Rendering은 큰 mesh를 작은 geometry cluster인 meshlet으로 나누고, 각 meshlet 단위로 frustum culling, cone culling, occlusion culling, LOD selection을 수행하는 렌더링 구조입니다. Object 단위 culling은 object가 보이면 전체 mesh를 처리하기 쉽지만, meshlet 단위 culling은 object 내부의 보이지 않는 triangle cluster까지 제거할 수 있습니다.

GPU-driven rendering과의 관계는 meshlet이 GPU가 직접 visibility를 판단하고 draw/dispatch list를 만들기 좋은 단위라는 점입니다. Compute shader나 task shader가 meshlet bounds와 cone data를 읽어 visible meshlet list를 만들고, indirect draw 또는 mesh shader pipeline이 이 list를 소비할 수 있습니다. Visibility Buffer Rendering에서는 meshlet id와 local primitive id를 저장해 후속 shading pass에서 vertex attribute와 material 정보를 재구성할 수도 있습니다. 단점은 meshlet preprocessing, metadata 관리, buffer layout, compaction, LOD hierarchy 설계가 복잡해진다는 점입니다.

## 8. 포트폴리오 / 커리어 연결

Meshlet Rendering은 포트폴리오에서 다음 메시지를 만든다.

> “나는 draw call 단위의 mesh rendering을 넘어, geometry를 GPU-friendly cluster로 재구성하고 visibility / LOD / shading data fetch 단위로 설계하는 modern renderer architecture를 이해한다.”

네 배경과 연결하면 다음 표현이 좋다.

- OpenGL renderer에서 mesh / submesh / material 구조를 설계한 경험
- CFD iso-surface나 VTK surface mesh를 cluster 단위로 분할하는 사고
- Sparse voxel / marching cubes 결과를 meshlet-like cluster로 관리하는 확장성
- WebGPU / Vulkan에서 storage buffer 기반 meshlet metadata와 indirect rendering 구조로 연결

면접에서는 다음처럼 말할 수 있다.

> “Meshlet은 단순한 triangle batch가 아니라, GPU culling, LOD, visibility buffer shading, mesh shader pipeline의 공통 단위입니다. 특히 large geometry나 scientific visualization에서는 object보다 작은 cluster 단위 visibility가 성능에 중요합니다.”

## 9. 내일 이어서 볼 개념

**Mesh Shader Pipeline**

Meshlet Rendering 다음에는 Mesh Shader Pipeline으로 이어지는 것이 자연스럽다. Meshlet이 geometry cluster라면, mesh shader는 이 cluster를 GPU thread group이 직접 읽고 primitive를 생성하는 modern programmable geometry stage다.

## 10. 참고 키워드

- Meshlet Rendering
- Meshlet
- Cluster Culling
- Cone Culling
- Mesh Shader
- Task Shader
- GPU-Driven Rendering
- Visibility Buffer
- Primitive ID
- Local Primitive ID
- Meshlet ID
- Indirect Draw
- MultiDrawIndirect
- LOD Selection
- Screen-space Error
- Virtual Geometry
- Nanite-style Rendering
- Sparse Voxel Brick
- Marching Cubes Cluster
- Scientific Visualization
