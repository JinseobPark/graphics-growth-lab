---
title: "Incremental GPU Meshing on Dynamic Sparse SDFs: Dirty-Brick Propagation, Topology Halos, and Stable Surface Handles"
date: "2026-09-05"
category: Graphics
tags: [GPU, Rendering, SDF, Level Set, Dual Contouring, Sparse Volume, Incremental Meshing, Dirty Region, Topology Halo, Stable Handle, CUDA, Vulkan, Compute Shader, Stream Compaction, Memory Layout, C++]
level: intermediate
---

# [Daily Graphics Growth] 2026-09-05 - Incremental GPU Meshing on Dynamic Sparse SDFs: Dirty-Brick Propagation, Topology Halos, and Stable Surface Handles

## 1. 오늘의 개념

어제는 **Topology-Stable Adaptive Dual Contouring**에서 한 시점의 adaptive octree가 주어졌을 때 coarse/fine boundary를 crack-free하게 연결하고, manifoldness와 connectivity를 별도 invariant로 관리하는 관점을 봤다.

오늘은 geometry가 계속 변하는 경우로 넘어간다.

> **Dynamic SDF/level-set에서 전체 mesh를 매번 다시 만들지 않고, 실제로 영향받은 sparse brick과 그 topology dependency만 GPU에서 갱신하려면 dirty region을 어떻게 정의하고 persistent mesh identity를 어떻게 유지해야 하는가?**

Incremental GPU meshing의 핵심은 단순히 “변경된 voxel만 다시 Marching Cubes/Dual Contouring 한다”가 아니다. Field update, derivative/QEF update, topology component update, vertex placement, connectivity, compaction, draw command까지 서로 다른 dependency radius와 lifetime을 가진다.

따라서 production pipeline에서는 적어도 다음 상태를 구분하는 편이 좋다.

- **Dirty Field Region**: scalar field 값이 실제로 변경된 영역
- **Dirty Geometry Region**: vertex position/QEF를 다시 계산해야 하는 영역
- **Dirty Topology Region**: sign transition·component decomposition·connectivity가 다시 계산되어야 하는 영역
- **Dirty Allocation Region**: vertex/index storage가 생성·제거·재배치되는 영역
- **Dirty Draw Region**: renderer가 사용하는 mesh range 또는 indirect command가 갱신되어야 하는 영역

이들을 하나의 `dirty=true`로 묶으면 구현은 단순해 보이지만, 실제로는 불필요한 rebuild를 크게 늘리거나 반대로 topology invalidation을 누락하기 쉽다.

## 2. 한 줄 핵심

> Incremental GPU meshing은 **field change를 topology-safe dirty halo로 확장하고, logical surface identity를 physical compacted mesh storage와 분리한 뒤, scan/compaction과 generation handle로 변경된 부분만 재생성하는 persistent geometry pipeline**이다.

## 3. 왜 중요한가

Dynamic implicit geometry에서는 매 frame 전체 mesh를 재생성하는 방식이 가장 단순하지만, sparse field의 장점을 상당 부분 잃는다.

예를 들어 전체 virtual volume은 매우 크더라도 실제 process step에서 변하는 영역이 wafer 표면의 얇은 narrow band뿐일 수 있다. 그럼에도 full-volume classification과 full-mesh rebuild를 수행하면 다음 비용이 반복된다.

- 모든 brick의 scalar read
- 모든 active cell의 edge/sign classification
- 전체 vertex/QEF regeneration
- 전체 index generation
- large prefix scan 또는 global allocator pressure
- vertex/index buffer 전체 또는 대규모 부분 write
- renderer용 acceleration structure나 draw range invalidation

반대로 dirty brick만 너무 좁게 잡으면 더 위험하다. Cell의 corner sample 하나가 바뀌면 그 sample을 공유하는 neighboring cell의 sign mask도 달라진다. Adaptive hierarchy라면 refinement/merge가 이웃으로 전파되고, topology event가 canonical primal edge를 기준으로 구성될 경우 해당 edge 주변 incident leaf/component도 다시 평가되어야 한다.

즉 **field dependency radius와 topology dependency radius는 같지 않다.**

이 차이를 명시적으로 모델링하면 update cost가 `전체 scene 크기`보다 `실제 surface change 면적 + dependency halo`에 비례하도록 만들 수 있다.

이 접근은 실무에서도 이미 중요한 방향이다. NVIDIA의 nvblox는 sparse voxel blocks 위에서 GPU 가속 volumetric mapping과 mesh reconstruction을 수행하며, 2026년 4월 v0.0.10 changelog에는 더 빠른 mesh update를 위한 `FlatMeshIntegrator`가 추가되었다. 최근 Isaac ROS nvblox 성능 표도 TSDF/ESDF와 별도로 meshing 시간을 측정하고 있어, dynamic sparse volume에서 meshing update 자체가 독립적인 GPU workload라는 점을 보여준다.

## 4. 구현 관점

### 4.1 Dirty propagation은 단일 bit가 아니라 dependency graph다

가장 먼저 구분할 것은 **source dirtiness**와 **derived dirtiness**다.

예를 들어 field producer가 brick `B`의 SDF sample을 수정했다고 하자.

`FieldDirty(B)`가 바로 `TopologyDirty(B)`를 의미하지는 않는다. 실제 pipeline은 대략 다음 dependency를 가진다.

`field -> gradient/Hermite -> cell classification/QEF -> topology component -> dual vertex -> connectivity -> compacted buffers -> draw metadata`

각 단계는 서로 다른 neighborhood를 본다.

- scalar interpolation: 보통 local cell corner
- gradient finite difference: 1-cell 이상 halo
- Hessian/curvature: 더 넓은 stencil 가능
- Dual Contouring connectivity: shared primal edge 주변 leaf/component
- adaptive balancing: neighbor LOD까지 전파

따라서 dirty system은 `DirtyMask` 하나보다 stage별 bitfield 또는 generation counter를 가지는 편이 낫다.

예시:

`DIRTY_FIELD | DIRTY_DERIVATIVE | DIRTY_VERTEX | DIRTY_TOPOLOGY | DIRTY_DRAW`

이 구조는 “field는 바뀌었지만 sign topology는 그대로”인 경우와 “sign transition까지 바뀐 경우”를 분리할 수 있다.

### 4.2 Topology halo는 geometry halo보다 넓을 수 있다

Uniform grid Marching Cubes에서는 한 cell의 case index가 8개 corner sample에 의존한다. Sample 하나의 sign이 바뀌면 그 sample을 공유하는 인접 cell들이 영향을 받는다.

Adaptive Dual Contouring에서는 더 넓어진다.

- shared primal edge 주변 incident leaves 재평가
- component split/merge 재검사
- coarse/fine transition relation 재평가
- 2:1 balancing을 쓰면 refinement propagation
- canonical edge owner가 바뀌면 event ownership 변경

그래서 incremental update는 보통

`dirty field bricks -> expanded geometry halo -> expanded topology halo`

처럼 단계적으로 확장하는 것이 안전하다.

중요한 점은 halo 크기를 “대충 한 brick”으로 고정하는 것이 아니라 **사용 중인 reconstruction과 topology algorithm의 실제 dependency radius에서 유도해야 한다**는 것이다.

### 4.3 Brick-level dirtiness와 cell-level dirtiness를 혼합한다

Sparse volume에서는 brick 단위 추적이 allocator와 residency에 유리하지만, surface update가 brick 전체를 덮지 않을 수 있다.

두 극단은 다음과 같다.

- **Brick-only:** metadata가 작고 scheduling이 단순하지만 over-update가 큼
- **Cell-only:** 정확하지만 bitset/queue/scan overhead가 커짐

실용적인 구조는 hierarchical dirty tracking이다.

`DirtyBrick -> local cell bitmask/range -> compacted active cell queue`

먼저 brick 단위로 coarse culling한 뒤, 실제 mesh kernel에서는 brick 내부의 changed cell만 compact하게 처리한다.

### 4.4 Dirty queue는 GPU가 직접 compact하는 편이 자연스럽다

Field update 자체가 CUDA/compute shader에서 발생한다면 CPU가 changed brick 목록을 다시 읽어 판단하는 것은 pipeline bubble을 만들기 쉽다.

GPU에서는 일반적으로 다음 구조가 잘 맞는다.

1. producer kernel이 dirty bitset 또는 dirty flag를 기록
2. predicate 생성
3. prefix sum / scan
4. scatter로 compact dirty queue 생성
5. queue count를 다음 compute dispatch 또는 indirect operation에 전달

Stream compaction은 predicate + scan + scatter의 전형적인 GPU primitive다. NVIDIA GPU Gems의 parallel scan 설명도 compaction을 이 구조로 설명한다.

Dirty ratio가 매우 낮으면 atomic append queue가 더 싸고, dirty ratio가 높거나 deterministic ordering/정확한 output size가 중요하면 count-scan-scatter가 유리할 수 있다.

### 4.5 “Dirty bit clear” 시점은 write 완료 시점이 아니다

자주 생기는 race다.

Meshing kernel이 dirty brick을 consume하기 시작했다고 해서 바로 dirty flag를 clear하면 안 된다. 그 brick이 처리 중인 동안 field producer가 다시 같은 brick을 수정할 수 있기 때문이다.

안전한 방식 중 하나는 **generation/epoch**를 사용하는 것이다.

예시:

- `fieldGeneration[B]`
- `meshedGeneration[B]`

`fieldGeneration != meshedGeneration`이면 dirty다.

Meshing 시작 시 generation 값을 snapshot하고, 완료 후 `meshedGeneration = processedGeneration`으로 기록한다. 그 사이 producer가 generation을 다시 올렸다면 다음 pass에서도 자동으로 dirty 상태가 유지된다.

이 방식은 boolean flag보다 lost-update 문제에 강하다.

### 4.6 Mesh identity와 physical slot을 분리한다

Incremental meshing에서 가장 어려운 문제 중 하나는 **persistent identity**다.

어떤 cell의 vertex가 이번 frame에는 `vertexBuffer[1200]`, 다음 frame compaction 후에는 `vertexBuffer[511]`에 있을 수 있다.

Physical index를 application-level identity로 사용하면 다음 기능이 깨진다.

- picking
- metrology anchor
- selection
- temporal visualization state
- mesh-to-field correspondence
- debug labeling

따라서 logical handle을 따로 둔다.

예시:

`SurfaceHandle = {brickKey, localComponent, generation}`

그리고

`SurfaceHandle -> HandleTable -> PhysicalVertexSlot`

으로 연결한다.

이 구조는 이전 path-state virtualization과 scene generational handle에서 봤던 원칙과 같다.

> **Stable logical identity와 compact physical placement는 서로 다른 문제다.**

### 4.7 Generational handle이 stale reference를 잡는다

Brick이 surface를 잃으면 기존 vertex slot은 해제될 수 있다. 이후 같은 physical slot이 다른 geometry에 재사용될 수 있다.

raw integer index만 저장하면 오래된 reference가 “유효한 다른 vertex”를 조용히 가리키는 ABA-style 문제가 생긴다.

Generational handle은 이 문제를 줄인다.

`handle = {index, generation}`

lookup 시 table의 current generation과 비교하여 stale handle을 reject한다.

Graphics/visualization에서는 crash보다 **잘못된 geometry를 정상처럼 보여주는 silent corruption**이 더 위험할 수 있으므로 generation validation의 가치가 크다.

### 4.8 Per-brick fixed capacity와 global compact buffer의 trade-off

Incremental mesh storage에는 대표적으로 두 전략이 있다.

#### 전략 A: Per-brick mesh block

각 brick에 일정 capacity의 vertex/index storage를 준다.

장점:

- brick update가 local write로 끝남
- 다른 brick의 physical offset이 변하지 않음
- stable range와 incremental upload가 쉬움

단점:

- capacity slack / fragmentation
- worst-case geometry에 맞춘 over-allocation
- large surface complexity에서 overflow 처리 필요

#### 전략 B: Global compact mesh

모든 active geometry를 하나의 compact buffer에 넣는다.

장점:

- memory efficiency와 rendering locality가 좋음
- contiguous draw가 쉬움

단점:

- 작은 local change가 downstream offsets를 대량 변경할 수 있음
- compaction/remap 비용
- persistent identity 관리가 필요

실무적으로는 hybrid가 좋다.

예를 들어 **stable brick-local chunks + periodic global defragmentation** 또는 **logical handle table + compact backing store**를 사용할 수 있다.

### 4.9 Free-list, slab, ring allocator의 성격이 다르다

Dynamic mesh payload는 크기가 가변이다.

- **Free-list:** 일반적이지만 fragmentation과 contention 관리 필요
- **Slab/class allocator:** 몇 개 size class로 제한하면 GPU allocator가 단순해짐
- **Ring allocator:** frame-local transient geometry에 좋지만 persistent topology에는 부적합
- **Buddy allocator:** 큰 range 관리에 유리하지만 metadata/branch 비용 존재

Incremental surface mesh는 보통 persistent lifetime이므로 단순 ring buffer보다 free-list/slab 계열 또는 indirection 기반 compact pool이 더 자연스럽다.

### 4.10 Vertex update와 topology update를 분리하면 rebuild를 줄일 수 있다

Field 값은 바뀌었지만 corner sign pattern과 local surface component connectivity가 그대로인 경우가 많다.

이 경우 필요한 것은

- crossing/Hermite 위치 갱신
- QEF 재평가
- vertex position/normal 갱신

뿐이고 index topology는 그대로 유지할 수 있다.

반대로 sign mask나 component decomposition이 바뀌면 connectivity rebuild가 필요하다.

따라서 fast path를 두 개로 나눌 수 있다.

`Geometry-only Update`

`Topology-changing Update`

이는 dynamic level-set animation에서 매우 중요하다. Surface가 조금 이동하는 대부분의 frame은 topology가 그대로일 수 있기 때문이다.

### 4.11 Topology fingerprint로 fast path를 판정할 수 있다

Topology 변화 여부를 매번 full connectivity build 후 비교하는 것은 비싸다.

Cell/brick마다 compact fingerprint를 유지할 수 있다.

예시 요소:

- sign mask
- component count
- edge-crossing mask
- child/LOD relation
- canonical topology event signature

Old fingerprint와 new fingerprint가 같으면 geometry-only path를 선택하고, 다르면 topology path로 승격한다.

단, hash collision을 correctness에 허용해서는 안 되는 경우 full deterministic bit representation 또는 collision-safe validation이 필요하다.

### 4.12 Partial compaction은 “hole을 모두 없애는 것”이 목적이 아니다

Persistent mesh pool에서 매 frame hole을 완전히 제거하면 오히려 이동량이 커질 수 있다.

Incremental system의 목적은 항상 최소 메모리가 아니라 **update bandwidth와 fragmentation 사이의 균형**이다.

예를 들어 다음 정책이 가능하다.

- 작은 hole은 유지
- chunk occupancy가 threshold 아래로 떨어질 때만 local compaction
- VRAM pressure가 높을 때 global defrag
- hot visible bricks를 contiguous region으로 우선 배치

즉 compaction은 boolean operation이 아니라 policy다.

### 4.13 SoA hot metadata가 dirty scheduling에 유리하다

Dirty propagation/compaction kernel은 geometry payload 전체를 필요로 하지 않는다.

Hot metadata 예시:

- `brickKey[]`
- `fieldGeneration[]`
- `meshedGeneration[]`
- `dirtyStageMask[]`
- `topologyFingerprint[]`
- `componentCount[]`
- `vertexRange[]`
- `indexRange[]`
- `visibilityPriority[]`

Cold payload:

- Hermite samples
- QEF matrices
- full debug diagnostics
- per-component attributes

이렇게 분리하면 scheduling pass가 적은 bytes만 읽고 높은 occupancy를 유지하기 쉽다.

### 4.14 Mesh extraction도 wavefront pipeline처럼 볼 수 있다

Dynamic sparse meshing은 irregular work가 많다.

한 brick은 아무 surface가 없고, 다른 brick은 수백 개 active cell을 가질 수 있다.

따라서 하나의 거대한 kernel에서 모든 일을 처리하기보다 stage queue로 나누는 것이 자연스럽다.

예시:

`DirtyBricks`
→ `ActiveCells`
→ `GeometryUpdateCells`
→ `TopologyUpdateCells`
→ `VertexEmit`
→ `ConnectivityEvents`
→ `IndexEmit`
→ `DrawMetadataUpdate`

이 구조는 이전에 본 **wavefront queue scheduling**과 같은 사고방식이다. Work class별 cost와 divergence 특성이 다르므로 queue를 나누는 것이다.

### 4.15 Compute 결과를 GPU-driven draw로 직접 연결한다

Mesh update가 GPU에서 끝났는데 CPU가 vertex/index count를 readback해 draw count를 다시 기록하면 synchronization 비용이 생긴다.

Vulkan에서는 indirect draw parameters와 draw count를 buffer에 두고 `vkCmdDrawIndirectCount` / `vkCmdDrawIndexedIndirectCount`로 GPU가 실행 시점의 draw count를 읽게 할 수 있다. Mesh shader 경로라면 mesh task indirect count 계열도 같은 철학을 가진다.

즉 pipeline은 다음처럼 이어질 수 있다.

`compute mesh update -> indirect command/count buffer -> graphics draw`

다만 **GPU-generated != synchronization-free**다. Compute shader가 indirect/index/vertex buffer를 쓴 뒤 graphics pipeline이 읽기 전에 올바른 stage/access dependency가 필요하다.

### 4.16 CUDA → Vulkan에서는 allocation 공유와 visibility handoff를 분리한다

CUDA가 SDF와 mesh buffer를 만들고 Vulkan이 렌더링한다면 두 문제가 있다.

1. **Memory sharing / allocation identity**
2. **Producer completion / consumer visibility**

External memory로 같은 allocation을 공유해도 CUDA write가 끝났다는 사실이 Vulkan에 자동 전달되지는 않는다. External semaphore/timeline semaphore 또는 적절한 interop synchronization contract가 필요하다.

또 mesh snapshot은 단순히 `vertexBuffer ready`만으로 충분하지 않을 수 있다.

- vertex buffer epoch
- index buffer epoch
- handle table epoch
- indirect draw count epoch

가 같은 semantic snapshot을 가리켜야 한다.

### 4.17 Double buffering보다 multi-versioning이 필요한 경우가 있다

Compute와 graphics overlap을 위해 `Mesh[N]`을 렌더링하는 동안 `Mesh[N+1]`을 생성할 수 있다.

하지만 dynamic meshing은 update 시간이 일정하지 않고 frame-in-flight가 여러 개일 수 있으므로 단순 ping-pong보다 **versioned mesh snapshot**이 더 일반적이다.

`MeshVersion -> vertex/index/handle/indirect metadata`

Renderer는 완전히 publish된 version만 consume한다.

이렇게 하면 partial update 중간 상태가 화면에 노출되지 않는다.

### 4.18 Dirty priority는 visibility와 simulation importance를 함께 볼 수 있다

모든 dirty brick을 같은 frame에 처리할 필요가 없을 수도 있다.

VRAM이나 frame budget이 제한되면 priority를 둘 수 있다.

예시:

`priority = visibility + screen_error + topology_risk + simulation_activity + age`

- 화면에 크게 보이는 brick
- thin/high-curvature feature
- topology sign change가 발생한 brick
- simulation에서 active한 interface
- 오래 기다린 dirty brick

을 우선 처리한다.

단, topology dependency가 있는 이웃 brick 일부만 publish하면 crack이 생길 수 있으므로 scheduling unit은 개별 brick이 아니라 **topology-consistent update island**가 되어야 할 수 있다.

### 4.19 Update island는 transaction처럼 publish할 수 있다

서로 연결된 dirty topology 영역을 하나의 **update island**로 묶고, island 전체가 준비된 뒤 한 번에 visible version으로 publish하면 partial seam을 막을 수 있다.

개념적으로:

`collect -> expand halo -> rebuild island -> validate -> publish epoch`

이때 renderer는 previous island version 또는 new complete island version 중 하나만 본다.

이는 database transaction과 비슷한 원리의 **geometry atomicity**다.

### 4.20 실패를 찾기 위한 지표

Incremental meshing은 평균 frame time만 보면 correctness bug를 놓치기 쉽다.

중요한 metric:

- dirty bricks/frame
- expanded geometry bricks/frame
- expanded topology bricks/frame
- dirty amplification ratio
- geometry-only vs topology-update 비율
- vertices/indices rewritten per frame
- compaction bytes/frame
- allocator fragmentation
- stale-handle rejection count
- mesh generation lag
- topology epoch mismatch count
- visible stale-region count
- indirect draw count update latency
- L1/L2 hit rate
- atomic contention / scan time
- update island size distribution

특히 **Dirty Amplification Ratio = topology dirty volume / source field dirty volume**은 algorithm과 halo policy가 얼마나 많은 추가 work를 만드는지 보여주는 좋은 시스템 지표다.

## 5. 내 관심 분야와 연결

Semiconductor process emulation/visualization에서는 이 구조가 매우 직접적이다.

Etch, deposition, CMP, oxidation 같은 process step은 전체 wafer volume을 항상 크게 바꾸지 않는다. 실제 변화는 interface 주변의 narrow band에 집중되는 경우가 많다. 따라서 dense full rebuild보다 **changed interface brick 중심의 incremental meshing**이 훨씬 자연스럽다.

특히 다음 요구와 연결된다.

- thin oxide/gate layer의 topology 유지
- trench sidewall이나 corner의 local high-resolution mesh
- process parameter 변경 후 빠른 visual feedback
- mesh 기반 picking/metrology의 persistent identity
- CUDA simulation 결과를 GPU에서 그대로 rendering pipeline으로 전달
- sparse NanoVDB/brick grid와 explicit mesh의 hybrid representation

여기서 중요한 실무 포인트는 **simulation grid identity와 rendered mesh identity를 직접 같다고 가정하지 않는 것**이다.

Simulation brick/cell은 merge/refine되거나 storage slot이 이동할 수 있고, mesh vertex도 compaction으로 이동한다. 따라서 두 계층 사이에는 stable logical mapping이 필요하다.

예를 들어 metrology tool이 어떤 sidewall point를 추적한다면 raw vertex index 대신 `surface component handle + field-space anchor` 같은 표현이 더 안정적이다.

게임 엔진 관점에서는 voxel terrain destruction, runtime boolean/CSG, fluid surface, deformable implicit geometry에서도 동일하다. Full remesh 대신 local update를 수행하면서 renderer의 indirect draw와 GPU culling까지 직접 연결하면 CPU involvement를 줄일 수 있다.

CFD/visualization 관점에서는 moving interface나 iso-surface가 매 step 변할 때, solver field와 visualization mesh의 update frequency를 분리할 수도 있다. Simulation은 매 step 진행하되 meshing은 visual importance와 topology change에 따라 budgeted update하는 식이다.

## 6. 머릿속에 남길 질문 3개

1. **왜 dynamic SDF에서 field가 바뀐 brick만 remesh하는 것으로는 충분하지 않고, topology halo를 별도로 확장해야 하는가?**
2. **persistent picking/metrology가 필요한 renderer에서 raw vertex/index offset을 identity로 사용하면 compaction 이후 어떤 종류의 silent bug가 생기는가?**
3. **geometry-only update와 topology-changing update를 분리하려면 어떤 compact topology fingerprint나 generation state를 유지하는 것이 좋은가?**

## 7. graphics engineer 면접 질문 1개와 답변

### 질문

**“Sparse SDF의 일부 brick만 바뀌었다면 그 brick의 mesh만 다시 생성하면 incremental meshing이 되는 것 아닌가요?”**

### 답변

그렇게 하면 누락될 수 있다. SDF sample은 cell과 brick boundary에서 공유되고, gradient/Hermite/QEF 계산은 neighborhood stencil을 사용하며, Dual Contouring connectivity는 sign-changing primal edge 주변의 여러 cell 또는 adaptive leaf에 의존한다. 따라서 source field dirty region보다 geometry/topology dirty region이 더 넓다.

실무적으로는 먼저 field update가 발생한 brick을 기록한 뒤, 사용 중인 reconstruction stencil과 connectivity rule에 따라 **geometry halo와 topology halo**를 확장한다. 그 후 GPU에서 dirty work를 compaction해 queue를 만들고, topology fingerprint가 유지된 영역은 vertex/QEF만 갱신하는 fast path로, sign/component/connectivity가 바뀐 영역은 topology rebuild path로 보낸다.

또 persistent system에서는 vertex/index buffer를 compact할 수 있으므로 raw physical offset을 identity로 쓰면 안 된다. `logical surface handle -> indirection table -> physical slot` 구조와 generation을 사용해 stale reference를 검출해야 한다.

마지막으로 compute와 rendering이 겹친다면 mesh update를 island/version 단위로 publish해 renderer가 partial topology snapshot을 보지 않게 해야 한다. 즉 incremental meshing의 핵심은 **local recomputation뿐 아니라 dependency propagation, identity, allocation, synchronization을 함께 설계하는 것**이다.

## 8. 포트폴리오 / 커리어 연결

이 주제는 graphics/GPU engineer 포트폴리오에서 algorithm과 systems design을 동시에 보여주기 좋다.

설명 포인트:

- **Sparse geometry:** 전체 volume이 아니라 changed working set만 처리
- **Dependency design:** field/derivative/geometry/topology dirty region을 분리
- **GPU primitives:** atomic append, bitset, prefix scan, stream compaction
- **Dynamic allocation:** slab/free-list/compaction trade-off
- **Stable identity:** generational handle과 physical mesh slot 분리
- **Render pipeline:** compute-generated vertex/index/indirect buffers
- **Synchronization:** CUDA/Vulkan interop와 versioned snapshot publish
- **Temporal stability:** geometry-only vs topology-changing fast path
- **Debugging:** dirty amplification, stale handle, epoch mismatch, fragmentation 측정

면접에서는 단순히 “Marching Cubes를 GPU로 돌렸다”보다 다음 설명이 훨씬 강하다.

> “Field가 변하는 범위와 topology가 invalidate되는 범위가 다르기 때문에 stage별 dirty generation을 유지했고, topology fingerprint로 geometry-only fast path를 분리했다. Mesh storage는 stable handle과 compact physical pool을 분리해 incremental compaction 뒤에도 picking/metrology identity를 유지했다. 결과는 indirect draw buffer까지 GPU에서 생성하고 semantic epoch으로 renderer에 publish했다.”

이 문장은 C++, CUDA/Vulkan, rendering pipeline, memory layout, synchronization, geometry correctness를 한 번에 드러낸다.

## 9. 내일 이어서 볼 개념

**GPU Mesh Pool Defragmentation and Relocation: Slab Allocation, Indirection Tables, and Bandwidth-Bounded Compaction**

오늘은 local field change를 dirty topology island로 확장하고 stable surface handle을 유지하면서 부분 remesh하는 architecture를 봤다. 그러나 update가 누적되면 mesh pool에는 hole과 fragmentation이 생긴다.

내일은

`incremental update -> fragmented persistent mesh pool -> relocation/defragmentation -> cache-coherent draw layout`

흐름으로 이어서 본다.

핵심 질문은 다음이다.

> **Mesh의 logical identity와 draw correctness를 유지하면서 physical vertex/index ranges를 언제, 얼마나, 어떤 GPU bandwidth budget으로 재배치해야 하는가?**

Slab allocator, size class, free-list, compaction threshold, relocation table, copy bandwidth, in-flight frame lifetime, indirect draw remap을 중심으로 연결한다.

## 10. 참고 키워드

- Incremental GPU Meshing
- Dynamic Signed Distance Field (SDF)
- Level Set Narrow Band
- Sparse Voxel / Sparse Brick Grid
- Dirty Region Propagation
- Geometry Halo / Topology Halo
- Dirty Amplification Ratio
- Topology Fingerprint
- Geometry-Only Fast Path
- Topology-Changing Update
- Dual Contouring / Adaptive Dual Contouring
- Hermite Data / QEF
- GPU Stream Compaction
- Parallel Prefix Sum / Scan
- Atomic Append Queue
- Persistent Mesh Pool
- Slab Allocator / Free List
- Generational Handle
- Stable Surface Identity
- Mesh Relocation / Compaction
- Update Island / Geometry Transaction
- Semantic Epoch / Snapshot Versioning
- GPU-Driven Rendering
- Vulkan `vkCmdDrawIndirectCount` / `vkCmdDrawIndexedIndirectCount`
- CUDA–Vulkan External Memory / Semaphore
- SoA Hot Metadata / Cold Payload
- NVIDIA **nvblox** — GPU-accelerated incremental sparse volumetric mapping and mesh reconstruction
- nvblox v0.0.10 (2026-04-16) — `FlatMeshIntegrator` for faster mesh updates
- NVIDIA GPU Gems 3, Chapter 39 — **Parallel Prefix Sum (Scan) with CUDA**, stream compaction의 scan/scatter 구조
- Khronos Vulkan Guide — **VK_KHR_draw_indirect_count**, GPU-side dynamic draw count
- Carrera et al., **“Dual Contouring of Signed Distance Data,” 2026**
- Ju et al., **“Dual Contouring of Hermite Data,” 2002**
