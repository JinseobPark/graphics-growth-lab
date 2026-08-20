---
title: "State Locality Restoration After Compaction: Morton/Material Binning, Handle Remapping, and Cache-Coherent Path Pools"
date: "2026-08-20"
category: Graphics
tags: [GPU, Memory Locality, Compaction, Morton Order, Material Binning, Handle Remapping, C++]
level: intermediate
---

# [Daily Graphics Growth] 2026-08-20 - State Locality Restoration After Compaction: Morton/Material Binning, Handle Remapping, and Cache-Coherent Path Pools

## 1. 오늘의 개념
Compaction 이후에도 physical PathState가 흩어지면 token indirection 뒤의 memory access는 비연속적일 수 있다. 오늘은 stable logical handle을 유지한 채 physical slot을 재배치해 cache locality와 execution coherence를 회복하는 방법을 본다.

## 2. 한 줄 핵심
> Compaction은 빈 공간을 없애지만 locality를 보장하지 않으므로, handle과 physical slot을 분리하고 workload에 맞는 ordering으로 backing store를 재배치해야 한다.

## 3. 왜 중요한가
Queue가 작은 `pathToken`으로 조밀해도 `pathToken -> handle table -> physical slot`이 서로 먼 주소를 가리키면 같은 warp의 backing-store load가 scattered해질 수 있다. CUDA에서 coalesced global-memory access가 중요한 이유와 연결된다. Morton/Z-order는 spatial locality를, material binning은 shader/BSDF execution coherence를 높이는 서로 다른 heuristic이다. 정렬 자체에도 key generation, scan, scatter 비용이 있으므로 end-to-end frame time으로 판단해야 한다.

## 4. 구현 관점
핵심 invariant는 `logical identity != physical location`이다. `PathToken -> HandleTable -> PhysicalSlot` 구조를 쓰면 compaction/reordering 후에도 token을 유지할 수 있다. 재배치에서는 `oldSlot -> newSlot` remap과 `newSlot -> oldSlot` permutation을 구분해야 한다. Locality key는 `[stage | materialClass | depthBucket | mortonXY]`처럼 계층화할 수 있으며 priority에 따라 execution locality와 spatial locality의 비중이 달라진다. SoA는 field layout을, reordering은 entity placement를 해결하므로 서로 보완적이다. Hot state는 자주 reorder하고 cold state는 비교적 안정적으로 유지하는 계층화도 가능하다. SER는 runtime execution coherence, wavefront binning은 queue/order coherence, physical pool reordering은 data-placement coherence를 다룬다.

## 5. 내 관심 분야와 연결
C++에서는 `PathHandle`, `PoolGeneration`, `PhysicalSlot`을 별도 타입으로 표현하면 relocation contract가 명확해진다. CUDA/compute에서는 coalescing, L2 locality, radix sort, prefix sum, scatter가 하나의 dataflow로 연결된다. Particle, sparse voxel, active-cell, semiconductor 3D visualization에서도 logical ID와 physical slot을 분리하면 compaction과 stable reference를 동시에 관리하기 쉽다.

## 6. 머릿속에 남길 질문 3개
1. Compaction 이후에도 memory performance가 나빠질 수 있는 이유는 무엇인가?
2. Material-first와 Morton-first ordering은 각각 어떤 coherence를 개선하는가?
3. Stable handle과 physical slot을 분리했을 때 얻는 architecture-level 장점은 무엇인가?

## 7. graphics engineer 면접 질문 1개와 답변
### 질문
GPU wavefront renderer에서 active queue를 compact했는데 shading이 빨라지지 않았다. memory layout 관점에서 원인을 설명하라.

### 답변
Queue 자체는 dense해졌어도 token이 random physical PathState를 가리키면 backing-store load가 scattered할 수 있다. 또한 material/BSDF가 섞여 있으면 shader divergence도 유지된다. 따라서 queue locality와 backing-store locality를 분리해 보고, workload에 따라 material binning 또는 Morton/spatial ordering을 적용할 수 있다. Stable handle을 유지한 채 physical pool만 remap하면 temporal reference를 깨지 않고 재배치할 수 있다. 단, sort/bin/scatter 비용까지 포함한 전체 pass 시간이 최종 판단 기준이다.

## 8. 포트폴리오 / 커리어 연결
이 주제는 graphics systems 관점을 보여준다. active compaction과 locality restoration을 분리해 설명하고, relocation-safe handle architecture, SoA/AoSoA, material-vs-spatial coherence trade-off, render-graph synchronization, end-to-end profiling을 하나의 시스템으로 연결할 수 있다.

## 9. 내일 이어서 볼 개념
**GPU Data-Oriented Scene State: Sparse Set, Generational Handles, and Multi-Buffer Residency for Rendering/Simulation Interop**

Path-state pool에서 배운 logical handle/physical layout 분리를 scene-wide data-oriented architecture로 확장한다.

## 10. 참고 키워드
Morton Code, Z-Order Curve, Material Binning, Handle Remapping, Relocation Table, SoA, AoSoA, Coalesced Global Memory, L2 Locality, Radix Sort, Prefix Sum, Scatter, Wavefront Path Tracing, SER, Render Graph Resource Lifetime, CUDA C++ Best Practices Guide.
