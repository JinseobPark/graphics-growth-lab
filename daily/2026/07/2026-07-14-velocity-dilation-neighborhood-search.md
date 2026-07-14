---
title: "Velocity Dilation and Neighborhood Search"
date: "2026-07-14"
category: "Graphics"
tags: ["GPU", "Rendering Pipeline", "Motion Vector", "Velocity Buffer", "Temporal Reprojection", "TAA", "Temporal Upscaling", "Disocclusion", "Compute Shader", "Real-time Rendering"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-07-14 - Velocity Dilation and Neighborhood Search

## 1. 오늘의 개념

**Velocity Dilation and Neighborhood Search**는 motion vector가 비어 있거나 신뢰하기 어려운 pixel에 대해 주변의 유효한 velocity를 탐색하고 확장해, temporal reprojection이 사용할 수 있는 안정적인 대응점(correspondence)을 만드는 과정이다.

Temporal rendering pipeline은 보통 현재 pixel의 motion vector를 이용해 previous frame의 history 위치를 찾는다. 그러나 화면 경계, thin geometry, moving object silhouette, particle, alpha-tested surface, dynamic-resolution boundary에서는 현재 pixel에 적절한 motion vector가 존재하지 않거나 잘못된 surface의 vector가 기록될 수 있다.

대표적인 문제 상황은 다음과 같다.

- 움직이는 foreground object 뒤에서 새로 드러난 background pixel
- low-resolution velocity buffer에서 coverage가 사라진 thin geometry
- alpha-tested foliage, hair card, fence의 불안정한 pixel coverage
- particle spawn / death와 transparent layer
- camera jitter와 rasterization 차이로 생긴 edge mismatch
- temporal upscaling에서 input pixel과 output pixel의 footprint 차이
- sparse voxel / marching cubes surface의 LOD 또는 topology 변화
- volume ray marching에서 frame마다 달라지는 first-hit depth

Velocity dilation은 주변 pixel의 velocity를 단순 복사하는 작업처럼 보이지만, 실제 핵심은 다음 질문에 있다.

> 현재 pixel이 temporal reprojection에 사용할 수 있는 가장 합리적인 주변 surface의 motion은 무엇인가?

따라서 dilation은 morphological expansion이면서 동시에 **surface-aware neighborhood selection** 문제다.

## 2. 한 줄 핵심

**Velocity Dilation은 불완전한 motion vector 주변에서 depth, normal, validity, motion magnitude를 이용해 가장 적합한 velocity를 선택하고 확장하여 temporal reprojection의 주소를 안정화하는 과정이다.**

## 3. 왜 중요한가

Temporal reprojection의 첫 단계는 previous history를 어디서 읽을지 결정하는 것이다. Motion vector가 비어 있거나 edge에서 잘못된 vector를 사용하면 이후의 history validation, reactive mask, neighborhood clipping이 아무리 잘 설계되어 있어도 잘못된 위치에서 history를 가져오게 된다.

Velocity dilation이 중요한 이유는 다음과 같다.

- object silhouette 주변의 velocity hole을 줄인다.
- thin geometry와 alpha-tested edge의 temporal instability를 완화한다.
- dynamic resolution과 temporal upscaling에서 input/output footprint 차이를 보완한다.
- disocclusion 경계에서 주변 surface의 motion 후보를 확보한다.
- motion blur, TAA, temporal denoising이 동일한 velocity contract를 공유하도록 만든다.
- low-resolution effect buffer의 reprojection 품질을 높인다.
- sparse voxel / volume renderer에서 불안정한 hit point 주변의 history lookup을 보조한다.

다만 dilation은 invalid history를 valid하게 만드는 기술이 아니다. 주변 velocity를 확보한 뒤에도 depth, normal, material, object identity를 이용한 history validation이 필요하다.

즉 역할을 구분하면 다음과 같다.

```text
Velocity dilation: previous history를 찾을 후보 주소를 만든다.
History validation: 그 주소의 history가 같은 surface인지 검사한다.
Reactive mask: 같은 surface여도 history를 얼마나 믿을지 조절한다.
```

## 4. 구현 관점

### 4.1 Velocity hole은 왜 생기는가

Velocity buffer는 보통 geometry pass 또는 별도의 velocity pass에서 rasterization된다. 이때 current frame에서 보이는 모든 pixel이 항상 안정적인 velocity를 갖는 것은 아니다.

주요 원인은 다음과 같다.

- foreground edge가 sub-pixel 단위로 이동해 coverage가 바뀐다.
- current frame과 previous frame의 visibility가 다르다.
- velocity buffer가 color buffer보다 낮은 resolution으로 생성된다.
- transparent object가 velocity buffer에 기록되지 않는다.
- alpha test 결과가 frame마다 달라진다.
- newly spawned primitive는 previous position이 없다.
- procedural geometry가 topology를 바꾼다.
- ray-marched surface는 rasterized primitive처럼 안정적인 primitive identity가 없다.

이 때문에 velocity buffer에는 `zero velocity`, invalid flag, unrelated background motion, foreground motion이 섞일 수 있다.

### 4.2 Dilation pass의 위치

일반적인 pipeline에서는 다음 순서가 자연스럽다.

```text
Depth / G-buffer / Velocity 생성
        ↓
Velocity validity 판정
        ↓
Velocity dilation + neighborhood search
        ↓
Temporal reprojection
        ↓
History validation / clipping
        ↓
Temporal accumulation
```

Velocity dilation은 compute shader로 처리하기 좋다. 입력은 보통 다음과 같다.

- current depth
- motion vector
- velocity validity 또는 confidence
- optional normal
- optional object / material id
- optional reactive / transparency information

출력은 다음 중 일부가 될 수 있다.

- dilated velocity
- dilated depth
- source pixel coordinate
- confidence
- disocclusion hint

### 4.3 가장 단순한 neighborhood search

가장 단순한 방식은 현재 pixel 주변의 `3×3` 또는 `5×5` 영역에서 유효한 motion vector를 찾는 것이다.

```glsl
vec2 selectedVelocity = vec2(0.0);
float selectedScore = INF;

for each neighbor:
    if (!IsValidVelocity(neighbor))
        continue;

    float score = EvaluateCandidate(neighbor);

    if (score < selectedScore) {
        selectedScore = score;
        selectedVelocity = neighbor.velocity;
    }
```

중요한 것은 `EvaluateCandidate`가 무엇을 우선하는가이다. 단순히 가장 가까운 pixel을 선택할 수도 있지만, silhouette 경계에서는 screen distance보다 surface compatibility가 더 중요할 수 있다.

### 4.4 Nearest-depth selection

많이 사용하는 기준 중 하나는 neighborhood에서 camera에 가장 가까운 surface의 velocity를 선택하는 것이다.

Foreground object의 silhouette 주변에서는 current color pixel이 foreground coverage에 속할 가능성이 높기 때문에 closest depth의 velocity를 확장하는 방식이 안정적인 경우가 많다.

```text
Candidate score = camera-space depth
Choose nearest valid surface
```

그러나 depth convention을 주의해야 한다.

- 일반 depth: 작은 값이 camera에 가까움
- reversed-Z: 큰 값이 camera에 가까움
- linear depth: 일반적으로 작은 값이 가까움

Reversed-Z renderer에서 `min depth`를 무조건 선택하면 가장 먼 surface를 선택하는 반대 결과가 발생할 수 있다.

Nearest-depth 방식의 장점은 foreground velocity를 silhouette 밖으로 확장해 thin edge를 보존하기 쉽다는 점이다. 단점은 background pixel까지 foreground motion이 과도하게 번질 수 있다는 점이다.

### 4.5 Maximum-magnitude selection

또 다른 heuristic은 neighborhood에서 motion magnitude가 가장 큰 velocity를 선택하는 것이다.

```text
Choose candidate with max length(velocity)
```

이 방식은 빠르게 움직이는 foreground object의 motion을 edge 주변에 보존하는 데 유리하다. Motion blur나 temporal upscaling에서 빠른 object가 zero velocity로 사라지는 문제를 줄일 수 있다.

하지만 camera motion이 크거나 background가 빠르게 움직이는 장면에서는 잘못된 vector가 선택될 수 있다. 또한 작은 fast-moving particle이 주변 pixel 전체에 velocity를 퍼뜨리는 문제도 생길 수 있다.

따라서 magnitude만 사용하기보다 depth compatibility, object identity, validity confidence와 함께 사용하는 편이 안전하다.

### 4.6 Surface-aware candidate score

실무적인 neighborhood search는 여러 정보를 결합해 candidate를 평가한다.

예시는 다음과 같다.

```text
score =
    depthDifferenceWeight   × |currentDepth - candidateDepth|
  + normalDifferenceWeight  × (1 - dot(currentNormal, candidateNormal))
  + screenDistanceWeight    × pixelDistance
  + confidencePenalty
  + objectMismatchPenalty
```

현재 pixel 자체가 valid depth를 갖지 않는 disocclusion 영역이라면 current depth와 비교하기 어렵다. 이 경우 주변 후보의 depth ordering, velocity magnitude, edge direction을 이용하거나 dilation 결과를 단지 reprojection candidate로만 사용하고 이후 validation에서 강하게 제한할 수 있다.

핵심은 dilation 결과를 절대적인 정답으로 다루지 않는 것이다.

### 4.7 Dilated depth를 함께 저장하는 이유

Velocity만 확장하면 그 vector가 어느 surface에서 왔는지 잃어버린다. 따라서 많은 temporal pipeline은 dilated velocity와 함께 candidate의 depth도 저장한다.

```text
DilatedVelocity = selected neighbor velocity
DilatedDepth    = selected neighbor depth
```

Reprojection 이후에는 selected source surface의 depth를 previous frame depth와 비교해 history validity를 판단할 수 있다.

Dilated depth가 있으면 다음을 구분하기 쉬워진다.

- foreground velocity가 background로 번진 경우
- background vector가 foreground edge에 선택된 경우
- newly revealed region에서 history가 존재하지 않는 경우
- temporal upscaling 중 low-resolution depth footprint가 달라진 경우

### 4.8 Kernel 크기와 품질의 관계

`3×3` kernel은 비용이 낮고 local edge를 보완하기 좋다. 하지만 빠른 motion이나 low-resolution input에서는 hole이 kernel보다 클 수 있다.

`5×5`, `7×7`로 확장하면 더 많은 candidate를 찾을 수 있지만 다음 비용이 증가한다.

- texture load 수 증가
- cache pressure 증가
- foreground velocity bleeding 증가
- unrelated surface 선택 가능성 증가
- wave / workgroup 내부 synchronization 비용 증가

따라서 kernel 크기를 무조건 키우기보다 다음 방식이 사용될 수 있다.

- 3×3 local search 후 invalid pixel만 추가 pass
- cross-shaped 또는 sparse tap pattern
- hierarchical search
- depth edge 방향을 따른 anisotropic search
- confidence가 낮은 pixel만 넓은 kernel 사용

### 4.9 Compute shader와 memory layout

Velocity dilation은 neighborhood read가 많은 pass이므로 memory access pattern이 중요하다.

대표적인 buffer format은 다음과 같다.

- motion vector: `RG16F`, `RG16_SNORM`, `RG32F`
- depth: `R32F`, `D32F`, linearized `R16F/R32F`
- validity/confidence: `R8_UNORM`, bit mask, packed metadata
- output: dilated velocity + depth 또는 packed structure

Compute shader에서는 workgroup tile과 halo 영역을 shared memory에 올려 texture read를 줄일 수 있다.

예를 들어 `8×8` output tile에 1-pixel halo를 포함하면 `10×10` 입력을 group shared memory에 적재한 뒤 각 thread가 `3×3` neighborhood를 재사용할 수 있다.

```text
Global memory → shared tile + halo → neighborhood search → output
```

주의할 점은 다음과 같다.

- depth와 velocity가 서로 다른 resolution인지 확인한다.
- UV-space velocity인지 pixel-space velocity인지 convention을 통일한다.
- dynamic resolution scale을 motion vector에 반영한다.
- current / previous projection jitter convention을 일관되게 유지한다.
- invalid velocity를 `0`과 동일시하지 않는다. 실제 static pixel도 zero velocity일 수 있다.
- branch divergence보다 불필요한 global memory load가 더 큰 병목일 수 있다.

### 4.10 Input resolution과 output resolution

Temporal upscaler에서는 input color, velocity, depth가 render resolution에 있고 output history는 display resolution에 있을 수 있다.

예를 들어 render resolution이 `1920×1080`, output이 `3840×2160`이라면 motion vector가 다음 중 어떤 단위인지 명확해야 한다.

- normalized UV delta
- input pixel delta
- output pixel delta
- NDC delta

Dilation 자체는 input resolution에서 수행되더라도 reprojection에서는 output history 좌표로 변환되어야 한다.

```text
motionOutputPixels = motionInputPixels × outputResolution / inputResolution
```

UV 단위라면 resolution scale과 독립적일 수 있지만, sampling convention과 texture coordinate orientation을 여전히 맞춰야 한다.

### 4.11 Over-dilation의 위험

Velocity dilation은 hole을 줄이는 대신 wrong-surface motion을 퍼뜨릴 수 있다.

대표적인 artifact는 다음과 같다.

- foreground motion이 background로 번져 background history가 끌려옴
- 빠른 particle velocity가 주변 opaque surface로 확장됨
- thin geometry 주변이 과도하게 smear됨
- disocclusion 영역이 이전 foreground history를 재사용함
- object boundary에서 double image가 생김

이를 줄이는 방법은 다음과 같다.

- depth threshold 적용
- normal threshold 적용
- object / material id compatibility 확인
- dilation confidence 저장
- dilated pixel의 history weight 제한
- disocclusion mask와 함께 사용
- reactive mask로 accumulation weight 감소
- source distance가 멀수록 confidence 감소

즉 dilation은 history를 살리기 위한 장치지만, dilation된 velocity일수록 더 보수적인 accumulation 정책이 필요하다.

### 4.12 Neighborhood search와 velocity dilation의 차이

두 용어는 함께 사용되지만 초점이 조금 다르다.

- **Velocity dilation**: 유효 velocity 영역을 주변 pixel로 확장하는 결과 중심 표현
- **Neighborhood search**: 주변 후보 중 어떤 velocity가 현재 pixel에 적합한지 선택하는 과정 중심 표현

단순 dilation은 nearest 또는 max-magnitude 같은 고정 rule만 사용할 수 있다. Advanced temporal pipeline에서는 neighborhood search를 통해 surface-aware candidate를 고르고, 선택 결과와 confidence를 함께 출력한다.

## 5. 내 관심 분야와 연결

### Real-time rendering / game engine

Game engine에서 velocity dilation은 다음 기능의 공통 기반이 된다.

- TAA / TAAU
- temporal super resolution
- motion blur
- ray tracing denoising
- screen-space reflection history
- volumetric fog / cloud reprojection
- particle temporal filtering

Renderer architecture 관점에서는 각 feature가 개별 velocity dilation을 구현하기보다, engine-wide motion convention과 validity metadata를 정의하는 것이 중요하다.

예를 들어 opaque velocity, transparent velocity, particle velocity, camera-only velocity를 어떻게 합성할지 정해야 한다. 또한 motion blur가 원하는 dilation과 temporal upscaler가 원하는 dilation 정책이 다를 수 있으므로, 하나의 buffer를 공유하더라도 confidence와 source type을 함께 관리하는 편이 좋다.

### CFD / scientific visualization

CFD visualization에서는 velocity라는 단어가 screen-space motion vector와 simulation velocity field를 동시에 의미할 수 있다. 두 개념을 분리해야 한다.

- simulation velocity: fluid의 physical vector field
- screen-space velocity: rendered sample이 frame 사이에 이동한 화면 좌표 변화

Particle trace는 particle id별 previous position이 존재하므로 screen-space velocity를 비교적 안정적으로 만들 수 있다. 반면 iso-surface나 volume rendering은 field 변화 때문에 visible surface가 생성·소멸하므로 neighborhood dilation만으로 correspondence를 보장하기 어렵다.

Scientific visualization에서는 잘못된 temporal history가 실제 scalar boundary나 vortex structure를 흐리게 만들 수 있다. 따라서 dilation된 velocity는 field gradient, time-step difference, iso-value crossing, material boundary를 이용해 더 강하게 검증해야 한다.

### Sparse voxel / octree / NanoVDB

Sparse representation에서는 다음 변화가 velocity hole과 invalid correspondence를 만든다.

- coarse brick에서 fine brick으로 LOD 전환
- non-resident brick이 새로 로드됨
- SDF sign 또는 density field 변화
- ray marching hit depth 변화
- marching cubes topology 변화

이 경우 neighborhood search candidate score에 다음 metadata를 포함할 수 있다.

- brick id
- LOD level
- residency generation
- field version
- material id
- ray hit confidence

같은 screen neighborhood라도 서로 다른 brick generation에서 나온 velocity라면 temporal history를 강하게 누적하지 않는 것이 안전하다.

### Vulkan / WebGPU / CUDA compute

Velocity dilation은 API보다 data layout과 synchronization 설계가 핵심인 compute workload다.

- Vulkan: storage image, sampled image, pipeline barrier, subgroup operation
- WebGPU: storage texture, workgroup memory, textureLoad, dispatch sizing
- CUDA: texture object, shared memory tile, coalesced write
- DirectX: UAV/SRV, group shared memory, wave intrinsic
- Metal: threadgroup memory, texture read, simdgroup operation

API가 달라도 공통 문제는 neighborhood data reuse, edge handling, velocity convention, depth convention, confidence propagation이다.

## 6. 머릿속에 남길 질문 3개

1. Velocity dilation이 motion vector hole을 줄이면서도 foreground velocity가 background로 과도하게 번지는 문제를 어떻게 제어할 수 있는가?
2. Nearest-depth selection과 maximum-magnitude selection은 각각 어떤 장면에서 유리하며, reversed-Z에서는 depth 비교가 어떻게 달라지는가?
3. CFD volume이나 sparse voxel surface에서 dilation된 screen-space velocity를 그대로 신뢰하기 어려운 이유는 무엇인가?

## 7. graphics engineer 면접 질문 1개와 답변

### Q. Temporal reprojection에서 velocity dilation이 필요한 이유와 구현 시 주의할 점을 설명해 주세요.

**A.** Velocity dilation은 object silhouette, thin geometry, alpha-tested edge, low-resolution velocity buffer처럼 motion vector가 비어 있거나 불안정한 pixel에서 주변의 유효한 velocity를 찾아 확장하는 과정입니다. Temporal reprojection은 motion vector를 이용해 previous history sample 위치를 계산하므로, velocity hole이 있으면 zero motion이나 unrelated surface motion을 사용해 ghosting과 smearing이 발생할 수 있습니다.

구현은 보통 compute shader에서 `3×3` 또는 `5×5` neighborhood를 탐색하고, nearest depth, maximum motion magnitude, depth/normal compatibility, validity confidence 등을 이용해 candidate를 선택합니다. 선택된 velocity와 함께 source depth 또는 confidence를 저장하면 이후 history validation에 사용할 수 있습니다.

주의할 점은 dilation이 wrong history를 valid하게 만드는 것은 아니라는 것입니다. Foreground velocity가 background로 번질 수 있으므로 depth threshold, object identity, disocclusion mask, reactive mask를 함께 사용해야 합니다. 또한 reversed-Z depth convention, UV/pixel/NDC velocity 단위, dynamic resolution scale, projection jitter convention을 renderer 전체에서 일관되게 관리해야 합니다.

## 8. 포트폴리오 / 커리어 연결

Velocity Dilation and Neighborhood Search는 temporal rendering pipeline을 단순 효과가 아니라 **data correspondence system**으로 이해하고 있음을 보여주기 좋은 주제다.

포트폴리오 문서에서는 다음 흐름으로 설명할 수 있다.

```text
Problem
- Dynamic geometry edge에서 velocity hole과 wrong-surface motion 발생

Pipeline
- Velocity validity 생성
- Surface-aware neighborhood search
- Dilated velocity/depth/confidence 출력
- History validation 및 reactive mask와 연동

Engineering decisions
- RG16F velocity와 depth format 선택
- 3×3 shared-memory tile search
- reversed-Z 지원
- input/output resolution 단위 변환
- over-dilation 방지를 위한 confidence 정책

Evaluation
- silhouette ghosting
- thin geometry stability
- disocclusion artifact
- GPU pass cost와 bandwidth
```

Graphics engineer 면접이나 경력 기술에서는 다음처럼 연결할 수 있다.

> “Temporal artifact를 단순히 blending weight 문제로 보지 않고, motion vector 생성 → dilation → reprojection → validation → accumulation으로 분해해 원인을 추적할 수 있습니다. 특히 procedural geometry, sparse voxel, scientific visualization처럼 stable primitive correspondence가 약한 renderer에서는 velocity confidence와 field-aware validation을 함께 설계합니다.”

이는 game engine rendering, temporal upscaling, ray tracing denoising뿐 아니라 CFD post-processing과 semiconductor 3D visualization에서도 차별화되는 관점이다.

## 9. 내일 이어서 볼 개념

**Neighborhood Clamping and Variance Clipping**

Velocity dilation으로 previous history의 candidate 위치를 찾았다면, 다음 단계는 reprojected history color가 current neighborhood의 합리적인 범위 안에 있는지 제한하는 것이다. Neighborhood min/max clamping, variance clipping, color space 선택이 ghosting과 flickering 사이의 균형에 어떤 영향을 주는지가 핵심이다.

## 10. 참고 키워드

- Velocity Dilation
- Neighborhood Search
- Motion Vector
- Velocity Buffer
- Temporal Reprojection
- Temporal Anti-Aliasing
- Temporal Upscaling
- Motion Vector Confidence
- Dilated Depth
- Nearest-depth Selection
- Maximum-magnitude Velocity
- Surface-aware Reprojection
- Disocclusion
- Reactive Mask
- Reversed-Z
- Dynamic Resolution
- Compute Shader
- Shared Memory Tile
- Neighborhood Clamping
- Variance Clipping
- Sparse Voxel LOD
- Field-aware Temporal Filtering
