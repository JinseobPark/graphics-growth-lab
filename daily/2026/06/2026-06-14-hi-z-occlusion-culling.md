---
title: "Hi-Z Occlusion Culling"
date: "2026-06-14"
category: "Graphics"
tags: ["GPU", "Rendering Pipeline", "Hi-Z", "Occlusion Culling", "Depth Pyramid", "GPU-Driven Rendering", "Compute Shader", "Indirect Draw", "Visibility"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-06-14 - Hi-Z Occlusion Culling

## 1. 오늘의 개념

**Hi-Z Occlusion Culling**은 이전 frame 또는 depth pre-pass에서 얻은 depth buffer를 mipmap 형태의 **depth pyramid**로 만들고, object / meshlet / voxel brick의 screen-space bounding area가 이미 그려진 geometry 뒤에 가려지는지 빠르게 판단하는 GPU visibility 기법이다.

일반적인 frustum culling은 camera view volume 밖에 있는 object를 제거한다. 하지만 frustum 안에 있어도 실제 화면에서는 다른 geometry 뒤에 가려져 보이지 않는 object가 많다. Hi-Z occlusion culling은 이 문제를 해결하기 위해 depth buffer를 계층적으로 압축한다.

핵심 구조는 다음과 같다.

1. Scene의 depth를 먼저 만든다.
2. Depth buffer를 1/2, 1/4, 1/8 ... 해상도의 mip chain으로 축소한다.
3. 각 mip level에는 해당 영역의 conservative depth 값을 저장한다.
4. Object의 bounding volume을 screen-space rectangle로 투영한다.
5. Rectangle 크기에 맞는 depth mip level을 선택한다.
6. Object의 nearest depth와 pyramid depth를 비교해 가려졌는지 판단한다.

즉 Hi-Z는 “화면 전체 depth buffer를 매번 정밀하게 검사하지 않고, coarse depth level에서 빠르게 occlusion을 판단하는 구조”다.

## 2. 한 줄 핵심

**Hi-Z Occlusion Culling은 depth buffer를 계층적 pyramid로 만들어, GPU가 object / meshlet / voxel brick이 이미 보이는 geometry 뒤에 가려졌는지 빠르게 판정하도록 하는 visibility optimization이다.**

## 3. 왜 중요한가

GPU-driven rendering에서 frustum culling만으로는 충분하지 않다. Camera 안에 들어와 있어도 실제로는 벽, 지형, 건물, 큰 mesh, volume block 뒤에 숨어 있는 object가 많다. 이들을 모두 draw하면 rasterization, vertex processing, shading, ray marching 비용이 낭비된다.

Hi-Z occlusion culling이 중요한 이유는 다음과 같다.

- 보이지 않는 object의 draw를 GPU 내부에서 제거한다.
- CPU readback 없이 visibility result를 GPU buffer에 유지할 수 있다.
- indirect draw command 생성 전에 visible list를 줄일 수 있다.
- meshlet / cluster / voxel brick 단위 culling과 잘 맞는다.
- large scene, sparse voxel, CFD block rendering, scientific visualization에 유리하다.

특히 GPU-driven renderer에서는 CPU가 object별 visibility를 관리하지 않기 때문에, GPU가 사용할 수 있는 depth-based visibility filter가 필요하다. Hi-Z는 그 대표적인 도구다.

## 4. 구현 관점

### 4.1 Depth pyramid란 무엇인가

Depth pyramid는 depth buffer의 mipmap이다. 하지만 color texture mipmap처럼 평균값을 저장하는 것이 아니라, occlusion 판정을 위해 보수적인 depth 값을 저장한다.

일반적인 reversed-Z가 아닌 depth convention에서는 가까운 값이 작고 먼 값이 크다. 이 경우 pyramid에는 보통 영역 내 **minimum depth** 또는 목적에 맞는 conservative value를 저장한다.

Reversed-Z에서는 가까운 값이 크고 먼 값이 작으므로, pyramid reduction에서 사용하는 min/max 방향이 반대로 바뀐다.

핵심은 API나 depth convention마다 비교 방향이 달라진다는 점이다.

- Normal Z: near = small, far = large
- Reversed Z: near = large, far = small
- OpenGL / Vulkan / DirectX NDC depth range 차이 고려
- Linear depth인지 non-linear depth인지 고려

Hi-Z 구현에서 가장 흔한 실수는 depth compare 방향을 잘못 잡는 것이다. Reversed-Z를 쓰면 precision은 좋아지지만 occlusion comparison logic도 함께 바뀐다.

### 4.2 Pyramid 생성 pass

Depth pyramid는 compute shader 또는 graphics fullscreen pass로 생성할 수 있다.

Compute shader 방식에서는 mip level N을 읽고 mip level N+1에 2x2 block의 min/max depth를 기록한다.

개념적으로는 다음과 같다.

```glsl
float d0 = depthPrev[base + ivec2(0, 0)];
float d1 = depthPrev[base + ivec2(1, 0)];
float d2 = depthPrev[base + ivec2(0, 1)];
float d3 = depthPrev[base + ivec2(1, 1)];

float conservativeDepth = max(max(d0, d1), max(d2, d3));
depthMipNext[pixel] = conservativeDepth;
```

위 예시는 depth convention에 따라 min/max가 달라질 수 있다. 중요한 것은 “이 영역에서 occluder가 얼마나 가까이 있는가”를 보수적으로 표현해야 한다는 점이다.

### 4.3 Object bounds를 screen-space로 투영하기

Occlusion culling 대상은 보통 object의 bounding sphere, AABB, OBB, meshlet cone, voxel brick bounds다.

GPU culling shader에서는 다음 일을 한다.

1. Object bounds의 corner 또는 sphere를 clip space로 변환한다.
2. NDC로 나눈다.
3. screen-space rectangle을 만든다.
4. rectangle의 width/height를 기준으로 적절한 mip level을 고른다.
5. rectangle 영역의 pyramid depth를 샘플링한다.
6. object의 nearest depth와 비교한다.

Screen-space rectangle이 클수록 낮은 해상도 mip을 사용하면 너무 거칠 수 있고, 너무 높은 해상도 mip을 쓰면 샘플 수가 많아진다. 일반적으로 rectangle 크기에 맞춰 `log2(max(width, height))`에 가까운 mip level을 고른다.

### 4.4 Occlusion 판정의 핵심

Occlusion 판정은 대략 다음 사고방식이다.

> Object의 가장 가까운 depth가 해당 화면 영역의 occluder depth보다 더 뒤에 있다면, object는 가려졌을 가능성이 높다.

하지만 이것은 conservative하게 처리해야 한다. 잘못 culling하면 실제로 보여야 하는 object가 사라지는 popping artifact가 생긴다.

따라서 실무에서는 다음 보정이 들어간다.

- bounding volume을 약간 확장
- depth bias 적용
- 너무 작은 object는 culling하지 않음
- 이전 frame depth 사용 시 camera motion 고려
- near plane을 가로지르는 bounds는 안전하게 visible 처리
- alpha-tested geometry나 transparent object는 occluder로 조심스럽게 사용

Hi-Z는 aggressive하게 쓰면 성능은 좋아지지만 visual correctness가 깨질 수 있다. Conservative culling이 기본이다.

### 4.5 Previous frame depth vs current frame depth

Hi-Z는 보통 두 방식으로 사용된다.

#### 1. Current frame depth pre-pass 기반

먼저 depth pre-pass를 수행해 현재 frame의 depth buffer를 만든다. 이후 pyramid를 만들고, main pass 전에 occlusion culling을 한다.

장점:

- 현재 frame 기준이라 정확도가 높다.
- camera motion artifact가 적다.

단점:

- depth pre-pass 비용이 추가된다.
- opaque geometry를 최소 한 번은 먼저 그려야 한다.

#### 2. Previous frame depth 기반

이전 frame의 depth pyramid를 사용해 현재 frame의 object visibility를 예측한다.

장점:

- current frame에서 depth pre-pass 없이 culling 가능하다.
- GPU-driven pipeline에서 더 자연스럽다.

단점:

- camera가 빠르게 움직이면 false occlusion이 생길 수 있다.
- dynamic object나 newly visible object에서 popping이 발생할 수 있다.

실무 renderer에서는 둘을 혼합하거나, temporal coherence를 활용하면서도 conservative bias를 둔다.

### 4.6 GPU-driven rendering과 연결

Hi-Z는 GPU-driven rendering pipeline에서 visible list를 줄이는 중간 filter로 들어간다.

대표 흐름은 다음과 같다.

1. CPU가 scene buffer를 GPU에 업로드한다.
2. Compute shader가 frustum culling을 수행한다.
3. Frustum 안의 object만 Hi-Z occlusion test를 통과시킨다.
4. Visible object index를 append / compact buffer에 기록한다.
5. Compute shader가 indirect draw command를 생성한다.
6. Graphics pass가 indirect draw를 실행한다.

이 구조에서 CPU는 object별 occlusion result를 읽지 않는다. 결과는 GPU buffer에 남고, 다음 draw command 생성에 바로 사용된다.

이 점이 중요하다.

> Hi-Z의 목적은 “CPU가 알기 위한 visibility”가 아니라, “GPU가 다음 draw list를 만들기 위한 visibility”다.

### 4.7 Meshlet / voxel brick 단위의 Hi-Z

Hi-Z는 object 단위보다 더 작은 단위에서 강력해진다.

Meshlet renderer에서는 meshlet bounding sphere를 screen-space로 투영해 occlusion test를 할 수 있다. 큰 object 하나가 frustum 안에 있더라도, 그 내부 meshlet 중 일부는 다른 geometry에 가려질 수 있다.

Sparse voxel renderer에서는 voxel brick 또는 octree node의 bounding box를 screen-space로 투영해 테스트할 수 있다. 가려진 brick은 ray marching이나 surface rendering pass로 보내지 않아도 된다.

CFD visualization에서는 block 단위 scalar field rendering, volume brick, streamline seed region visibility에도 유사한 사고를 적용할 수 있다.

## 5. 내 관심 분야와 연결

### Sparse voxel / octree / NanoVDB

Sparse voxel structure에서 모든 node나 brick을 순회하며 렌더링하면 비용이 크다. Hi-Z는 frustum culling 이후에도 실제 화면에서 가려진 brick을 제거하는 데 사용할 수 있다.

예를 들어 반도체 3D visualization에서 여러 layer와 taper angle이 있는 구조를 volume / SDF / mesh hybrid로 렌더링한다고 가정하면, camera 뒤쪽 또는 앞 layer에 가려진 brick은 shading하거나 ray marching할 필요가 없다.

이때 pipeline은 다음처럼 생각할 수 있다.

- Octree node bounds 생성
- Frustum culling
- Hi-Z occlusion culling
- Screen-space error 기반 LOD 선택
- Visible brick list 생성
- Indirect dispatch / indirect draw

이 구조는 dense 3D grid를 sparse grid로 바꾸는 것의 다음 단계다. Sparse data structure만으로는 충분하지 않고, screen-space visibility filter까지 붙어야 real-time renderer가 된다.

### CFD / scientific visualization

CFD 대용량 데이터에서는 block, cell group, streamline segment, volume brick이 많다. Hi-Z는 실제 화면에서 보이지 않는 block을 제거해 rendering cost를 줄이는 데 사용할 수 있다.

예를 들어 volume rendering에서는 모든 brick을 ray marching하지 않고, visible brick list만 대상으로 dispatch할 수 있다. Iso-surface나 slice rendering에서도 bounds 기반으로 occluded block을 제외할 수 있다.

단, scientific visualization에서는 “정확히 보여야 할 데이터가 사라지는 문제”가 치명적일 수 있으므로 game renderer보다 conservative bias를 더 강하게 두는 편이 안전하다.

### Game engine architecture

Game engine에서는 Hi-Z가 다음 시스템과 연결된다.

- GPU occlusion culling
- Nanite-like cluster visibility
- depth pre-pass
- virtual geometry
- software rasterization culling
- indirect draw command generation
- GPU scene buffer

Hi-Z를 이해하면 Unreal Engine의 virtualized geometry, large world renderer, GPU culling paper를 읽을 때 핵심 구조가 더 잘 보인다.

## 6. 머릿속에 남길 질문 3개

1. Hi-Z occlusion culling에서 depth pyramid의 mip level을 object screen-space size에 맞춰 선택하는 이유는 무엇인가?
2. Reversed-Z를 사용하는 renderer에서 Hi-Z depth reduction과 comparison 방향은 어떻게 달라져야 하는가?
3. Sparse voxel renderer에서 brick 단위 Hi-Z culling이 ray marching 비용을 줄이는 방식은 무엇인가?

## 7. graphics engineer 면접 질문 1개와 답변

### Q. Hi-Z Occlusion Culling이 무엇이며, GPU-driven rendering에서 어떻게 사용되나요?

**A.** Hi-Z Occlusion Culling은 depth buffer를 mipmap 형태의 depth pyramid로 만들어 object나 meshlet의 screen-space bounds가 이미 그려진 geometry 뒤에 가려졌는지 빠르게 판단하는 기법입니다. 일반 frustum culling은 camera 밖의 object만 제거하지만, Hi-Z는 frustum 안에 있어도 실제 화면에서 가려진 object를 제거할 수 있습니다.

GPU-driven rendering에서는 compute shader가 object bounds를 screen-space rectangle로 투영하고, rectangle 크기에 맞는 depth mip level을 샘플링합니다. Object의 nearest depth가 해당 영역의 conservative occluder depth보다 뒤에 있으면 visible list에 넣지 않고 제거합니다. 이후 visible object list를 바탕으로 indirect draw command를 생성합니다. 장점은 CPU readback 없이 GPU 내부에서 visibility result를 유지할 수 있다는 점입니다. 단점은 depth convention, reversed-Z, temporal artifact, conservative bias, alpha-tested geometry 처리 등을 신중히 설계해야 한다는 점입니다.

## 8. 포트폴리오 / 커리어 연결

Hi-Z Occlusion Culling은 포트폴리오에서 다음 역량을 보여준다.

> “나는 단순히 object를 그리는 것이 아니라, GPU visibility pipeline을 설계하고 draw workload를 줄이는 구조를 이해한다.”

특히 다음 경험과 연결할 수 있다.

- OpenGL / Vulkan renderer에서 depth pre-pass와 depth texture 관리
- Compute shader 기반 culling pipeline 설계
- VTK / CFD 대용량 block rendering에서 visible block list 구성
- Sparse voxel / octree brick visibility 판단
- WebGPU storage buffer와 indirect draw 기반 renderer 설계

면접에서는 다음 표현이 좋다.

> “Hi-Z는 GPU-driven renderer에서 frustum culling 이후의 depth-based visibility filter로 사용되며, depth pyramid를 통해 coarse screen-space occlusion을 빠르게 판단하고 indirect draw list를 줄이는 역할을 합니다.”

이 표현은 rendering pipeline, GPU memory, compute culling, indirect draw를 하나의 구조로 이해하고 있다는 신호가 된다.

## 9. 내일 이어서 볼 개념

**Depth Pre-Pass vs Forward/Deferred Visibility Strategy**

Hi-Z를 이해한 다음에는 depth pre-pass를 언제 쓰고, forward / deferred renderer에서 visibility strategy가 어떻게 달라지는지 보는 것이 자연스럽다. Hi-Z가 depth buffer에 의존하기 때문에, renderer가 depth를 언제 만들고 어떻게 재사용하는지가 전체 pipeline 설계에 큰 영향을 준다.

## 10. 참고 키워드

- Hi-Z Occlusion Culling
- Hierarchical Z
- Depth Pyramid
- Depth Mipmap
- GPU Occlusion Culling
- Compute Culling
- Frustum Culling
- Indirect Draw
- GPU-Driven Rendering
- Visible List
- Append Buffer
- Reversed-Z
- Conservative Depth
- Screen-space Bounds
- Meshlet Culling
- Cluster Culling
- Sparse Voxel Brick Culling
- Volume Rendering Optimization
- Depth Pre-Pass
- Temporal Occlusion Culling
