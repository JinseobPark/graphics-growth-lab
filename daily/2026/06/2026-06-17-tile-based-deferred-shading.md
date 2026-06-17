---
title: "Tile-Based Deferred Shading"
date: "2026-06-17"
category: "Graphics"
tags: ["GPU", "Rendering Pipeline", "Deferred Shading", "Tile-Based Deferred", "Light Culling", "Compute Shader", "G-buffer", "Tiled Lighting", "Real-time Rendering"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-06-17 - Tile-Based Deferred Shading

## 1. 오늘의 개념

**Tile-Based Deferred Shading**은 deferred renderer에서 G-buffer를 만든 뒤, 화면을 작은 tile로 나누고 각 tile에 영향을 주는 light list만 이용해 lighting을 계산하는 기법이다.

일반적인 deferred shading에서는 geometry pass에서 다음 정보를 G-buffer에 저장한다.

- depth
- normal
- albedo
- roughness / metallic
- material id
- motion vector 또는 extra attribute

그 다음 lighting pass에서 screen-space pixel마다 G-buffer를 읽고 조명을 계산한다. 문제는 light 수가 많아질 때 모든 pixel이 모든 light를 순회하면 비용이 너무 커진다는 점이다.

Tile-Based Deferred Shading은 이 문제를 다음 방식으로 해결한다.

1. G-buffer와 depth buffer를 만든다.
2. 화면을 16x16 또는 32x32 tile로 나눈다.
3. 각 tile의 depth min/max 또는 tile frustum을 계산한다.
4. Compute shader가 light volume과 tile의 교차를 검사한다.
5. Tile별 light index list를 만든다.
6. Deferred lighting pass에서 pixel이 속한 tile의 light만 평가한다.

즉 핵심은 다음이다.

> Deferred shading의 screen-space lighting을 tile 단위 light culling과 결합해, 많은 light를 효율적으로 처리한다.

## 2. 한 줄 핵심

**Tile-Based Deferred Shading은 G-buffer 이후 lighting pass에서 tile별 light list를 사용해, deferred renderer의 many-light 비용을 줄이는 screen-space light culling 전략이다.**

## 3. 왜 중요한가

Deferred shading은 원래 many-light에 강한 구조로 알려져 있다. Geometry를 먼저 G-buffer에 저장하고, lighting은 screen-space에서 수행하기 때문에 light 계산이 object 수가 아니라 pixel 수에 가까워진다.

하지만 light 수가 많아지면 deferred lighting도 결국 비싸진다. 모든 pixel이 전체 light array를 순회하면 화면 해상도와 light 수의 곱만큼 비용이 증가한다.

Tile-Based Deferred Shading은 이 문제를 줄인다.

- tile별로 관련 light만 평가한다.
- CPU가 object-light assignment를 하지 않아도 된다.
- G-buffer depth를 light culling에 재사용한다.
- light volume draw 방식보다 compute 기반으로 제어하기 쉽다.
- 많은 point light / spot light를 처리하는 game renderer에 적합하다.

Graphics engineer 관점에서는 이 기법이 **deferred renderer의 lighting pass를 GPU data filtering 문제로 바꾸는 구조**라는 점이 중요하다.

## 4. 구현 관점

### 4.1 기본 deferred renderer 구조

Deferred renderer는 보통 두 단계로 구성된다.

```cpp
// Pass 1: Geometry pass
RenderSceneToGBuffer();

// Pass 2: Lighting pass
DispatchOrDrawFullscreenLighting();
```

Geometry pass에서는 material 정보를 여러 render target에 저장한다. Lighting pass에서는 각 pixel의 G-buffer 값을 읽고 조명을 계산한다.

단순 lighting shader는 다음처럼 보일 수 있다.

```glsl
vec3 P = ReconstructWorldPosition(uv, depth);
vec3 N = DecodeNormal(gNormal);
vec3 color = vec3(0.0);

for (int i = 0; i < lightCount; ++i)
{
    color += EvaluateLight(lights[i], P, N, material);
}
```

이 방식은 이해하기 쉽지만 light가 많아질수록 비효율적이다. 대부분의 light는 현재 pixel에 영향을 주지 않는다.

### 4.2 Tile light list 생성

Tile-Based Deferred Shading에서는 lighting pass 전에 compute shader로 tile별 light list를 만든다.

대표적인 buffer 구조는 다음과 같다.

```cpp
struct LightData
{
    vec4 positionRadius;
    vec4 colorIntensity;
    vec4 directionAngle;
};

struct TileLightRange
{
    uint offset;
    uint count;
};
```

전체 light index는 하나의 global light index buffer에 저장하고, 각 tile은 `offset/count`로 자신이 사용할 light 범위를 참조한다.

이 구조는 Forward+와 매우 비슷하다. 차이는 Forward+는 final forward shading pass에서 tile list를 사용하고, Tile-Based Deferred는 G-buffer 이후 screen-space lighting pass에서 tile list를 사용한다는 점이다.

### 4.3 Tile bounds와 depth range

각 tile은 단순한 2D 사각형이 아니라, depth range를 가진 view-space frustum 조각으로 볼 수 있다. G-buffer depth를 읽어 tile 안의 min depth와 max depth를 구하면, tile에 대한 conservative view-space bounds를 만들 수 있다.

Light culling은 대략 다음 문제다.

> Point light sphere 또는 spot light cone이 이 tile frustum과 겹치는가?

겹친다면 해당 light index를 tile list에 추가한다. 겹치지 않으면 lighting shader에서 평가하지 않는다.

이때 false positive는 허용된다. 실제로 영향을 주지 않는 light가 list에 들어가면 성능만 손해다. 하지만 false negative는 조명이 사라지는 visual bug이므로 피해야 한다.

### 4.4 Compute shader lighting 방식

Tile-Based Deferred Shading은 두 가지 방식으로 구현할 수 있다.

#### 1. Light list만 compute로 만들고 lighting은 fullscreen pixel shader에서 수행

이 방식은 render pass 구조가 익숙하고, 기존 deferred lighting shader를 확장하기 쉽다.

#### 2. Compute shader에서 tile 단위 lighting까지 수행

Compute shader가 tile별 workgroup을 담당하고, group shared memory에 tile light list를 올린 뒤 각 pixel lighting을 계산할 수 있다.

장점은 다음과 같다.

- tile 단위 shared memory 활용 가능
- thread group 단위 제어가 쉬움
- async compute와 결합 가능
- light list와 shading을 같은 compute pipeline 안에서 다루기 쉬움

단점은 render target write, blending, MSAA, post-process pipeline과의 연결을 더 신중하게 설계해야 한다는 점이다.

### 4.5 Light volume rendering과의 비교

전통적인 deferred renderer에서는 light마다 sphere/cone mesh를 그려 light volume 내부 pixel만 shading하는 방식도 사용한다.

Light volume 방식의 장점:

- 구현 개념이 직관적이다.
- 작은 light는 작은 screen area만 shading한다.
- stencil optimization과 결합할 수 있다.

단점:

- light 수가 많으면 draw call이 많아진다.
- overlapping light volume에서 overdraw가 커질 수 있다.
- CPU submission과 state change가 늘어날 수 있다.

Tile-Based Deferred는 light volume draw를 줄이고, compute shader에서 light assignment를 처리한다. 많은 light를 하나의 GPU data pipeline으로 다루기 좋다.

### 4.6 Forward+와의 차이

Forward+와 Tile-Based Deferred는 모두 tile light list를 사용한다. 하지만 shading이 일어나는 위치가 다르다.

- Forward+: geometry를 그리면서 material shading 중 light list를 사용
- Tile-Based Deferred: G-buffer를 만든 뒤 screen-space lighting에서 light list를 사용

Forward+는 transparency, MSAA, material 다양성에 유리하다. Tile-Based Deferred는 opaque lighting에서 G-buffer 기반으로 조명을 분리하기 좋고, light 수가 많은 장면에서 안정적이다.

중요한 점은 둘 다 같은 사고를 공유한다는 것이다.

> Screen tile 또는 cluster별로 필요한 light subset을 만들고, shader는 그 subset만 순회한다.

### 4.7 Bandwidth와 G-buffer 비용

Deferred renderer의 병목은 항상 ALU만이 아니다. G-buffer는 여러 render target을 쓰고 읽기 때문에 memory bandwidth가 큰 비용이 된다.

Tile-Based Deferred는 light loop 비용을 줄이지만, G-buffer bandwidth 자체를 없애지는 않는다. 따라서 성능을 판단할 때 다음을 함께 봐야 한다.

- G-buffer write bandwidth
- G-buffer read bandwidth
- depth reconstruction cost
- light list generation cost
- tile list memory access
- lighting ALU cost
- post-process와의 attachment reuse

즉 Tile-Based Deferred Shading은 many-light 문제에는 강하지만, G-buffer bandwidth 문제는 별도로 관리해야 한다.

## 5. 내 관심 분야와 연결

### CFD / scientific visualization

Scientific visualization에서는 game처럼 많은 point light가 주된 병목이 아닐 수도 있다. 하지만 Tile-Based Deferred의 사고방식은 매우 유용하다.

핵심은 “화면 tile별로 필요한 데이터 목록을 먼저 만들고, shading pass에서 그 목록만 읽는다”는 구조다.

이를 다음으로 확장할 수 있다.

- tile별 visible scalar field block list
- tile별 volume brick list
- iso-surface lighting과 volume compositing 분리
- screen-space annotation / probe list
- field value 기반 highlight region list

즉 light list 대신 field list, brick list, material region list를 쓰는 방식으로 대용량 visualization renderer에 적용할 수 있다.

### Sparse voxel / octree / NanoVDB

Sparse voxel renderer에서는 모든 brick을 모든 ray가 검사하면 비효율적이다. Tile-Based Deferred의 tile list 사고는 voxel brick binning으로 확장될 수 있다.

예를 들어 화면 tile마다 ray가 통과할 가능성이 높은 brick list를 만들고, ray marching pass에서 해당 tile의 brick list만 순회하게 할 수 있다. 이때 depth buffer는 ray start/end range를 제한하는 데도 사용할 수 있다.

반도체 3D visualization에서는 layer, material, SDF brick, density brick을 screen-space tile 또는 view-space cluster에 매핑하는 방식으로 연결할 수 있다.

### Game engine architecture

Game engine에서는 Tile-Based Deferred가 다음 구조와 연결된다.

- deferred lighting optimization
- tiled / clustered light culling
- compute shader lighting
- G-buffer bandwidth optimization
- light volume rendering 대체
- GPU-driven renderer의 screen-space binning

이 개념을 이해하면 Forward+, Clustered Shading, Deferred Shading이 서로 분리된 기술이 아니라 **light/data subset selection 문제의 다른 해법**이라는 점이 보인다.

## 6. 머릿속에 남길 질문 3개

1. Tile-Based Deferred Shading에서 G-buffer depth가 tile별 light culling에 어떻게 사용되는가?
2. Forward+와 Tile-Based Deferred는 둘 다 tile light list를 쓰지만, shading 위치가 어떻게 다른가?
3. Light list 사고를 sparse voxel brick list나 CFD field block list로 바꾸면 어떤 visualization pipeline을 만들 수 있는가?

## 7. graphics engineer 면접 질문 1개와 답변

### Q. Tile-Based Deferred Shading은 무엇이며, Forward+와 어떤 차이가 있나요?

**A.** Tile-Based Deferred Shading은 deferred renderer에서 G-buffer를 생성한 뒤 화면을 tile로 나누고, 각 tile에 영향을 주는 light list를 compute shader로 만든 다음 lighting pass에서 해당 tile의 light만 평가하는 방식입니다. 모든 pixel이 모든 light를 순회하지 않도록 해 many-light 비용을 줄입니다.

Forward+와 공통점은 tile별 light list를 만든다는 점입니다. 차이는 shading이 일어나는 위치입니다. Forward+는 geometry를 forward로 렌더링하면서 fragment shader에서 tile light list를 사용합니다. Tile-Based Deferred는 먼저 G-buffer를 만든 뒤 screen-space deferred lighting pass에서 tile light list를 사용합니다. Forward+는 transparency, MSAA, material 다양성에 유리하고, Tile-Based Deferred는 opaque lighting을 G-buffer 기반으로 분리해 많은 light를 안정적으로 처리하기 좋습니다.

## 8. 포트폴리오 / 커리어 연결

Tile-Based Deferred Shading은 포트폴리오에서 다음 메시지를 만든다.

> “나는 deferred renderer의 G-buffer 구조뿐 아니라, lighting pass의 many-light 병목을 tile-based light culling과 GPU buffer layout으로 최적화하는 방식을 이해한다.”

네 배경과 연결하면 다음 표현이 좋다.

- OpenGL deferred renderer에서 G-buffer와 lighting pass를 분리한 경험
- Compute shader로 tile별 data subset을 구성하는 구조 이해
- CFD / VTK visualization에서 screen tile별 field block list로 확장 가능
- Sparse voxel / octree renderer에서 tile별 brick list로 ray marching 비용을 줄이는 사고

면접에서는 다음처럼 말할 수 있다.

> “Tile-Based Deferred Shading은 deferred lighting을 전체 light loop로 처리하지 않고, tile별 light list를 만들어 pixel shader 또는 compute shader가 필요한 light subset만 평가하게 하는 방식입니다. 이 구조는 light뿐 아니라 voxel brick이나 field block 같은 visualization data binning으로도 확장할 수 있습니다.”

## 9. 내일 이어서 볼 개념

**Visibility Buffer Rendering**

Tile-Based Deferred Shading 다음에는 G-buffer에 많은 material attribute를 저장하는 대신, triangle/object/material id와 depth 중심으로 visibility만 저장하고 shading을 나중에 수행하는 Visibility Buffer Rendering을 보는 것이 자연스럽다. 이는 deferred renderer의 bandwidth 문제를 줄이기 위한 modern rendering architecture와 연결된다.

## 10. 참고 키워드

- Tile-Based Deferred Shading
- Tiled Deferred Lighting
- Deferred Shading
- G-buffer
- Light Culling
- Compute Shader
- Tile Light List
- Light Volume Rendering
- Forward+
- Clustered Shading
- Depth Range
- Screen-space Lighting
- Many-Light Rendering
- Bandwidth Optimization
- Storage Buffer
- Atomic Counter
- Prefix Sum
- GPU Data Filtering
- Sparse Voxel Brick List
- Scientific Visualization
