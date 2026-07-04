---
title: "GPU Hash Grid for Spatial Lookup"
date: "2026-07-04"
category: "Graphics"
tags: ["GPU", "Rendering Pipeline", "Hash Grid", "Spatial Lookup", "Particles", "Collision", "Fluid Simulation", "Voxel", "Scientific Visualization"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-07-04 - GPU Hash Grid for Spatial Lookup

## 1. 오늘의 개념

**GPU Hash Grid for Spatial Lookup**은 particle, voxel, cell, point, mesh element 같은 많은 spatial element를 GPU에서 빠르게 주변 검색하기 위해 사용하는 acceleration structure다.

문제는 단순하다. N개의 particle이나 cell이 있을 때, 각 element가 주변 neighbor를 찾으려면 naive하게는 모든 element를 비교해야 한다.

```text
for each particle i:
    for each particle j:
        distance(i, j) 검사
```

이 방식은 O(N²)이고, particle 수가 조금만 늘어도 사용할 수 없다. Hash Grid는 공간을 일정한 cell로 나누고, 각 element를 자신이 속한 grid cell에 넣는다. 이후 neighbor search는 전체 N개가 아니라 주변 grid cell 몇 개만 검사한다.

핵심 변화는 다음이다.

> 모든 element를 서로 비교하는 구조에서, 공간 cell을 통해 근처에 있을 가능성이 높은 후보만 검사하는 구조로 바뀐다.

GPU Hash Grid는 particle simulation, SPH fluid, PBD collision, voxel processing, point cloud neighbor query, CFD field sampling, broad-phase collision, marching cubes active cell grouping에서 매우 중요하다.

## 2. 한 줄 핵심

**GPU Hash Grid는 spatial element를 grid cell key로 분류하고, 주변 cell만 조회해 neighbor search와 collision 후보 수를 줄이는 GPU-friendly spatial lookup 구조다.**

## 3. 왜 중요한가

GPU simulation과 visualization에서는 주변 검색이 자주 등장한다.

- SPH fluid에서 주변 particle 찾기
- PBD / XPBD cloth collision 후보 찾기
- particle-particle interaction
- point cloud normal estimation
- voxel 주변 cell sampling
- marching cubes active cell grouping
- CFD field block lookup
- broad-phase collision detection
- screen-space 또는 world-space binning

이 작업을 naive all-pairs로 처리하면 GPU 병렬성이 있어도 데이터 수가 커질수록 감당하기 어렵다. Hash Grid는 공간을 cell로 나누어 neighbor candidate를 제한한다.

Graphics engineer 관점에서 Hash Grid는 단순 자료구조가 아니라, **simulation / rendering / visualization에서 O(N²) 문제를 GPU-friendly O(N + local candidates) 문제로 바꾸는 기본 패턴**이다.

## 4. 구현 관점

### 4.1 Grid cell size 선택

Hash Grid에서 가장 먼저 결정해야 하는 것은 cell size다.

일반적으로 cell size는 search radius와 비슷하게 둔다.

```text
cellSize ≈ neighborSearchRadius
```

SPH fluid라면 smoothing radius, particle collision이라면 collision radius, voxel field라면 sampling footprint가 기준이 될 수 있다.

Cell size가 너무 작으면 한 query가 확인해야 하는 주변 cell 수가 늘어난다. Cell size가 너무 크면 한 cell 안의 element 수가 많아져 후보가 많아진다. 따라서 cell size는 neighbor radius와 data density를 함께 고려해야 한다.

### 4.2 World position을 grid coordinate로 변환

각 element의 world position을 grid coordinate로 변환한다.

```cpp
int3 cell = floor((position - gridOrigin) / cellSize);
```

이 cell coordinate를 hash key로 바꾼다.

```cpp
uint HashCell(int3 c)
{
    uint h = c.x * 73856093u ^ c.y * 19349663u ^ c.z * 83492791u;
    return h % tableSize;
}
```

위 방식은 간단한 hash function 예시다. 실제 구현에서는 collision rate, table size, negative coordinate handling, power-of-two mask 사용 여부를 고려한다.

### 4.3 두 가지 대표 구현: hash table vs sorted grid

GPU Hash Grid는 크게 두 방식으로 구현할 수 있다.

#### 1. Atomic append 기반 hash table

각 particle이 자신의 cell hash를 계산하고, atomic counter로 해당 bucket에 append한다.

장점:

- 구현 개념이 단순하다.
- dynamic data에 바로 적용하기 쉽다.

단점:

- atomic contention이 발생할 수 있다.
- bucket overflow 처리가 필요하다.
- memory layout이 불규칙할 수 있다.
- query 시 linked list나 bucket list traversal이 필요하다.

#### 2. Sort 기반 uniform grid

각 element에 대해 `(cellKey, elementIndex)` pair를 만들고, cellKey 기준으로 GPU sort를 수행한다. 이후 cell별 start/end range를 만든다.

```text
particle → cellKey 생성
(cellKey, particleIndex) pair 생성
cellKey 기준 sort
cellStart[cellKey], cellEnd[cellKey] 생성
neighbor query 시 주변 cell range만 순회
```

장점:

- 같은 cell의 element가 memory상 연속된다.
- query access가 더 coherent하다.
- bucket overflow 문제가 줄어든다.

단점:

- GPU sort 비용이 든다.
- 매 frame dynamic data에서 sort overhead가 크다.

실무에서는 particle 수, movement, query 빈도, target GPU에 따라 두 방식 중 하나를 선택하거나 hybrid를 사용한다.

### 4.4 Neighbor query

3D uniform grid에서 search radius가 cell size와 비슷하다면 보통 현재 cell과 주변 26개 cell, 총 27개 cell만 확인하면 된다.

```cpp
for dz in -1..1
for dy in -1..1
for dx in -1..1
{
    int3 neighborCell = cell + int3(dx, dy, dz);
    range = GetCellRange(neighborCell);
    for each candidate in range:
        distance check
}
```

중요한 점은 hash grid가 정확한 neighbor list를 바로 주는 것이 아니라, **후보(candidate)를 줄여주는 broad-phase**라는 것이다. 최종적으로는 distance check나 shape intersection test가 필요하다.

### 4.5 GPU memory layout

Hash Grid 성능은 memory layout에 크게 좌우된다.

좋은 layout의 조건은 다음이다.

- 같은 cell의 element가 연속 memory에 있다.
- query thread들이 비슷한 cell range를 읽는다.
- position / velocity / attribute가 coalesced하게 읽힌다.
- cell range buffer가 compact하다.
- hash collision이 적다.
- atomic contention이 낮다.

Particle data는 보통 SoA 구조가 유리하다.

```cpp
positions[]
velocities[]
phases[]
cellKeys[]
indices[]
```

AoS보다 SoA가 GPU memory coalescing에 유리한 경우가 많다.

### 4.6 Morton Order와의 연결

이전 노트의 Morton Order는 Hash Grid와 잘 연결된다. Cell coordinate를 hash key로 바로 넣는 대신 Morton key를 사용하면 spatial locality가 key ordering에 반영된다.

Sort 기반 grid에서는 `(mortonKey, elementIndex)`를 정렬해 같은 공간 근처의 element가 memory상 가까이 배치되게 할 수 있다.

```text
cell coordinate → Morton key → sort → cell range 생성
```

Hash function은 빠른 bucket indexing에 유리하고, Morton key는 spatial ordering과 range query에 유리하다. 어떤 key를 쓸지는 lookup 방식에 따라 달라진다.

### 4.7 Collision / simulation에서의 사용

Particle simulation에서는 Hash Grid가 broad-phase 역할을 한다.

- SPH: smoothing radius 안의 neighbor particle 탐색
- PBD fluid: density constraint neighbor 탐색
- Cloth: particle-edge / particle-triangle 후보 탐색
- Rigid body: broad-phase AABB candidate 탐색
- Granular material: local contact 후보 탐색

Hash Grid가 없으면 neighbor search가 병목이 된다. 하지만 grid build, sort, range generation 비용도 있으므로 simulation step 전체에서 어느 정도 비중을 차지하는지 봐야 한다.

### 4.8 Rendering / visualization에서의 사용

Rendering에서도 Hash Grid는 유용하다.

- point cloud splatting에서 nearby point query
- voxelization 후 occupied cell grouping
- volume brick lookup
- screen-space tile/binning
- particle rendering binning
- CFD field block lookup
- sparse marching cubes active cell grouping

예를 들어 CFD field block을 world-space grid cell로 분류하면 slice plane이나 ray marching pass에서 필요한 block 후보를 빠르게 찾을 수 있다.

## 5. 내 관심 분야와 연결

### CFD / scientific visualization

CFD 후처리에서는 cell, block, particle trace, streamline seed, iso-surface patch가 공간적으로 분포한다. Hash Grid를 사용하면 특정 위치 주변의 field block이나 particle 후보를 빠르게 찾을 수 있다.

가능한 응용은 다음과 같다.

- streamline integration 중 현재 위치가 속한 field block 찾기
- point sample query에서 nearby cell 후보 찾기
- particle trace collision / clustering
- iso-surface active cell grouping
- slice plane 주변 block 후보 추출

Scientific visualization에서는 lookup 속도뿐 아니라 정확한 cell mapping이 중요하다. Hash Grid는 후보를 줄이는 broad-phase이고, 최종 field interpolation은 원본 mesh/cell topology 기준으로 검증되어야 한다.

### Sparse voxel / octree / NanoVDB

Sparse voxel에서는 occupied brick이나 active cell을 hash grid로 관리할 수 있다. 특히 dynamic sparse voxel data에서는 tree보다 hash grid가 update하기 쉬운 경우가 있다.

예를 들어 다음 구조가 가능하다.

- world-space brick coordinate → hash key
- hash table에서 brick pool index lookup
- missing brick fallback
- active brick list generation
- marching cubes active cell grouping

NanoVDB나 octree처럼 hierarchical structure가 강한 경우에도, 특정 level의 brick lookup에는 hash grid나 hash map 사고가 사용될 수 있다.

### Game engine architecture

Game engine에서는 Hash Grid가 simulation과 rendering 양쪽에 등장한다.

- particle collision
- cloth self-collision
- fluid simulation
- debris / crowd local query
- decal / light / probe spatial binning
- broad-phase physics
- GPU particle renderer

면접에서는 Hash Grid를 “해시맵”으로만 말하기보다, “공간을 cell로 양자화하고 주변 cell만 검사해 all-pairs를 local candidates 문제로 바꾸는 구조”라고 설명하는 것이 좋다.

## 6. 머릿속에 남길 질문 3개

1. GPU Hash Grid가 all-pairs neighbor search를 local candidate search로 바꾸는 원리는 무엇인가?
2. Atomic append 기반 hash table과 sort 기반 uniform grid는 각각 어떤 장단점이 있는가?
3. CFD field block lookup이나 SPH particle neighbor search에서 cell size를 search radius와 비슷하게 잡는 이유는 무엇인가?

## 7. graphics engineer 면접 질문 1개와 답변

### Q. GPU Hash Grid는 무엇이며 particle simulation이나 spatial lookup에서 왜 사용하나요?

**A.** GPU Hash Grid는 3D 공간을 일정한 cell로 나누고, 각 particle이나 spatial element를 자신이 속한 cell bucket에 넣어 주변 검색 후보를 줄이는 자료구조입니다. Naive all-pairs 방식은 N개의 particle에 대해 O(N²) 비교가 필요하지만, Hash Grid를 사용하면 각 particle은 자신이 속한 cell과 주변 cell만 검사하면 됩니다. 따라서 neighbor search, collision broad-phase, SPH fluid, point cloud lookup, voxel brick lookup에서 매우 유용합니다.

구현 방식은 크게 atomic append 기반 hash table과 sort 기반 uniform grid가 있습니다. Atomic 방식은 단순하고 dynamic update가 쉽지만 contention과 bucket overflow 문제가 생길 수 있습니다. Sort 기반 방식은 `(cellKey, index)` pair를 정렬해 같은 cell의 element를 연속 memory에 배치하므로 query locality가 좋지만, 매 frame sort 비용이 듭니다. 핵심 trade-off는 grid build 비용, query coherence, memory layout, hash collision, atomic contention을 어떻게 균형 잡는가입니다.

## 8. 포트폴리오 / 커리어 연결

GPU Hash Grid for Spatial Lookup은 포트폴리오에서 다음 메시지를 만든다.

> “나는 particle / voxel / field block처럼 많은 spatial element를 GPU에서 효율적으로 조회하기 위해 spatial acceleration structure와 memory layout을 설계할 수 있다.”

네 배경과 연결하면 다음 표현이 좋다.

- SPH fluid나 particle simulation에서 neighbor search 최적화 이해
- CFD / VTK field block lookup에서 broad-phase spatial indexing 사고
- Sparse voxel / marching cubes active cell grouping에서 hash grid 활용 가능
- WebGPU / Vulkan compute shader에서 cell key, sort, range buffer 기반 lookup 구조로 확장 가능

면접에서는 다음처럼 말할 수 있다.

> “GPU Hash Grid는 공간을 cell로 나누고 cell key를 기반으로 candidate를 제한해 all-pairs 문제를 local neighborhood query로 바꾸는 구조입니다. 성능은 hash function보다 cell size, build 방식, memory layout, query coherence에 크게 좌우됩니다.”

## 9. 내일 이어서 볼 개념

**Prefix Sum / Scan for GPU Compaction**

GPU Hash Grid 다음에는 Prefix Sum / Scan을 보는 것이 자연스럽다. Sort 기반 grid, visible list compaction, active cell extraction, indirect draw command generation 모두에서 prefix sum은 GPU data pipeline을 구성하는 핵심 building block이다.

## 10. 참고 키워드

- GPU Hash Grid
- Spatial Hashing
- Uniform Grid
- Neighbor Search
- Broad-phase Collision
- SPH Fluid
- PBD / XPBD
- Particle Simulation
- Cell Key
- Atomic Append
- GPU Sort
- Cell Range Buffer
- Prefix Sum
- Morton Key
- Voxel Brick Lookup
- Field Block Lookup
- Point Cloud
- Collision Detection
- Scientific Visualization
- Compute Shader
