---
title: "Primitive Amplification and Culling Trade-off"
date: "2026-06-24"
category: "Graphics"
tags: ["GPU", "Rendering Pipeline", "Mesh Shader", "Primitive Amplification", "Culling", "GPU Occupancy", "Meshlet", "LOD", "Performance"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-06-24 - Primitive Amplification and Culling Trade-off

## 1. 오늘의 개념

**Primitive Amplification and Culling Trade-off**는 mesh shader, geometry shader, tessellation, particle expansion, impostor generation, marching cubes 같은 programmable geometry pipeline에서 매우 중요한 성능 판단 기준이다.

Primitive amplification은 입력보다 더 많은 primitive를 GPU 안에서 생성하는 것이다. 예를 들어 point 하나를 quad로 확장하거나, meshlet 하나에서 여러 triangle을 emit하거나, voxel cell 하나에서 marching cubes triangle을 생성하는 것이 여기에 해당한다.

Culling은 반대로 보이지 않거나 필요 없는 primitive를 가능한 한 이른 단계에서 제거하는 것이다. Frustum culling, backface cone culling, Hi-Z occlusion culling, LOD selection, small primitive culling 등이 여기에 해당한다.

문제는 두 작업이 서로 다른 방향의 비용을 만든다는 점이다.

> Amplification은 GPU가 만들 수 있는 geometry 유연성을 높이지만, 너무 많이 만들면 occupancy, bandwidth, rasterization 비용을 압박한다. Culling은 workload를 줄이지만, culling 자체가 너무 비싸면 절약보다 비용이 커질 수 있다.

즉 핵심은 “GPU에서 더 많이 만들 수 있다”가 아니라, “만들기 전에 얼마나 줄이고, 줄이는 비용이 실제로 이득인가”를 판단하는 것이다.

## 2. 한 줄 핵심

**Primitive Amplification and Culling Trade-off는 programmable geometry pipeline에서 생성할 primitive 수, 제거할 primitive 수, culling 비용, GPU occupancy 사이의 균형을 잡는 성능 설계 문제다.**

## 3. 왜 중요한가

Mesh Shader Pipeline은 GPU가 meshlet을 직접 읽고 primitive를 생성하게 해준다. 이 구조는 매우 유연하지만, 유연성이 곧 성능을 보장하지는 않는다.

예를 들어 mesh shader에서 모든 meshlet에 대해 복잡한 culling과 LOD 판단을 수행한다고 해도, 실제로 제거되는 primitive가 적다면 culling pass가 오히려 낭비가 된다. 반대로 많은 geometry가 가려지는 장면에서는 culling이 큰 이득을 만든다.

이 trade-off는 다음 상황에서 중요하다.

- meshlet / cluster culling
- mesh shader primitive emission
- particle billboard expansion
- tessellation / displacement
- marching cubes surface generation
- impostor / procedural geometry
- sparse voxel brick rendering
- GPU-driven renderer의 visible list generation

Graphics engineer 관점에서는 primitive amplification과 culling을 각각의 기능이 아니라, **GPU workload budget을 어디에서 쓰고 어디에서 줄일지 결정하는 architecture 문제**로 봐야 한다.

## 4. 구현 관점

### 4.1 Primitive Amplification이란 무엇인가

Primitive amplification은 입력 데이터보다 더 많은 geometry를 생성하는 과정이다.

대표 예시는 다음과 같다.

- point particle → screen-facing quad
- hair strand control point → curve segments
- terrain patch → tessellated triangles
- voxel cell → marching cubes triangles
- meshlet → local primitive output
- impostor data → billboard geometry
- procedural shape parameter → actual triangles

Amplification 자체는 나쁘지 않다. CPU에서 geometry를 미리 모두 만들어 저장하는 대신, GPU에서 필요한 시점에 생성하면 memory를 줄이고 flexibility를 얻을 수 있다.

하지만 문제는 생성된 primitive가 실제 화면에 기여하지 않을 때다. 보이지 않는 primitive까지 많이 생성하면 vertex processing, primitive assembly, rasterization, depth test, shading 비용이 모두 증가한다.

### 4.2 Culling은 언제 이득인가

Culling이 이득이 되려면 다음 조건을 만족해야 한다.

```text
culling cost < saved rendering cost
```

즉 culling에 드는 compute / memory fetch / synchronization 비용보다, 제거된 primitive를 그리지 않아서 절약되는 비용이 더 커야 한다.

Culling이 유리한 장면은 다음과 같다.

- occlusion이 강한 large scene
- meshlet 수는 많지만 실제 visible 비율이 낮은 장면
- fragment shader가 무겁고 overdraw가 큰 장면
- ray marching 또는 volume shading처럼 per-pixel 비용이 큰 장면
- dense CAD / photogrammetry / scientific mesh

반대로 culling 이득이 작을 수 있는 경우는 다음과 같다.

- 대부분의 geometry가 화면에 보이는 장면
- culling bounds가 너무 느슨해 false positive가 많은 경우
- meshlet이 너무 작아 metadata overhead가 큰 경우
- culling shader가 복잡하고 memory fetch가 많은 경우
- vertex-bound가 아니라 culling pass 자체가 bottleneck이 되는 경우

### 4.3 Mesh Shader에서의 trade-off

Mesh Shader Pipeline에서는 한 workgroup이 meshlet을 읽고 primitive를 emit한다. 이때 다음 판단이 필요하다.

- meshlet 단위 frustum culling을 할 것인가?
- cone culling을 할 것인가?
- Hi-Z occlusion culling을 할 것인가?
- LOD를 mesh shader 안에서 선택할 것인가?
- primitive를 모두 emit할 것인가, 일부만 emit할 것인가?

각 판단은 비용을 만든다. 예를 들어 Hi-Z culling을 위해 depth pyramid를 샘플링하면 memory access가 추가된다. Cone culling은 비교적 저렴하지만 meshlet normal cone이 잘 구성되어 있어야 효과가 있다.

좋은 mesh shader pipeline은 모든 culling을 무조건 넣는 것이 아니라, cheap culling부터 expensive culling 순서로 배치한다.

예시는 다음과 같다.

1. Frustum culling
2. Cone culling
3. Screen-space size / LOD 판단
4. 필요할 때만 Hi-Z occlusion culling
5. Visible meshlet만 primitive emit

### 4.4 GPU occupancy 관점

Primitive amplification은 GPU occupancy와도 연결된다. Mesh shader workgroup이 너무 많은 shared memory, register, output primitive buffer를 사용하면 동시에 실행 가능한 workgroup 수가 줄어든다.

Occupancy를 낮추는 요인은 다음과 같다.

- workgroup당 많은 register 사용
- 큰 payload / shared memory 사용
- output vertex / primitive count가 너무 큼
- divergent branch가 많음
- meshlet마다 primitive count 차이가 큼
- memory fetch latency를 숨길 만큼 wave가 충분하지 않음

즉 mesh shader에서 많은 일을 한 번에 처리하려고 하면 오히려 parallelism이 줄어들 수 있다. Amplification이 많아질수록 output buffer pressure와 rasterization pressure도 커진다.

핵심은 다음이다.

> Mesh shader는 유연하지만, workgroup resource budget 안에서 작고 예측 가능한 workload를 유지해야 한다.

### 4.5 Small primitive 문제

Amplification이 지나치면 tiny triangle 또는 sub-pixel primitive가 많아질 수 있다. 작은 triangle은 실제 화면 기여도는 낮지만 rasterization overhead는 만들 수 있다.

특히 dense mesh, tessellation, marching cubes surface, displacement geometry에서 small primitive 문제가 자주 생긴다.

이를 줄이는 방법은 다음과 같다.

- screen-space error 기반 LOD
- meshlet 단위 small triangle culling
- cluster simplification
- impostor 또는 normal map 대체
- voxel / SDF surface의 adaptive extraction
- triangle density budget 관리

즉 amplification을 사용할수록 “얼마나 많이 만들 수 있는가”보다 “화면에서 의미 있는 크기인가”를 함께 봐야 한다.

### 4.6 Marching Cubes와 amplification

Marching Cubes는 CFD / voxel / SDF / level-set visualization에서 대표적인 primitive amplification 사례다. 하나의 voxel cell에서 0개에서 최대 여러 개의 triangle이 생성된다.

GPU에서 marching cubes를 수행하면 다음 trade-off가 생긴다.

- 전체 surface mesh를 CPU에 저장하지 않아도 된다.
- iso-value 변경에 동적으로 대응할 수 있다.
- 하지만 active cell이 많으면 triangle amplification이 커진다.
- generated triangle을 compact하는 비용이 필요하다.
- 작은 triangle이 많으면 rasterization 비용이 커진다.

따라서 sparse voxel / CFD renderer에서는 먼저 active cell culling, brick-level min/max scalar range test, LOD selection을 수행한 뒤 marching cubes amplification을 하는 것이 유리하다.

즉 순서는 다음이 좋다.

```text
brick culling → scalar range test → LOD 선택 → active cell compaction → triangle generation
```

### 4.7 Particle / billboard expansion

Particle renderer에서도 amplification trade-off가 중요하다. Particle 하나를 quad로 확장하면 primitive 수가 2 triangles로 늘어난다. 수십만 particle에서는 이 비용이 커진다.

Culling 전략은 다음과 같다.

- frustum 밖 particle 제거
- 너무 작은 particle은 skip 또는 point 처리
- depth pre-pass / Hi-Z 기반 occlusion
- tile-based particle binning
- transparency sorting 비용 제한
- screen-space density 기반 LOD

Particle은 transparency와 blending 때문에 depth culling이 완벽하지 않다. 따라서 opaque meshlet보다 더 보수적인 전략이 필요하다.

### 4.8 Trade-off를 판단하는 성능 질문

Primitive amplification과 culling을 설계할 때는 다음 질문을 던져야 한다.

1. 이 stage에서 생성되는 primitive 수는 input 대비 얼마나 증가하는가?
2. 생성된 primitive 중 실제 screen contribution이 있는 비율은 얼마인가?
3. culling cost는 ALU-bound인가 memory-bound인가?
4. culling 결과를 compact하는 비용이 큰가?
5. amplification이 occupancy를 낮추는가?
6. rasterization / fragment shading 비용을 실제로 줄이는가?
7. scene이 camera angle에 따라 성능 변동이 큰가?

이 질문은 mesh shader뿐 아니라 compute-driven visualization, particle simulation, voxel rendering 모두에 적용된다.

## 5. 내 관심 분야와 연결

### CFD / scientific visualization

CFD visualization에서는 primitive amplification이 자주 등장한다.

- iso-surface extraction
- streamline tube generation
- glyph rendering
- particle trace rendering
- vector field arrow generation
- slice contour line generation

이때 모든 field cell에서 geometry를 만들면 비용이 크다. 먼저 block / brick / scalar range / screen-space visibility 기준으로 줄인 뒤, 실제 필요한 부분에서만 geometry를 생성해야 한다.

예를 들어 iso-surface에서는 brick별 scalar min/max를 가지고 iso-value가 해당 범위에 들어가는 brick만 active로 표시할 수 있다. 그 다음 active cell만 marching cubes 대상으로 삼으면 amplification을 크게 줄일 수 있다.

### Sparse voxel / octree / NanoVDB

Sparse voxel renderer에서는 brick-level culling과 primitive amplification의 순서가 중요하다. Marching Cubes 또는 SDF surface patch generation을 하기 전에 다음을 먼저 수행하는 것이 좋다.

- empty brick 제거
- frustum culling
- Hi-Z occlusion culling
- screen-space error 기반 LOD
- scalar / SDF sign range test

그 다음 필요한 brick만 surface triangle으로 amplify한다. 이 구조는 NanoVDB나 sparse volume renderer에서도 핵심이다.

### Game engine architecture

Game engine에서는 amplification과 culling trade-off가 다음 시스템에 직접 연결된다.

- mesh shader
- tessellation / displacement
- particle billboard
- foliage rendering
- virtual geometry
- impostor system
- GPU-driven culling

면접에서는 “mesh shader가 더 빠르다”보다 “mesh shader는 amplification과 culling을 shader 안에서 제어할 수 있지만, occupancy와 output primitive pressure를 관리해야 한다”고 말하는 것이 훨씬 강하다.

## 6. 머릿속에 남길 질문 3개

1. Primitive amplification이 geometry flexibility를 높이지만 성능을 악화시킬 수 있는 이유는 무엇인가?
2. Mesh shader에서 frustum / cone / Hi-Z culling을 모두 넣는 것이 항상 좋은 선택이 아닌 이유는 무엇인가?
3. CFD marching cubes pipeline에서 brick-level culling을 triangle generation보다 먼저 해야 하는 이유는 무엇인가?

## 7. graphics engineer 면접 질문 1개와 답변

### Q. Mesh Shader나 procedural geometry pipeline에서 primitive amplification과 culling 사이의 trade-off를 어떻게 판단해야 하나요?

**A.** Primitive amplification은 point, voxel cell, meshlet 같은 작은 입력에서 더 많은 primitive를 생성하는 과정입니다. Mesh shader, particle billboard, tessellation, marching cubes 등이 대표 예입니다. 이 방식은 유연하지만, 보이지 않거나 화면 기여도가 낮은 primitive까지 많이 생성하면 vertex processing, rasterization, fragment shading, memory bandwidth 비용이 커집니다.

Culling은 이 workload를 줄이기 위한 단계지만, culling 자체도 비용이 있습니다. 따라서 기본 판단 기준은 culling cost가 제거된 primitive를 렌더링하지 않아 절약되는 비용보다 작은가입니다. Frustum이나 cone culling처럼 저렴한 culling은 앞단에 배치하고, Hi-Z처럼 memory access가 필요한 culling은 제거 효과가 큰 경우에 사용하는 것이 좋습니다. Mesh shader에서는 amplification이 workgroup register, shared memory, output primitive buffer를 압박해 occupancy를 낮출 수 있으므로, meshlet 크기와 culling granularity를 함께 조정해야 합니다.

## 8. 포트폴리오 / 커리어 연결

Primitive Amplification and Culling Trade-off는 포트폴리오에서 다음 메시지를 만든다.

> “나는 GPU가 geometry를 더 유연하게 생성하는 기능을 단순히 사용하는 것이 아니라, 생성 비용과 제거 비용, occupancy, bandwidth를 함께 고려해 rendering pipeline을 설계할 수 있다.”

네 배경과 연결하면 다음 표현이 좋다.

- CFD iso-surface / marching cubes에서 active brick culling 후 triangle generation으로 비용을 줄이는 사고
- Particle / streamline / glyph visualization에서 screen contribution 기반 LOD를 고려하는 설계
- Meshlet / mesh shader pipeline에서 culling 순서와 output primitive pressure를 판단하는 능력
- Sparse voxel / NanoVDB 계열 renderer에서 brick culling과 surface amplification을 분리하는 구조

면접에서는 다음처럼 말할 수 있다.

> “Programmable geometry pipeline에서는 더 많은 primitive를 만들 수 있다는 점보다, 만들기 전에 얼마나 줄일 수 있는지가 중요합니다. Culling cost와 saved rendering cost를 비교하고, amplification이 GPU occupancy와 rasterization pressure를 얼마나 증가시키는지 함께 봐야 합니다.”

## 9. 내일 이어서 볼 개념

**Screen-Space Error Based LOD**

Primitive amplification과 culling trade-off 다음에는 screen-space error 기반 LOD를 보는 것이 자연스럽다. Geometry를 만들지 말지, 어떤 LOD를 사용할지, meshlet / voxel brick / surface patch를 얼마나 세밀하게 렌더링할지를 화면 기여도 기준으로 판단하는 개념이다.

## 10. 참고 키워드

- Primitive Amplification
- Culling Trade-off
- Mesh Shader
- Meshlet Culling
- GPU Occupancy
- Workgroup Resource
- Output Primitive Pressure
- Cone Culling
- Hi-Z Culling
- Frustum Culling
- Small Primitive Problem
- Tessellation
- Particle Billboard
- Marching Cubes
- Active Cell Compaction
- Sparse Voxel Brick Culling
- Screen-space Error
- LOD Selection
- GPU-driven Rendering
- Scientific Visualization
