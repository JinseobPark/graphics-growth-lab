---
title: "Prefix Sum / Parallel Scan for GPU Graphics Pipelines"
date: "2026-05-24"
category: "Graphics"
tags: ["Prefix Sum", "Parallel Scan", "GPU Compute", "Stream Compaction", "Compute Shader", "GPGPU", "Rendering Pipeline"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-05-24 - Prefix Sum / Parallel Scan for GPU Graphics Pipelines

## 1. 오늘의 개념

**Prefix Sum**, 또는 **Parallel Scan**, 은 배열의 각 원소 앞에 있는 값들의 누적합을 병렬로 계산하는 GPU 기본 연산이다.

단순히 보면 합계를 구하는 알고리즘처럼 보이지만, graphics / GPU pipeline에서는 훨씬 더 중요한 의미를 가진다. Prefix Sum은 “각 thread가 output buffer의 어디에 써야 하는가?”를 결정하는 offset generation 도구다.

예를 들어 각 cell, particle, tile, light cluster가 생성할 output 개수가 다를 때, GPU thread는 자기 결과를 쓸 정확한 위치가 필요하다. Prefix Sum은 variable-sized output을 compact buffer로 바꾸는 핵심 단계다.

```text
input count per element
        ↓
prefix sum / scan
        ↓
output offset per element
        ↓
compact GPU buffer write
```

## 2. 한 줄 핵심

**Prefix Sum은 GPU에서 variable-sized output을 deterministic한 compact buffer layout으로 바꾸는 기본 building block이다.**

## 3. 왜 중요한가

GPU는 수많은 thread가 동시에 실행된다. 각 thread가 같은 크기의 output을 만든다면 threadID를 그대로 output index로 사용할 수 있다. 하지만 graphics pipeline에서는 이런 경우보다 아닌 경우가 더 많다.

- 어떤 voxel cell은 triangle 0개를 만들고, 어떤 cell은 5개를 만든다.
- 어떤 particle은 살아 있고, 어떤 particle은 죽었다.
- 어떤 tile은 light가 2개 있고, 어떤 tile은 100개 있다.
- 어떤 meshlet은 visible이고, 어떤 meshlet은 culled된다.

이처럼 output 개수가 element마다 다르면 단순 병렬 write가 불가능하다. Atomic add로 전역 counter를 증가시키며 위치를 받을 수도 있지만, contention이 커지고 순서가 불안정해질 수 있다. Prefix Sum은 각 element가 사용할 output range를 미리 계산해 GPU 메모리 write를 예측 가능하게 만든다.

그래서 Prefix Sum은 Marching Cubes, particle compaction, tiled light culling, visibility buffer, GPU sorting, sparse voxel allocation 같은 여러 graphics 시스템의 공통 기반이다.

## 4. 구현 관점

### 4.1 Inclusive Scan vs Exclusive Scan

Prefix Sum에는 보통 두 가지 형태가 있다.

```text
input:          [3, 1, 0, 2]
inclusive scan: [3, 4, 4, 6]
exclusive scan: [0, 3, 4, 4]
```

GPU output offset을 만들 때는 보통 **exclusive scan**이 더 자연스럽다.

예를 들어 각 element가 생성할 triangle 개수가 다음과 같다고 하자.

```text
triCount: [0, 2, 1, 0, 3]
offset:   [0, 0, 2, 3, 3]
```

두 번째 element는 triangle 2개를 만들고 offset 0부터 쓴다. 세 번째 element는 offset 2부터 쓴다. 마지막 element는 offset 3부터 3개를 쓴다.

즉 exclusive scan 결과는 각 element의 output start index가 된다.

### 4.2 Two-pass GPU Pattern

실무적인 compute shader pipeline에서는 보통 다음 흐름이 사용된다.

```text
Pass 1: Count generation
  - 각 element가 몇 개의 output을 만들지 계산
  - countBuffer[elementID]에 저장

Pass 2: Prefix sum / scan
  - countBuffer를 scan해서 offsetBuffer 생성
  - 마지막 값으로 total output count 계산

Pass 3: Output generation
  - offsetBuffer[elementID]를 읽고 outputBuffer에 write
```

이 구조는 Marching Cubes뿐 아니라 GPU particle system에서도 그대로 반복된다.

```text
Particle alive flag → prefix sum → compact alive particle buffer
```

### 4.3 Workgroup-local Scan

GPU에서는 전체 배열을 한 번에 scan하기 어렵기 때문에 보통 계층적으로 처리한다.

```text
1. 각 workgroup 내부에서 local scan
2. workgroup별 partial sum 저장
3. partial sum 배열을 다시 scan
4. 각 workgroup 결과에 group offset 추가
```

이 구조는 GPU memory hierarchy와 직접 연결된다.

- workgroup shared memory: 빠른 local scan
- storage buffer / global memory: 전체 결과 저장
- barrier: local synchronization
- dispatch boundary: global synchronization 역할

Vulkan, WebGPU, OpenGL compute shader에서는 workgroup 내부 barrier는 가능하지만 전체 dispatch 내부의 global barrier는 제한적이다. 그래서 큰 scan은 여러 dispatch pass로 나누어야 한다.

### 4.4 Atomic Add와의 비교

Atomic add 방식은 간단하다.

```text
index = atomicAdd(globalCounter, outputCount)
write output at index
```

이 방식은 output이 적거나 prototype 단계에서는 유용하다. 하지만 active thread가 많아질수록 global counter contention이 커지고, output order가 thread scheduling에 의존할 수 있다.

Prefix Sum 방식은 pass 수가 늘어나지만 대규모 데이터에서 더 안정적이다.

```text
count → scan → offset → write
```

Graphics engineer 관점에서는 “무조건 prefix sum이 정답”이 아니라, 데이터 크기와 update frequency, deterministic output 필요성, 구현 복잡도에 따라 선택해야 한다.

### 4.5 Rendering Pipeline과 연결

Prefix Sum 결과는 단순 output buffer offset에만 쓰이지 않는다. 최종적으로 draw call을 GPU-driven으로 만들기 위한 indirect argument 생성에도 연결된다.

```text
visible object count
        ↓
prefix sum / compaction
        ↓
visible object list
        ↓
indirect draw argument
        ↓
GPU-driven rendering
```

현대 rendering engine에서 CPU가 매 프레임 object list를 직접 만들기보다, GPU가 visibility를 판단하고 compact list를 만든 뒤 indirect draw로 넘기는 구조가 중요해지고 있다.

## 5. 내 관심 분야와 연결

### Marching Cubes

이전 노트의 GPU Marching Cubes compaction에서 prefix sum은 cell별 triangle offset을 만드는 핵심 단계다. cell마다 triangle 개수가 다르기 때문에 prefix sum 없이는 안정적인 output buffer layout을 만들기 어렵다.

### Particle Simulation

GPU particle simulation에서는 dead particle removal, alive particle compaction, emitter allocation에 prefix sum이 사용된다. 특히 fluid particle이나 VFX particle이 많을수록 compact alive buffer는 memory bandwidth와 draw cost를 줄이는 데 중요하다.

### Tiled / Clustered Lighting

각 screen tile 또는 cluster마다 영향을 주는 light 개수가 다르다. light index list를 만들 때 tile별 light count를 계산하고 prefix sum을 통해 global light list offset을 만든다.

### Sparse Voxel / Sparse Volume

Sparse voxel allocation에서도 active voxel, active block, active brick을 compact list로 만드는 과정이 필요하다. 반도체 3D visualization이나 CFD volume visualization에서 empty space를 건너뛰려면 prefix sum 기반 compaction이 핵심이 된다.

### WebGPU / Vulkan / CUDA

CUDA는 CUB 같은 scan primitive를 활용할 수 있어 구현 부담이 낮다. Vulkan이나 WebGPU에서는 직접 compute shader scan을 설계해야 하며, buffer barrier와 dispatch sequencing을 명확히 이해해야 한다.

## 6. 머릿속에 남길 질문 3개

1. GPU에서 variable-sized output이 발생할 때 왜 threadID만으로 output index를 정할 수 없는가?
2. Exclusive scan 결과가 왜 output buffer의 start offset으로 사용되기 좋은가?
3. Atomic add 방식과 prefix sum 방식은 각각 어떤 데이터 크기와 상황에서 더 적합한가?

## 7. Graphics Engineer 면접 질문 1개와 답변

### Q. Prefix Sum이 graphics pipeline에서 어디에 사용되나요?

A. Prefix Sum은 GPU에서 element마다 output 개수가 다른 경우, 각 element가 output buffer의 어느 위치에 결과를 쓸지 결정하는 데 사용됩니다. 예를 들어 Marching Cubes에서는 cell마다 생성하는 triangle 수가 다르기 때문에 triangle count 배열을 prefix sum하여 cell별 output offset을 만듭니다. Particle system에서는 alive particle만 compact buffer로 모으는 데 사용되고, tiled lighting에서는 tile별 light list offset을 만드는 데 사용됩니다. 즉 Prefix Sum은 stream compaction과 GPU-driven rendering pipeline의 핵심 building block입니다.

## 8. 포트폴리오 / 커리어 연결

이 개념은 포트폴리오에서 다음처럼 표현할 수 있다.

```text
Designed GPU stream compaction workflows using prefix-sum based offset generation for variable-sized outputs in rendering and visualization pipelines.
```

또는 Marching Cubes / visualization 프로젝트에서는 다음처럼 말할 수 있다.

```text
Built a compute-driven isosurface extraction pipeline with count generation, parallel scan, compact triangle output, and GPU-renderable buffer generation.
```

이 표현은 단순히 알고리즘을 공부했다는 의미가 아니라, GPU memory layout과 compute-to-render pipeline을 이해하고 있다는 신호가 된다.

## 9. 내일 이어서 볼 개념

**GPU Stream Compaction**

다음 개념은 Prefix Sum을 실제로 사용하는 대표 패턴인 stream compaction이다. active particle, visible object, active voxel, valid triangle을 compact list로 만드는 구조를 중심으로 본다.

## 10. 참고 키워드

- Prefix Sum
- Parallel Scan
- Exclusive Scan
- Inclusive Scan
- Stream Compaction
- GPU Compute
- Compute Shader
- Workgroup Shared Memory
- Atomic Add
- GPU-driven Rendering
- Indirect Draw
- Particle Compaction
- Tiled Lighting
- Sparse Voxel Allocation
- Marching Cubes Compaction
