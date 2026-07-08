---
title: "Wavefront Path Tracing Scheduling"
date: "2026-07-08"
category: "Graphics"
tags: ["GPU", "Rendering Pipeline", "Path Tracing", "Wavefront", "Ray Tracing", "Work Queue", "Persistent Threads", "Divergence", "GPU Scheduling"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-07-08 - Wavefront Path Tracing Scheduling

## 1. 오늘의 개념

**Wavefront Path Tracing Scheduling**은 path tracing을 하나의 recursive shader 흐름으로 처리하지 않고, ray generation, intersection, material shading, shadow ray, next bounce 같은 단계를 queue로 분리해 GPU에서 더 coherent하게 처리하는 scheduling 방식이다.

Path tracing에서는 ray마다 실행 경로가 달라진다.

- 어떤 ray는 바로 miss된다.
- 어떤 ray는 diffuse surface에 hit된다.
- 어떤 ray는 glass material을 만나 refraction을 만든다.
- 어떤 ray는 shadow ray를 만든다.
- 어떤 ray는 여러 bounce를 수행한다.
- 어떤 ray는 Russian roulette으로 종료된다.

이 모든 작업을 하나의 mega-kernel 안에서 처리하면 GPU thread마다 branch가 달라지고, wave/warp divergence가 커진다. Wavefront 방식은 ray state를 queue에 저장하고, stage별 kernel이 같은 종류의 작업을 모아서 처리한다.

핵심 변화는 다음이다.

> Ray 하나가 recursive하게 끝까지 실행되는 구조에서, ray state가 queue를 이동하며 stage별로 처리되는 구조로 바뀐다.

이 구조는 이전 노트의 GPU Work Queue and Persistent Threads와 직접 연결된다. Path tracing은 불균일한 ray workload의 대표 사례이며, queue 기반 scheduling이 divergence와 load imbalance를 줄이는 데 중요하다.

## 2. 한 줄 핵심

**Wavefront Path Tracing Scheduling은 ray 상태를 queue로 분리하고 intersection / shading / shadow / bounce stage를 따로 처리해 GPU divergence를 줄이는 path tracing execution model이다.**

## 3. 왜 중요한가

GPU는 같은 wave 안의 thread가 같은 instruction path를 실행할 때 효율적이다. 하지만 path tracing은 ray마다 material, bounce count, visibility, termination 조건이 달라서 branch divergence가 매우 크다.

Mega-kernel path tracer는 구현이 직관적이다.

```text
Generate ray → Trace → Shade → Spawn next ray → Trace → Shade → ...
```

하지만 이 방식은 다음 문제가 생긴다.

- material branch divergence
- ray traversal cost imbalance
- bounce count 차이로 인한 idle lane 증가
- register pressure 증가
- recursion-like state가 커짐
- long-running thread 때문에 occupancy 저하

Wavefront path tracing은 이 문제를 stage별 queue로 나눈다.

```text
Ray Queue
→ Intersection Queue
→ Hit Shading Queue
→ Shadow Ray Queue
→ Next Bounce Queue
→ Accumulation
```

같은 stage의 ray를 모아 처리하면 kernel이 더 단순해지고, material 또는 ray type별로 sorting/batching할 수 있다.

Graphics engineer 관점에서 wavefront scheduling은 path tracer를 **recursive algorithm이 아니라 GPU data pipeline**으로 바꾸는 구조다.

## 4. 구현 관점

### 4.1 Mega-kernel path tracing

Mega-kernel 방식은 하나의 shader/kernel에서 path tracing loop를 모두 처리한다.

```cpp
for each pixel:
    Ray ray = GenerateCameraRay(pixel);
    Spectrum throughput = 1;

    for bounce in 0..maxBounce:
        Hit hit = Trace(ray);
        if miss:
            accumulate environment;
            break;

        Material mat = LoadMaterial(hit);
        Shade(mat, hit, throughput);
        ray = SpawnNextRay(mat, hit);
```

장점은 구조가 단순하고 ray state를 local variable로 유지할 수 있다는 점이다. 단점은 material branch, ray traversal, bounce termination이 모두 한 kernel 안에서 섞여 divergence와 register pressure가 커진다는 점이다.

### 4.2 Wavefront path tracing 기본 구조

Wavefront 방식은 ray state를 buffer에 저장하고, 각 stage가 queue를 소비하고 다음 queue를 만든다.

대표적인 ray state는 다음과 같다.

```cpp
struct RayState
{
    vec3 origin;
    vec3 direction;
    vec3 throughput;
    uint pixelIndex;
    uint bounce;
    uint rngState;
};
```

Hit state는 별도로 저장할 수 있다.

```cpp
struct HitState
{
    uint rayIndex;
    float t;
    uint primitiveId;
    uint materialId;
    vec2 barycentric;
};
```

Pipeline은 다음처럼 구성된다.

1. Camera ray generation queue 생성
2. Intersection kernel이 rays를 trace하고 hit/miss를 분리
3. Shading kernel이 material별로 ray를 처리
4. Shadow ray queue 생성
5. Next bounce ray queue 생성
6. 종료 ray는 accumulation buffer에 기여
7. bounce queue가 빌 때까지 반복

### 4.3 Queue 분리의 의미

Queue를 분리하면 다음 이점이 있다.

- Intersection만 하는 kernel은 traversal에 집중할 수 있다.
- Shading kernel은 material evaluation에 집중할 수 있다.
- Shadow ray는 occlusion test 전용으로 가볍게 처리할 수 있다.
- Bounce별 ray count를 추적할 수 있다.
- Material별 queue 또는 sorting이 가능하다.
- Long path와 short path를 더 유연하게 관리할 수 있다.

예를 들어 shadow ray는 closest-hit 정보가 필요 없고, occluded 여부만 필요할 수 있다. 이를 일반 ray와 같은 경로로 처리하면 불필요한 비용이 생긴다.

### 4.4 Divergence 줄이기

Wavefront 방식은 divergence를 완전히 없애지는 않는다. 하지만 divergence를 stage별로 제한하고, 유사한 작업을 모을 수 있다.

예를 들어 material shading에서 다음처럼 queue를 나눌 수 있다.

```text
Diffuse material queue
Metal material queue
Glass material queue
Emissive material queue
```

그러면 하나의 shading kernel 안에서 거대한 switch 문을 도는 대신, material type별 specialized kernel을 실행할 수 있다.

또는 material id / shader id 기준으로 ray를 정렬할 수도 있다. 이는 GPU Radix Sort와 연결된다.

```text
hit rays → material key sort → material-specific shading
```

즉 wavefront scheduling은 work queue, compaction, radix sort와 함께 동작하는 구조다.

### 4.5 Ray compaction

각 bounce가 끝날 때 모든 ray가 살아남는 것은 아니다.

- miss된 ray는 종료된다.
- light에 hit한 ray는 accumulation 후 종료될 수 있다.
- Russian roulette으로 일부 ray가 종료된다.
- max bounce에 도달한 ray는 종료된다.

따라서 다음 bounce queue에는 active ray만 compact해야 한다.

```text
activeFlag → prefix sum → nextRayQueue
```

이전 노트의 Prefix Sum / Scan이 여기서 다시 등장한다. Wavefront path tracing은 매 stage마다 queue compaction을 반복하는 GPU data pipeline이다.

### 4.6 Register pressure와 memory traffic trade-off

Wavefront 방식은 mega-kernel보다 register pressure를 줄일 수 있다. 각 kernel이 한 stage만 처리하므로 필요한 local state가 작아진다.

하지만 ray state를 global memory queue에 저장하고 다시 읽어야 하므로 memory traffic이 증가한다.

Trade-off는 다음이다.

- Mega-kernel: 적은 global memory queue traffic, 높은 divergence와 register pressure
- Wavefront: 더 많은 queue read/write, 낮은 divergence와 stage specialization

따라서 wavefront 방식이 항상 빠른 것은 아니다. Scene complexity, material 다양성, ray depth, GPU architecture에 따라 다르다.

### 4.7 GPU occupancy와 scheduling

Wavefront 방식은 stage별 kernel이 짧고 단순해질 수 있어 occupancy에 유리할 수 있다. 하지만 queue가 너무 작아지면 GPU를 충분히 채우지 못한다.

예를 들어 후반 bounce에서 살아남은 ray가 적으면 kernel launch overhead와 low occupancy 문제가 생긴다.

해결 전략은 다음과 같다.

- 여러 pixel/sample의 ray를 batch로 처리
- path depth별 queue를 합치거나 threshold 이하에서는 mega-kernel fallback
- persistent threads로 small queue 처리
- material queue를 너무 잘게 나누지 않음
- queue size statistics를 기반으로 adaptive scheduling

즉 wavefront scheduling은 queue granularity를 잘 선택해야 한다.

### 4.8 Hardware ray tracing과 software traversal

Hardware ray tracing API에서도 wavefront scheduling 사고는 유용하다. DXR/Vulkan RT는 shader binding table과 hit/miss shader 구조를 제공하지만, material divergence와 ray type 분리는 여전히 중요하다.

Software ray tracing, sparse voxel ray marching, SDF tracing에서는 더 직접적으로 queue 기반 scheduling을 설계할 수 있다.

Sparse voxel renderer에서는 다음 구조가 가능하다.

```text
Ray queue
→ voxel traversal queue
→ hit shading queue
→ secondary ray queue
→ volume scattering queue
```

즉 wavefront path tracing은 traditional triangle ray tracing뿐 아니라 sparse voxel / volume renderer에도 적용 가능한 scheduling pattern이다.

## 5. 내 관심 분야와 연결

### CFD / scientific visualization

CFD visualization에서 path tracing 자체를 바로 쓰지 않더라도, wavefront scheduling 사고는 volume ray marching과 연결된다.

Volume rendering에서는 ray마다 traversal 길이와 sampling cost가 다르다.

- 빈 공간만 지나는 ray
- dense volume을 지나는 ray
- iso-surface를 hit하는 ray
- high-gradient region에서 step이 늘어나는 ray
- early termination되는 ray

이를 queue로 나누면 다음 구조가 가능하다.

```text
ray generation queue
→ empty-space skipping queue
→ dense brick sampling queue
→ shading/compositing queue
→ terminated ray accumulation
```

즉 wavefront path tracing의 핵심은 ray tracing 전용이 아니라, 불균일한 ray workload를 stage별로 나누는 scheduling 방식이다.

### Sparse voxel / octree / NanoVDB

Sparse voxel traversal은 wavefront scheduling과 잘 맞는다. Ray마다 방문하는 node/brick 수가 다르기 때문이다.

가능한 구조는 다음이다.

- ray queue
- node traversal queue
- brick sampling queue
- surface hit shading queue
- secondary ray / shadow ray queue
- missing brick streaming request queue

NanoVDB-style sparse volume traversal에서도 ray state와 node traversal state를 queue로 관리하면 divergence를 줄일 수 있다.

### Game engine architecture

Game engine에서는 wavefront scheduling이 real-time ray tracing, path tracing, GI, reflection, shadow ray, denoising input generation과 연결된다.

- material sorting
- ray type queue 분리
- shadow ray fast path
- bounce별 compaction
- persistent thread scheduling
- GPU queue statistics

면접에서는 wavefront path tracing을 “path tracing을 queue 기반 data pipeline으로 바꾸는 구조”라고 설명하는 것이 좋다.

## 6. 머릿속에 남길 질문 3개

1. Mega-kernel path tracer에서 material branch와 bounce count 차이가 GPU divergence를 만드는 이유는 무엇인가?
2. Wavefront path tracing이 queue read/write memory traffic을 증가시키면서도 성능상 유리할 수 있는 이유는 무엇인가?
3. Sparse voxel volume ray marching에서 ray queue를 stage별로 나누면 어떤 workload imbalance를 줄일 수 있는가?

## 7. graphics engineer 면접 질문 1개와 답변

### Q. Wavefront Path Tracing은 무엇이며 Mega-kernel Path Tracing과 비교했을 때 어떤 장단점이 있나요?

**A.** Wavefront Path Tracing은 path tracing을 하나의 큰 recursive-like kernel로 처리하지 않고, ray generation, intersection, material shading, shadow ray, next bounce 같은 stage를 queue로 분리해 처리하는 방식입니다. Mega-kernel 방식은 하나의 shader 안에서 trace, shade, bounce loop를 모두 수행하므로 구현이 단순하고 ray state를 local variable로 유지할 수 있습니다. 하지만 ray마다 material, branch, bounce count, traversal cost가 달라 GPU divergence와 register pressure가 커질 수 있습니다.

Wavefront 방식은 ray state를 global memory queue에 저장하고 stage별 kernel이 queue를 소비합니다. 같은 종류의 작업을 모아 처리할 수 있고, material별 sorting이나 shadow ray fast path 같은 최적화가 가능합니다. 단점은 queue read/write memory traffic, compaction pass, scheduling complexity가 증가한다는 점입니다. 따라서 material 다양성과 ray divergence가 큰 장면에서는 유리할 수 있지만, 단순한 장면에서는 mega-kernel이 더 효율적일 수 있습니다.

## 8. 포트폴리오 / 커리어 연결

Wavefront Path Tracing Scheduling은 포트폴리오에서 다음 메시지를 만든다.

> “나는 ray tracing을 단순 recursive shader가 아니라, queue / compaction / sorting / stage specialization으로 구성되는 GPU data pipeline으로 이해한다.”

네 배경과 연결하면 다음 표현이 좋다.

- Sparse voxel / volume ray marching에서 ray마다 다른 traversal cost를 queue 기반으로 관리하는 사고
- CFD volume rendering에서 dense brick sampling과 empty-space skipping을 stage로 분리하는 구조 이해
- GPU Work Queue, Prefix Sum, Radix Sort와 path tracing scheduling을 하나의 pipeline으로 연결 가능
- Vulkan / DX12 ray tracing 또는 software ray tracing에서 ray type별 queue와 material sorting을 설명할 수 있음

면접에서는 다음처럼 말할 수 있다.

> “Wavefront path tracing은 ray state를 queue에 저장하고 intersection, shading, shadow ray, bounce를 stage별로 처리해 divergence를 줄이는 방식입니다. Queue traffic은 늘지만, material sorting과 specialized kernel을 통해 GPU coherence를 높일 수 있습니다.”

## 9. 내일 이어서 볼 개념

**Denoising Inputs for Ray Tracing**

Wavefront path tracing 다음에는 ray tracing 결과를 실시간으로 사용하기 위한 denoising input을 보는 것이 자연스럽다. Normal, depth, albedo, motion vector, roughness 같은 auxiliary buffer가 temporal/spatial denoiser에서 어떤 역할을 하는지 이어서 볼 수 있다.

## 10. 참고 키워드

- Wavefront Path Tracing
- Mega-kernel Path Tracing
- Ray Queue
- Shadow Ray Queue
- Material Queue
- Bounce Queue
- Persistent Threads
- GPU Work Queue
- Prefix Sum Compaction
- Radix Sort Material Sorting
- Divergence
- Register Pressure
- Occupancy
- Path Tracing
- Ray Tracing
- Sparse Voxel Ray Marching
- Volume Rendering
- Scientific Visualization
- GPU Scheduling
