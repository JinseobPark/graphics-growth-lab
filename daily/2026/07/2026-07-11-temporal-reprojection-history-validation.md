---
title: "Temporal Reprojection and History Validation"
date: "2026-07-11"
category: "Graphics"
tags: ["GPU", "Rendering Pipeline", "Temporal Reprojection", "History Validation", "TAA", "Denoising", "Motion Vector", "Ray Tracing", "Real-time Rendering"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-07-11 - Temporal Reprojection and History Validation

## 1. 오늘의 개념

**Temporal Reprojection and History Validation**은 현재 frame의 pixel이 이전 frame에서 어디에 있었는지 motion vector로 추적하고, 이전 frame의 color / lighting / denoising history를 재사용하되, depth, normal, material, velocity consistency를 검사해 잘못된 history를 버리거나 가중치를 낮추는 rendering 기법이다.

Real-time rendering에서는 한 frame 안에서 충분한 sample을 얻기 어렵다. TAA, ray tracing denoising, SSR, RTGI, RTAO, temporal upscaling은 모두 이전 frame의 정보를 재사용해 시간 방향으로 sample 수를 늘린다.

하지만 previous frame history를 무조건 믿으면 artifact가 생긴다.

- camera movement로 새로 드러난 disocclusion 영역
- object movement로 previous pixel과 current pixel이 다른 surface를 가리키는 경우
- thin geometry / foliage / particle edge에서 잘못된 history 사용
- reflection / shadow / GI처럼 view-dependent result의 mismatch
- motion vector가 부정확한 object

따라서 temporal reprojection의 핵심은 단순히 이전 frame을 가져오는 것이 아니라 다음 질문을 계속 던지는 것이다.

> 이 previous history는 현재 pixel에 재사용해도 되는 정보인가?

## 2. 한 줄 핵심

**Temporal Reprojection and History Validation은 motion vector로 previous frame history를 현재 pixel에 재사용하되, depth/normal/material consistency로 history 신뢰도를 판단해 ghosting과 disocclusion artifact를 줄이는 temporal reconstruction 구조다.**

## 3. 왜 중요한가

Modern real-time renderer는 temporal reuse 없이는 높은 품질을 유지하기 어렵다. Ray tracing은 sample 수가 부족하고, TAA는 sub-pixel jitter를 시간축으로 누적하며, temporal upscaling은 낮은 해상도 render result를 이전 frame 정보와 결합한다.

Temporal reprojection이 중요한 이유는 다음과 같다.

- 적은 sample count로 더 안정적인 image를 만든다.
- ray tracing denoising에서 frame-to-frame noise를 줄인다.
- TAA에서 sub-pixel detail을 누적한다.
- SSR / RT reflection / RTGI에서 temporal stability를 확보한다.
- expensive computation을 여러 frame에 분산해 사용한다.

하지만 validation이 약하면 ghosting이 생기고, validation이 너무 강하면 history가 자주 끊겨 flickering과 noise가 남는다.

Graphics engineer 관점에서 temporal reconstruction은 **quality와 stability, responsiveness 사이의 균형을 잡는 pipeline 설계 문제**다.

## 4. 구현 관점

### 4.1 Reprojection 기본 흐름

현재 pixel의 screen position과 motion vector를 이용해 previous frame의 UV를 계산한다.

```glsl
vec2 currentUV = pixelCoord / viewportSize;
vec2 prevUV = currentUV + motionVector;
vec4 historyColor = texture(historyBuffer, prevUV);
```

Motion vector는 보통 current pixel이 previous frame에서 어디에 있었는지를 나타낸다. Convention은 engine마다 다르기 때문에 `currentUV - prevUV`인지 `prevUV - currentUV`인지 명확히 관리해야 한다.

기본 temporal accumulation은 다음과 같다.

```glsl
vec3 result = mix(currentColor, historyColor, historyWeight);
```

하지만 실제로는 historyWeight를 고정하지 않는다. History validation 결과에 따라 weight를 조절한다.

### 4.2 Depth validation

가장 기본적인 validation은 depth 비교다. Reprojected previous pixel의 depth와 현재 pixel의 depth가 크게 다르면 다른 surface일 가능성이 높다.

```glsl
float depthDiff = abs(currentDepth - prevDepth);
if (depthDiff > depthThreshold)
    historyValid = false;
```

보통 non-linear depth보다는 view-space linear depth를 사용하는 편이 안정적이다. Perspective projection에서는 멀리 있는 depth 차이와 가까운 depth 차이의 의미가 다르기 때문이다.

Depth validation은 disocclusion detection에 매우 중요하다. 새로 드러난 영역은 previous history가 없거나, previous frame에서 다른 object였을 수 있다.

### 4.3 Normal validation

Depth가 비슷해도 normal이 다르면 다른 surface일 수 있다.

```glsl
float normalSimilarity = dot(currentNormal, prevNormal);
if (normalSimilarity < normalThreshold)
    historyValid = false;
```

Normal validation은 edge, crease, curved surface, character silhouette에서 ghosting을 줄이는 데 도움을 준다.

단, normal map normal을 너무 직접적으로 사용하면 작은 texture detail 때문에 history가 과도하게 reject될 수 있다. 경우에 따라 geometric normal과 shading normal을 분리해 사용한다.

### 4.4 Material / object id validation

같은 depth와 normal을 가져도 material이 다르면 history를 섞으면 안 될 수 있다. 예를 들어 같은 평면 위에 금속과 플라스틱이 붙어 있거나, object id가 다른 dynamic object가 지나간 경우다.

Validation input으로 다음을 사용할 수 있다.

- material id
- object id
- primitive id
- meshlet id
- stencil / rendering layer
- roughness bucket

```glsl
if (currentMaterialId != prevMaterialId)
    historyWeight *= 0.0;
```

Material id validation은 ray traced reflection, GI, denoising에서 특히 중요하다.

### 4.5 Velocity / motion vector quality

Motion vector가 정확하지 않으면 reprojection 자체가 틀어진다. Motion vector가 필요한 대상은 camera movement뿐 아니라 object movement, skinned mesh, particle, deformation, vertex animation, scrolling texture 등이다.

자주 생기는 문제는 다음이다.

- static object만 motion vector가 있고 dynamic object는 빠짐
- skinned mesh의 previous transform이 부정확함
- alpha-tested foliage가 unstable motion을 가짐
- particle이 history를 잘못 가져옴
- reflection / transparent object에 motion vector가 애매함

Temporal pipeline에서 motion vector는 단순 velocity buffer가 아니라, history reuse의 주소 계산이다. 주소가 틀리면 validation 이전에 이미 잘못된 history를 가져온다.

### 4.6 History clamping / neighborhood clipping

Validation을 통과해도 history가 현재 frame과 너무 다르면 ghosting이 생길 수 있다. 이를 줄이기 위해 neighborhood clipping 또는 history clamping을 사용한다.

아이디어는 현재 pixel 주변의 color min/max 범위 안으로 history를 제한하는 것이다.

```text
current neighborhood min/max 계산
historyColor를 min/max 범위로 clamp
```

이는 TAA에서 흔히 사용되는 방식이다. Previous history가 현재 neighborhood에서 설명 가능한 값이면 유지하고, 너무 벗어나면 현재 frame에 가깝게 끌어온다.

주의할 점은 clamping이 너무 강하면 temporal accumulation이 약해지고 noise가 남는다. 너무 약하면 ghosting이 남는다.

### 4.7 Disocclusion 처리

Disocclusion은 이전 frame에서는 가려져 있었지만 현재 frame에서 새로 드러난 영역이다. 이 영역은 사용할 history가 없다.

Disocclusion detection은 보통 다음으로 수행한다.

- depth mismatch
- motion vector가 screen 밖을 가리킴
- previous depth가 current depth와 불일치
- reactive mask / disocclusion mask
- object id mismatch

Disoccluded pixel은 historyWeight를 낮추고 current sample을 더 많이 반영해야 한다.

```glsl
if (disoccluded)
    historyWeight = 0.0;
```

Ray tracing denoising에서는 disocclusion 영역이 noisy하게 보일 수 있으므로, spatial filter나 adaptive sample을 함께 사용할 수 있다.

### 4.8 Temporal stability와 responsiveness trade-off

History weight가 높으면 image가 안정적이고 noise가 줄어든다. 하지만 움직임에 둔감해지고 ghosting이 생길 수 있다.

History weight가 낮으면 current frame에 빠르게 반응하지만 noise와 flickering이 남는다.

```text
높은 history weight → 안정적이지만 ghosting 위험
낮은 history weight → responsive하지만 noisy
```

따라서 renderer는 상황별로 history weight를 조절한다.

- camera fast motion: history weight 감소
- disocclusion: history reject
- stable surface: history weight 증가
- high variance: history accumulation 강화
- material boundary: history 제한
- animated object: validation 강화

Temporal reconstruction의 핵심은 단일 공식이 아니라, surface stability와 motion에 따른 adaptive policy다.

## 5. 내 관심 분야와 연결

### CFD / scientific visualization

CFD / scientific visualization에서도 temporal reprojection 사고는 중요하다. Volume rendering, stochastic sampling, ray marching, streamline rendering에서 frame-to-frame stability를 확보할 수 있기 때문이다.

하지만 scientific visualization에서는 ghosting보다 더 중요한 문제가 있다. 잘못된 history가 data interpretation을 왜곡할 수 있다는 점이다.

예를 들어 vortex, shock, interface, material boundary, doping concentration gradient 같은 feature가 움직이거나 camera에 의해 새로 드러날 때, history가 잘못 섞이면 실제 field structure가 흐려져 보일 수 있다.

따라서 scientific visualization에서는 다음 validation이 필요하다.

- depth / normal consistency
- scalar value range consistency
- material / region id consistency
- field gradient consistency
- time step consistency
- ROI mask consistency

### Sparse voxel / octree / NanoVDB

Sparse voxel ray marching에서는 temporal accumulation이 유용하지만, brick LOD 변화와 streaming 상태가 history validation에 영향을 준다.

예를 들어 previous frame에서는 coarse brick을 사용했고 current frame에서는 high-resolution brick이 들어왔다면, 같은 pixel이라도 field value가 달라질 수 있다. 이때 history를 그대로 쓰면 popping이나 ghosting이 생긴다.

Validation input으로 다음을 고려할 수 있다.

- brick id
- LOD level
- material id
- hit depth
- gradient normal
- density / SDF value
- streaming residency state

즉 sparse renderer에서는 history validation이 geometry뿐 아니라 data residency와도 연결된다.

### Game engine architecture

Game engine에서는 temporal reprojection이 거의 모든 modern post-process와 연결된다.

- TAA
- TSR / DLSS-like upscaling
- ray tracing denoising
- SSR temporal accumulation
- RTGI accumulation
- volumetric fog temporal reprojection
- shadow temporal filtering

면접에서는 temporal reprojection을 “motion vector로 이전 frame 가져오기”로 끝내지 말고, “history validation과 clamping으로 ghosting/disocclusion을 관리하는 reconstruction pipeline”으로 설명하는 것이 좋다.

## 6. 머릿속에 남길 질문 3개

1. Temporal reprojection에서 motion vector가 정확해도 depth/normal/material validation이 필요한 이유는 무엇인가?
2. History weight를 높이면 noise는 줄지만 ghosting이 늘어날 수 있는 이유는 무엇인가?
3. Sparse voxel 또는 CFD visualization에서 LOD / field value / material boundary가 history validation에 포함되어야 하는 이유는 무엇인가?

## 7. graphics engineer 면접 질문 1개와 답변

### Q. Temporal Reprojection과 History Validation은 무엇이며, ghosting을 줄이기 위해 어떤 정보를 사용하나요?

**A.** Temporal Reprojection은 current pixel이 previous frame에서 어디에 있었는지 motion vector를 이용해 찾고, previous frame의 history color나 lighting result를 현재 frame에 재사용하는 기법입니다. TAA, ray tracing denoising, SSR, RTGI, temporal upscaling에서 sample 수를 시간축으로 늘리기 위해 사용됩니다.

하지만 previous history를 무조건 사용하면 ghosting과 disocclusion artifact가 생깁니다. 이를 줄이기 위해 depth, normal, material id, object id, primitive id, motion vector consistency를 검사합니다. Depth가 크게 다르면 다른 surface일 가능성이 높고, normal이 다르면 surface orientation이 다르며, material id가 다르면 같은 위치라도 shading 결과를 섞으면 안 될 수 있습니다. 또한 history clamping이나 neighborhood clipping으로 previous color가 현재 neighborhood에서 가능한 값인지 제한합니다. 핵심은 history를 많이 쓰는 것이 아니라, 신뢰할 수 있는 history만 adaptive하게 사용하는 것입니다.

## 8. 포트폴리오 / 커리어 연결

Temporal Reprojection and History Validation은 포트폴리오에서 다음 메시지를 만든다.

> “나는 temporal accumulation을 단순 frame blending이 아니라, motion vector와 G-buffer consistency를 기반으로 한 reconstruction pipeline으로 이해한다.”

네 배경과 연결하면 다음 표현이 좋다.

- Deferred renderer / G-buffer 경험을 TAA와 denoising validation으로 확장 가능
- CFD / volume visualization에서 scalar field와 material boundary를 보존하는 temporal filtering 사고
- Sparse voxel renderer에서 brick id, LOD level, residency state를 history validation에 포함하는 구조 이해
- WebGPU / Vulkan renderer에서 history buffer, motion vector, depth/normal validation pass로 temporal stability 설계 가능

면접에서는 다음처럼 말할 수 있다.

> “Temporal reprojection은 previous frame history를 motion vector로 가져오는 단계이고, history validation은 그 history가 현재 pixel과 같은 surface/material/data를 의미하는지 검사하는 단계입니다. Ghosting을 줄이려면 depth, normal, material id, object id, neighborhood clipping, disocclusion detection을 함께 사용해야 합니다.”

## 9. 내일 이어서 볼 개념

**Motion Vector Generation for Dynamic Geometry**

Temporal reprojection을 이해한 다음에는 motion vector generation으로 이어지는 것이 자연스럽다. Static camera motion뿐 아니라 skinned mesh, particle, vertex animation, deformation, procedural geometry에서 previous position을 어떻게 만들고 velocity buffer를 채우는지가 temporal stability의 핵심이다.

## 10. 참고 키워드

- Temporal Reprojection
- History Validation
- Motion Vector
- Velocity Buffer
- TAA
- Temporal Accumulation
- History Clamping
- Neighborhood Clipping
- Disocclusion
- Ghosting
- Depth Validation
- Normal Validation
- Material ID
- Object ID
- Ray Tracing Denoising
- Temporal Upscaling
- SSR
- RTGI
- Sparse Voxel
- Scientific Visualization
