---
title: "Wavefront Queue Scheduling and Path-State Binning: Divergence Control for Irregular ReSTIR Workloads"
date: "2026-08-14"
category: Graphics
tags: [GPU, Wavefront Rendering, Path Tracing, ReSTIR, SIMT, Warp Divergence, Work Queue, Binning, Compaction, Memory Layout, C++, Vulkan, DirectX 12]
level: intermediate
---

# [Daily Graphics Growth] 2026-08-14 - Wavefront Queue Scheduling and Path-State Binning: Divergence Control for Irregular ReSTIR Workloads

## 1. 오늘의 개념

어제의 **persistent subpath pool / reservoir indirection**은 temporal ReSTIR가 참조하는 path state를 GPU 메모리에 안전하게 유지하는 문제였다. 오늘은 그 state를 실제 GPU가 어떻게 효율적으로 처리할지, 즉 **wavefront queue scheduling**과 **path-state binning**을 본다.

Path tracing과 ReSTIR PT는 매우 불규칙한(irregular) workload다. 어떤 ray는 즉시 miss로 끝나고, 다른 ray는 여러 bounce를 진행한다. 같은 bounce에서도 diffuse, glossy, emissive, transmissive, medium 등 서로 다른 code path가 발생한다. ReSTIR PT의 hybrid shift가 추가되면 compatibility fail, random replay, reconnection, visibility revalidation처럼 비용이 다른 작업까지 섞인다.

하나의 큰 **megakernel**에서 모든 경우를 처리하면 warp 내부 thread들이 서로 다른 branch를 타며 **warp divergence**가 발생하고, 많은 local state가 동시에 살아 있어 **register pressure**와 occupancy 저하가 생길 수 있다.

Wavefront rendering은 이를 stage별 work queue로 분리한다.

```text
Ray Queue -> Trace -> Surface/Miss Queues
Surface Queue -> Material Bins -> BSDF/NEE
BSDF -> Shadow Queue + Next-Ray Queue
ReSTIR Reuse -> Shift-Class Bins -> Replay/Reconnection/Visibility
```

핵심은 단순히 kernel을 나누는 것이 아니라 **비슷한 실행 특성을 가진 path를 같은 queue/bin에 모아 같은 warp가 유사한 일을 하도록 재배열하는 것**이다.

## 2. 한 줄 핵심

> **Wavefront scheduling은 불규칙한 path를 execution state별 queue와 bin으로 재배열해 divergence와 register pressure를 줄이는 대신, queue traffic·compaction·kernel scheduling 비용을 지불하는 GPU 실행 전략이다.**

## 3. 왜 중요한가

### SIMT divergence

warp 안의 thread들이 서로 다른 branch를 수행하면 일부 lane은 비활성 상태로 같은 instruction stream을 기다리게 된다. 단순화한 lane 효율은 다음처럼 생각할 수 있다.

\[
\eta_{lane}=\frac{\text{useful active lane-instructions}}{\text{issued lane-instructions}}
\]

material과 bounce 상태가 섞일수록 `η_lane`이 낮아질 수 있다. Wavefront는 diffuse work, glossy work, medium work 등을 별도 queue에 모아 coherence를 높인다.

### Register pressure

큰 megakernel은 intersection, BSDF, reservoir, shift mapping, visibility, RNG 등 여러 branch의 state를 한 kernel에 끌어안기 쉽다. thread당 register가 커지면 SM에 동시에 resident할 수 있는 warp 수가 줄어든다.

\[
N_{resident}\le\min(N_{thread},N_{register},N_{shared},N_{block})
\]

Wavefront decomposition은 stage마다 필요한 state만 유지해 register live range를 줄일 여지가 있다.

### ReSTIR PT의 추가 irregularity

ReSTIR PT에서는 같은 spatial/temporal reuse pass 안에서도 candidate마다 비용이 크게 다르다. 어떤 candidate는 surface compatibility test에서 바로 reject되지만, 다른 candidate는 여러 bounce random replay, reconnection, visibility ray까지 수행할 수 있다. 따라서 단순한 pass 단위 grouping만으로는 충분하지 않고 **shift complexity** 자체가 scheduling 기준이 될 수 있다.

### 비용은 공짜가 아니다

Wavefront에서는 stage 사이의 state를 global memory queue에 기록하고 다음 kernel에서 다시 읽는다.

\[
T_{wavefront}=T_{compute}+T_{queue}+T_{compaction}+T_{launch}+T_{barrier}
\]

반대로 megakernel은 대략

\[
T_{mega}=T_{compute}+T_{divergence}+T_{register/occupancy}
\]

로 볼 수 있다. 따라서 wavefront가 항상 더 빠른 것은 아니다. scene complexity, material 다양성, path depth, GPU architecture, queue layout에 따라 trade-off가 달라진다.

NVIDIA의 *Megakernels Considered Harmful*은 복잡한 material workload에서 wavefront formulation이 divergence와 register pressure를 줄여 실전적인 이점을 가질 수 있음을 보였다. PBRT v4 GPU integrator 역시 ray, emissive hit, material, medium, shadow ray 등을 queue 기반 stage로 분리한다.

## 4. 구현 관점

### Stage graph

Wavefront renderer는 함수 호출보다 **stage graph**로 보는 편이 명확하다.

```text
Primary Ray
  -> Trace
  -> Surface Queue / Miss Queue
  -> Material Evaluation
  -> Shadow Queue / Indirect Queue
  -> Visibility / Next Bounce
```

ReSTIR PT에서는 다음 stage가 추가된다.

```text
Initial Path
 -> Reservoir Construction
 -> Temporal Reuse
 -> Spatial Reuse
 -> Final Shading
```

그리고 reuse 내부에서도 `compatibility rejected`, `short replay`, `reconnection`, `long replay`, `visibility required`처럼 execution class가 갈릴 수 있다.

### Hot state와 cold state

모든 queue가 full path payload를 복사하면 bandwidth가 커진다. 따라서 queue에는 현재 stage에서 자주 쓰는 **hot state**만 두고, 큰 subpath payload는 어제 다룬 persistent pool을 통해 간접 참조하는 구조가 자연스럽다.

Hot state 예:

```text
ray origin / direction
throughput
pixel/path ID
path depth
material class
reservoir handle
flags
```

Cold state 예:

```text
full subpath vertices
historical provenance
large material payload
reconnection metadata
```

중요한 구분은 다음과 같다.

```text
pathId     = logical identity
queueSlot  = temporary execution location
poolHandle = persistent payload identity
```

Compaction으로 queue slot이 바뀌어도 logical path와 reservoir provenance는 유지되어야 한다.

### SoA와 coalescing

PBRT v4의 work queue는 **SoA(Structure of Arrays)** layout을 사용한다. 같은 warp가 동일 field를 읽을 때 연속 주소 접근을 만들기 쉽기 때문이다.

```text
origin[]
direction[]
throughput[]
materialClass[]
pathId[]
```

GPU global memory는 warp의 가까운 주소 접근을 적은 transaction으로 coalesce할 수 있으므로 queue layout은 software 구조이면서 동시에 memory-system 최적화다.

### Queue append와 warp-aggregated enqueue

가장 단순한 append는 global counter에 대한 `atomicAdd`다. 더 scalable한 개념은 warp 안에서 enqueue할 lane 수를 먼저 세고 warp당 한 번만 global counter를 증가시키는 것이다.

\[
N_{warp}=\sum_i active_i
\]

warp leader가 base slot을 얻고 각 lane은 prefix rank를 더해 연속 slot을 받는다. 이 방식은 atomic 수를 줄이고 같은 warp의 output을 contiguous하게 배치하는 데 도움이 된다.

### Compaction과 occupancy의 균형

bounce가 깊어질수록 active path는 감소한다. compact queue는 종료된 path를 제거해 dispatch size를 실제 active work에 맞춘다.

\[
N_{dispatch}\approx N_{active}
\]

하지만 bin을 너무 잘게 쪼개면 각 queue의 work가 작아져 GPU를 충분히 채우지 못한다. 따라서 **coherence gain vs. available parallelism**의 균형이 필요하다.

### Material binning과 shift-class binning

material binning은 exact material ID보다 execution cost가 비슷한 **execution class** 기준이 더 유용할 수 있다.

```text
simple opaque
complex layered
alpha-tested
transmissive
volume
```

ReSTIR PT에서도 비슷하게 shift work를 다음처럼 분류할 수 있다.

```text
compatibility reject
short random replay
reconnection candidate
long replay
visibility revalidation
```

단, 이 binning은 sample probability를 바꾸는 estimator 단계가 아니라 **이미 결정된 work의 execution order를 바꾸는 scheduling layer**여야 한다.

\[
\text{Estimator Semantics} \neq \text{Execution Scheduling}
\]

### C++ render graph 관점

queue는 GPU resource이자 synchronization boundary다.

```text
TracePass writes SurfaceQueue
MaterialPass reads SurfaceQueue, writes ShadowQueue
VisibilityPass reads ShadowQueue, writes LightingBuffer
```

각 queue에는 data buffer, counter, capacity, optional bin offsets, overflow state가 필요하다. Vulkan/DirectX 12에서는 shader write -> shader read barrier뿐 아니라 queue count를 indirect dispatch argument로 쓴다면 compute write -> indirect argument read dependency도 execution contract의 일부다.

### Profiling 지표

Execution coherence:
- active lanes per warp
- branch/warp execution efficiency
- divergent branch count

Kernel pressure:
- registers per thread
- achieved occupancy
- active warps per SM
- local-memory spill

Queue/memory:
- queue peak occupancy
- bytes read/written per path
- enqueue atomic count
- L2 hit rate
- compaction ratio
- tiny-dispatch count
- overflow count

ReSTIR-specific:
- shift success ratio
- average replay depth
- reconnection ratio
- visibility revalidation ratio
- work items per shift class

이 지표를 함께 보면 “ReSTIR가 느리다”가 아니라 “long-replay bin이 전체 path의 8%지만 spatial reuse GPU time의 30% 이상을 소비한다”처럼 병목을 구체화할 수 있다.

## 5. 내 관심 분야와 연결

### GPU compute / CUDA

Wavefront scheduling은 path tracing에만 국한되지 않는다. irregular simulation에서도 active/inactive/boundary/collision/refinement state를 compact하고 execution class별 kernel로 처리하는 구조가 동일하게 나타난다.

### CFD / scientific visualization

GPU streamline tracing은 각 particle이 domain exit, stagnation, adaptive step, block migration 등 서로 다른 상태를 가지므로 path tracing과 구조적으로 비슷하다. alive particle compaction, block-based binning, adaptive-step classification이 coherence에 직접 영향을 준다.

### Sparse voxel / level-set

Level-set narrow band나 sparse brick processing도 전체 volume 대신 active voxel/brick list를 compact해 처리한다. 여기서 stable logical ID, temporary queue slot, SoA, compaction, indirect dispatch 개념이 그대로 이어진다.

### WebGPU / modern graphics API

WebGPU에서도 multi-stage compute queue 구조는 가능하지만 native CUDA보다 dispatch/command overhead가 상대적으로 더 크게 느껴질 수 있다. 따라서 지나치게 세밀한 stage 분할보다 coherence gain과 intermediate buffer traffic의 균형이 중요하다.

### 게임 엔진

이 개념은 ray tracing 외에도 GPU-driven rendering, meshlet culling, clustered lighting, particles, async compute, device-generated work와 연결된다. 공통 질문은 **수백만 개의 heterogeneous work item을 GPU가 가장 coherent하게 처리하도록 어떻게 재배열할 것인가**다.

## 6. 머릿속에 남길 질문 3개

1. **Wavefront renderer에서 divergence 감소 이득과 queue global-memory traffic 증가 비용을 어떤 profiler 지표 조합으로 비교해야 할까?**
2. **ReSTIR PT hybrid shift처럼 work cost 편차가 큰 경우 estimator semantics를 건드리지 않으면서 어떤 state를 binning key로 삼는 것이 가장 효과적일까?**
3. **Queue를 세분화해 kernel coherence는 높아졌지만 queue당 work가 작아졌을 때 어느 시점부터 occupancy와 launch overhead가 더 큰 병목이 될까?**

## 7. graphics engineer 면접 질문 1개와 답변

### 질문

**GPU path tracer에서 megakernel 대신 wavefront architecture를 사용하면 왜 성능이 좋아질 수 있으며, 반대로 더 느려질 수 있는 경우는 무엇인가요?**

### 답변

Megakernel은 traversal, material, BSDF, shadow test, path continuation을 한 kernel에서 처리하므로 intermediate state를 register에 유지하고 kernel launch 및 queue traffic을 줄일 수 있다. 그러나 path tracing은 thread마다 material, hit/miss, bounce depth가 달라 SIMT divergence가 커질 수 있고, 큰 kernel은 register pressure 증가로 occupancy를 낮출 수 있다.

Wavefront architecture는 작업을 stage별 queue로 분리하고 비슷한 work를 같은 kernel에서 처리해 branch coherence를 높이며 stage별 register footprint를 줄일 수 있다. Material/path-state binning을 추가하면 같은 warp가 더 유사한 code path를 실행하게 된다.

반대로 stage 사이의 state를 global memory에 저장하고 다시 읽기 때문에 bandwidth와 cache traffic이 늘고, queue append·atomic·compaction·barrier·kernel launch 비용이 생긴다. binning을 너무 세밀하게 하면 queue별 parallelism이 줄어 GPU utilization이 떨어질 수도 있다.

따라서 핵심 trade-off는 다음과 같다.

\[
\text{Divergence + Register Pressure}
\quad vs. \quad
\text{Queue Traffic + Scheduling Overhead}
\]

복잡한 material과 긴 path처럼 divergence가 큰 workload에서는 wavefront의 장점이 커지는 경향이 있고, 단순하고 coherent한 workload에서는 megakernel이 더 효율적일 수 있다.

## 8. 포트폴리오 / 커리어 연결

이 주제는 “path tracer 구현”보다 **GPU execution architecture를 설계하고 프로파일링했다**는 포트폴리오 스토리로 연결하기 좋다.

강한 설명 구조는 기능 목록보다 다음과 같다.

```text
Problem
- material/path divergence 증가
- monolithic kernel register pressure 증가

Architecture
- trace/material/visibility/continuation queue 분리
- path ID와 queue slot 분리
- hot-state SoA + cold-state indirection
- material/shift execution-class binning

Measurement
- branch efficiency
- achieved occupancy
- queue traffic
- bytes/path
- L2 hit rate
- stage별 GPU time
```

이 구조를 설명할 수 있으면 단순 shader 작성이 아니라 **renderer architecture, memory layout, synchronization, profiler를 함께 다루는 graphics engineer** 역량을 보여줄 수 있다.

## 9. 내일 이어서 볼 개념

### GPU-Driven Work Graphs and Persistent Threads: Device-Side Scheduling for Irregular Rendering Pipelines

오늘은 host가 여러 wavefront queue와 kernel stage를 순서대로 dispatch하는 구조를 보았다. 다음에는 **GPU 자체가 다음 work를 생성하고 scheduling하는 구조**로 확장한다.

핵심 연결 키워드는 persistent threads, indirect dispatch, device-side work generation, work graph/shader execution graph, queue draining, dynamic load balancing, producer-consumer scheduling이다.

## 10. 참고 키워드

- Wavefront Path Tracing
- Megakernel Path Tracing
- SIMT / Warp Divergence
- Register Pressure / Occupancy
- Work Queue / Work Item
- Queue Compaction / Stream Compaction
- Prefix Sum / Scan
- Warp-Aggregated Atomics
- Material Binning / Path-State Binning
- Execution Class
- Structure of Arrays (SoA)
- Global Memory Coalescing
- Hot / Cold Data Split
- Persistent Path State
- Stable Handle / Reservoir Indirection
- ReSTIR PT / Hybrid Shift / Random Replay / Reconnection
- Indirect Dispatch / GPU-Driven Rendering
- Persistent Threads / GPU Work Graphs
- NVIDIA, *Megakernels Considered Harmful: Wavefront Path Tracing on GPUs*
- PBRT v4, *Wavefront Rendering on GPUs*
- NVIDIA RTXDI / ReSTIR PT