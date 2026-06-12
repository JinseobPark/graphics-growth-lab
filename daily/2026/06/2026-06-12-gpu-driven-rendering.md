---
title: "GPU-Driven Rendering"
date: "2026-06-12"
category: "Graphics"
tags: ["GPU", "Rendering Pipeline", "GPU-Driven Rendering", "Indirect Draw", "Compute Culling", "Meshlet", "Bindless", "Vulkan", "DirectX12", "WebGPU"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-06-12 - GPU-Driven Rendering

## 1. 오늘의 개념

**GPU-Driven Rendering**은 CPU가 모든 object를 순회하며 draw call을 직접 제출하는 대신, GPU가 visibility 판단, culling, draw command 생성, material/resource 선택의 상당 부분을 담당하도록 만드는 렌더링 구조다.

전통적인 renderer에서는 CPU가 대략 다음 일을 한다.

- scene object 순회
- frustum culling
- occlusion culling 결과 확인
- material / pipeline sorting
- resource binding
- draw call 제출

GPU-driven renderer에서는 이 중 일부 또는 대부분을 compute shader와 GPU buffer 기반으로 옮긴다. CPU는 전체 scene data와 몇 개의 dispatch / indirect draw 명령만 제출하고, GPU가 보이는 object 또는 cluster를 선별해 draw command buffer를 만든다.

핵심 변화는 다음과 같다.

> CPU가 “무엇을 그릴지” 매 프레임 결정하는 구조에서, GPU가 “무엇이 보이고 무엇을 그릴지” 데이터 기반으로 결정하는 구조로 이동한다.

이 개념은 어제의 **Bindless Resources**와 강하게 연결된다. GPU가 draw command를 직접 만들려면 shader가 object, material, texture, buffer를 index 기반으로 접근할 수 있어야 하기 때문이다.

## 2. 한 줄 핵심

**GPU-Driven Rendering은 CPU draw submission 병목을 줄이기 위해 culling과 draw command 생성을 GPU로 옮기고, indirect draw와 bindless resource table을 결합하는 modern renderer architecture다.**

## 3. 왜 중요한가

현대 렌더러는 단순히 삼각형 몇 개를 그리는 문제가 아니다. 많은 object, meshlet, particle, voxel brick, point cloud, instance, material, LOD를 매 프레임 빠르게 선별해야 한다.

CPU-driven renderer에서는 object 수가 많아질수록 다음 비용이 커진다.

- CPU scene traversal 비용
- draw call submission 비용
- command buffer recording 비용
- resource binding / descriptor update 비용
- visibility 결과를 CPU가 관리하는 동기화 비용

GPU-driven renderer는 이러한 병목을 줄이기 위해 scene visibility 판단을 GPU에 맡긴다. 특히 다음 상황에서 중요하다.

- open world / large scene renderer
- foliage / crowd / particle instance가 많은 장면
- meshlet 또는 cluster 기반 renderer
- voxel / sparse volume / point cloud visualization
- CFD / scientific visualization의 large dataset
- WebGPU / Vulkan / DX12 기반 explicit renderer

Graphics engineer 관점에서 GPU-driven rendering은 단순 최적화 기법이 아니라, **scene representation, memory layout, command generation, resource binding을 하나의 데이터 파이프라인으로 설계하는 방식**이다.

## 4. 구현 관점

### 4.1 CPU-driven rendering 구조

전통적인 CPU-driven renderer는 다음과 같은 흐름을 가진다.

```cpp
for (Object& obj : scene.objects)
{
    if (!FrustumCullCPU(obj.bounds))
        continue;

    BindPipeline(obj.material.pipeline);
    BindMaterial(obj.material);
    BindMesh(obj.mesh);
    DrawIndexed(obj.mesh.indexCount);
}
```

이 구조는 명확하지만 object 수가 많아질수록 CPU가 병목이 된다. 특히 rendering API가 explicit해질수록 command buffer recording과 descriptor management도 CPU 설계의 큰 부분이 된다.

### 4.2 GPU-driven rendering 구조

GPU-driven 구조에서는 scene object 정보를 GPU buffer에 저장하고, compute shader가 visibility를 판단한다.

대표적인 buffer 구성은 다음과 같다.

```cpp
struct ObjectData
{
    mat4 world;
    vec4 boundingSphere;
    uint meshIndex;
    uint materialIndex;
    uint lodIndex;
    uint flags;
};

struct DrawCommand
{
    uint indexCount;
    uint instanceCount;
    uint firstIndex;
    int  vertexOffset;
    uint firstInstance;
};
```

흐름은 보통 다음과 같다.

1. CPU가 object / mesh / material buffer를 GPU에 준비한다.
2. Compute shader가 object bounds를 camera frustum과 비교한다.
3. 보이는 object만 append buffer 또는 compacted draw list에 기록한다.
4. Compute shader가 indirect draw command를 생성한다.
5. Graphics pass에서 `DrawIndexedIndirect` 또는 `MultiDrawIndirect`로 그린다.

이때 CPU는 개별 object를 하나씩 draw하지 않는다. CPU는 compute dispatch와 indirect draw 명령을 제출하고, 실제 draw 목록은 GPU buffer에 의해 결정된다.

### 4.3 Indirect Draw의 의미

**Indirect Draw**는 draw call의 인자를 CPU 함수 인자로 직접 넘기지 않고, GPU buffer에 저장된 command 구조체에서 읽어 실행하는 방식이다.

일반 draw는 다음과 같다.

```cpp
DrawIndexed(indexCount, instanceCount, firstIndex, vertexOffset, firstInstance);
```

Indirect draw는 다음과 같은 개념이다.

```cpp
DrawIndexedIndirect(commandBuffer, commandOffset);
```

즉 draw parameter 자체가 GPU memory에 존재한다. GPU-driven renderer에서는 compute shader가 이 commandBuffer를 생성하거나 갱신한다.

이 구조가 강력한 이유는 visibility result가 CPU로 돌아올 필요가 없기 때문이다. GPU가 culling하고, GPU가 command를 만들고, GPU가 그린다.

### 4.4 Compute culling

GPU-driven renderer의 첫 번째 핵심 pass는 보통 **compute culling**이다.

대표적인 culling 종류는 다음과 같다.

- Frustum culling
- Hi-Z occlusion culling
- Backface / cone culling
- LOD selection
- Meshlet / cluster culling
- Distance culling

Frustum culling은 object bounding sphere 또는 AABB를 camera frustum plane과 비교한다. Hi-Z occlusion culling은 depth pyramid를 사용해 object가 이미 그려진 geometry 뒤에 가려졌는지 판단한다.

중요한 점은 culling 결과를 CPU로 읽어오지 않는다는 것이다. CPU readback은 pipeline stall을 만들 수 있으므로, 결과는 GPU 내부 draw list로 유지된다.

### 4.5 Compaction과 append buffer

Culling 결과로 보이는 object만 draw list에 넣으려면 compaction이 필요하다.

일반적인 방식은 다음과 같다.

- atomic counter로 visible object count 증가
- append buffer에 visible object index 기록
- prefix sum / scan으로 compacted list 생성
- draw command count buffer 갱신

Atomic append는 구현이 단순하지만 많은 thread가 동시에 counter를 증가시키면 contention이 생길 수 있다. Prefix sum 기반 compaction은 더 복잡하지만 대량 데이터에서 더 안정적인 성능을 낼 수 있다.

GPU-driven renderer의 성능은 단순히 “GPU에서 culling한다”가 아니라, **visible list를 얼마나 효율적으로 compact하고 draw command로 변환하는가**에 달려 있다.

### 4.6 Bindless Resources와의 연결

GPU가 draw command를 직접 만들면 CPU가 object별로 material과 texture를 bind할 수 없다. 따라서 shader는 object index 또는 material index를 통해 필요한 resource를 찾아야 한다.

이때 필요한 구조가 어제 정리한 Bindless Resources다.

```cpp
ObjectData obj = objectBuffer[objectIndex];
MaterialData mat = materialBuffer[obj.materialIndex];
vec4 albedo = texture(textures[mat.albedoTex], uv);
```

GPU-driven rendering에서 bindless는 선택이 아니라 거의 필수에 가깝다. GPU가 object를 선택하는데 CPU가 resource binding을 object마다 해줘야 한다면 구조가 깨진다.

따라서 modern renderer의 큰 흐름은 다음처럼 연결된다.

> GPU scene buffer → compute culling → compacted visible list → indirect draw command → bindless material/resource access

### 4.7 Meshlet / cluster renderer와의 연결

GPU-driven rendering은 object 단위보다 더 작은 단위인 **meshlet** 또는 **cluster**와 결합할 때 강력해진다.

Meshlet은 mesh를 작은 triangle cluster로 나눈 단위다. 각 meshlet은 자체 bounding volume과 cone culling 정보를 가진다. GPU는 object 전체가 아니라 meshlet 단위로 visibility를 판단할 수 있다.

장점은 다음과 같다.

- object 내부에서도 보이지 않는 geometry 제거 가능
- LOD와 streaming 단위를 세밀하게 제어 가능
- mesh shader pipeline과 자연스럽게 연결
- large model / CAD / scientific mesh에 유리

CFD visualization이나 반도체 3D visualization에서도 유사한 사고가 가능하다. 전체 volume이나 mesh를 한 번에 그리는 대신 brick, tile, cluster, chunk 단위로 visibility와 LOD를 판단한다.

## 5. 내 관심 분야와 연결

### Sparse voxel / octree / NanoVDB

Sparse voxel renderer에서는 전체 voxel volume을 매번 순회하면 비효율적이다. GPU-driven 구조에서는 visible voxel brick 또는 octree node만 선별해 ray marching 또는 surface extraction pass로 넘길 수 있다.

예를 들어 다음 흐름이 가능하다.

1. Camera frustum 기준으로 octree node culling
2. Hi-Z 또는 screen-space error 기준으로 brick LOD 선택
3. Visible brick list compact
4. Brick별 material / density / SDF resource index 참조
5. Indirect dispatch 또는 indirect draw로 rendering

이 구조는 dense grid에서 sparse grid로 넘어갈 때 매우 중요하다. Sparse representation은 CPU 자료구조만으로 끝나지 않고, GPU가 빠르게 순회할 수 있는 memory layout과 visible list generation으로 이어져야 한다.

### CFD / scientific visualization

CFD 후처리에서는 streamline, slice, contour, volume rendering, particle trace가 모두 대량 field data를 다룬다. GPU-driven 사고를 적용하면 다음과 같은 구조를 만들 수 있다.

- field block 단위 visibility culling
- screen-space error 기반 LOD 선택
- visible block list 생성
- scalar/vector field resource index 참조
- indirect rendering 또는 compute dispatch

특히 1억 격자 이상의 데이터를 다룰 때 CPU가 모든 block을 순회하고 draw를 제출하는 방식은 한계가 있다. GPU-driven 구조는 large scientific data를 interactive하게 다루기 위한 핵심 설계가 될 수 있다.

### Game engine architecture

게임 엔진에서는 GPU-driven rendering이 다음 시스템과 연결된다.

- occlusion culling
- instance rendering
- Nanite-like cluster rendering
- virtual geometry
- mesh shader
- bindless material system
- async compute
- GPU particle rendering

즉 renderer는 더 이상 “CPU가 정렬한 draw call 목록”만 실행하는 구조가 아니다. GPU 내부에서 scene visibility와 draw list를 갱신하는 동적 시스템에 가깝다.

## 6. 머릿속에 남길 질문 3개

1. GPU-driven renderer에서 culling 결과를 CPU로 readback하지 않아야 하는 이유는 무엇인가?
2. Indirect draw command buffer를 compute shader가 생성할 때, visible object list와 material/resource table은 어떤 관계를 가져야 하는가?
3. Sparse voxel renderer에서 object 단위 culling보다 brick/node 단위 culling이 중요한 이유는 무엇인가?

## 7. graphics engineer 면접 질문 1개와 답변

### Q. GPU-Driven Rendering이란 무엇이며, CPU-driven rendering과 비교했을 때 어떤 장단점이 있나요?

**A.** GPU-Driven Rendering은 CPU가 모든 object를 순회하며 draw call을 제출하는 대신, GPU가 compute shader 등을 이용해 visibility culling, LOD selection, draw command generation을 수행하고 indirect draw로 렌더링하는 구조입니다. CPU-driven 방식에서는 CPU가 object별로 culling, material binding, draw submission을 처리하기 때문에 object 수가 많아질수록 CPU overhead가 커집니다. GPU-driven 방식은 이 작업을 GPU buffer와 compute pass로 옮겨 대량 object, instance, meshlet, voxel brick을 더 확장성 있게 처리할 수 있습니다.

장점은 CPU draw submission 병목을 줄이고, GPU 내부에서 culling 결과를 유지해 readback stall을 피할 수 있다는 점입니다. 또한 bindless resource table, indirect draw, meshlet renderer와 잘 결합됩니다. 단점은 구현 복잡도가 높고, buffer synchronization, atomic contention, compaction, descriptor indexing, debugging이 어려워진다는 점입니다. 또한 GPU workload가 증가하므로 culling 비용이 실제로 줄어드는 geometry 비용보다 커지지 않도록 scene 특성에 맞게 설계해야 합니다.

## 8. 포트폴리오 / 커리어 연결

GPU-Driven Rendering은 포트폴리오에서 매우 강한 신호를 만든다.

단순히 “OpenGL로 렌더러를 만들었다”보다 다음 메시지가 훨씬 강하다.

> “대량 object와 scientific visualization data를 처리하기 위해 GPU buffer 기반 scene representation, compute culling, indirect draw, bindless material access 구조를 이해하고 설계할 수 있다.”

특히 네 경험과 연결하면 다음 방향이 좋다.

- CFD 후처리의 block / voxel / field data를 GPU visible list로 관리
- Sparse voxel octree에서 visible brick list 생성
- WebGPU에서 indirect draw와 storage buffer 기반 scene table 설계
- Vulkan / DX12 학습 시 descriptor indexing과 indirect command 구조 연결
- 반도체 3D visualization에서 layer / brick / material resource를 index 기반으로 관리

면접에서 이 개념을 말할 때는 “GPU가 더 빠르다”가 아니라, 다음처럼 말하는 것이 좋다.

> “CPU submission overhead와 GPU visibility 판단 사이의 경계를 재설계하여, large scene을 object list가 아니라 GPU data pipeline으로 처리하는 방식입니다.”

이 표현은 graphics engineer가 단순 API 사용자가 아니라 renderer architecture를 이해하고 있다는 인상을 준다.

## 9. 내일 이어서 볼 개념

**Hi-Z Occlusion Culling**

GPU-driven rendering에서 frustum culling만으로는 부족하다. 실제 화면에서는 frustum 안에 있지만 다른 geometry 뒤에 가려진 object가 많다. 이를 GPU에서 빠르게 제거하기 위해 depth pyramid를 사용하는 Hi-Z occlusion culling을 이어서 보는 것이 자연스럽다.

## 10. 참고 키워드

- GPU-Driven Rendering
- Indirect Draw
- MultiDrawIndirect
- DrawIndexedIndirect
- Compute Culling
- Frustum Culling
- Hi-Z Occlusion Culling
- Depth Pyramid
- Visible List Compaction
- Append Buffer
- Atomic Counter
- Prefix Sum / Scan
- Meshlet
- Cluster Culling
- Bindless Resources
- Descriptor Indexing
- Vulkan Indirect Draw
- DirectX 12 ExecuteIndirect
- WebGPU Indirect Draw
- GPU Scene
- GPU Culling Pipeline
