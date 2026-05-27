---
title: "Sparse Volume Brick Layout"
date: "2026-05-27"
category: "Graphics"
tags: ["Sparse Volume", "Brick Layout", "GPU Memory", "SDF", "Level Set", "Marching Cubes", "Scientific Visualization"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-05-27 - Sparse Volume Brick Layout

## 1. 오늘의 개념

**Sparse Volume Brick Layout**은 거대한 3D volume 데이터를 전체 dense grid로 저장하지 않고, 실제 의미 있는 영역만 작은 3D block, 즉 **brick** 단위로 나누어 저장하는 구조다.

예를 들어 전체 domain이 `1024³` voxel이라고 해도, 실제 표면 근처나 물질이 존재하는 영역만 active하다면 전체 volume을 모두 저장할 필요가 없다. 이때 공간을 `8³`, `16³`, `32³` 같은 작은 brick으로 나누고, active brick만 GPU memory에 배치한다.

이 구조는 다음과 같은 graphics / simulation 데이터에서 자주 등장한다.

- SDF, Signed Distance Field
- Level Set, `phi` field
- Sparse voxel volume
- Particle density field
- Fluid simulation grid
- Smoke / fog volume
- Semiconductor layer field
- Marching Cubes용 scalar field
- Scientific visualization volume data

핵심은 단순히 “메모리를 아낀다”가 아니다.  
GPU가 접근하기 좋은 단위로 volume을 재배치하고, empty space를 건너뛰며, compute/rendering pipeline의 dispatch 단위를 명확하게 만든다는 점이 중요하다.

---

## 2. 한 줄 핵심

**Sparse Volume Brick Layout은 3D 공간 데이터를 active brick 단위로 압축·배치하여 GPU memory bandwidth, cache locality, compute dispatch 효율을 동시에 개선하는 구조다.**

---

## 3. 왜 중요한가

Dense 3D texture는 직관적이다. `x, y, z` 좌표가 있으면 바로 texel 또는 voxel에 접근할 수 있다. 하지만 문제는 메모리다.

예를 들어 `1024³` volume을 `float` 하나로 저장하면 약 4GB가 필요하다.

```text
1024 * 1024 * 1024 * 4 bytes ≈ 4 GB
```

여기에 gradient, material id, velocity, temperature, density, flags 같은 attribute가 추가되면 메모리 사용량은 빠르게 증가한다.

Graphics engineer 관점에서 더 중요한 문제는 단순 용량보다 **bandwidth와 access pattern**이다.

GPU는 많은 thread가 연속된 memory를 읽을 때 강하다. 하지만 sparse한 3D 공간에서 무작위로 voxel을 읽거나, 대부분이 empty region인 volume을 전부 순회하면 GPU core는 바쁘지만 실제 유효한 작업량은 낮아진다.

Sparse brick layout은 이 문제를 다음 방식으로 해결한다.

1. **Empty space skipping**  
   비어 있는 공간은 brick 자체를 만들지 않는다.

2. **Active region 중심 처리**  
   표면, 경계, material transition, high-gradient region만 처리한다.

3. **GPU dispatch 단위 명확화**  
   compute shader에서 brick 하나를 workgroup 또는 여러 workgroup의 작업 단위로 매핑하기 쉽다.

4. **Cache locality 개선**  
   하나의 brick 내부 voxel들은 연속된 memory에 저장될 수 있어 locality가 좋아진다.

5. **LOD / streaming 확장성**  
   brick 단위로 resolution, priority, visibility, streaming 상태를 관리할 수 있다.

---

## 4. 구현 관점

Sparse volume brick layout은 보통 다음 구성 요소로 생각할 수 있다.

```text
World Space
    ↓
Brick Coordinate
    ↓
Indirection Table
    ↓
Brick Pool
    ↓
Voxel Data
```

### 4.1 Brick Coordinate

전체 volume을 brick 단위 grid로 나눈다.

예를 들어 brick size가 `16³`이라면 voxel coordinate `(x, y, z)`는 다음처럼 분리된다.

```text
brickCoord = voxelCoord / 16
localCoord = voxelCoord % 16
```

즉 하나의 voxel 위치는 두 단계 주소로 해석된다.

```text
global voxel position
= brick coordinate + local coordinate inside brick
```

이 구조는 CPU의 page table과 비슷하게 볼 수 있다.  
전체 주소 공간은 크지만, 실제 물리 memory에는 필요한 page 또는 brick만 존재한다.

---

### 4.2 Indirection Table

**Indirection Table**은 logical brick coordinate가 실제 brick pool의 어느 위치에 있는지 알려주는 lookup table이다.

```text
brickCoord → brickIndex
```

예를 들어:

```text
(0, 0, 0) → empty
(1, 0, 0) → brick pool index 12
(1, 1, 0) → brick pool index 13
(5, 2, 8) → empty
```

GPU shader에서는 다음과 같은 흐름으로 접근한다.

```text
1. world position을 voxel coordinate로 변환
2. voxel coordinate를 brickCoord/localCoord로 분리
3. indirection table에서 brick index 조회
4. empty라면 skip
5. active라면 brick pool에서 voxel data 조회
```

이 방식은 direct addressing보다 한 번의 lookup이 더 필요하다.  
하지만 empty region을 통째로 제거할 수 있다면 전체 비용은 훨씬 낮아진다.

---

### 4.3 Brick Pool

**Brick Pool**은 active brick들의 실제 voxel data가 저장되는 큰 buffer 또는 3D texture array다.

구현 선택지는 여러 가지다.

```text
Option A: SSBO / Storage Buffer
- WebGPU storage buffer
- Vulkan storage buffer
- OpenGL SSBO
- CUDA global memory

Option B: 3D Texture Atlas
- 여러 brick을 큰 3D texture에 packing
- hardware texture filtering 활용 가능

Option C: Texture Array
- brick 하나를 layer 또는 tile처럼 관리
- sampling에는 편하지만 packing 효율은 구조에 따라 달라짐
```

C++ / GPU engine 관점에서는 SSBO 기반이 가장 일반적인 compute-friendly 구조다.  
반면 volume rendering에서 trilinear filtering을 hardware sampler로 적극 활용하고 싶다면 3D texture atlas 방식도 고려할 수 있다.

---

### 4.4 Brick Size 선택

Brick size는 성능을 크게 좌우한다.

작은 brick:

```text
8³
```

장점:
- sparse region을 더 세밀하게 표현
- empty space 제거 효율이 좋음
- 얇은 구조나 복잡한 boundary에 유리

단점:
- brick 개수가 많아짐
- indirection table lookup 증가
- metadata overhead 증가

큰 brick:

```text
32³ 또는 64³
```

장점:
- brick 수가 줄어듦
- metadata overhead 감소
- sequential access에 유리

단점:
- brick 내부에 empty voxel이 많이 포함될 수 있음
- sparse 효율 저하
- thin layer나 narrow band SDF에는 낭비 발생

반도체 3D visualization처럼 얇은 layer, taper angle, CD-bias, multi-material boundary가 중요한 경우에는 너무 큰 brick이 불리할 수 있다.  
반대로 smoke volume이나 넓게 퍼진 density field는 큰 brick이 더 단순하고 빠를 수 있다.

---

### 4.5 Compute Shader 관점

Sparse brick layout은 compute shader와 잘 맞는다.

대표적인 dispatch 구조는 다음과 같다.

```text
1 brick = 1 workgroup
```

예를 들어 brick size가 `8³`이면 voxel 수는 512개다.  
workgroup thread 수를 128 또는 256으로 두고 thread 하나가 여러 voxel을 처리할 수 있다.

```text
workgroup 0 → active brick 0
workgroup 1 → active brick 1
workgroup 2 → active brick 2
...
```

이 방식의 장점은 global dense volume 전체를 순회하지 않는다는 점이다.  
active brick list만 dispatch하면 된다.

GPU pipeline에서는 다음 데이터가 중요하다.

```text
ActiveBrickList
BrickMetadataBuffer
IndirectionTable
VoxelDataBuffer
OutputVertexBuffer
OutputIndexBuffer
```

Marching Cubes를 수행한다면 active brick마다 cell을 순회하고, iso-surface가 존재하는 cell만 triangle을 생성한다.

---

### 4.6 Rendering Pipeline 관점

Sparse volume은 rendering pipeline에서 여러 방식으로 사용될 수 있다.

#### Volume Ray Marching

Ray가 volume을 통과할 때 empty brick은 건너뛰고 active brick에서만 sampling한다.

```text
ray → brick grid traversal → active brick sampling → shading
```

이때 brick-level bounding box와 occupancy flag가 중요하다.

#### Marching Cubes

Sparse scalar field에서 active brick만 polygonization한다.

```text
active brick → local marching cubes → triangle append buffer
```

표면 근처 narrow band만 brick으로 유지하면 dense volume 대비 triangle extraction 비용이 크게 줄어든다.

#### Scientific Visualization

CFD, FEM, TCAD, semiconductor visualization에서는 scalar/vector field를 brick 단위로 관리할 수 있다.

```text
pressure
velocity
temperature
density
material id
signed distance
process layer id
```

이 attribute들을 SoA, Structure of Arrays 형태로 분리하면 shader에서 필요한 field만 읽을 수 있어 bandwidth를 절약할 수 있다.

---

## 5. 내 관심 분야와 연결

이 개념은 사용자의 관심 분야와 매우 직접적으로 연결된다.

### 5.1 SDF / Level Set

Level set에서 `phi = 0`은 surface를 의미한다.  
하지만 전체 공간에서 중요한 것은 대개 surface 주변 narrow band다.

```text
important region = abs(phi) < threshold
```

Sparse brick layout은 이 narrow band만 저장하기 좋다.

```text
empty far region → skip
near-surface region → active brick
```

따라서 반도체 layer geometry를 SDF나 level-set field로 표현할 때, 전체 dense grid 대신 active band 중심의 sparse brick layout이 현실적인 선택지가 된다.

---

### 5.2 Marching Cubes

Marching Cubes는 scalar field에서 iso-surface를 mesh로 추출한다.

Dense grid에서는 모든 cell을 검사한다.

```text
for every cell in volume:
    check iso-surface crossing
```

Sparse brick layout에서는 active brick 내부 cell만 검사한다.

```text
for every active brick:
    for every local cell:
        check iso-surface crossing
```

즉 marching cubes의 병목을 줄이는 핵심은 triangle generation 자체뿐 아니라, **검사할 필요 없는 공간을 얼마나 빨리 제외하느냐**다.

---

### 5.3 Semiconductor 3D Visualization

반도체 geometry는 매우 얇은 layer, material boundary, etch/deposition 결과, taper angle, CD-bias 같은 정보가 중요하다.

이 경우 전체 3D domain을 균일한 dense grid로 표현하면 낭비가 크다.

```text
많은 빈 공간
얇은 막 구조
복잡한 boundary
multi-material interface
```

Sparse brick layout은 이런 구조에서 유리하다.

- layer가 존재하는 영역만 active brick화
- material id를 brick metadata에 저장
- boundary 근처만 high-resolution 유지
- 멀리 있거나 변화가 적은 영역은 coarse brick 또는 skip
- GPU compute로 surface extraction 또는 field update 수행

특히 “simulation 결과가 아니라 이미 thickness / CD-bias / taper angle 같은 결과 파라미터를 가지고 있고, 이를 rendering하고 싶다”는 목표에서는 sparse field representation이 geometry generation의 중간 표현으로 적합하다.

---

### 5.4 WebGPU / Vulkan / CUDA 관점

WebGPU에서는 storage buffer와 compute shader를 사용할 수 있으므로 sparse brick layout을 구성하기 좋다.

```text
StorageBuffer<BrickMetadata>
StorageBuffer<Indirection>
StorageBuffer<VoxelField>
StorageBuffer<GeneratedVertices>
```

Vulkan이나 CUDA에서는 더 세밀한 memory control과 compaction, prefix sum, indirect dispatch 최적화가 가능하다.

실무적으로는 다음 흐름이 중요하다.

```text
1. CPU 또는 GPU에서 active brick 생성
2. active brick list compact
3. brick metadata upload/update
4. compute shader로 field sampling 또는 mesh extraction
5. render pass에서 generated mesh 또는 volume render
```

이 구조는 OpenGL compute shader, Vulkan compute pipeline, WebGPU compute pipeline 모두에 개념적으로 대응된다.

---

## 6. 머릿속에 남길 질문 3개

1. **Brick size는 왜 단순히 메모리 절약 기준이 아니라 GPU cache, thread group size, wavefront/warp access pattern까지 고려해서 정해야 할까?**

2. **Indirection table lookup 비용이 추가되는데도 sparse brick layout이 dense 3D texture보다 빠를 수 있는 조건은 무엇일까?**

3. **Marching Cubes에서 active brick culling은 triangle generation 비용보다 앞단의 어떤 병목을 줄여주는가?**

---

## 7. Graphics Engineer 면접 질문 1개와 답변

### Q. Dense 3D texture 대신 sparse brick volume을 사용하는 이유는 무엇인가?

Dense 3D texture는 direct addressing이 단순하고 hardware sampling을 활용하기 좋지만, volume 전체를 메모리에 올려야 하기 때문에 큰 domain에서 memory usage와 bandwidth 비용이 매우 커진다.

Sparse brick volume은 공간을 작은 brick 단위로 나누고 active region만 저장한다. 이를 통해 empty space를 건너뛸 수 있고, compute shader dispatch를 active brick 중심으로 수행할 수 있다. 특히 SDF, level set, voxelized geometry, volume rendering, marching cubes처럼 실제로 중요한 영역이 surface 주변이나 material boundary 근처에 집중된 경우 효율이 높다.

단점도 있다. Indirection table lookup이 필요하고, brick metadata 관리, compaction, streaming, boundary sampling 처리가 복잡해진다. 따라서 dense access가 대부분인 데이터라면 dense 3D texture가 더 단순하고 빠를 수 있다. 하지만 sparse한 3D scene이나 scientific visualization에서는 sparse brick layout이 더 확장성 있는 선택이다.

---

## 8. 포트폴리오 / 커리어 연결

이 개념은 포트폴리오에서 단순히 “volume rendering을 했다”보다 훨씬 강한 기술적 메시지를 만든다.

예를 들어 다음처럼 표현할 수 있다.

```text
Designed a sparse brick-based volume representation for large-scale scalar fields,
reducing empty-space traversal and enabling GPU-friendly active-brick compute dispatch
for SDF-based surface extraction and scientific visualization.
```

한국어로 풀면 다음과 같다.

```text
대규모 scalar field를 dense grid로 처리하지 않고 sparse brick 기반 구조로 재설계하여,
empty space traversal을 줄이고 GPU compute shader에서 active brick 단위로 surface extraction을 수행할 수 있는 구조를 설계했다.
```

이 문장은 다음 역량을 동시에 보여준다.

- GPU memory layout 이해
- sparse data structure 이해
- volume rendering / marching cubes 이해
- compute shader dispatch 설계
- scientific visualization 최적화
- 대규모 데이터 처리 관점

Graphics engineer, visualization engineer, rendering engineer, game engine graphics programmer 면접에서 모두 강하게 연결되는 주제다.

---

## 9. 내일 이어서 볼 개념

**Active Brick Compaction & Prefix Sum**

Sparse brick layout에서 active brick을 찾는 것만으로는 충분하지 않다.  
GPU가 효율적으로 처리하려면 active brick들을 연속된 list로 압축해야 한다.

내일은 다음 질문으로 이어간다.

```text
GPU에서 active voxel/brick을 어떻게 compact하고,
prefix sum은 왜 sparse rendering pipeline의 핵심 primitive가 되는가?
```

---

## 10. 참고 키워드

- Sparse Volume
- Brick Pool
- Indirection Table
- Active Brick List
- Empty Space Skipping
- Voxel Grid
- SDF, Signed Distance Field
- Level Set
- Narrow Band
- Marching Cubes
- Volume Ray Marching
- GPU Memory Layout
- Cache Locality
- Storage Buffer
- SSBO
- WebGPU Compute Shader
- Vulkan Compute
- CUDA
- Prefix Sum
- Stream Compaction
- Scientific Visualization
- Semiconductor Visualization
