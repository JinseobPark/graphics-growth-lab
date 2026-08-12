---
title: "[Daily Graphics Growth] 2026-08-12 - Conditional GRIS and Subpath Reuse: Joint UCW, Final Gather, and Correlation Control"
date: "2026-08-12"
category: Graphics
tags:
  - ReSTIR
  - GRIS
  - Conditional Resampling
  - Path Tracing
  - Subpath Reuse
  - Joint UCW
  - Final Gather
  - GPU
  - Compute Shader
  - C++
  - Memory Layout
level: intermediate
---

# [Daily Graphics Growth] 2026-08-12 - Conditional GRIS and Subpath Reuse: Joint UCW, Final Gather, and Correlation Control

## 1. 오늘의 개념

어제는 **Shift Mapping and Jacobian-Aware Path Reuse**를 통해 서로 다른 pixel/frame의 path를 현재 적분 영역으로 옮길 때 **domain change**, **support**, **Jacobian**을 어떻게 생각해야 하는지 보았다. 오늘은 그 다음 질문으로 넘어간다.

> 전체 path를 현재 pixel에 맞게 옮기지 않고, path의 일부인 **subpath / suffix**만 재사용할 수 있을까?

**Conditional GRIS (Conditional Generalized Resampled Importance Sampling)** 는 이 질문을 다루는 이론적 틀이다. 핵심은 path를 하나의 불가분한 sample로 보지 않고, 대략적으로 다음처럼 나누어 생각하는 것이다.

- 현재 pixel에서 새로 생성되는 **prefix / conditioning variables**
- 이전 pixel이나 frame에서 가져오는 **reusable suffix / conditioned variables**

개념적으로 path sample을

\[
X=(Z,Y)
\]

로 두면, `Z`는 현재 camera path의 앞부분이나 local shading state처럼 **조건(condition)** 을 만드는 변수이고, `Y`는 재사용하려는 suffix subpath이다. 공동 분포 역시 개념적으로

\[
p(Z,Y)=p(Z)\,p(Y\mid Z)
\]

처럼 볼 수 있다.

문제는 suffix `Y`가 원래 source prefix `Z_s` 아래에서 생성되었다는 점이다. source prefix를 버렸다고 해서 그 생성 과정의 확률적 의존성이 사라지는 것은 아니다. 따라서 source reservoir의 결과를 단순히 "좋은 light subpath 하나"라고 생각하고 현재 prefix `Z_c`에 붙이는 것은 충분하지 않다.

Conditional GRIS는 이 조건부 구조를 명시적으로 다루고, **conditional UCW (Unbiased Contribution Weight)** 와 **joint UCW**를 통해 `(현재 조건, 재사용 sample)`의 조합을 올바른 적분 estimator로 해석할 수 있게 한다.

NVIDIA의 2023년 *Conditional Resampled Importance Sampling and ReSTIR*은 이를 ReSTIR PT에 적용해 **path reuse를 한 bounce 이상 뒤로 미루고(deferred reuse)**, camera 쪽 prefix는 새로 유지하면서 뒤쪽 suffix만 재사용하는 **final gather** 형태를 제시했다. 이 구조는 전체 path를 반복해서 복제하는 것보다 sample correlation을 낮추는 데 도움이 된다.

중요한 구분은 다음과 같다.

| 관점 | Full-path reuse / shift | Conditional subpath reuse |
|---|---|---|
| 재사용 단위 | path 전체 | suffix / subpath |
| 현재 pixel에서 새로 만드는 부분 | 작거나 없음 | prefix 또는 최소 한 segment 이상 |
| 핵심 수학 문제 | path-domain mapping, Jacobian, support | conditional domain, joint UCW, support |
| correlation | 공유 path가 길수록 커질 수 있음 | fresh prefix가 shared ancestry를 일부 끊음 |
| GPU 비용 | path mapping / replay 비용 | extra prefix/connection + suffix lookup 비용 |
| 장점 | 충분히 좋은 mapping이면 직접적인 reuse | 전체 path bijection이 어려운 상황에서도 부분 reuse 가능 |

즉, 오늘의 주제는 단순한 ReSTIR 변형이 아니라 **"재사용할 sample의 경계를 어디에 둘 것인가"**에 대한 문제다.

---

## 2. 한 줄 핵심

**Conditional GRIS는 전체 path를 억지로 동일한 domain으로 shift하지 않고, 현재 pixel에서 새로 생성한 prefix와 재사용한 suffix를 조건부 sample로 결합한 뒤 joint UCW로 그 결합 자체를 올바르게 weighting한다.**

---

## 3. 왜 중요한가

### 3.1 긴 path일수록 full-path correspondence는 깨지기 쉽다

ReSTIR 계열 알고리즘에서 spatial/temporal reuse는 매우 강력하지만, indirect path가 길어질수록 "neighbor pixel의 path가 현재 pixel에도 같은 의미를 가지는가?"라는 문제가 커진다.

특히 다음 상황에서는 full-path shift가 어렵다.

- 여러 번의 glossy/specular interaction
- visibility topology 변화
- disocclusion
- 서로 다른 path length
- 작은 geometric perturbation이 뒤쪽 bounce를 크게 바꾸는 경우
- source와 target이 서로 다른 transport support를 갖는 경우

어제 본 shift mapping은 이런 변화를 수학적으로 보정하기 위한 방법이다. 하지만 mapping 자체가 복잡하거나 valid support가 좁으면 전체 path를 유지하는 전략의 효율이 떨어질 수 있다.

Conditional reuse는 문제를 분리한다.

> "앞부분은 현재 pixel의 실제 transport를 새로 따른다. 뒤쪽에서 이미 비싼 탐색을 거쳐 얻은 subpath만 빌려온다."

이렇게 하면 full path 전체에 대해 exact correspondence를 요구하는 대신, **새 prefix와 borrowed suffix가 만나는 conditional interface**가 핵심 correctness 지점이 된다.

### 3.2 source reservoir의 weight는 보편적인 값이 아니다

Reservoir에 저장된 normalized weight를 "이 sample의 품질"처럼 생각하면 위험하다.

그 weight는 항상 어떤 source 상태 아래에서 만들어졌다.

- source pixel의 geometry
- source BSDF
- source visibility
- source target function
- candidate proposal family
- 이전 resampling history
- sample multiplicity 또는 reservoir provenance

현재 prefix가 바뀌면 contribution도 바뀐다. 예를 들어 동일 suffix라도 현재 첫 연결점의 normal과 roughness가 달라지면 BSDF factor가 크게 바뀔 수 있고, visibility가 막히면 contribution이 0이 될 수도 있다.

따라서 **source normalized reservoir weight를 그대로 새로운 prefix에 붙이는 것**은 일반적으로 올바른 확률 해석이 아니다.

Conditional GRIS의 중요한 메시지는 weight가 suffix 하나에만 속한 scalar가 아니라는 것이다. **conditioning variable과 reused sample의 관계**, 더 일반적으로는 joint sample의 생성 경로와 contribution까지 고려한 UCW가 필요하다.

### 3.3 unbiased와 low-correlation은 서로 다른 목표다

Estimator가 unbiased라고 해서 image가 안정적인 것은 아니다.

ReSTIR가 반복적으로 같은 reservoir ancestry를 공유하면 인접 pixel이 동일하거나 매우 비슷한 path suffix를 선택할 수 있다. 평균값은 맞더라도 화면에서는 다음과 같은 형태로 보일 수 있다.

- 넓은 blotchy patch
- 시간이 지나도 같이 움직이는 structured noise
- denoiser가 독립적인 noise로 해석하기 어려운 low-frequency error
- reservoir duplication에 따른 effective sample diversity 감소

Conditional ReSTIR의 **final gather** 관점은 reuse를 적어도 한 bounce 뒤로 미룸으로써 current pixel마다 fresh prefix를 갖게 하고, shared sample ancestry가 최종 pixel contribution에 그대로 복제되는 정도를 낮춘다.

즉,

\[
\text{more reused samples} \neq \text{more independent information}
\]

이다.

`M`이 커지는 것과 **effective independent sample count**가 커지는 것은 다른 문제다.

### 3.4 real-time path tracing에서 path depth budget을 다르게 쓸 수 있다

낮은 rays-per-pixel 환경에서 deep indirect transport는 매우 비싸다. 매 pixel이 긴 path를 새로 추적하면 spatial redundancy를 버리게 되고, path 전체를 재사용하면 correlation과 mapping 문제가 커진다.

Subpath reuse는 이 둘 사이의 중간 지점이다.

- local prefix: 현재 pixel의 geometry/material을 정확히 반영
- remote suffix: 이미 탐색된 indirect transport를 재사용
- conditional connection: 두 영역 사이의 estimator contract를 담당

이 구조는 **ray budget**, **reuse distance**, **correlation**, **bias/variance**를 하나의 설계 공간에서 볼 수 있게 한다.

---

## 4. 구현 관점

여기서의 구현 관점은 특정 논문의 코드를 그대로 재현하는 방법이 아니라, graphics engineer가 Conditional GRIS 계열 시스템을 읽거나 설계할 때 확인해야 할 **data contract와 GPU execution contract**를 중심으로 본다.

### 4.1 estimator contract: prefix와 suffix의 경계를 데이터에 남겨야 한다

전체 path를 단일 opaque sample로 저장하면 "이 suffix가 어떤 조건 아래 생성되었는가"를 추적하기 어렵다.

개념적으로 reservoir state는 다음 두 층으로 나눠 생각할 수 있다.

**Reservoir header**

- selected suffix reference
- accumulated contribution / UCW 관련 state
- reservoir multiplicity `M` 또는 이에 준하는 provenance
- history age
- validity / interface flags
- source generation 또는 frame identity

**Subpath payload**

- path vertex position
- incoming/outgoing direction
- throughput
- BSDF / phase-function interface에 필요한 state
- forward/reverse sampling information
- emissive endpoint 또는 light information
- path length / termination metadata

여기서 핵심은 메모리 구조보다 **확률적 provenance를 잃지 않는 것**이다.

Subpath가 source prefix에서 분리되어도 그 suffix를 생성한 conditional distribution과 UCW를 재평가할 수 있어야 한다. 즉, rendering data structure가 estimator의 수학적 가정을 파괴하면 안 된다.

### 4.2 joint UCW를 이해하는 실무적 관점

일반적인 Monte Carlo estimator는 sample `x`를 PDF `p(x)`로 생성했다면 `f(x)/p(x)` 형태를 떠올릴 수 있다. GRIS는 pointwise PDF를 직접 알기 어려운 resampled sample에도 사용할 수 있도록 **UCW**를 도입한다.

Conditional GRIS에서는 더 복잡하다.

현재 prefix를 `Z_c`, borrowed suffix를 `Y_s`라고 하면 최종 contribution은

\[
f(Z_c,Y_s)
\]

처럼 **두 변수의 결합**에 의해 결정된다.

이때 중요한 것은 `Y_s`가 source condition에서 나온 이력을 지니고 있다는 점이다. 그래서 weight 역시 "suffix 하나의 inverse PDF"처럼 독립적으로 취급하는 것이 아니라, **joint/conditional contribution weight** 관점에서 결합되어야 한다.

실무적으로 기억할 문장은 다음과 같다.

> **Reservoir weight is not a portable importance score.**

Reservoir의 weight는 다른 shading domain에 무조건 복사할 수 있는 quality score가 아니라, 특정 target/proposal/resampling history에 속한 estimator state다.

### 4.3 final gather를 rendering pipeline 관점으로 보기

Conditional ReSTIR의 final-gather 스타일 구조는 논리적으로 다음 단계로 분해된다.

1. **Current prefix state**: 현재 pixel의 camera-side geometry와 local transport state
2. **Reusable suffix reservoir**: 이전/주변 sample에서 선택된 indirect subpath
3. **Conditional connection evaluation**: 현재 prefix와 suffix가 결합될 때의 contribution과 support 평가
4. **Visibility / transport validation**: 실제 연결 segment가 성립하는지 확인
5. **Reservoir resolve**: joint estimator state를 최종 radiance로 해석

여기서 3번과 4번의 순서는 성능에 큰 영향을 준다. 저렴한 compatibility/target 검사를 통과하지 못할 candidate에 비싼 visibility ray를 쓰면 ray budget이 낭비된다. 반대로 visibility를 너무 공격적으로 reuse하면 estimator의 가정이나 temporal stability가 깨질 수 있다.

따라서 ray tracing pipeline에서는 **cheap rejection → expensive validation**이라는 일반적인 cost hierarchy가 중요하다.

### 4.4 GPU memory layout: reservoir와 variable-length subpath를 분리하는 이유

Subpath는 길이가 가변적이다. 이를 reservoir record 안에 직접 포함하면 record stride가 커지고 memory bandwidth가 크게 늘어난다.

GPU 관점에서는 다음과 같은 분리가 자연스럽다.

| 영역 | 성격 | 주요 요구 |
|---|---|---|
| Reservoir header | 작고 고정 크기 | coalesced access, temporal reuse |
| Subpath index / handle | 간접 참조 | stable lifetime, generation check |
| Path vertex pool | 큰 가변 데이터 | bandwidth, locality |
| Connection metadata | 짧은-lived transient | register/shared/transient buffer |
| Visibility result | compact output | producer-consumer synchronization |

이 구조에서 새로운 문제가 생긴다. **indirection**이다.

Reservoir header는 연속적으로 읽히지만 서로 다른 thread가 가리키는 suffix가 path pool 전체에 흩어져 있으면 memory access가 random해진다. 특히 path length가 다르면 warp/wave 내부의 control-flow divergence도 커진다.

따라서 conditional path reuse의 성능 병목은 ray tracing cost만이 아니다.

- scattered suffix fetch
- cache miss
- large vertex payload
- register pressure
- variable loop trip count
- visibility-ray divergence

도 함께 본다.

### 4.5 SoA, AoS, AoSoA의 선택은 "한 vertex를 얼마나 완전히 읽는가"에 따라 달라진다

Path data는 전형적인 GPU layout trade-off를 갖는다.

**AoS (Array of Structures)**

한 vertex 평가에 position, direction, throughput, PDF 정보를 대부분 함께 읽는다면 locality가 좋을 수 있다. 반면 일부 pass가 position이나 flag만 읽을 때 불필요한 bandwidth가 생긴다.

**SoA (Structure of Arrays)**

candidate validation처럼 position/normal/roughness 일부만 대량으로 읽는 pass에 유리하다. 하지만 하나의 suffix를 깊게 traverse할 때 여러 buffer fetch가 필요하다.

**AoSoA (Array of Structures of Arrays)**

wave/subgroup 단위 처리와 vectorized load 사이의 절충점이 될 수 있다.

Conditional ReSTIR에서는 모든 pass가 전체 vertex state를 필요로 하지 않기 때문에 **hot metadata와 cold payload를 분리하는 layout**이 특히 중요하다.

### 4.6 precision: throughput보다 weight/probability state가 더 민감할 수 있다

실시간 renderer는 bandwidth를 줄이기 위해 일부 path state를 FP16으로 압축하기 쉽다. 하지만 모든 값이 동일하게 안전한 것은 아니다.

상대적으로 압축 후보가 되기 쉬운 것:

- 일부 normalized direction encoding
- bounded material parameter
- low-dynamic-range auxiliary state

더 주의가 필요한 것:

- accumulated reservoir weight
- UCW 관련 값
- 매우 작은 path probability
- HDR throughput product
- 여러 bounce를 거친 multiplicative state

특히 긴 path에서는 probability와 throughput이 여러 번 곱해지며 dynamic range가 커진다. 이러한 값의 precision loss는 단순한 shading error가 아니라 **resampling selection probability와 estimator normalization**에 영향을 줄 수 있다.

즉, temporal accumulator와 마찬가지로 reservoir state에서도 FP16 여부는 "보기에 비슷한가"가 아니라 **확률적 contract가 유지되는가**로 판단해야 한다.

### 4.7 compute shader / ray tracing shader의 register pressure

Conditional candidate를 평가하는 shader는 동시에 많은 state를 들고 있을 가능성이 높다.

- current prefix position / normal / BSDF frame
- suffix endpoint state
- throughput
- conditional contribution
- UCW / reservoir state
- RNG
- visibility query state

이 상태가 늘어나면 register pressure가 커지고 occupancy가 떨어질 수 있다. 따라서 이 계열 알고리즘은 "ray 수를 줄였으니 무조건 빠르다"고 볼 수 없다.

GPU profile에서는 최소한 다음 비용 축을 따로 보아야 한다.

\[
T_{frame}
=
T_{trace}
+T_{suffix\ fetch}
+T_{resample}
+T_{visibility}
+T_{memory/sync}
\]

ray count 하나만으로는 전체 비용을 설명하지 못한다.

### 4.8 subgroup / wave 관점

같은 wave가 서로 다른 path depth와 서로 다른 material lobe를 처리하면 divergence가 커진다. 반면 같은 유형의 conditional connection을 묶어 평가할 수 있다면 subgroup 연산은 다음 종류의 작업에 잘 맞는다.

- candidate validity ballot
- active lane compaction
- weight reduction
- shared random-selection 단계
- duplicate suffix detection

다만 subgroup optimization은 estimator semantics보다 아래 계층이다. lane을 합치거나 candidate를 제거하는 최적화가 **candidate multiplicity와 selection probability**를 바꾸면 결과적으로 UCW 계산도 달라질 수 있다.

그래서 GPU optimization에서도 "수학적으로 동일한 candidate set을 유지하는가"가 먼저다.

### 4.9 C++ / Render Graph에서 가장 위험한 문제: persistent suffix lifetime

Temporal reuse를 하면 reservoir가 이전 frame의 suffix를 계속 가리킬 수 있다. 이때 path pool을 transient resource처럼 너무 빨리 recycle하면 매우 위험하다.

예를 들어 reservoir가 `suffixIndex = 1042`를 들고 있는데 다음 frame에서 pool slot 1042가 다른 path로 재사용되었다면, index 자체는 유효해 보여도 estimator 관점에서는 완전히 다른 sample을 읽는다.

이 문제는 일반적인 dangling pointer와 비슷하지만 GPU에서는 더 찾기 어렵다.

- crash가 나지 않을 수 있음
- 값도 finite일 수 있음
- artifact가 stochastic noise처럼 보일 수 있음
- temporal history 때문에 몇 frame 뒤에 나타날 수 있음

따라서 persistent path pool은 C++ frame graph에서 다음과 같은 개념과 연결된다.

- frame ownership
- generation ID
- history version
- pool epoch
- deferred recycle
- fence / queue completion
- resize invalidation

특히 resolution change, camera cut, scene topology update, material table rebuild처럼 history를 무효화하는 이벤트는 reservoir header뿐 아니라 **referenced suffix pool의 lifetime**까지 함께 끊어야 한다.

이 지점은 graphics engineer에게 중요한 통찰이다.

> **Memory lifetime bug가 곧 estimator correctness bug가 될 수 있다.**

---

## 5. 내 관심 분야와 연결

Conditional GRIS는 graphics 분야에서 보기 드물게 **Monte Carlo estimator 이론과 GPU systems engineering이 거의 같은 비중으로 만나는 주제**다.

### C++ / explicit GPU API

Vulkan·DirectX 12 계열의 explicit resource model로 보면 중요한 것은 단순 shader dispatch가 아니다.

- persistent reservoir resource ownership
- suffix pool lifetime
- cross-frame synchronization
- descriptor stability
- frame-in-flight 간 resource version
- resize / history reset contract

즉, 수학적으로 올바른 reuse 알고리즘도 C++ resource layer에서 stale handle을 허용하면 즉시 잘못된 estimator가 된다.

### GPU compute

Conditional reuse는 전형적인 **irregular GPU workload**다.

- variable-length path
- pointer/index indirection
- branch-heavy material evaluation
- visibility ray
- large dynamic range weight

따라서 FLOPS보다 **memory access pattern, wave coherence, occupancy, ray divergence**가 더 중요한 경우가 많다.

### rendering pipeline

기존 deferred/forward rendering에서는 G-buffer나 shading input이 frame-local한 경우가 많다. 반면 ReSTIR/GRIS 계열에서는 reservoir가 단순 중간 버퍼가 아니라 **확률적 history state**다.

Temporal resource를 다루는 관점이 TAA history보다 더 엄격해진다.

TAA history가 stale하면 ghosting이 생기지만, reservoir/subpath provenance가 stale하면 **sampling distribution 자체가 잘못될 수 있다.**

### denoising과의 연결

Final gather가 흥미로운 이유는 raw variance만 줄이려는 것이 아니라 **spatial correlation 구조**를 바꾸기 때문이다.

Denoiser는 보통 noise가 지나치게 큰 low-frequency structured pattern으로 뭉치면 제거하기 어렵다. fresh prefix를 통해 neighboring pixel이 동일한 suffix ancestry를 그대로 공유하는 정도를 낮추면, 같은 variance라도 denoiser가 처리하기 더 좋은 noise structure가 될 수 있다.

따라서 modern real-time path tracing에서는

\[
\text{sampling} \rightarrow \text{correlation structure} \rightarrow \text{denoiser behavior}
\]

까지 하나의 pipeline으로 보는 시각이 필요하다.

---

## 6. 머릿속에 남길 질문 3개

1. **conditioning boundary를 camera 쪽으로 한 bounce, 두 bounce, 세 bounce 뒤로 옮길수록 reuse 가능한 suffix의 범위와 sample correlation, ray cost는 각각 어떻게 변할까?**

2. **새로운 prefix에 source reservoir의 normalized weight를 그대로 붙이면 왜 target/proposal mismatch가 생기며, joint UCW는 어떤 확률적 provenance를 보존하려는 것일까?**

3. **두 reservoir가 동일한 `M`을 갖더라도 실제 unique suffix 수와 shared ancestry가 다르면 effective sample diversity와 temporal stability는 왜 달라질까?**

---

## 7. graphics engineer 면접 질문 1개와 답변

### Q. 왜 subpath reservoir를 새로운 camera-prefix에 연결할 때 source reservoir의 normalized weight를 그대로 사용할 수 없는가?

**A.** Reservoir weight는 sample 자체의 절대적인 "importance score"가 아니라, source domain의 target function, proposal/resampling 과정, conditioning state에 의해 정의된 estimator state이기 때문이다.

새 prefix에 suffix를 연결하면 다음 요소가 달라질 수 있다.

- connection geometry term
- BSDF / phase-function value
- visibility
- current target contribution
- conditional sampling support
- source와 current domain 사이의 proposal 관계

따라서 source에서 유효했던 normalized weight가 현재 joint sample `(new prefix, reused suffix)`에도 그대로 유효하다고 볼 수 없다.

Conditional GRIS는 이를 **conditional UCW / joint UCW** 관점으로 확장한다. 핵심은 suffix만의 weight를 운반하는 것이 아니라, current conditioning variable과 reused suffix가 하나의 joint estimator를 이룰 때 필요한 확률적 정보를 보존하는 것이다.

Full-path shift mapping이 항상 필요한 것은 아니지만, 그 대신 **conditional support와 joint weighting의 correctness**가 새로운 핵심 조건이 된다.

좋은 면접 답변은 여기서 한 단계 더 나아가 다음을 연결한다.

> "이 수학적 provenance가 GPU data structure의 provenance와 일치해야 한다. reservoir가 가리키는 suffix pool entry가 stale하면, memory safety는 유지되어도 estimator는 깨진다."

이 설명은 rendering algorithm과 C++/GPU systems 양쪽을 함께 이해하고 있음을 보여준다.

---

## 8. 포트폴리오 / 커리어 연결

Conditional GRIS는 포트폴리오에서 단순히 "ReSTIR를 구현했다"보다 훨씬 강한 설명 구조를 만들 수 있는 주제다.

특히 다음 네 층을 연결해서 설명할 수 있다는 점이 중요하다.

### 1) Estimator layer

- full-path reuse와 subpath reuse의 차이
- conditional domain
- UCW와 joint UCW
- support와 unbiasedness

### 2) Sampling / quality layer

- fresh prefix가 correlation을 낮추는 이유
- `M`과 independent information의 차이
- final gather와 blotchy artifact의 관계
- raw variance와 spatial covariance의 차이

### 3) GPU architecture layer

- fixed-size reservoir header
- variable-size subpath pool
- SoA / AoS / AoSoA trade-off
- indirection과 cache locality
- register pressure / occupancy
- wave divergence

### 4) C++ engine layer

- persistent resource lifetime
- generation/version 관리
- temporal history invalidation
- render graph dependency
- GPU fence와 deferred recycle

이 네 층을 한 장의 architecture diagram으로 설명할 수 있으면, 단순 shader 작성자가 아니라 **probability estimator → GPU memory → render graph → temporal image quality**를 연결하는 graphics engineer 관점을 보여준다.

면접에서도 "왜 이 알고리즘이 좋나?"보다 다음처럼 trade-off를 이야기할 수 있는 것이 더 강하다.

> Full-path reuse는 reuse efficiency가 높지만 mapping validity와 correlation 문제가 커질 수 있다. Conditional subpath reuse는 local prefix를 새로 생성해 correlation을 줄이고 full-path correspondence 의존도를 낮추는 대신, extra connection cost와 conditional UCW/provenance 관리가 필요하다. GPU에서는 그 provenance가 persistent subpath pool lifetime과 직접 연결된다.

이 수준의 설명은 연구 논문을 단순 요약하는 것을 넘어 **engine integration 관점으로 번역할 수 있음**을 보여준다.

---

## 9. 내일 이어서 볼 개념

**Persistent Subpath Pools and Reservoir Indirection: GPU Lifetime, Compaction, and Memory Coherence**

오늘은 "subpath를 조건부로 재사용해도 되는가"라는 estimator 문제를 보았다. 다음에는 그 subpath를 실제 GPU renderer에서 여러 frame에 걸쳐 어떻게 표현하고 유지하는지가 중심이다.

이어질 핵심 연결점은 다음과 같다.

- variable-length path를 fixed-size reservoir에서 분리하는 이유
- persistent path pool과 index indirection
- path compaction / dead-path reclamation
- frame-in-flight와 suffix lifetime
- generation ID와 stale handle 방지
- SoA / AoSoA layout
- wavefront path tracing queue와 path pool의 관계
- memory bandwidth와 cache locality
- reservoir가 참조하는 sample의 temporal ownership

즉, 내일은 오늘의 **probability provenance**를 GPU의 **memory provenance**로 옮긴다.

---

## 10. 참고 키워드

- Conditional GRIS
- Conditional RIS / CRIS
- Conditional ReSTIR
- Generalized Resampled Importance Sampling
- Unbiased Contribution Weight (UCW)
- Conditional UCW
- Joint UCW
- Conditional probability space
- Randomized conditional domain
- Subpath reuse
- Suffix reuse
- Path prefix / path suffix
- Deferred path reuse
- Final gather
- ReSTIR PT
- Path-space support
- Conditional shift mapping
- Reservoir provenance
- Effective sample diversity
- Sample correlation
- Shared ancestry
- Blotchy artifacts
- Spatial covariance
- Temporal covariance
- Denoiser-friendly noise
- Reservoir multiplicity `M`
- Persistent reservoir
- Path vertex pool
- Subpath index / handle
- Generation ID
- Resource lifetime
- Deferred recycle
- SoA / AoS / AoSoA
- GPU memory indirection
- Cache locality
- Register pressure
- Occupancy
- Wave / subgroup divergence
- Visibility validation
- Ray budget
- Kettunen et al., *Conditional Resampled Importance Sampling and ReSTIR* (SIGGRAPH Asia 2023)
- Lin et al., *Generalized Resampled Importance Sampling: Foundations of ReSTIR* (2022)
