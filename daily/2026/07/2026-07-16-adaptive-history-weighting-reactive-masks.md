---
title: "Adaptive History Weighting and Reactive Masks"
date: "2026-07-16"
category: "Graphics"
tags: ["GPU", "Temporal Anti-Aliasing", "Temporal Upscaling", "Adaptive History Weighting", "Reactive Mask", "History Confidence", "Compute Shader", "R8_UNORM", "Particles", "Real-time Rendering"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-07-16 - Adaptive History Weighting and Reactive Masks

## 1. 오늘의 개념

**Adaptive History Weighting**은 temporal reprojection으로 가져온 이전 프레임의 history를 모든 pixel에 같은 비율로 섞지 않고, 현재 pixel의 신뢰도와 변화량에 따라 **history contribution을 동적으로 조절하는 방식**이다.

기본 temporal accumulation은 다음 형태로 표현할 수 있다.

```text
C_out = (1 - w_h) · C_current + w_h · H_rectified
```

- `C_current`: 현재 프레임에서 계산한 색
- `H_rectified`: reprojection 후 validation과 clipping을 통과한 history 색
- `w_h`: history weight

어제 살펴본 **Neighborhood Clamping / Variance Clipping**이 “history 값을 어느 범위까지 허용할 것인가”를 결정했다면, 오늘의 개념은 그 다음 단계에서 **허용된 history를 실제로 얼마나 믿을 것인가**를 결정한다.

**Reactive Mask**는 이 판단을 engine 또는 material pipeline이 보조하는 per-pixel signal이다. 일반적으로 값이 클수록 해당 pixel은 현재 프레임의 새 정보를 빠르게 반영해야 하며, history 의존도를 낮춰야 한다.

대표적인 reactive 영역은 다음과 같다.

- alpha-blended particle, smoke, fire
- transparent surface와 refraction
- animated emissive와 빠르게 이동하는 specular highlight
- ray-traced reflection 또는 screen-space reflection의 불안정 영역
- animated texture, video texture, flipbook
- topology가 프레임마다 바뀌는 marching cubes surface
- volume ray marching의 hit point가 급격히 달라지는 영역
- motion vector와 depth에 충분한 footprint를 남기지 않는 visual effect

Temporal rendering에서 중요한 질문은 단순히 “history가 유효한가?”가 아니다.

> **유효하더라도, 지금 이 pixel에서는 history를 얼마나 오래 유지해야 하는가?**

---

## 2. 한 줄 핵심

**Adaptive History Weighting은 geometry·motion·color·luminance·material reactivity를 종합해 history weight를 pixel마다 조절함으로써, 고정 blend ratio가 만드는 ghosting과 flickering의 전역적 trade-off를 국소적인 confidence decision으로 바꾼다.**

---

## 3. 왜 중요한가

### 3.1 고정 history weight의 구조적 한계

모든 pixel에 다음과 같은 고정 weight를 적용한다고 가정하자.

```text
w_h = 0.9
```

정적인 벽, 느리게 움직이는 opaque object, sub-pixel edge에는 높은 history weight가 유리하다.

- jittered sample이 여러 프레임에 걸쳐 누적됨
- edge와 thin geometry가 안정됨
- stochastic noise가 줄어듦
- temporal upscaling의 detail reconstruction이 좋아짐

하지만 동일한 weight를 particle이나 transparency에 적용하면 문제가 생긴다.

- 이전 particle 색이 현재 위치에 남음
- fire와 emissive가 이동 방향으로 길게 번짐
- alpha coverage가 바뀌어도 과거 색이 유지됨
- reflection이 실제 장면 변화보다 늦게 반응함

반대로 모든 pixel의 weight를 낮추면 ghosting은 감소하지만 temporal accumulation의 장점을 잃는다.

```text
높은 고정 weight
→ 안정성 증가
→ 반응성 감소
→ ghosting / smearing 증가

낮은 고정 weight
→ 반응성 증가
→ history 재사용 감소
→ flickering / noise / shimmering 증가
```

즉, 고정 weight는 서로 성격이 다른 pixel을 하나의 정책으로 처리한다는 점에서 한계가 있다.

### 3.2 History validation과 weighting은 다른 문제다

Depth, normal, object ID를 비교해 history가 같은 surface에서 왔다고 판정해도, 그 history를 높은 비율로 누적해야 한다는 뜻은 아니다.

예를 들어 opaque geometry 위에 alpha-blended fire가 합성되었다면 다음 상황이 가능하다.

```text
depth       : 동일
normal      : 동일
object ID   : 동일
motion      : background 기준으로 정상
final color : fire animation 때문에 급격히 변화
```

Geometric validation만 보면 history는 유효하다. 그러나 최종 색은 빠르게 바뀌므로 history weight는 낮아야 한다.

따라서 temporal resolve는 보통 두 단계로 생각해야 한다.

```text
1. History validity
   → 이 sample을 재사용해도 되는가?

2. History confidence / weight
   → 재사용한다면 얼마나 강하게 믿을 것인가?
```

### 3.3 Reactive Mask가 engine integration 문제인 이유

Temporal algorithm은 최종 color, depth, motion vector만 보고는 해당 pixel이 어떤 방식으로 만들어졌는지 완전히 알 수 없다.

특히 alpha-blended object는 다음 정보가 부족할 수 있다.

- depth를 쓰지 않음
- motion vector를 쓰지 않음
- background와 foreground가 한 pixel에 합성됨
- 하나의 motion vector로 두 layer의 움직임을 표현할 수 없음

Reactive mask는 renderer가 알고 있는 **material semantic**을 temporal resolve에 전달한다.

```text
Temporal pass가 추론하기 어려운 정보
← material / VFX / composition pass가 명시적으로 제공
```

이 관점에서 reactive mask는 단순한 artifact 보정 texture가 아니라, **frame graph의 앞 단계가 temporal reconstruction 단계에 전달하는 confidence metadata**다.

---

## 4. 구현 관점

### 4.1 Temporal resolve 안에서의 위치

일반적인 흐름은 다음과 같다.

```text
Current color / depth / normal / velocity 생성
                    ↓
Velocity dilation / reprojection
                    ↓
Depth · normal · object ID validation
                    ↓
Neighborhood clamping / variance clipping
                    ↓
Reactive mask와 변화량 sampling
                    ↓
Adaptive history weight 계산
                    ↓
Current와 rectified history accumulation
                    ↓
History color · confidence · history length 갱신
```

Clipping과 weighting은 역할이 다르다.

```text
Clipping
→ history color 자체를 현재 neighborhood의 합리적 범위로 수정

Weighting
→ 수정된 history가 최종 출력에 미치는 영향 결정
```

잘못된 history를 weight만 낮춰 섞으면 잔상이 여전히 남을 수 있다. 반대로 올바른 history를 지나치게 clamp한 뒤 높은 weight로 누적하면 detail이 뭉개질 수 있다.

두 단계는 함께 설계해야 한다.

### 4.2 기본 accumulation 식

가장 단순한 accumulation은 exponential moving average와 비슷하다.

```text
C_out = lerp(C_current, H_rectified, w_h)
```

여기서 `w_h`가 높을수록 history를 많이 유지한다.

```text
w_h → 1 : 느린 변화, 높은 안정성
w_h → 0 : 빠른 변화, 현재 프레임 우선
```

하지만 `w_h`를 바로 하나의 heuristic으로 만들기보다 여러 confidence를 분리해 생각하는 것이 디버깅에 유리하다.

```text
c_geometry
c_motion
c_color
c_luminance
c_reactive
c_exposure
c_reset
```

개념적으로는 다음처럼 구성할 수 있다.

```text
w_h = w_base
    · c_geometry
    · c_motion
    · c_color
    · c_luminance
    · c_reactive
    · c_exposure
    · c_reset
```

각 confidence는 보통 `[0, 1]` 범위다.

다만 모든 값을 단순 곱하면 작은 오차가 연속해서 누적되어 weight가 과도하게 0에 가까워질 수 있다. 실무에서는 다음 방식도 사용한다.

```text
w_h = min(w_base, w_geometry, w_motion, w_color, w_reactive)
```

또는 일부 signal만 곱하고 나머지는 `min`, `max`, `lerp`로 조합한다.

핵심은 수식의 형태보다 **각 signal이 어떤 artifact를 책임지는지 분리 가능해야 한다는 것**이다.

### 4.3 Motion 기반 weight

화면 공간 motion magnitude가 크면 history footprint의 mismatch 가능성도 커진다.

```text
v = length(motionVectorPixels)
c_motion = saturate(1 - v / v_threshold)
```

하지만 motion이 크다는 이유만으로 history를 모두 버리면 빠르게 이동하는 rigid object의 edge가 불안정해질 수 있다.

따라서 motion signal은 다음과 함께 보아야 한다.

- motion vector gradient
- local depth discontinuity
- velocity dilation 결과
- object 내부인지 silhouette인지
- reprojection footprint가 화면 안에 존재하는지

예를 들어 object 내부의 일관된 큰 motion은 비교적 신뢰할 수 있지만, foreground와 background velocity가 섞이는 silhouette에서는 더 낮은 weight가 필요하다.

```text
큰 motion + 낮은 velocity gradient
→ 이동은 빠르지만 correspondence는 비교적 안정적

큰 motion + 높은 velocity gradient
→ 경계 또는 disocclusion 가능성
→ history weight 강하게 감소
```

### 4.4 Color와 luminance 변화 기반 weight

현재 color와 rectified history의 차이가 크면 shading change 또는 잘못된 correspondence일 가능성이 있다.

```text
ΔL = abs(L_current - L_history)
c_luminance = exp(-k · ΔL)
```

또는 threshold 기반으로 구성할 수 있다.

```text
c_luminance = 1 - smoothstep(T_low, T_high, ΔL)
```

HDR에서는 raw luminance 차이가 매우 밝은 emissive에 의해 과도하게 커질 수 있다. 따라서 다음 기준 중 하나를 사용한다.

- pre-exposed color 기준 비교
- log luminance 비교
- luminance compression 후 차이 계산
- relative difference 사용

```text
ΔL_relative =
    abs(L_current - L_history)
    / max(max(L_current, L_history), ε)
```

Exposure가 바뀐 경우에는 history를 현재 exposure scale로 맞춘 뒤 비교해야 한다. 그렇지 않으면 동일한 radiance도 큰 shading change로 오인된다.

### 4.5 Reactive mask가 history weight에 들어가는 방식

Reactive mask를 `R ∈ [0, 1]`이라고 하면 단순한 방식은 다음과 같다.

```text
c_reactive = 1 - R
w_h = w_base · c_reactive
```

하지만 `R = 1`에서 history를 완전히 제거하면 particle edge나 thin effect가 매 프레임 크게 흔들릴 수 있다.

따라서 최소 history weight를 남기는 형태가 더 안정적일 수 있다.

```text
w_h = lerp(w_base, w_min, R)
```

- `R = 0`: 일반 temporal accumulation
- `R = 1`: history를 완전히 버리기보다 `w_min`까지 낮춤

또 다른 방식은 weight뿐 아니라 clipping 강도와 함께 조절하는 것이다.

```text
reactivity 증가
→ history weight 감소
→ clipping range 축소
→ history length 감소
→ lock / stability 상태 약화
```

Reactive mask를 accumulation weight에만 연결하면 잘못된 history color가 낮은 비율로 계속 남을 수 있다. 반대로 mask가 강한 영역에서 clipping과 lock까지 동시에 완화하면 현재 frame으로 더 빠르게 복귀할 수 있다.

### 4.6 Reactive mask 생성 전략

#### A. Material-driven mask

Renderer가 material type을 알고 있을 때 가장 의미가 명확하다.

```text
opaque static material     → 낮은 reactivity
alpha-blended smoke        → 높은 reactivity
animated emissive          → 중간~높은 reactivity
ray-traced reflection      → confidence에 따라 조절
video / flipbook texture   → 높은 reactivity
```

Alpha-blended object에서는 실제 composition alpha가 좋은 출발점이 될 수 있다.

```text
reactive = alpha
```

다만 alpha는 항상 temporal instability와 일치하지 않는다.

- 반투명하지만 정적인 유리: alpha는 높아도 history가 안정적일 수 있음
- additive particle: alpha 정의가 coverage와 다를 수 있음
- premultiplied alpha: stored alpha와 color energy의 관계가 다름
- refraction: alpha보다 background distortion이 더 중요한 signal일 수 있음

따라서 material-driven mask는 shader category와 effect semantics를 함께 반영하는 것이 좋다.

#### B. Opaque-only와 composed color의 차이

Opaque pass 결과와 transparency까지 합성된 결과를 비교한다.

```text
C_opaque
C_composed

D = distance(C_opaque, C_composed)
R = saturate(scale · D)
```

이 방식은 alpha object가 최종 color에 얼마나 영향을 주었는지를 직접 측정한다.

장점:

- material별 수동 튜닝 없이 시작 가능
- alpha 값보다 실제 color contribution을 반영
- additive effect에도 반응 가능

한계:

- opaque-only color buffer를 보존해야 함
- 추가 bandwidth와 resource lifetime 필요
- tone mapping 전후 위치에 따라 mask 의미가 달라짐
- 밝기 차이만 크고 temporal instability는 낮은 영역도 reactive로 판정될 수 있음

AMD FSR2의 자동 reactive mask 생성도 opaque-only color와 transparency 포함 color를 입력으로 사용하고, luminance 기반 heuristic을 적용한다.

#### C. Temporal difference 기반 자동 mask

현재 프레임과 reprojected history의 차이를 이용한다.

```text
R_auto =
    smoothstep(T_low, T_high,
               abs(L_current - L_reprojected))
```

이 방식은 renderer integration이 단순하지만 원인과 결과가 뒤섞인다.

- 잘못된 motion vector
- disocclusion
- exposure 변화
- 실제 animated material
- stochastic noise

모두 큰 temporal difference를 만들 수 있다.

따라서 자동 mask는 **fallback signal**로 유용하지만, material semantics를 알고 있는 명시적 mask보다 오탐 가능성이 높다.

### 4.7 Reactive mask와 Transparency / Composition mask의 구분

Reactive mask는 일반적으로 “현재 frame을 얼마나 빠르게 반영할 것인가”를 나타낸다.

```text
Reactive mask
→ accumulation balance 조절
→ current sample contribution 증가
```

반면 transparency / composition 계열 mask는 history protection이나 lock mechanism을 해제하는 데 사용될 수 있다.

```text
Composition mask
→ 기존 history를 보호하던 stability state 약화
→ 오래된 lock 제거
→ color rectification과 history reset을 더 적극적으로 허용
```

두 mask는 비슷해 보이지만 의미가 다르다.

- reactive: blend ratio에 대한 신호
- composition: history 상태 머신에 대한 신호

하나의 mask로 모든 temporal policy를 제어하면 tuning은 단순하지만, particle의 반응성과 thin-detail lock 유지 같은 서로 다른 요구를 분리하기 어렵다.

### 4.8 History length와 adaptive weight

Temporal denoising이나 stochastic rendering에서는 history sample count 또는 history length를 저장하기도 한다.

```text
N_history = min(N_history + 1, N_max)
```

단순한 running average weight는 다음과 같다.

```text
w_h = N_history / (N_history + 1)
```

Reactive 영역에서는 history length를 줄이거나 빠르게 decay시킬 수 있다.

```text
N_history' = lerp(N_history, N_min, R)
```

이 방식은 한 프레임의 blend만 바꾸는 것이 아니라, 앞으로 몇 프레임 동안 history가 얼마나 빨리 다시 안정화되는지도 결정한다.

```text
weight만 감소
→ 이번 frame에서 current 비중 증가
→ 다음 frame에는 오래된 history confidence가 다시 살아날 수 있음

history length도 감소
→ temporal state 자체를 약화
→ 이후 frame에서도 천천히 재축적
```

따라서 reactive event가 짧고 강한지, 지속적이고 완만한지에 따라 weight와 history length를 다르게 다루는 것이 좋다.

### 4.9 Mask dilation과 edge coverage

Reactive mask가 render resolution에서 생성되고 temporal resolve가 presentation resolution에서 동작하면, 단순 point sampling으로는 thin particle edge를 놓칠 수 있다.

또한 bilinear sampling은 높은 reactive value를 주변에 부드럽게 퍼뜨리지만, 작은 feature의 peak를 낮출 수 있다.

실무적으로 고려할 수 있는 정책은 다음과 같다.

- `2×2` 또는 `3×3` max dilation
- depth-aware dilation
- motion direction을 따라 anisotropic dilation
- output pixel footprint에 맞춘 conservative sampling
- alpha coverage와 reactive value의 분리

Reactive 영역을 넓히면 ghosting은 줄지만 주변 opaque geometry의 history까지 약해져 shimmering이 증가할 수 있다.

```text
과소 dilation
→ particle boundary에 ghost trail

과대 dilation
→ 주변 background의 temporal stability 손실
```

Velocity dilation과 마찬가지로 mask dilation도 **어떤 sample의 속성을 주변 pixel에 대신 부여하는 문제**다. 따라서 단순 max filter가 항상 최선은 아니다.

### 4.10 GPU resource와 memory layout

Reactive mask는 대체로 single-channel low-precision texture로 충분하다.

```text
Format     : R8_UNORM
Range      : 0..1
Resolution : render resolution
Lifetime   : current frame
Usage      : render target 또는 UAV → shader resource
```

`R8_UNORM`의 장점:

- pixel당 1 byte
- cache footprint가 작음
- blend 또는 atomic 없이 material pass에서 기록하기 쉬움
- temporal resolve에서 texture sampling 비용이 낮음

1920×1080 기준 단일 `R8_UNORM` mask의 raw 크기는 약 2 MB다.

```text
1920 × 1080 × 1 byte ≈ 1.98 MiB
```

하지만 실제 비용은 크기만으로 결정되지 않는다.

- 별도 render target transition
- VFX pass에서의 write bandwidth
- resolve pass에서의 read bandwidth
- resource lifetime 때문에 발생하는 aliasing 제약
- render resolution과 presentation resolution 사이의 sampling
- 여러 mask를 별도 texture로 둘 때의 descriptor와 cache 비용

Reactive, transparency, confidence, material flag를 한 texture에 pack할 수도 있다.

```text
R8G8_UNORM
R : reactive
G : composition / transparency
```

또는 bit field와 low-precision channel을 조합할 수 있다.

다만 서로 다른 sampling filter와 precision이 필요한 signal을 과도하게 pack하면 이후 pipeline의 유연성이 떨어진다.

### 4.11 Compute shader 관점

Temporal resolve는 보통 presentation resolution의 compute shader로 구성하기 좋다.

각 thread는 대략 다음 데이터를 읽는다.

```text
Current color neighborhood
Depth / normal / motion vector
Previous history color
Previous history metadata
Reactive mask
Exposure / frame constants
```

주요 GPU 비용은 다음에서 발생한다.

- current neighborhood texture fetch
- history reprojection의 filtered sample
- mask와 metadata fetch
- clipping 통계 계산
- branch divergence
- output history와 metadata write

Adaptive weighting signal을 많이 추가하면 ALU보다 texture bandwidth가 먼저 문제가 될 수 있다.

따라서 실무에서는 다음 선택이 중요하다.

```text
새 confidence texture 추가
vs
기존 G-buffer / mask channel 재사용
vs
shader에서 heuristic 재계산
```

- texture 추가: 의미가 명확하지만 bandwidth와 memory 증가
- channel packing: bandwidth 절약, format과 lifetime coupling 증가
- shader 계산: memory 절약, ALU와 register pressure 증가

Graphics engineer는 artifact 개선뿐 아니라 frame graph, resource barrier, cache behavior, occupancy까지 함께 판단해야 한다.

### 4.12 디버깅해야 할 중간 결과

Adaptive history weighting은 최종 이미지 하나만 보고 튜닝하기 어렵다. 다음 buffer를 개별적으로 확인할 수 있어야 한다.

```text
Reactive mask
Final history weight
History validity
History length
Motion confidence
Luminance difference
Clipped history color
Current / history contribution
```

특히 다음 두 visualizer가 유용하다.

```text
History contribution heatmap
→ 어디에서 과거 frame이 강하게 남는지 확인

Reactive reason view
→ material, motion, luminance, disocclusion 중
   어떤 signal이 weight를 낮췄는지 확인
```

하나의 final weight만 저장하면 artifact의 원인을 알기 어렵다. 개발 단계에서는 confidence source를 분리해 관찰하고, shipping 단계에서만 compact하게 합치는 것이 좋다.

---

## 5. 내 관심 분야와 연결

### Real-time rendering과 game engine

TAA와 temporal upscaling은 단순한 post-process가 아니라 renderer 전체의 정보를 요구한다.

- base pass의 motion vector
- transparency pass의 reactive mask
- exposure system
- reflection과 particle pipeline
- camera cut와 dynamic resolution event
- frame graph resource lifetime

따라서 adaptive weighting은 engine architecture 관점에서 **서로 다른 rendering subsystem의 정보를 temporal resolve로 모으는 integration point**다.

### Particle simulation과 visual effects

Particle은 reactive mask가 가장 직접적으로 필요한 영역이다.

- alpha blending
- depth 미기록
- motion vector 누락
- 짧은 lifetime
- 빠른 opacity 변화
- flipbook animation

Particle color가 simulation state보다 훨씬 빠르게 변할 수 있으므로, world-space velocity가 정확해도 color history는 신뢰하기 어렵다.

즉 particle에서는 다음을 구분해야 한다.

```text
Geometric motion confidence
≠
Appearance stability
```

### Fluid / volume visualization

Volume rendering에서는 한 pixel의 color가 고정 surface가 아니라 ray를 따라 적분된 결과다.

Transfer function, density field, camera motion이 조금만 변해도 dominant contribution 위치가 달라질 수 있다.

Reactive signal 후보:

- ray termination depth 변화
- accumulated opacity 변화
- gradient magnitude 변화
- transfer-function response 변화
- current와 history의 representative depth 차이

단순 screen-space color difference보다 volume integration 과정에서 얻는 metadata를 사용하면 더 정확한 history weight를 만들 수 있다.

### Marching Cubes와 sparse voxel

Marching cubes topology가 프레임마다 달라지는 simulation에서는 동일한 screen pixel에 대응하는 triangle identity가 안정적이지 않을 수 있다.

다음 영역을 reactive하게 볼 수 있다.

- iso-surface가 새로 생성되거나 사라지는 영역
- topology change가 발생한 voxel cell
- normal gradient가 급격히 바뀌는 영역
- LOD transition boundary
- sparse brick의 resident state가 바뀐 영역

Simulation pipeline이 알고 있는 change mask를 temporal renderer에 전달하면 final color만 비교하는 것보다 의미 있는 confidence를 제공할 수 있다.

### WebGPU / Vulkan / DirectX

API가 달라도 구조는 비슷하다.

- mask resource 생성
- VFX 또는 composition pass에서 write
- UAV / render attachment에서 sampled texture로 transition
- temporal compute pass에서 sample
- history ping-pong resource 갱신

WebGPU에서는 texture usage를 생성 시 명시해야 하므로 `RENDER_ATTACHMENT`, `STORAGE_BINDING`, `TEXTURE_BINDING` 중 실제 생성·소비 경로를 미리 설계해야 한다.

Vulkan과 DirectX 12에서는 reactive mask의 producer와 temporal resolve consumer 사이 resource state와 synchronization이 중요하다. Mask가 작더라도 별도 pass와 barrier가 늘어나면 frame graph scheduling 비용이 생긴다.

### Computer vision 관점

Adaptive history weighting은 영상 처리의 confidence-weighted temporal filtering과 닮아 있다.

- optical flow confidence
- occlusion mask
- photometric consistency
- temporal change detection
- robust estimator

Graphics에서는 추가로 renderer 내부의 semantic data를 사용할 수 있다는 점이 다르다.

Computer vision은 image에서 material과 motion을 추론해야 하지만, renderer는 object ID, depth, material type, simulation change flag를 직접 알고 있다. 이 정보를 활용하는 것이 real-time graphics의 강점이다.

---

## 6. 머릿속에 남길 질문 3개

1. **History가 geometric validation을 통과했더라도 appearance 변화 때문에 weight를 낮춰야 하는 대표적인 rendering case는 무엇인가?**

2. **Reactive mask를 material pass에서 직접 생성하는 방식과 final color difference로 자동 생성하는 방식은 각각 어떤 정보를 알고 있으며, 어떤 오탐을 만들 수 있는가?**

3. **Reactive 영역에서 history weight만 낮추는 것과 history length·lock·clipping range까지 함께 변경하는 것은 시간적으로 어떤 차이를 만드는가?**

---

## 7. Graphics Engineer 면접 질문 1개와 답변

### 질문

**Temporal Anti-Aliasing에서 모든 pixel에 고정 history weight를 사용하지 않고 adaptive history weighting과 reactive mask를 사용하는 이유를 설명해 주세요.**

### 답변

고정 history weight는 정적인 opaque geometry와 빠르게 변하는 transparency, particle, reflection을 같은 temporal policy로 처리한다는 문제가 있습니다.

History weight가 높으면 jittered sample이 잘 누적되어 edge와 sub-pixel detail이 안정되지만, alpha-blended object나 animated emissive처럼 appearance가 빠르게 변하는 영역에서는 이전 색이 오래 남아 ghosting과 smearing이 발생합니다. 반대로 weight를 전역적으로 낮추면 ghosting은 줄지만 정적인 영역에서도 history 재사용이 약해져 flickering과 shimmering이 증가합니다.

Adaptive history weighting은 depth·normal 기반 validity, motion magnitude와 gradient, current-history luminance difference, exposure 변화, material reactivity 같은 signal을 이용해 pixel마다 history confidence를 계산합니다.

Reactive mask는 특히 depth나 motion vector만으로 판별하기 어려운 alpha-blended particle, transparency, animated texture 같은 영역을 renderer가 명시적으로 표시하는 입력입니다. Reactive value가 높을수록 current frame의 contribution을 높이고 history weight를 낮춥니다.

좋은 구현에서는 weight만 바꾸는 데 그치지 않고, 필요한 경우 history clipping range, history length, lock 또는 stability state도 함께 조절합니다. 또한 reactive mask는 보통 render-resolution `R8_UNORM` texture로 관리할 수 있지만, 별도 resource의 bandwidth와 frame graph transition 비용까지 고려해야 합니다.

결국 목적은 ghosting과 flickering 사이에서 하나의 전역 상수를 선택하는 것이 아니라, 각 pixel의 temporal confidence에 맞는 accumulation policy를 적용하는 것입니다.

---

## 8. 포트폴리오 / 커리어 연결

Temporal rendering 포트폴리오에서 단순히 “TAA를 구현했다”는 설명보다 다음 구조를 보여주는 것이 graphics engineer 역량을 더 잘 드러낸다.

```text
Temporal input
- jitter
- motion vector
- depth / normal
- exposure

History validation
- reprojection
- disocclusion detection
- velocity dilation

History rectification
- neighborhood clamping
- variance clipping
- YCoCg

Adaptive accumulation
- reactive mask
- motion / luminance confidence
- history length
- lock state

GPU integration
- resource format
- ping-pong history
- frame graph barrier
- compute dispatch
- debug visualization
```

특히 다음 내용을 설명할 수 있으면 강점이 된다.

- 왜 particle은 정확한 motion vector만으로 해결되지 않는가
- reactive mask를 어느 pass에서 생성하는가
- mask를 `R8_UNORM`으로 둘 때 precision이 충분한 이유
- render-resolution mask를 output-resolution resolve에서 어떻게 보수적으로 sample하는가
- artifact를 final image가 아닌 confidence buffer로 어떻게 진단하는가
- 높은 history weight가 좋은 영역과 나쁜 영역을 어떻게 분리하는가
- visual quality 개선이 bandwidth와 frame time에 어떤 비용을 추가하는가

게임 엔진, real-time renderer, scientific visualization 직무 모두에서 이 주제는 “shader를 작성할 수 있는가”보다 한 단계 더 나아가, **여러 rendering pass의 semantic data를 통합해 temporal artifact를 해결할 수 있는가**를 보여준다.

---

## 9. 내일 이어서 볼 개념

**History Length, Lock Status, and Temporal Stability State**

내일은 history weight를 한 프레임의 blend factor로만 보지 않고, pixel이 여러 프레임 동안 얼마나 안정적으로 관측되었는지를 저장하는 **history length / lock / stability state** 관점으로 확장한다.

이어지는 핵심 질문은 다음과 같다.

> **Temporal system은 어떤 pixel을 오래 보호하고, 어떤 event에서 그 보호 상태를 즉시 해제해야 하는가?**

---

## 10. 참고 키워드

- Adaptive History Weighting
- History Confidence
- Temporal Accumulation
- Exponential Moving Average
- Reactive Mask
- Transparency and Composition Mask
- History Length
- Temporal Lock
- Stability Factor
- Alpha-blended Particles
- Motion Vector Gradient
- Luminance Difference
- Exposure Compensation
- Disocclusion
- History Rectification
- Neighborhood Clamping
- Variance Clipping
- R8_UNORM
- Temporal Upscaling
- FidelityFX Super Resolution 2
- TAA
- TSR
- Temporal Denoising

### 참고 자료

- [AMD FidelityFX Super Resolution 2 — Reactive Mask and Reproject & Accumulate](https://github.com/GPUOpen-Effects/FidelityFX-FSR2/blob/master/README.md)
- [Brian Karis — High Quality Temporal Supersampling](https://advances.realtimerendering.com/s2014/)
- [Lei Yang, Shiqiu Liu, Marco Salvi — A Survey of Temporal Antialiasing Techniques](https://doi.org/10.1111/cgf.14018)
