---
title: "GPU Radix Sort for Spatial Data"
date: "2026-07-06"
category: "Graphics"
tags: ["GPU", "Compute Shader", "Radix Sort", "Spatial Data", "Morton Key", "Hash Grid", "Particles", "Voxel", "Scientific Visualization"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-07-06 - GPU Radix Sort for Spatial Data

## 1. 오늘의 개념

**GPU Radix Sort for Spatial Data**는 particle, voxel brick, field block, meshlet, virtual texture page 같은 spatial element를 GPU에서 integer key 기준으로 정렬하는 data-parallel sorting 기법이다.

GPU rendering / simulation에서는 다음과 같은 key가 자주 등장한다.

- particle의 grid cell key
- voxel brick의 Morton key
- virtual texture page id
- meshlet의 cluster key
- material / pipeline sorting key
- light tile / cluster key
- visible object depth key

이 key를 정렬하면 같은 cell, 같은 brick, 가까운 공간, 같은 material을 가진 element가 buffer 안에서 서로 가까워진다. 그러면 후속 pass에서 range query, neighbor search, compaction, indirect draw, page streaming을 훨씬 쉽게 구성할 수 있다.

핵심 변화는 다음이다.

> 흩어진 GPU data를 key 기준으로 정렬해, 다음 pass가 연속 memory range를 순회할 수 있는 구조로 바꾼다.

Radix Sort는 comparison sort가 아니라 integer key의 bit 또는 digit을 여러 pass로 나누어 정렬한다. GPU에서는 비교 기반 sort보다 key 분해, histogram, prefix sum, scatter로 구성되는 radix sort가 data-parallel 구조에 잘 맞는다.

## 2. 한 줄 핵심

**GPU Radix Sort는 spatial key나 material key를 bit 단위로 정렬해, particle / voxel / field block / meshlet data를 coherent한 memory range로 재배치하는 GPU data pipeline의 핵심 building block이다.**

## 3. 왜 중요한가

GPU Hash Grid, Morton Order, Prefix Sum을 이해했다면 다음 단계는 sort다. 많은 GPU 알고리즘은 단순히 flag를 compact하는 것만으로는 충분하지 않다. 같은 cell 또는 같은 key를 가진 element들을 연속된 range로 모아야 한다.

예를 들어 particle simulation에서는 각 particle이 cell key를 가진다. 이들을 cell key 기준으로 정렬하면 같은 cell의 particle이 memory상 연속된다. 이후 cellStart / cellEnd range를 만들면 neighbor query가 빠르고 예측 가능해진다.

Radix Sort가 중요한 이유는 다음과 같다.

- GPU Hash Grid의 sorted grid build에 필요하다.
- Morton key sorting으로 spatial locality를 높일 수 있다.
- visible meshlet / brick list를 coherent하게 만들 수 있다.
- material / pipeline key sort로 state change를 줄일 수 있다.
- virtual texture page request를 spatially grouped streaming으로 만들 수 있다.
- CFD field block이나 voxel brick을 range 기반으로 처리할 수 있다.

Graphics engineer 관점에서 Radix Sort는 단순 알고리즘 문제가 아니라, **GPU에서 random-looking data를 coherent data stream으로 바꾸는 reordering primitive**다.

## 4. 구현 관점

### 4.1 Comparison sort와 Radix sort의 차이

CPU에서 흔히 쓰는 sort는 두 원소를 비교하는 comparison sort다.

```text
compare(a, b) → a < b ?
```

하지만 GPU에서는 수많은 thread가 동시에 동일한 패턴으로 작업하기 좋다. Radix Sort는 key의 일부 bit를 보고 bucket을 나누고, prefix sum으로 각 bucket의 output offset을 계산한 뒤 scatter한다.

예를 들어 32-bit key를 4-bit씩 처리하면 8번의 pass가 필요하다.

```text
pass 0: bit 0~3 기준 정렬
pass 1: bit 4~7 기준 정렬
...
pass 7: bit 28~31 기준 정렬
```

각 pass는 다음 구조를 가진다.

1. digit 추출
2. digit별 histogram 계산
3. histogram prefix sum
4. 각 element의 output offset 계산
5. scatter
6. ping-pong buffer swap

### 4.2 Key-value pair 정렬

Graphics에서는 key만 정렬하는 경우보다 key-value pair를 정렬하는 경우가 많다.

```cpp
struct KeyValue
{
    uint key;
    uint value; // particle index, brick index, meshlet index 등
};
```

정렬 후에는 value를 통해 원본 data를 참조한다.

```text
(cellKey, particleIndex)
(mortonKey, brickIndex)
(materialKey, drawIndex)
```

이 방식은 원본 particle/brick/object buffer를 직접 재배치하지 않고, index list만 정렬할 수 있다는 장점이 있다. 큰 attribute buffer를 매 frame 움직이지 않아도 된다.

### 4.3 Histogram pass

Radix Sort의 첫 단계는 각 digit 값이 몇 개 있는지 세는 것이다. 예를 들어 4-bit digit이면 bucket은 16개다.

```text
digit = (key >> shift) & 0xF
histogram[digit]++
```

GPU에서는 workgroup별 local histogram을 만들고, 이후 global histogram으로 합치는 방식이 자주 사용된다. Atomic을 사용할 수도 있지만 contention을 줄이기 위해 shared memory histogram을 먼저 만들기도 한다.

Histogram은 다음 prefix sum 단계의 입력이 된다.

### 4.4 Prefix sum과 scatter

각 bucket의 시작 위치는 histogram에 prefix sum을 적용해 얻는다.

```text
bucketCounts: [3, 5, 2, 4]
bucketOffsets: [0, 3, 8, 10]
```

이제 각 element는 자신의 digit bucket 안에서 몇 번째인지 local rank를 계산하고, 최종 output index를 얻는다.

```text
outputIndex = bucketOffset[digit] + localRankWithinBucket
```

이 구조가 바로 Prefix Sum / Scan과 Radix Sort가 강하게 연결되는 이유다.

### 4.5 Stable sort가 중요한 이유

Radix Sort는 보통 least significant digit부터 처리할 때 stable sort가 필요하다. 이전 pass에서 정렬된 lower bits의 순서를 유지해야 다음 higher bits를 처리한 뒤 전체 key order가 올바르게 된다.

GPU에서 scatter pass가 deterministic하고 stable하게 작동하도록 local rank를 정확히 계산해야 한다.

Stable property는 spatial data에서도 중요하다. 예를 들어 같은 cell key 안에서 이전 frame order나 material order를 어느 정도 유지하면 후속 access가 더 안정적일 수 있다.

### 4.6 Hash Grid와 연결

GPU Hash Grid의 sorted grid 방식은 Radix Sort의 대표 응용이다.

흐름은 다음과 같다.

```text
particle position → cell coordinate
cell coordinate → cellKey 또는 Morton key
(cellKey, particleIndex) 생성
cellKey 기준 radix sort
cell boundary detection
cellStart / cellEnd range 생성
neighbor query
```

정렬된 particle index buffer는 같은 cell의 particle을 연속 memory에 배치한다. Neighbor query는 주변 cell의 range만 순회하면 된다.

Atomic append 기반 hash grid보다 build cost는 더 크지만, query locality와 range traversal이 좋아지는 경우가 많다.

### 4.7 Morton key sorting과 spatial locality

Morton key를 radix sort하면 spatial locality를 가진 order가 만들어진다.

```text
3D coordinate → Morton key → radix sort → spatially coherent list
```

이 방식은 다음에 유용하다.

- voxel brick list 정렬
- visible meshlet list 정렬
- CFD field block streaming order
- virtual texture page request grouping
- particle grid grouping
- sparse voxel node ordering

정렬된 결과는 단순히 보기 좋은 순서가 아니라, 다음 GPU pass에서 cache locality와 memory coherence를 높이는 기반이 된다.

### 4.8 Material / pipeline sorting과의 차이

Spatial sorting과 material sorting은 목표가 다르다.

- Spatial sorting: memory locality, neighbor query, brick/page traversal 개선
- Material sorting: texture/material/pipeline state locality 개선
- Depth sorting: transparency, overdraw, front-to-back culling 개선

실무에서는 하나의 key에 여러 정보를 pack하기도 한다.

```text
sortKey = (pipelineId << 24) | (materialId << 16) | mortonOrDepthKey
```

하지만 어떤 locality를 우선할지 결정해야 한다. Spatial locality와 material locality가 항상 같은 방향은 아니다. 예를 들어 가까운 voxel brick들이 서로 다른 material을 가질 수 있고, 같은 material object들이 공간적으로 멀리 떨어져 있을 수 있다.

### 4.9 성능 trade-off

GPU Radix Sort는 강력하지만 공짜가 아니다.

비용 요소는 다음과 같다.

- 여러 pass의 global memory read/write
- histogram 생성 비용
- prefix sum 비용
- scatter 비용
- ping-pong buffer memory
- synchronization / barrier
- key-value buffer bandwidth

따라서 매 frame 전체 sort가 필요한지 판단해야 한다.

대안은 다음과 같다.

- frame마다 sort하지 않고 일정 주기마다 sort
- coarse binning만 수행
- CPU/offline preprocessing으로 static data 정렬
- visible subset만 sort
- material key와 spatial key를 분리해 필요할 때만 sort

즉 Radix Sort는 random data를 coherent stream으로 바꾸는 강력한 도구지만, sorting cost보다 후속 pass의 절약 비용이 커야 한다.

## 5. 내 관심 분야와 연결

### CFD / scientific visualization

CFD field block, streamline seed, particle trace, iso-surface active cell은 공간적으로 분포한다. Morton key나 block id를 기준으로 radix sort하면 다음이 가능하다.

- visible block list를 spatially coherent하게 순회
- field block streaming request를 근접 순서로 묶기
- slice / clip plane과 교차하는 cell 후보 정렬
- active iso-surface cell을 output triangle generation 전에 group화
- particle trace lookup locality 개선

Scientific visualization에서는 data가 크기 때문에 단순 알고리즘보다 memory order와 streaming order가 체감 성능에 크게 영향을 준다.

### Sparse voxel / octree / NanoVDB

Sparse voxel renderer에서는 brick coordinate를 Morton key로 변환하고 radix sort해 brick pool, visible brick list, active cell list를 구성할 수 있다.

예시:

```text
active voxel/brick → Morton key 생성 → radix sort → contiguous brick range → rendering / marching cubes / ray marching
```

NanoVDB나 sparse volume hierarchy를 이해할 때도 spatial key ordering과 cache-friendly traversal이 중요하다.

### Game engine architecture

Game engine에서는 Radix Sort가 다음에 사용된다.

- GPU particle sorting
- transparency depth sorting
- hash grid build
- clustered light / decal binning
- draw call sorting
- material/pipeline sorting
- virtual texture page request grouping
- meshlet / cluster visible list ordering

면접에서는 Radix Sort를 단순히 “빠른 sort”라고 말하기보다, “GPU에서 key-value pair를 정렬해 후속 pass가 range 기반으로 처리하게 만드는 data pipeline primitive”라고 말하는 것이 좋다.

## 6. 머릿속에 남길 질문 3개

1. GPU Hash Grid에서 `(cellKey, particleIndex)` pair를 radix sort하면 neighbor query가 왜 쉬워지는가?
2. Radix Sort가 Prefix Sum / Scan과 강하게 연결되는 이유는 무엇인가?
3. Spatial key sorting과 material key sorting이 서로 충돌할 수 있는 상황은 무엇인가?

## 7. graphics engineer 면접 질문 1개와 답변

### Q. GPU Radix Sort는 rendering이나 simulation에서 왜 사용되며, 어떤 trade-off가 있나요?

**A.** GPU Radix Sort는 integer key를 bit 또는 digit 단위로 여러 pass에 걸쳐 정렬하는 data-parallel sorting 방식입니다. Rendering과 simulation에서는 particle의 cell key, voxel brick의 Morton key, material sorting key, visible meshlet key 같은 key-value pair를 정렬하는 데 사용됩니다. 예를 들어 GPU Hash Grid에서는 `(cellKey, particleIndex)` pair를 cellKey 기준으로 정렬하면 같은 cell의 particle이 연속 memory range에 모이므로 neighbor query가 쉬워집니다.

Radix Sort는 histogram, prefix sum, scatter로 구성되므로 GPU parallel execution에 잘 맞습니다. 장점은 spatial data를 coherent한 memory order로 재배치하고, range generation이나 streaming request grouping을 쉽게 만든다는 점입니다. 단점은 여러 pass의 global memory read/write, histogram, scan, scatter 비용이 크다는 것입니다. 따라서 매 frame 전체 sort가 필요한지, visible subset만 sort할지, spatial locality와 material locality 중 무엇을 우선할지 판단해야 합니다.

## 8. 포트폴리오 / 커리어 연결

GPU Radix Sort for Spatial Data는 포트폴리오에서 다음 메시지를 만든다.

> “나는 GPU에서 random한 spatial data를 key-value 정렬로 coherent한 memory stream으로 바꾸고, 후속 rendering/simulation pass가 range 기반으로 처리하도록 data pipeline을 설계할 수 있다.”

네 배경과 연결하면 다음 표현이 좋다.

- SPH / particle simulation에서 cell key sort 기반 neighbor search 구조 이해
- CFD field block이나 iso-surface active cell을 Morton key로 정렬해 locality 개선 가능
- Sparse voxel / octree brick list를 spatially coherent하게 정렬하는 사고
- GPU-driven renderer에서 visible meshlet/object list를 material or spatial key로 정렬하는 판단력
- WebGPU / Vulkan compute pipeline에서 histogram / scan / scatter 기반 sort 구조로 확장 가능

면접에서는 다음처럼 말할 수 있다.

> “GPU Radix Sort는 key-value pair를 정렬해 같은 cell, 같은 material, 가까운 spatial region을 연속 memory range로 만드는 도구입니다. Hash grid, Morton ordering, visible list, virtual texture page request grouping에서 후속 pass의 locality와 range traversal을 개선하는 데 사용됩니다.”

## 9. 내일 이어서 볼 개념

**GPU Work Queue and Persistent Threads**

Radix Sort까지 이해하면 GPU data pipeline에서 list를 만들고 정렬하는 흐름을 볼 수 있다. 다음에는 dynamic workload를 GPU 내부 queue로 관리하고, persistent threads가 작업을 가져가 처리하는 구조를 보는 것이 자연스럽다.

## 10. 참고 키워드

- GPU Radix Sort
- Key-Value Sort
- Spatial Key
- Morton Key
- Cell Key
- Histogram
- Prefix Sum
- Scatter
- Ping-Pong Buffer
- Sort-Based Hash Grid
- Particle Binning
- Voxel Brick Sorting
- Field Block Sorting
- Visible List Ordering
- Material Sorting
- Pipeline Sorting
- GPU-driven Rendering
- Compute Shader
- Scientific Visualization
