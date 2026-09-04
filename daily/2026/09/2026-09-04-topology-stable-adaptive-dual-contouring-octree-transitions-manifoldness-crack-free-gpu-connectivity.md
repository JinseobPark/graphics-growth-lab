---
title: "Topology-Stable Adaptive Dual Contouring: Octree Transitions, Manifoldness, and Crack-Free GPU Connectivity"
date: "2026-09-04"
category: Graphics
tags: [GPU, Rendering, SDF, Level Set, Dual Contouring, Adaptive Octree, Manifold Mesh, Crack-Free Meshing, QEF, CUDA, Vulkan, Compute Shader, Sparse Volume, Memory Layout, C++]
level: intermediate
---

# [Daily Graphics Growth] 2026-09-04 - Topology-Stable Adaptive Dual Contouring: Octree Transitions, Manifoldness, and Crack-Free GPU Connectivity

## 1. 오늘의 개념

최근 흐름은 `SDF hit refinement → gradient/normal → Hessian/principal curvature → curvature-aware adaptive surface extraction`으로 이어졌다. 어제는 local feature radius와 QEF conditioning을 이용해 **어디를 더 세밀하게 meshing할 것인가**를 봤다.

오늘은 그 다음 문제다.

> **인접한 octree cell의 해상도가 서로 다른데도, surface connectivity를 끊지 않고 2-manifold topology를 유지하려면 무엇을 별도의 contract로 관리해야 하는가?**

Adaptive Dual Contouring(Adaptive DC)의 핵심 장점은 coarse cell과 fine cell이 섞인 octree에서도 sharp feature를 비교적 잘 보존하면서 sparse polygonization을 만들 수 있다는 점이다. 그러나 여기에는 서로 다른 세 가지 correctness가 섞여 있기 쉽다.

1. **Crack-free / watertight connectivity**  
   LOD boundary에 구멍이나 T-junction seam이 생기지 않는가.

2. **Manifoldness**  
   mesh의 vertex/edge neighborhood가 국소적으로 disk 또는 half-disk처럼 해석 가능한가. Closed interior surface라면 보통 각 mesh edge가 정확히 두 face에 incident해야 한다.

3. **Intersection-free geometry**  
   connectivity는 올바르더라도 triangle들이 공간에서 서로 관통하지 않는가.

이 셋은 같은 문제가 아니다. Adaptive Dual Contouring 시스템을 production 수준으로 설계하려면 **QEF vertex placement와 topology/connectivity generation을 분리해서 생각해야 한다.**

## 2. 한 줄 핵심

> Adaptive Dual Contouring의 안정성은 **“cell마다 좋은 vertex를 찾는 문제”와 “서로 다른 LOD의 leaf들이 동일한 sign-changing primal edge를 일관되게 공유하는 문제”를 분리하고, crack-free·manifold·intersection-free를 각각 독립된 mesh invariant로 관리하는 것**에서 나온다.

## 3. 왜 중요한가

Uniform grid에서는 한 primal grid edge 주변에 네 cell이 규칙적으로 배치되고, sign-changing edge를 기준으로 그 cell들의 dual vertex를 연결하면 connectivity가 비교적 단순하다.

Adaptive octree에서는 상황이 달라진다.

- 한쪽은 coarse leaf, 반대쪽은 여러 fine leaf일 수 있다.
- 같은 physical boundary를 서로 다른 resolution의 cell들이 표현할 수 있다.
- coarse cell의 한 vertex가 여러 fine-level polygon에 참여할 수 있다.
- local cell classification만 보고 face를 만들면 coarse/fine seam에 hole, duplicate face, T-junction이 생길 수 있다.
- 한 cell 안에 서로 분리된 surface component가 여러 개 지나가는데 vertex 하나만 두면 non-manifold merge가 발생할 수 있다.

즉 adaptive meshing의 비용은 단순히 octree traversal이 아니라 **cross-level topology agreement**에 있다.

이 문제는 특히 sparse SDF/level-set, dynamic simulation, semiconductor process geometry처럼 geometry가 매 frame 또는 process step마다 바뀌는 경우 중요하다. Surface가 조금 이동했을 뿐인데 refinement boundary가 바뀌면 mesh connectivity가 대규모로 재구성될 수 있고, topology가 불안정하면 shading artifact를 넘어 picking, metrology anchor, object ID, downstream mesh processing까지 깨질 수 있다.

또한 crack-free는 최종 품질의 필요조건이지 충분조건이 아니다. 2002년 Dual Contouring은 adaptive octree에서 별도 crack patching 없이 contour를 연결하는 구조를 제시했지만, 이후 Manifold Dual Contouring은 adaptive simplification에서 생길 수 있는 non-manifold vertex/edge를 별도로 해결했다. 다시 말해 **“seam이 없다”와 “mesh topology가 올바르다”는 다른 보장**이다.

## 4. 구현 관점

### 4.1 Primal과 dual의 역할을 분리해서 본다

Dual Contouring에서 geometry의 기준은 두 공간에 걸쳐 있다.

- **Primal grid:** SDF sample과 sign-changing grid edge가 존재한다.
- **Dual mesh:** active cell 내부의 representative vertex와 polygon connectivity가 존재한다.

Uniform grid에서는 sign-changing primal edge 하나가 주변 dual cell vertex들을 묶는 자연스러운 connectivity primitive가 된다.

Adaptive octree에서도 핵심은 같다. 다만 “주변 cell”이 동일 level이라는 보장이 없다. 따라서 topology stage는 단순한 `cell -> face`가 아니라 **canonical primal transition -> incident leaf set -> dual polygon**이라는 관점이 더 정확하다.

### 4.2 QEF와 connectivity는 다른 correctness domain이다

QEF(Quadratic Error Function)는 각 active cell에서 representative vertex 위치를 결정한다.

`E(x) = Σ (n_i · (x - p_i))²`

QEF가 매우 잘 풀려 sharp corner 위치를 정확하게 찾았더라도 다음은 자동으로 해결되지 않는다.

- 그 vertex가 어떤 이웃 vertex와 연결되는가
- coarse/fine boundary에서 polygon ordering이 일관적인가
- 한 cell에 여러 surface component가 있을 때 vertex를 분리해야 하는가
- 만들어진 polygon이 manifold한가
- triangle이 서로 교차하지 않는가

따라서 production pipeline에서는 **Vertex Placement Pass**와 **Topology Generation Pass**를 서로 다른 단계와 validation domain으로 보는 편이 안전하다.

### 4.3 Crack-free와 manifold는 서로 다른 invariant다

Crack-free는 주로 **coverage와 adjacency agreement** 문제다.

- 동일한 physical boundary를 양쪽 LOD가 같은 topology event로 해석하는가
- coarse/fine transition에서 누락된 polygon이 없는가
- 같은 polygon이 중복 생성되지 않는가

반면 manifoldness는 **local neighborhood topology** 문제다.

Closed 2-manifold surface라면 대표적으로:

- interior mesh edge의 incident face 수가 2
- vertex 주변 incident faces가 하나의 fan으로 연결
- 서로 다른 surface sheet가 하나의 vertex로 잘못 merge되지 않음
- orientation/winding이 component 내부에서 일관됨

이 두 종류의 검사를 같은 `watertight = true` flag로 뭉개면 debugging이 어려워진다.

### 4.4 Adaptive DC는 transition “patch”보다 shared event ownership이 핵심이다

Marching Cubes 기반 LOD에서 잘 알려진 Transvoxel은 coarse/fine boundary에 **transition cell**을 넣어 seam을 연결한다. 이는 매우 중요한 비교 대상이지만 Adaptive Dual Contouring의 topology mechanism과 동일하지는 않다.

Dual Contouring에서는 original octree contouring formulation 자체가 서로 다른 level의 cell을 재귀적으로 함께 처리해 sign-changing primal edge 주변의 dual vertices를 연결한다. 즉 개념적으로는 seam이 생긴 뒤 patch를 덧붙이는 것보다 **처음부터 coarse/fine leaf들이 같은 connectivity event를 공유하게 하는 것**에 가깝다.

GPU에서는 이 recursive idea를 그대로 call stack으로 표현하기보다 flat work item으로 변환하는 경우가 많다.

예를 들면 topology event는 개념적으로 다음 정보를 가진다.

`EdgeKey = {axis, spatial_code, logical_level}`

`EdgeEvent -> incident leaf/component handles -> ordered polygon`

핵심은 정확한 struct 이름이 아니라 **하나의 physical sign transition에 하나의 canonical ownership**을 부여하는 것이다.

### 4.5 2:1 balanced octree는 필수 조건이라기보다 GPU complexity knob다

Classical Dual Contouring은 restricted octree만을 요구하지 않는 formulation을 가질 수 있다. 그러나 GPU production code에서는 neighboring leaf level 차이를 1 이하로 제한하는 **2:1 balancing**이 실용적인 선택이 될 수 있다.

장점:

- coarse/fine adjacency case 수가 제한된다.
- incident leaf fan-in의 upper bound가 작아진다.
- transition topology template 또는 flat kernel 분기가 단순해진다.
- halo width와 dirty-region propagation 범위를 예측하기 쉽다.

비용:

- geometry error상 필요하지 않은 cell도 refinement될 수 있다.
- leaf count와 field residency가 증가한다.
- simulation hierarchy와 mesh hierarchy의 refinement pressure가 충돌할 수 있다.

따라서 2:1 balancing은 “정답”이라기보다 **메모리를 더 쓰고 topology scheduling을 단순화하는 시스템 trade-off**로 보는 편이 좋다.

### 4.6 한 cell에 vertex 하나라는 가정이 topology를 깨뜨릴 수 있다

가장 중요한 manifold 문제 중 하나다.

한 octree cell 내부를 서로 연결되지 않은 surface sheet 두 개가 지나간다고 생각하자. 이들을 QEF vertex 하나로 합치면 두 component가 cell 내부에서 인위적으로 연결된다.

결과는 다음 중 하나가 될 수 있다.

- non-manifold vertex
- pinched surface
- topology change
- 잘못된 tunnel/handle 생성 또는 제거

Manifold Dual Contouring 계열은 이런 상황에서 cell 내부의 topology component를 구분하고, 필요하면 **한 leaf에 여러 dual vertex**를 허용하는 방향으로 확장한다.

따라서 GPU data model도 장기적으로는

`Leaf -> VertexHandle`

보다

`Leaf -> ComponentRange -> VertexHandle[]`

형태가 더 일반적이다.

### 4.7 Sign mask만으로 충분하지 않은 경우가 있다

Cell corner의 inside/outside 8-bit sign mask는 classification에 매우 유용하지만, adaptive topology에서는 그것만으로 surface component의 실제 연결성을 항상 충분히 설명하지 못한다.

특히 다음 요소가 의미를 바꾼다.

- ambiguous configuration
- Hermite crossing 위치
- fine-level child topology
- simplification 전/후 component connectivity
- local feature preservation rule

따라서 `signMask`는 topology source의 일부이지 전체 topology state는 아니다.

### 4.8 Canonical edge ownership이 duplicate topology를 막는다

GPU에서 모든 active leaf가 자기 주변 edge를 독립적으로 검사하면 같은 polygon을 여러 thread가 생성하기 쉽다.

한 가지 일반적 설계는 **topology primitive마다 canonical owner를 정의하는 것**이다.

예를 들어 owner는 다음과 같은 logical ordering으로 결정될 수 있다.

- finest relevant edge segment
- axis
- Morton/Z-order spatial key
- minimum incident leaf key

그러면 topology generation이 `many cells -> one event`의 reduction 문제로 바뀐다.

이 구조는 두 가지 장점이 있다.

1. duplicate face 제거를 사후 hash 단계에 의존하지 않는다.
2. incremental update에서 어떤 topology event가 dirty인지 추적하기 쉬워진다.

### 4.9 Face winding은 sign transition과 coordinate convention의 contract다

Dual polygon의 vertex set이 같아도 winding이 뒤집히면 normal orientation이 깨진다.

Typical orientation contract는 다음 세 정보를 함께 사용한다.

- primal edge axis
- scalar sign transition direction
- world/grid handedness

Adaptive hierarchy에서는 coarse/fine leaf ordering이 매번 동일하지 않을 수 있으므로, incident vertex를 단순 thread-arrival order로 연결해서는 안 된다.

Connectivity generation은 **topological cyclic order**를 결정해야 하며, GPU append order와 mesh winding order는 별개의 개념이다.

### 4.10 Self-intersection은 crack-free 다음 단계다

Adaptive mesh는 crack-free이고 manifold하면서도 geometric triangle intersection을 만들 가능성이 있다. 특히 coarse cell의 representative vertex가 cell 밖으로 크게 이동하거나, neighboring QEF vertices가 extreme feature를 따라 비정상적으로 배치되는 경우다.

이 때문에 adaptive contouring 문헌에는 **intersection-free contouring**이라는 별도 문제 설정이 존재한다.

즉 robust extractor의 validation hierarchy를 다음처럼 보는 것이 유용하다.

`coverage/crack-free -> manifoldness -> orientation -> self-intersection -> geometric error`

각 단계는 다른 failure mode를 잡는다.

### 4.11 GPU-friendly topology metadata

Adaptive DC의 hot metadata는 가능한 작고 predictable한 편이 좋다.

예시 logical SoA:

- `leafMorton[]`
- `leafLevel[]`
- `signMask[]`
- `componentOffset[]`
- `componentCount[]`
- `vertexHandle[]`
- `topologyFlags[]`
- `epoch[]`

Cold/debug metadata:

- full Hermite constraints
- QEF spectrum/residual
- manifold component labels
- invalid edge diagnostics
- source brick ID
- LOD transition reason

Topology event buffer는 다음처럼 생각할 수 있다.

- canonical edge key
- transition axis
- incident component handles
- output polygon count/index
- validity/epoch

이런 **hot/cold split**은 topology kernel의 bandwidth와 debug observability를 동시에 관리하는 방법이다.

### 4.12 Recursive octree traversal을 GPU에서는 flat scheduling으로 바꾼다

CPU reference implementation에서는 `cellProc / faceProc / edgeProc`처럼 octree node pair를 재귀적으로 내려가는 표현이 자연스럽다.

GPU에서는 recursion 자체보다 다음 형태가 더 예측 가능하다.

1. active leaf/component metadata 생성
2. cross-level adjacency 또는 edge-event 후보 생성
3. canonical key 기준 deduplication/compaction
4. event별 incident component 수 계산
5. prefix sum으로 face/index output offset 계산
6. connectivity emit
7. validation counter 집계

이 구조는 topology extraction을 **irregular traversal problem에서 stream-compaction problem**으로 바꾼다.

### 4.13 Atomics와 prefix sum의 선택

Face/index buffer를 atomic append로 만들면 architecture가 단순하다. 그러나 event당 polygon size가 다르고 workload가 크면 contention과 output ordering variability가 생긴다.

두-pass 구조에서는

`count -> prefix sum -> deterministic write`

로 바꿀 수 있다.

장점:

- exact output size를 사전에 알 수 있음
- deterministic buffer offset
- debug/replay가 쉬움
- allocator fragmentation이 적음

비용:

- metadata pass가 추가됨
- scan buffer와 synchronization이 필요함

Dynamic sparse volume에서 topology update가 작은 영역에 국한된다면 atomic append가 더 싸고, 대규모 rebuild에서는 count/scan이 더 안정적인 경우가 많다.

### 4.14 Stable logical handle과 physical compaction을 분리한다

Adaptive refinement/merge가 일어나면 vertex/index buffer의 physical slot이 크게 바뀔 수 있다.

Application layer가 raw vertex index를 persistent identity로 사용하면 다음 frame에 selection/metrology anchor가 엉뚱한 geometry를 가리킬 수 있다.

따라서 이전에 path-state와 scene-state에서 봤던 원칙이 다시 등장한다.

`Logical Surface Handle != Physical Mesh Slot`

예를 들어 stable identity는

`{brick/cell key, topology component, generation}`

처럼 표현할 수 있고, compacted vertex buffer는 별도 remap table을 통해 연결될 수 있다.

### 4.15 Dirty region은 geometry halo보다 topology halo가 더 넓을 수 있다

Dynamic SDF에서 scalar sample 몇 개만 바뀌어도 영향은 해당 cell에만 머물지 않는다.

- edge sign transition이 바뀐다.
- cell component decomposition이 바뀐다.
- coarse/fine neighbor connectivity가 바뀐다.
- refinement/merge가 연쇄될 수 있다.
- 2:1 balancing을 쓰면 주변 cell refinement가 전파된다.

따라서 incremental meshing에서는

`dirty field region`

`dirty vertex-placement region`

`dirty topology region`

을 동일한 bitmask로 취급하지 않는 편이 좋다.

Topology는 인접 leaf 관계를 사용하므로 **topology halo**가 별도의 dependency radius를 가진다.

### 4.16 Epoch consistency는 topology에서도 필요하다

한 polygon의 incident vertex 중 일부는 current SDF epoch, 일부는 previous epoch에서 생성됐다면 index buffer는 메모리상 유효해도 geometry semantics는 깨진다.

특히 CUDA compute가 field/QEF를 갱신하고 Vulkan rendering이 mesh를 소비하는 구조에서는 다음 snapshot이 함께 맞아야 한다.

- field epoch
- hierarchy/refinement epoch
- component decomposition epoch
- vertex placement epoch
- topology/index epoch

이것은 단순 fence 문제가 아니라 **semantic snapshot consistency** 문제다.

### 4.17 어떤 metric을 프로파일링할 것인가

Adaptive topology는 triangle count 하나로 판단하기 어렵다.

유용한 지표:

- active leaf 수
- surface component 수 / leaf
- cross-level edge event 비율
- generated polygon 수
- duplicate topology event 수
- non-manifold edge/vertex count
- boundary crack count
- self-intersection candidate count
- topology rebuild bytes/frame
- compaction bytes/frame
- L1/L2 hit rate
- average incident leaf/component fan-in
- dirty topology region / dirty field region 비율
- topology epoch mismatch count

특히 **“triangle 수는 줄었는데 topology metadata traffic이 늘어 frame time이 악화되는 경우”**를 잡으려면 geometry count와 memory traffic을 함께 봐야 한다.

## 5. 내 관심 분야와 연결

Semiconductor process visualization은 adaptive topology 문제와 잘 맞는다.

예를 들어 다음 geometry가 한 장면 안에 동시에 존재할 수 있다.

- 넓고 평평한 substrate/wafer
- 매우 얇은 oxide
- 좁은 trench
- sharp spacer corner
- deposition/etch로 계속 변하는 curved profile
- 서로 거의 닿는 thin gap

이때 curvature-aware hierarchy는 geometry budget을 줄여주지만, thin layer가 coarse/fine boundary를 가로지르면 topology stability가 더 중요해진다.

Level-set/SDF compute가 CUDA에서 수행되고 mesh가 Vulkan/Babylon.js 계층으로 전달되는 pipeline에서는 **mesh extraction을 단순 visual export가 아니라 persistent GPU data product**로 보는 편이 좋다.

특히 다음 연결이 중요하다.

- `Sparse Field LOD`와 `Mesh LOD`는 동일하지 않을 수 있다.
- mesh vertex handle은 physical compacted buffer index와 분리될 수 있다.
- changed brick만 remesh할 때 topology halo가 필요하다.
- metrology/picking용 identity는 adaptive refinement 이후에도 안정적이어야 한다.
- thin-film component가 coarse cell에서 잘못 merge되면 visual artifact가 아니라 process geometry semantics가 변한다.

게임 엔진 관점에서도 같은 원리는 voxel terrain, destructible geometry, runtime SDF meshing, fluid/particle surface extraction에 그대로 연결된다.

## 6. 머릿속에 남길 질문 3개

1. **왜 adaptive mesh에서 “crack-free”가 곧 “manifold”를 의미하지 않으며, 두 보장은 어떤 서로 다른 local invariant를 요구하는가?**
2. **2:1 balanced octree를 사용하지 않아도 Adaptive Dual Contouring은 가능하지만, GPU implementation에서는 왜 balancing이 memory-vs-control-flow trade-off가 되는가?**
3. **dynamic sparse SDF에서 field sample 하나가 바뀌었을 때 dirty geometry 영역보다 dirty topology 영역이 더 커질 수 있는 이유는 무엇인가?**

## 7. graphics engineer 면접 질문 1개와 답변

### 질문

**“Adaptive Dual Contouring에서 coarse/fine LOD 경계의 crack만 없애면 topology 문제는 해결된 것 아닌가요?”**

### 답변

아니다. Crack-free는 서로 다른 LOD가 boundary를 같은 connectivity로 덮는다는 **coverage/adjacency 보장**이다. 하지만 2-manifoldness는 각 mesh edge와 vertex 주변의 local neighborhood가 올바른 surface fan을 이루는지에 대한 별도 조건이다.

예를 들어 한 coarse cell 안을 서로 분리된 surface component 두 개가 지나가는데 cell당 dual vertex 하나만 만들면, coarse/fine seam에는 crack이 없어도 두 component가 하나의 vertex에서 합쳐져 non-manifold 또는 pinched topology가 될 수 있다.

또 crack-free와 manifold를 만족해도 coarse QEF vertex가 크게 이동하면서 triangle이 공간에서 서로 교차할 수 있으므로 intersection-free geometry는 또 다른 보장이다.

따라서 robust Adaptive DC pipeline은 대략 다음을 분리해서 본다.

`vertex placement -> crack-free connectivity -> manifold component handling -> orientation -> self-intersection validation`

GPU 관점에서는 topology event를 canonical key로 만들고, leaf가 여러 topology component를 가질 수 있는 data model을 사용하며, physical vertex compaction과 logical surface identity를 분리하는 것이 중요하다.

## 8. 포트폴리오 / 커리어 연결

이 주제는 graphics engineer가 단순 polygonization algorithm을 넘어 **geometry system architecture**를 이해한다는 것을 보여주기 좋다.

포트폴리오에서 설명할 수 있는 관점은 다음과 같다.

- **Geometry algorithm:** primal sign transition과 dual connectivity의 관계
- **Topology:** watertight, manifold, orientation, self-intersection을 별도 invariant로 정의
- **Adaptive hierarchy:** unrestricted octree와 2:1 balancing의 trade-off
- **GPU compute:** recursive topology logic을 edge-event + compaction + prefix-sum pipeline으로 flatten
- **Memory layout:** leaf/component/edge-event의 SoA와 hot/cold split
- **Incremental update:** dirty field region과 dirty topology halo의 분리
- **Engine architecture:** stable logical surface handle과 compacted vertex/index buffer의 분리
- **Interop:** CUDA-produced geometry snapshot과 Vulkan/renderer consumer epoch 관리
- **Debugging:** non-manifold count, duplicate event, topology mismatch를 profiler/debug overlay로 관측

면접에서는 “Dual Contouring을 안다”보다 **왜 adaptive connectivity가 QEF와 별개이고, GPU에서는 어떤 state/layout/synchronization 문제로 바뀌는지 설명할 수 있는가**가 더 강한 신호가 된다.

## 9. 내일 이어서 볼 개념

**Incremental GPU Meshing on Dynamic Sparse SDFs: Dirty-Brick Propagation, Topology Halos, and Stable Surface Handles**

오늘은 adaptive hierarchy의 한 snapshot에서 crack-free/manifold connectivity를 만드는 문제를 봤다. 다음은 field가 매 frame 또는 simulation step마다 변할 때 **전체 mesh를 재생성하지 않고 changed region만 갱신하는 문제**다.

연결 흐름은 다음과 같다.

`curvature-aware refinement -> topology-stable adaptive connectivity -> dirty-region incremental remeshing -> stable mesh identity`

다음 노트에서는 dirty bitset, hierarchy dependency propagation, topology halo, generation handle, partial vertex/index compaction, draw-indirect update, CUDA/Vulkan snapshot handoff를 중심으로 본다.

## 10. 참고 키워드

- Dual Contouring of Hermite Data
- Adaptive Dual Contouring
- Primal Edge / Dual Polygon
- Octree Contouring
- Crack-Free / Watertight Mesh
- 2-Manifold Surface
- Non-Manifold Vertex / Edge
- Intersection-Free Contouring
- Topology-Preserving Simplification
- Manifold Dual Contouring
- Surface Component Decomposition
- QEF (Quadratic Error Function)
- Hermite Data
- 2:1 Balanced Octree
- Restricted / Unrestricted Octree
- Coarse-Fine Transition
- Canonical Edge Ownership
- Morton / Z-order Key
- GPU Prefix Sum / Stream Compaction
- Stable Surface Handle
- Topology Halo
- Semantic Epoch / Snapshot Consistency
- Transvoxel Transition Cell — Marching Cubes 계열 multiresolution seam 처리와 비교하기 좋은 구조
- Tao Ju et al., **“Dual Contouring of Hermite Data,” ACM SIGGRAPH / TOG, 2002**
- Scott Schaefer, Tao Ju, Joe Warren, **“Manifold Dual Contouring,” IEEE TVCG, 2007**
- Tao Ju, Tushar Udeshi, **“Intersection-Free Contouring on an Octree Grid,” 2006**
- Eric Lengyel, **“Transition Cells for Dynamic Multiresolution Marching Cubes,” 2010/2011**
- Xiana Carrera et al., **“Dual Contouring of Signed Distance Data,” SIGGRAPH 2026**
- Ramaijane et al., **“Evaluation of Manifold Dual Contouring Algorithms Based on K-d Tree and Octree Data Structures,” Informatica, 2024**
