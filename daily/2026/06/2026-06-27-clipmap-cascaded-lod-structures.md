---
title: "Clipmap and Cascaded LOD Structures"
date: "2026-06-27"
category: "Graphics"
tags: ["GPU", "Rendering Pipeline", "LOD", "Clipmap", "Cascaded LOD", "Sparse Voxel", "Volume Rendering", "Terrain", "Scientific Visualization"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-06-27 - Clipmap and Cascaded LOD Structures

## 1. 오늘의 개념

**Clipmap**과 **Cascaded LOD Structures**는 large-scale terrain, volume, sparse voxel, shadow, scientific visualization data를 렌더링할 때, 카메라 주변은 고해상도로 유지하고 멀리 있는 영역은 점점 낮은 해상도로 표현하는 계층적 LOD 구조다.

Screen-space error based LOD가 “이 object나 brick을 어떤 LOD로 그릴 것인가”를 결정하는 기준이라면, clipmap과 cascade는 **공간 자체를 여러 해상도 ring 또는 layer로 나누어 관리하는 구조**다.

핵심 질문은 다음이다.

> 카메라 주변의 제한된 GPU memory와 shading budget을 어디에 고해상도로 배치할 것인가?

Clipmap은 보통 카메라 주변을 중심으로 여러 resolution level을 유지한다. 가까운 level은 작은 cell size를 사용하고, 멀리 있는 level은 큰 cell size를 사용한다. Cascaded 구조도 유사하게 view distance나 depth range를 여러 구간으로 나누어 각 구간에 다른 해상도 resource를 사용한다.

대표 예시는 다음과 같다.

- Geometry Clipmap
- Texture Clipmap
- Cascaded Shadow Maps
- Sparse Voxel Clipmap
- Volume Clipmap
- Terrain LOD Ring
- Cascaded Voxel / SDF LOD

즉 clipmap/cascade는 “모든 공간을 최고 해상도로 들고 있지 않고, 카메라 기준 중요도에 맞춰 해상도 계층을 배치하는 방식”이다.

## 2. 한 줄 핵심

**Clipmap and Cascaded LOD Structures는 카메라 주변은 고해상도, 먼 영역은 저해상도로 유지해 large-scale terrain, volume, voxel, shadow, scientific data를 제한된 GPU memory와 bandwidth 안에서 다루는 계층적 LOD 구조다.**

## 3. 왜 중요한가

Large-scale renderer에서는 전체 world나 simulation domain을 최고 해상도로 GPU에 올릴 수 없다. Terrain, volume, sparse voxel, CFD field, semiconductor process volume, shadow map은 모두 공간 범위가 커질수록 memory와 bandwidth가 폭발한다.

예를 들어 3D volume을 uniform grid로만 관리하면 해상도를 조금만 올려도 memory가 세제곱으로 증가한다. Terrain도 전체 지형을 최고 resolution mesh로 유지하면 vertex 수와 texture memory가 너무 커진다.

Clipmap과 cascade는 이 문제를 다음 방식으로 해결한다.

- 카메라 주변의 detail만 높게 유지한다.
- 멀리 있는 영역은 coarse resolution으로 표현한다.
- level 간 update를 incremental하게 수행한다.
- GPU memory를 고정 크기 ring buffer처럼 재사용한다.
- screen-space error 기반 LOD와 자연스럽게 결합한다.

Graphics engineer 관점에서 이 개념은 단순 LOD가 아니라, **camera-relative data streaming과 GPU memory residency 관리**에 가깝다.

## 4. 구현 관점

### 4.1 Clipmap의 기본 구조

Clipmap은 여러 resolution level을 가진다. 각 level은 카메라 주변 일정 범위를 덮지만, level이 올라갈수록 cell size가 커지고 표현 범위가 넓어진다.

```text
Level 0: 작은 cell size, 카메라 근처
Level 1: 2배 cell size, 더 넓은 범위
Level 2: 4배 cell size, 더 넓은 범위
Level 3: 8배 cell size, 매우 넓은 범위
```

각 level은 전체 world를 저장하지 않고, 카메라 주변 window만 저장한다. 카메라가 이동하면 window도 이동하고, 새로 들어온 region만 업데이트한다.

이 방식은 terrain heightmap, texture, voxel density, SDF, velocity field 등 다양한 grid-based data에 적용할 수 있다.

### 4.2 Ring buffer와 toroidal addressing

Clipmap 구현에서 중요한 개념은 **toroidal addressing**이다. 각 level의 texture나 buffer를 고정 크기로 유지하고, 카메라가 이동할 때 전체 데이터를 다시 만들지 않고 wrap-around 방식으로 새 region만 갱신한다.

예를 들어 256x256 clipmap level이 있다고 할 때, 카메라가 한 cell 이동하면 전체 256x256을 새로 업로드하지 않고 한 줄 또는 한 열만 업데이트할 수 있다.

핵심은 다음이다.

- GPU resource 크기는 고정한다.
- World coordinate를 clipmap local coordinate로 변환한다.
- 이동한 만큼 offset을 갱신한다.
- 새로 필요한 border region만 streaming/update한다.

이 구조는 memory allocation churn을 줄이고, large world에서 stable한 performance를 만들기 좋다.

### 4.3 Cascaded 구조와의 차이

Cascade는 clipmap보다 더 일반적인 개념으로 볼 수 있다. View distance나 depth range를 여러 구간으로 나누고, 각 구간에 다른 resolution resource를 할당한다.

대표 예시는 **Cascaded Shadow Maps**다.

- 가까운 cascade: 높은 shadow map resolution
- 중간 cascade: 중간 resolution
- 먼 cascade: 낮은 resolution

Clipmap은 보통 카메라 중심의 nested grid/ring 구조에 가깝고, cascade는 view frustum이나 distance slice를 기준으로 resource를 분할하는 구조에 가깝다.

둘 다 핵심은 같다.

> 가까운 영역에는 많은 texel / voxel / sample budget을 쓰고, 먼 영역에는 적은 budget을 쓴다.

### 4.4 Terrain clipmap

Terrain rendering에서 geometry clipmap은 고전적이면서도 중요한 구조다. 카메라 주변은 촘촘한 grid mesh를 사용하고, 멀리 갈수록 더 coarse한 grid를 사용한다.

구현상 중요한 문제는 level 간 seam이다. 해상도가 다른 grid가 만나는 경계에서 crack이 생길 수 있다.

해결 방식은 다음과 같다.

- stitching geometry
- skirt mesh
- morphing transition
- shared edge constraint
- dithered LOD transition

Terrain에서는 height sampling과 normal reconstruction도 level 간 일관성이 중요하다. Normal이 갑자기 바뀌면 geometry seam이 없어도 lighting seam이 보일 수 있다.

### 4.5 Volume / Voxel clipmap

Volume clipmap은 3D grid에서 특히 중요하다. Dense 3D volume은 memory가 빠르게 증가하기 때문에 전체 domain을 최고 resolution으로 유지하기 어렵다.

3D clipmap 구조는 다음처럼 생각할 수 있다.

```text
Level 0: camera 주변 high-resolution voxel grid
Level 1: 더 넓은 medium-resolution voxel grid
Level 2: 전체 domain을 덮는 low-resolution voxel grid
```

Sparse voxel renderer에서는 brick 단위 clipmap을 사용할 수 있다. 각 level은 다른 voxel size를 가지며, visible region이나 camera-near region만 GPU에 resident하게 유지한다.

Volume rendering에서는 clipmap level 선택이 ray marching step size와 연결된다. 가까운 곳은 작은 step size, 먼 곳은 큰 step size를 사용한다.

### 4.6 Scientific visualization에서의 clipmap

CFD나 semiconductor visualization에서는 clipmap을 단순 visual LOD로만 보면 위험하다. Scientific data는 field accuracy가 중요하기 때문이다.

예를 들어 velocity, pressure, temperature, concentration field가 있을 때, camera에서 멀다고 무조건 낮은 resolution을 사용하면 중요한 gradient나 boundary layer가 사라질 수 있다.

따라서 scientific clipmap은 다음 기준을 함께 고려해야 한다.

- screen-space projected size
- scalar/vector field gradient
- material boundary
- iso-surface error
- feature importance
- user-selected region of interest

즉 scientific visualization에서는 clipmap level 선택이 visual distance뿐 아니라 data importance와 연결된다.

### 4.7 GPU-driven renderer와 연결

Clipmap / cascade는 GPU-driven renderer의 culling, LOD, streaming과 연결된다.

대표 흐름은 다음과 같다.

1. Camera position 기준으로 clipmap level origin을 갱신한다.
2. 필요한 region만 CPU/GPU streaming으로 업데이트한다.
3. Compute shader가 level별 visible brick / tile / meshlet을 culling한다.
4. Screen-space error 기준으로 적절한 level을 선택한다.
5. Indirect draw 또는 indirect dispatch로 visible region만 렌더링한다.

이 구조에서는 clipmap이 단순 data container가 아니라, visibility와 LOD selection의 spatial acceleration structure 역할을 한다.

### 4.8 Memory layout 관점

Clipmap에서 중요한 것은 level별 resource layout이다.

고려할 점은 다음이다.

- level별 texture / buffer resolution
- cell size와 world-space scale
- origin offset 저장 방식
- wrap-around addressing
- border update region
- level 간 interpolation
- cache-friendly brick ordering
- streaming latency와 prefetch margin

특히 voxel / volume clipmap에서는 3D texture update 비용이 크기 때문에 brick 단위 sparse update가 유리하다. WebGPU / Vulkan / DirectX 12에서는 copy queue, staging buffer, storage texture, sparse residency 지원 여부에 따라 설계가 달라진다.

## 5. 내 관심 분야와 연결

### CFD / scientific visualization

CFD 대용량 field visualization에서는 전체 domain을 uniform high-resolution으로 렌더링하기 어렵다. Clipmap 구조를 사용하면 카메라 주변 또는 관심 영역은 고해상도로 유지하고, 나머지는 coarse field로 유지할 수 있다.

응용 예시는 다음과 같다.

- pressure / velocity volume clipmap
- vortex region high-resolution brick 유지
- iso-surface 주변 adaptive clipmap
- user-selected ROI 중심 clipmap
- streamline seed 주변 high-resolution field sampling

중요한 것은 scientific visualization에서는 LOD가 단순 시각 품질이 아니라 field interpretation에 영향을 준다는 점이다.

### Sparse voxel / octree / NanoVDB

Sparse voxel / NanoVDB 계열에서는 clipmap이 매우 자연스럽다. Voxel brick은 level별 resolution을 가질 수 있고, camera 주변 brick만 high-resolution으로 resident하게 유지할 수 있다.

구조는 다음과 같다.

- Level 0: fine voxel brick
- Level 1: medium voxel brick
- Level 2: coarse voxel brick
- Page table: world coordinate → resident brick mapping
- GPU culling: visible brick list 생성

이 구조는 ray marching, SDF rendering, marching cubes surface extraction 모두에 연결된다.

### Game engine architecture

Game engine에서는 clipmap과 cascade가 여러 곳에 사용된다.

- terrain geometry clipmap
- virtual texture clipmap
- cascaded shadow map
- reflection / probe cascade
- voxel GI clipmap
- signed distance field cascade

즉 이 개념을 이해하면 “large world를 어떻게 제한된 GPU resource 안에 유지하는가”라는 엔진 아키텍처 관점을 얻을 수 있다.

## 6. 머릿속에 남길 질문 3개

1. Clipmap이 전체 world를 최고 해상도로 저장하지 않고도 카메라 주변 detail을 유지할 수 있는 이유는 무엇인가?
2. Cascaded Shadow Map과 volume clipmap은 서로 다른 시스템처럼 보이지만 어떤 공통된 LOD 사고를 공유하는가?
3. CFD / semiconductor visualization에서 clipmap level 선택에 field gradient나 material boundary를 함께 고려해야 하는 이유는 무엇인가?

## 7. graphics engineer 면접 질문 1개와 답변

### Q. Clipmap 또는 Cascaded LOD 구조는 무엇이며, large-scale rendering에서 왜 중요한가요?

**A.** Clipmap과 Cascaded LOD 구조는 카메라 주변에는 높은 해상도 resource를 배치하고, 멀리 있는 영역에는 낮은 해상도 resource를 배치하는 계층적 LOD 구조입니다. Terrain, volume, sparse voxel, shadow map처럼 전체 공간을 최고 해상도로 유지하기 어려운 경우에 사용됩니다. Clipmap은 보통 카메라 중심의 nested grid나 ring 구조로 구현되고, cascade는 view distance나 depth range를 여러 구간으로 나누어 각 구간에 다른 resolution을 할당합니다.

장점은 제한된 GPU memory와 bandwidth 안에서 가까운 영역의 detail을 유지할 수 있다는 점입니다. 또한 카메라 이동 시 전체 resource를 다시 만들지 않고 새로 들어온 border region만 업데이트할 수 있습니다. 단점은 level 간 seam, LOD transition, streaming latency, wrap-around addressing, field accuracy 관리가 필요하다는 점입니다. Scientific visualization에서는 단순 distance뿐 아니라 scalar gradient, material boundary, feature importance를 함께 고려해야 합니다.

## 8. 포트폴리오 / 커리어 연결

Clipmap and Cascaded LOD Structures는 포트폴리오에서 다음 메시지를 만든다.

> “나는 LOD를 object 선택 문제가 아니라, large-scale data를 camera-relative hierarchy와 GPU memory residency로 관리하는 architecture 문제로 이해한다.”

네 배경과 연결하면 다음 표현이 좋다.

- CFD / VTK 대용량 field를 block / brick / clipmap level로 관리하는 사고
- Sparse voxel / NanoVDB renderer에서 camera-near high-resolution brick residency 설계
- WebGPU / Vulkan에서 storage buffer와 3D texture update를 이용한 clipmap 구조 이해
- Game engine terrain, shadow, voxel GI cascade와 scientific visualization LOD를 연결하는 관점

면접에서는 다음처럼 말할 수 있다.

> “Clipmap은 카메라 주변의 고해상도 region을 고정 크기 GPU resource 안에서 유지하고, 이동에 따라 새 border region만 갱신하는 구조입니다. Cascaded LOD는 view range를 여러 resolution 구간으로 나누어 제한된 sample budget을 가까운 영역에 더 많이 배치하는 방식입니다.”

## 9. 내일 이어서 볼 개념

**Sparse Residency and Virtual Texturing**

Clipmap과 cascaded LOD 다음에는 sparse residency와 virtual texturing으로 이어지는 것이 자연스럽다. Clipmap이 카메라 주변의 계층적 LOD를 유지하는 구조라면, sparse residency는 필요한 tile/page만 실제 GPU memory에 올리는 resource management 기술이다.

## 10. 참고 키워드

- Clipmap
- Geometry Clipmap
- Texture Clipmap
- Volume Clipmap
- Cascaded LOD
- Cascaded Shadow Map
- Sparse Voxel Clipmap
- Voxel GI Clipmap
- Toroidal Addressing
- Ring Buffer
- Camera-relative Rendering
- GPU Residency
- Streaming
- Terrain LOD
- Volume Rendering
- Sparse Voxel
- NanoVDB
- Scientific Visualization
- Field-aware LOD
- Virtual Texturing
