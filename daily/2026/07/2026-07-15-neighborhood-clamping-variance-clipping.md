---
title: "Neighborhood Clamping and Variance Clipping"
date: "2026-07-15"
category: "Graphics"
tags: ["GPU", "Temporal Anti-Aliasing", "Temporal Reprojection", "Neighborhood Clamping", "Variance Clipping", "History Validation", "YCoCg", "Compute Shader", "Memory Layout", "Real-time Rendering"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-07-15 - Neighborhood Clamping and Variance Clipping

## 1. 오늘의 개념

**Neighborhood Clamping**과 **Variance Clipping**은 temporal reprojection으로 가져온 이전 프레임의 history color가 현재 프레임 주변에서 관측되는 색 분포와 지나치게 다를 때, history를 현재 프레임이 허용하는 범위 안으로 제한하는 기술이다.

Temporal Anti-Aliasing(TAA), temporal upscaling, ray tracing denoising은 이전 프레임의 정보를 재사용해 적은 현재 샘플로 더 안정적인 결과를 만든다. 하지만 motion vector가 가리킨 위치가 기하학적으로 유효하더라도, 다음과 같은 이유로 history color가 현재 pixel과 맞지 않을 수 있다.

- foreground가 이동하며 새로 드러난 **disocclusion** 영역
- 얇은 geometry가 jitter에 따라 나타났다 사라지는 경우
- specular highlight가 빠르게 이동하는 경우
- alpha-tested foliage, hair, particle의 coverage가 바뀌는 경우
- LOD, tessellation, marching cubes topology가 변한 경우
- dynamic exposure 또는 lighting이 급격히 바뀐 경우
- low-resolution effect를 output resolution으로 재구성하는 경우

핵심 아이디어는 단순하다.

> 이전 프레임의 색이 현재 pixel 주변에서 관측 가능한 색 범위 밖에 있다면, 그대로 누적하지 않는다.

여기서 **Neighborhood Min/Max Clamping**은 주변 색의 최소·최대 범위를 사용하고, **Variance Clipping**은 주변 색의 평균과 표준편차를 이용해 통계적인 허용 범위를 만든다.

---

## 2. 한 줄 핵심

**Neighborhood Clamping과 Variance Clipping은 reprojected history를 현재 frame의 local color distribution 안으로 제한하여 ghosting을 줄이되, 허용 범위를 너무 좁히면 temporal stability가 무너지고 flickering이 증가한다.**

---

## 3. 왜 중요한가

Temporal rendering의 품질은 단순히 history weight를 크게 주는가, 작게 주는가로 결정되지 않는다. 먼저 **재사용하려는 history가 현재 frame에서 합리적인 값인지**를 판정해야 한다.

History가 현재 surface와 맞지 않는데 높은 weight로 누적되면 다음 artifact가 발생한다.

- moving object 뒤에 이전 색이 남는 **ghost trail**
- 밝은 specular가 화면에 끌려 다니는 현상
- particle과 transparency가 번지는 현상
- camera cut 이후 이전 장면이 잔류하는 현상
- volume rendering에서 이전 hit point의 색이 남는 현상

반대로 history를 지나치게 강하게 제한하면 temporal accumulation이 사실상 매 frame 초기화된다.

- thin line과 fence가 깜빡임
- sub-pixel geometry가 안정적으로 누적되지 않음
- stochastic effect의 noise가 줄지 않음
- temporal upscaling에서 세부 디테일이 불안정해짐

따라서 이 기술의 본질은 다음 trade-off다.

```text
넓은 허용 범위
→ history를 많이 유지
→ temporal stability 증가
→ ghosting 위험 증가

좁은 허용 범위
→ history를 자주 수정 또는 제거
→ ghosting 감소
→ flickering과 noise 증가
```

Graphics engineer에게 중요한 점은 clamp를 단순한 후처리 보정으로 보지 않고, **temporal correspondence의 신뢰도를 color domain에서 검증하는 단계**로 이해하는 것이다.

---

## 4. 구현 관점

### 4.1 Temporal resolve 안에서의 위치

일반적인 temporal resolve는 다음 흐름을 가진다.

```text
Current color / depth / normal / velocity 생성
                ↓
Velocity dilation 또는 motion vector 보정
                ↓
Previous history 위치 reprojection
                ↓
Depth / normal / object ID 기반 geometric validation
                ↓
Current neighborhood color distribution 계산
                ↓
Neighborhood clamping 또는 variance clipping
                ↓
History weight 계산
                ↓
Current와 clipped history accumulation
```

Geometric validation은 “같은 surface인가?”를 판단하고, neighborhood clipping은 “같은 surface라고 가정해도 이 color가 현재 관측 범위에 들어오는가?”를 판단한다.

둘은 대체 관계가 아니다.

```text
Geometry validation → correspondence 검사
Color clipping      → radiometric consistency 검사
```

### 4.2 Neighborhood Min/Max Clamping

현재 pixel 주변 `3×3` 영역의 current color를 읽어 component-wise minimum과 maximum을 구한다.

```text
Cmin = min(C0, C1, ... C8)
Cmax = max(C0, C1, ... C8)
```

Reprojected history `H`는 다음 범위로 제한된다.

```text
H_clamped = clamp(H, Cmin, Cmax)
```

장점은 단순하고 빠르다는 것이다.

- `3×3` 기준 9개 current sample
- component-wise min/max reduction
- history 1회 clamp
- branchless shader로 구성하기 쉬움

하지만 local neighborhood에 강한 outlier가 하나만 있어도 허용 box가 크게 확장된다.

예를 들어 어두운 surface 위에 단 하나의 매우 밝은 specular pixel이 존재하면 RGB max가 커지고, 잘못된 밝은 history도 유효 범위로 받아들여질 수 있다.

### 4.3 Clamp와 Clip의 차이

실무 문서에서 clamping과 clipping은 혼용되기도 하지만, 엄밀히 구분하면 결과가 다르다.

**Component-wise clamp**는 history의 각 channel을 독립적으로 box 안에 넣는다.

```text
H' = clamp(H, Cmin, Cmax)
```

**Segment clipping**은 current sample `C`에서 history `H`로 향하는 선분을 color AABB와 교차시켜 경계점까지 history를 당긴다.

```text
P(t) = C + t(H - C),  0 ≤ t ≤ 1
```

History가 box 밖에 있으면 가장 큰 유효 `t`를 찾아 다음 값을 사용한다.

```text
H' = C + t_valid(H - C)
```

Segment clipping은 history를 current color 방향으로 되돌리므로, channel별 독립 clamp보다 원래 color 변화 방향을 더 잘 보존하는 경우가 있다.

### 4.4 Variance Clipping

Variance Clipping은 neighborhood의 첫 번째와 두 번째 raw moment를 계산한다.

```text
m1 = E[C]
m2 = E[C²]
```

평균과 분산은 다음과 같다.

```text
μ  = m1
σ² = max(0, m2 - μ²)
σ  = sqrt(σ²)
```

허용 범위는 평균을 중심으로 표준편차의 배수만큼 설정한다.

```text
Cmin_variance = μ - γσ
Cmax_variance = μ + γσ
```

여기서 `γ(gamma)`는 temporal stability와 ghosting 사이의 핵심 parameter다.

- 큰 `γ`: 넓은 분포를 허용, history 유지 증가, ghosting 위험 증가
- 작은 `γ`: 강한 rejection, ghosting 감소, flickering 위험 증가

Marco Salvi의 temporal supersampling 발표에서는 `γ = 1`을 실용적인 출발점으로 제시한다. 실제 engine에서는 motion, confidence, material, luminance contrast에 따라 adaptive하게 바꿀 수 있다.

### 4.5 Min/Max와 Variance 범위를 함께 사용하는 이유

Variance box만 사용하면 표준편차가 큰 영역에서 허용 범위가 실제 neighborhood의 관측 최소·최대를 넘어갈 수 있다.

따라서 다음처럼 두 범위를 교차시키는 방식이 안정적이다.

```text
Cmin_final = max(Cmin_neighborhood, μ - γσ)
Cmax_final = min(Cmax_neighborhood, μ + γσ)
```

이 방식은 다음 장점을 가진다.

- isolated outlier가 min/max box를 과도하게 키우는 문제 완화
- variance가 실제 관측 범위 밖까지 확장되는 문제 방지
- 매우 균일한 영역에서는 history를 강하게 제한
- 복잡한 texture 영역에서는 필요한 분산을 일부 허용

### 4.6 RGB 공간의 한계

RGB에서 component-wise box를 만들면 channel 간 상관관계를 무시한다.

예를 들어 neighborhood가 red와 green 계열만 포함해도 RGB AABB 내부에는 실제로 존재하지 않았던 yellow 계열이 포함될 수 있다. 즉 AABB는 실제 color distribution의 convex hull보다 훨씬 큰 공간을 허용한다.

이 때문에 temporal resolve에서는 종종 **YCoCg** 같은 decorrelated color space를 사용한다.

```text
RGB → YCoCg
Y  : luminance-like component
Co : orange-cyan chroma axis
Cg : green-magenta chroma axis
```

YCoCg에서 clipping하면 luminance와 chroma를 어느 정도 분리해 제어할 수 있다.

예를 들면 다음과 같은 정책이 가능하다.

- luminance `Y`에는 더 엄격한 범위
- chroma `Co/Cg`에는 조금 더 넓은 범위
- low-luminance 영역에서 chroma confidence 축소

다만 YCoCg가 perceptually uniform한 color space는 아니다. 장점은 변환 비용이 낮고 RGB보다 channel correlation을 줄이기 쉽다는 데 있다.

### 4.7 HDR과 exposure 처리

HDR linear color에서 variance를 계산하면 매우 밝은 emissive나 specular가 통계를 지배할 수 있다.

```text
대부분의 sample: 0.1 ~ 1.0
specular outlier: 50.0
```

이 경우 평균과 표준편차가 커져 history rejection이 약해질 수 있다.

실무에서는 다음 관점을 고려한다.

- pre-exposed color에서 current와 history의 exposure 기준 통일
- luminance compression 후 통계 계산
- luminance와 chroma에 서로 다른 gamma 사용
- emissive 또는 specular responsive mask 연동
- exposure가 크게 바뀐 frame에서 history confidence 감소

Current와 history가 서로 다른 exposure scale에 있으면 clipping 이전에 history exposure compensation을 해야 한다. 그렇지 않으면 올바른 history도 범위 밖으로 판정된다.

### 4.8 Numerical precision과 memory layout

Variance 계산은 `m2 - μ²` 형태이므로 두 값이 매우 가까울 때 cancellation error가 발생할 수 있다.

따라서 입력 color가 `RGBA16F`여도 accumulation은 FP32 register에서 수행하는 편이 안전하다.

```text
Input texture : RGBA16F 또는 R11G11B10F
Moments       : shader 내부 FP32
History       : RGBA16F / RGBA32F
Metadata      : R8_UNORM confidence 또는 packed mask
```

분산은 반드시 음수 방어를 해야 한다.

```text
variance = max(secondMoment - mean * mean, 0)
```

특히 dark region, 거의 균일한 gradient, FP16 intermediate에서 이 처리가 없으면 `sqrt`에 NaN이 들어갈 수 있다.

### 4.9 Compute Shader와 tile reuse

Neighborhood statistics는 인접 pixel이 같은 sample을 반복해서 읽는다. Compute shader에서는 workgroup tile을 group shared memory에 적재해 global texture load를 줄일 수 있다.

예를 들어 `8×8` output tile과 `3×3` kernel을 사용하면 1-pixel halo를 포함한 `10×10` 영역을 shared memory에 적재한다.

```text
Global current color
        ↓
Shared tile + halo
        ↓
각 thread가 3×3 min/max, m1, m2 계산
        ↓
History clipping + accumulation
```

메모리와 실행 관점에서 확인할 항목은 다음과 같다.

- halo load의 중복과 synchronization 비용
- texture cache가 충분한 경우 shared memory가 실제로 이득인지
- RGB와 YCoCg 변환을 load 직후 한 번만 수행하는지
- min/max와 moment 계산을 같은 loop에서 처리하는지
- wave operation으로 reduction할 필요가 있는지
- dynamic resolution에서 current/output coordinate가 일치하는지

작은 `3×3` kernel은 texture cache가 잘 동작할 수 있으므로, shared memory 사용이 항상 빠른 것은 아니다. GPU architecture와 pass fusion 여부를 함께 프로파일링해야 한다.

### 4.10 Kernel 크기의 의미

`3×3` neighborhood는 경계 보존과 비용 사이의 일반적인 선택이다.

- 장점: local detail 유지, 낮은 bandwidth, 적은 bleeding
- 단점: stochastic sample이나 sparse coverage의 분포를 충분히 추정하지 못할 수 있음

`5×5` 또는 `7×7`은 더 안정적인 통계를 제공할 수 있지만 서로 다른 surface와 material을 섞을 가능성이 커진다.

큰 kernel은 특히 다음 문제를 만든다.

- silhouette 양쪽의 foreground/background color 혼합
- thin geometry가 neighborhood 통계에서 희석
- texture edge를 지나치게 넓게 허용
- ghosting 증가
- bandwidth와 register pressure 증가

큰 영역이 필요한 경우 단순히 kernel을 키우기보다 moment prefilter, hierarchical statistics, depth-aware neighborhood를 고려할 수 있다.

### 4.11 Variance Clipping의 대표적인 실패 사례

Variance clipping은 ghosting을 줄이지만 모든 temporal artifact를 해결하지 않는다.

#### Thin feature가 현재 frame에서 사라진 경우

Jitter에 의해 한 frame에서 얇은 선이 sample되지 않으면 neighborhood variance가 급격히 작아진다. 이전 frame에 존재하던 올바른 line history도 범위 밖으로 clip되어 accumulation이 reset된다.

이 현상이 반복되면 ghosting 대신 flickering이 발생한다.

#### 서로 다른 surface가 같은 color를 가진 경우

Depth나 object ID 검증 없이 color 범위만 사용하면, 다른 surface라도 비슷한 color일 경우 history가 통과할 수 있다.

#### 매우 noisy한 current input

Monte Carlo noise가 큰 current neighborhood에서는 variance가 넓어져 잘못된 history까지 허용할 수 있다. 반대로 sparse stochastic sample에서는 실제 분포를 충분히 추정하지 못해 과도한 clipping이 발생할 수 있다.

#### Topology가 바뀌는 geometry

Particle spawn/death, marching cubes cell activation, sparse voxel LOD transition처럼 stable correspondence가 없는 경우에는 color clipping보다 history confidence와 topology change mask가 더 중요하다.

### 4.12 Adaptive gamma와 confidence

고정 gamma는 구현이 단순하지만 모든 상황에 최적이지 않다.

실무적인 temporal resolve에서는 다음 정보로 gamma 또는 history weight를 조절할 수 있다.

```text
빠른 motion             → gamma 축소 또는 history weight 감소
높은 depth discontinuity → gamma 축소
높은 reactive mask       → gamma 축소
안정적인 static surface  → gamma 확대 가능
높은 sample count         → gamma 축소 가능
noisy stochastic input    → gamma 확대 또는 별도 denoising 통계 사용
```

중요한 점은 gamma와 history blend weight가 서로 다른 역할을 가진다는 것이다.

```text
Gamma        : history가 허용되는 color domain의 범위
History weight: 허용된 history를 최종 결과에 얼마나 반영할지
```

두 parameter를 하나의 heuristic으로 묶으면 artifact 원인을 분석하기 어려워진다.

---

## 5. 내 관심 분야와 연결

### Real-time Rendering / Game Engine

TAA, TSR, DLSS류 temporal reconstruction을 이해하려면 motion vector만큼 neighborhood clipping을 중요하게 봐야 한다. Ghosting은 흔히 blend factor 문제처럼 보이지만 실제 원인은 다음 단계 중 하나일 수 있다.

```text
Wrong velocity
→ Wrong reprojection
→ Failed geometry validation
→ Over-wide clipping bounds
→ Excessive history weight
```

Engine architecture에서는 각 단계를 debug visualization으로 분리할 수 있어야 한다.

- raw history color
- clipped history color
- neighborhood min/max
- mean / variance
- history rejection mask
- final history weight

### GPU / Compute Shader

이 pass는 연산량보다 neighborhood texture read와 cache behavior가 중요하다. `3×3` kernel에서 min/max와 first/second moment를 한 loop에서 계산하면 arithmetic intensity를 높이고 중복 load를 줄일 수 있다.

또한 FP16 texture를 사용하더라도 variance accumulator는 FP32로 유지하고, 출력 metadata는 `R8` 또는 packed bitfield로 분리하는 memory layout이 실용적이다.

### WebGPU / Vulkan / DirectX / Metal

API가 달라도 핵심 resource contract는 같다.

- current color sampled texture
- previous history sampled texture
- motion vector / depth / normal
- output history storage texture 또는 render target
- temporal metadata buffer

WebGPU에서는 storage texture format 제약과 read/write aliasing을 고려해야 하고, Vulkan/DirectX에서는 pass barrier와 history ping-pong resource state가 중요하다. Metal에서는 threadgroup memory와 tile-based GPU의 bandwidth 특성을 함께 고려해야 한다.

### CFD / Scientific Visualization

Scientific visualization에서 color는 물리량의 colormap 결과일 수 있다. 이때 screen-space RGB를 clipping하면 실제 scalar field 변화가 아닌 colormap 특성에 따라 history가 제한된다.

예를 들어 pressure scalar가 크게 변했지만 colormap RGB 차이가 작으면 잘못된 history가 통과할 수 있다. 반대로 colormap 경계가 강하면 작은 scalar 변화도 큰 color 변화로 보일 수 있다.

따라서 scientific temporal filtering에서는 가능하면 다음 순서가 더 의미 있다.

```text
Physical scalar/vector validation
→ field gradient / material / region ID 검사
→ temporal accumulation
→ colormap 적용
```

이미 color buffer 단계에서 처리해야 한다면 depth, cell ID, block ID, scalar confidence를 추가 metadata로 활용하는 편이 안전하다.

### Sparse Voxel / Marching Cubes / Volume Rendering

LOD 전환이나 topology 변화가 발생하면 이전 frame의 surface color가 현재 neighborhood 분포 안에 우연히 들어올 수 있다. Color clipping만으로는 stale history를 완전히 제거할 수 없다.

따라서 다음 신호와 함께 사용해야 한다.

- voxel brick generation/version ID
- marching cubes cell activation change
- first-hit depth difference
- volume transmittance difference
- material 또는 phase ID
- topology change mask

즉 이 영역에서는 variance clipping보다 **history identity**가 먼저다.

---

## 6. 머릿속에 남길 질문 3개

1. **현재 neighborhood의 color 범위가 좁다는 것은 history가 틀렸다는 의미인가, 아니면 현재 frame이 thin feature를 놓쳤다는 의미인가?**

2. **Ghosting을 줄이기 위해 gamma를 줄이는 것과 history blend weight를 줄이는 것은 artifact와 temporal convergence 측면에서 어떻게 다른가?**

3. **Scientific visualization에서는 RGB neighborhood가 아니라 scalar field 또는 material identity를 기준으로 history를 검증해야 하는 이유가 무엇인가?**

---

## 7. Graphics Engineer 면접 질문 1개와 답변

### 질문

**Temporal Anti-Aliasing에서 neighborhood min/max clamping과 variance clipping의 차이, 그리고 variance clipping이 flickering을 만들 수 있는 이유를 설명해 주세요.**

### 답변

Neighborhood min/max clamping은 현재 pixel 주변의 color component별 최소·최대를 구해 reprojected history를 그 AABB 안으로 제한하는 방식입니다. 구현이 단순하고 빠르지만, neighborhood에 밝은 specular 같은 outlier가 하나만 있어도 허용 범위가 과도하게 넓어져 ghosting이 남을 수 있습니다.

Variance clipping은 주변 sample의 평균 `μ`와 표준편차 `σ`를 계산해 `μ ± γσ` 범위를 만들고 history를 그 범위로 clip합니다. 통계적으로 outlier의 영향을 줄일 수 있지만, current frame이 jitter 때문에 thin geometry나 high-frequency feature를 놓치면 local variance가 급격히 작아집니다. 그러면 이전 frame의 올바른 history도 범위 밖으로 제거되어 temporal accumulation이 반복적으로 reset되고 flickering이 발생할 수 있습니다.

따라서 실무에서는 variance box를 neighborhood min/max box와 교차시키고, depth·normal·object ID validation, reactive mask, adaptive history weight와 함께 사용합니다. 또한 RGB보다는 YCoCg처럼 luminance와 chroma가 덜 결합된 공간에서 clipping하고, variance 계산은 FP32로 수행해 numerical error를 줄이는 것이 일반적입니다.

---

## 8. 포트폴리오 / 커리어 연결

Neighborhood Clamping과 Variance Clipping은 temporal rendering을 단순한 화면 효과가 아니라 **history validation과 statistical reconstruction 문제**로 이해하고 있음을 보여주기 좋은 주제다.

포트폴리오에서는 다음 구조로 설명할 수 있다.

```text
Problem
- Reprojected history가 current surface와 맞지 않아 ghosting 발생
- 과도한 rejection은 thin geometry flickering 유발

Pipeline
- Velocity reprojection
- Depth / normal / object identity validation
- 3×3 neighborhood statistics
- Min/max + variance bounds
- YCoCg clipping
- Adaptive history weighting

GPU decisions
- RGBA16F history
- FP32 moment accumulation
- 3×3 texture cache 또는 shared tile 비교
- Packed rejection/confidence metadata
- Ping-pong history resource 관리

Evaluation
- Camera motion ghosting
- Disocclusion trail
- Thin geometry flickering
- Specular stability
- Temporal convergence
- GPU bandwidth와 resolve pass cost
```

경력 기술이나 면접에서는 다음처럼 연결할 수 있다.

> “Temporal artifact를 blend factor 하나로 조정하지 않고, velocity correctness, geometric validation, color-domain clipping, adaptive history weight로 분리해 분석합니다. Variance clipping은 ghosting을 줄이지만 thin geometry에서 variance collapse로 flickering을 만들 수 있으므로, YCoCg bounds와 reactive metadata를 함께 설계합니다.”

이 관점은 game engine의 TAA/temporal upscaling뿐 아니라 ray tracing denoising, particle rendering, CFD field visualization, semiconductor 3D structure의 temporal interaction에도 직접 연결된다.

---

## 9. 내일 이어서 볼 개념

**Adaptive History Weighting and Reactive Masks**

Neighborhood clipping으로 history color를 합리적인 범위 안에 넣었다면, 다음 단계는 그 history를 최종 pixel에 얼마나 반영할지 결정하는 것이다.

다음 개념에서는 motion magnitude, disocclusion, transparency, luminance change, material response, accumulated sample count를 이용해 history weight를 동적으로 조절하는 방법과, reactive mask가 temporal upscaling에서 ghosting과 detail recovery를 어떻게 제어하는지 살펴본다.

---

## 10. 참고 키워드

- Neighborhood Clamping
- Neighborhood Clipping
- Variance Clipping
- Temporal Anti-Aliasing (TAA)
- Temporal Supersampling
- Temporal Reprojection
- History Validation
- History Rejection
- Color AABB
- Convex Hull Approximation
- First Raw Moment
- Second Raw Moment
- Mean and Variance
- Gamma Parameter
- YCoCg Color Space
- HDR Pre-Exposure
- Reactive Mask
- Disocclusion
- Ghosting
- Flickering
- Temporal Stability
- Thin Geometry
- Compute Shader
- Group Shared Memory
- FP16 / FP32 Precision
- History Ping-Pong Buffer
- Temporal Upscaling
- Ray Tracing Denoising
- Field-aware Temporal Filtering

### 참고 자료

- Marco Salvi, *Temporal Supersampling*, GDC 2016  
  https://developer.download.nvidia.com/gameworks/events/GDC2016/msalvi_temporal_supersampling.pdf
- Lei Yang, Shiqiu Liu, Marco Salvi, *A Survey of Temporal Antialiasing Techniques*, Computer Graphics Forum, 2020  
  https://research.nvidia.com/publication/2020-05_survey-temporal-antialiasing-techniques
