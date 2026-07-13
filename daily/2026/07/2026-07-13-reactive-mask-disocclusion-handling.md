---
title: "Reactive Mask and Disocclusion Handling"
date: "2026-07-13"
category: "Graphics"
tags: ["GPU", "Rendering Pipeline", "Reactive Mask", "Disocclusion", "Temporal Reprojection", "TAA", "Denoising", "Motion Vector", "Real-time Rendering"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-07-13 - Reactive Mask and Disocclusion Handling

## 1. 오늘의 개념

**Reactive Mask and Disocclusion Handling**은 temporal reprojection, TAA, temporal upscaling, ray tracing denoising에서 previous frame history를 얼마나 신뢰할지 pixel별로 조절하는 구조다.

Temporal pipeline은 이전 frame의 color / lighting / denoising result를 현재 frame에 재사용한다. 하지만 어떤 pixel은 history를 강하게 믿으면 안 된다.

대표적인 경우는 다음과 같다.

- 새로 드러난 disocclusion 영역
- particle, fire, smoke, water foam 같은 빠르게 변하는 effect
- alpha-tested foliage / hair / fence
- transparency와 refraction
- fast-changing lighting
- procedural geometry 또는 topology change
- sparse voxel / marching cubes surface의 LOD 변화
- newly streamed high-resolution brick

Reactive mask는 이런 영역에 대해 “history를 덜 믿고 current frame을 더 믿어라”라는 신호를 준다. Disocclusion handling은 이전 frame에 존재하지 않았거나 다른 surface였던 영역을 감지해 history를 reject하거나 weight를 줄이는 과정이다.

핵심 질문은 다음이다.

> 이 pixel은 previous history를 누적해도 안정적인가, 아니면 현재 frame 정보를 더 강하게 반영해야 하는가?

## 2. 한 줄 핵심

**Reactive Mask and Disocclusion Handling은 temporal reconstruction에서 history를 신뢰하기 어려운 pixel을 표시하고, history weight를 낮춰 ghosting, smearing, stale lighting artifact를 줄이는 신뢰도 제어 구조다.**

## 3. 왜 중요한가

Temporal reprojection은 image stability를 크게 높인다. 하지만 history를 많이 사용할수록 반응성이 떨어지고 ghosting이 생긴다. 특히 particle, transparent object, fast-changing lighting처럼 현재 frame 변화가 큰 영역에서는 previous history가 오히려 잘못된 정보를 만든다.

Reactive mask와 disocclusion handling이 중요한 이유는 다음과 같다.

- 새로 드러난 영역의 invalid history를 제거한다.
- particle / transparency 주변의 ghost trail을 줄인다.
- fast-changing lighting과 emissive effect의 lag를 줄인다.
- temporal upscaling에서 thin geometry와 alpha-tested edge를 안정화한다.
- ray tracing denoising에서 history accumulation이 잘못된 surface를 섞지 않게 한다.
- sparse voxel / volume LOD 전환에서 stale history를 줄인다.

Graphics engineer 관점에서 reactive mask는 단순 post-process 옵션이 아니라, **temporal reconstruction의 confidence control buffer**다.

## 4. 구현 관점

### 4.1 History weight의 의미

Temporal accumulation은 보통 current frame과 previous history를 섞는다.

```glsl
vec3 result = mix(currentColor, historyColor, historyWeight);
```

`historyWeight`가 높으면 image가 안정적이고 noise가 줄어든다. 하지만 변화에 둔감해지고 ghosting이 생길 수 있다.

Reactive mask는 이 weight를 낮추는 데 사용된다.

```glsl
float reactive = reactiveMask[pixel];
historyWeight *= (1.0 - reactive);
```

즉 reactive 값이 높을수록 current frame을 더 많이 반영한다.

### 4.2 Disocclusion detection

Disocclusion은 이전 frame에서 가려져 있었지만 현재 frame에서 새로 보이는 영역이다. 이 영역은 신뢰할 previous history가 없다.

대표적인 detection 방법은 다음과 같다.

- reprojected previous UV가 screen 밖에 있음
- current depth와 previous depth가 크게 다름
- previous normal과 current normal이 크게 다름
- object id / material id가 다름
- motion vector가 불연속적임
- neighborhood depth comparison에서 foreground/background mismatch 발생

```glsl
bool disoccluded = false;
if (!InsideScreen(prevUV)) disoccluded = true;
if (abs(currentDepth - prevDepth) > depthThreshold) disoccluded = true;
if (dot(currentNormal, prevNormal) < normalThreshold) disoccluded = true;

if (disoccluded)
    historyWeight = 0.0;
```

Depth validation만으로는 부족할 수 있다. Thin geometry나 alpha-tested object는 depth coverage가 frame마다 바뀌기 때문이다.

### 4.3 Reactive mask 생성 방식

Reactive mask는 material pass, post-process pass, effect system, temporal upscaler input pass에서 생성할 수 있다.

대표 기준은 다음이다.

- material이 transparent / refractive인가?
- emissive가 빠르게 변하는가?
- particle / effect layer인가?
- alpha coverage가 크게 변하는가?
- current color가 previous color와 크게 다른가?
- lighting contribution이 high variance인가?
- procedural geometry topology가 바뀌었는가?
- sparse data LOD / residency가 바뀌었는가?

Material이 직접 reactive 값을 출력할 수도 있다.

```glsl
reactiveMask = material.reactiveFactor;
```

또는 color difference를 기반으로 자동 추정할 수 있다.

```glsl
reactive = saturate(length(currentColor - reprojectedHistoryColor) * scale);
```

자동 추정은 편하지만 noise나 lighting change를 과하게 reactive로 판단할 수 있다.

### 4.4 Transparency와 particle

Particle, fire, smoke, water, magic effect 같은 transparency는 temporal accumulation에 취약하다.

이유는 다음과 같다.

- depth가 명확하지 않다.
- 여러 layer가 blending된다.
- motion vector가 surface point를 정확히 표현하지 못한다.
- alpha coverage가 frame마다 크게 바뀐다.
- particle spawn/death가 history correspondence를 깨뜨린다.

따라서 많은 renderer는 transparent / particle 영역에 높은 reactive mask를 준다. Temporal upscaling에서는 이런 영역의 history weight를 낮춰 ghost trail을 줄인다.

하지만 reactive를 너무 강하게 주면 noise와 flickering이 증가한다. 안정성과 반응성의 균형이 필요하다.

### 4.5 Alpha-tested geometry

Foliage, hair card, fence 같은 alpha-tested geometry는 opaque처럼 depth를 쓰지만 coverage가 sub-pixel에서 빠르게 변한다.

문제는 다음이다.

- motion vector는 맞아도 coverage가 달라질 수 있다.
- thin edge가 previous frame과 current frame에서 다른 pixel을 차지한다.
- TAA history가 edge를 따라 끌려가 ghosting이 생긴다.

대응 방식은 다음이다.

- alpha-tested material에 reactive mask 부여
- alpha coverage change를 기반으로 history weight 감소
- neighborhood clipping 강화
- velocity dilation 사용
- depth/normal validation threshold 조정

### 4.6 Fast-changing lighting

Temporal denoising은 lighting history를 재사용한다. 하지만 light가 빠르게 변하거나 emissive object가 깜빡이면 history가 stale해진다.

예시는 다음과 같다.

- flashing light
- muzzle flash
- animated emissive material
- moving shadow caster
- dynamic GI change
- reflection에서 갑자기 나타나는 bright object

이런 경우 reactive mask는 material/object뿐 아니라 lighting contribution에도 필요하다. Denoiser가 previous lighting을 너무 오래 유지하면 light lag가 생긴다.

### 4.7 Sparse voxel / procedural topology change

Procedural geometry나 sparse voxel renderer에서는 topology가 변할 수 있다. 예를 들어 marching cubes iso-surface가 field 변화로 새 triangle을 만들거나 제거하면, previous history와 current geometry의 correspondence가 깨진다.

이때 reactive 신호로 사용할 수 있는 정보는 다음이다.

- brick id 변경
- LOD level 변경
- residency state 변경
- field value change
- iso-value crossing change
- material boundary change
- topology generation version

예를 들어 current frame에서 high-resolution brick이 새로 로드되었다면 previous coarse history를 그대로 쓰면 blur나 ghosting이 생길 수 있다. 이 영역은 reactive를 높이거나 history validation을 강화해야 한다.

### 4.8 Reactive mask와 history validation의 차이

History validation은 “이 history가 같은 surface인가?”를 검사한다.

Reactive mask는 “같은 surface여도 history를 얼마나 믿을 것인가?”를 조절한다.

예를 들어 물 표면은 same surface일 수 있지만 reflection이나 refraction이 frame마다 빠르게 변한다. Depth/normal validation은 통과해도 history를 강하게 쓰면 ghosting이 생길 수 있다. 이때 reactive mask가 필요하다.

즉 둘은 보완 관계다.

```text
History validation: wrong history를 reject
Reactive mask: unstable history의 weight를 낮춤
```

## 5. 내 관심 분야와 연결

### CFD / scientific visualization

CFD / scientific visualization에서는 reactive mask를 field-aware temporal filtering으로 확장할 수 있다.

예를 들어 다음 영역은 history를 덜 믿어야 한다.

- shock / vortex / interface가 빠르게 이동하는 영역
- scalar gradient가 큰 영역
- material boundary가 바뀌는 영역
- time step 간 field value 변화가 큰 영역
- iso-surface topology가 바뀌는 영역
- user-selected ROI가 바뀐 영역

Scientific visualization에서는 ghosting이 단순 미관 문제가 아니라 data 해석 오류가 될 수 있다. 따라서 reactive mask는 visual effect용이 아니라 field reliability signal로 볼 수 있다.

### Sparse voxel / octree / NanoVDB

Sparse voxel renderer에서는 LOD와 streaming 상태가 reactive mask에 직접 영향을 준다.

- coarse brick에서 fine brick으로 교체됨
- missing brick이 resident brick으로 바뀜
- SDF sign이 바뀌어 topology가 변함
- ray marching hit depth가 크게 바뀜
- material id가 변경됨

이런 영역은 previous history를 강하게 누적하면 artifact가 생긴다. Brick id, LOD level, page residency, field version을 reactive mask에 반영할 수 있다.

### Game engine architecture

Game engine에서는 reactive mask가 temporal upscaling과 denoising의 핵심 input이다.

- particle ghosting 감소
- transparency history weight 조절
- emissive flicker 대응
- foliage / hair edge 안정화
- dynamic lighting lag 감소
- RT reflection / GI stale history 방지

면접에서는 reactive mask를 “투명 효과용 마스크”라고 말하기보다, “temporal accumulation에서 history confidence를 조절하는 signal”이라고 설명하는 것이 좋다.

## 6. 머릿속에 남길 질문 3개

1. History validation을 통과한 pixel이라도 reactive mask로 history weight를 낮춰야 하는 경우는 무엇인가?
2. Particle / transparency / alpha-tested geometry에서 motion vector와 depth validation만으로 ghosting을 완전히 막기 어려운 이유는 무엇인가?
3. Sparse voxel이나 CFD visualization에서 LOD change, field value change, topology change를 reactive signal로 사용해야 하는 이유는 무엇인가?

## 7. graphics engineer 면접 질문 1개와 답변

### Q. Reactive Mask는 temporal upscaling이나 denoising에서 어떤 역할을 하나요?

**A.** Reactive Mask는 temporal accumulation에서 previous frame history를 얼마나 신뢰할지 조절하는 mask입니다. Motion vector와 history validation이 previous frame의 같은 surface를 찾는 역할을 한다면, reactive mask는 같은 surface라고 판단되더라도 current frame 변화가 큰 영역에서는 history weight를 낮추도록 도와줍니다. Particle, transparency, alpha-tested foliage, emissive flicker, fast-changing lighting, procedural topology change가 대표적인 예입니다.

Reactive mask가 없으면 temporal upscaler나 denoiser가 previous history를 오래 유지해 ghost trail, smearing, stale lighting artifact가 생길 수 있습니다. 반대로 reactive를 너무 강하게 주면 temporal stability가 약해져 flickering이나 noise가 증가할 수 있습니다. 따라서 reactive mask는 history rejection이 아니라 history confidence를 조절하는 신호로 보고, material property, alpha coverage change, color difference, object/field state change를 함께 사용해 생성하는 것이 좋습니다.

## 8. 포트폴리오 / 커리어 연결

Reactive Mask and Disocclusion Handling은 포트폴리오에서 다음 메시지를 만든다.

> “나는 temporal reconstruction에서 motion vector와 history validation만으로 부족한 영역을 reactive signal로 제어해 ghosting과 stale history를 줄이는 구조를 이해한다.”

네 배경과 연결하면 다음 표현이 좋다.

- Deferred renderer / temporal pipeline에서 reactive mask를 별도 buffer로 관리하는 설계 이해
- CFD / volume visualization에서 scalar gradient와 topology change를 temporal reliability signal로 활용하는 관점
- Sparse voxel renderer에서 brick residency, LOD level, field version을 history confidence에 반영하는 구조
- WebGPU / Vulkan renderer에서 motion vector, disocclusion mask, reactive mask, history buffer를 연결하는 temporal reconstruction pipeline 설명 가능

면접에서는 다음처럼 말할 수 있다.

> “Reactive mask는 previous history를 무조건 reject하는 것이 아니라, history weight를 낮춰 current frame 반응성을 높이는 signal입니다. Transparency, particle, alpha-tested edge, fast-changing lighting, topology change처럼 motion vector만으로 안정적 correspondence를 만들기 어려운 영역에서 중요합니다.”

## 9. 내일 이어서 볼 개념

**Velocity Dilation and Neighborhood Search**

Reactive mask와 disocclusion handling 다음에는 velocity dilation을 보는 것이 자연스럽다. Thin geometry나 disocclusion edge에서 motion vector가 비어 있거나 불안정할 때, 주변 pixel의 velocity를 어떻게 확장해 temporal reprojection을 안정화하는지가 핵심이다.

## 10. 참고 키워드

- Reactive Mask
- Disocclusion Handling
- Temporal Reprojection
- History Weight
- History Validation
- TAA
- Temporal Upscaling
- Ray Tracing Denoising
- Motion Vector
- Alpha-tested Geometry
- Transparency
- Particle Ghosting
- Emissive Flicker
- Neighborhood Clipping
- Velocity Dilation
- Sparse Voxel LOD
- Field-aware Temporal Filtering
- Scientific Visualization
