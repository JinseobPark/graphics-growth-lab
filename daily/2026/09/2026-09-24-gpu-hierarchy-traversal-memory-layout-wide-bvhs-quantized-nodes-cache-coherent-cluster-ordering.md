---
title: "GPU Hierarchy Traversal Memory Layout: Wide BVHs, Quantized Nodes, and Cache-Coherent Cluster Ordering"
date: "2026-09-24"
category: Graphics
tags: [GPU, Rendering, GPU-Driven Rendering, Hierarchy Traversal, Wide BVH, Quantization, Cache Locality, Cluster Ordering, Meshlet, LOD, Vulkan, CUDA, Memory Layout, C++]
level: intermediate
---

# [Daily Graphics Growth] 2026-09-24 - GPU Hierarchy Traversal Memory Layout: Wide BVHs, Quantized Nodes, and Cache-Coherent Cluster Ordering

## 1. 오늘의 개념

어제는 **GPU LOD Traversal Architectures**에서 수백만 hierarchy node를 GPU가 top-down으로 평가하고, persistent queue 또는 multi-pass frontier를 사용해 current-frame LOD cut과 render worklist를 만드는 구조를 봤다.

오늘은 그 traversal의 성능을 실제로 좌우하는 더 낮은 층으로 내려간다.

> **같은 hierarchy algorithm이라도 node를 몇 byte로 저장하는지, 한 cache line에서 몇 child를 읽는지, child/cluster가 memory에서 어떤 순서로 배치되는지에 따라 traversal 성능은 크게 달라진다.**

Hierarchy traversal은 계산보다 메모리 접근이 병목이 되기 쉽다.

```text
node fetch
→ bounds/error test
→ child index fetch
→ next node fetch
→ ...
```

각 node에서 수행하는 산술은 적은데 VRAM에서 random node를 계속 읽으면 GPU의 높은 arithmetic throughput을 활용하지 못한다.

그래서 오늘은 다음 세 축을 하나의 **memory-layout problem**으로 본다.

1. **Wide Hierarchy Nodes**
   - binary node 두 개씩 내려가는 대신 4/8-way child를 한 번에 평가해 depth와 pointer chasing을 줄인다.

2. **Quantized Node Metadata**
   - bounding volume, error, child metadata를 conservative하게 압축해 node당 byte 수와 memory traffic을 줄인다.

3. **Cache-Coherent Cluster Ordering**
   - 함께 방문될 가능성이 높은 node/cluster를 가까운 address에 배치해 L1/L2 locality와 burst load 효율을 높인다.

여기서 중요한 구분이 하나 있다.

오늘 말하는 **Wide BVH**는 반드시 ray tracing용 `VkAccelerationStructureKHR`의 내부 구조를 뜻하지 않는다. LOD hierarchy, visibility tree, sparse geometry hierarchy처럼 application이 직접 traverse하는 **BVH-like wide hierarchy layout**에도 같은 원리가 적용된다.

최근 NVIDIA `vk_lod_clusters`는 continuous LOD hierarchy를 GPU에서 traverse해 renderable cluster list를 만들고, geometry streaming과 같은 traversal path를 공유한다. 또 현재 sample은 model processing에서 cluster geometry를 binary blob으로 pack하고, vertex position/UV mantissa bit를 줄이는 방식의 compression을 권장한다. 즉 최신 cluster-based renderer에서도 **hierarchy traversal + compact representation + streaming locality**가 하나의 system 문제로 연결된다.

---

## 2. 한 줄 핵심

> GPU hierarchy traversal의 memory-layout 핵심은 **node를 넓고 작게 만들어 depth와 pointer chasing을 줄이고, quantization은 항상 conservative하게 설계하며, spatially/temporally 함께 방문되는 child·cluster를 연속 배치해 “node test당 유용한 byte 수”를 최대화하는 것**이다.

---

## 3. 왜 중요한가

LOD traversal node 하나가 하는 일은 대체로 작다.

```text
load bounds
load error
project/test
load child range
branch/compact
```

Compute cost는 수십 instruction 수준일 수 있지만 node metadata가 크고 access가 random이면 memory latency가 지배한다.

### 3.1 같은 정보를 더 작은 byte로 읽으면 traversal throughput이 오른다

가상의 binary node가 다음처럼 32 byte라고 하자.

```text
bbox min      12 B
bbox max      12 B
child index    4 B
metadata       4 B
-----------------
               32 B
```

한 level에서 2 child를 확인하려면 64 B를 읽는다.

반면 8-wide node를 64 B로 compact하게 저장할 수 있다면 한 node fetch로 최대 8 child 후보를 평가한다.

```text
64 B / 8 children = 8 B per child candidate
```

물론 실제 layout과 cache transaction은 hardware마다 다르지만, 중요한 관점은:

> **Traversal cost를 node 수가 아니라 “candidate child 하나를 평가하기 위해 몇 byte를 가져오는가”로 본다.**

### 3.2 Wide Node는 depth를 줄인다

균형 잡힌 tree에서 branching factor가 증가하면 depth가 감소한다.

```text
binary: depth ≈ log2(N)
8-wide: depth ≈ log8(N)
```

예를 들어 1,048,576개의 leaf를 이상적으로 분할하면:

```text
binary depth ≈ 20
8-wide depth ≈ 7
```

실제 LOD hierarchy는 불균형하고 DAG/level structure가 있어 이론값과 다르지만, pointer chase 횟수가 줄어드는 방향은 같다.

### 3.3 하지만 Wide Node가 항상 빠른 것은 아니다

8-wide node는 한 번에 더 많은 child를 테스트해야 한다.

```text
binary:
2 bounds tests

8-wide:
8 bounds/error tests
```

Pruning이 매우 강하고 대부분 한 child만 유효한 workload에서는 불필요한 child test가 늘어날 수 있다.

따라서 wide node의 trade-off는:

```text
depth reduction + fewer random loads
vs
more child tests + larger node
```

다.

### 3.4 Compressed Wide BVH의 고전적 결과가 여전히 중요한 이유

Ylitie, Karras, Laine의 2017년 **Compressed Wide BVH** 연구는 GPU incoherent ray traversal에서 compressed wide BVH가 memory traffic을 크게 줄이고, uncompressed BVH 대비 hierarchy memory를 35~60% 수준으로 낮추며 traversal 성능을 향상시켰다.

Ray traversal과 LOD traversal은 알고리즘 목적이 다르지만 공통 lesson은 명확하다.

> **GPU traversal에서 node compression은 단순 memory saving이 아니라 bandwidth/latency optimization이다.**

### 3.5 2025년 연구도 Quantized Traversal을 계속 밀고 있다

2025년 공개된 **Minimizing Ray Tracing Memory Traffic through Quantized Structures and Ray Stream Tracing**은 8-wide BVH와 8-bit quantized box/triangle representation을 이용해 전통적인 구조 대비 memory traffic을 크게 줄이는 방향을 제시한다.

이 역시 오늘 LOD hierarchy에 그대로 복사할 수 있는 구현은 아니지만, GPU에서 hierarchy traversal을 최적화할 때 **precision budget을 memory traffic과 교환하는 것**이 현재도 중요한 연구 방향임을 보여준다.

---

## 4. 구현 관점

### 4.1 Binary Node와 Wide Node

Binary hierarchy:

```text
Node
├─ left
└─ right
```

Wide hierarchy:

```text
Node
├─ child 0
├─ child 1
├─ child 2
├─ child 3
├─ child 4
├─ child 5
├─ child 6
└─ child 7
```

Wide node의 장점:

- depth 감소
- pointer chasing 감소
- child metadata를 한 cache line/transaction에 묶기 쉬움
- subgroup/warp가 child를 lane별로 평가하기 좋음

단점:

- empty child slot 가능
- node size 증가
- child test가 많아질 수 있음
- SIMD lane utilization은 좋아도 bandwidth가 늘 수 있음

### 4.2 Wide Node와 Subgroup Mapping

8-wide node라면 subgroup 일부 lane이 child 하나씩 맡을 수 있다.

```text
lane 0 → child 0
lane 1 → child 1
...
lane 7 → child 7
```

결과:

```text
ballot(validChild)
→ prefix sum
→ compact child tasks
```

어제 다룬 wave-coherent traversal과 자연스럽게 연결된다.

즉 node fanout은 단순 tree shape가 아니라 **GPU execution width와 queue compaction pattern**까지 영향을 준다.

### 4.3 Fanout은 Warp Size와 같을 필요가 없다

NVIDIA warp가 32 lane이라고 해서 32-wide node가 자동으로 좋은 것은 아니다.

32-wide node는:

- node metadata가 매우 커지고
- 대부분 child가 reject되면 bandwidth 낭비가 크며
- builder quality가 낮으면 overlap이 커질 수 있다.

그래서 실무에서는:

```text
4-wide
8-wide
16-wide
```

같은 중간 fanout이 cache line, node size, pruning, subgroup occupancy 측면에서 더 균형적일 수 있다.

정답은 scene/hierarchy characteristic과 profiler에 달려 있다.

### 4.4 AoS Node

직관적인 구조:

```cpp
struct Node {
    Child child[8];
};
```

각 child:

```text
bounds
error
childIndex
flags
```

장점:

- 한 node의 child metadata가 연속
- wide-node traversal이 단순

단점:

- traversal 단계마다 필요하지 않은 field까지 함께 읽을 수 있음

### 4.5 SoA-in-Node

한 wide node 내부에서도 SoA처럼 배치할 수 있다.

```text
minX[8]
minY[8]
minZ[8]
maxX[8]
maxY[8]
maxZ[8]
error[8]
childIndex[8]
```

장점:

- SIMD/vector/subgroup evaluation에 유리
- 같은 field가 contiguous

단점:

- child 하나만 읽을 때 여러 array access 필요
- padding/alignment 설계가 복잡

Wide traversal에서는 **AoSoA(Array of Structures of Arrays)**가 자주 좋은 mental model이다.

### 4.6 AoSoA

전체 hierarchy:

```text
WideNode[0]
WideNode[1]
WideNode[2]
...
```

각 WideNode 내부:

```text
child bounds arrays
child error array
child index array
```

즉:

```text
Hierarchy = Array<WideNodeSoA>
```

이 구조는 node fetch locality와 lane-parallel child test를 동시에 노린다.

### 4.7 Hot / Cold Metadata Split

Traversal hot path에서 필요한 field:

```text
bounds
error
child index/range
leaf/group flag
residency summary
```

Cold field:

```text
debug id
streaming statistics
build provenance
full semantic metadata
```

를 분리한다.

```text
HotNodeBuffer
ColdDebugBuffer
```

Traversal이 cold field를 전혀 읽지 않도록 한다.

### 4.8 Quantized Bounds

Node의 child bounds를 parent-relative하게 quantize할 수 있다.

Parent AABB:

```text
Pmin, Pmax
```

Child min/max를:

```text
q = round((child - Pmin) / (Pmax-Pmin) * 255)
```

처럼 8-bit 또는 16-bit로 저장하고 decode한다.

그러면 child AABB 6개 float:

```text
6 × 4 B = 24 B
```

가 8-bit quantization이면:

```text
6 B
```

까지 줄 수 있다.

실제 encoding에는 exponent/scale, padding, conservative expansion이 필요하다.

### 4.9 Conservative Quantization이 핵심

Traversal bound는 false-negative가 위험하다.

잘못된 quantization:

```text
quantized child bounds
⊂ true child bounds
```

이면 실제 visible/refinement-needed geometry를 prune할 수 있다.

원하는 방향:

```text
quantized child bounds
⊇ true child bounds
```

즉:

```text
min → round down
max → round up
```

방향으로 quantize한다.

LOD error도 마찬가지다.

```text
quantizedError >= trueConservativeError
```

가 되도록 upward bias한다.

### 4.10 Quantization은 Error Budget을 사용한다

Bounds quantization 오차와 LOD geometric error를 분리한다.

```text
Total Conservative Error
=
Geometry Approximation Error
+
Metadata Quantization Margin
```

필요하면 quantization margin을 LOD threshold에 포함한다.

즉 compression이 quality guarantee를 몰래 깨뜨리지 않게 한다.

### 4.11 Absolute Coordinate Quantization의 문제

Scene 전체 범위가 매우 크면 16-bit absolute quantization도 local feature에 충분하지 않을 수 있다.

예:

```text
scene extent = 1 km
feature = nm~µm scale
```

이 경우 parent/local-coordinate quantization이 훨씬 적합하다.

```text
local child bounds relative to parent/tile
```

로 dynamic range를 줄인다.

### 4.12 Semiconductor Geometry와 Local Quantization

사용자의 domain처럼 thin layer가 중요한 경우:

```text
XY extent: large
Z thickness: tiny
```

isotropic quantization은 Z precision을 낭비할 수 있다.

대안:

```text
per-axis scale
```

또는:

```text
XY tile-local bounds
Z layer-relative encoding
```

처럼 anisotropic encoding을 고려할 수 있다.

### 4.13 Error Quantization

LOD error는 범위가 넓을 수 있다.

```text
1e-6 → 1e3
```

Linear 16-bit보다:

```text
log2(error)
```

quantization이 relative precision을 더 일정하게 제공할 수 있다.

단 decode된 값은 conservative하도록 위쪽으로 round한다.

### 4.14 Child Index Compression

Hierarchy가 memory에서 local하게 배치되면 child index를 32-bit absolute index 대신 relative offset으로 저장할 수 있다.

예:

```text
childOffset = childIndex - currentNodeIndex
```

범위가 작으면 16-bit/24-bit로 줄일 수 있다.

하지만 builder/order가 바뀌어 offset 범위를 넘으면 fallback encoding이 필요하다.

### 4.15 Base + Delta Encoding

Wide node children이 contiguous하면:

```text
firstChild
childCount
```

만 저장할 수 있다.

더 복잡한 DAG에서는:

```text
baseIndex + packedDelta[i]
```

를 사용할 수 있다.

Continuous LOD hierarchy가 DAG 관계를 갖더라도 **spatial traversal hierarchy 자체는 별도 tree/forest**로 두면 child range를 더 compact하게 만들 수 있다.

### 4.16 Logical LOD DAG와 Spatial Traversal Tree를 분리한다

어제의 LOD relation:

```text
Generating / Generated DAG
```

오늘 traversal acceleration structure:

```text
Spatial hierarchy
```

는 같은 구조일 필요가 없다.

NVIDIA의 continuous LOD builder도 cluster-group relation과 별도로 spatial hierarchy를 생성해 runtime search space를 줄인다.

이 separation이 memory layout을 더 자유롭게 만든다.

### 4.17 Cache-Coherent Node Ordering

Hierarchy의 logical relation만 맞으면 node index 순서는 자유로운 경우가 많다.

나쁜 배치:

```text
parent at 10
child at 500000
neighbor child at 1200
```

좋은 배치:

```text
parent
children
likely next nodes
```

가 가까이 있다.

목표는:

```text
traversal adjacency
≈
memory adjacency
```

이다.

### 4.18 Depth-First Layout

DFS preorder:

```text
parent
child0 subtree
child1 subtree
...
```

장점:

- 한 subtree를 깊게 탐색할 때 locality 좋음

단점:

- GPU가 한 level의 여러 sibling을 parallel하게 보는 traversal에서는 siblings가 멀어질 수 있음

### 4.19 Breadth-First Layout

BFS:

```text
level 0
level 1 all nodes
level 2 all nodes
```

장점:

- multi-pass frontier처럼 level-oriented traversal에 유리

단점:

- subtree 깊게 들어갈 때 parent-child 거리가 커질 수 있음

따라서 traversal architecture와 node ordering은 같이 설계해야 한다.

### 4.20 Persistent Traversal에는 Hybrid Layout이 유리할 수 있다

Persistent queue는 depth/region이 섞일 수 있다.

가능한 layout:

```text
small treelets contiguous
```

로 만든다.

Treelet 내부는 DFS/local ordering,
Treelet 사이에는 spatial/Morton ordering을 사용할 수 있다.

이 방식은 cache-sized chunk를 만들려는 압축 BVH 연구와 같은 방향이다.

### 4.21 Treelet

Treelet은 hierarchy의 작은 contiguous subgraph다.

예:

```text
Treelet 0 = 2~4 KB
Treelet 1 = 2~4 KB
```

Traversal이 한 treelet에 들어오면 여러 next node가 같은 L1/L2 cache line group에 있을 가능성을 높인다.

Treelet size는:

- cache behavior
- node size
- branching factor
- typical traversal depth

에 맞춘다.

### 4.22 Spatial Cluster Ordering

Leaf cluster도 hierarchy node와 별개로 ordering이 중요하다.

`meshoptimizer`는 `meshopt_spatialSortTriangles`와 spatial clusterization을 통해 positional locality가 좋은 순서를 만들 수 있다.

Cluster order를 공간적으로 정렬하면:

- neighboring visible clusters가 가까운 memory에 위치
- streaming request가 contiguous range로 묶일 가능성 증가
- prefetch/read-ahead 효율 개선
- vertex/page reuse 가능성 증가

가 있다.

### 4.23 Morton Ordering

Cluster centroid의 Morton code(Z-order)를 이용하면 간단하게 spatial locality를 1D order로 변환할 수 있다.

```text
(x,y,z)
→ MortonKey
→ sort
```

완벽한 cache-optimal ordering은 아니지만:

- builder 단순
- parallel radix sort 쉬움
- spatial neighbors가 어느 정도 연속

이라는 장점이 있다.

GPU dynamic hierarchy에서도 매우 유용하다.

### 4.24 Hilbert Ordering

Hilbert curve는 Morton보다 locality 특성이 더 좋은 경우가 있지만:

- key generation 복잡
- implementation cost 증가

가 있다.

따라서 real-time rebuild에서는 Morton이 더 현실적인 경우가 많다.

### 4.25 SAH와 Spatial Ordering은 목적이 다르다

SAH는 hierarchy quality:

```text
expected traversal cost
```

를 줄이려 한다.

Morton/spatial ordering은 memory locality:

```text
which node addresses are near?
```

를 개선하려 한다.

둘은 보완 관계다.

좋은 hierarchy라도 random memory layout이면 느릴 수 있고,
좋은 memory ordering이라도 hierarchy overlap이 크면 traversal node visit이 늘어난다.

### 4.26 Cluster Connectivity도 Ordering Signal이 될 수 있다

Spatially 가까워도 mesh topology상 연결되지 않은 cluster가 있을 수 있다.

Ordering score에:

```text
spatial proximity
+
shared vertices / adjacency
```

를 넣을 수 있다.

현재 NVIDIA `nv_cluster_builder` 계열도 spatial locality뿐 아니라 optional weighted adjacency를 clustering cost에 포함하는 방향을 사용했다.

### 4.27 Cache Coherence와 Streaming Coherence를 함께 본다

GPU-local access에 좋은 순서:

```text
nearby cluster contiguous
```

는 streaming에도 좋은 경우가 많다.

```text
visible region
→ contiguous page ranges
→ fewer I/O/upload batches
```

즉 offline ordering 하나가:

- traversal cache
- geometry streaming
- disk/cache file layout

까지 영향을 줄 수 있다.

### 4.28 Binary Blob Packing

현재 `vk_lod_clusters`는 processed cluster group을 binary blob으로 pack해 runtime representation과 streaming cache에 사용한다.

이런 architecture에서는 file ordering과 GPU ordering을 동일하거나 변환 가능한 형태로 설계하면:

```text
disk read
→ RAM blob
→ VRAM upload
```

사이에서 scatter/repack 비용을 줄일 수 있다.

### 4.29 Compression과 Random Access의 Trade-off

일반-purpose compression은 압축률이 높아도 random access가 어렵다.

Traversal metadata는:

```text
random small reads
```

가 많으므로:

- fixed-size quantized node
- block-level compression
- independent treelet compression

이 더 적합할 수 있다.

Geometry payload는 더 큰 block compression을 사용할 수 있다.

### 4.30 Lossless vs Lossy Compression

Hierarchy indices/flags:

```text
lossless required
```

Bounds/error:

```text
conservative lossy quantization possible
```

Vertex positions:

```text
quality budget에 따라 mantissa reduction 가능
```

처럼 field별 policy를 다르게 한다.

현재 `vk_lod_clusters`도 cluster group compression에서 position/UV mantissa bit reduction을 권장한다.

### 4.31 Wide Node의 메모리 계산 예시

가상의 8-wide LOD node를 설계해 보자.

Child별:

```text
quantized center/radius or bounds   8 B
quantized error                     2 B
child delta/index                   2 B
flags                               1 B
padding                             1 B
---------------------------------------
                                   14 B
```

8 child면 112 B다.

여기에 node header 16 B:

```text
128 B / node
```

가 될 수 있다.

128 B는 많은 GPU cache-line/sector 구조와 잘 맞을 가능성이 있지만, 실제 hardware transaction size와 alignment는 architecture마다 다르므로 profiler가 필요하다.

핵심은 **의도적으로 node size target을 잡는 것**이다.

### 4.32 64 B Node를 목표로 하면 어떤 일이 생기는가

8 child를 64 B에 넣으려면 child당 평균 8 B 미만이다.

따라서:

- bounds를 6×8-bit
- error를 8/16-bit
- child index를 base+delta
- flags bit-pack

같은 aggressive encoding이 필요하다.

Decode ALU는 늘지만 VRAM traffic은 줄어든다.

GPU에서는 종종:

```text
more ALU
<
less random memory
```

교환이 유리하다.

### 4.33 Decode Cost

Quantized bound decode:

```text
float3 min = parentMin + qMin * scale;
float3 max = parentMin + qMax * scale;
```

정도의 FMA가 추가된다.

Traversal이 memory-bound라면 이 ALU는 latency hiding에 도움이 될 수도 있다.

하지만 compute-bound visibility pipeline에서는 trade-off가 달라질 수 있다.

### 4.34 Register Pressure

Wide node의 모든 child metadata를 register에 펼치면 register pressure가 높아질 수 있다.

```text
8 children × bounds/error/index
```

를 한 thread가 모두 들고 있지 말고 subgroup lanes에 분산하는 이유다.

즉 wide node는 **data layout + thread mapping**을 함께 설계해야 한다.

### 4.35 Shared Memory Staging

여러 warp가 같은 node/treelet을 반복 읽는다면 workgroup shared memory에 stage할 수 있다.

하지만 hierarchy traversal은 path가 쉽게 갈라지므로 reuse가 낮을 수 있다.

Shared staging은:

- coherent frontier
- same subtree batch

에서만 가치가 있다.

항상 적용하면 copy/barrier overhead가 더 클 수 있다.

### 4.36 Read-Only Cache / L1/L2

Traversal metadata는 write가 거의 없고 read-mostly다.

따라서:

- contiguous node fetch
- compact node size
- spatial ordering

이 cache hit rate를 올린다.

Explicit cache control보다 먼저 **data layout 자체로 locality를 만든다**는 것이 중요하다.

### 4.37 Prefetch

Next node가 어느 정도 예측 가능하면 software prefetch-like load를 일찍 발행할 수 있다.

하지만 GPU shader에서는 register pressure와 dependency chain 때문에 이득이 workload-dependent다.

더 강한 prefetch는 offline layout:

```text
parent and likely children in same cache neighborhood
```

자체다.

### 4.38 Wide Node와 Queue Granularity

Traversal queue item이 node 하나라면 wide node는 한 task의 work량을 늘린다.

```text
binary node task → tiny
8-wide node task → larger
```

이는 persistent traversal에서 task scheduling overhead를 amortize할 수 있다.

즉 어제의 **task grain size** 관점과도 연결된다.

### 4.39 Empty Child Packing

Wide node가 항상 full하지 않다면:

```text
childMask
```

를 둔다.

```text
validMask = 0b00111101
```

subgroup ballot과 결합해 valid child만 평가/emit한다.

Underfilled node가 많으면 wide layout 이득이 줄어든다.

Builder가 fanout utilization을 높이는 것이 중요하다.

### 4.40 Node Occupancy Metric

Profiler/build metric:

```text
average valid children / max children
```

8-wide인데 평균 3.1 child라면 metadata 낭비가 클 수 있다.

Hierarchy quality metric에:

```text
fanout utilization
```

을 포함한다.

### 4.41 Traversal Node Visit와 Bytes를 함께 본다

좋은 optimization은:

```text
node visits ↓
```

만 보는 것이 아니다.

예:

```text
Binary:
100 visits × 32 B = 3.2 KB

Wide:
40 visits × 128 B = 5.1 KB
```

라면 node 수는 줄었어도 traffic은 늘 수 있다.

따라서 metric:

```text
HierarchyBytesRead / selected cluster
```

가 중요하다.

### 4.42 Useful Child Test Ratio

Wide node에서 8 child를 테스트했지만 실제 다음 frontier에 1개만 남았다면:

```text
useful child ratio = 1/8
```

가 낮다.

Scene overlap/LOD distribution에 따라 fanout을 선택해야 한다.

### 4.43 Cache-Line Waste

Node가 80 B인데 64 B aligned cache transaction을 사용한다고 가정하면 두 line에 걸쳐 읽힐 수 있다.

실제 hardware는 sector/cache line 구조가 다양하지만 general lesson은:

> **node size와 alignment를 의도적으로 정하고 struct padding을 방치하지 않는다.**

C++ `sizeof(Node)`, shader layout, serialized blob format을 모두 검증해야 한다.

### 4.44 C++ / Shader ABI

Host struct와 GLSL/SPIR-V layout이 다르면 치명적이다.

명시적으로:

```text
uint32 fields
packed integer arrays
fixed offsets
static_assert(sizeof(...))
```

를 사용한다.

`glm::vec3` 같은 타입은 ABI/padding이 플랫폼 설정에 따라 예상과 다를 수 있어 serialized node에 바로 쓰지 않는 편이 안전하다.

### 4.45 Explicit Packed Types

예:

```text
PackedBounds8
PackedError16
PackedChildDelta16
NodeFlags8
```

처럼 semantic type을 나눈다.

C++ builder가 packing하고 shader가 decode하는 contract를 명확히 한다.

### 4.46 Versioned Node Format

Streaming cache file이 장기간 유지된다면 format version이 필요하다.

```text
HierarchyFormatVersion
EncodingMode
QuantizationBits
Endian/Layout Version
```

현재 `vk_lod_clusters`도 cache format 변경 시 old cache를 reprocess하는 흐름이 있다.

Large preprocessing pipeline에서는 매우 현실적인 문제다.

### 4.47 Dynamic Geometry Rebuild

Static asset은 offline optimal ordering을 만들 수 있다.

Dynamic SDF는 hierarchy 일부가 매번 바뀔 수 있다.

완전 재정렬 비용이 크다면:

```text
stable page/treelet slots
+
local rebuild
```

가 필요하다.

즉 static hierarchy의 cache-optimal layout과 dynamic updateability 사이 trade-off가 있다.

### 4.48 Local Rebuild Island

Dirty region만:

```text
Treelet A
→ rebuild
→ repack
```

하고 상위 hierarchy는 refit할 수 있다.

장점:

- update bandwidth 제한
- stable addresses 유지 가능

단점:

- 장기적으로 global ordering quality 저하

주기적 global rebuild/defrag와 조합할 수 있다.

### 4.49 Refit vs Rebuild

Bounds/error만 바뀌고 topology relation은 유지되면:

```text
refit
```

이 싸다.

Cluster 구성/LOD relation이 크게 바뀌면:

```text
rebuild
```

이 필요하다.

Dynamic geometry system은 refit threshold를 가져야 한다.

### 4.50 Cache-Coherent Rebuild

Rebuild 시 단순 hierarchy quality뿐 아니라 final memory order도 함께 만든다.

```text
build topology
→ choose wide nodes
→ treelet formation
→ spatial/tree traversal ordering
→ serialize/quantize
```

즉 memory layout은 build 마지막 단계의 `memcpy` 문제가 아니다.

### 4.51 Meshlet Payload Ordering

Selected cluster list가 hierarchy order와 비슷하게 나오면 payload ordering도 같은 locality를 가지는 것이 좋다.

```text
hierarchy leaf order
≈
cluster metadata order
≈
geometry payload page order
```

이면 pointer/ID resolve 후 next memory fetch도 locality가 이어진다.

### 4.52 Visibility + Mesh Shader

Vulkan task/mesh shader는 workgroup 기반으로 mesh workload를 만들 수 있고 task shader는 LOD/culling 및 dynamic mesh-workgroup 생성에 사용할 수 있다.

Hierarchy traversal을 compute에서 끝내든 task shader와 일부 결합하든, 최종 cluster IDs가 spatially ordered되어 있으면 mesh shader input fetch locality에 도움이 된다.

### 4.53 Ray Tracing과 Raster의 공통 Metadata

현재 `vk_lod_clusters`는 같은 LOD traversal/streaming 결과를 rasterization과 ray tracing에서 공유한다.

따라서 hierarchy layout optimization은:

```text
raster cluster selection
+
ray tracing CLAS/BLAS work generation
```

양쪽에 영향을 준다.

하지만 hardware RTAS 내부 node layout은 application이 직접 제어하는 hierarchy와 별개다.

### 4.54 프로파일링에서 볼 지표

- hierarchy bytes
- node size distribution
- average fanout utilization
- hierarchy depth
- nodes visited/frame
- children tested/frame
- useful child ratio
- hierarchy bytes read/frame
- hierarchy bytes / selected cluster
- L1/L2 hit rate
- memory transactions/node
- queue tasks/node
- traversal GPU time
- traversal occupancy
- register count
- subgroup active lanes
- node decode ALU instructions
- quantization false-positive rate
- conservative error inflation
- child delta overflow/fallback count
- treelet hit/miss rate
- spatial-order page locality
- selected cluster address span
- streaming batch contiguity
- rebuild/refit bytes
- dynamic treelet fragmentation

가장 중요한 비교는:

```text
Nodes Visited
×
Bytes per Node
```

와 실제 profiler의:

```text
DRAM/L2 Bytes
```

를 함께 보는 것이다.

---

## 5. 내 관심 분야와 연결

### Semiconductor Process Visualization

사용자의 geometry는 일반 game mesh보다 anisotropic하다.

```text
XY: wide wafer/device region
Z : thin films / stacked layers
```

따라서 hierarchy bounds를 단순 isotropic 3D AABB로 저장하는 것보다:

- XY tile-local quantization
- Z layer-relative quantization
- thin-layer semantic error

를 결합하면 memory precision을 더 효율적으로 쓸 수 있다.

### Sparse SDF / Level Set

가능한 구조:

```text
Sparse SDF Brick Hierarchy
      ↓
LOD/Visibility Wide Nodes
      ↓
Surface Chunk / Meshlet IDs
      ↓
Persistent Payload Pool
```

SDF brick 자체의 spatial hierarchy와 render LOD hierarchy는 분리할 수 있지만, brick/cluster order를 spatially 맞추면 meshing output과 rendering payload locality가 함께 좋아질 수 있다.

### Dynamic Meshing

Dirty region이 localized되어 있다면 전체 hierarchy를 매 frame global reorder할 필요가 없다.

```text
local treelet rebuild
+
upper refit
+
periodic global reorder
```

같은 hybrid가 실용적이다.

### CFD / Scientific Visualization

CFD volume/iso-surface는 spatial locality가 매우 중요하다.

Neighboring cells/blocks가 비슷한 timestep/field data를 읽기 때문에:

```text
hierarchy node order
field page order
mesh output order
```

를 함께 맞추면 cache reuse 가능성이 커진다.

### Vulkan / CUDA

CUDA/Warp:

```text
SDF update
mesh generation
local hierarchy/treelet rebuild
```

Vulkan Compute:

```text
wide-node LOD traversal
visibility
render list generation
```

Vulkan Mesh Shader:

```text
cluster payload fetch
primitive emission
```

처럼 역할을 나눌 수 있다.

### Graphics Engineer Career

이 주제는 다음 역할과 직접 연결된다.

- virtualized geometry
- meshlet renderer
- BVH traversal
- GPU scene database
- ray tracing infrastructure
- open-world streaming
- scientific visualization
- GPU memory optimization

면접에서 강한 설명은:

> **“Wide node가 depth를 줄여서 빠릅니다.”**

에서 끝나는 것이 아니라:

> **“Wide node는 node visit을 줄이지만 bytes/node와 child-test 수가 증가하므로, fanout utilization·bytes per selected cluster·cache hit rate까지 함께 봐야 합니다.”**

라고 말하는 것이다.

---

## 6. 머릿속에 남길 질문 3개

1. **8-wide node로 hierarchy depth는 절반 이하로 줄었지만 node size가 4배 커졌다면, `nodes visited`, `bytes/node`, `useful child ratio`, L2 hit rate 중 어떤 metric 조합으로 실제 이득을 판단해야 할까?**
2. **반도체 thin-layer geometry에서 parent-relative 8/16-bit bounds quantization을 사용할 때 XY와 Z를 같은 precision으로 저장하면 왜 비효율적일 수 있으며, conservative anisotropic quantization은 어떻게 설계할 수 있을까?**
3. **Static asset에는 DFS/treelet ordering이 좋고 multi-pass frontier에는 BFS ordering이 좋을 수 있는데, persistent queue traversal에서는 어떤 hybrid ordering이 cache locality와 dynamic load balance를 함께 만족시킬까?**

---

## 7. graphics engineer 면접 질문 1개와 답변

### 질문

**“GPU hierarchy traversal에서 binary BVH를 8-wide BVH로 바꾸면 항상 빨라지나요?”**

### 답변

항상 그렇지는 않다.

Wide node의 가장 큰 장점은 depth와 pointer chasing을 줄이고 여러 child metadata를 한 번에 가져올 수 있다는 점이다. Subgroup lane들이 child를 병렬 평가하기도 쉽다.

하지만 비용도 있다.

```text
Binary node:
작은 node
적은 child tests
깊은 hierarchy

8-wide node:
큰 node
많은 child tests
얕은 hierarchy
```

따라서 scene에서 pruning이 매우 강해 매 node마다 실제로 한두 child만 유효하다면 8-wide의 나머지 child bounds를 읽고 테스트하는 비용이 낭비가 될 수 있다.

또 8-wide node가 128 B이고 binary node가 32 B라면 node visit 수가 4배 이상 줄지 않는 한 raw metadata traffic이 더 커질 수도 있다.

그래서 profiler에서는 다음을 함께 본다.

```text
nodes visited
bytes per node
children tested
useful child ratio
L1/L2 hit rate
DRAM bytes
register pressure
subgroup efficiency
```

Compression도 중요하다. Wide node를 parent-relative quantized bounds, packed error, base+delta child indices로 줄이면 depth 감소의 이득을 유지하면서 bytes/node 증가를 완화할 수 있다.

단 quantization은 반드시 conservative해야 한다.

```text
bounds min → round outward down
bounds max → round outward up
error      → round upward
```

으로 false-negative pruning을 막는다.

핵심은:

> **Wide hierarchy의 목적은 node 수를 줄이는 것이 아니라 traversal 한 번에 필요한 memory traffic과 dependency depth를 줄이는 것이다.**

따라서 가장 좋은 fanout은 algorithm 이름이 아니라 실제 `bytes/read × node visits × useful-child ratio`가 결정한다.

---

## 8. 포트폴리오 / 커리어 연결

이 주제는 graphics engineer 포트폴리오에서 **algorithm optimization을 memory-system optimization으로 연결**하기 좋다.

### GPU Data Structure

- wide BVH-like hierarchy
- fanout trade-off
- treelet
- parent-relative encoding
- base+delta indices

### Memory Layout

- AoS / SoA / AoSoA
- hot/cold split
- quantized bounds
- packed error
- alignment
- serialized GPU ABI

### Rendering

- cluster/meshlet LOD hierarchy
- visibility traversal
- mesh/task shader worklist
- continuous LOD

### GPU Architecture

- memory-bound traversal
- L1/L2 locality
- subgroup child evaluation
- register pressure
- memory transaction efficiency

### Dynamic Geometry

- local treelet rebuild
- hierarchy refit
- periodic reorder
- stable logical IDs

### Scientific Visualization

- anisotropic bounds
- thin-layer precision
- semantic error preservation
- SDF/CFD spatial coherence

포트폴리오에서는 다음처럼 설명할 수 있다.

> **“GPU LOD traversal hierarchy를 wide-node AoSoA로 구성하고, parent-relative conservative bounds/error quantization과 compact child offsets로 node traffic을 줄였습니다. Hierarchy topology와 physical memory ordering을 분리해 treelet-local DFS와 spatial cluster ordering을 적용했고, dynamic SDF update에서는 dirty treelet만 rebuild/refit하도록 했습니다. 최적화 평가는 node visit 수뿐 아니라 hierarchy bytes per selected cluster, L2 hit rate, fanout utilization, useful-child ratio를 함께 사용했습니다.”**

이 설명은 C++ memory layout, GPU cache, hierarchy algorithm, rendering pipeline을 하나로 연결한다.

---

## 9. 내일 이어서 볼 개념

**GPU Treelet Scheduling and Frontier Locality: Cache-Sized Work Batches, Queue Reordering, and Traversal Compaction**

오늘은 hierarchy node와 cluster가 memory에서 어떻게 배치되어야 하는지 봤다.

다음 질문은:

> **좋은 memory layout을 만들어 놓았더라도 persistent queue가 서로 먼 subtree의 task를 무작위로 섞으면 cache locality가 다시 깨지는데, traversal work 자체를 어떻게 cache-coherent batch로 scheduling할 것인가?**

학습 흐름:

```text
GPU LOD Traversal
→ Wide / Quantized Node Layout
→ Treelet-Local Scheduling
→ Frontier Reordering
→ Cache-Coherent Work Batches
```

다음 노트에서는:

- cache-sized treelet
- frontier sort/binning
- spatial key
- queue reordering cost
- locality vs load balance
- subgroup batch traversal
- task stealing과 locality 충돌
- treelet prefetch
- traversal compaction
- dynamic scene에서 stable locality

를 중심으로 이어간다.

---

## 10. 참고 키워드

- GPU Hierarchy Memory Layout
- Wide BVH
- BVH4 / BVH8
- Compressed Wide BVH
- LOD Hierarchy
- Spatial Hierarchy
- Meshlet / Cluster Hierarchy
- AoS / SoA / AoSoA
- Hot / Cold Metadata Split
- Treelet
- Quantized Bounding Volume
- Parent-Relative Quantization
- Conservative Quantization
- Quantized Error
- Log Error Encoding
- Child Delta Encoding
- Base + Delta
- Fanout Utilization
- Useful Child Ratio
- Node Bytes
- Hierarchy Bytes / Selected Cluster
- Cache-Coherent Ordering
- DFS Layout
- BFS Layout
- Morton Order
- Hilbert Order
- Spatial Sorting
- Cluster Connectivity
- L1 / L2 Cache Locality
- Register Pressure
- Subgroup Child Evaluation
- GPU Persistent Traversal
- Dynamic Treelet Rebuild
- BVH Refit / Rebuild
- Dynamic SDF
- Scientific Visualization
- NVIDIA — **vk_lod_clusters**
  - https://github.com/nvpro-samples/vk_lod_clusters
- NVIDIA — **vk_lod_clusters LOD Generation**
  - https://github.com/nvpro-samples/vk_lod_clusters/blob/main/docs/lod_generation.md
- NVIDIA Research — **Efficient Incoherent Ray Traversal on GPUs Through Compressed Wide BVHs**, HPG 2017
  - https://research.nvidia.com/publication/2017-07_efficient-incoherent-ray-traversal-gpus-through-compressed-wide-bvhs
- Grauer, Hanika, Dachsbacher — **Minimizing Ray Tracing Memory Traffic through Quantized Structures and Ray Stream Tracing**, 2025
  - https://arxiv.org/abs/2505.24653
- meshoptimizer — **Spatial Sorting / Meshlet Utilities**
  - https://github.com/zeux/meshoptimizer
- Vulkan Specification — **Shaders / Subgroups / Mesh Shaders**
  - https://docs.vulkan.org/spec/latest/chapters/shaders.html
- Khronos — **VK_EXT_mesh_shader proposal**
  - https://github.com/KhronosGroup/Vulkan-Docs/blob/main/proposals/VK_EXT_mesh_shader.adoc
