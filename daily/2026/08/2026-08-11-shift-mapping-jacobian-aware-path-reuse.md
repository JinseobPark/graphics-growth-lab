---
title: "Shift Mapping and Jacobian-Aware Path Reuse: Reconnection, Domain Change, and Path-Space Support"
date: "2026-08-11"
category: Graphics
tags: ["GPU", "ReSTIR", "GRIS", "Shift Mapping", "Jacobian", "Path Space", "Reconnection", "Ray Tracing", "Monte Carlo", "Compute Shader", "Memory Layout", "Real-Time Rendering"]
level: intermediate
---

# [Daily Graphics Growth] 2026-08-11 - Shift Mapping and Jacobian-Aware Path Reuse: Reconnection, Domain Change, and Path-Space Support

## 1. 오늘의 개념

어제는 **Reservoir Compatibility and Bias Correction**을 통해 “어떤 neighbor reservoir를 가져올 것인가?”를 보았다. 오늘은 그 다음 질문인 **“가져온 path를 현재 pixel의 적분 domain으로 어떻게 옮길 것인가?”**를 다룬다.

ReSTIR PT/GRIS 계열에서 다른 pixel이나 frame의 path를 재사용하려면, source domain의 sample을 current domain의 sample로 바꾸는 **shift mapping**이 필요하다.

이를 단순화하면 source path `x`와 current-domain path `y` 사이에

`y = T(x)`

라는 mapping을 둔다. 여기서 `T`는 임의의 변환이 아니라, 적어도 mapping이 정의되는 영역에서는 역변환을 생각할 수 있는 **partial bijection**으로 보는 것이 중요하다. 즉 모든 source path가 반드시 current domain으로 옮겨질 수 있는 것은 아니다.

Source domain을 `Ω_s`, current domain을 `Ω_c`라고 하면,

`T : Ω_s' -> Ω_c'`

처럼 실제로 reuse 가능한 부분집합 `Ω_s'`에서만 mapping이 정의될 수 있다.

이때 path를 위치만 바꾸는 것으로 끝내면 안 된다. Monte Carlo estimator는 **측도(measure)** 위의 적분이므로 domain을 바꾸면 sample density와 contribution도 함께 변한다. 일반적인 change-of-variables 관점에서 Jacobian determinant가 등장하는 이유다.

`y = T(x)`일 때 개념적으로

`p_y(y) = p_x(x) / |det J_T(x)|`

이고, source contribution을 current domain에서 비교하려면

`f_c(T(x)) |det J_T(x)|`

처럼 Jacobian-corrected contribution을 생각해야 한다.

좋은 shift mapping은 단순히 “valid path를 만든다”를 넘어서

`f_c(T(x)) |det J_T(x)| ≈ f_s(x)`

가 되도록, 즉 원래 high-contribution path가 옮겨진 뒤에도 비슷한 중요도를 유지하도록 설계된다.

이 관점에서 오늘의 핵심 요소는 네 가지다.

1. **Path-space support**: source path가 current domain에서도 실제로 존재 가능한가?
2. **Reconnection**: path 일부를 current shading point 또는 다른 vertex에 연결해 재사용할 수 있는가?
3. **Jacobian correction**: domain mapping이 sample measure를 얼마나 늘리거나 줄였는가?
4. **Shift validity**: glossy/specular chain, visibility, geometry singularity 때문에 mapping이 깨지는가?

ReSTIR의 핵심 난이도는 reservoir 자료구조 자체보다, 결국 **“서로 다른 domain의 sample을 통계적으로 올바른 sample로 바꾸는 법”**에 있다.

---

## 2. 한 줄 핵심

**Shift mapping은 이웃 path를 현재 pixel로 복사하는 기능이 아니라, path-space domain을 바꾸면서 support·visibility·Jacobian을 함께 보존해야 하는 Monte Carlo 변수변환(change of variables)이다.**

---

## 3. 왜 중요한가

### 3.1 같은 path처럼 보여도 적분 domain은 다르다

화면에서 인접한 두 pixel `A`, `B`가 같은 벽을 보고 있다고 하자. 두 pixel의 primary hit는 가깝지만 camera ray, surface position, BSDF frame, visibility, secondary bounce geometry는 모두 조금씩 다르다.

따라서 `A`의 path를 `B`에서 재사용하는 것은

`A의 sample을 B가 그대로 사용`

하는 문제가 아니라

`A-domain의 sample을 B-domain의 동등한 random variable로 변환`

하는 문제다.

Direct lighting에서는 동일한 light sample을 현재 surface에서 다시 평가하는 것만으로 비교적 단순하게 shift할 수 있다. 그러나 multi-bounce GI에서는 path vertex 전체가 연쇄적으로 연결되어 있기 때문에 한 vertex를 움직이면 뒤쪽 vertex까지 영향을 받는다.

이 차이가 **ReSTIR DI와 ReSTIR PT의 구현 복잡도 차이**를 만든다.

### 3.2 Jacobian은 “면적이 얼마나 변했는가”를 estimator에 알려준다

Shift mapping의 Jacobian을 기하학적 미분값 정도로만 기억하면 실무에서 의미를 놓치기 쉽다.

Monte Carlo 관점에서 Jacobian은 더 직접적으로

**“이 mapping이 sample-space의 작은 영역을 얼마나 압축 또는 팽창시켰는가?”**

를 나타낸다.

Source에서 작은 영역 `dμ_s`가 current domain에서 `dμ_c`로 변했다면

`dμ_c = |det J_T| dμ_s`

로 볼 수 있다.

이 보정 없이 path contribution만 가져오면, 변환 과정에서 sample density가 달라졌음에도 동일한 확률로 생성된 sample처럼 취급하게 된다. 결과적으로 brightness bias나 특정 path class의 과대/과소 대표가 생길 수 있다.

또한 Jacobian의 값은 어떤 measure를 쓰는지에 의존한다. Path tracing에서는 solid-angle measure, area measure, path-space measure가 변환 과정에서 섞일 수 있으므로 **PDF가 어떤 measure 기준인지**가 API contract만큼 중요하다.

Graphics engineer 입장에서 기억할 것은 다음이다.

`PDF 값만 저장하면 충분하지 않다.`

`그 PDF가 무엇에 대한 density인지까지 알아야 한다.`

### 3.3 Reconnection shift: path의 앞부분만 바꾸고 뒷부분을 재사용한다

**Reconnection shift**는 source path 전체를 다시 생성하지 않고, path 일부를 current path의 새 vertex와 연결해 나머지 subpath를 재사용하는 전략이다.

개념적으로 source path가

`x0 -> x1 -> x2 -> x3 -> light`

이고 current pixel의 primary hit가 `y1`이라면,

`y0 -> y1 -> x2 -> x3 -> light`

처럼 `y1`에서 source path의 이후 vertex로 연결할 수 있다.

이 전략이 매력적인 이유는 source reservoir가 이미 찾아낸 **valuable suffix**를 유지할 수 있기 때문이다. 특히 어려운 간접광 경로를 처음부터 다시 추적하지 않아도 된다.

하지만 reconnection은 항상 안전하지 않다.

- `y1 -> x2`가 occluded일 수 있다.
- 두 vertex가 지나치게 가까워 geometry term이 수치적으로 불안정할 수 있다.
- source의 glossy/specular event가 current geometry에서 거의 불가능할 수 있다.
- BSDF lobe 방향이 크게 달라 contribution이 거의 0이 될 수 있다.
- delta/specular event는 연속적인 area-domain mapping 자체가 성립하지 않을 수 있다.

즉 reconnection의 성공 여부는 단순한 line-of-sight test가 아니라 **path support test**다.

### 3.4 Random replay와 hybrid shift

Reconnection이 불가능한 경우 path를 처음부터 완전히 버리는 것만이 선택지는 아니다.

**Random replay**는 source path를 생성했던 random sequence를 current shading configuration에서 다시 재생해 새로운 path를 만든다는 관점이다. 동일한 random number가 다른 geometry/BSDF에서 사용되므로 결과 path는 달라지지만, sampling procedure의 대응 관계는 유지된다.

장점은 specular/glossy chain처럼 reconnection이 어려운 구간을 따라갈 수 있다는 점이다. 반면 여러 bounce를 다시 trace해야 하므로 ray cost가 크고, geometry가 조금만 달라져도 path가 완전히 다른 위치로 갈 수 있다.

그래서 ReSTIR PT 계열에서는 **hybrid shift**가 중요하다.

개념적으로는

`hard-to-reconnect segment -> random replay`

`connectable segment 발견 -> reconnection`

처럼 두 방식을 섞는다.

2026년 **ReSTIR PT Enhanced**는 이 연결 판단을 더 robust하게 만들기 위해 **footprint-based reconnection criteria**를 도입해 shift mapping의 실패와 불안정성을 줄이는 방향을 제시했다.

### 3.5 Path-space support가 맞지 않으면 weight 문제 이전에 sample 자체가 무효다

어제는 Pairwise MIS와 neighbor compatibility를 다뤘다. 하지만 아무리 좋은 MIS weight를 사용해도 mapping의 support가 맞지 않으면 해결되지 않는다.

Source path `x`가 존재하지만 current domain에서 `T(x)`가 정의되지 않는다면,

`T(x) = undefined`

이다.

또는 mapping은 형식상 만들어졌지만 current target function이 zero-support라면

`f_c(T(x)) = 0`

가 된다.

이 상황에서 무리하게 weight ratio를 계산하면 `0/0`, 매우 큰 ratio, NaN/Inf 같은 numerical failure가 생기기 쉽다.

따라서 production pipeline에서 중요한 순서는 대체로

`support validity`
`-> mapping validity`
`-> visibility/connectability`
`-> target evaluation`
`-> Jacobian / UCW evaluation`
`-> reservoir resampling`

이다.

**Estimator의 안정성은 weight clamp보다 invalid domain을 일찍 제거하는 데서 시작한다.**

### 3.6 Specular chain은 왜 특별히 어렵나

Diffuse bounce는 outgoing direction 변화에 대해 비교적 넓은 support를 가진다. 그래서 primary hit가 약간 이동해도 source suffix와 reconnect했을 때 non-zero contribution이 남을 가능성이 높다.

반면 roughness가 매우 낮은 glossy 또는 perfect specular event는 유효한 direction set이 매우 좁다.

이때 current point에서 source suffix로 강제로 reconnect하면

- BSDF value가 사실상 0에 가까워지거나
- delta constraint를 위반하거나
- Jacobian/geometry ratio가 극단적으로 커질 수 있다.

결국 **“screen-space로 가까운 path”와 “path-space에서 가까운 path”는 다르다.**

이 차이가 ReSTIR neighbor selection, hybrid shift, footprint-aware reconnection이 따로 필요한 이유다.

### 3.7 최근 연구가 shift mapping을 계속 확장하는 이유

Shift mapping은 ReSTIR PT 하나의 구현 세부사항이 아니라, reuse 가능한 domain을 확장하는 핵심 abstraction으로 발전하고 있다.

- **Area ReSTIR (2024)**: pixel center만이 아니라 film/lens의 4D ray space까지 domain을 확장해 antialiasing과 depth of field에서 sample reuse를 유지한다.
- **ReSTIR BDPT (2025)**: sampling-technique-aware extended path space와 bidirectional hybrid shift를 사용해 caustic과 difficult light path reuse를 확장한다.
- **ReSTIR PT Enhanced (2026)**: footprint-based reconnection criteria로 shift mapping을 더 robust하게 만든다.
- **LoD-aware ReSTIR (2026)**: mesh topology가 달라지는 LoD 전환에서도 surface point mapping을 통해 temporal reuse를 유지한다.

공통점은 하나다.

**“Reuse할 수 있는 sample의 정의를 넓히려면, 먼저 domain mapping을 정확히 정의해야 한다.”**

---

## 4. 구현 관점

### 4.1 Reservoir가 path reuse를 위해 보관해야 하는 정보

ReSTIR PT에서 reservoir는 단순히 RGB radiance와 weight만 보관하는 구조가 아니다.

Shift를 다시 구성하려면 구현에 따라 다음과 같은 state가 필요할 수 있다.

- selected path/sample identity
- path generation seed 또는 replay 가능한 random state
- reconnection vertex 정보
- path length / bounce classification
- material 또는 lobe classification
- sample count `M`
- unbiased contribution weight(UCW) 또는 normalization state
- temporal age
- shift-validity / visibility metadata

즉 reservoir는 **statistical state + minimal path reconstruction state**의 결합이다.

Path 전체 vertex를 저장하면 재사용은 쉬워지지만 메모리 비용이 커지고, seed만 저장하면 memory는 줄지만 replay ray cost가 증가한다.

이 trade-off는 전형적인 GPU architecture 문제다.

`more state -> more bandwidth / VRAM`

`less state -> more recomputation / ray tracing`

### 4.2 Shift pass의 논리적 단계

Shader 관점에서 shift/resampling pass는 다음 state machine으로 보는 것이 이해하기 쉽다.

`Load source reservoir`
`-> validate source domain`
`-> choose shift mode`
`-> construct shifted path`
`-> test connectability / visibility`
`-> evaluate current target`
`-> evaluate Jacobian / contribution weight`
`-> update current reservoir`

여기서 중요한 것은 실패가 정상적인 결과라는 점이다.

Shift mapping은 모든 candidate에 대해 성공하는 함수가 아니라 **partial mapping**이다. 따라서 `invalid shift`를 exception처럼 취급하기보다 estimator가 기대하는 정상적인 rejection state로 설계해야 한다.

### 4.3 GPU divergence와 shift mode

Hybrid shift는 알고리즘적으로 좋지만 GPU에는 불편하다.

같은 warp/wave 안에서

- 어떤 lane은 immediate reconnection
- 어떤 lane은 2 bounce random replay
- 어떤 lane은 specular chain 때문에 5 bounce replay
- 어떤 lane은 early reject

가 되면 ray count와 shader execution length가 크게 달라진다.

따라서 ReSTIR PT의 병목은 단순한 arithmetic throughput이 아니라

- ray traversal divergence
- variable path length
- incoherent memory access
- reservoir random read
- reconnection branch divergence

가 된다.

이 때문에 path reuse 알고리즘의 품질 평가에서는 단순 `candidate count`보다

`useful shifted candidates per traced ray`

같은 효율 개념이 더 중요하다.

### 4.4 Jacobian 계산의 numerical contract

Jacobian과 target ratio는 high dynamic range를 가진다. 특히 작은 거리의 geometry term, grazing angle, narrow glossy lobe가 섞이면 값이 크게 변할 수 있다.

Production 관점에서 구분해야 할 것은 두 가지다.

**1) 수학적으로 invalid한 값**

- mapping undefined
- zero-support
- invalid PDF measure
- NaN / Inf
- degenerate geometry

이 경우 candidate를 invalid로 처리하는 것이 estimator 의미와 맞다.

**2) 수학적으로 valid하지만 매우 큰 contribution**

이 경우 단순 hard clamp는 firefly를 줄일 수 있지만 bias를 만든다. 며칠 전 다룬 firefly suppression과 동일하게, numerical guard와 statistical bias control은 다른 문제다.

즉

`finite check != firefly clamp`

이다.

### 4.5 Memory layout

Reservoir path state를 GPU buffer에 저장할 때 AoS와 SoA 선택도 중요하다.

**AoS (Array of Structures)**

한 candidate를 처리할 때 필요한 state가 연속되어 있어 단일 reservoir fetch에는 유리하다. 하지만 shift validation 단계에서 일부 field만 대량으로 읽는 경우 bandwidth 낭비가 생길 수 있다.

**SoA (Structure of Arrays)**

`age`, `flags`, `materialClass`, `reconnectionVertex` 등을 따로 두면 metadata-first rejection에 유리하다. Invalid candidate를 early reject한 뒤 expensive path state를 늦게 읽을 수 있다.

실무에서는 완전한 AoS/SoA보다

- compact header: ID, age, flags, M, shift class
- payload: replay seed, reconnection data, weight state

처럼 **header/payload split**이 reasoning과 bandwidth 양쪽에서 유용하다.

핵심은 expensive ray work 이전에 작은 metadata로 reject 가능성을 최대한 높이는 것이다.

### 4.6 C++ render graph 관점

C++ engine에서는 shift mapping을 단순한 shader 내부 로직으로 숨기기보다 pass contract로 드러내는 편이 디버깅에 유리하다.

개념적인 resource 흐름은 다음처럼 볼 수 있다.

`PrevReservoirBuffer`
`CurrentGBuffer`
`Motion / SurfaceIdentity`
`PathReplayState`
`-> TemporalShiftPass`
`-> SpatialShiftPass`
`-> FinalReservoirBuffer`
`-> Shading / Denoiser`

여기서 각 reservoir가 어떤 domain에 속하는지가 중요하다.

- previous-frame domain
- current canonical domain
- shifted current domain
- finalized shading domain

이 semantic을 잃으면 “weight는 맞는데 brightness가 흔들리는” 유형의 버그가 생긴다.

따라서 graphics debugging에서는 buffer 값만 보는 것보다

**“이 reservoir의 sample은 지금 어느 domain의 random variable인가?”**

를 먼저 묻는 것이 좋다.

---

## 5. 내 관심 분야와 연결

### C++ / GPU Rendering

Shift mapping은 Monte Carlo 이론과 GPU system design이 가장 직접적으로 만나는 주제다.

수식 관점에서는

- change of variables
- PDF measure
- Jacobian
- unbiased contribution weight

를 이해해야 한다.

GPU 관점에서는

- reservoir packing
- path replay state
- ray divergence
- metadata-first rejection
- cache locality
- render graph lifetime

을 동시에 판단해야 한다.

이 조합은 senior graphics engineer가 단순 shader 구현자를 넘어 **estimator와 architecture를 함께 설계하는 능력**을 보여주는 영역이다.

### Real-Time Rendering / Game Engine

게임 엔진에서 full path tracing을 적용할 때 가장 큰 제약은 sample budget이다. ReSTIR은 sample을 새로 생성하는 비용보다 과거와 주변에서 얻은 good path를 재사용하는 데 투자한다.

하지만 reuse가 강해질수록 다음 문제가 커진다.

`reuse efficiency ↑`

동시에

`correlation / domain mismatch / stale visibility risk ↑`

따라서 real-time path tracer의 품질은 단순 sample count보다 **shiftable sample의 비율과 shift quality**에 크게 좌우된다.

### Simulation / Scientific Visualization

CFD·volume·semiconductor visualization에서도 동일한 사고방식이 유용하다.

예를 들어 서로 다른 grid resolution, LOD, timestep 사이에서 sample을 재사용하려면

- 좌표 mapping
- support validity
- interpolation domain
- conservation quantity
- mapping determinant

를 생각해야 한다.

ReSTIR의 Jacobian은 light transport estimator를 위한 것이므로 simulation Jacobian과 동일한 수식으로 그대로 가져갈 수는 없지만, **“domain을 바꾸면 값뿐 아니라 measure도 바뀐다”**는 사고방식은 volume resampling, conservative remapping, adaptive mesh visualization에서도 중요한 공통 기반이다.

특히 최근 LoD-aware ReSTIR이 서로 다른 topology 사이의 surface point mapping을 다룬다는 점은 mesh/voxel/LOD 기반 visualization과도 개념적으로 연결된다.

---

## 6. 머릿속에 남길 질문 3개

1. **왜 neighbor reservoir의 path contribution만 current pixel에서 다시 평가하는 것으로는 부족하고, shift mapping의 Jacobian까지 고려해야 하는가?**

2. **Diffuse path에서는 reconnection이 잘 동작하지만 low-roughness specular chain에서 실패하기 쉬운 이유를 path-space support 관점으로 설명할 수 있는가?**

3. **Path 전체를 저장하는 방식과 random seed + reconnection vertex만 저장하는 방식은 GPU memory bandwidth, ray cost, divergence 측면에서 어떤 trade-off를 만드는가?**

---

## 7. graphics engineer 면접 질문 1개와 답변

### Q. ReSTIR PT에서 shift mapping이 무엇이며, 왜 Jacobian correction이 필요한가?

**A.** Shift mapping은 source pixel/frame의 path sample을 current pixel의 path-space domain으로 변환하는 mapping이다. Reuse는 sample을 단순 복사하는 것이 아니라 서로 다른 integration domain 사이의 change of variables이므로, mapping이 sample-space measure를 압축하거나 팽창시키면 sample density도 변한다. Jacobian determinant는 이 measure 변화를 보정해 shifted sample의 contribution과 probability 의미를 current domain에 맞춘다.

좋은 shift는 mapping이 정의되는 영역에서 invertible한 partial mapping을 가지며, source의 high-contribution path가 shift 후에도 비슷한 Jacobian-corrected contribution을 갖도록 설계하는 것이 이상적이다. Diffuse 구간에서는 reconnection으로 path suffix를 재사용하기 쉽지만, glossy/specular chain에서는 support가 좁고 delta constraint나 geometry singularity가 발생할 수 있어 random replay와 reconnection을 섞는 hybrid shift가 사용된다.

Implementation 측면에서는 shift validity를 먼저 확인하고, visibility/connectability와 target function을 평가한 뒤 Jacobian/UCW를 reservoir resampling에 반영해야 한다. GPU에서는 variable replay length와 reconnection success 여부 때문에 ray divergence와 memory layout이 중요한 성능 요소가 된다.

---

## 8. 포트폴리오 / 커리어 연결

ReSTIR를 포트폴리오에서 설명할 때 “reservoir sampling을 구현했다”는 표현만으로는 깊이가 잘 드러나지 않는다.

더 강한 설명은 다음 질문에 답하는 구조다.

- Reservoir가 어떤 integration domain의 sample을 표현하는가?
- Temporal/spatial reuse에서 domain이 어떻게 달라지는가?
- Shift mapping은 어떤 candidate에서 invalid한가?
- Reconnection과 random replay를 언제 구분하는가?
- Jacobian과 UCW는 어떤 estimator 의미를 갖는가?
- Reservoir state를 어느 precision과 layout으로 저장하는가?
- Reuse quality와 ray divergence 사이의 성능 trade-off를 어떻게 분석하는가?

이 수준까지 설명할 수 있으면 포트폴리오는 단순한 ray tracing demo가 아니라 **Monte Carlo estimator + GPU architecture + real-time systems**를 함께 이해한다는 신호가 된다.

특히 NVIDIA/AAA engine/real-time graphics 팀 면접에서는 “논문 알고리즘을 알고 있다”보다

**“알고리즘의 statistical contract를 유지하면서 GPU에 어떻게 배치할 것인가”**

를 설명하는 능력이 훨씬 강한 차별점이 된다.

---

## 9. 내일 이어서 볼 개념

**Conditional GRIS and Subpath Reuse: Joint UCW, Final Gather, and Correlation Control**

오늘은 전체 path를 다른 domain으로 옮기는 shift mapping을 보았다. 다음에는 path 전체가 아니라 **일부 subpath만 reuse할 때 확률 공간을 어떻게 conditioning해야 하는지**로 이어간다.

특히 다음 연결이 핵심이다.

`full-path reuse`
`-> reused suffix + non-reused prefix`
`-> conditional domain`
`-> joint unbiased contribution weight`
`-> final gather`
`-> reduced path correlation`

이 주제는 ReSTIR PT에서 blotchy correlation을 줄이고 denoiser-friendly signal을 만드는 이론적 기반으로 이어진다.

---

## 10. 참고 키워드

### 핵심 용어

- Shift Mapping
- Path-Space Domain
- Partial Bijection
- Change of Variables
- Jacobian Determinant
- Jacobian-Corrected Contribution
- Path-Space Support
- Reconnection Shift
- Random Replay
- Hybrid Shift
- Unbiased Contribution Weight (UCW)
- Generalized Resampled Importance Sampling (GRIS)
- ReSTIR PT
- ReSTIR GI
- Reservoir Sampling
- Path Replay State
- Specular Chain
- Delta BSDF
- Geometry Term
- Area Measure / Solid-Angle Measure
- Ray Divergence
- Metadata-First Rejection
- Reservoir Packing
- Extended Path Space

### 이어서 읽을 연구

- [Generalized Resampled Importance Sampling: Foundations of ReSTIR](https://research.nvidia.com/labs/rtr/publication/lin2022generalized/)
- [ReSTIR PT Enhanced: Algorithmic Advances for Faster and More Robust ReSTIR Path Tracing](https://research.nvidia.com/labs/rtr/publication/lin2026restirptenhanced/)
- [ReSTIR BDPT: Bidirectional ReSTIR Path Tracing with Caustics](https://research.nvidia.com/labs/rtr/publication/hedstrom2025restir/)
- [Area ReSTIR: Resampling for Real-Time Defocus and Antialiasing](https://research.nvidia.com/labs/rtr/publication/zhang2024area/)
- [Real-Time Level-of-Detail Rendering with ReSTIR](https://research.nvidia.com/labs/rtr/publication/wang2026levelofdetail/)
- [Conditional Resampled Importance Sampling and ReSTIR](https://research.nvidia.com/labs/rtr/publication/kettunen2023conditional/)
