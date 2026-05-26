---
title: "GPU-Driven Rendering"
date: "2026-05-26"
category: "Graphics"
tags: ["GPU", "Rendering Pipeline", "Compute Shader", "Vulkan", "WebGPU", "Indirect Draw", "Culling", "Game Engine"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-05-26 - GPU-Driven Rendering

## 1. 오늘의 개념

**GPU-Driven Rendering**은 CPU가 모든 draw call, visibility 판단, 객체 정렬, LOD 선택을 직접 통제하는 방식에서 벗어나, GPU가 스스로 렌더링할 객체를 선별하고 draw command를 생성하거나 갱신하는 렌더링 구조다.

전통적인 렌더링 파이프라인에서는 CPU가 scene graph를 순회하고, frustum culling, material sorting, draw call submission을 수행한 뒤 GPU에 명령을 넘긴다. 하지만 객체 수가 수십만~수백만 개로 증가하면 CPU submission cost와 driver overhead가 병목이 된다. GPU-Driven Rendering은 이 병목을 줄이기 위해 visibility, LOD, batching, indirect draw command generation을 GPU compute stage로 이동시킨다.

핵심 구성 요소는 보통 다음과 같다.

- **GPU culling**: frustum culling, occlusion culling, cone culling 등을 compute shader에서 수행
- **Indirect draw**: `DrawIndirect`, `MultiDrawIndirect`, `ExecuteIndirect` 같은 API를 사용해 GPU buffer에 저장된 draw command를 실행
- **Compact visible list**: 보이는 객체만 append/compact하여 후속 draw stage에서 사용
- **GPU-side LOD selection**: 거리, screen-space error, projected size 기반으로 GPU에서 LOD 선택
- **Bindless / descriptor indexing**: 많은 mesh/material/texture를 CPU bind 호출 없이 GPU index 기반으로 접근

## 2. 한 줄 핵심

**GPU-Driven Rendering은 CPU가 “무엇을 그릴지” 매번 결정하는 구조에서, GPU가 visibility와 draw command를 직접 구성하는 구조로 렌더링 주도권을 이동시키는 방식이다.**

## 3. 왜 중요한가

현대 real-time rendering은 단순히 shader를 빠르게 작성하는 문제를 넘어, **얼마나 많은 객체를 낮은 CPU overhead로 GPU에 공급할 수 있는가**의 문제로 바뀌었다. 특히 open world, city-scale visualization, massive particle rendering, CAD/CAE/CFD visualization, semiconductor 3D structure visualization에서는 draw 대상이 매우 많고, 매 프레임 보이는 subset이 계속 변한다.

CPU-driven 구조에서는 객체 수가 증가할수록 `for object -> cull -> bind -> draw` 흐름이 병목이 된다. 반면 GPU-driven 구조에서는 객체 metadata, bounding volume, transform, material index, LOD 정보가 GPU buffer에 올라가 있고, compute shader가 병렬로 visible object list를 만든다. 그 결과 CPU는 scene 전체를 세밀하게 순회하기보다, 큰 단위의 dispatch와 indirect draw 실행만 담당한다.

이 개념은 Vulkan, DirectX 12, Metal, WebGPU 같은 modern graphics API와 잘 맞는다. 이 API들은 명시적 resource management, descriptor model, command buffer, indirect draw, compute pipeline을 제공하기 때문에 GPU-driven architecture를 설계하기 좋다.

## 4. 구현 관점

GPU-Driven Rendering을 구현 관점에서 보면 가장 중요한 데이터는 **object metadata buffer**다. CPU가 객체별로 draw call을 직접 호출하는 대신, 모든 객체의 렌더링 정보를 GPU buffer에 구조화해서 올려둔다.

예시적인 데이터 구성은 다음과 같다.

```cpp
struct ObjectData
{
    float4x4 worldMatrix;
    float4 boundingSphere;   // xyz: center, w: radius
    uint meshIndex;
    uint materialIndex;
    uint lodBaseIndex;
    uint lodCount;
};
```

Compute shader는 `ObjectData` 배열을 병렬로 읽고, camera frustum과 bounding volume을 비교한다. 보이는 객체는 visible list에 기록된다. 이때 단순 append buffer를 사용할 수도 있고, prefix sum / stream compaction을 사용할 수도 있다.

가장 단순한 흐름은 다음과 같다.

1. CPU가 camera, frustum plane, frame constants를 uniform/storage buffer에 기록한다.
2. GPU compute shader가 모든 object bounding volume을 검사한다.
3. visible object index를 append buffer 또는 compacted buffer에 저장한다.
4. compute shader가 indirect draw command buffer를 갱신한다.
5. graphics pipeline이 indirect draw buffer를 사용해 렌더링한다.

여기서 핵심 난점은 **동기화와 메모리 레이아웃**이다. compute shader가 작성한 indirect command buffer를 graphics draw stage가 읽어야 하므로, Vulkan 기준으로는 storage write 이후 indirect command read에 대한 pipeline barrier가 필요하다. WebGPU에서도 compute pass 이후 render pass에서 해당 buffer를 읽는 사용 패턴을 명확히 구성해야 한다.

메모리 레이아웃 관점에서는 AoS(Array of Structures)와 SoA(Structure of Arrays) 선택이 중요하다. culling shader가 bounding sphere만 대량으로 읽는다면 transform/material까지 함께 들어 있는 AoS는 cache 효율이 떨어질 수 있다. 반대로 draw stage에서 object payload를 한 번에 접근해야 한다면 AoS가 단순하다. 대규모 객체에서는 다음처럼 분리하는 방식이 자주 유리하다.

- `BoundsBuffer`: culling 전용 bounding volume
- `TransformBuffer`: visible object의 transform 접근
- `MaterialBuffer`: material parameter index
- `MeshBuffer`: vertex/index buffer offset, draw range
- `DrawCommandBuffer`: indirect draw command 저장

## 5. 내 관심 분야와 연결

이 개념은 사용자의 관심 분야인 **CFD / scientific visualization / semiconductor 3D visualization / sparse voxel / octree / WebGPU**와 강하게 연결된다.

CFD 후처리 viewer에서는 수천~수백만 개의 cell, block, particle, streamline segment, glyph, voxel brick이 존재할 수 있다. 이때 CPU가 매 프레임 모든 요소를 순회하며 draw call을 구성하면 WebGL/Three.js 계층에서는 빠르게 한계에 도달한다. GPU-driven 방식으로 생각하면, CPU는 전체 데이터셋을 GPU-friendly buffer와 spatial structure로 업로드하고, GPU가 camera 기준으로 visible block, active voxel, selected LOD를 고르는 구조가 된다.

반도체 3D visualization에서도 유사하다. multi-layer thin structure, SDF/level-set field, sparse grid, brick-based volume, marching cubes 결과 mesh를 다룰 때 모든 primitive를 CPU에서 재구성하는 것은 비용이 크다. GPU-driven 사고방식에서는 다음 방향이 자연스럽다.

- sparse brick metadata를 GPU buffer로 관리
- camera/frustum 기준 visible brick만 선택
- screen-space error 기반 brick LOD 선택
- compute shader에서 active brick list 생성
- indirect draw 또는 indirect dispatch로 surface/volume rendering 연결

게임 엔진 관점에서는 Unreal Engine의 Nanite 같은 virtualized geometry 시스템도 넓은 의미에서 GPU-driven philosophy와 연결된다. 핵심은 CPU가 개별 object draw를 제어하는 것이 아니라, GPU가 cluster visibility와 rasterization workload를 세밀하게 관리한다는 점이다.

## 6. 머릿속에 남길 질문 3개

1. CPU가 draw call을 직접 제출하는 구조에서 병목이 생기는 지점은 object count, material state change, driver overhead 중 어디인가?
2. GPU culling 결과를 draw stage에서 사용하려면 어떤 buffer layout과 synchronization이 필요한가?
3. CFD block, voxel brick, mesh cluster, particle chunk를 모두 “GPU가 선택할 렌더링 단위”로 추상화할 수 있는가?

## 7. Graphics Engineer 면접 질문 1개와 답변

**Q. GPU-Driven Rendering이 CPU-Driven Rendering보다 유리한 상황은 언제이며, 구현 시 주의할 점은 무엇인가?**

A. 객체 수가 많고, 매 프레임 visibility가 크게 변하며, draw call submission overhead가 큰 장면에서 GPU-Driven Rendering이 유리하다. 예를 들어 open world, vegetation, particle system, massive CAD/CAE visualization, voxel/brick-based renderer에서는 CPU가 모든 객체를 순회하고 draw call을 발행하는 비용이 커진다. GPU-driven 구조에서는 object metadata를 GPU buffer에 저장하고 compute shader가 frustum culling, occlusion culling, LOD selection을 수행한 뒤 indirect draw command를 생성하거나 갱신한다.

주의할 점은 synchronization, buffer layout, atomic contention, debugging complexity다. compute pass가 작성한 visible list나 indirect command buffer를 graphics pass가 읽기 전에 적절한 barrier가 필요하다. 또한 culling에 필요한 bounds data와 draw에 필요한 material/mesh data를 같은 구조에 넣을지 분리할지에 따라 memory bandwidth 효율이 달라진다. append buffer를 사용할 경우 atomic 증가가 병목이 될 수 있으므로 prefix sum 기반 compaction이나 per-workgroup binning도 고려해야 한다.

## 8. 포트폴리오 / 커리어 연결

GPU-Driven Rendering은 그래픽스 엔지니어 포트폴리오에서 “modern rendering architecture를 이해한다”는 강한 신호가 된다. 단순히 OpenGL draw call을 많이 호출하는 수준이 아니라, Vulkan/WebGPU/DirectX 12 시대의 렌더링 병목을 어떻게 구조적으로 줄이는지 설명할 수 있기 때문이다.

포트폴리오 문장으로는 다음처럼 연결할 수 있다.

> Designed a GPU-driven visibility and rendering pipeline for large-scale scientific visualization, where object metadata, bounding volumes, LOD information, and indirect draw commands are managed through GPU buffers to reduce CPU submission overhead.

사용자의 CFD/VTK 후처리 viewer 경험과 연결하면, large VTK dataset을 octree/brick 단위로 나누고 GPU에서 visible block list를 구성하는 방향으로 설명할 수 있다. 이는 game engine company, visualization company, semiconductor software company 모두에서 설득력 있는 주제다.

## 9. 내일 이어서 볼 개념

**Indirect Draw / Multi-Draw Indirect**

GPU-Driven Rendering의 실제 실행 단계는 indirect draw command buffer에 달려 있다. 내일은 `DrawIndirect`, `MultiDrawIndirect`, `ExecuteIndirect`, WebGPU의 indirect draw 개념을 중심으로, draw command가 메모리에서 어떤 형태로 표현되고 GPU가 이를 어떻게 소비하는지 이어서 보면 좋다.

## 10. 참고 키워드

- GPU-Driven Rendering
- Compute Shader Culling
- Frustum Culling
- Occlusion Culling
- Indirect Draw
- Multi-Draw Indirect
- ExecuteIndirect
- Draw Command Buffer
- Visible List Compaction
- Prefix Sum
- Append Buffer
- Bindless Rendering
- Descriptor Indexing
- Meshlet
- Cluster Culling
- Sparse Voxel Rendering
- Brick-based Volume Rendering
- WebGPU Indirect Draw
- Vulkan Pipeline Barrier
- GPU Memory Layout
