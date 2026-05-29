---
title: "Sparse Voxel Representation"
date: "2026-05-29"
category: "Graphics"
tags: ["Sparse Voxel", "Voxel", "Octree", "SVO", "GPU Memory", "Scientific Visualization", "Rendering Pipeline", "Compute Shader"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-05-29 - Sparse Voxel Representation

## 1. 오늘의 개념

**Sparse Voxel Representation**은 3D 공간 전체를 조밀한 격자(dense grid)로 저장하지 않고, 실제로 의미 있는 영역만 선택적으로 저장하는 공간 표현 방식이다.  
대표적인 구조로는 **Sparse Voxel Octree(SVO)**, **brick-based sparse grid**, **NanoVDB-style sparse tree**, **clipmap-based sparse volume** 등이 있다.

Dense grid는 개념적으로 단순하다. 예를 들어 `512^3` volume이면 모든 cell을 연속된 3D 배열로 저장한다. 문제는 실제 시뮬레이션/렌더링 데이터에서는 대부분의 공간이 비어 있거나, 화면에 필요하지 않거나, 낮은 해상도로 충분하다는 점이다. Sparse voxel 구조는 이 낭비를 줄이고, GPU memory bandwidth와 cache locality를 더 의식한 형태로 데이터를 배치한다.

그래픽스 엔지니어 관점에서 이 개념은 단순한 자료구조가 아니라, **렌더링 가능한 데이터로 변환하기 전 단계의 공간 압축, LOD, culling, streaming의 기반**이다.

---

## 2. 한 줄 핵심

> **Sparse voxel은 거대한 3D 공간을 “전부 저장하는 문제”에서 “필요한 공간만 GPU가 빠르게 찾고 읽는 문제”로 바꾸는 표현 방식이다.**

---

## 3. 왜 중요한가

대규모 CFD, 반도체 3D 구조, point cloud, volume rendering, marching cubes 기반 surface extraction에서는 데이터 크기가 빠르게 폭발한다. Dense grid는 구현은 쉽지만, 해상도를 조금만 올려도 memory footprint가 `O(N^3)`로 증가한다.

예를 들어 반도체 stack 구조처럼 얇은 layer와 빈 공간이 섞여 있거나, CFD 결과처럼 관심 영역이 특정 region에 집중되어 있다면 전체 domain을 동일 해상도로 저장하는 것은 GPU 관점에서 비효율적이다. Sparse voxel은 이런 상황에서 **memory usage, traversal cost, rendering granularity**를 동시에 제어할 수 있게 해준다.

또 하나 중요한 점은 sparse representation이 단순 저장 최적화가 아니라는 것이다. 잘 설계된 sparse voxel 구조는 다음 기능들의 기반이 된다.

- **Frustum culling / occlusion culling**
- **LOD selection**
- **out-of-core streaming**
- **ray traversal acceleration**
- **marching cubes 대상 영역 축소**
- **compute shader 기반 filtering / classification**
- **simulation result visualization의 interactive update**

즉, sparse voxel은 “압축된 3D 배열”이라기보다, **GPU rendering pipeline에 들어가기 전의 spatial scheduling structure**로 보는 것이 더 정확하다.

---

## 4. 구현 관점

### 4.1 Dense grid의 기본 한계

Dense volume은 보통 다음처럼 접근된다.

```cpp
index = x + y * width + z * width * height;
value = volume[index];
```

장점은 명확하다. 주소 계산이 단순하고, memory가 연속적이며, GPU texture 또는 storage buffer로 넘기기 쉽다. 그러나 해상도가 증가하면 모든 cell에 대해 memory를 할당해야 한다.

`1024^3` grid에서 scalar field 하나를 `float`로 저장하면 약 4GB가 필요하다. 여기에 material id, normal, velocity, temperature, mask, gradient 등을 추가하면 GPU memory budget을 빠르게 넘는다.

Sparse voxel은 이 문제를 다음 질문으로 바꾼다.

> “모든 cell을 저장할 것인가?”가 아니라, “어떤 block이 실제로 의미 있고, GPU가 그것을 어떻게 찾을 것인가?”

---

### 4.2 Octree 기반 표현

**Sparse Voxel Octree(SVO)**는 3D 공간을 재귀적으로 8분할한다. 상위 node는 큰 공간을 의미하고, 하위 node로 갈수록 더 세밀한 영역을 나타낸다.

렌더링 관점에서 SVO는 다음 장점을 가진다.

- 빈 공간을 빠르게 skip할 수 있다.
- 멀리 있는 영역은 coarse level에서 처리할 수 있다.
- ray traversal 또는 voxel cone tracing 같은 기법과 잘 맞는다.
- hierarchical culling이 가능하다.

하지만 단점도 명확하다.

- pointer-heavy 구조는 GPU에서 비효율적이다.
- random access가 많아 cache miss가 커질 수 있다.
- dynamic update가 어렵다.
- leaf data layout을 잘못 설계하면 traversal은 빨라도 shading 단계가 느려진다.

따라서 GPU에서는 일반적인 CPU pointer tree보다는 다음 형태가 더 적합하다.

```text
NodeBuffer[]
- childMask
- firstChildIndex
- brickIndex
- bounds / level metadata

BrickBuffer[]
- scalar values
- material ids
- occupancy mask
- min/max range
```

핵심은 pointer가 아니라 **index-based flat buffer**로 만드는 것이다. 그래야 SSBO, storage buffer, structured buffer, CUDA global memory에서 일관된 접근이 가능하다.

---

### 4.3 Brick-based sparse grid

실무에서 자주 유용한 방식은 순수 octree보다 **brick-based sparse grid**다.  
전체 domain을 작은 block, 예를 들어 `8^3`, `16^3`, `32^3` voxel brick으로 나누고, 실제 데이터가 있는 brick만 저장한다.

```text
Domain
 ├─ Brick (32 x 32 x 32)
 ├─ Brick (32 x 32 x 32)
 └─ Empty region skipped
```

이 방식은 GPU에 특히 좋다.

- brick 내부는 dense array라서 memory access가 단순하다.
- brick 단위로 culling, LOD, streaming이 가능하다.
- marching cubes를 brick 단위로 dispatch할 수 있다.
- compute shader workgroup과 매핑하기 쉽다.
- min/max metadata를 저장하면 threshold, iso-value filtering을 빠르게 skip할 수 있다.

예를 들어 iso-surface extraction에서 brick의 scalar min/max가 iso-value를 포함하지 않으면 해당 brick 전체를 건너뛸 수 있다.

```text
if isoValue < brick.minValue or isoValue > brick.maxValue:
    skip brick
else:
    run marching cubes for this brick
```

이 구조는 네가 관심 있는 **CFD post-processing**, **VTK 대용량 데이터**, **반도체 layer visualization**, **GPU marching cubes**와 직접 연결된다.

---

### 4.4 GPU memory layout 관점

Sparse voxel에서 진짜 어려운 부분은 “자료구조를 만들었다”가 아니라, GPU에서 읽기 좋은 형태로 배치하는 것이다.

좋은 layout은 다음 특성을 가진다.

1. **metadata와 payload를 분리한다.**  
   traversal/culling에 필요한 정보와 실제 scalar/material 값을 다른 buffer로 둔다.

2. **brick 내부는 최대한 linear하게 둔다.**  
   `x + y * B + z * B * B` 방식으로 단순 index 계산이 가능해야 한다.

3. **mask를 적극적으로 사용한다.**  
   occupancy mask, active cell mask, material mask는 empty region skip에 중요하다.

4. **GPU dispatch 단위를 brick과 맞춘다.**  
   compute shader에서 하나의 workgroup이 하나의 brick 또는 brick tile을 처리하도록 설계하면 scheduling이 단순해진다.

5. **압축보다 접근 패턴이 우선이다.**  
   memory를 조금 더 쓰더라도 random access를 줄이는 쪽이 실시간 렌더링에서는 유리할 수 있다.

이 관점에서 sparse voxel은 “자료구조 알고리즘”이라기보다 **GPU data-oriented design** 문제다.

---

## 5. 내 관심 분야와 연결

### CFD / Scientific Visualization

CFD 결과는 domain 전체에 scalar/vector field가 존재할 수 있지만, 시각화에서 항상 모든 cell을 동일하게 볼 필요는 없다. Clip, slice, threshold, streamline, iso-surface 같은 post-processing은 관심 영역을 계속 바꾼다.

Sparse voxel 또는 brick metadata를 사용하면 다음과 같은 최적화가 가능하다.

- threshold 범위에 포함되지 않는 brick skip
- velocity magnitude가 낮은 영역 skip
- slice plane과 교차하지 않는 brick skip
- timestep별 changed brick만 update
- camera distance에 따른 LOD 선택

이것은 단순히 FPS를 높이는 것이 아니라, **사용자가 대용량 시뮬레이션 결과를 interactive하게 탐색할 수 있는가**를 결정한다.

### 반도체 3D Visualization

반도체 구조는 얇은 layer, taper angle, CD-bias, via, trench, etch profile처럼 local geometric detail이 중요하다. 전체 domain을 dense grid로 잡으면 z 방향의 얇은 layer 때문에 전체 resolution이 과도하게 커질 수 있다.

Sparse voxel 접근은 다음 방향과 맞다.

- layer 주변만 high resolution
- 빈 oxide/air 영역 skip
- material id 기반 brick classification
- level-set / SDF field를 sparse brick에 저장
- 최종 표면은 marching cubes 또는 dual contouring 계열로 추출

특히 “공정 시뮬레이션 자체”가 아니라 “이미 주어진 thickness / taper / bias 결과를 GPU에서 빠르게 geometry로 표현”하려는 방향이라면, sparse voxel은 geometry generation 전 단계의 핵심 표현 방식이 된다.

### Real-time Rendering / Game Engine

게임 엔진에서는 sparse voxel이 global illumination, destruction, voxel terrain, volumetric fog, visibility structure와 연결된다. 실무적으로는 다음 질문이 중요하다.

- 이 voxel data는 shading에 직접 쓰이는가?
- mesh extraction을 위한 중간 표현인가?
- ray traversal acceleration structure인가?
- LOD와 streaming의 기준인가?

같은 sparse voxel이라도 목적에 따라 layout과 update strategy가 달라진다.

---

## 6. 머릿속에 남길 질문 3개

1. **Sparse voxel 구조에서 “node traversal 비용”과 “brick 내부 dense access 효율”은 어디서 균형을 잡아야 하는가?**
2. **GPU marching cubes를 할 때, 전체 grid dispatch보다 brick metadata 기반 dispatch가 왜 더 유리한가?**
3. **반도체처럼 얇은 layer가 많은 구조에서는 octree, brick grid, NanoVDB-style tree 중 어떤 표현이 가장 자연스러운가?**

---

## 7. Graphics Engineer 면접 질문 1개와 답변

### Q. Dense voxel grid와 sparse voxel representation의 차이를 GPU 렌더링 관점에서 설명해보세요.

**A.**  
Dense voxel grid는 전체 3D domain을 동일 해상도의 배열로 저장하기 때문에 index 계산이 단순하고 GPU texture 또는 buffer로 넘기기 쉽습니다. 하지만 해상도가 증가하면 memory usage가 `O(N^3)`로 증가하고, 빈 공간이나 시각적으로 중요하지 않은 영역까지 모두 처리해야 합니다.

Sparse voxel representation은 실제 데이터가 있거나 렌더링에 필요한 영역만 저장합니다. Octree나 brick-based sparse grid를 사용하면 empty space skipping, LOD, culling, streaming이 가능해집니다. GPU 관점에서는 pointer 기반 구조보다 flat buffer와 index 기반 node layout이 중요하고, brick 내부는 dense layout으로 유지해서 cache locality와 compute shader dispatch 효율을 확보하는 것이 좋습니다.

결론적으로 dense grid는 단순성과 random access에 강하고, sparse voxel은 대규모 데이터의 memory 절약과 hierarchical processing에 강합니다. 실시간 scientific visualization이나 volume rendering에서는 sparse 구조가 interactive performance를 만들기 위한 핵심 기반이 됩니다.

---

## 8. 포트폴리오 / 커리어 연결

Sparse voxel representation은 포트폴리오에서 단순한 렌더링 기능보다 더 강한 메시지를 줄 수 있다.

> “나는 큰 데이터를 그냥 그리는 사람이 아니라, GPU가 처리 가능한 구조로 재구성해서 interactive visualization pipeline을 설계할 수 있다.”

특히 네 커리어 방향에서는 다음 식으로 연결하기 좋다.

- **CFD Viewer**: VTK/VTU 데이터를 brick 또는 octree 기반 GPU-friendly layout으로 변환
- **Semiconductor 3D Viewer**: thickness / taper / CD-bias 기반 implicit field를 sparse voxel에 저장 후 surface extraction
- **WebGPU Renderer**: storage buffer 기반 sparse metadata + compute pass 기반 filtering
- **Engine Architecture**: resource streaming, LOD, culling, extraction pass를 분리한 render pipeline 설계

면접이나 포트폴리오에서 강조할 키워드는 다음이다.

```text
Large-scale simulation visualization
GPU-friendly sparse data layout
Brick-based volume representation
Hierarchical culling and LOD
Compute shader driven surface extraction
Interactive post-processing pipeline
```

---

## 9. 내일 이어서 볼 개념

**Brick-based GPU Marching Cubes**

오늘은 sparse voxel이 “어떻게 저장할 것인가”의 문제였다.  
내일은 이 sparse brick 구조 위에서 **iso-surface를 어떻게 GPU에서 추출하고, append buffer / prefix sum / indirect draw와 어떻게 연결할 것인가**를 이어서 보면 좋다.

---

## 10. 참고 키워드

- Sparse Voxel Octree, SVO
- Brick-based Sparse Grid
- NanoVDB
- OpenVDB
- Voxel Bricks
- Occupancy Mask
- Active Voxel Mask
- Empty Space Skipping
- GPU Memory Layout
- Storage Buffer / SSBO
- Compute Shader Dispatch
- Marching Cubes
- Dual Contouring
- Volume Rendering
- Out-of-core Streaming
- LOD Selection
- Scientific Visualization
- CFD Post-processing
- Semiconductor 3D Visualization
