---
title: "Morton Order and GPU Memory Locality"
date: "2026-07-02"
category: "Graphics"
tags: ["GPU", "Rendering Pipeline", "Morton Order", "Z-order Curve", "Memory Locality", "Sparse Voxel", "Page Table", "Cache", "Scientific Visualization"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-07-02 - Morton Order and GPU Memory Locality

## 1. 오늘의 개념

**Morton Order**, 또는 **Z-order Curve**, 는 2D/3D spatial coordinate를 1D integer key로 변환할 때 x/y/z bit를 번갈아 interleave해서 공간적으로 가까운 data가 memory상에서도 비교적 가까이 배치되도록 만드는 ordering 방식이다.

일반적인 2D row-major layout은 다음처럼 저장된다.

```text
index = y * width + x
```

이 방식은 x 방향으로 연속 접근할 때는 좋지만, 2D/3D 공간에서 block, tile, voxel, page, cell을 hierarchical하게 접근할 때는 locality가 깨질 수 있다.

Morton Order는 좌표 bit를 섞어 다음처럼 하나의 key를 만든다.

```text
2D: x bits + y bits interleaving
3D: x bits + y bits + z bits interleaving
```

예를 들어 2D 좌표 `(x, y)`가 있을 때, x의 bit와 y의 bit를 번갈아 배치하면 spatially nearby coordinate가 비슷한 Morton code를 갖게 된다.

핵심 변화는 다음이다.

> 2D/3D 공간 데이터를 단순 행 단위로 저장하는 대신, 공간적 근접성이 1D memory order에도 어느 정도 유지되도록 배치한다.

이 개념은 sparse voxel, octree, page table, virtual texture tile, field block, BVH node, meshlet cluster, particle grid에서 매우 자주 등장한다.

## 2. 한 줄 핵심

**Morton Order는 2D/3D spatial coordinate의 bit를 interleave해 1D key로 만들고, 공간적으로 가까운 tile/voxel/block을 memory상에서도 가깝게 배치해 GPU cache locality를 개선하는 ordering 전략이다.**

## 3. 왜 중요한가

GPU 성능은 ALU 연산만으로 결정되지 않는다. 많은 rendering / simulation / visualization workload는 memory access pattern에 의해 크게 좌우된다.

특히 다음 구조에서는 data가 공간적으로 인접한 순서로 접근된다.

- voxel brick traversal
- sparse voxel octree node lookup
- virtual texture page lookup
- CFD field block sampling
- particle spatial grid
- meshlet cluster culling
- volume ray marching
- page table access

만약 memory layout이 spatial locality를 반영하지 않으면, GPU thread들이 서로 멀리 떨어진 memory를 읽게 되고 cache 효율이 낮아진다. Morton Order는 2D/3D 공간의 locality를 1D buffer layout에 최대한 보존하려는 실용적인 방식이다.

Graphics engineer 관점에서 Morton Order는 단순 bit manipulation이 아니라, **spatial data structure와 GPU memory hierarchy를 연결하는 layout design**이다.

## 4. 구현 관점

### 4.1 Row-major layout의 한계

2D texture나 grid를 row-major로 저장하면 x 방향 접근에는 강하다.

```cpp
uint index = y * width + x;
```

하지만 2D tile 단위나 quad-tree / mip / block traversal에서는 이웃 block이 memory상 멀어질 수 있다. 3D에서는 문제가 더 커진다.

```cpp
uint index = z * width * height + y * width + x;
```

x 방향은 연속이지만 y/z 방향 이웃은 큰 stride를 가진다. Volume ray marching이나 voxel neighborhood sampling에서는 y/z 이웃 접근이 자주 발생하기 때문에 cache locality가 나빠질 수 있다.

### 4.2 Morton code 생성

Morton code는 좌표 bit를 interleave해서 만든다.

2D 예시는 다음과 같다.

```text
x = x2 x1 x0
y = y2 y1 y0

Morton = y2 x2 y1 x1 y0 x0
```

3D에서는 x/y/z bit를 번갈아 섞는다.

```text
Morton = z2 y2 x2 z1 y1 x1 z0 y0 x0
```

실제 구현에서는 bit spread 함수를 사용한다.

```cpp
uint Part1By1(uint x)
{
    x &= 0x0000ffff;
    x = (x | (x << 8)) & 0x00FF00FF;
    x = (x | (x << 4)) & 0x0F0F0F0F;
    x = (x | (x << 2)) & 0x33333333;
    x = (x | (x << 1)) & 0x55555555;
    return x;
}

uint Morton2D(uint x, uint y)
{
    return Part1By1(x) | (Part1By1(y) << 1);
}
```

이 코드는 x와 y bit 사이에 빈 bit를 만들고 interleave한다. 3D는 `Part1By2` 형태로 두 칸씩 bit를 벌린다.

### 4.3 Morton Order와 cache locality

Morton Order의 장점은 작은 공간 영역이 1D key 공간에서도 비교적 가까운 range에 모인다는 점이다.

예를 들어 8x8 tile grid에서 2x2, 4x4 block은 Morton order에서 일정 범위로 묶이는 경향이 있다. 이는 quadtree/octree traversal과 잘 맞는다.

장점은 다음과 같다.

- 2D/3D neighborhood access의 locality 향상
- mip / LOD / hierarchical traversal과 잘 맞음
- tile / brick / page indexing에 유리
- GPU cache line 활용 개선 가능
- spatial sorting key로 사용 가능

하지만 완벽한 locality를 보장하지는 않는다. Morton curve는 특정 경계에서 jump가 발생한다. 그래도 row-major보다 multidimensional locality를 더 균형 있게 보존하는 경우가 많다.

### 4.4 Sparse voxel / octree와 연결

Sparse voxel octree에서는 Morton code가 매우 자연스럽다. Octree의 각 level에서 child index는 x/y/z bit 조합으로 결정된다.

```text
childIndex = (xBit) | (yBit << 1) | (zBit << 2)
```

이것은 3D Morton code의 level별 bit와 같은 구조다. 따라서 Morton code는 octree path를 표현하는 key로도 볼 수 있다.

Sparse voxel renderer에서는 다음에 사용할 수 있다.

- voxel coordinate → Morton key
- brick coordinate → Morton key
- Morton key sort로 spatial grouping
- page table lookup key
- sparse hash table key
- LOD level별 node ordering

Morton ordering으로 brick을 정렬하면 ray marching 중 인접 brick 접근이나 visible brick list traversal에서 cache locality가 좋아질 수 있다.

### 4.5 Virtual texture / page table과 연결

Virtual texturing에서도 tile coordinate를 Morton order로 저장할 수 있다. Page table이나 physical page pool에서 인접 virtual page를 인접 memory에 배치하면 screen-space sampling locality가 좋아질 수 있다.

특히 feedback pass에서 필요한 page id를 모은 뒤 Morton key로 sorting하면 streaming request를 spatially coherent하게 만들 수 있다.

```text
visible page list → Morton sort → spatially grouped upload/request
```

이 방식은 disk / CPU memory / GPU upload에서도 도움이 된다. 큰 data streaming에서는 request order도 성능에 영향을 준다.

### 4.6 CFD / scientific field block과 연결

CFD field block은 3D grid나 block-structured grid로 구성되는 경우가 많다. Block id를 단순 z-major 또는 row-major로 배치하면 camera ray나 slice가 지나가는 block들이 memory상 흩어질 수 있다.

Morton key를 사용하면 3D 공간에서 가까운 field block을 memory상 가까이 둘 수 있다.

응용 예시는 다음과 같다.

- block coordinate → Morton key
- visible block list Morton sort
- volume brick traversal ordering
- streamline integration 중 field block cache 향상
- page table / brick pool layout 개선
- multi-resolution block hierarchy 구성

Scientific visualization에서는 data correctness뿐 아니라 interaction latency도 중요하므로, field block layout과 streaming order가 체감 성능에 직접 영향을 준다.

### 4.7 GPU-driven rendering에서의 sorting key

GPU-driven renderer에서는 visible object, meshlet, brick, page list를 compact한 뒤 정렬하는 경우가 있다. 이때 Morton key는 spatial sorting key로 사용할 수 있다.

예를 들어 visible meshlet list를 Morton order로 정렬하면 인접 화면/공간 영역의 meshlet이 비슷한 material/texture/field data를 접근할 가능성이 커진다.

다만 sorting 자체도 비용이 있다. 따라서 매 frame 전체 sorting을 할지, coarse binning만 할지, CPU preprocessing으로 정렬해 둘지 판단해야 한다.

실무적으로는 다음 선택지가 있다.

- offline Morton sort
- per-frame visible list coarse binning
- tile/cluster 단위 grouping
- material sort와 Morton sort의 혼합
- LOD level별 Morton ordering

### 4.8 Morton Order의 한계

Morton Order는 좋은 기본값이지만 모든 경우에 최적은 아니다.

한계는 다음과 같다.

- 특정 경계에서 locality jump가 발생한다.
- Hilbert curve보다 locality 보존이 약할 수 있다.
- bit manipulation 비용이 추가된다.
- non-power-of-two grid에서 padding이나 mapping이 필요하다.
- material / temporal locality와 spatial locality가 충돌할 수 있다.
- GPU wave access pattern이 Morton order와 항상 맞지는 않는다.

즉 Morton Order는 “항상 빠른 magic layout”이 아니라, spatial hierarchy와 cache locality를 개선하기 위한 practical compromise다.

## 5. 내 관심 분야와 연결

### CFD / scientific visualization

CFD visualization에서는 block, cell, voxel, particle, streamline segment가 모두 spatial data다. Morton order를 사용하면 block-based field sampling이나 volume rendering에서 인접 block 접근이 더 안정적일 수 있다.

특히 다음에 유용하다.

- VTK/VTU/VTM block ordering
- volume brick layout
- scalar/vector field block cache
- streamline integration의 field lookup
- slice / clip plane에서 visible block traversal
- CFD field streaming request ordering

네가 다루는 대용량 후처리와 scientific visualization에서는 rendering algorithm 자체만큼 data layout이 중요하다.

### Sparse voxel / octree / NanoVDB

Sparse voxel과 octree에서는 Morton code가 거의 기본 언어처럼 등장한다. Octree child path, brick coordinate, sparse page table key, brick pool ordering 모두 Morton structure와 연결된다.

NanoVDB 같은 sparse volume 구조도 hierarchical index와 cache-friendly layout이 핵심이다. Morton order를 이해하면 sparse volume traversal과 memory layout을 훨씬 잘 해석할 수 있다.

### Game engine architecture

Game engine에서는 Morton order가 다음 곳에 등장한다.

- clustered light grid
- virtual texture page id
- voxel GI brick layout
- terrain tile ordering
- BVH / spatial index sorting
- particle spatial hash
- meshlet spatial grouping

면접에서는 Morton order를 단순히 “Z-order curve”라고 정의하기보다, “spatial locality를 1D memory layout에 반영하는 방식”이라고 설명하는 것이 좋다.

## 6. 머릿속에 남길 질문 3개

1. Row-major layout이 3D volume이나 voxel brick traversal에서 cache locality를 깨뜨릴 수 있는 이유는 무엇인가?
2. Morton code가 octree child traversal과 자연스럽게 연결되는 이유는 무엇인가?
3. CFD field block이나 sparse voxel brick을 Morton order로 정렬하면 어떤 access pattern에서 이득을 얻을 수 있는가?

## 7. graphics engineer 면접 질문 1개와 답변

### Q. Morton Order는 무엇이며 GPU rendering에서 왜 사용하나요?

**A.** Morton Order 또는 Z-order Curve는 2D/3D 좌표의 bit를 interleave해서 하나의 1D key로 만드는 방식입니다. 예를 들어 3D 좌표의 x/y/z bit를 번갈아 섞으면 공간적으로 가까운 voxel, tile, block이 비슷한 Morton code를 갖게 됩니다. 이 ordering을 사용하면 2D/3D spatial data를 1D buffer에 저장할 때 공간 locality를 어느 정도 유지할 수 있습니다.

GPU rendering에서는 sparse voxel brick, virtual texture page, CFD field block, meshlet cluster, particle grid 같은 spatial data를 cache-friendly하게 배치하는 데 사용됩니다. 특히 octree child index는 x/y/z bit 조합으로 결정되므로 Morton code와 자연스럽게 연결됩니다. 장점은 neighborhood access와 hierarchical traversal의 memory locality를 개선할 수 있다는 점입니다. 단점은 모든 access pattern에 최적인 것은 아니며, 특정 경계에서 locality jump가 있고, material sorting이나 temporal locality와 충돌할 수 있다는 점입니다.

## 8. 포트폴리오 / 커리어 연결

Morton Order and GPU Memory Locality는 포트폴리오에서 다음 메시지를 만든다.

> “나는 대용량 graphics / visualization data에서 알고리즘뿐 아니라 memory layout과 cache locality가 성능을 결정한다는 점을 이해한다.”

네 배경과 연결하면 다음 표현이 좋다.

- CFD / VTK block data를 Morton order로 정렬해 spatial traversal locality를 개선하는 사고
- Sparse voxel / octree renderer에서 Morton key 기반 brick indexing 이해
- WebGPU / Vulkan storage buffer에서 3D grid를 1D buffer로 배치하는 방식 설명 가능
- Page table, virtual texture, voxel brick pool에서 spatial key와 physical layout을 연결하는 능력

면접에서는 다음처럼 말할 수 있다.

> “Morton Order는 2D/3D 좌표를 bit interleaving으로 1D key로 변환해 spatial locality를 memory layout에 반영하는 방식입니다. Sparse voxel, virtual texture page, field block, meshlet 같은 data를 cache-friendly하게 정렬하는 데 사용할 수 있습니다.”

## 9. 내일 이어서 볼 개념

**GPU Hash Grid for Spatial Lookup**

Morton Order 다음에는 GPU Hash Grid를 보는 것이 자연스럽다. Morton key가 spatial coordinate를 locality-friendly key로 만드는 방식이라면, hash grid는 particle, voxel, nearest-neighbor, collision, field sampling에서 spatial lookup을 빠르게 만들기 위한 GPU data structure다.

## 10. 참고 키워드

- Morton Order
- Z-order Curve
- Bit Interleaving
- Spatial Locality
- GPU Cache
- Memory Coalescing
- Sparse Voxel
- Octree
- Voxel Brick
- Page Table
- Virtual Texture Page
- Field Block
- CFD Visualization
- Morton Key Sorting
- Brick Pool
- Volume Rendering
- NanoVDB
- GPU-driven Rendering
- Hilbert Curve
- Spatial Hashing
