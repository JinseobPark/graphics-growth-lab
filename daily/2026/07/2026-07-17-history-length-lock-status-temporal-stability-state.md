---
title: "History Length, Lock Status, and Temporal Stability State"
date: "2026-07-17"
category: "Graphics"
tags: ["GPU", "Temporal Anti-Aliasing", "Temporal Upscaling", "History Length", "Temporal Lock", "Stability State", "History Confidence", "Compute Shader", "Memory Layout", "Scientific Visualization"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-07-17 - History Length, Lock Status, and Temporal Stability State

## 1. 오늘의 개념

Temporal rendering에서 이전 프레임의 결과를 재사용할 때, 단순히 `history color`만 저장해서는 충분하지 않다. 같은 색이라도 그 값이 **한 프레임 전에 처음 만들어진 값인지**, **수십 프레임 동안 안정적으로 누적된 값인지**, **얇은 형상을 보존하기 위해 특별히 보호된 값인지**에 따라 처리 방식이 달라져야 한다.

오늘 다룰 세 가지 상태는 다음과 같다.

- **History Length**: 해당 pixel의 history가 연속적으로 유효했다고 판단된 기간 또는 유효 sample 수
- **Lock Status**: 얇은 geometry나 sub-pixel detail을 clipping·rectification으로부터 일시적으로 보호하는 상태
- **Temporal Stability State**: history length, confidence, reactive signal, shading change 등을 종합한 pixel별 시간적 상태

전날 살펴본 **Adaptive History Weighting**이 현재 프레임에서 history를 얼마나 섞을지 결정했다면, 오늘의 주제는 그 판단에 필요한 **시간적 기억 자체를 어떻게 저장하고 갱신할 것인가**에 해당한다.

```text
History color
→ 과거에 어떤 색이 누적되었는가?

History length
→ 그 누적이 얼마나 오래 유지되었는가?

Lock status
→ 이 pixel의 detail을 얼마나 적극적으로 보호해야 하는가?

Stability state
→ 현재 이 pixel을 안정, 불안정, 반응 중, 재수렴 중 어느 상태로 볼 것인가?
```

Temporal system은 결국 pixel마다 작은 상태 머신(state machine)을 운영하는 것과 비슷하다.

---

## 2. 한 줄 핵심

**History length와 temporal stability state는 pixel의 과거가 얼마나 신뢰할 만한지를 시간축으로 기억하고, lock status는 안정적으로 검출된 detail이 일시적인 clipping이나 low-resolution sampling 때문에 사라지지 않도록 보호한다.**

---

## 3. 왜 중요한가

### 3.1 동일한 history weight라도 의미가 다르다

다음 두 pixel이 모두 `historyWeight = 0.9`를 사용한다고 가정하자.

```text
Pixel A
- 1프레임 전에 disocclusion 발생
- 현재 history length = 1
- 아직 수렴하지 않음

Pixel B
- 30프레임 동안 동일한 surface를 추적
- 현재 history length = 30
- 안정적으로 수렴함
```

두 pixel이 같은 blend factor를 사용하더라도 신뢰도는 전혀 다르다.

Pixel A는 한 번의 잘못된 reprojection이나 noisy sample에 크게 흔들릴 수 있다. 반면 Pixel B는 장기간 누적된 결과이므로 작은 shading 변화나 neighborhood fluctuation 때문에 즉시 history를 버리면 오히려 shimmer가 증가한다.

따라서 temporal resolve는 현재 프레임의 signal뿐 아니라 다음 정보를 함께 보아야 한다.

- history가 몇 프레임 동안 유효했는가
- 최근에 invalidation 또는 reactive event가 있었는가
- 현재 상태가 warming-up인지 stable인지
- thin feature 보호 lock이 활성화되어 있는가
- shading change가 lock을 해제할 만큼 큰가

### 3.2 History Length는 시간적 sample count다

가장 직관적인 history length는 연속적으로 유효했던 프레임 수다.

```text
historyLength_t =
    validHistory
        ? min(historyLength_{t-1} + 1, maxLength)
        : 1
```

단순 running average에서는 history length가 blend ratio를 직접 결정할 수 있다.

```text
C_t = (N · H_{t-1} + C_current) / (N + 1)

historyWeight = N / (N + 1)
currentWeight = 1 / (N + 1)
```

여기서 `N`은 이전까지 누적된 유효 sample 수다.

초기에는 current frame의 영향이 크고, 시간이 지날수록 history가 강해진다.

```text
N = 1   → historyWeight = 0.50
N = 3   → historyWeight = 0.75
N = 9   → historyWeight = 0.90
N = 31  → historyWeight ≈ 0.97
```

하지만 real-time renderer에서는 history를 무한히 강하게 만들 수 없다. 장기간 누적된 pixel이 조명 변화나 animation에 지나치게 늦게 반응하기 때문이다.

따라서 일반적으로 다음 중 하나를 사용한다.

- history length를 `N_max`에서 clamp
- fixed minimum current weight 유지
- reactive signal이 들어오면 history length 감소
- motion·shading change에 따라 fractional age 적용
- stable state와 responsive state의 weight policy 분리

History length는 literal frame count일 수도 있지만, 실무에서는 **effective sample count** 또는 **confidence accumulator**에 더 가깝게 사용되는 경우가 많다.

### 3.3 안정성은 binary가 아니라 누적되는 성질이다

History validation은 보통 현재 frame에서 유효/무효를 판정한다.

```text
valid =
    depthMatch
    && normalMatch
    && objectMatch
    && insideViewport
```

하지만 실제 temporal stability는 binary보다 연속적인 값에 가깝다.

```text
stability_t =
    valid
        ? saturate(stability_{t-1} + growthRate)
        : stability_{t-1} · decayRate
```

예를 들어 다음 pixel들은 모두 geometric validation을 통과할 수 있다.

- 정적인 벽
- 천천히 움직이는 rigid object 내부
- 빠르게 점멸하는 emissive surface
- alpha-blended smoke가 합성된 background
- 매 프레임 topology가 바뀌는 marching-cubes surface

Geometry만 보면 같은 surface일 수 있지만 appearance stability는 다르다. 따라서 안정성은 depth·normal뿐 아니라 motion, luminance, material reactivity, topology 변화까지 반영해야 한다.

### 3.4 Lock은 history validity와 다른 개념이다

Temporal upscaling에서는 저해상도 입력의 thin feature가 jitter sequence의 일부 frame에서만 관측될 수 있다.

대표적인 예는 다음과 같다.

- 철망과 울타리
- 전선
- 얇은 나뭇가지
- sub-pixel edge
- 작은 specular highlight
- 1~2 pixel 두께의 UI-like world geometry

이러한 detail은 현재 frame neighborhood만 기준으로 history를 강하게 clamp하면 쉽게 사라진다.

```text
현재 neighborhood에 detail이 약하게 보임
→ history color가 outlier로 판단됨
→ clipping으로 제거됨
→ 다음 frame에서 다시 나타남
→ 깜빡임 발생
```

**Temporal Lock**은 해당 pixel이 안정적인 thin feature라고 판단되면 일정 기간 rectification 또는 clipping을 완화해 detail을 보호한다.

중요한 점은 다음과 같다.

> Lock은 “history가 항상 옳다”는 의미가 아니라, “현재 frame의 불완전한 sampling만으로 과거 detail을 너무 빨리 제거하지 말라”는 의미다.

따라서 lock에는 반드시 해제 조건이 필요하다.

- disocclusion
- 큰 luminance 변화
- reactive mask 증가
- object teleport
- camera cut
- motion vector 불연속
- depth·normal mismatch
- dynamic resolution 또는 jitter sequence 변경

### 3.5 안정 상태를 저장하지 않으면 정책이 매 frame 흔들린다

Temporal resolve가 매 frame 독립적으로 confidence를 계산하면 threshold 주변에서 상태가 반복적으로 바뀔 수 있다.

```text
Frame 1: stable 판정
Frame 2: 작은 차이로 unstable 판정
Frame 3: 다시 stable 판정
Frame 4: 다시 unstable 판정
```

이 현상은 temporal policy 자체가 flicker를 만드는 결과로 이어질 수 있다.

Stability state에 hysteresis를 두면 상태 전환을 완화할 수 있다.

```text
Stable 진입 threshold   = 0.85
Stable 해제 threshold   = 0.55
```

즉, stable 상태에 들어가기 위한 조건과 빠져나오기 위한 조건을 다르게 둔다.

이 방식은 graphics뿐 아니라 simulation visualization에서도 중요하다. camera가 정지한 상태에서 noisy volume rendering이나 streamline antialiasing이 수렴했다면, 작은 수치 변화만으로 매 frame history를 초기화하지 않는 편이 시각적으로 안정적이다.

---

## 4. 구현 관점

### 4.1 Pixel별 temporal state를 작은 상태 머신으로 본다

개념적으로 다음과 같은 상태를 둘 수 있다.

```text
Invalid
  ↓ valid sample 확보
Warming
  ↓ history length 증가, confidence 상승
Stable
  ↓ thin feature 검출
Locked
  ↓ reactive/shading change
Responsive 또는 Cooldown
  ↓ 다시 안정화
Warming
```

각 상태의 목적은 다르다.

| 상태 | 의미 | 일반적인 정책 |
|---|---|---|
| Invalid | history를 재사용할 수 없음 | current 100%, length reset |
| Warming | history가 아직 짧음 | 빠른 누적, 보수적 lock |
| Stable | 여러 frame 동안 일관됨 | 높은 history weight, 낮은 noise |
| Locked | detail 보호 필요 | clipping 완화, lock lifetime 감소 |
| Responsive | 변화가 감지됨 | history weight 감소, length decay |
| Cooldown | 변화 직후 재수렴 중 | 빠른 반응 후 점진적 안정화 |

실제 shader에서 enum을 그대로 저장할 필요는 없다. `historyLength`, `lockLifetime`, `stability`, `reactivity`의 조합으로 암묵적인 상태를 만들 수도 있다.

### 4.2 History Length 갱신 방식

#### 정수형 frame count

```text
if (!valid)
    length = 0;
else
    length = min(previousLength + 1, maxLength);
```

장점:

- 의미가 명확함
- 디버그 visualization이 쉬움
- running average weight 계산에 직접 사용 가능

단점:

- 유효/무효 사이의 중간 상태 표현이 어려움
- 작은 reactive event에도 완전 reset하면 불안정할 수 있음

#### Fractional history age

```text
if (!valid)
    length = 0;
else
    length = clamp(previousLength + growth - penalty, 0, maxLength);
```

`penalty`에는 다음이 들어갈 수 있다.

- motion magnitude
- luminance difference
- reactive mask
- depth gradient
- velocity divergence
- topology change signal

이 방식에서는 history가 완전히 무효하지 않더라도 신뢰도가 감소할 수 있다.

```text
정적인 opaque surface
→ growth = 1.0, penalty = 0.0

빠른 rigid motion 내부
→ growth = 0.7, penalty = 0.1

particle overlap
→ growth = 0.2, penalty = 1.0

큰 disocclusion
→ immediate reset
```

### 4.3 History Length와 blend weight를 분리한다

History length를 그대로 blend weight로 사용하면 오래된 pixel이 지나치게 느리게 반응할 수 있다.

따라서 다음처럼 분리하는 편이 좋다.

```text
w_from_age       = ageToWeight(historyLength)
w_from_motion    = motionConfidence
w_from_reactive  = 1 - reactiveMask
w_from_shading   = shadingConfidence

historyWeight = min(
    w_from_age,
    w_from_motion,
    w_from_reactive,
    w_from_shading
)
```

History length는 “최대 얼마까지 믿을 수 있는가”를 제공하고, 현재 frame의 signal은 그 상한을 낮춘다.

```text
오래 안정된 pixel
→ 높은 weight가 허용됨

오래 안정되었지만 현재 reactive event 발생
→ 즉시 낮은 weight 적용
```

즉, age는 inertia를 주지만 현재 변화보다 우선해서는 안 된다.

### 4.4 Lock 생성

Lock은 모든 안정 pixel에 만드는 것이 아니다. 일반적으로 다음 성질을 가진 pixel이 후보가 된다.

- local luminance contrast가 높음
- feature width가 작음
- jitter에 따라 coverage가 크게 바뀜
- neighborhood에서 고립된 밝거나 어두운 detail
- 여러 frame 동안 유사한 위치로 reproject됨

개념적인 thin-feature detector는 다음과 같이 생각할 수 있다.

```text
localRange = maxLuma3x3 - minLuma3x3
centerContrast = abs(centerLuma - neighborhoodMean)
thinFeature = localRange > threshold
              && centerContrast > featureThreshold
```

FSR2의 공개 설명에서는 current luminance의 `3×3 neighborhood`를 이용해 thin feature 후보를 찾고, lock lifetime과 lock 생성 당시 luminance를 저장한다. 이후 shading change가 크면 lock을 해제한다.

Lock 생성은 보통 render resolution에서 feature를 검출하더라도, temporal upscaler에서는 presentation resolution의 output pixel 상태로 관리될 수 있다. 이때 하나의 low-resolution input pixel이 여러 output pixel에 영향을 주므로 좌표계와 footprint 해석이 중요하다.

### 4.5 Lock lifetime과 decay

Lock은 영구 상태가 아니다.

```text
lockLifetime_t = max(
    lockLifetime_{t-1} - decayPerFrame,
    0
)
```

새 lock이 검출되면 lifetime을 다시 채운다.

```text
if (newLock)
    lockLifetime = initialLockLifetime;
```

Jitter sequence 길이와 lock decay를 연결할 수 있다.

```text
한 jitter cycle 동안 feature를 충분히 관측
→ lock lifetime이 cycle 길이와 비슷한 시간 규모를 가짐
```

하지만 다음 event에서는 즉시 또는 빠르게 lock을 해제해야 한다.

```text
if (disoccluded || cameraCut || largeShadingChange || highlyReactive)
    lockLifetime = 0;
```

Lock을 너무 오래 유지하면 ghosting과 stale detail이 생기고, 너무 짧게 유지하면 thin feature가 다시 shimmer한다.

### 4.6 Lock 상태와 color rectification의 연결

Lock이 활성화된 pixel에서는 neighborhood clipping을 완전히 끄기보다 강도를 조절하는 편이 안전하다.

```text
clipStrength = lerp(
    normalClipStrength,
    relaxedClipStrength,
    lockConfidence
)
```

또는 clipping box를 확장할 수 있다.

```text
boxExtent = baseExtent · lerp(1.0, lockExpansion, lockConfidence)
```

좋은 설계는 lock을 binary switch가 아니라 `lockConfidence`로 취급한다.

```text
lockConfidence =
    lockLifetime
    · stability
    · shadingConsistency
    · (1 - reactivity)
```

이렇게 하면 lock 생성 직후와 만료 직전의 변화가 부드럽다.

### 4.7 Stability state가 제어할 수 있는 항목

Temporal stability는 단순히 history weight만 제어하지 않는다.

```text
Temporal stability
├─ history blend weight
├─ clipping / variance box 크기
├─ spatial filter radius
├─ denoiser variance estimate
├─ sharpening strength
├─ reactive response speed
├─ sample count confidence
└─ reconstruction kernel 선택
```

예를 들어 temporal denoiser에서는 history length가 짧은 영역일수록 spatial filter를 더 강하게 적용할 수 있다.

```text
history 짧음
→ temporal sample 부족
→ variance 높음
→ spatial filter radius 증가

history 김
→ temporal convergence 충분
→ spatial blur 감소
→ detail 보존
```

반대로 temporal upscaling에서는 stable thin feature에 lock을 적용해 rectification을 완화하고, unstable 영역에서는 current sample과 neighborhood 제약을 더 강하게 사용할 수 있다.

### 4.8 GPU memory layout

Temporal metadata는 full-screen resource이므로 format 선택이 bandwidth에 직접 영향을 준다.

#### 선택지 A: `R8_UINT` history length

```text
0~255 frame의 정확한 age 저장
```

장점:

- 정수 의미가 명확함
- nearest sampling과 load/store에 적합
- 1 byte per pixel

주의점:

- 일부 API와 hardware에서 typed UAV 지원 확인 필요
- linear filtering 대상이 아님
- temporal upscaling에서 bilinear reprojection 시 직접 gather 정책 필요

#### 선택지 B: `R8_UNORM` stability

```text
0.0 = invalid
1.0 = fully stable
```

장점:

- normalized confidence 표현에 적합
- filtering과 packing이 쉬움
- 1 byte per pixel

단점:

- 정확한 frame count가 아님
- decay와 growth가 quantization에 민감할 수 있음

#### 선택지 C: `RG16_FLOAT` lock state

```text
R = lock lifetime 또는 lock confidence
G = lock 생성 당시 luminance
```

장점:

- lock 해제용 shading comparison을 함께 저장 가능
- HDR luminance 대응이 쉬움

단점:

- 4 bytes per pixel
- output resolution에서 유지하면 bandwidth가 커짐

#### 선택지 D: packed `R32_UINT`

예시:

```text
bits  0~7   : history length
bits  8~15  : stability
bits 16~23  : lock lifetime
bits 24~27  : state flags
bits 28~31  : reset/debug flags
```

장점:

- 하나의 resource load로 여러 metadata 확보
- format 수와 descriptor 수 감소

단점:

- bit operation 증가
- bilinear sampling 불가
- precision과 range 변경이 어려움
- shader debugging과 tool inspection이 불편함

실무에서는 bandwidth 절감만 보고 과도하게 packing하기보다, **debug 가능성·API format 지원·향후 알고리즘 변경 가능성**까지 고려해야 한다.

### 4.9 Ping-pong history와 frame graph

History color와 metadata는 이전 frame을 읽고 현재 frame 결과를 써야 한다.

```text
Frame N
Read : historyColor[A], temporalState[A]
Write: historyColor[B], temporalState[B]

Frame N+1
Read : historyColor[B], temporalState[B]
Write: historyColor[A], temporalState[A]
```

같은 texture를 동시에 read/write하면 pixel 간 dependency와 race condition이 생길 수 있으므로 일반적으로 ping-pong resource를 사용한다.

Compute shader 기준으로는 다음 항목을 확인해야 한다.

- SRV/UAV transition
- previous/current descriptor swap
- dynamic resolution 변경 시 resource 재할당
- camera cut 시 clear 또는 reset flag
- async compute 사용 시 graphics queue와 semaphore/barrier
- output resolution state와 render resolution input의 coordinate transform

Temporal metadata는 작아 보이지만 full-screen pass마다 읽고 쓰므로 cache locality와 resource 수가 중요하다.

예를 들어 4K output에서 `RG16_FLOAT` lock texture 하나는 약 31.6 MiB의 raw pixel storage가 필요하다.

```text
3840 × 2160 × 4 bytes ≈ 31.6 MiB
```

Ping-pong이면 약 63.3 MiB이며, 매 frame read/write bandwidth까지 발생한다. 따라서 lock lifetime을 더 작은 format으로 표현할 수 있는지, lock luminance가 별도 resource에 꼭 필요한지 검토할 가치가 있다.

### 4.10 Reprojection 시 state sampling

History color는 bilinear 또는 bicubic filter로 sample할 수 있지만, metadata는 같은 방식으로 다루기 어렵다.

예를 들어 history length를 bilinear sample하면 서로 다른 surface의 age가 섞일 수 있다.

```text
foreground history length = 30
background history length = 2
bilinear result = 16
```

이 값은 실제로 존재하지 않는 temporal state다.

따라서 metadata에는 다음 정책이 적합할 수 있다.

- nearest sample
- 2×2 gather 후 depth와 가장 일치하는 sample 선택
- valid sample 중 minimum history length 사용
- lock은 conservative max 또는 matching surface sample 사용
- confidence는 bilinear 후 geometric validity로 감쇠

선택 기준은 metadata의 의미에 따라 다르다.

```text
History length
→ 잘못 높게 평가하는 것이 위험
→ conservative min이 유리할 수 있음

Reactive mask
→ 놓치는 것이 위험
→ conservative max가 유리할 수 있음

Lock status
→ 잘못 유지하면 ghosting
→ surface match가 중요
```

### 4.11 Reset과 decay를 구분한다

모든 변화가 full reset을 요구하는 것은 아니다.

#### 즉시 reset이 적합한 경우

- camera cut
- object teleport
- invalid motion vector
- history coordinate가 viewport 밖
- 큰 depth mismatch
- render target 재생성
- simulation dataset 교체

#### 점진적 decay가 적합한 경우

- moderate motion 증가
- 작은 shading 변화
- reactive mask의 일시적 증가
- volume transfer function의 작은 조정
- CFD field의 연속적인 timestep 변화

```text
Reset
→ 과거를 완전히 폐기

Decay
→ 과거의 영향만 빠르게 낮춤
```

Reset을 남발하면 flicker와 noise가 증가하고, decay만 사용하면 명백히 잘못된 history가 남을 수 있다.

### 4.12 Debug visualization

Temporal artifact는 최종 color만 보면 원인을 구분하기 어렵다. 다음 buffer를 직접 시각화하는 것이 효과적이다.

```text
History Length
- 검정: 0
- 밝음: max age

Stability
- 낮음: reactive / invalid
- 높음: stable

Lock Lifetime
- lock이 생성되고 감소하는 과정

Reset Reason
- depth mismatch
- motion invalid
- shading change
- camera cut
- reactive event
```

특히 lock이 ghosting을 만드는지, clipping이 thin detail을 제거하는지 구분하려면 다음 비교가 필요하다.

```text
Lock OFF + clipping ON
Lock ON  + clipping ON
Lock ON  + clipping OFF
```

최종 영상의 품질만 비교하기보다 state buffer와 artifact 위치를 함께 보는 것이 graphics engineer 관점에서 중요하다.

---

## 5. 내 관심 분야와 연결

### 5.1 Real-time rendering과 game engine

TAA, TSR, FSR 계열 temporal upscaler는 모두 history를 사용하지만, 좋은 품질을 만드는 핵심은 단순 reprojection이 아니라 **history의 수명 관리**다.

Game engine에서는 다음 event가 temporal state에 영향을 준다.

- camera cut과 cinematic transition
- dynamic resolution scaling
- LOD 전환
- animated material
- particle와 transparency
- skeletal mesh teleport
- Nanite-like geometry visibility 변화
- reflection과 shadow의 stochastic sampling 변화

Renderer architecture에서는 각 subsystem이 temporal reset 또는 reactive metadata를 전달할 수 있어야 한다.

```text
Material/VFX system
→ reactive signal

Camera system
→ cut/reset signal

Resolution manager
→ resize/reset policy

Geometry system
→ object motion / teleport

Temporal resolve
→ history length / lock / stability update
```

이는 temporal algorithm이 post-process 하나가 아니라 engine-wide integration 문제라는 뜻이다.

### 5.2 GPU compute와 shader programming

History state update는 compute shader에 잘 맞는다.

- output pixel당 독립적인 state update
- depth·normal·velocity·history metadata gather
- shared memory를 이용한 neighborhood luminance 분석
- wave operation을 이용한 local min/max 계산
- packed metadata의 bit manipulation

다만 neighborhood 기반 lock 검출과 reprojection gather는 texture cache 사용 패턴이 복잡하다. Workgroup 크기와 tile halo를 설계할 때 다음을 고려해야 한다.

- 8×8 또는 16×16 tile
- 3×3 neighborhood를 위한 shared-memory border
- output resolution과 render resolution의 비율
- history color와 state의 descriptor 수
- divergent branch보다 mask 기반 연산이 유리한지

### 5.3 CFD와 scientific visualization

Scientific visualization에서도 temporal state는 유용하지만, game rendering과 다른 invalidation 원인이 존재한다.

#### Volume rendering

Camera가 정지한 상태에서 stochastic ray marching 또는 low-sample volume rendering을 temporal accumulation으로 안정화할 수 있다.

History를 reset 또는 decay해야 하는 event:

- transfer function 변경
- clipping plane 이동
- timestep 변경
- volume data streaming 완료
- sampling step size 변경
- opacity correction 변경

특히 transfer function은 geometry depth를 바꾸지 않으면서 appearance를 크게 바꾼다. 따라서 depth 기반 validation만으로는 부족하고, transfer-function revision ID 또는 explicit reset signal이 필요하다.

#### Marching Cubes와 Level-set surface

Level-set이 시간에 따라 변하면 surface topology가 바뀔 수 있다.

- vertex가 생성되거나 사라짐
- triangle connectivity 변경
- normal 방향 급변
- motion vector correspondence 불완전

이 경우 object ID가 같더라도 history length를 그대로 유지하면 ghosting이 발생할 수 있다. Scalar field 변화량, cell activation 변화, topology revision signal을 stability penalty에 반영할 수 있다.

#### Streamline과 particle visualization

Streamline seed나 integration result가 갱신되면 screen-space 형태가 갑자기 바뀔 수 있다. Geometry motion vector만으로는 새 streamline과 이전 streamline의 correspondence를 보장하지 못한다.

```text
동일한 dataset + 동일한 seed
→ history 유지 가능

seed 변경 또는 vector field 변경
→ length reset 또는 빠른 decay
```

### 5.4 Sparse voxel과 octree

Octree LOD가 바뀌면 동일한 volume을 표현하더라도 sample footprint와 surface detail이 달라질 수 있다.

- brick streaming 완료
- LOD refinement
- missing brick 대체
- transmittance 결과 변화

이러한 event는 temporal state에 명시적으로 전달하는 것이 좋다.

```text
새 brick이 로드됨
→ 해당 screen region의 stability 감소
→ current contribution 증가
→ 몇 frame 동안 재수렴
```

전체 screen history를 reset하지 않고, 영향받은 region만 invalidation하는 방식이 품질과 안정성에 유리하다.

### 5.5 C++ renderer architecture

C++ 측에서는 temporal pass에 단순 texture handle만 넘기기보다 frame-level invalidation 정보를 구조화할 수 있다.

```text
TemporalFrameState
- cameraCut
- resolutionChanged
- jitterSequenceReset
- exposureScale
- sceneRevision
- simulationTimestepRevision
- transferFunctionRevision
- topologyRevision
```

이 정보는 shader constant 또는 dispatch configuration으로 전달된다.

좋은 renderer는 reset reason을 bit flag로 남겨 GPU debug output과 연결할 수 있다.

```text
RESET_CAMERA_CUT
RESET_RESOLUTION
RESET_SCENE_REVISION
RESET_SIMULATION_DATA
RESET_INVALID_MOTION
```

이 구조는 artifact를 재현하고 원인을 추적하는 데 큰 도움이 된다.

---

## 6. 머릿속에 남길 질문 3개

1. **History length를 단순 frame count로 저장하는 방식과 continuous stability confidence로 저장하는 방식은 각각 어떤 temporal artifact를 다루기 쉬운가?**

2. **Temporal lock이 thin detail을 보존하면서도 stale history와 ghosting을 만들지 않으려면 어떤 생성 조건, lifetime, 해제 조건이 필요한가?**

3. **CFD volume, marching-cubes surface, octree streaming처럼 geometry와 appearance가 비동기적으로 변하는 시스템에서는 어떤 revision signal을 temporal state invalidation에 연결해야 하는가?**

---

## 7. Graphics Engineer 면접 질문 1개와 답변

### 질문

**Temporal accumulation에서 history length와 lock status를 별도로 관리하는 이유와, GPU resource로 구현할 때 고려해야 할 점을 설명해 주세요.**

### 답변

History length는 해당 pixel의 history가 몇 frame 동안 유효하게 유지되었는지 또는 어느 정도의 effective sample count를 확보했는지를 나타냅니다. History가 짧은 pixel은 아직 temporal convergence가 부족하므로 current sample의 비중을 높이거나 spatial filtering을 강화할 수 있고, history가 긴 pixel은 높은 history weight를 허용해 flicker와 noise를 줄일 수 있습니다.

Lock status는 history validity와 목적이 다릅니다. Temporal upscaling에서는 전선이나 철망 같은 thin feature가 jitter sequence의 일부 frame에서만 관측될 수 있는데, 현재 neighborhood만으로 history를 clamp하면 이러한 detail이 반복적으로 사라지고 나타납니다. Lock은 안정적인 thin feature를 일정 기간 clipping이나 rectification으로부터 보호합니다.

다만 lock은 영구적이면 안 됩니다. Disocclusion, 큰 luminance 변화, reactive mask, invalid motion, camera cut이 발생하면 lock을 해제해야 stale detail과 ghosting을 막을 수 있습니다. 따라서 lock lifetime과 생성 당시 luminance 또는 shading confidence를 함께 저장하는 방식이 유용합니다.

GPU 구현에서는 history state가 full-screen read/write resource이므로 format과 bandwidth가 중요합니다. History length는 `R8_UINT` 또는 normalized stability라면 `R8_UNORM`으로 저장할 수 있고, lock lifetime과 기준 luminance는 `RG16_FLOAT` 같은 format을 사용할 수 있습니다. 여러 값을 `R32_UINT`에 packing할 수도 있지만 filtering이 불가능하고 디버깅과 알고리즘 변경이 어려워질 수 있습니다.

또한 previous state와 current state를 동시에 사용하므로 ping-pong texture가 일반적이며, metadata reprojection에서는 bilinear sampling으로 서로 다른 surface의 상태가 섞이지 않도록 nearest 또는 depth-aware gather를 고려해야 합니다. Camera cut, dynamic resolution, scene revision 같은 reset event는 C++ renderer에서 명시적으로 temporal pass에 전달해야 합니다.

---

## 8. 포트폴리오 / 커리어 연결

Temporal rendering 포트폴리오에서 단순히 “TAA history buffer를 사용했다”는 설명보다 다음 구조를 제시하면 엔진 설계 역량이 더 잘 드러난다.

```text
Temporal State Architecture

Input signals
- motion vector
- depth / normal
- luminance difference
- reactive mask
- scene revision

Persistent metadata
- history length
- stability confidence
- lock lifetime
- lock reference luminance

State transitions
- invalid
- warming
- stable
- locked
- responsive

Output policies
- history weight
- clipping strength
- spatial filter radius
- reset / decay
```

포트폴리오에서 특히 강한 요소는 다음과 같다.

- history length heatmap
- lock lifetime visualization
- reset reason visualization
- camera cut 전후 비교
- thin geometry에서 lock on/off 비교
- particle와 transparency의 state decay 비교
- dynamic resolution 전환 시 history policy
- 4K에서 metadata bandwidth 계산
- `R8_UINT`, `RG16_FLOAT`, packed `R32_UINT` format trade-off
- compute shader dispatch와 frame graph barrier 설명

Scientific visualization 포트폴리오라면 다음 연결이 차별점이 된다.

- transfer function revision에 따른 volume history invalidation
- timestep 변화량에 따른 stability decay
- marching-cubes topology revision과 history reset
- octree brick streaming region의 부분 invalidation
- camera는 정적이지만 dataset이 변하는 경우의 temporal policy

이 주제는 graphics engineer 면접에서 다음 역량을 동시에 보여준다.

- temporal algorithm 이해
- GPU memory layout 설계
- shader와 C++ renderer integration
- frame graph와 resource lifetime
- artifact 원인 분석
- game rendering과 scientific visualization의 차이 이해

---

## 9. 내일 이어서 볼 개념

**Temporal Reset Policies: Camera Cuts, Resolution Changes, and Scene Revisions**

다음에는 history state를 언제 완전히 버리고 언제 점진적으로 decay해야 하는지 살펴본다.

이어지는 핵심 질문은 다음과 같다.

> **Camera cut, dynamic resolution, LOD change, simulation timestep update처럼 서로 다른 event를 하나의 reset flag로 처리하지 않고, event별 invalidation policy로 나누면 어떤 품질과 성능 이점이 생기는가?**

---

## 10. 참고 키워드

- History Length
- Effective Sample Count
- Temporal Stability
- Temporal Lock
- Lock Lifetime
- Thin Feature Preservation
- Sub-pixel Detail
- Temporal Accumulation
- History Confidence
- History Validation
- History Rectification
- Neighborhood Clamping
- Variance Clipping
- Adaptive History Weighting
- Reactive Mask
- Shading Change Detection
- Hysteresis
- Temporal State Machine
- Ping-pong History Buffer
- Metadata Reprojection
- R8_UINT
- R8_UNORM
- RG16_FLOAT
- Packed R32_UINT
- Dynamic Resolution
- Camera Cut
- Scene Revision
- Temporal Denoising
- Temporal Upscaling
- TAA
- TSR
- FSR2
- SVGF
- Scientific Visualization
- Volume Rendering
- Marching Cubes
- Octree Streaming

### 참고 자료

- [AMD FidelityFX Super Resolution 2 — Create Locks / Reproject and Accumulate](https://github.com/GPUOpen-Effects/FidelityFX-FSR2)
- [Lei Yang, Shiqiu Liu, Marco Salvi — A Survey of Temporal Antialiasing Techniques](https://research.nvidia.com/labs/rtr/publication/yang2020survey/)
- [Christoph Schied et al. — Spatiotemporal Variance-Guided Filtering](https://research.nvidia.com/labs/rtr/publication/schied2017spatiotemporal/)
- [Brian Karis — High Quality Temporal Supersampling](https://advances.realtimerendering.com/s2014/)
