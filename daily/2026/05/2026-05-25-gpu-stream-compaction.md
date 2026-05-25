---
title: "GPU Stream Compaction"
date: "2026-05-25"
category: "Graphics"
tags: ["GPU", "Stream Compaction", "Compute Shader", "Prefix Sum", "GPGPU", "Rendering Pipeline"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-05-25 - GPU Stream Compaction

## 1. 오늘의 개념

GPU Stream Compaction은 대량의 GPU 데이터 중 “유효한(active) 데이터만 조밀하게 다시 배치하는 과정”이다.

예를 들어 particle system에서 죽은 particle을 제거하거나, visibility culling 이후 visible object만 남기거나, Marching Cubes에서 triangle을 생성하는 active cell만 모으는 작업이 모두 stream compaction에 해당한다.

핵심은 sparse한 입력 데이터를 GPU-friendly한 compact buffer로 바꾸는 것이다.

```text
Sparse Input Buffer
    ↓
Active Flag Generation
    ↓
Prefix Sum / Scan
    ↓
Compacted Output Buffer
```

## 2. 한 줄 핵심

**Stream Compaction은 GPU pipeline에서 “실제로 필요한 데이터만 남기는 메모리 재배치 과정”이다.**

## 3. 왜 중요한가

GPU는 massive parallel hardware이지만, 실제 성능은 memory bandwidth와 cache coherence 영향을 크게 받는다.

inactive particle, invisible mesh, empty voxel처럼 필요 없는 데이터를 계속 순회하면:

- memory bandwidth 낭비
- cache pollution
- branch divergence
- draw call 증가
- unnecessary compute workload

가 발생한다.

그래서 현대 GPU rendering과 simulation에서는:

```text
가능한 빨리
불필요한 데이터를 제거
```

하는 것이 중요하다.

Stream Compaction은 이를 가능하게 만드는 핵심 primitive다.

## 4. 구현 관점

### 4.1 Active Flag 생성

먼저 각 element가 살아있는지 판단한다.

```text
aliveFlag[i] = particleLife[i] > 0 ? 1 : 0
```

또는:

- visible mesh = 1
- active voxel = 1
- valid triangle = 1
- alive particle = 1

처럼 사용할 수 있다.

### 4.2 Prefix Sum

aliveFlag 배열에 exclusive scan을 수행한다.

```text
flag:   [1, 0, 1, 1, 0]
offset: [0, 1, 1, 2, 3]
```

offset은 각 active element가 compact buffer의 어느 위치에 들어갈지 알려준다.

### 4.3 Compact Write

active element만 compact buffer로 복사한다.

```text
if aliveFlag[i]:
    compactBuffer[offset[i]] = input[i]
```

결과:

```text
input:   [A, X, B, C, X]
compact: [A, B, C]
```

GPU 입장에서는 이후 pipeline이 훨씬 효율적인 dense workload가 된다.

### 4.4 GPU Rendering Pipeline 연결

Stream Compaction은 GPU-driven rendering의 핵심이다.

```text
Visibility Test
    ↓
Visible Object Compaction
    ↓
Indirect Draw Buffer
    ↓
GPU-driven Rendering
```

즉 CPU가 visible object list를 만들지 않고, GPU가 직접 visible list를 생성한다.

### 4.5 Particle Simulation

Particle simulation에서는 dead particle 제거가 대표 사례다.

```text
All Particle Buffer
    ↓
Alive Particle Compaction
    ↓
Compact Alive Buffer
    ↓
Simulation / Rendering
```

alive particle만 대상으로 simulation하면 bandwidth와 compute cost를 줄일 수 있다.

## 5. 내 관심 분야와 연결

### CFD / Fluid Simulation

SPH fluid simulation에서 active particle만 compact buffer로 관리하면 simulation cost를 줄일 수 있다.

### Marching Cubes

surface를 포함하는 active cell만 compact list로 만들면 empty voxel traversal 비용을 크게 줄일 수 있다.

### Sparse Voxel / NanoVDB

sparse block traversal에서도 active brick만 compact list로 만드는 과정이 중요하다.

### WebGPU / Vulkan

WebGPU와 Vulkan compute pipeline에서는 stream compaction이 GPU-driven architecture의 핵심 building block이다.

## 6. 머릿속에 남길 질문 3개

1. GPU는 왜 sparse workload보다 compact workload에서 더 효율적인가?
2. Stream Compaction과 Prefix Sum은 왜 거의 항상 함께 등장하는가?
3. GPU-driven rendering에서 visible object compaction은 CPU workload를 어떻게 줄이는가?

## 7. Graphics Engineer 면접 질문 1개와 답변

### Q. GPU Stream Compaction은 어디에 사용되나요?

A. GPU Stream Compaction은 active element만 compact buffer로 재배치할 때 사용됩니다. 예를 들어 particle system의 alive particle 관리, Marching Cubes의 active cell extraction, tiled lighting의 visible light list 생성, GPU-driven rendering의 visible object list 생성 등에 사용됩니다. 핵심은 sparse workload를 dense workload로 바꿔 memory bandwidth와 compute 효율을 높이는 것입니다.

## 8. 포트폴리오 / 커리어 연결

포트폴리오에서는 다음처럼 표현할 수 있다.

```text
Implemented GPU stream compaction pipelines for active-particle filtering and compact renderable buffer generation using prefix-sum based offset allocation.
```

또는:

```text
Designed GPU-driven visibility compaction workflows for scalable rendering of sparse simulation datasets.
```

## 9. 내일 이어서 볼 개념

GPU-driven Rendering and Indirect Draw Pipeline

## 10. 참고 키워드

- Stream Compaction
- Prefix Sum
- Parallel Scan
- GPU Compute
- Compute Shader
- GPU-driven Rendering
- Indirect Draw
- Particle Simulation
- Marching Cubes
- Sparse Voxel
- NanoVDB
