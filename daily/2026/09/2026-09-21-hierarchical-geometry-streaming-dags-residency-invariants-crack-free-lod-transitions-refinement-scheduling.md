---
title: "Hierarchical Geometry Streaming DAGs: Residency Invariants, Crack-Free LOD Transitions, and Refinement Scheduling"
date: "2026-09-21"
category: Graphics
tags: [GPU, Rendering, GPU-Driven Rendering, Geometry Streaming, Continuous LOD, Cluster LOD, DAG, Meshlet, Residency, Refinement Scheduling, Crack-Free LOD, Vulkan, CUDA, Memory Layout, C++]
level: intermediate
---

# [Daily Graphics Growth] 2026-09-21 - Hierarchical Geometry Streaming DAGs: Residency Invariants, Crack-Free LOD Transitions, and Refinement Scheduling

## 1. 오늘의 개념

어제는 **GPU Geometry Page Fault Avoidance**에서 desired geometry page가 없더라도 frame을 멈추지 않고, 항상 resident한 coarse/ancestor representation을 사용한 뒤 비동기 request로 detail을 복구하는 구조를 봤다.

오늘은 그 fallback hierarchy 자체를 더 깊게 본다.

직관적으로는 다음과 같은 tree를 떠올리기 쉽다.

```text
Root
 ├─ Child A
 │   ├─ A0
 │   └─ A1
 └─ Child B
     ├─ B0
     └─ B1
```

하지만 continuous geometry LOD에서는 실제 관계가 항상 단순 tree는 아니다.

최근 NVIDIA의 `vk_lod_clusters`와 `nv_cluster_lod_builder`는 cluster group을 simplification하고 다시 regroup하는 방식으로 continuous LOD를 구성한다. 이 과정에서는 이전 group boundary를 다음 level에서 다시 가로질러 grouping할 수 있기 때문에 coarse/fine 관계가 단순한 1-parent tree보다 **DAG(Directed Acyclic Graph)** 에 가까워질 수 있다.

즉 runtime에서 중요한 것은 “부모 포인터” 자체가 아니라:

> **어떤 coarse representation이 어떤 fine representation을 대체할 수 있고, 어떤 dependency가 만족되어야 refine/coarsen transition을 안전하게 만들 수 있는가?**

다.

오늘은 세 가지를 중심으로 본다.

1. **Residency Invariant**
   - fine child를 사용하려면 어떤 coarse/fallback state가 반드시 유지되어야 하는가.

2. **Crack-Free LOD Transition**
   - 서로 다른 detail의 cluster/chunk가 경계에서 만나도 geometry hole이나 T-junction을 만들지 않으려면 어떤 offline/runtime contract가 필요한가.

3. **Refinement Scheduling**
   - streaming budget이 제한된 상태에서 어떤 region부터 coarse → fine으로 승격할 것인가.

핵심 흐름은 다음과 같다.

```text
Always-Resident Coarse Coverage
        ↓
LOD Error / Visibility Evaluation
        ↓
Refinement Candidate Generation
        ↓
Dependency + Residency Check
        ↓
Streaming / Build
        ↓
Atomic Snapshot Promotion
        ↓
Parent/Generating Representation Retained
until Fine Set is Safe
```

오늘의 가장 중요한 관점은:

> **Refinement는 “fine page 하나를 load하는 것”이 아니라, coarse coverage를 끊지 않으면서 representation set을 transaction처럼 교체하는 것**이다.

---

## 2. 한 줄 핵심

> Hierarchical geometry streaming의 핵심은 **fine detail을 개별 page로 즉시 교체하지 않고, coarse representation이 coverage를 유지한 상태에서 필요한 child/dependency set이 모두 ready된 뒤 snapshot 단위로 refine하며, 경계 topology와 residency invariant가 깨지지 않도록 representation 전환을 원자적으로 다루는 것**이다.

---

## 3. 왜 중요한가

Streaming renderer에서 가장 위험한 bug는 단순 “page missing”보다 **부분적으로 refine된 상태**다.

예를 들어 coarse parent가 4개의 child region으로 대체된다고 하자.

```text
Parent P
→ C0 C1 C2 C3
```

그런데 C0, C1만 resident하고 C2, C3는 아직 streaming 중이라면 다음 선택이 생긴다.

```text
P + C0 + C1
```

를 동시에 그릴 것인가?

아니면:

```text
C0 + C1
```

만 그리고 P를 제거할 것인가?

둘 다 위험할 수 있다.

- Parent와 child가 겹치면 **overlap / z-fighting / duplicated shading**가 생길 수 있다.
- Parent를 너무 빨리 제거하면 **coverage hole**이 생긴다.
- Neighbor chunk와 LOD가 맞지 않으면 **crack / T-junction**이 생길 수 있다.

따라서 geometry LOD streaming은 texture streaming보다 더 강한 **topological transition contract**를 필요로 한다.

### 3.1 Continuous LOD는 단순 discrete mesh swap과 다르다

전통적 discrete LOD:

```text
LOD0 mesh
LOD1 mesh
LOD2 mesh
```

는 object 전체를 한 번에 바꾸는 경우가 많다.

Continuous LOD / cluster LOD에서는:

```text
near region  → fine clusters
far region   → coarse clusters
```

처럼 하나의 object 안에서도 서로 다른 detail이 동시에 존재한다.

따라서 runtime이 선택하는 것은 “LOD level 하나”가 아니라 **hierarchy/DAG의 cut(cut set)** 에 가깝다.

### 3.2 좋은 LOD 선택은 유효한 Cut을 만들어야 한다

유효한 geometry cut은 다음을 만족해야 한다.

- visible surface coverage가 끊기지 않는다.
- 같은 surface region을 coarse/fine이 중복해서 그리지 않는다.
- 경계가 허용된 topology transition을 따른다.
- 필요한 geometry가 resident하다.
- screen-space error budget을 만족한다.

즉:

```text
LOD selection
=
error test
+
topology constraint
+
residency constraint
```

다.

### 3.3 현재 NVIDIA 샘플도 Hierarchy Traversal + Streaming을 함께 다룬다

현재 `vk_lod_clusters`는 runtime에서 LOD hierarchy를 traverse해 renderable cluster list를 만들고, 같은 traversal이 streaming과 상호작용한다. Streaming active 시 newly loaded cluster group의 추가 GPU work도 필요하다.

이 구조가 보여주는 중요한 점은:

> **LOD traversal과 streaming manager는 독립 subsystem이 아니라 하나의 representation-selection pipeline으로 연결된다.**

---

## 4. 구현 관점

### 4.1 Tree보다 DAG로 생각하면 더 일반적이다

직관적인 tree:

```text
Parent
 ├─ Child 0
 └─ Child 1
```

는 설명이 쉽다.

하지만 cluster group simplification에서는 다음 level grouping이 이전 group boundary를 다시 가로지를 수 있다.

개념적으로:

```text
Fine Groups:
A    B    C

Simplify / Regroup:
 \  / \  /
  X     Y
```

처럼 한 coarse group이 여러 fine group에서 만들어지고, fine group 하나가 여러 coarse relationship에 참여할 수 있다.

그래서 runtime metadata를:

```text
single parent pointer
```

에 과하게 의존시키기보다:

```text
generatingGroup
generatedGroups
dependency range
```

처럼 표현하는 편이 general하다.

### 4.2 “Parent”보다 Generating / Generated 관계

Continuous LOD에서 두 의미를 분리할 수 있다.

#### Generating Geometry

현재 cluster가 만들어지기 전의 더 high-detail geometry.

#### Generated Geometry

그 geometry를 simplification해서 만들어진 더 coarse representation.

방향을 헷갈리지 않게 해야 한다.

```text
Fine Geometry
  ↓ simplify
Coarse Geometry
```

Runtime refine는 반대 방향으로 움직인다.

```text
Coarse
  ↓ refine
Fine
```

따라서 C++ type/API에서도:

```text
GeneratingGroupId
GeneratedGroupId
```

같이 의미를 명확히 하는 것이 좋다.

### 4.3 LOD Error는 Monotonic해야 Traversal이 쉬워진다

Hierarchy traversal에서 internal node의 error bound가 child보다 작아졌다 커졌다 불규칙하면 pruning이 어렵다.

이상적인 관계:

```text
coarser representation
→ equal or larger approximation error
```

즉 root 방향으로 갈수록 error가 monotonic하게 커지는 것이 좋다.

그러면:

```text
projectedError <= threshold
```

를 만족하는 subtree는 더 깊게 볼 필요 없이 coarse/fine 선택을 빠르게 결정할 수 있다.

NVIDIA의 현재 cluster LOD builder도 hierarchy node에 child의 worst-case error를 전파해 conservative traversal을 가능하게 한다.

### 4.4 Screen-Space Error

World-space error `e`를 screen-space로 바꾼다.

개념적으로:

```text
screenError
≈
projected(e, distance, projection)
```

Camera와 가까워지면 같은 geometric error가 큰 pixel error가 된다.

Runtime selection:

```text
if screenError > threshold:
    refine
else:
    keep coarse
```

하지만 streaming이 들어오면 조건이 바뀐다.

```text
if need refine AND children resident:
    refine now

if need refine AND children missing:
    keep coarse
    request children
```

즉 error criterion은 **desire**를 만들고 residency가 **feasibility**를 결정한다.

### 4.5 Desired LOD와 Committed LOD를 분리한다

중요한 두 state:

```text
DesiredRepresentation
CommittedRepresentation
```

예:

```text
desired = fine children
committed = coarse parent
```

Children이 모두 ready될 때까지 committed state는 유지한다.

이 분리가 없으면 streaming latency가 traversal state를 불안정하게 만든다.

### 4.6 Parent Residency Invariant

가장 단순하고 강한 invariant:

> **Fine replacement set이 fully committed되기 전에는 그 영역을 cover하는 coarse representation을 evict하지 않는다.**

즉:

```text
Parent P resident
Children C0..Cn loading

P stays renderable
until C0..Cn become a valid replacement cut
```

이 invariant는 miss를 soft하게 만든다.

### 4.7 Replacement Set은 한 Page가 아닐 수 있다

Fine detail가 여러 page에 분산될 수 있다.

```text
P
→ C0 C1 C2 C3
```

안전한 transition이:

```text
all(C0..C3 ready)
```

를 요구할 수 있다.

또 neighbor transition page까지 dependency일 수 있다.

```text
RequiredSet(P→Fine)
=
children
+ transition/stitch data
+ required material/attribute pages
```

즉 refine request는 **dependency set**을 가질 수 있다.

### 4.8 Refinement Transaction

Refine를 transaction처럼 본다.

```text
IDLE
→ REQUESTED
→ CHILDREN_LOADING
→ CHILDREN_READY
→ SNAPSHOT_PENDING
→ COMMITTED_FINE
```

Rollback:

```text
budget pressure
camera moves away
request stale
```

이면 `CHILDREN_LOADING` 단계에서 cancel할 수 있다.

`COMMITTED_FINE` 이후 parent는 cold/evictable이 될 수 있다.

### 4.9 Partial Child Residency를 어떻게 다룰 것인가

세 가지 policy가 있다.

#### Policy A: All-or-Nothing

모든 child가 ready되기 전까지 parent 유지.

장점:
- reasoning 단순
- crack/coverage 관리 쉬움

단점:
- 일부 ready child를 활용하지 못함

#### Policy B: Partial Refinement

ready child만 refine하고 나머지 region은 parent 유지.

장점:
- progressive detail

단점:
- overlap/coverage mask가 필요
- topology transition 복잡

#### Policy C: Region Masked Parent

Parent geometry를 region mask로 일부만 render.

장점:
- partial refinement 가능

단점:
- parent representation이 spatial subregions를 mask할 수 있어야 함
- extra metadata/branching

대부분의 처음 구현에서는 All-or-Nothing이 훨씬 안전하다.

### 4.10 Coverage Mask

Partial refinement를 지원하려면 coarse representation이 어느 child region을 cover하는지 알아야 한다.

예:

```text
Parent coverage bits:
1111

C0 committed:
parent mask = 0111
```

하지만 arbitrary cluster geometry에서는 region masking이 간단하지 않다.

그래서 offline build 단계에서 replacement unit을 명확히 만들지 않으면 runtime partial refine가 어려워진다.

### 4.11 Crack이 생기는 이유

LOD boundary에서 vertex sample이 다르면:

```text
Fine Edge:
v0 -- v1 -- v2 -- v3

Coarse Edge:
v0 ----------- v3
```

fine side의 `v1`, `v2`가 coarse edge 위에 정확히 있지 않거나 topology가 다르면 crack/T-junction이 생긴다.

특히 adaptive surface extraction에서는 coarse/fine cell boundary가 자주 문제다.

### 4.12 Offline Border Locking

Cluster LOD simplification의 일반적인 방법 중 하나는 group boundary vertex를 simplification 중 고정(lock)하는 것이다.

개념:

```text
Inside group:
simplify aggressively

Shared border:
keep constrained
```

이렇게 하면 동시에 사용될 수 있는 neighboring group 사이의 boundary compatibility를 유지하기 쉽다.

현재 `vk_lod_clusters`의 LOD generation 문서도 cluster group 내부를 decimate하면서 group 간 border를 lock하는 과정을 설명한다.

### 4.13 왜 다음 Level에서 Regroup하는가

항상 같은 group boundary를 lock하면 해당 boundary는 영원히 simplify되지 않는다.

그 결과:

- unnecessary vertices
- poor coarse LOD
- visible grid pattern

이 남을 수 있다.

그래서 다음 iteration에서는 새로운 group이 old border를 가로지르도록 regroup한다.

```text
Level 0 groups:
| A | B | C |

Level 1 groups:
  | X | Y |
```

이 방식이 DAG-like relationship을 만든다.

### 4.14 Runtime Crack-Free는 Offline Contract에 크게 의존한다

Runtime에서 모든 crack을 shader로 고치는 것보다 offline preprocessing에서:

- compatible boundaries
- controlled replacement sets
- monotonic error
- group relations

을 만드는 것이 훨씬 강하다.

Runtime은 그 contract를 지키는 cut만 선택한다.

> **좋은 runtime LOD는 좋은 offline hierarchy 위에 세워진다.**

### 4.15 Transition Mesh / Stitching

Adaptive SDF/voxel에서는 offline precompute가 어려울 수 있다.

그때는 runtime transition data가 필요할 수 있다.

예:

```text
coarse cell
↔ fine cells
```

사이에:

- transition polygons
- Transvoxel-like transition cells
- skirts
- seam patch

를 사용할 수 있다.

이 경우 transition data도 residency dependency다.

### 4.16 Transition Data도 Snapshot Versioning 대상이다

Fine child가 ready됐는데 stitching page가 old version이면 crack이 생길 수 있다.

따라서:

```text
Geometry children
+
Transition data
```

를 같은 refine transaction에서 commit해야 한다.

즉 LOD transition correctness도 어제의 immutable snapshot principle과 연결된다.

### 4.17 Neighbor Dependency

Refinement가 local decision처럼 보여도 neighboring region의 LOD와 관계가 있을 수 있다.

Constraint 예:

```text
|LOD(A) - LOD(B)| <= 1
```

이런 2:1 balance rule을 사용하면 crack-free transition이 훨씬 단순해진다.

하지만 한 region refine가 neighbor refinement까지 연쇄적으로 요구할 수 있다.

### 4.18 Refinement Cascade

```text
A wants LOD0
B is LOD3
```

2:1 balance라면:

```text
B → LOD2
```

를 먼저 또는 함께 요청해야 할 수 있다.

즉 한 refine request가 neighbor dependency graph를 만든다.

이때 request budget이 폭발하지 않도록 cascade depth를 제한하거나 staged refinement가 필요하다.

### 4.19 Screen-Space Error와 Neighbor Constraint의 충돌

A는 screen-space 기준으로 high detail이 필요하지만 B는 멀어서 coarse로 충분할 수 있다.

그러나 topology constraint 때문에 B도 조금 refine해야 한다.

그래서 실제 cost:

```text
RefineCost(A)
=
A bytes
+
required neighbor bytes
+
transition bytes
```

이다.

Quality-per-byte score도 dependency-expanded cost를 써야 한다.

### 4.20 Refinement Benefit

Candidate score:

```text
RefinementBenefit
≈
Projected Error Reduction
× Visual/Semantic Importance
```

Cost:

```text
RefinementCost
≈
Required Bytes
+ Build/Transfer Time
+ Dependency Expansion
```

Priority:

```text
Benefit / Cost
```

로 생각할 수 있다.

### 4.21 Refine와 Coarsen은 대칭이 아니다

Refine:

```text
load fine
→ verify
→ commit fine
→ parent may become cold
```

Coarsen:

```text
ensure parent/coarse representation resident
→ commit coarse
→ retire fine
```

Fine을 먼저 evict한 뒤 parent를 load하면 hole이 생길 수 있다.

따라서 coarsen도 **coarse-first** invariant를 가져야 한다.

### 4.22 Parent Eviction Condition

Parent를 항상 resident하게 유지하면 memory overhead가 커진다.

Root/coarse 몇 level만 always resident하고 intermediate parent는 필요 시 evict할 수 있다.

조건:

```text
Parent evictable
if
all active descendants are committed
AND
fallback above parent remains available
AND
no in-flight snapshot references parent
```

즉 parent residency invariant는 “모든 parent 영구 resident”라는 뜻이 아니다.

### 4.23 Root Residency Budget

Always-resident root가 너무 크면 hierarchy의 의미가 약해진다.

Root budget은:

```text
fast camera-cut coverage
vs
permanent memory cost
```

trade-off다.

Coarse root triangle density를 매우 낮추되:

- silhouette
- material region
- selection identity

를 유지하는 것이 중요할 수 있다.

### 4.24 DAG에서는 Reference/Lifetime가 더 복잡하다

Tree에서는 parent 하나가 child를 소유한다고 생각하기 쉽다.

DAG에서는 coarse/fine group 관계가 여러 source/target을 가질 수 있다.

따라서 physical allocation lifetime을 hierarchy ownership과 직접 결합하지 않는 편이 좋다.

```text
Logical LOD Relation
≠
Memory Ownership
```

Allocation은 snapshot/timeline 기반 reclamation으로 따로 관리한다.

### 4.25 Logical DAG와 Physical Page Table을 분리한다

```text
LOD DAG:
quality/dependency relation

Residency Table:
logical node → physical page

Snapshot:
which DAG cut is committed
```

이 세 개를 분리하면:

- streaming
- defrag
- COW
- LOD traversal

을 독립적으로 바꿀 수 있다.

### 4.26 DAG Cut

Current committed representation을 **DAG cut**으로 생각한다.

조건:

```text
Each visible surface region
is represented by one valid selected representation set.
```

그리고 selected node/group은 모두 resident해야 한다.

Traversal 결과는 단순 node list가 아니라:

```text
ValidCut + ResidencyFeasible
```

이어야 한다.

### 4.27 Desired Cut vs Resident Cut

두 cut을 유지한다.

```text
DesiredCut
= error criterion만 봤을 때 이상적

ResidentCut
= 현재 resident/dependency 조건으로 가능한 cut
```

Difference:

```text
StreamingDebt
= DesiredCut - ResidentCut
```

이렇게 보면 streaming request generation이 자연스럽다.

### 4.28 Refinement Queue

Candidate:

```text
RefinementRequest {
    region/group
    currentRepresentation
    targetRepresentation
    errorReduction
    byteCost
    dependencyCost
    priority
    requestEpoch
}
```

Queue는 quality-per-byte 순서로 처리할 수 있다.

### 4.29 Coarsening Queue

Memory pressure 시:

```text
CoarsenCandidate {
    fineSet
    coarseFallback
    qualityLoss
    bytesFreed
}
```

Score:

```text
BytesFreed / QualityLoss
```

가 높은 것부터 coarsen할 수 있다.

즉 refine와 eviction을 동일 hierarchy에서 양방향으로 볼 수 있다.

### 4.30 Refinement Budget

Frame마다 제한:

```text
MaxTransferBytes
MaxRemeshMs
MaxNewPages
MaxSnapshotPatchEntries
```

을 둔다.

Hierarchy가 많은 candidate를 내도 budget 내에서 일부만 commit한다.

### 4.31 Traversal Budget

LOD traversal 자체도 매우 큰 scene에서는 비용이다.

Current NVIDIA `vk_lod_clusters`는 spatial hierarchy로 LOD group search를 줄이고, renderable cluster list를 GPU에서 생성한다.

핵심은:

```text
subtree error bound
```

로 detail이 너무 fine하거나 불필요한 region을 early exit하는 것이다.

### 4.32 Wide Hierarchy

GPU에서는 binary tree보다 4/8-way node가 유리할 수 있다.

장점:

- depth 감소
- traversal step 감소

단점:

- node fetch 커짐
- child tests 증가

Task/warp 단위로 child를 평가하기 좋은 fanout을 profiler로 선택한다.

### 4.33 SoA Hierarchy Metadata

Hot traversal fields:

```text
bounds[]
error[]
childOffset[]
childCount[]
groupId[]
residencyState[]
```

를 SoA로 두면 필요한 field만 읽는다.

Streaming/debug metadata는 별도 cold buffer로 둔다.

### 4.34 Quantized Error / Bounds

Huge hierarchy에서 metadata bandwidth가 커진다.

- quantized bounding sphere
- 16-bit error
- packed child range

같은 압축이 가능하다.

Traversal이 conservative하면 quantization error를 안전한 방향으로 bias한다.

### 4.35 C++ Strong Type

다음 값은 모두 integer여도 의미가 다르다.

```text
LodNodeId
GroupId
ResidencyPageId
SnapshotEpoch
RefinementEpoch
```

Host builder/runtime API에서 strong type을 쓰면 DAG relation과 physical page index를 혼동하는 bug를 줄일 수 있다.

### 4.36 Mesh Shader와 Hierarchical Streaming

Mesh/task shader pipeline에서는:

```text
Traversal
→ selected cluster logical IDs
→ snapshot residency resolve
→ mesh shader
```

로 late binding이 가능하다.

이 구조는 physical `firstIndex/vertexOffset`에 강하게 결합된 classic indirect draw보다 streaming/defrag에 유연하다.

### 4.37 Ray Tracing과 Hierarchical Streaming

Ray tracing에서는 selected cluster set이 BLAS/CLAS build state와 연결된다.

Current NVIDIA sample은 LOD traversal 결과로 render clusters를 만들고, streaming으로 새 cluster group이 들어오면 필요한 CLAS를 build한다.

즉 refine cost가 단순 geometry upload bytes가 아니라:

```text
geometry load
+
acceleration structure build
```

일 수 있다.

Quality-per-byte보다:

```text
Quality / (Bytes + BuildCost)
```

가 더 정확할 수 있다.

### 4.38 Dynamic SDF에서는 Hierarchy Build도 Incremental해야 할 수 있다

Static asset은 offline DAG를 precompute할 수 있다.

Dynamic SDF geometry에서는:

- topology 변화
- brick 생성/삭제
- meshlet 재생성

이 있으므로 hierarchy 일부를 rebuild해야 할 수 있다.

전체 DAG rebuild보다:

```text
dirty subtree / dirty group
```

단위 incremental rebuild가 필요할 수 있다.

### 4.39 SDF Brick Hierarchy

사용자의 domain에서는 이미 spatial hierarchy를 만들기 좋다.

```text
Wafer Region
→ Brick Group
→ Brick
→ Surface Chunk
→ Meshlet
```

LOD hierarchy와 sparse spatial hierarchy를 완전히 같은 tree로 만들 필요는 없다.

Spatial hierarchy:

```text
where?
```

LOD DAG:

```text
which representation?
```

을 답한다.

두 역할을 분리하면 좋다.

### 4.40 ColumnStack과 Hierarchy

Thin layered semiconductor geometry에서는 XY region은 넓고 Z 방향 layer가 얇을 수 있다.

Hierarchy를 isotropic octree처럼 만들면 비효율적일 수 있다.

예:

```text
XY tile hierarchy
+
vertical layer/column representation
```

이 더 자연스러울 수 있다.

LOD error metric도:

- top-view silhouette
- cross-section boundary
- thin-layer thickness

를 별도로 고려해야 한다.

### 4.41 Crack-Free는 Visual뿐 아니라 Semantic 문제다

Semiconductor layer boundary crack은 단순 1-pixel artifact가 아닐 수 있다.

- material region 연결성
- PN interface
- trench continuity
- measurement cross-section

이 잘못 보일 수 있다.

따라서 scientific visualization에서는 crack-free invariant의 우선순위가 게임 visual LOD보다 높을 수 있다.

### 4.42 Material Boundary Lock

Simplification이 material interface를 움직이면 semantic error가 커질 수 있다.

Border lock 대상으로:

```text
cluster boundary
+
material boundary
+
measurement-critical boundary
```

를 포함할 수 있다.

단 너무 많은 lock은 simplification ratio를 떨어뜨린다.

### 4.43 Geometric Error와 Semantic Error

Traditional metric:

```text
quadric/geometric error
```

만으로 충분하지 않을 수 있다.

Scientific viewer:

```text
TotalError
=
geometricError
+
materialBoundaryPenalty
+
selectedRegionPenalty
+
thinLayerPenalty
```

같은 domain-specific priority를 고려할 수 있다.

Offline simplification과 runtime residency priority 모두에 같은 semantic signal을 재사용할 수 있다.

### 4.44 Refine Hysteresis

Threshold 하나만 쓰면 camera가 경계에서 움직일 때:

```text
refine
coarsen
refine
coarsen
```

이 반복된다.

두 threshold:

```text
refine if error > T_high
coarsen if error < T_low
```

로 hysteresis를 둔다.

`T_high > T_low`.

이렇게 하면 geometry streaming thrash를 줄인다.

### 4.45 Temporal LOD Stability

LOD choice를 바꿀 때 cost도 고려한다.

```text
SwitchPenalty
```

를 score에 넣으면 작은 quality 차이 때문에 representation을 자주 바꾸지 않는다.

특히 streaming latency가 큰 경우 중요하다.

### 4.46 Partial Residency와 Temporal Stability

Fine child 중 90%가 ready됐다고 매 frame child set이 조금씩 바뀌면 flicker/mesh pop이 생길 수 있다.

Transaction commit은:

```text
replacement set complete
→ one snapshot switch
```

로 묶는 편이 안정적이다.

### 4.47 Snapshot State

```text
Snapshot E:
Region R → coarse P

Pending:
C0 C1 C2 C3

Snapshot E+1:
Region R → C0 C1 C2 C3
```

Current frame에는 중간 상태가 노출되지 않는다.

이전 며칠간 다룬 immutable snapshot이 hierarchy streaming의 중심 invariant가 된다.

### 4.48 In-Flight Parent Retirement

Snapshot E가 P를 사용하고 E+1이 children을 사용한다면 P는 E+1 publish 직후 free할 수 없다.

```text
P retire
only after
last consumer of Snapshot E completes
```

즉 LOD transition과 epoch reclamation은 분리할 수 없다.

### 4.49 Refinement Telemetry

필수 지표:

- desired cut triangles/clusters
- resident cut triangles/clusters
- streaming debt
- refinement requests/frame
- coarsening requests/frame
- replacement-set ready latency
- partial-ready child ratio
- canceled refinement ratio
- dependency expansion bytes
- neighbor cascade depth
- transition/stitch bytes
- crack/debug violation count
- refine/coarsen oscillation count
- root/fallback resident bytes
- parent retained bytes
- in-flight retired parent bytes
- traversal node visits
- early-exit ratio
- hierarchy metadata bandwidth
- projected error distribution
- screen-space error after residency constraints

중요한 derived metric:

```text
Useful Error Reduction
----------------------
Streamed + Built Bytes
```

그리고:

```text
DesiredCut Error
-
ResidentCut Error
```

로 streaming debt를 볼 수 있다.

---

## 5. 내 관심 분야와 연결

### Semiconductor Process Visualization

사용자의 current domain에서는 hierarchy streaming의 기준이 일반 game asset보다 더 domain-specific할 수 있다.

예:

```text
Substrate bulk
→ very coarse representation 가능

Gate/trench vicinity
→ high-detail importance

Cross-section plane 주변
→ fine detail pin

Far non-selected layer
→ aggressive coarsen
```

특히 thin film과 material interface는 geometric error가 작아 보여도 semantic importance가 크다.

따라서 simplification error에:

- material boundary
- layer thickness
- selected structure
- measurement region

을 반영할 가치가 있다.

### Sparse SDF

```text
Sparse SDF Brick
→ Surface Chunk
→ Meshlet
```

계층에서:

- SDF hierarchy는 spatial ownership
- mesh LOD DAG는 render representation
- snapshot table은 current committed cut

으로 역할을 나눌 수 있다.

Fine mesh가 준비되지 않았을 때 parent/coarse mesh 또는 SDF raymarch를 fallback으로 유지하고, replacement set이 모두 ready된 뒤 commit한다.

### CFD

Adaptive mesh/refinement에서도 2:1 neighbor balance와 coarse/fine transition이 중요하다.

Rendering LOD와 simulation AMR은 목적은 다르지만:

```text
neighbor level constraint
transition region
refine/coarsen hysteresis
```

라는 시스템 패턴은 매우 비슷하다.

### CUDA / Vulkan

CUDA/Warp:

```text
dynamic mesh build
dependency completion
transition data generation
```

Vulkan Compute:

```text
LOD traversal
desired cut
resident cut
request generation
```

Vulkan Graphics:

```text
committed cut rendering
```

처럼 분리할 수 있다.

### Game Engine / Graphics Career

이 개념은 다음과 직접 연결된다.

- Nanite-like virtualized geometry
- continuous LOD
- meshlet/cluster hierarchy
- virtual geometry streaming
- open-world mesh residency
- Mega Geometry
- procedural terrain
- destructible geometry

면접에서 강한 설명은:

> **“LOD를 error threshold로 고르면 끝 아닌가요?”**

에 대해:

> **“실제 streaming renderer는 valid hierarchy cut, residency feasibility, transition topology, dependency completion까지 만족해야 합니다.”**

라고 설명하는 것이다.

---

## 6. 머릿속에 남길 질문 3개

1. **Desired LOD cut과 Resident LOD cut을 분리하지 않고 traversal에서 non-resident child를 즉시 선택하면 어떤 coverage·snapshot·streaming race가 생길 수 있을까?**
2. **Continuous cluster LOD에서 fixed group border를 모든 level에서 계속 lock하면 crack은 줄어도 coarse simplification quality가 나빠지는 이유는 무엇이며, regrouping이 왜 DAG-like relationship을 만들까?**
3. **Dynamic SDF geometry에서 refinement cost를 단순 streamed bytes가 아니라 neighbor dependency, transition mesh, remesh compute까지 포함해 계산해야 하는 이유는 무엇인가?**

---

## 7. graphics engineer 면접 질문 1개와 답변

### 질문

**“Continuous LOD streaming에서 fine child page 하나가 준비되면 바로 coarse parent를 fine child로 교체하면 되지 않나요?”**

### 답변

대부분의 경우 그렇게 단순하게 교체하면 안 된다.

Coarse parent가 여러 fine child region을 cover한다면 child 하나만 준비된 상태에서 parent를 제거하면 coverage hole이 생길 수 있다. 반대로 parent를 그대로 그리면서 child도 겹쳐 그리면 z-fighting이나 duplicated shading이 생길 수 있다.

또 neighboring region과 LOD가 달라지면 crack이나 T-junction이 생길 수 있고, transition/stitch geometry가 필요한 경우도 있다.

그래서 refine를 **replacement transaction**으로 보는 것이 좋다.

```text
1. Desired fine set 결정
2. 필요한 child/dependency page request
3. coarse parent는 계속 renderable 유지
4. fine replacement set + transition data finalize
5. next immutable snapshot에서 fine set을 atomic하게 commit
6. old snapshot consumer가 끝난 뒤 parent allocation retire
```

Hierarchy가 단순 tree가 아니라 cluster regrouping으로 만들어진 DAG라면 `parent` 하나보다 generating/generated group relationship과 replacement set을 명시적으로 관리하는 편이 더 안전하다.

또 runtime LOD 선택은 screen-space error만으로 끝나지 않는다.

```text
LOD Selection
=
Error Criterion
+
Residency Feasibility
+
Topology Constraint
+
Streaming Budget
```

다.

핵심은:

> **Refinement는 page upload가 아니라 surface coverage의 versioned replacement다.**

---

## 8. 포트폴리오 / 커리어 연결

이 주제는 graphics engineer가 **LOD 알고리즘, streaming, GPU memory lifetime, topology correctness를 하나의 system으로 이해한다**는 것을 보여주기 좋다.

### Geometry / LOD

- continuous LOD
- cluster / meshlet hierarchy
- DAG cut
- screen-space error
- monotonic error
- refine/coarsen hysteresis

### Streaming

- desired vs resident cut
- replacement set
- partial residency
- dependency-expanded cost
- streaming debt

### Topology

- crack-free transition
- border locking
- regrouping
- transition mesh
- 2:1 neighbor balance

### GPU Memory

- parent residency invariant
- parent retirement
- immutable snapshot
- epoch reclamation

### Vulkan / Mesh Shader

- hierarchy traversal
- logical cluster ID
- late residency resolve
- mesh shader rendering
- cluster LOD streaming

### C++ / Data Layout

- DAG node metadata
- SoA hierarchy
- strong IDs
- packed error/bounds
- replacement transaction state

포트폴리오에서는 다음처럼 설명할 수 있다.

> **“Continuous LOD를 단순 level index가 아니라 GPU-traversed cluster DAG의 valid cut으로 관리했습니다. Desired cut과 resident cut을 분리하고, fine replacement set이 모두 ready되기 전에는 coarse coverage를 유지했습니다. Transition dependency와 neighbor LOD constraint를 refinement cost에 포함하고, commit은 immutable snapshot 단위로 수행해 partial residency가 crack이나 hole로 노출되지 않도록 했습니다. Parent allocation은 이전 snapshot의 GPU consumer가 끝난 뒤 timeline 기반으로 retire했습니다.”**

이 설명은 GPU-driven rendering, memory lifetime, LOD preprocessing, streaming architecture를 한 번에 보여준다.

---

## 9. 내일 이어서 볼 개념

**GPU LOD Error Metrics: Projected Geometric Error, Semantic Importance, and Stable Refinement Thresholds**

오늘은 hierarchy/DAG의 valid cut을 어떻게 residency-safe하게 refine/coarsen하는지 봤다.

다음 질문은:

> **어떤 region을 refine해야 한다고 판단하는 “error” 자체를 어떻게 설계해야 하는가?**

학습 흐름:

```text
Residency Miss Avoidance
→ Hierarchical Streaming DAG
→ Stable LOD Error Metric
→ Budget-Aware Refinement
```

다음 노트에서는:

- world-space simplification error
- screen-space projected error
- bounding-sphere conservative error
- monotonic error propagation
- perceptual/semantic importance
- material boundary penalty
- thin-layer penalty
- refine/coarsen hysteresis
- temporal stability
- quality-per-byte integration

을 중심으로 이어간다.

---

## 10. 참고 키워드

- Continuous Level of Detail
- Cluster LOD
- Meshlet Hierarchy
- Geometry Streaming DAG
- Directed Acyclic Graph
- Generating Group
- Generated Group
- LOD Cut
- Desired Cut / Resident Cut
- Replacement Set
- Parent Residency Invariant
- Refinement Transaction
- Coarsening Transaction
- Partial Residency
- Screen-Space Error
- Monotonic Error
- Quadric Error
- Bounding Sphere
- Border Locking
- Cluster Regrouping
- Crack-Free LOD
- T-Junction
- Transition Mesh
- Skirt
- 2:1 Balance
- Neighbor Dependency
- Refinement Cascade
- Refinement Budget
- Streaming Debt
- Refine/Coarsen Hysteresis
- Immutable Snapshot
- Epoch Reclamation
- Mesh Shader
- GPU-Driven Traversal
- RTX Mega Geometry
- `VK_NV_cluster_acceleration_structure`
- `VK_EXT_mesh_shader`
- Dynamic SDF
- Adaptive Meshing
- NVIDIA nvpro-samples — **vk_lod_clusters**
  - https://github.com/nvpro-samples/vk_lod_clusters
- NVIDIA nvpro-samples — **nv_cluster_lod_builder**
  - https://github.com/nvpro-samples/nv_cluster_lod_builder
- NVIDIA vk_lod_clusters — **LOD Generation Documentation**
  - https://github.com/nvpro-samples/vk_lod_clusters/blob/main/docs/lod_generation.md
- meshoptimizer — **clusterlod.h / Continuous LOD Demo**
  - https://github.com/zeux/meshoptimizer/blob/master/demo/clusterlod.h
- Brian Karis et al. — **A Deep Dive into Nanite Virtualized Geometry**, 2021
  - https://advances.realtimerendering.com/s2021/
