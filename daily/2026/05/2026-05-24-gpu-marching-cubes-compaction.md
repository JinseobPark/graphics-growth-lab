---
title: "GPU Marching Cubes Compaction Pipeline"
date: "2026-05-24"
category: "Graphics"
tags: ["Marching Cubes", "GPU Compute", "Compaction", "Isosurface", "Sparse Volume", "SDF", "Level Set", "Scientific Visualization"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-05-24 - GPU Marching Cubes Compaction Pipeline

## 1. 오늘의 개념

**GPU Marching Cubes Compaction Pipeline**은 3D scalar field, SDF, level-set, density grid 같은 볼륨 데이터에서 iso-surface를 추출할 때, 실제로 삼각형을 생성하는 voxel만 골라내고 GPU 메모리에 조밀하게 배치하는 파이프라인이다.

Marching Cubes 자체는 각 grid cell의 8개 corner 값을 보고 iso-value와 비교한 뒤, 256개 case table 중 하나를 선택해 triangle을 만든다. 하지만 GPU 실무 관점에서 중요한 문제는 “triangle을 어떻게 만들 것인가”보다 “triangle을 만들 cell만 어떻게 효율적으로 찾고, 출력 buffer에 충돌 없이 쓰는가”다.

즉 핵심은 다음 흐름이다.

```text
Scalar Field / SDF / Level-set
        ↓
Active Cell Classification
        ↓
Triangle Count Estimation
        ↓
Prefix Sum / Scan
        ↓
Compacted Output Offset
        ↓
Triangle Generation
        ↓
Render / Mesh Export / BVH Build
```

## 2. 한 줄 핵심

**GPU Marching Cubes의 병목은 case table 계산보다, variable-sized triangle output을 안정적으로 압축(compaction)해서 GPU buffer에 쓰는 구조다.**

## 3. 왜 중요한가

Marching Cubes는 cell마다 0개에서 최대 5개의 triangle을 만든다. 즉 output size가 cell마다 다르다. CPU에서는 vector에 push_back하면 되지만, GPU에서는 수십만~수백만 thread가 동시에 output buffer에 쓰기 때문에 각 thread가 쓸 위치를 미리 알아야 한다.

이때 단순 atomic add 방식은 구현은 쉽지만 active cell이 많아질수록 contention이 커지고, 출력 순서도 불안정해진다. 반대로 prefix sum 기반 compaction은 한 번 triangle count를 계산한 뒤, 각 cell의 시작 offset을 결정하므로 deterministic하고 대규모 볼륨에 적합하다.

Graphics engineer 관점에서 이 개념은 단순 알고리즘 지식이 아니라 **GPU memory allocation, indirect rendering, compute-to-render synchronization, sparse volume traversal**과 연결된다. 특히 SDF 기반 modeling, fluid surface reconstruction, semiconductor process visualization, voxel terrain, particle-to-surface 변환에서 반복적으로 등장한다.

## 4. 구현 관점

### 4.1 Active Cell Classification

각 voxel cell에 대해 8개 corner의 scalar value를 읽고 iso-value보다 안쪽인지 바깥쪽인지 bit mask를 만든다.

```text
caseIndex = 0
for each corner i in 0..7:
    if value[i] < isoValue:
        caseIndex |= (1 << i)
```

caseIndex가 0 또는 255이면 surface가 cell을 통과하지 않으므로 triangle이 생성되지 않는다. 이 cell은 inactive cell이다.

구현에서 중요한 점은 corner value fetch pattern이다. dense 3D texture, SSBO, storage buffer, CUDA global memory, NanoVDB-style sparse accessor 중 어떤 구조를 쓰느냐에 따라 memory coherence가 크게 달라진다.

### 4.2 Triangle Count Estimation

caseIndex를 triangle count lookup table에 넣으면 해당 cell이 몇 개의 triangle을 만들지 알 수 있다.

```text
triCount[cellID] = triCountTable[caseIndex]
```

이 단계는 geometry를 아직 만들지 않는다. “얼마나 만들지”만 계산한다. 이 separation이 중요하다. GPU에서는 output buffer 크기와 offset을 알아야 다음 pass에서 안전하게 쓸 수 있기 때문이다.

### 4.3 Prefix Sum / Scan

모든 cell의 triCount 배열에 prefix sum을 수행하면 각 cell이 output buffer에서 사용할 시작 위치를 얻는다.

```text
triOffset[cellID] = prefixSum(triCount)[cellID]
```

예를 들어 triCount가 다음과 같다면:

```text
cell:      0  1  2  3  4
triCount: 0  2  1  0  3
offset:   0  0  2  3  3
```

cell 1은 output triangle 0번부터 2개, cell 2는 2번부터 1개, cell 4는 3번부터 3개를 쓴다. 이 구조 덕분에 각 thread는 atomic 없이 자기 output 위치를 알 수 있다.

### 4.4 Triangle Generation

두 번째 compute pass에서 다시 caseIndex를 계산하거나, 이전 pass에서 저장한 caseIndex를 읽는다. edge table과 tri table을 이용해 edge crossing point를 계산하고, triOffset 위치에 vertex/index를 쓴다.

```text
outTriBase = triOffset[cellID]
for each triangle in caseTable[caseIndex]:
    interpolate edge vertices
    write triangle to outputBuffer[outTriBase + localTri]
```

GPU renderer로 바로 연결할 경우 output은 다음 중 하나가 된다.

- vertex buffer + index buffer
- triangle soup buffer
- meshlet-like compact primitive buffer
- indirect draw argument buffer
- ray tracing acceleration structure input buffer

실시간 rendering에서는 triangle soup로 바로 그리는 구조가 가장 단순하다. 하지만 normal smoothing, material boundary, topology consistency, mesh export까지 고려하면 vertex deduplication이나 post-process pass가 필요할 수 있다.

### 4.5 Dense Grid vs Sparse Grid

Dense grid에서는 모든 cell을 순회하기 때문에 구현은 단순하지만 empty space 비용이 크다. Sparse grid에서는 active block, tile, leaf node 단위로만 순회할 수 있어 대규모 데이터에 유리하다.

반도체 3D visualization이나 CFD volume에서는 전체 domain이 크지만 실제 surface가 존재하는 영역은 일부일 수 있다. 이 경우 sparse block list를 먼저 만들고, block 내부에서만 Marching Cubes를 수행하는 구조가 적합하다.

```text
Sparse Blocks
    ↓
Block-level active classification
    ↓
Cell-level Marching Cubes inside active blocks
    ↓
Global compaction
```

여기서 어려운 점은 block마다 triangle 수가 다르다는 것이다. 그래서 block-level prefix sum과 cell-level prefix sum이 계층적으로 필요할 수 있다.

## 5. 내 관심 분야와 연결

### CFD / Scientific Visualization

CFD 결과에서 pressure, density, vorticity magnitude 같은 scalar field의 iso-surface를 실시간으로 추출할 수 있다. 사용자가 iso-value를 바꾸면 mesh가 매번 달라지므로 CPU mesh rebuild보다 GPU compute 기반 rebuild가 유리하다.

### Particle Simulation / Fluid Surface

SPH particle을 density grid로 splat한 뒤 Marching Cubes를 적용하면 fluid surface mesh를 만들 수 있다. 여기서 compaction은 fluid 표면이 있는 cell만 골라내는 역할을 한다. particle 수가 많아질수록 전체 grid를 도는 비용보다 active region을 줄이는 전략이 중요해진다.

### Semiconductor 3D Visualization

SDF / level-set 기반으로 증착, 식각, taper angle, CD-bias가 반영된 implicit field를 만들고, iso-surface를 mesh로 추출하는 데 사용할 수 있다. 단, 최종 제품 렌더링에서는 둥글둥글한 surface만 원하는 것이 아니라 layer boundary, sharp edge, material interface를 보존해야 한다. 따라서 Marching Cubes는 “surface extraction stage”이며, CAD-like sharpness를 위해서는 dual contouring, feature-preserving reconstruction, material-aware meshing과 함께 검토해야 한다.

### WebGPU / Vulkan / CUDA

WebGPU, Vulkan compute, CUDA 모두 이 구조를 구현할 수 있다. 차이는 prefix sum library, memory barrier, buffer binding, indirect draw 지원, sparse memory 접근 방식에서 나타난다.

- CUDA: scan/compaction primitive가 강력하고 debugging ecosystem이 성숙함
- Vulkan: rendering pipeline과 compute pipeline을 같은 GPU queue 안에서 정교하게 연결 가능
- WebGPU: browser 기반 visualization에 적합하지만 prefix sum과 indirect draw 구조를 직접 설계해야 함
- OpenGL compute shader: 기존 C++ OpenGL renderer와 연결하기 좋지만 synchronization과 buffer lifecycle 관리가 중요함

## 6. 머릿속에 남길 질문 3개

1. Marching Cubes에서 cell마다 triangle 개수가 다르다는 사실이 GPU output buffer 설계에 어떤 문제를 만드는가?
2. atomic add 기반 output allocation과 prefix sum 기반 compaction은 각각 어떤 상황에서 유리한가?
3. 반도체처럼 sharp material boundary가 중요한 데이터에서 Marching Cubes만 사용하면 어떤 시각적 문제가 생길 수 있는가?

## 7. Graphics Engineer 면접 질문 1개와 답변

### Q. GPU에서 Marching Cubes를 구현할 때, 왜 prefix sum이 필요한가요?

A. Marching Cubes는 각 voxel cell이 생성하는 triangle 수가 고정되어 있지 않습니다. 어떤 cell은 0개, 어떤 cell은 여러 개의 triangle을 생성합니다. GPU thread가 병렬로 실행될 때 각 thread가 output buffer의 어느 위치에 triangle을 써야 하는지 알 수 없으면 write conflict가 발생합니다.  

그래서 첫 번째 pass에서 각 cell의 triangle count를 계산하고, 이 배열에 prefix sum을 수행해 cell별 output offset을 만듭니다. 두 번째 pass에서는 각 thread가 자신의 offset에 triangle을 쓰기 때문에 atomic contention을 줄이고, deterministic한 compact mesh buffer를 만들 수 있습니다. 이 구조는 compute shader에서 mesh generation 후 indirect draw로 연결할 때 특히 중요합니다.

## 8. 포트폴리오 / 커리어 연결

이 개념은 포트폴리오에서 단순히 “Marching Cubes 구현”이라고 쓰기보다 다음처럼 표현하는 것이 좋다.

```text
Implemented GPU-based isosurface extraction pipeline using active-cell classification, prefix-sum compaction, and compute-generated vertex buffers for real-time visualization of volumetric simulation data.
```

또는 반도체 visualization 맥락에서는 다음처럼 말할 수 있다.

```text
Designed a GPU-driven implicit-surface meshing pipeline for semiconductor process visualization, focusing on sparse volume traversal, compact triangle generation, and material-aware rendering constraints.
```

이 표현은 알고리즘만 아는 것이 아니라 GPU pipeline 전체를 이해하고 있다는 인상을 준다. 특히 Unity, Nintendo, visualization software company, semiconductor modeling software company 모두에서 “GPU memory-aware thinking”은 강한 시그널이 된다.

## 9. 내일 이어서 볼 개념

**Prefix Sum / Parallel Scan for GPU Graphics Pipelines**

내일은 Marching Cubes compaction의 핵심 연산인 prefix sum을 graphics engineer 관점에서 본다. 단순 병렬 알고리즘이 아니라 particle compaction, light culling, visibility buffer, GPU sorting, sparse voxel allocation과 연결해서 이해한다.

## 10. 참고 키워드

- Marching Cubes
- Isosurface Extraction
- GPU Compaction
- Prefix Sum / Parallel Scan
- Stream Compaction
- Compute Shader
- Indirect Draw
- Triangle Count Buffer
- Active Cell Classification
- SDF
- Level Set
- Sparse Volume
- NanoVDB
- Dual Contouring
- Feature-preserving Meshing
- Scientific Visualization
- CFD Visualization
- Semiconductor Process Visualization
