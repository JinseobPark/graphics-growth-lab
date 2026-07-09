---
title: "Denoising Inputs for Ray Tracing"
date: "2026-07-09"
category: "Graphics"
tags: ["GPU", "Rendering Pipeline", "Ray Tracing", "Denoising", "Temporal Accumulation", "G-buffer", "Motion Vector", "Path Tracing", "Real-time Rendering"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-07-09 - Denoising Inputs for Ray Tracing

## 1. 오늘의 개념

**Denoising Inputs for Ray Tracing**은 ray tracing 또는 path tracing 결과를 실시간으로 사용하기 위해, noisy radiance buffer만 후처리하는 것이 아니라 depth, normal, albedo, roughness, motion vector 같은 auxiliary buffer를 함께 사용해 sample noise를 안정적으로 제거하는 구조다.

실시간 ray tracing에서는 pixel당 ray 수가 제한적이다. Path tracing에서 pixel당 수백~수천 sample을 쓸 수 있으면 noise가 줄어들지만, real-time renderer에서는 1~4 spp(samples per pixel) 수준으로 처리해야 하는 경우가 많다. 이때 raw ray traced result는 매우 noisy하다.

단순 blur를 적용하면 noise는 줄어들지만 edge, detail, material boundary, shadow contact, reflection feature까지 뭉개진다. 그래서 denoiser는 radiance 외에도 scene structure를 알려주는 input을 사용한다.

대표적인 denoising input은 다음과 같다.

- noisy radiance / illumination buffer
- depth
- normal
- albedo / base color
- roughness / material id
- motion vector
- previous frame history
- hit distance
- variance / sample count
- object id / primitive id

핵심 변화는 다음이다.

> Ray tracing denoising은 단순 blur가 아니라, geometry/material/temporal 정보를 이용해 어디까지 섞고 어디서 멈출지 판단하는 reconstruction 문제다.

## 2. 한 줄 핵심

**Ray tracing denoising input은 noisy radiance를 depth, normal, albedo, roughness, motion vector, history와 결합해 edge와 material detail을 보존하면서 부족한 sample을 재구성하기 위한 auxiliary signal이다.**

## 3. 왜 중요한가

Real-time ray tracing의 핵심 병목은 sample count다. 완전한 path tracing은 많은 sample이 필요하지만, 게임이나 interactive visualization에서는 frame time이 제한되어 있다. 따라서 적은 sample을 쏘고, denoiser가 결과를 안정화한다.

Denoising input이 중요한 이유는 다음과 같다.

- Depth는 geometry edge를 알려준다.
- Normal은 surface orientation boundary를 알려준다.
- Albedo는 lighting과 texture color를 분리하는 데 도움을 준다.
- Roughness는 reflection blur와 specular stability에 영향을 준다.
- Motion vector는 temporal history reprojection에 필요하다.
- Hit distance는 shadow / AO / reflection denoising radius를 조절하는 데 도움을 준다.
- Variance는 어느 pixel이 더 noisy한지 알려준다.

Graphics engineer 관점에서 denoising은 ray tracing output을 예쁘게 만드는 후처리가 아니라, **renderer가 어떤 auxiliary buffer를 생산하고 어떻게 temporal/spatial reconstruction에 연결할지 결정하는 pipeline 설계**다.

## 4. 구현 관점

### 4.1 Raw ray traced buffer의 문제

Ray tracing result는 보통 다음과 같은 buffer로 나온다.

```text
rayTracedRadiance[pixel] = noisy lighting / reflection / shadow / GI result
```

Pixel당 sample 수가 적으면 다음 문제가 생긴다.

- random noise
- firefly
- shadow flickering
- reflection instability
- indirect lighting blotchiness
- temporal shimmering

단순 box blur나 Gaussian blur를 적용하면 noise는 줄어들지만 geometry edge와 texture detail이 같이 뭉개진다. 따라서 denoiser는 주변 pixel을 섞을 때 “같은 surface인지”를 판단해야 한다.

### 4.2 Depth input

Depth는 가장 기본적인 edge-stopping signal이다. 주변 pixel과 depth 차이가 크면 서로 다른 surface일 가능성이 높다.

Denoiser는 spatial filter에서 다음처럼 판단할 수 있다.

```text
if abs(depthCenter - depthNeighbor) > threshold:
    neighbor weight 감소
```

Depth를 사용하면 foreground object와 background object의 lighting이 섞이는 것을 줄일 수 있다. 특히 contact shadow, AO, reflection denoising에서 depth edge 보존이 중요하다.

다만 depth는 non-linear depth일 수 있으므로 view-space linear depth로 변환해 사용하는 편이 안정적이다.

### 4.3 Normal input

Normal은 surface orientation을 알려준다. Depth가 비슷해도 normal이 크게 다르면 같은 평면이 아닐 수 있다.

```text
normalWeight = max(dot(normalCenter, normalNeighbor), 0)^k
```

Normal weight를 사용하면 모서리, creased surface, curved geometry의 detail을 보존할 수 있다. Ray traced GI나 diffuse indirect denoising에서는 normal similarity가 특히 중요하다.

주의할 점은 normal map normal과 geometry normal 중 무엇을 사용할지다. Denoising에는 너무 noisy하거나 high-frequency normal map을 그대로 쓰면 history rejection이 과도하게 발생할 수 있다. 경우에 따라 geometric normal과 shading normal을 구분해 사용한다.

### 4.4 Albedo input

Albedo는 lighting과 surface color를 분리하는 데 도움을 준다. Path traced color는 대략 다음처럼 볼 수 있다.

```text
radiance ≈ albedo * illumination
```

Denoiser가 radiance를 직접 필터링하면 texture pattern이 noise처럼 취급되어 뭉개질 수 있다. 그래서 diffuse denoising에서는 albedo를 제거한 illumination만 denoise하고, 마지막에 albedo를 다시 곱하는 방식이 사용될 수 있다.

```text
illumination = noisyRadiance / max(albedo, epsilon)
denoisedIllumination = Denoise(illumination)
final = denoisedIllumination * albedo
```

이 방식은 texture detail을 보존하면서 lighting noise를 줄이는 데 유리하다.

### 4.5 Roughness / material input

Roughness는 reflection denoising에서 매우 중요하다. Rough surface의 reflection은 넓게 blur되어도 자연스럽지만, mirror-like surface는 sharp reflection을 유지해야 한다.

Denoiser는 roughness에 따라 filter radius를 조절할 수 있다.

```text
low roughness  → 작은 filter radius
high roughness → 큰 filter radius
```

Material id나 object id도 중요하다. 같은 depth와 normal을 가져도 서로 다른 material이면 radiance를 섞으면 안 되는 경우가 있다.

예를 들어 금속 표면과 플라스틱 표면이 같은 평면에 붙어 있을 때, material boundary를 넘어 reflection result를 섞으면 artifact가 생긴다.

### 4.6 Motion vector와 temporal reprojection

Real-time denoising은 현재 frame만 보지 않는다. 이전 frame history를 재사용한다. 이를 위해 motion vector가 필요하다.

Temporal reprojection 흐름은 다음과 같다.

1. 현재 pixel의 motion vector로 previous frame 위치를 찾는다.
2. Previous history buffer에서 radiance를 sample한다.
3. Depth / normal / material consistency를 검사한다.
4. 유효하면 history와 current sample을 accumulate한다.
5. 유효하지 않으면 history를 reject하거나 weight를 낮춘다.

```text
prevUV = currentUV + motionVector
history = SampleHistory(prevUV)
```

Motion vector가 부정확하면 ghosting, smearing, lag artifact가 생긴다. 특히 reflection, transparent object, disocclusion 영역에서는 history validation이 중요하다.

### 4.7 Hit distance와 variance

Ray traced shadow, AO, reflection에서는 hit distance가 filter radius를 결정하는 데 유용하다.

예를 들어 가까운 occluder에 의한 contact shadow는 sharp해야 하고, 먼 occluder에 의한 soft shadow는 더 넓게 blur할 수 있다.

Variance는 pixel이 얼마나 noisy한지 나타낸다. Variance가 높은 영역은 더 강한 denoising이나 더 많은 temporal accumulation이 필요하다. Variance가 낮은 영역은 detail을 보존하기 위해 filter를 약하게 적용할 수 있다.

### 4.8 Spatial + temporal denoising 구조

실시간 denoiser는 보통 temporal accumulation과 spatial filtering을 결합한다.

대표 흐름은 다음과 같다.

```text
Noisy ray traced result
→ Temporal reprojection / accumulation
→ History validation
→ Spatial edge-aware filter
→ Optional second pass / à-trous filter
→ Final compositing
```

Spatial filter는 depth/normal/albedo/roughness를 사용해 edge-aware하게 작동한다. Temporal pass는 motion vector와 history를 사용해 sample count를 시간축으로 늘린다.

즉 denoiser는 한 frame의 blur가 아니라, frame-to-frame reconstruction system이다.

## 5. 내 관심 분야와 연결

### CFD / scientific visualization

Scientific visualization에서 path tracing denoising을 직접 적용하지 않더라도, auxiliary-buffer 기반 reconstruction 사고는 중요하다.

예를 들어 volume rendering이나 noisy sampling 기반 visualization에서도 다음 input이 유용할 수 있다.

- depth: surface / volume boundary
- normal or gradient: iso-surface orientation
- scalar value: field-aware filtering
- material id: layer / region boundary
- motion vector: temporal interaction 안정화
- variance: adaptive sampling 필요 영역 판단

CFD나 semiconductor visualization에서는 단순 image smoothing이 data interpretation을 왜곡할 수 있다. 따라서 denoising을 한다면 geometry edge뿐 아니라 field boundary와 scalar gradient를 보존해야 한다.

### Sparse voxel / octree / NanoVDB

Sparse voxel ray marching이나 SDF tracing에서도 noisy result가 생길 수 있다. 특히 cone tracing, stochastic sampling, path traced volume lighting을 사용하면 denoising이 필요하다.

이때 voxel renderer의 auxiliary input은 다음과 같다.

- hit depth
- gradient normal
- material id
- brick id / object id
- density / SDF value
- motion vector
- transmittance / hit distance

Brick boundary나 LOD transition에서 history validation이 잘못되면 ghosting이나 flickering이 생길 수 있다. 따라서 denoising input은 sparse data structure의 id와도 연결된다.

### Game engine architecture

Game engine의 real-time ray tracing은 denoiser 없이는 거의 성립하기 어렵다. Reflection, shadow, ambient occlusion, global illumination 모두 적은 ray count로 noisy result를 만들고 denoiser가 이를 안정화한다.

- RT reflection denoising
- RT shadow denoising
- RTAO denoising
- RTGI denoising
- ReSTIR / reservoir 기반 sampling 후 denoising
- Temporal accumulation + spatial filter

면접에서는 “denoising은 blur”라고 말하기보다, “G-buffer와 motion vector를 활용한 temporal/spatial reconstruction”이라고 말하는 것이 좋다.

## 6. 머릿속에 남길 질문 3개

1. Ray tracing denoiser가 noisy radiance만 보지 않고 depth, normal, albedo, roughness를 함께 사용하는 이유는 무엇인가?
2. Motion vector가 temporal denoising에서 중요하지만 ghosting artifact를 만들 수 있는 이유는 무엇인가?
3. CFD / volume visualization에서 denoising을 적용할 때 scalar gradient나 material boundary를 보존해야 하는 이유는 무엇인가?

## 7. graphics engineer 면접 질문 1개와 답변

### Q. Real-time ray tracing denoising에서 어떤 auxiliary buffer가 필요하며, 각각 어떤 역할을 하나요?

**A.** Real-time ray tracing은 pixel당 ray 수가 적기 때문에 raw result가 noisy합니다. Denoiser는 noisy radiance만 blur하지 않고 depth, normal, albedo, roughness, motion vector, hit distance, variance 같은 auxiliary buffer를 사용합니다. Depth는 geometry edge를 보존하고, normal은 surface orientation이 다른 영역이 섞이지 않도록 도와줍니다. Albedo는 texture color와 illumination을 분리해 texture detail이 blur되는 것을 줄입니다. Roughness는 reflection denoising에서 filter radius를 조절하는 데 중요합니다. Motion vector는 previous frame history를 현재 frame으로 reprojection해 temporal accumulation을 가능하게 합니다.

이 구조의 장점은 적은 sample count로도 안정적인 ray traced shadow, reflection, GI를 만들 수 있다는 점입니다. 단점은 motion vector나 history validation이 부정확하면 ghosting, smearing, flickering이 생길 수 있고, auxiliary buffer 자체의 precision이나 material boundary 처리도 신중해야 한다는 점입니다.

## 8. 포트폴리오 / 커리어 연결

Denoising Inputs for Ray Tracing은 포트폴리오에서 다음 메시지를 만든다.

> “나는 ray tracing denoising을 단순 blur가 아니라, G-buffer와 temporal history를 이용한 reconstruction pipeline으로 이해한다.”

네 배경과 연결하면 다음 표현이 좋다.

- Deferred renderer / G-buffer 경험을 ray tracing denoising input 설계로 확장 가능
- Sparse voxel / volume ray marching에서 depth, gradient normal, material id 기반 filtering 사고
- CFD / scientific visualization에서 scalar gradient와 material boundary를 보존하는 field-aware denoising 관점
- Vulkan / DX12 / WebGPU renderer에서 motion vector, history buffer, edge-aware filter pipeline으로 연결 가능

면접에서는 다음처럼 말할 수 있다.

> “Real-time ray tracing denoiser는 noisy radiance만 필터링하는 것이 아니라 depth, normal, albedo, roughness, motion vector 같은 auxiliary signal을 이용해 temporal/spatial reconstruction을 수행합니다. 핵심은 noise를 줄이면서 geometry edge, material boundary, texture detail, temporal stability를 유지하는 것입니다.”

## 9. 내일 이어서 볼 개념

**Temporal Reprojection and History Validation**

Denoising input을 이해한 다음에는 temporal reprojection과 history validation을 보는 것이 자연스럽다. Motion vector로 previous frame history를 가져오되, depth/normal/material consistency를 검사해 ghosting과 disocclusion artifact를 줄이는 구조가 핵심이다.

## 10. 참고 키워드

- Ray Tracing Denoising
- Denoising Input
- Auxiliary Buffer
- G-buffer
- Depth
- Normal
- Albedo
- Roughness
- Motion Vector
- Temporal Accumulation
- Temporal Reprojection
- History Validation
- Variance Estimation
- Hit Distance
- Edge-aware Filter
- à-trous Filter
- RT Reflection
- RTGI
- RTAO
- Scientific Visualization
