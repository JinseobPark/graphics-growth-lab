---
title: "Screen-Space Error Based LOD"
date: "2026-06-25"
category: "Graphics"
tags: ["GPU", "Rendering Pipeline", "LOD", "Screen-Space Error", "Meshlet", "Voxel Brick", "Sparse Voxel", "GPU-Driven Rendering", "Scientific Visualization"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-06-25 - Screen-Space Error Based LOD

## 1. 오늘의 개념

**Screen-Space Error Based LOD**는 meshlet, terrain tile, voxel brick, surface patch를 어떤 해상도로 렌더링할지 결정할 때, 단순 거리 기준이 아니라 **현재 화면에서 그 차이가 몇 pixel로 보이는가**를 기준으로 LOD를 선택하는 방식이다.

단순 distance-based LOD는 다음처럼 동작한다.

```text
가까움: LOD0
중간 거리: LOD1
먼 거리: LOD2
```

하지만 같은 거리라도 object 크기, FOV, 화면 해상도, projection scale에 따라 실제로 보이는 차이는 달라진다. Screen-space error 기반 LOD는 world-space error를 pixel 단위 오차로 바꿔 판단한다.

핵심 질문은 다음이다.

> 이 낮은 LOD를 사용했을 때 생기는 오차가 화면에서 눈에 띄는가?

## 2. 한 줄 핵심

**Screen-Space Error Based LOD는 world-space simplification error를 camera projection을 통해 pixel 단위 오차로 변환하고, 화면에서 의미 있는 차이가 보일 때만 높은 LOD를 선택하는 방식이다.**

## 3. 왜 중요한가

GPU-driven renderer, meshlet renderer, sparse voxel renderer, scientific visualization에서는 렌더링 대상이 너무 많다. 모든 cluster, brick, patch를 최고 해상도로 그리면 성능을 유지하기 어렵다.

거리 기반 LOD만 사용하면 다음 문제가 생긴다.

- 큰 object는 멀어도 화면에서 크게 보일 수 있다.
- 작은 object는 가까워도 낮은 LOD로 충분할 수 있다.
- FOV와 해상도에 따라 같은 거리의 시각적 오차가 달라진다.
- voxel brick, terrain tile, scientific block처럼 크기가 다른 데이터를 일관되게 처리하기 어렵다.

Screen-space error는 LOD 판단 기준을 world-space 거리에서 final image contribution으로 옮긴다. 즉 GPU 비용을 화면에서 실제로 중요한 영역에 배분하는 방식이다.

## 4. 구현 관점

### 4.1 기본 계산

각 LOD나 cluster는 world-space error 값을 가진다. 예를 들어 simplified mesh와 원본 mesh의 최대 편차, voxel cell size, terrain height error, SDF sampling resolution이 이에 해당한다.

개념식은 다음과 같다.

```text
screenErrorPixels ≈ worldError * projectionScale / distanceToCamera
```

이 값이 threshold보다 작으면 낮은 LOD를 사용하고, threshold보다 크면 더 높은 LOD를 선택한다.

```text
screenError < 1 px  → 낮은 LOD 가능
screenError > 2 px  → 높은 LOD 필요
```

Projection scale은 viewport height와 projection matrix에서 얻을 수 있다.

```cpp
float projectionScale = viewportHeight * projectionMatrix[1][1] * 0.5f;
float screenError = worldError * projectionScale / distance;
```

여기서 중요한 점은 LOD가 단순 distance가 아니라 FOV, viewport resolution, object error를 모두 포함한다는 것이다.

### 4.2 Conservative distance

Object center까지의 거리만 쓰면 큰 object나 큰 voxel brick에서 LOD가 너무 낮게 선택될 수 있다. 그래서 bounding sphere radius를 고려한다.

```cpp
float dist = length(cameraPos - bounds.center) - bounds.radius;
dist = max(dist, nearDistanceEpsilon);
```

이렇게 하면 camera에 가까운 surface를 더 보수적으로 처리할 수 있다.

### 4.3 Meshlet LOD

Meshlet renderer에서는 cluster hierarchy traversal에 screen-space error를 사용할 수 있다.

1. Meshlet cluster가 frustum 안에 있는지 확인한다.
2. Hi-Z로 가려졌는지 확인한다.
3. Screen-space error가 threshold 이하인지 판단한다.
4. 충분히 작으면 현재 LOD cluster를 사용한다.
5. 너무 크면 child cluster로 내려간다.

이 구조는 virtual geometry renderer와 연결된다. 중요한 점은 LOD 선택이 object distance가 아니라 screen contribution에 의해 결정된다는 것이다.

### 4.4 Voxel brick / sparse volume LOD

Sparse voxel이나 octree에서는 voxel size 또는 brick resolution이 world-space error 역할을 한다.

```text
voxelScreenSize ≈ voxelSize * projectionScale / distance
```

Projected voxel size가 0.5 pixel보다 작으면 더 높은 resolution을 사용할 필요가 적다. 반대로 몇 pixel 이상으로 커지면 blocky artifact가 보일 수 있으므로 더 높은 LOD가 필요하다.

이 방식은 sparse voxel rendering, SDF ray marching, marching cubes surface extraction에 모두 연결된다.

### 4.5 Scientific visualization에서의 확장

Scientific visualization에서는 geometry error만 보면 부족하다. Field value의 변화도 함께 고려해야 한다.

추가로 볼 수 있는 error는 다음과 같다.

- scalar field gradient
- vector direction error
- iso-surface position error
- material boundary error
- volume density variation

예를 들어 scalar gradient가 큰 영역은 멀리 있어도 낮은 LOD를 쓰면 iso-surface 위치가 흔들릴 수 있다. 반대로 변화가 완만한 영역은 낮은 LOD로도 충분할 수 있다.

### 4.6 LOD popping 방지

Threshold 근처에서 LOD가 계속 바뀌면 popping이 생긴다. 이를 줄이려면 hysteresis를 사용한다.

```text
LOD 낮춤: screenError < 0.8 px
LOD 높임: screenError > 1.2 px
```

또는 temporal smoothing, geomorphing, dithered transition을 사용할 수 있다. GPU-driven renderer에서는 LOD 변화가 visible list와 indirect command 수에도 영향을 주므로, visual popping뿐 아니라 frame-to-frame workload fluctuation도 관리해야 한다.

## 5. 내 관심 분야와 연결

### CFD / scientific visualization

CFD visualization에서는 LOD를 단순 거리로 정하면 위험하다. 멀리 있어도 vortex, interface, boundary layer, high-gradient region은 높은 detail이 필요할 수 있다.

좋은 방향은 다음이다.

```text
screen-space geometry error + field gradient error
```

즉 화면에서 보이는 크기와 데이터 변화량을 함께 보고 LOD를 선택해야 한다.

### Sparse voxel / octree / NanoVDB

Sparse voxel / octree에서는 screen-space error가 node traversal 기준이 된다.

- projected voxel size가 작으면 coarse node 사용
- projected voxel size가 크면 child node로 내려감
- Hi-Z로 가려진 node는 traversal 중단
- SDF error가 큰 영역은 higher resolution 사용

이 구조는 sparse representation을 real-time renderer로 연결하는 핵심이다.

### Game engine architecture

Game engine에서는 screen-space error가 terrain LOD, meshlet LOD, virtual geometry, texture streaming, shadow LOD와 연결된다.

Nanite-style renderer를 이해하려면 distance LOD보다 screen-space error와 cluster hierarchy를 먼저 이해해야 한다.

## 6. 머릿속에 남길 질문 3개

1. Distance-based LOD가 큰 object나 다양한 크기의 voxel brick에서 불안정한 이유는 무엇인가?
2. World-space geometric error를 screen-space pixel error로 바꿀 때 camera projection과 viewport height가 필요한 이유는 무엇인가?
3. CFD visualization에서 screen-space error에 field gradient error를 함께 고려해야 하는 이유는 무엇인가?

## 7. graphics engineer 면접 질문 1개와 답변

### Q. Screen-Space Error Based LOD란 무엇이며, 단순 distance-based LOD보다 어떤 점이 더 좋은가요?

**A.** Screen-Space Error Based LOD는 LOD를 선택할 때 카메라와 object 사이의 거리만 보는 것이 아니라, 낮은 LOD를 사용했을 때 발생하는 world-space error가 현재 화면에서 몇 pixel 정도로 보이는지 계산하는 방식입니다. 같은 거리라도 object 크기, viewport resolution, FOV, projection scale에 따라 화면에서 보이는 오차가 달라지기 때문에 distance-based LOD보다 일관된 quality/performance trade-off를 만들 수 있습니다.

예를 들어 meshlet, terrain tile, voxel brick은 각자 world-space error 또는 cell size를 가질 수 있습니다. 이를 `screenError ≈ worldError * projectionScale / distance` 형태로 pixel error로 변환하고, threshold보다 작으면 낮은 LOD를 사용합니다. 장점은 화면에 거의 차이가 보이지 않는 곳에는 GPU 비용을 줄이고, 크게 보이는 영역에는 detail을 유지할 수 있다는 점입니다. 단점은 error metric, bounding volume distance, LOD popping, hysteresis, field-aware error를 신중히 설계해야 한다는 점입니다.

## 8. 포트폴리오 / 커리어 연결

Screen-Space Error Based LOD는 포트폴리오에서 다음 메시지를 만든다.

> “나는 LOD를 단순 거리 튜닝이 아니라, projection과 pixel error 기반의 visual importance metric으로 설계할 수 있다.”

네 배경과 연결하면 다음 표현이 좋다.

- CFD / VTK visualization에서 block 또는 mesh의 projected size 기반 LOD 설계
- Sparse voxel / octree renderer에서 projected voxel size로 node traversal 결정
- Marching Cubes surface patch를 screen-space error 기준으로 생성하거나 생략하는 사고
- WebGPU / Vulkan GPU-driven renderer에서 compute culling pass 안에 LOD selection을 포함하는 구조

## 9. 내일 이어서 볼 개념

**Clipmap and Cascaded LOD Structures**

Screen-space error 기반 LOD 다음에는 clipmap과 cascaded LOD 구조를 보는 것이 자연스럽다. Terrain, volume, sparse voxel, large-scale visualization에서는 카메라 주변은 고해상도, 멀리 있는 영역은 낮은 해상도로 유지하는 계층 구조가 중요하다.

## 10. 참고 키워드

- Screen-Space Error
- LOD Selection
- Meshlet LOD
- Voxel Brick LOD
- Sparse Voxel Octree
- Projected Size
- Projection Scale
- Viewport Height
- FOV
- Geometric Error
- Field Error
- Scalar Gradient
- Hysteresis
- LOD Popping
- GPU-driven Rendering
- Cluster Hierarchy
- Virtual Geometry
- Terrain LOD
- Scientific Visualization
