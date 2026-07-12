---
title: "Motion Vector Generation for Dynamic Geometry"
date: "2026-07-12"
category: "Graphics"
tags: ["GPU", "Rendering Pipeline", "Motion Vector", "Velocity Buffer", "Temporal Reprojection", "TAA", "Denoising", "Dynamic Geometry", "Real-time Rendering"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-07-12 - Motion Vector Generation for Dynamic Geometry

## 1. 오늘의 개념

**Motion Vector Generation for Dynamic Geometry**는 temporal reprojection, TAA, ray tracing denoising, temporal upscaling에서 현재 pixel이 이전 frame에서 어디에 있었는지를 알려주기 위해, 현재 frame의 clip/screen position과 previous frame의 clip/screen position 차이를 계산해 velocity buffer에 기록하는 과정이다.

Motion vector는 단순히 camera movement만으로 계산되지 않는다. Static mesh는 previous view-projection matrix와 current view-projection matrix만으로 처리할 수 있지만, dynamic geometry는 vertex 자체가 frame마다 변한다.

대표적인 dynamic geometry는 다음과 같다.

- skinned mesh
- morph target animation
- vertex animation
- cloth / hair simulation
- particle / billboard
- procedural geometry
- tessellation / displacement
- water surface
- GPU-generated meshlet / marching cubes surface
- sparse voxel / volume hit surface

Temporal reprojection에서 motion vector가 틀리면 previous history를 잘못된 위치에서 가져오게 된다. 그 결과 ghosting, smearing, flickering, double image, denoising artifact가 생긴다.

핵심 질문은 다음이다.

> 현재 pixel을 만든 surface point는 이전 frame에서 어디에 있었는가?

## 2. 한 줄 핵심

**Motion Vector Generation은 current position과 previous position을 같은 geometry identity 기준으로 계산해, temporal reprojection이 올바른 history 위치를 찾도록 만드는 velocity buffer 생성 과정이다.**

## 3. 왜 중요한가

Modern renderer는 temporal reuse에 강하게 의존한다. TAA, temporal denoising, RT reflection, RTGI, RTAO, temporal upscaling은 모두 이전 frame 정보를 현재 frame에 재사용한다.

Motion vector가 중요한 이유는 다음과 같다.

- previous history sample 위치를 결정한다.
- camera motion과 object motion을 모두 표현한다.
- disocclusion과 ghosting 판단의 시작점이 된다.
- temporal denoiser의 history accumulation 품질을 좌우한다.
- dynamic object 주변의 smearing artifact를 줄인다.
- reconstruction / upscaling에서 sub-pixel stability를 만든다.

Motion vector가 정확하지 않으면 history validation이 아무리 좋아도 이미 잘못된 history를 가져온 상태가 된다. 즉 motion vector는 temporal pipeline의 주소 계산이다.

Graphics engineer 관점에서 velocity buffer는 단순 post-process input이 아니라, **renderer가 current frame과 previous frame 사이의 geometry correspondence를 보장하는 contract**다.

## 4. 구현 관점

### 4.1 기본 계산

Motion vector는 보통 current clip position과 previous clip position을 screen-space로 변환한 뒤 차이를 계산한다.

```glsl
vec4 currentClip = currentViewProj * currentWorldPos;
vec4 previousClip = previousViewProj * previousWorldPos;

vec2 currentNDC = currentClip.xy / currentClip.w;
vec2 previousNDC = previousClip.xy / previousClip.w;

vec2 motion = currentNDC - previousNDC;
```

Engine convention에 따라 motion vector를 `current - previous`로 저장할 수도 있고, reprojection을 위해 `previous - current`로 저장할 수도 있다. 중요한 것은 temporal pass에서 같은 convention을 사용하는 것이다.

Screen UV 기준으로 저장한다면 보통 NDC 차이에 0.5 scale을 적용한다.

```glsl
vec2 motionUV = (currentNDC - previousNDC) * 0.5;
```

### 4.2 Static mesh

Static mesh는 vertex position이 object local space에서 변하지 않는다. 따라서 current object transform과 previous object transform, current/previous view-projection matrix만 있으면 motion vector를 만들 수 있다.

```glsl
vec4 currentWorld = currentModel * localPos;
vec4 previousWorld = previousModel * localPos;

currentClip = currentViewProj * currentWorld;
previousClip = previousViewProj * previousWorld;
```

Camera만 움직인 경우에도 previousViewProj가 다르므로 motion vector가 생긴다. Object가 움직이면 previousModel과 currentModel 차이도 반영된다.

### 4.3 Skinned mesh

Skinned mesh는 가장 흔한 dynamic geometry 문제다. 현재 frame vertex는 current bone matrices로 skinning되고, previous frame vertex는 previous bone matrices로 skinning되어야 한다.

```glsl
vec4 currentSkinned = Skin(localPos, currentBoneMatrices);
vec4 previousSkinned = Skin(localPos, previousBoneMatrices);
```

여기서 중요한 점은 previous transform만 저장하는 것으로는 부족할 수 있다는 것이다. Bone animation 자체가 변하므로 previous bone palette가 필요하다.

문제가 생기는 경우는 다음이다.

- previous bone matrix를 저장하지 않음
- animation blending 결과의 previous state가 없음
- LOD가 바뀌며 vertex correspondence가 깨짐
- CPU skinning과 GPU skinning path가 다름
- motion vector pass에서 simplified skeleton을 사용함

Skinned mesh motion vector가 틀리면 캐릭터 주변에 강한 ghosting이 생긴다.

### 4.4 Morph target / vertex animation

Morph target이나 vertex animation은 vertex position 자체가 시간에 따라 변한다. 이 경우 previous frame의 morph weight 또는 previous vertex animation sample이 필요하다.

```glsl
currentPos = Base + MorphDelta * currentWeight;
previousPos = Base + MorphDelta * previousWeight;
```

Texture-driven vertex animation이나 VAT(Vertex Animation Texture)를 사용하는 경우, current time sample과 previous time sample을 모두 읽어야 한다.

```glsl
currentPos = SampleVAT(currentTime, vertexId);
previousPos = SampleVAT(previousTime, vertexId);
```

이전 position을 단순 current object transform으로만 추정하면 deformation motion이 빠져 ghosting이 생긴다.

### 4.5 Particle / billboard

Particle motion vector는 까다롭다. Particle은 point sprite나 billboard quad로 확장될 수 있고, vertex가 camera-facing으로 생성된다.

고려할 점은 다음이다.

- particle center의 previous position
- previous size / rotation
- previous camera-facing basis
- particle spawn / death
- particle id stability
- sorting 변화

새로 spawn된 particle은 previous history가 없으므로 history weight를 낮추거나 reactive mask를 사용해야 한다. 죽은 particle의 history가 남으면 ghost trail이 생길 수 있다.

Billboard는 quad vertex가 camera 방향에 따라 달라지므로, previous frame에서도 previous camera basis로 quad vertex를 재구성해 previous position을 계산해야 더 정확하다.

### 4.6 Procedural geometry / GPU-generated geometry

Mesh shader, marching cubes, tessellation, displacement, water, grass, voxel surface 같은 procedural geometry는 previous position 계산이 더 어렵다.

핵심은 geometry identity를 유지하는 것이다.

예를 들어 marching cubes surface는 iso-value나 field가 변하면 triangle topology가 바뀔 수 있다. 이 경우 current triangle과 previous triangle의 1:1 correspondence가 깨진다. Motion vector를 정확히 만들기 어렵기 때문에 history validation을 더 강하게 하거나, field-space velocity를 사용하거나, history를 제한적으로 사용해야 한다.

Procedural geometry에서 가능한 접근은 다음이다.

- same procedural parameter로 previous position 재평가
- previous simulation state 저장
- stable primitive id 유지
- world-space velocity field 사용
- topology change 영역은 history reject
- reactive mask / disocclusion mask 사용

### 4.7 Alpha-tested / transparent geometry

Foliage, hair, fence, particle 같은 alpha-tested geometry는 motion vector가 특히 어렵다. Pixel coverage가 frame마다 바뀌고, 얇은 geometry가 sub-pixel로 움직인다.

주의할 점은 다음이다.

- alpha test pass에서도 velocity를 써야 하는가?
- coverage 변화에 history를 얼마나 믿을 것인가?
- transparent object는 velocity buffer에 기록할 것인가?
- hair strand와 card의 previous position을 어떻게 만들 것인가?

많은 renderer는 opaque velocity buffer와 별도 transparent velocity / reactive mask를 구분한다. Temporal upscaling에서는 foliage나 particle 영역에 reactive mask를 사용해 history weight를 낮추기도 한다.

### 4.8 Velocity buffer precision과 encoding

Motion vector는 screen-space UV delta로 저장되는 경우가 많다. Format은 보통 RG16F, RG16_SNORM, RG32F 등으로 선택한다.

고려할 점은 다음이다.

- fast camera motion에서 range가 충분한가?
- precision 부족으로 jitter가 생기지 않는가?
- half float으로 충분한가?
- NDC 기준인지 UV 기준인지 명확한가?
- jittered projection을 사용할 때 jitter를 포함할지 제거할지 결정했는가?

TAA에서는 projection jitter가 들어간 current/previous matrix를 그대로 쓰면 motion vector에 jitter 차이가 섞여 artifact가 생길 수 있다. 보통 motion vector 계산에서는 jittered / unjittered matrix 사용 convention을 renderer 전체에서 일관되게 관리해야 한다.

## 5. 내 관심 분야와 연결

### CFD / scientific visualization

CFD visualization에서 motion vector는 camera motion뿐 아니라 field animation, particle trace, moving iso-surface, time-varying volume에 중요하다.

예를 들어 time step이 바뀌는 volume field에서 same world position의 scalar 값이 바뀌면, surface가 실제로 움직인 것처럼 보일 수 있다. Iso-surface나 level-set surface의 motion vector를 만들려면 단순 mesh transform이 아니라 field evolution을 고려해야 한다.

가능한 접근은 다음이다.

- particle trace는 particle id별 previous position 저장
- streamline animation은 segment id와 previous seed mapping 유지
- iso-surface는 field velocity 또는 previous extracted surface correspondence 사용
- volume rendering은 depth / gradient / scalar consistency로 history validation 강화
- time step change가 큰 경우 history weight 감소

Scientific visualization에서는 잘못된 motion vector가 단순 visual artifact를 넘어 field 해석을 왜곡할 수 있다.

### Sparse voxel / octree / NanoVDB

Sparse voxel / NanoVDB renderer에서는 brick LOD, streaming state, field update 때문에 motion vector와 history validation이 연결된다.

예를 들어 current frame에서 high-resolution brick이 들어오고 previous frame은 coarse brick을 사용했다면, 같은 pixel의 hit point가 다르게 계산될 수 있다. 이 경우 motion vector만으로는 충분하지 않고 brick id / LOD level / residency state를 validation에 포함해야 한다.

Procedural voxel surface에서 motion vector를 만들려면 다음을 고려해야 한다.

- previous SDF / density field sample
- previous iso-value
- previous brick transform
- stable voxel/brick id
- topology change detection

### Game engine architecture

Game engine에서는 motion vector generation이 temporal stability의 핵심이다.

- static mesh velocity
- skinned mesh velocity
- morph target velocity
- particle velocity
- vertex animation velocity
- water / foliage velocity
- procedural mesh velocity
- camera jitter handling
- transparent/reactive mask

면접에서는 motion vector를 “object velocity”라고 말하기보다, “현재 pixel의 surface point가 previous frame에서 어디에 있었는지를 계산하는 screen-space correspondence buffer”라고 설명하는 것이 좋다.

## 6. 머릿속에 남길 질문 3개

1. Skinned mesh motion vector에서 previous model matrix만으로 부족하고 previous bone matrices가 필요한 이유는 무엇인가?
2. Particle이나 alpha-tested geometry에서 motion vector가 ghosting을 완전히 해결하지 못하는 이유는 무엇인가?
3. CFD time-varying volume이나 sparse voxel surface에서 topology change가 motion vector와 history validation을 어렵게 만드는 이유는 무엇인가?

## 7. graphics engineer 면접 질문 1개와 답변

### Q. Motion Vector는 어떻게 생성하며, dynamic geometry에서는 어떤 점이 어려운가요?

**A.** Motion Vector는 현재 frame의 surface point가 previous frame에서 screen-space 어디에 있었는지를 나타내는 velocity buffer입니다. 일반적으로 current clip position과 previous clip position을 각각 NDC 또는 UV로 변환한 뒤 그 차이를 저장합니다. Static mesh는 current/previous model matrix와 current/previous view-projection matrix로 계산할 수 있습니다.

Dynamic geometry에서는 vertex 자체가 변하기 때문에 더 어렵습니다. Skinned mesh는 previous bone matrices로 previous skinned position을 계산해야 하고, morph target은 previous morph weight가 필요하며, vertex animation texture는 previous time sample을 읽어야 합니다. Particle과 billboard는 previous center, size, rotation, camera-facing basis를 고려해야 합니다. Procedural geometry나 marching cubes처럼 topology가 변하는 경우에는 stable correspondence가 깨질 수 있으므로 history validation을 강화하거나 reactive mask를 사용해야 합니다. Motion vector가 틀리면 temporal reprojection이 잘못된 history를 가져와 ghosting, smearing, flickering이 발생합니다.

## 8. 포트폴리오 / 커리어 연결

Motion Vector Generation for Dynamic Geometry는 포트폴리오에서 다음 메시지를 만든다.

> “나는 temporal reprojection 품질이 단순 post-process가 아니라, renderer가 previous position을 얼마나 정확히 생성하느냐에 달려 있다는 점을 이해한다.”

네 배경과 연결하면 다음 표현이 좋다.

- Deferred renderer / G-buffer 구조에서 velocity buffer를 별도 render target으로 관리하는 설계
- CFD time-varying volume, particle trace, iso-surface animation에서 previous state 기반 motion vector 사고
- Sparse voxel / procedural geometry에서 topology change와 history validation을 연결하는 관점
- WebGPU / Vulkan renderer에서 current/previous transform buffer, previous bone palette, previous simulation state를 관리하는 구조 이해

면접에서는 다음처럼 말할 수 있다.

> “Motion vector는 object velocity가 아니라 current pixel을 만든 surface point의 previous screen position을 찾기 위한 buffer입니다. Dynamic geometry에서는 previous vertex state, previous bone matrices, previous morph weight, previous simulation state가 필요하고, topology가 바뀌는 경우에는 history validation을 더 강하게 해야 합니다.”

## 9. 내일 이어서 볼 개념

**Reactive Mask and Disocclusion Handling**

Motion vector를 이해한 다음에는 reactive mask와 disocclusion handling으로 이어지는 것이 자연스럽다. Motion vector와 history validation만으로 해결하기 어려운 particle, transparency, fast-changing lighting, topology change 영역에서 history weight를 어떻게 조절하는지가 핵심이다.

## 10. 참고 키워드

- Motion Vector
- Velocity Buffer
- Temporal Reprojection
- Previous Position
- Current Position
- Skinned Mesh Velocity
- Previous Bone Matrix
- Morph Target
- Vertex Animation Texture
- Particle Velocity
- Billboard Velocity
- Procedural Geometry
- TAA
- Ray Tracing Denoising
- Temporal Upscaling
- Jittered Projection
- Reactive Mask
- Disocclusion
- Sparse Voxel
- Scientific Visualization
