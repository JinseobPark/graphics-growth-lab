---
title: "GPU LOD Error Metrics: Projected Geometric Error, Semantic Importance, and Stable Refinement Thresholds"
date: "2026-09-22"
category: Graphics
tags: [GPU, Rendering, GPU-Driven Rendering, LOD, Screen-Space Error, Geometric Error, Semantic Error, Mesh Simplification, Cluster LOD, Meshlet, Refinement, Hysteresis, Vulkan, CUDA, Memory Layout, C++]
level: intermediate
---

# [Daily Graphics Growth] 2026-09-22 - GPU LOD Error Metrics: Projected Geometric Error, Semantic Importance, and Stable Refinement Thresholds

## 1. 오늘의 개념

어제는 **Hierarchical Geometry Streaming DAGs**에서 continuous LOD를 단순한 “LOD0/LOD1/LOD2 선택”이 아니라, geometry hierarchy/DAG에서 **valid cut**을 선택하는 문제로 보았다.

오늘은 그 cut을 움직이게 만드는 핵심 값인 **LOD error metric**을 다룬다.

Runtime이 refinement 여부를 결정하려면 결국 다음 질문에 답해야 한다.

```text
이 coarse representation이
현재 camera/view에서
얼마나 틀려 보이는가?
```

가장 기본적인 답은 **geometric error**다.

예:

```text
original surface
↕ e
simplified surface
```

여기서 `e`가 object/world space 거리라면, camera와의 거리 및 projection을 사용해 **screen-space error**로 변환할 수 있다.

하지만 engineering/scientific visualization에서는 단순 거리 오차만으로 충분하지 않다.

예를 들어 반도체 구조를 생각하면:

```text
0.5 nm의 gate oxide thickness 변화
```

는 화면상 작은 위치 오차일 수 있지만 의미는 매우 크다.

반면 넓은 substrate의 평평한 surface가 수 nm 이동하는 것은 시각적으로 거의 중요하지 않을 수 있다.

그래서 오늘은 LOD error를 세 층으로 본다.

1. **Geometric Error**
   - 실제 surface 위치가 얼마나 달라지는가.

2. **Projected Error**
   - 그 차이가 현재 view에서 몇 pixel 수준으로 보이는가.

3. **Semantic Importance**
   - material boundary, thin layer, selected region, measurement target처럼 domain 의미상 얼마나 중요한가.

그리고 마지막으로:

4. **Stable Thresholding**
   - error가 threshold 근처에서 흔들릴 때 refine/coarsen이 매 frame 반복되지 않도록 어떻게 hysteresis를 둘 것인가.

핵심은 다음과 같다.

> **LOD metric은 “mesh가 얼마나 틀렸는가”만 측정하는 숫자가 아니라, 현재 view·domain semantics·streaming cost를 하나의 refinement decision으로 연결하는 policy interface다.**

---

## 2. 한 줄 핵심

> 좋은 GPU LOD error metric은 **object-space simplification error를 screen-space로 투영하되, material/thin-layer/selection 같은 semantic importance를 별도 weight로 반영하고, refine/coarsen에 서로 다른 threshold를 사용해 temporal oscillation을 억제하는 것**이다.

---

## 3. 왜 중요한가

LOD system이 잘못되면 두 종류의 비용이 생긴다.

### Too Fine

필요 이상으로 fine geometry를 유지한다.

```text
cluster count ↑
meshlet count ↑
streaming bytes ↑
memory residency ↑
visibility work ↑
raster/ray tracing cost ↑
```

### Too Coarse

필요한 detail을 잃는다.

```text
silhouette error
material boundary drift
thin structure collapse
measurement mismatch
LOD popping
```

따라서 LOD error metric은 quality와 GPU cost 사이의 **control signal**이다.

### 3.1 Triangle Size와 Geometric Error는 다른 개념이다

“Triangle이 1 pixel보다 작으면 더 줄여도 된다”는 접근은 직관적이지만 항상 맞지 않는다.

큰 triangle 하나라도 원래 surface를 정확히 근사한다면 문제없을 수 있다.

반대로 작은 triangle이 많더라도 simplification이 sharp feature를 크게 움직였으면 error가 클 수 있다.

그래서 modern cluster LOD에서는 보통:

```text
triangle count / triangle size
```

보다:

```text
approximation error
```

를 중심으로 LOD를 결정하는 편이 더 일반적이다.

현재 NVIDIA `vk_lod_clusters` 문서도 “pixel-sized triangles”와 “sub-pixel-sized geometric error”를 별개의 목표로 소개하고, 후자에서 quadric-based object-space error를 camera에 대한 angular/screen-space error로 해석한다.

### 3.2 Relative Error와 Absolute Error의 차이

`meshoptimizer`의 simplifier는 기본적으로 error를 mesh extents에 대해 normalize된 relative 값으로 다룰 수 있고, `meshopt_SimplifyErrorAbsolute`를 사용하면 mesh coordinate space의 absolute error로 다룬다.

두 방식은 용도가 다르다.

```text
Relative error
→ object 크기에 자동 비례
→ 일반 asset LOD에 편리

Absolute error
→ nm / µm / m 같은 실제 단위 유지
→ engineering/scientific visualization에 유리
```

반도체 visualization에서는 실제 geometry scale이 의미를 갖기 때문에 absolute error가 특히 중요할 수 있다.

### 3.3 단순 geometric error만으로는 semantic 중요도를 놓칠 수 있다

예:

```text
wide substrate surface: 5 nm error
thin oxide interface: 1 nm error
```

화면상 projected error가 비슷해도 후자가 훨씬 중요한 경우가 있다.

즉 LOD policy는:

```text
geometry fidelity
+
semantic fidelity
```

를 함께 봐야 한다.

---

## 4. 구현 관점

### 4.1 Object-Space Geometric Error

Simplification 결과에 대한 기본 error:

```text
e_obj
```

를 생각한다.

의미:

```text
원래 surface를 coarse representation이
최대 어느 정도 벗어날 수 있는가
```

정확한 정의는 simplifier에 따라 다르다.

예:

- Quadric Error Metric(QEM)
- Hausdorff-like bound
- conservative vertex displacement bound
- application-specific distance bound

Runtime LOD에서는 **error가 보수적으로 upper bound 역할을 하는가**가 중요하다.

### 4.2 QEM은 정확한 Hausdorff Distance가 아니다

Quadric Error Metric은 edge collapse/simplification을 평가하는 매우 실용적인 metric이지만:

```text
QEM value
≠ exact maximum surface distance
```

다.

따라서 runtime에서 screen-space threshold에 직접 연결하려면:

- simplifier가 제공하는 scale
- cumulative error
- conservative bound

를 명확히 이해해야 한다.

현재 NVIDIA cluster LOD builder는 group의 cumulative quadric error와 cumulative bounding sphere를 사용해 conservative LOD selection이 가능하도록 값을 “massaging”한다.

### 4.3 Error의 Monotonicity

Hierarchy에서 coarse 방향으로 갈수록 error가 줄어들면 traversal이 불안정해진다.

원하는 invariant:

```text
error(parent/coarse)
>=
error(child/fine)
```

이다.

이렇게 하면:

```text
if coarse error acceptable:
    subtree 전체를 refine하지 않아도 됨
```

이라는 pruning이 가능하다.

Monotonic error는 GPU traversal의 중요한 property다.

### 4.4 Cumulative Error

한 번의 simplification만 보면 안 된다.

```text
LOD0 → LOD1 → LOD2 → LOD3
```

에서 LOD3의 error는 LOD2 대비 오차만이 아니라 원본 surface 대비 누적 오차를 의미해야 한다.

따라서 hierarchy metadata에는 conceptually:

```text
localSimplificationError
cumulativeErrorToOriginal
```

를 구분할 수 있다.

Runtime selection에는 보통 cumulative error가 더 유용하다.

### 4.5 Bounding Volume과 Error

Cluster/group이 bounding sphere를 가진다고 하자.

```text
center = c
radius = r
error = e
camera = p
```

가장 가까운 가능한 surface distance를 보수적으로:

```text
d_near = max(epsilon, |p-c| - r)
```

처럼 볼 수 있다.

그리고 angular error approximation:

```text
theta ≈ e / d_near
```

작은 angle에서는:

```text
tan(theta) ≈ theta
```

가 성립한다.

### 4.6 Screen-Space Error

Perspective projection에서 대략:

```text
pixelsPerWorldUnit
≈
viewportHeight
/
(2 * tan(FOV_y / 2) * distance)
```

따라서:

```text
screenErrorPixels
≈
e_world
*
viewportHeight
/
(2 * tan(FOV_y / 2) * d_near)
```

로 생각할 수 있다.

즉:

```text
same geometric error
+ closer camera
→ larger pixel error
```

다.

### 4.7 Projection-Aware Threshold

LOD policy:

```text
if screenErrorPixels > threshold:
    refine
```

예:

```text
threshold = 0.5 px
1.0 px
2.0 px
```

하지만 threshold 자체는 renderer의:

- antialiasing
- upscaling
- display resolution
- content type
- motion

과 관련된다.

즉 “1 pixel”은 universal constant가 아니다.

### 4.8 Dynamic Resolution / Upscaling

Renderer가 internal resolution:

```text
1920×1080
```

에서 render하고 DLSS/FSR로:

```text
3840×2160
```

을 출력한다면 LOD error는 어느 pixel grid 기준인가?

선택:

```text
internal render resolution
output resolution
perceptual effective resolution
```

이 있다.

Geometry cost는 internal rendering과 더 직접적이지만 visual error는 final output과 관계가 있다.

보통 engine은 **LOD reference resolution**을 별도로 정의하는 편이 reasoning하기 쉽다.

### 4.9 FOV 변화

Zoom camera에서 FOV가 줄면 같은 world-space error가 더 크게 보인다.

따라서 distance만으로 LOD를 고르면 zoom에서 detail이 부족해질 수 있다.

```text
error / distance
```

가 유용한 이유는 angular error를 근사하기 때문이다.

### 4.10 Orthographic Projection

Orthographic camera에서는 distance가 screen scale에 영향을 주지 않는다.

Perspective formula를 그대로 쓰면 잘못된다.

Orthographic:

```text
pixelsPerWorldUnit
=
viewportHeight / orthoHeight
```

따라서:

```text
screenError
=
e_world * pixelsPerWorldUnit
```

이다.

CAD/scientific viewer에서는 orthographic view가 흔하기 때문에 중요하다.

### 4.11 Projection Type도 Error Metric State다

따라서 runtime function은 단순:

```text
error / distance
```

가 아니라 conceptually:

```text
projectError(
    objectError,
    bounds,
    view,
    projection,
    viewport
)
```

이어야 한다.

### 4.12 Bounding Sphere의 Conservative Nearest Distance

Cluster center 거리만 쓰면 큰 cluster에서 error를 underestimate할 수 있다.

예:

```text
center distance = 100
radius = 40
```

실제 nearest geometry는 camera에서 60만큼 떨어져 있을 수 있다.

따라서 conservative screen error는:

```text
distance to nearest possible point
```

를 사용한다.

현재 NVIDIA `vk_lod_clusters` 문서도 group bounding sphere의 camera-nearest point를 이용해 largest possible angular error를 평가한다고 설명한다.

### 4.13 Off-Screen / Reflection / Ray Tracing

Rasterization frustum 밖이라도:

- shadow
- reflection
- ray tracing
- mirror

에서는 geometry가 중요할 수 있다.

따라서 pure screen-space metric이 primary camera 하나만 본다면 reflection quality가 떨어질 수 있다.

가능한 policy:

```text
main-view error
reflection-view error
shadow importance
```

를 합치거나 minimum distance / maximum projected error를 사용한다.

현재 NVIDIA sample도 ray tracing에서 camera-frustum culling만으로 충분하지 않기 때문에 raster와 다른 heuristic을 사용한다.

### 4.14 Semantic Importance Weight

기본:

```text
effectiveError
=
screenGeometricError
```

에 weight를 추가한다.

```text
effectiveError
=
screenGeometricError
× semanticWeight
```

Semantic weight 후보:

```text
material boundary
thin layer
selected object
measurement region
high curvature
silhouette importance
process-critical feature
```

### 4.15 Material Boundary Penalty

Simplification이 서로 다른 material/interface 경계를 움직이면 geometric distance가 작아도 semantic error가 크다.

예:

```text
Si / SiO2 interface
PN boundary
metal / dielectric boundary
```

이런 vertex/edge는:

- simplification lock
- higher priority
- larger semantic weight

를 줄 수 있다.

### 4.16 meshoptimizer Attribute-Aware Simplification

현재 `meshoptimizer`는 `meshopt_simplifyWithAttributes`로 position뿐 아니라 attribute error를 함께 고려할 수 있고, 중요 vertex에 priority/lock을 둘 수 있다.

이 아이디어를 domain에 확장하면:

```text
position
+
normal
+
material label
+
scalar boundary
```

를 simplification policy에 반영할 수 있다.

단 material ID 같은 categorical attribute는 일반 continuous numeric weight로 직접 다루기보다 **boundary constraint**로 보는 편이 명확할 수 있다.

### 4.17 Thin-Layer Penalty

반도체 구조에서 layer 두께가 2 nm인데 geometric error bound가 3 nm이면 화면상 pixel error가 작더라도 layer 자체가 사라질 수 있다.

따라서 다음 ratio가 중요할 수 있다.

```text
relativeFeatureError
=
geometricError / localFeatureThickness
```

즉:

```text
LOD acceptable
if
error < k × localFeatureThickness
```

처럼 screen-space criterion과 별도의 semantic geometry constraint를 둘 수 있다.

### 4.18 Dual Constraint

Engineering viewer에서는:

```text
screenError <= pixelThreshold
AND
featureRelativeError <= featureThreshold
```

처럼 두 조건을 모두 만족해야 coarse representation을 허용하는 것이 안전하다.

이 방식은 semantic weight 하나에 모든 것을 억지로 합치는 것보다 review하기 쉽다.

### 4.19 Selected Region Importance

사용자가 특정 gate/trench를 선택하면 해당 region은 화면상 작더라도 detail을 유지해야 할 수 있다.

```text
selectedRegion
→ lower refinement threshold
```

또는:

```text
effectiveError *= selectionWeight
```

로 표현할 수 있다.

### 4.20 Semantic Importance와 Residency Priority 공유

LOD system의 `semanticWeight`를 streaming system에서도 재사용할 수 있다.

```text
LOD priority
→ refinement

Residency priority
→ keep resident
```

따라서 하나의 domain importance signal이:

- simplification
- LOD selection
- streaming
- eviction

에 일관되게 사용될 수 있다.

### 4.21 Refine Threshold와 Coarsen Threshold를 분리한다

단일 threshold:

```text
if error > 1.0:
    refine
else:
    coarse
```

를 쓰면 error가:

```text
0.99
1.01
0.98
1.02
```

처럼 움직일 때 LOD가 매 frame 바뀔 수 있다.

그래서:

```text
refine if error > T_high
coarsen if error < T_low
```

를 사용한다.

```text
T_high > T_low
```

이다.

### 4.22 Hysteresis Band

예:

```text
T_low  = 0.75 px
T_high = 1.25 px
```

현재 fine인 경우:

```text
error < 0.75
→ coarse 가능
```

현재 coarse인 경우:

```text
error > 1.25
→ refine
```

그 사이에서는 current state 유지.

이것이 temporal stability를 크게 높인다.

### 4.23 Hysteresis는 Streaming Cost를 줄인다

LOD oscillation은 visual popping뿐 아니라:

```text
load
evict
load
evict
```

를 유발한다.

따라서 hysteresis는 geometry quality 안정화뿐 아니라 **residency thrashing 방지**다.

### 4.24 Threshold는 LOD Level마다 다를 수 있다

Coarse root에서 fine level로 갈수록 transition cost가 다를 수 있다.

예:

```text
root → mid:
large visual gain, large bytes

mid → fine:
small gain, many bytes
```

따라서 level별 threshold 또는 cost-aware threshold를 둘 수 있다.

### 4.25 Cost-Aware Refinement

단순:

```text
screenError > T
```

보다:

```text
RefineScore
=
ErrorReduction × Importance
---------------------------
RefinementCost
```

를 사용할 수 있다.

Refinement cost:

- streamed bytes
- remesh compute
- neighbor dependency
- transition mesh
- acceleration-structure build

를 포함할 수 있다.

### 4.26 Quality-per-Byte와 LOD Error의 연결

어제까지 다룬 residency policy와 연결하면:

```text
QualityGain
≈
currentError - refinedError
```

따라서:

```text
QualityPerByte
=
(currentError - refinedError)
----------------------------
additionalResidentBytes
```

로 refinement priority를 만들 수 있다.

### 4.27 Error Reduction Curve

LOD hierarchy node가:

```text
currentError
childError
byteCost
```

를 갖고 있다면:

```text
deltaError = currentError - childError
```

를 계산할 수 있다.

Fine level로 갈수록 `deltaError/byte`가 급격히 작아질 수 있다.

이 값이 낮은 refinement는 memory pressure에서 뒤로 미룬다.

### 4.28 Semantic Quality Gain

Engineering viewer에서는:

```text
qualityGain
=
geometricErrorReduction
+
semanticErrorReduction
```

처럼 생각할 수 있다.

예:

- material boundary recovery
- thin layer restoration
- selected structure detail

이 큰 경우 fine LOD priority가 높아진다.

### 4.29 Scalar 하나로 모든 Error를 합치지 않아도 된다

시스템 review 관점에서는 다음처럼 분리하는 것이 더 안전할 수 있다.

```text
GeometricScreenError
FeatureRelativeError
SemanticConstraintFlags
ResidencyCost
```

그리고 decision:

```text
mustRefine =
    screenError > threshold
    OR featureError > threshold
    OR semanticCritical
```

그 뒤 candidates끼리 cost-based scheduling을 한다.

Constraint와 optimization score를 분리하는 것이다.

### 4.30 Hard Constraint와 Soft Score

#### Hard Constraint

위반하면 coarse LOD를 허용하지 않는다.

예:

- thin layer disappears
- material boundary invalid
- measurement region exactness

#### Soft Score

어떤 candidate를 먼저 refine할지 결정한다.

예:

- projected error
- quality-per-byte
- distance
- visibility probability

이 구분은 production system에서 매우 유용하다.

### 4.31 Error Metadata Memory Layout

Hierarchy node의 hot fields:

```text
bounds
cumulativeError
childRange
residencyState
```

Semantic metadata:

```text
featureMinThickness
materialBoundaryFlags
importanceClass
```

를 별도 buffer로 둘 수 있다.

일반 traversal은 geometric fields만 읽고, special region에서만 semantic metadata를 읽는다.

### 4.32 Quantized Error

수백만 hierarchy node에서 `float32 error`는 큰 bandwidth가 될 수 있다.

Error를:

```text
FP16
log-encoded 16-bit
fixed-point
```

로 quantize할 수 있다.

단 conservative bound를 깨면 안 된다.

예:

```text
decode(quantizedError)
>=
trueError
```

가 되도록 upward rounding할 수 있다.

### 4.33 Log Encoding이 유용한 이유

LOD error range가:

```text
1e-6 ~ 1e3
```

처럼 넓다면 linear fixed-point는 비효율적이다.

```text
log2(error)
```

를 quantize하면 relative precision을 일정하게 유지할 수 있다.

Distance/scale가 매우 넓은 CAD/scientific scene에서 유용할 수 있다.

### 4.34 Bounding Data Quantization도 Conservative해야 한다

Bounding sphere radius를 작게 quantize하면 nearest distance를 크게 추정해 error를 underestimate할 수 있다.

따라서:

```text
radius → round up
error  → round up
```

같은 conservative direction을 선택한다.

LOD traversal metadata는 **false refine**보다 **false coarse**가 더 위험한 경우가 많다.

### 4.35 Floating-Point Precision

World coordinate가 매우 크고 feature가 매우 작다면:

```text
large world position
+
nanometer-scale detail
```

에서 float precision 문제가 생길 수 있다.

대안:

- local object/brick coordinates
- camera-relative transform
- double precision on CPU hierarchy build
- normalized error scale

를 사용할 수 있다.

### 4.36 Camera-Relative Evaluation

Screen-space error는 camera와의 상대 거리만 중요하므로:

```text
objectCenter - cameraPosition
```

을 local/camera-relative 좌표에서 계산하면 precision이 좋아질 수 있다.

Large-world rendering에서도 일반적인 패턴이다.

### 4.37 Multi-View Error

Stereo/VR 또는 multiple viewport가 있다면:

```text
error = max(error_view0, error_view1, ...)
```

로 모든 active view를 만족시킬 수 있다.

Engineering tool의:

- main 3D view
- cross-section view
- minimap

이 동시에 같은 geometry를 공유한다면 residency/LOD priority를 aggregate해야 할 수 있다.

### 4.38 Shadow / Reflection View

Main camera에는 coarse여도 shadow edge나 reflection에서 중요할 수 있다.

완전 정확한 multi-view 계산이 비싸면 importance bias를 줄 수 있다.

```text
castsCriticalShadow
reflectionVisible
```

같은 flags를 semantic importance에 포함한다.

### 4.39 Temporal Prediction

Camera가 빠르게 접근하는 object는 현재 error가 threshold 아래여도 곧 refine가 필요하다.

```text
predictedError(t + Δ)
```

를 계산해 미리 request할 수 있다.

이는 prefetch note와 직접 연결된다.

### 4.40 Refinement Lead Time

Streaming latency가 `L`이라면:

```text
predicted screen error at t + L
```

를 사용해 refine request를 미리 생성할 수 있다.

즉 LOD metric은 current rendering뿐 아니라 **future residency demand predictor**가 된다.

### 4.41 Motion Hysteresis

Camera가 object 쪽으로 빠르게 접근하면 refine threshold를 약간 낮추고, 멀어지는 중이면 coarsen을 지연할 수 있다.

다만 motion-dependent threshold가 지나치게 강하면 quality가 불안정해질 수 있으므로 bounded bias가 좋다.

### 4.42 Screen-Space Velocity와 LOD

Object가 빠르게 움직일 때 high-frequency geometry detail이 perceptually 덜 중요할 수 있다는 관점도 있다.

하지만 scientific visualization에서는 정확도 우선일 수 있다.

즉 motion-based perceptual LOD는 domain policy다.

### 4.43 Error Debug Visualization

LOD system은 눈으로 debug하기 어려우므로 overlay가 중요하다.

예:

```text
color by screen-space error
color by semantic weight
color by selected LOD
color by hysteresis state
```

또:

- desired cut
- resident cut
- error threshold
- refinement debt

를 overlay한다.

### 4.44 Error Histogram

Frame마다:

```text
selected node screenError histogram
```

을 본다.

이상적인 경우 대부분 threshold 근처 또는 아래에 분포한다.

많은 node가 threshold보다 훨씬 작은 error인데 fine LOD를 유지하면 over-refinement 가능성이 있다.

### 4.45 Refinement Efficiency

Metric:

```text
ErrorReduction / AddedClusters
ErrorReduction / AddedBytes
ErrorReduction / GPUTime
```

를 측정하면 어떤 hierarchy level이 비용 대비 효율이 좋은지 알 수 있다.

### 4.46 Semantic Override Rate

얼마나 많은 region이 geometric criterion은 coarse 허용인데 semantic rule 때문에 refine되는지 측정한다.

```text
semanticOverrideCount
semanticOverrideBytes
```

이 값이 지나치게 크면 semantic constraint가 hierarchy simplification을 사실상 무력화하고 있을 수 있다.

### 4.47 Thin-Feature Survival Metric

반도체 domain에서 유용한 QA metric:

```text
min visible feature thickness
```

또는:

```text
critical boundary displacement
```

을 LOD별로 검사한다.

Generic triangle count보다 훨씬 의미 있다.

### 4.48 LOD Error와 Mesh Simplification을 따로 보지 않는다

Offline simplifier가 만들어내는 error metric과 runtime selector가 해석하는 error metric이 다르면 문제가 생긴다.

예:

```text
offline error = relative mesh extent
runtime assumes world meters
```

이면 threshold가 의미 없어질 수 있다.

따라서 asset/runtime ABI에:

```text
error unit
error scale
coordinate scale
```

를 명시해야 한다.

### 4.49 Error ABI

Conceptual metadata:

```text
LodErrorMetadata {
    errorValue
    errorEncoding
    errorUnit
    boundType
}
```

실제 implementation은 더 compact할 수 있지만, 의미는 명확해야 한다.

### 4.50 C++ Strong Types

다음 값은 모두 float여도 의미가 다르다.

```text
ObjectSpaceError
ScreenSpaceErrorPx
FeatureRelativeError
LodThresholdPx
SemanticWeight
```

Strong type/wrapper를 사용하면 unit mismatch를 줄일 수 있다.

특히 CAD/scientific software에서 단위 오류는 치명적이다.

### 4.51 GPU Shader 관점

Traversal compute shader는 최소한:

```text
bounds
error
camera/projection parameters
current state
```

를 읽는다.

Semantic constraint가 rare하다면:

```text
if (semanticFlags != 0)
    fetch extended metadata
```

로 cold path를 만들 수 있다.

### 4.52 Compute → Rendering Pipeline

```text
Hierarchy traversal compute
→ desired/resident cut
→ refinement requests
→ render cluster list
→ mesh/task shader
```

LOD error metric은 compute pass의 branch 하나가 아니라 **whole rendering pipeline의 workload generator**다.

Threshold를 조금 바꾸면:

- traversal output
- streaming requests
- GPU memory
- mesh shader work
- ray tracing AS build

가 모두 바뀐다.

### 4.53 Ray Tracing Cost

Ray tracing cluster LOD에서는 refine가 geometry bytes뿐 아니라 CLAS/BLAS update cost를 만들 수 있다.

따라서:

```text
RefinementCost
=
stream bytes
+
AS build/update cost
```

로 봐야 한다.

NVIDIA `vk_lod_clusters` 역시 streaming된 new cluster group에 대해 CLAS를 build하는 경로를 가진다.

### 4.54 Stable Thresholds는 Budget Controller와 연결된다

Memory pressure가 높아지면 global threshold를 높여 coarse LOD를 허용할 수 있다.

```text
normal: T = 1.0 px
pressure: T = 1.5 px
```

하지만 frame마다 budget에 따라 threshold가 크게 흔들리면 전체 scene이 pumping한다.

따라서 threshold 자체에도:

- smoothing
- rate limit
- hysteresis

를 적용한다.

### 4.55 Global Threshold + Local Importance

좋은 구조:

```text
global quality target
× local semantic importance
```

예:

```text
effectiveThreshold
=
globalThreshold / importance
```

importance가 높을수록 threshold가 작아져 더 fine detail을 유지한다.

### 4.56 Global Threshold는 “Quality Knob”가 된다

사용자 설정:

```text
Performance
Balanced
Quality
```

를 실제로는:

```text
screen-space error budget
```

으로 mapping할 수 있다.

이렇게 하면 random LOD distance table보다 resolution/FOV에 더 잘 적응한다.

### 4.57 프로파일링에서 볼 지표

핵심 metrics:

- selected clusters / triangles
- desired vs resident cut size
- screen-space error p50/p95/p99
- over-threshold visible area
- geometric error histogram
- semantic override count
- semantic override bytes
- thin-feature violation count
- material-boundary displacement
- refine events/frame
- coarsen events/frame
- refine↔coarsen oscillation count
- average LOD lifetime
- hysteresis-band occupancy
- refinement bytes/frame
- error reduction / byte
- error reduction / GPU ms
- hierarchy node visits
- early-exit ratio
- error metadata bandwidth
- predicted-vs-actual screen error
- threshold changes over time
- memory-pressure quality degradation

중요한 dashboard:

```text
Visual/Semantic Error
vs
Resident Bytes
vs
GPU Time
```

세 축을 함께 본다.

---

## 5. 내 관심 분야와 연결

### Semiconductor Process Visualization

사용자의 domain에서는 generic geometric LOD보다 **semantic-aware LOD**가 특히 중요하다.

예:

```text
substrate flat bulk
→ aggressive simplification 가능

gate oxide
→ thin-feature protection

material interface
→ boundary penalty

selected trench
→ importance boost

cross-section near plane
→ low error threshold
```

따라서 다음처럼 두 개의 constraint를 병행하는 것이 좋다.

```text
Screen-Space Error
+
Feature/Material Constraint
```

단순 pixel error 하나보다 engineering fidelity가 높다.

### Dynamic SDF / Level Set

SDF로부터 생성된 mesh라면 simplification error 외에:

```text
surface displacement from φ=0
```

를 직접적인 geometric 의미로 해석할 수 있다.

또 thin region에서는 local SDF gradient / feature size를 이용해 semantic refinement signal을 만들 수 있다.

### ColumnStack / Thin Layers

ColumnStack처럼 Z 방향 layer thickness가 중요한 representation에서는 isotropic error보다:

```text
XY silhouette error
Z thickness error
```

를 분리할 수 있다.

특히 1~2 cell 두께 layer가 coarse LOD에서 사라지지 않도록 **minimum layer survival constraint**를 둘 수 있다.

### CFD / Scientific Visualization

CFD iso-surface에서는 geometric error 외에도:

- gradient magnitude
- shock/front proximity
- scalar threshold crossing
- selected streamline region

을 semantic importance에 반영할 수 있다.

### Vulkan / CUDA

CUDA/Warp가 geometry를 생성할 때:

```text
local geometric error
feature metadata
```

를 함께 만들고, Vulkan compute traversal은 이를 이용해 screen-space error와 semantic policy를 결합할 수 있다.

### Game Engine / Graphics Career

이 주제는:

- Nanite-like virtual geometry
- cluster LOD
- mesh simplification
- screen-space error
- streaming refinement
- mesh shader
- ray tracing geometry LOD

와 직접 연결된다.

면접에서 강한 설명은:

> **“distance-based LOD보다 screen-space error가 왜 낫나요?”**

에 대해 projection/FOV/resolution까지 설명하고, 더 나아가 **domain semantic constraint와 hysteresis**까지 연결하는 것이다.

---

## 6. 머릿속에 남길 질문 3개

1. **Perspective screen-space error를 계산할 때 cluster center distance 대신 bounding sphere의 nearest distance를 사용하는 것이 왜 conservative하며, 어떤 경우 over-refinement를 만들 수 있을까?**
2. **반도체 thin layer처럼 화면상 geometric error는 작지만 의미가 큰 feature를 단순 semantic weight 하나로 처리하는 것과 hard feature constraint로 분리하는 것의 장단점은 무엇일까?**
3. **Refine/coarsen hysteresis를 크게 하면 streaming thrash는 줄지만 stale LOD가 오래 유지될 수 있는데, error distribution·LOD lifetime·camera velocity 중 어떤 telemetry를 이용해 band를 조절하는 것이 좋을까?**

---

## 7. graphics engineer 면접 질문 1개와 답변

### 질문

**“LOD를 camera distance만으로 선택하는 것보다 screen-space geometric error를 사용하는 것이 왜 더 좋은가요?”**

### 답변

Distance만 사용하면 object 크기, simplification quality, projection/FOV, viewport resolution을 직접 반영하지 못한다.

두 object가 같은 거리에 있어도:

```text
Object A: coarse mesh error = 1 mm
Object B: coarse mesh error = 10 cm
```

라면 필요한 LOD가 다르다.

Screen-space error는 object-space approximation error를 camera/projection을 통해 pixel domain으로 옮긴다.

Perspective에서 개념적으로:

```text
screenErrorPixels
≈
worldError
× viewportHeight
/
(2 × tan(FOV_y/2) × distance)
```

이므로 camera가 가까워지거나 FOV가 좁아지면 더 fine LOD가 선택된다.

다만 production renderer에서는 여기서 끝나지 않는다.

첫째, cluster가 크면 center distance보다 bounding sphere의 nearest distance를 사용해야 error를 underestimate하지 않는다.

둘째, hierarchy error는 원본 surface 대비 cumulative하고 monotonic해야 traversal pruning이 안전하다.

셋째, scientific/engineering viewer에서는 material boundary나 thin layer처럼 geometric pixel error만으로 표현되지 않는 semantic constraint가 있다.

넷째, threshold 하나만 쓰면 camera movement에 따라 refine/coarsen이 반복될 수 있으므로:

```text
refine if error > T_high
coarsen if error < T_low
```

의 hysteresis가 필요하다.

핵심은:

> **Screen-space error는 view-dependent quality를 정량화하는 기본 signal이고, 실제 production LOD는 여기에 hierarchy invariants, semantic constraints, residency cost, temporal stability를 결합한 decision system이다.**

---

## 8. 포트폴리오 / 커리어 연결

이 주제는 graphics engineer가 단순 LOD distance table이 아니라 **error-bounded GPU-driven geometry system**을 이해한다는 것을 보여주기 좋다.

### Geometry Processing

- Quadric Error Metric
- cumulative simplification error
- absolute vs relative error
- material/attribute-aware simplification
- feature preservation

### Rendering

- screen-space error
- perspective/orthographic projection
- multi-view error
- stable refine/coarsen

### GPU Runtime

- hierarchy traversal
- desired/resident cut
- refinement request
- cost-aware scheduling
- ray tracing AS rebuild cost

### Scientific Visualization

- material boundary penalty
- thin-layer survival
- selected-region importance
- measurement-safe LOD
- domain semantic fidelity

### Memory / Streaming

- quality-per-byte
- refinement debt
- residency pressure
- hysteresis against thrashing

### C++ / Data Layout

- error units
- strong types
- SoA metadata
- conservative quantization
- error ABI

포트폴리오에서는 다음처럼 설명할 수 있다.

> **“Continuous LOD hierarchy의 simplification error를 cumulative object-space bound로 저장하고, runtime에서 bounding-sphere nearest distance와 projection parameters를 사용해 screen-space pixel error로 변환했습니다. Refine/coarsen에는 별도 threshold를 사용해 temporal oscillation을 억제했고, semiconductor thin layer와 material interface는 generic pixel error와 분리된 hard semantic constraint로 보호했습니다. Streaming pressure에서는 error reduction per resident byte를 이용해 refinement priority를 조절했습니다.”**

이 설명은 geometry processing, renderer, memory streaming, domain visualization을 하나로 연결한다.

---

## 9. 내일 이어서 볼 개념

**GPU LOD Traversal Architectures: Top-Down Cuts, Persistent Queues, and Wave-Coherent Hierarchy Evaluation**

오늘은 어떤 node를 refine해야 하는지 결정하는 **error metric**을 봤다.

다음 질문은:

> **수백만 LOD node에서 이 error test를 GPU가 어떻게 빠르게 수행해 current-frame cut을 만드는가?**

학습 흐름:

```text
Hierarchical Streaming DAG
→ Stable Error Metric
→ GPU Hierarchy Traversal
→ Wave-Coherent Cut Generation
→ Render Worklist
```

다음 노트에서는:

- top-down traversal
- stack vs queue
- persistent traversal workers
- subtree pruning
- warp/wave-coherent node evaluation
- child compaction
- traversal divergence
- hierarchy SoA layout
- render list generation
- desired cut와 resident cut 동시 계산

을 중심으로 이어간다.

---

## 10. 참고 키워드

- LOD Error Metric
- Geometric Error
- Screen-Space Error
- Projected Error
- Angular Error
- Quadric Error Metric (QEM)
- Cumulative Error
- Monotonic Error
- Bounding Sphere
- Conservative Error Bound
- Relative Error
- Absolute Error
- `meshopt_SimplifyErrorAbsolute`
- `meshopt_simplifyScale`
- Attribute-Aware Simplification
- `meshopt_simplifyWithAttributes`
- Border Lock
- Feature Preservation
- Semantic Importance
- Material Boundary Penalty
- Thin-Layer Constraint
- Feature Relative Error
- Selection Importance
- Hard Constraint / Soft Score
- Refine Hysteresis
- Coarsen Hysteresis
- Temporal LOD Stability
- Error Reduction per Byte
- Quality-per-Byte
- Continuous LOD
- Cluster LOD
- Meshlet DAG
- GPU Hierarchy Traversal
- RTX Mega Geometry
- Dynamic SDF
- Scientific Visualization
- NVIDIA nvpro-samples — **vk_lod_clusters**
  - https://github.com/nvpro-samples/vk_lod_clusters
- NVIDIA — **vk_lod_clusters LOD Generation Documentation**
  - https://github.com/nvpro-samples/vk_lod_clusters/blob/main/docs/lod_generation.md
- NVIDIA nvpro-samples — **nv_cluster_lod_builder**
  - https://github.com/nvpro-samples/nv_cluster_lod_builder
- meshoptimizer — **Simplification Documentation**
  - https://github.com/zeux/meshoptimizer
- meshoptimizer — **meshoptimizer.h Simplification Flags**
  - https://github.com/zeux/meshoptimizer/blob/master/src/meshoptimizer.h
- Brian Karis et al. — **A Deep Dive into Nanite Virtualized Geometry**, 2021
  - https://advances.realtimerendering.com/s2021/
