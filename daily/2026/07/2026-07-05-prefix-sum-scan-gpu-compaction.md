---
title: "Prefix Sum / Scan for GPU Compaction"
date: "2026-07-05"
category: "Graphics"
tags: ["GPU", "Compute Shader", "Prefix Sum", "Scan", "Compaction", "GPU-Driven Rendering", "Hash Grid", "Indirect Draw", "Scientific Visualization"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-07-05 - Prefix Sum / Scan for GPU Compaction

## 1. 오늘의 개념

**Prefix Sum**, 또는 **Scan**, 은 배열의 각 원소 위치에 대해 그 앞에 있는 값들의 누적합을 계산하는 병렬 알고리즘이다. GPU rendering / simulation / visualization에서는 단순 수학 연산보다 더 중요한 의미가 있다.

Prefix Sum은 GPU에서 다음 질문에 답한다.

> 조건을 만족한 element들을 output buffer의 어디에 써야 하는가?

예를 들어 100만 개의 voxel cell 중 active cell만 모으고 싶다고 하자. 각 cell은 active인지 아닌지만 안다.

```text
activeFlag = [0, 1, 0, 1, 1, 0, 0, 1]
```

이 flag에 exclusive scan을 적용하면 각 active element가 output buffer에 기록될 위치를 알 수 있다.

```text
exclusiveScan = [0, 0, 1, 1, 2, 3, 3, 3]
```

이제 active인 element만 다음 위치에 쓸 수 있다.

```text
index 1 → output[0]
index 3 → output[1]
index 4 → output[2]
index 7 → output[3]
```

핵심 변화는 다음이다.

> 흩어진 sparse result를 GPU 안에서 compact한 dense list로 변환한다.

이 구조는 visible list, active cell list, particle bucket range, hash grid build, stream compaction, indirect draw command generation, marching cubes output allocation에서 반복적으로 등장한다.

## 2. 한 줄 핵심

**Prefix Sum / Scan은 GPU에서 boolean flag나 count 배열을 output offset으로 변환해, visible object / active cell / particle / voxel 결과를 compact buffer로 만드는 핵심 parallel primitive다.**

## 3. 왜 중요한가

GPU는 많은 thread가 동시에 실행된다. 문제는 각 thread가 조건을 만족했을 때 결과를 어디에 써야 하는지다.

가장 단순한 방식은 atomic counter를 사용하는 것이다.

```cpp
uint dst = atomicAdd(counter, 1);
output[dst] = item;
```

이 방식은 구현이 쉽지만 많은 thread가 동시에 counter를 증가시키면 contention이 생긴다. Output 순서도 불안정할 수 있고, 대규모 compaction에서는 병목이 될 수 있다.

Prefix Sum은 각 thread가 쓸 위치를 deterministic하게 계산하게 해준다. GPU data pipeline에서 다음 작업의 기반이 된다.

- Frustum / Hi-Z culling 결과 compact
- GPU Hash Grid cell range 생성
- Marching Cubes active cell extraction
- Particle alive/dead compaction
- Indirect draw command count 생성
- Tile/cluster light list offset 계산
- Sparse voxel visible brick list 생성
- CFD field block visible list 생성

Graphics engineer 관점에서 scan은 단순 알고리즘 문제가 아니라, **GPU에서 가변 길이 결과를 안정적인 buffer layout으로 바꾸는 핵심 building block**이다.

## 4. 구현 관점

### 4.1 Inclusive scan과 exclusive scan

Scan에는 두 가지 대표 형태가 있다.

Inclusive scan:

```text
input:     [1, 0, 1, 1]
inclusive: [1, 1, 2, 3]
```

Exclusive scan:

```text
input:     [1, 0, 1, 1]
exclusive: [0, 1, 1, 2]
```

Compaction에는 보통 exclusive scan이 편하다. 현재 element가 active라면 `exclusiveScan[i]`가 output index가 된다.

```cpp
if (active[i])
{
    uint dst = exclusiveScan[i];
    output[dst] = input[i];
}
```

총 active count는 마지막 scan 값과 마지막 flag를 더하면 얻을 수 있다.

```cpp
activeCount = exclusiveScan[N - 1] + activeFlag[N - 1];
```

### 4.2 Stream compaction 기본 흐름

GPU compaction은 보통 3단계로 구성된다.

1. Flag generation
2. Prefix sum / scan
3. Scatter

예를 들어 visible object list를 만든다면 다음과 같다.

```text
Object bounds culling → visibleFlag[i]
visibleFlag scan → outputOffset[i]
visibleFlag가 1인 object만 visibleList[outputOffset[i]]에 기록
```

이 구조는 culling 결과를 다음 pass에서 사용할 수 있는 dense list로 만든다.

### 4.3 Workgroup-level scan

작은 배열은 하나의 workgroup 안에서 shared memory를 사용해 scan할 수 있다.

개념적으로는 다음 흐름이다.

```text
각 thread가 input을 shared memory에 로드
stride 1, 2, 4, 8 ... 로 partial sum 계산
barrier로 동기화
exclusive result 출력
```

GPU에서는 workgroup barrier가 필요하다. 같은 workgroup 내부에서는 빠르지만, 전체 배열이 workgroup보다 크면 multi-pass scan이 필요하다.

### 4.4 Large array scan

큰 배열을 scan하려면 보통 hierarchical scan을 사용한다.

1. 각 block/workgroup별 local scan 수행
2. 각 block의 total sum을 blockSum buffer에 저장
3. blockSum buffer를 다시 scan
4. 각 block 결과에 scanned block offset을 더함

흐름은 다음과 같다.

```text
input → local scan per block → block sums
block sums → scan → block offsets
local result + block offset → global scan result
```

이 구조는 GPU parallel prefix sum의 기본이다. 구현 복잡도는 있지만 대규모 visible list나 active cell extraction에는 필수다.

### 4.5 Atomic append와 scan compaction 비교

Atomic append:

장점:

- 구현이 단순하다.
- 한 pass로 output을 만들 수 있다.
- 결과 수가 적을 때 빠를 수 있다.

단점:

- atomic contention이 생긴다.
- output order가 불안정할 수 있다.
- 대규모 결과에서 병목이 될 수 있다.

Scan compaction:

장점:

- output index가 deterministic하다.
- 병렬성이 좋다.
- large data에 안정적이다.
- cell range, draw count, offset 계산에 확장 가능하다.

단점:

- scan pass가 추가된다.
- memory bandwidth를 많이 사용한다.
- 구현이 복잡하다.

즉 작은 데이터나 sparse hit에는 atomic이 충분할 수 있고, large-scale culling / simulation / visualization에는 scan 기반 compaction이 더 안정적인 경우가 많다.

### 4.6 GPU Hash Grid와 연결

이전 노트의 GPU Hash Grid에서 sort 기반 uniform grid를 만들 때 scan이 등장한다.

흐름은 다음과 같다.

```text
particle → cellKey 생성
cellKey 기준 sort
cell boundary flag 생성
boundary flag scan
cellStart / cellEnd range 생성
```

또는 cell별 count를 먼저 계산한 뒤 prefix sum으로 offset을 만들 수 있다.

```text
cellCounts[cell] 계산
cellCounts scan → cellOffsets
particle scatter → sorted/cell-grouped buffer
```

즉 Hash Grid에서 scan은 “각 cell이 output buffer의 어느 range를 소유하는가”를 결정한다.

### 4.7 Marching Cubes와 연결

Marching Cubes에서도 scan은 핵심이다. 각 voxel cell이 생성할 triangle 수는 0개에서 여러 개까지 다르다. GPU에서 output triangle buffer를 만들려면 각 cell이 쓸 위치를 알아야 한다.

흐름은 다음과 같다.

1. 각 cell의 case index 계산
2. 각 cell이 생성할 triangle count 계산
3. triangleCount 배열 scan
4. 각 cell이 output triangle buffer의 offset을 얻음
5. triangle generation pass에서 해당 offset에 triangle 기록

```text
triangleCount[i] → prefix sum → triangleOffset[i]
```

이 구조가 없으면 variable-length geometry output을 안정적으로 만들기 어렵다.

### 4.8 Indirect draw와 연결

GPU-driven rendering에서는 culling 결과를 compact list로 만들고, 그 count를 indirect draw command에 넣어야 한다.

예를 들어 visible meshlet list는 다음처럼 생성된다.

```text
meshlet visibleFlag → scan → visibleMeshletList
activeCount → indirectDrawCommand.instanceCount 또는 dispatch count
```

GPU가 visible list와 draw command를 모두 만들 수 있으면 CPU readback 없이 rendering이 이어진다.

이것이 GPU-driven renderer에서 prefix sum이 중요한 이유다.

## 5. 내 관심 분야와 연결

### CFD / scientific visualization

CFD visualization에서는 active subset을 뽑는 일이 많다.

- iso-surface active cell extraction
- threshold 조건을 만족하는 cell list
- visible block list
- streamline seed compaction
- particle trace alive list
- slice plane과 교차하는 cell list

이때 flag → scan → scatter 구조를 사용하면 대용량 field에서 필요한 부분만 compact하게 모아 후속 rendering이나 analysis pass에 넘길 수 있다.

### Sparse voxel / octree / NanoVDB

Sparse voxel renderer에서는 visible brick, active brick, non-empty node, marching cubes 대상 cell을 compact list로 만드는 일이 많다.

예시는 다음과 같다.

```text
brick visible flag → scan → visibleBrickList
active voxel flag → scan → activeVoxelList
surface cell triangle count → scan → triangle offsets
```

Sparse data structure는 “없는 것을 건너뛰는 구조”이므로 scan compaction과 자연스럽게 연결된다.

### Game engine architecture

Game engine에서는 scan이 보이지 않는 곳에서 계속 쓰인다.

- GPU culling visible list
- particle alive compaction
- cluster light list offsets
- tiled shading range generation
- indirect draw command generation
- meshlet list compaction

면접에서는 Prefix Sum을 단순 알고리즘으로 말하기보다, GPU data pipeline에서 variable-length output을 다루기 위한 핵심 primitive라고 설명하는 것이 좋다.

## 6. 머릿속에 남길 질문 3개

1. GPU에서 activeFlag 배열에 exclusive scan을 적용하면 output index를 어떻게 얻을 수 있는가?
2. Atomic append와 scan-based compaction은 각각 어떤 상황에서 유리한가?
3. Marching Cubes나 GPU-driven visible list generation에서 prefix sum이 없으면 어떤 문제가 생기는가?

## 7. graphics engineer 면접 질문 1개와 답변

### Q. Prefix Sum / Scan은 GPU rendering이나 simulation에서 왜 중요한가요?

**A.** Prefix Sum 또는 Scan은 배열의 누적합을 병렬로 계산하는 알고리즘입니다. GPU rendering과 simulation에서는 단순 합산보다 stream compaction과 variable-length output allocation에 중요합니다. 예를 들어 각 object가 visible인지 나타내는 flag 배열이 있을 때 exclusive scan을 적용하면, visible object가 output visible list의 몇 번째 위치에 써야 하는지 알 수 있습니다.

이 구조는 GPU culling, particle alive compaction, marching cubes triangle generation, hash grid cell range 생성, tile light list offset 계산, indirect draw command generation에 사용됩니다. Atomic append도 비슷한 역할을 할 수 있지만, 많은 thread가 같은 counter를 증가시키면 contention이 생길 수 있습니다. Scan-based compaction은 pass가 더 필요하지만 deterministic output index를 만들고 large-scale data에서 안정적인 병렬성을 제공합니다.

## 8. 포트폴리오 / 커리어 연결

Prefix Sum / Scan for GPU Compaction은 포트폴리오에서 다음 메시지를 만든다.

> “나는 GPU compute에서 단순 병렬 연산뿐 아니라, sparse result를 compact buffer로 바꾸는 data-parallel pipeline building block을 이해한다.”

네 배경과 연결하면 다음 표현이 좋다.

- CFD iso-surface active cell extraction에서 triangle count scan으로 output offset 계산
- Sparse voxel visible brick list를 flag/scan/scatter로 생성하는 구조 이해
- GPU Hash Grid에서 cell count scan으로 cell offset/range 생성 가능
- WebGPU / Vulkan compute shader 기반 GPU-driven renderer에서 visible list와 indirect draw command 생성으로 확장 가능

면접에서는 다음처럼 말할 수 있다.

> “Prefix Sum은 GPU에서 조건을 만족한 element들이 output buffer의 어디에 기록될지를 계산하는 핵심 primitive입니다. Culling, compaction, hash grid, marching cubes, indirect draw generation처럼 variable-length output이 필요한 곳에 반복적으로 사용됩니다.”

## 9. 내일 이어서 볼 개념

**GPU Radix Sort for Spatial Data**

Prefix Sum / Scan 다음에는 GPU Radix Sort를 보는 것이 자연스럽다. Hash Grid, Morton key sorting, particle binning, visible list ordering에서 sort는 scan과 함께 GPU data pipeline을 구성하는 핵심 building block이다.

## 10. 참고 키워드

- Prefix Sum
- Parallel Scan
- Exclusive Scan
- Inclusive Scan
- Stream Compaction
- GPU Compaction
- Flag / Scan / Scatter
- Atomic Append
- Visible List
- Active Cell Extraction
- Marching Cubes
- Hash Grid
- Cell Range Buffer
- Indirect Draw
- GPU-driven Rendering
- Compute Shader
- Workgroup Scan
- Hierarchical Scan
- Scientific Visualization
