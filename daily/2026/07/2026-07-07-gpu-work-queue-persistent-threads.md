---
title: "GPU Work Queue and Persistent Threads"
date: "2026-07-07"
category: "Graphics"
tags: ["GPU", "Compute Shader", "Work Queue", "Persistent Threads", "Dynamic Workload", "GPU-Driven Rendering", "Ray Tracing", "Sparse Voxel", "Scientific Visualization"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-07-07 - GPU Work Queue and Persistent Threads

## 1. 오늘의 개념

**GPU Work Queue and Persistent Threads**는 GPU에서 처리해야 할 작업량이 균일하지 않거나, 실행 중에 새로운 작업이 생기는 경우에 사용하는 dynamic workload scheduling 구조다.

전통적인 GPU compute dispatch는 다음처럼 생각하기 쉽다.

```text
N개의 thread를 실행한다.
각 thread는 자신의 index에 해당하는 일을 한 번 처리한다.
끝난다.
```

이 방식은 모든 element의 작업량이 비슷할 때 잘 맞는다. 예를 들어 모든 pixel에 같은 shader를 실행하거나, 모든 particle에 비슷한 연산을 수행하는 경우다.

하지만 graphics / simulation / visualization에서는 작업량이 자주 불균일하다.

- 어떤 ray는 금방 끝나고, 어떤 ray는 많은 voxel brick을 통과한다.
- 어떤 meshlet은 culling되고, 어떤 meshlet은 많은 primitive를 만든다.
- 어떤 voxel brick은 empty이고, 어떤 brick은 marching cubes triangle이 많이 나온다.
- 어떤 particle은 neighbor가 적고, 어떤 particle은 neighbor가 많다.
- 어떤 field block은 단순하고, 어떤 block은 high-gradient라 더 많은 sampling이 필요하다.

이때 work queue 구조는 GPU buffer 안에 처리할 작업을 넣고, 여러 worker thread 또는 workgroup이 queue에서 작업을 가져가 처리한다.

핵심 변화는 다음이다.

> Thread index가 곧 작업 index인 구조에서, GPU worker가 queue에서 다음 작업을 동적으로 가져가는 구조로 이동한다.

Persistent Threads는 이런 구조에서 thread 또는 workgroup이 하나의 task만 처리하고 종료하지 않고, queue가 빌 때까지 반복적으로 작업을 가져가 처리하는 방식이다.

## 2. 한 줄 핵심

**GPU Work Queue and Persistent Threads는 불균일하거나 동적으로 증가하는 GPU workload를 queue에 넣고, worker thread가 atomic counter로 작업을 가져가 처리하게 만드는 dynamic scheduling 구조다.**

## 3. 왜 중요한가

GPU는 많은 thread를 병렬로 실행하는 데 강하지만, 모든 thread가 같은 양의 일을 한다는 보장은 없다. Workload imbalance가 크면 일부 thread는 빨리 끝나 idle 상태가 되고, 일부 thread만 오래 실행해 전체 성능이 떨어진다.

Work queue와 persistent threads가 중요한 이유는 다음과 같다.

- 불균일한 작업량을 동적으로 분배할 수 있다.
- GPU ray tracing / path tracing에서 ray workload imbalance를 줄일 수 있다.
- sparse voxel / volume traversal에서 긴 ray와 짧은 ray를 더 균형 있게 처리할 수 있다.
- marching cubes / active cell processing에서 variable output workload를 관리할 수 있다.
- GPU-driven renderer에서 visible task를 queue로 연결할 수 있다.
- CPU readback 없이 GPU 내부에서 다음 작업을 생성하고 소비할 수 있다.

Graphics engineer 관점에서는 work queue가 단순 병렬 loop를 넘어, **GPU 내부에서 task scheduling과 producer-consumer pipeline을 구성하는 방식**이라는 점이 핵심이다.

## 4. 구현 관점

### 4.1 Static dispatch의 한계

일반적인 compute shader는 thread id로 작업을 결정한다.

```glsl
uint id = gl_GlobalInvocationID.x;
ProcessItem(id);
```

모든 item의 비용이 비슷하면 좋다. 하지만 item마다 비용이 다르면 warp/wave divergence와 load imbalance가 생긴다.

예를 들어 voxel ray marching에서 어떤 pixel은 빈 공간만 지나 금방 끝나고, 어떤 pixel은 dense volume 내부에서 많은 step을 수행한다. 이때 static dispatch는 긴 작업에 전체 dispatch 시간이 묶일 수 있다.

### 4.2 Work queue의 기본 구조

Work queue는 처리할 task를 GPU buffer에 저장한다.

```cpp
struct Task
{
    uint type;
    uint dataIndex;
    uint lodLevel;
    uint flags;
};

RWStructuredBuffer<Task> taskQueue;
RWStructuredBuffer<uint> queueCounter;
```

Worker thread는 atomic counter로 다음 task index를 가져온다.

```glsl
while (true)
{
    uint taskIndex = atomicAdd(queueHead, 1);
    if (taskIndex >= taskCount)
        break;

    Task task = taskQueue[taskIndex];
    ProcessTask(task);
}
```

이 구조에서는 thread가 하나의 task를 끝내면 바로 다음 task를 가져간다. 작업량이 불균일해도 idle 시간을 줄일 수 있다.

### 4.3 Persistent Threads란 무엇인가

Persistent Threads는 GPU에 필요한 worker 수만큼 thread 또는 workgroup을 실행하고, 이들이 queue에서 계속 일을 가져가도록 하는 방식이다.

일반 dispatch:

```text
작업 수만큼 thread 생성 → 각 thread가 하나의 작업 처리 → 종료
```

Persistent thread 방식:

```text
worker thread/workgroup 생성 → queue에서 task 가져감 → 처리 → 다시 task 가져감 → queue가 빌 때 종료
```

이 방식은 CPU의 thread pool과 비슷한 사고를 GPU에 적용한 것이다. 다만 GPU에서는 atomic, memory coherence, occupancy, wave divergence, barrier 범위가 매우 중요하다.

### 4.4 Producer-consumer pipeline

Work queue의 강력한 점은 task를 처리하면서 새로운 task를 생성할 수 있다는 점이다.

예를 들어 sparse voxel traversal에서 coarse node를 처리하다가 더 세밀한 child node가 필요하면 child task를 queue에 push할 수 있다.

```text
Process node task
  ├─ visible and needs subdivision → push child node tasks
  └─ visible and leaf → push render task
```

이 구조는 다음에 유용하다.

- hierarchical culling
- adaptive LOD traversal
- sparse voxel node traversal
- ray/path tracing bounce queue
- tile-based workload expansion
- marching cubes active brick expansion

즉 work queue는 단순 task list가 아니라 GPU 내부 producer-consumer pipeline이 될 수 있다.

### 4.5 Atomic contention과 queue design

Work queue는 atomic counter에 의존하는 경우가 많다. 모든 worker가 같은 head counter를 atomicAdd하면 contention이 생길 수 있다.

완화 방법은 다음과 같다.

- workgroup 단위로 여러 task를 batch로 가져오기
- local shared queue 사용
- multi-queue 구조 사용
- task type별 queue 분리
- coarse task를 먼저 가져오고 내부에서 workgroup이 분배
- wave-level prefix sum으로 enqueue offset 계산

예를 들어 workgroup이 한 번에 32개 task를 가져오면 atomic 횟수를 줄일 수 있다.

```glsl
uint base = atomicAdd(queueHead, 32);
uint taskIndex = base + localThreadId;
```

단, task 수가 적거나 task 비용이 매우 불균일하면 batching이 오히려 load imbalance를 만들 수 있다.

### 4.6 Queue overflow와 capacity 관리

GPU queue는 finite buffer다. Task를 동적으로 push하는 구조에서는 queue overflow를 반드시 고려해야 한다.

필요한 설계는 다음과 같다.

- maximum task count 예측
- overflow flag 기록
- fallback path 제공
- multi-pass expansion 제한
- LOD subdivision depth 제한
- task compaction / recycling
- debug counter와 statistics 저장

Overflow를 무시하면 memory corruption이나 silent rendering bug가 생길 수 있다. GPU-driven architecture에서는 이런 bug가 CPU code보다 추적하기 어렵다.

### 4.7 Ray tracing / path tracing과 연결

Path tracing에서는 ray마다 길이가 다르다. 어떤 ray는 바로 miss되고, 어떤 ray는 여러 bounce를 수행한다. 이때 persistent threads와 ray queue 구조가 유용하다.

대표 구조는 다음과 같다.

```text
primary ray queue
→ hit/miss 처리
→ secondary ray queue
→ shadow ray queue
→ next bounce queue
```

Ray를 bounce별 또는 material별로 queue에 분리하면 divergence를 줄이고 coherent한 shading을 만들 수 있다.

이 개념은 real-time ray tracing, software ray tracing, sparse voxel ray marching 모두에 연결된다.

### 4.8 GPU-driven rendering과 연결

GPU-driven renderer에서는 여러 단계가 queue로 연결될 수 있다.

```text
visible object queue
→ meshlet culling queue
→ visible meshlet queue
→ draw command queue
```

또는 sparse voxel renderer에서는 다음 구조가 가능하다.

```text
root node queue
→ visible node queue
→ child traversal queue
→ visible brick queue
→ render dispatch queue
```

이 구조에서는 CPU가 중간 결과를 읽지 않고, GPU 내부 buffer가 다음 pass의 입력이 된다. Work queue는 GPU-driven rendering의 dynamic workload backbone 역할을 할 수 있다.

## 5. 내 관심 분야와 연결

### CFD / scientific visualization

CFD visualization에서는 block마다 처리 비용이 다를 수 있다. 예를 들어 high-gradient region, vortex region, iso-surface가 많이 생기는 block, streamline이 많이 통과하는 block은 더 많은 작업을 만든다.

Work queue를 사용하면 다음 구조가 가능하다.

- visible field block queue
- active iso-surface block queue
- streamline segment processing queue
- particle trace update queue
- high-gradient region refinement queue
- volume brick ray marching queue

Scientific visualization에서는 task priority도 중요하다. 단순히 먼저 들어온 순서가 아니라, 화면 기여도나 user-selected ROI를 기준으로 queue priority를 조정할 수 있다.

### Sparse voxel / octree / NanoVDB

Sparse voxel과 octree traversal은 work queue와 매우 잘 맞는다.

- root node에서 시작
- visible node만 child task 생성
- screen-space error가 큰 node만 subdivision
- leaf brick은 render queue로 이동
- missing brick은 streaming request queue로 이동

이 구조는 recursive traversal을 GPU에서 iterative queue processing으로 바꾸는 방식이다. Shader에서 재귀 호출이 어렵거나 비효율적인 경우 queue 기반 traversal이 좋은 대안이 된다.

### Game engine architecture

Game engine에서는 work queue와 persistent threads가 다음에 연결된다.

- GPU culling pipeline
- meshlet task dispatch
- software rasterization queue
- ray tracing wavefront scheduling
- particle simulation
- async compute task graph
- virtual texture feedback processing
- GPU-driven indirect command generation

면접에서는 Persistent Threads를 “계속 도는 thread”라고만 말하기보다, “불균일한 GPU workload를 dynamic scheduling하기 위한 queue-based worker model”이라고 설명하는 것이 좋다.

## 6. 머릿속에 남길 질문 3개

1. Static dispatch가 모든 item의 작업량이 불균일할 때 GPU utilization을 떨어뜨릴 수 있는 이유는 무엇인가?
2. Work queue에서 atomic counter를 사용할 때 contention과 overflow를 어떻게 관리해야 하는가?
3. Sparse voxel octree traversal이나 path tracing에서 queue-based scheduling이 recursive/static dispatch보다 유리한 이유는 무엇인가?

## 7. graphics engineer 면접 질문 1개와 답변

### Q. GPU Work Queue와 Persistent Threads는 무엇이며 어떤 상황에서 유용한가요?

**A.** GPU Work Queue는 처리할 task를 GPU buffer에 저장하고, worker thread나 workgroup이 atomic counter를 이용해 다음 task를 가져가 처리하는 구조입니다. Persistent Threads는 thread가 하나의 작업만 처리하고 끝나는 것이 아니라, queue가 빌 때까지 반복적으로 task를 가져가 처리하는 방식입니다. 이 구조는 작업량이 item마다 크게 다른 경우, 또는 작업 중에 새로운 task가 생성되는 경우에 유용합니다.

예를 들어 path tracing에서는 ray마다 bounce 수와 traversal 비용이 다르고, sparse voxel traversal에서는 어떤 node는 바로 culling되지만 어떤 node는 child task를 여러 개 생성합니다. Static dispatch는 이런 workload imbalance를 잘 처리하지 못할 수 있습니다. Work queue 기반 구조는 idle thread가 다음 task를 가져가므로 load balancing에 유리합니다. 단점은 atomic contention, queue overflow, memory synchronization, task ordering, debugging 복잡도를 관리해야 한다는 점입니다.

## 8. 포트폴리오 / 커리어 연결

GPU Work Queue and Persistent Threads는 포트폴리오에서 다음 메시지를 만든다.

> “나는 GPU compute를 단순 one-thread-per-element 모델로만 보지 않고, dynamic workload를 queue와 persistent worker로 scheduling하는 구조를 이해한다.”

네 배경과 연결하면 다음 표현이 좋다.

- Sparse voxel / octree traversal을 recursive CPU 구조가 아니라 GPU queue traversal로 바꾸는 사고
- CFD field block이나 iso-surface block을 visible/active queue로 관리하는 구조 이해
- Particle / SPH simulation에서 불균일 neighbor workload를 dynamic scheduling하는 관점
- GPU-driven renderer에서 visible object → meshlet → draw command pipeline을 queue로 연결하는 구조
- WebGPU / Vulkan compute shader에서 storage buffer와 atomic counter 기반 work queue 설계 가능

면접에서는 다음처럼 말할 수 있다.

> “Persistent threads는 GPU에 worker를 띄워두고 queue에서 task를 계속 가져가게 하는 구조입니다. 작업량이 균일하지 않은 ray tracing, sparse voxel traversal, active cell processing, GPU-driven culling에서 load imbalance를 줄이는 데 사용할 수 있습니다.”

## 9. 내일 이어서 볼 개념

**Wavefront Path Tracing Scheduling**

GPU Work Queue와 Persistent Threads 다음에는 Wavefront Path Tracing Scheduling으로 이어지는 것이 자연스럽다. Path tracing에서는 ray generation, intersection, material shading, shadow ray, bounce 처리를 queue로 분리해 divergence를 줄이는 구조가 중요하다.

## 10. 참고 키워드

- GPU Work Queue
- Persistent Threads
- Dynamic Workload
- Atomic Counter
- Producer Consumer
- Task Queue
- Queue Overflow
- Load Balancing
- GPU-driven Rendering
- Sparse Voxel Traversal
- Octree Traversal
- Path Tracing
- Wavefront Scheduling
- Ray Queue
- Active Cell Queue
- Meshlet Queue
- Indirect Dispatch
- Compute Shader
- Scientific Visualization
