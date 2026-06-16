---
title: "Forward+ / Clustered Shading"
date: "2026-06-16"
category: "Graphics"
tags: ["GPU", "Rendering Pipeline", "Forward+", "Clustered Shading", "Light Culling", "Compute Shader", "Depth Buffer", "Tiled Shading", "Real-time Rendering"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-06-16 - Forward+ / Clustered Shading

## 1. 오늘의 개념

**Forward+**와 **Clustered Shading**은 많은 dynamic light가 있는 장면에서 모든 fragment가 모든 light를 검사하지 않도록, 화면 또는 view frustum을 작은 영역으로 나누고 각 영역에 영향을 주는 light list만 사용해 shading하는 기법이다.

Forward+는 화면을 2D tile로 나눈다. 예를 들어 16x16 pixel tile마다 compute shader가 해당 tile에 영향을 주는 point light / spot light index list를 만든다. Fragment shader는 자신이 속한 tile의 light list만 순회한다.

Clustered Shading은 여기에 depth slice를 추가한다. 즉 screen tile을 X/Y로 나누고, view-space depth를 Z 방향으로 나누어 3D cluster를 만든다. 이 방식은 같은 화면 tile 안에서도 가까운 물체와 먼 배경이 서로 다른 light list를 사용할 수 있게 만든다.

핵심 변화는 다음이다.

> 모든 pixel이 모든 light를 보는 구조에서, 각 tile/cluster가 자신에게 영향을 주는 light subset만 보는 구조로 바뀐다.

## 2. 한 줄 핵심

**Forward+ / Clustered Shading은 depth buffer와 compute shader를 이용해 screen tile 또는 3D cluster별 light list를 만들고, fragment shader가 필요한 light만 평가하게 만드는 many-light rendering 전략이다.**

## 3. 왜 중요한가

Modern real-time renderer에서는 point light, spot light, emissive proxy, decal light, probe, volumetric light처럼 조명과 조명 유사 데이터가 많아진다. 단순 forward renderer에서 모든 fragment가 전체 light 배열을 순회하면 light 수가 늘어날수록 shading cost가 급격히 커진다.

Forward+와 Clustered Shading은 forward renderer의 장점인 MSAA, transparency 친화성, material 다양성은 유지하면서 many-light 문제를 GPU compute 기반 data filtering 문제로 바꾼다.

이 개념은 depth pre-pass, compute shader, storage buffer, light list compaction, GPU-driven rendering과 연결된다. 또한 light list 대신 volume brick list나 field block list를 구성하는 방식으로 scientific visualization에도 응용할 수 있다.

## 4. 구현 관점

### 4.1 Traditional Forward Rendering의 한계

전통적인 forward shader는 다음처럼 전체 light를 순회하기 쉽다.

```glsl
vec3 color = vec3(0.0);
for (int i = 0; i < lightCount; ++i)
{
    color += EvaluateLight(lights[i], material, normal, position);
}
```

실제로는 대부분의 light가 현재 fragment에 영향을 주지 않는다. 문제는 lighting 계산보다 관련 없는 light까지 검사하는 구조다.

### 4.2 Forward+ 기본 흐름

Forward+의 일반적인 흐름은 다음과 같다.

1. Depth pre-pass 또는 depth buffer를 준비한다.
2. 화면을 16x16 또는 32x32 tile로 나눈다.
3. 각 tile의 depth min/max를 계산한다.
4. Compute shader에서 light bounding volume과 tile frustum 교차를 검사한다.
5. Tile별 light index list를 만든다.
6. Forward shading pass에서 해당 tile의 light list만 순회한다.

대표적인 buffer 구조는 다음과 같다.

```cpp
struct TileLightRange
{
    uint offset;
    uint count;
};

StructuredBuffer<Light> lights;
StructuredBuffer<uint> tileLightIndices;
StructuredBuffer<TileLightRange> tileRanges;
```

Fragment shader는 screen coordinate로 tile index를 구하고, `offset/count` 범위의 light만 평가한다.

### 4.3 Clustered Shading의 확장

Forward+는 2D tile 기반이라 depth 방향으로 긴 영역에서는 light list가 과하게 커질 수 있다. Clustered Shading은 depth slice를 추가해 view frustum을 3D grid로 나눈다.

- X/Y: screen tile
- Z: view-space depth slice

Clustered Shading은 가까운 물체와 먼 배경을 같은 screen tile 안에서도 분리할 수 있어 point light / spot light가 많은 장면에서 더 정밀한 culling이 가능하다.

Depth slice는 linear하게 나눌 수도 있지만, 보통 near plane 근처에 geometry가 많기 때문에 logarithmic slice가 자주 사용된다. 이때 reversed-Z, linear depth 복원, cluster index 계산 비용을 함께 고려해야 한다.

### 4.4 Conservative light culling

Light culling은 false positive는 허용되지만 false negative는 피해야 한다. 실제로 영향이 없는 light가 list에 들어가면 성능만 조금 손해지만, 실제로 영향이 있는 light가 빠지면 조명이 사라진다.

따라서 point light sphere, spot light cone, tile frustum / cluster bounds 교차 검사는 conservative하게 설계한다. 실무에서는 light radius, depth range, near/far plane, projection precision 때문에 약간의 bias를 두는 경우가 많다.

### 4.5 Deferred Shading과 비교

Deferred Shading도 many-light 처리에 강하지만 G-buffer bandwidth가 크고, MSAA / transparency / 다양한 material model 처리에 불리할 수 있다.

Forward+ / Clustered Shading은 forward renderer 기반이므로 다음 장점이 있다.

- MSAA와 잘 맞는다.
- transparency 처리에 상대적으로 유리하다.
- material 다양성을 유지하기 쉽다.
- G-buffer bandwidth를 줄일 수 있다.

대신 compute pass로 light list를 만들어야 하므로 atomic append, prefix sum, list overflow, buffer layout, synchronization 같은 구현 복잡도가 생긴다.

## 5. 내 관심 분야와 연결

### CFD / scientific visualization

CFD visualization에서는 게임처럼 많은 point light가 핵심인 경우는 적다. 하지만 Forward+ / Clustered Shading의 본질은 “light”가 아니라 **screen-space 또는 view-space 영역별로 필요한 data subset만 고르는 구조**다.

이 사고는 다음으로 확장될 수 있다.

- tile별 visible field block list
- cluster별 volume brick list
- slice / iso-surface 주변 local highlight
- streamline depth-aware shading
- scalar field annotation probe

즉 light list 대신 field block list나 volume brick list를 만들면, clustered visibility / shading 구조로 확장할 수 있다.

### Sparse voxel / octree / NanoVDB

Sparse voxel renderer에서는 cluster 개념을 voxel brick selection과 연결할 수 있다. View frustum을 screen tile과 depth slice로 나누고, 각 cluster에 영향을 주는 brick 또는 node를 매핑하면 ray marching이나 shading 대상 범위를 줄일 수 있다.

반도체 3D visualization에서도 layer별 material region, SDF brick, density brick, transfer function index를 cluster별 active resource list로 관리하는 사고가 가능하다.

결국 Forward+의 본질은 many-light renderer가 아니라, **screen/view-space binning을 통한 GPU data filtering**이다.

### Game engine architecture

Game engine에서는 Forward+ / Clustered Shading이 modern forward renderer의 핵심이다. 많은 dynamic light를 처리하면서도 MSAA, transparency, material flexibility를 유지할 수 있기 때문이다.

Unity, Unreal, custom engine renderer를 이해할 때 “forward vs deferred”를 넘어서 “forward에서 many-light를 어떻게 처리할 것인가”를 설명할 수 있으면 강한 신호가 된다.

## 6. 머릿속에 남길 질문 3개

1. Forward+에서 모든 fragment가 전체 light를 순회하지 않고 tile light list만 순회할 수 있는 이유는 무엇인가?
2. Clustered Shading이 2D tile 기반 Forward+보다 depth 방향에서 더 정밀한 light culling을 할 수 있는 이유는 무엇인가?
3. Sparse voxel renderer에서 clustered light list 사고를 volume brick list 또는 field block list로 바꾸면 어떤 구조가 될 수 있는가?

## 7. graphics engineer 면접 질문 1개와 답변

### Q. Forward+와 Clustered Shading은 어떤 문제를 해결하며, Deferred Shading과 비교했을 때 어떤 장단점이 있나요?

**A.** Forward+와 Clustered Shading은 많은 light가 있는 장면에서 모든 fragment가 모든 light를 평가하는 비효율을 줄이기 위한 기법입니다. Forward+는 화면을 2D tile로 나누고, 각 tile에 영향을 주는 light list를 compute shader로 만든 뒤 fragment shader가 해당 tile의 light만 평가합니다. Clustered Shading은 여기에 depth slice를 추가해 view frustum을 3D cluster로 나누므로, depth 방향에서도 더 정밀한 light culling이 가능합니다.

Deferred Shading도 many-light 처리에 강하지만 G-buffer bandwidth가 크고, MSAA와 transparency, 다양한 material model 처리에 불리할 수 있습니다. Forward+ / Clustered Shading은 forward renderer의 장점인 MSAA, transparency 친화성, material 다양성을 유지하면서 light culling을 GPU에서 수행할 수 있다는 장점이 있습니다. 단점은 compute pass로 light list를 생성해야 하므로 buffer layout, atomic contention, list overflow, cluster indexing 같은 구현 복잡도가 증가한다는 점입니다.

## 8. 포트폴리오 / 커리어 연결

Forward+ / Clustered Shading은 포트폴리오에서 다음 메시지를 만든다.

> “나는 forward/deferred의 차이를 아는 수준을 넘어, depth buffer와 compute shader를 이용해 many-light 문제를 GPU data structure로 해결하는 renderer architecture를 이해한다.”

면접에서는 다음처럼 말할 수 있다.

> “Forward+는 screen tile별 light list를 만들어 forward renderer의 many-light 비용을 줄이고, Clustered Shading은 depth slice를 추가해 3D cluster 단위로 light culling을 더 정밀하게 수행합니다. 이 구조는 light뿐 아니라 voxel brick이나 field block 같은 large visualization data를 screen-space binning하는 사고로도 확장할 수 있습니다.”

## 9. 내일 이어서 볼 개념

**Tile-Based Deferred Shading**

Forward+ / Clustered Shading을 이해한 다음에는 deferred renderer에서 tile 단위로 light를 처리하는 Tile-Based Deferred Shading을 보는 것이 자연스럽다. 같은 screen-space light culling을 사용하지만, shading이 G-buffer 이후에 일어난다는 점에서 차이가 있다.

## 10. 참고 키워드

- Forward+
- Clustered Shading
- Tiled Shading
- Tile-Based Lighting
- Light Culling
- Compute Shader
- Depth Buffer
- Depth Range
- Cluster Grid
- Light List Buffer
- Atomic Counter
- Prefix Sum
- Deferred Shading
- Forward Rendering
- Many-Light Rendering
- MSAA
- Transparency
- Storage Buffer
- WebGPU Compute Pass
- Vulkan Descriptor Set
- DirectX 12 Structured Buffer
