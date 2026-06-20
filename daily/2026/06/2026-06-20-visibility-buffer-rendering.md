---
title: "Visibility Buffer Rendering"
date: "2026-06-20"
category: "Graphics"
tags: ["GPU", "Rendering Pipeline", "Visibility Buffer", "Deferred Shading", "G-buffer", "Material Evaluation", "Bandwidth", "GPU-Driven Rendering", "Triangle ID"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-06-20 - Visibility Buffer Rendering

## 1. 오늘의 개념

**Visibility Buffer Rendering**은 deferred renderer처럼 먼저 화면의 visibility를 결정하되, G-buffer에 normal, albedo, roughness, metallic 같은 material attribute를 모두 저장하지 않고, 각 pixel에 보이는 primitive / triangle / instance / material id와 depth 중심의 최소 정보만 저장하는 렌더링 구조다.

전통적인 deferred rendering에서는 geometry pass에서 여러 render target에 material 정보를 미리 기록한다.

- normal
- albedo
- roughness
- metallic
- emissive
- depth
- material id

이 방식은 lighting을 screen-space로 분리하기 좋지만 G-buffer bandwidth가 크다. 해상도가 높아지고 render target 수가 늘어날수록 memory write/read 비용이 커진다.

Visibility Buffer는 접근을 바꾼다.

> 먼저 “이 pixel에 어떤 primitive가 보이는가”만 저장하고, 실제 material attribute 계산은 shading pass에서 primitive id를 기반으로 나중에 수행한다.

즉 G-buffer가 “shading에 필요한 결과 값”을 저장한다면, Visibility Buffer는 “shading을 다시 계산할 수 있는 주소”를 저장한다.

## 2. 한 줄 핵심

**Visibility Buffer Rendering은 G-buffer bandwidth를 줄이기 위해 pixel별 material 결과 대신 primitive/material 참조 정보를 저장하고, 후속 shading pass에서 필요한 attribute를 다시 가져와 계산하는 renderer architecture다.**

## 3. 왜 중요한가

Deferred rendering의 가장 큰 약점 중 하나는 G-buffer bandwidth다. 고해상도에서 normal, albedo, roughness, metallic, velocity, material id 등을 여러 attachment로 쓰고 다시 읽는 것은 큰 비용이 된다.

Visibility Buffer는 이 문제를 줄이기 위해 geometry pass의 output을 작게 만든다. 대신 shading pass에서 triangle id, instance id, material id를 이용해 vertex attribute와 material texture를 다시 fetch하고, 필요한 값을 계산한다.

이 구조가 중요한 이유는 다음과 같다.

- G-buffer memory bandwidth를 줄일 수 있다.
- Material evaluation을 더 유연하게 지연시킬 수 있다.
- GPU-driven rendering, meshlet, bindless resource table과 잘 맞는다.
- Geometry visibility와 shading evaluation을 더 강하게 분리한다.
- High resolution renderer에서 bandwidth-bound 문제를 완화할 수 있다.

Graphics engineer 관점에서는 Visibility Buffer가 단순 deferred 변형이 아니라, **visibility pass와 material shading pass 사이의 data ownership을 다시 설계하는 방식**이라는 점이 핵심이다.

## 4. 구현 관점

### 4.1 G-buffer와 Visibility Buffer의 차이

Deferred G-buffer는 pixel별로 shading에 필요한 값을 직접 저장한다.

```text
GBuffer0: normal.xyz + roughness
GBuffer1: albedo.rgb + metallic
GBuffer2: emissive.rgb + materialFlags
Depth: depth
```

Visibility Buffer는 훨씬 작은 정보를 저장한다.

```text
VisibilityBuffer: primitiveID / instanceID / materialID
Depth: depth
```

후속 shading pass에서는 이 id를 통해 필요한 데이터를 다시 찾는다.

```glsl
Visibility v = LoadVisibility(pixel);
PrimitiveData prim = primitiveBuffer[v.primitiveID];
MaterialData mat = materialBuffer[v.materialID];

Attributes attr = InterpolateAttributes(prim, pixel, depth);
vec3 color = EvaluateMaterial(mat, attr);
```

즉 G-buffer는 “계산된 material attribute”를 저장하고, Visibility Buffer는 “계산할 수 있는 참조”를 저장한다.

### 4.2 Visibility pass

Visibility pass는 depth test를 통해 가장 앞에 보이는 primitive를 결정한다. 이때 fragment output은 color가 아니라 primitive id 계열의 packed value다.

예를 들어 다음 값을 packing할 수 있다.

- triangle id
- draw id
- instance id
- material id
- meshlet id
- barycentric coordinate 보조 정보

실제 설계에서는 bit packing이 중요하다. 32-bit 또는 64-bit 안에 어떤 정보를 넣을지 결정해야 한다.

예시:

```cpp
uint visibilityValue =
    (drawID << 20) |
    (triangleID & 0xFFFFF);
```

이 구조는 scene 규모에 따라 한계가 있으므로 draw id, mesh id, local triangle id를 나누거나 별도 indirection table을 둔다.

### 4.3 Attribute reconstruction

Visibility Buffer의 핵심 난점은 shading pass에서 attribute를 다시 복원해야 한다는 점이다.

G-buffer에서는 normal, uv, material value가 이미 저장되어 있다. Visibility Buffer에서는 pixel이 어떤 triangle에 속하는지 알고, 그 triangle의 vertex attribute를 다시 읽어 barycentric interpolation을 수행해야 한다.

필요한 데이터는 다음과 같다.

- index buffer
- vertex position
- vertex normal / tangent
- uv
- material index
- transform matrix
- texture index

Shading pass에서는 depth와 screen position을 이용해 world position을 복원하고, triangle의 vertex data를 fetch해 attribute를 계산한다.

장점은 G-buffer write/read bandwidth를 줄인다는 것이다. 단점은 shading pass의 buffer fetch와 interpolation ALU 비용이 증가한다는 점이다.

### 4.4 Barycentric coordinate 문제

Visibility Buffer에서는 pixel 내부의 barycentric coordinate가 필요할 수 있다. 이를 얻는 방법은 여러 가지다.

- hardware barycentric extension 사용
- screen-space derivative로 복원
- triangle vertex position을 fetch해 직접 계산
- visibility pass에서 보조 정보를 저장

Hardware barycentric을 사용할 수 있으면 구조가 간단해진다. 하지만 API와 GPU feature support를 고려해야 한다. Vulkan, DX12, Metal, WebGPU 환경마다 지원 방식이 다르므로 portable renderer에서는 fallback이 필요하다.

WebGPU에서는 native barycentric access가 제한적이므로, visibility buffer 기반 renderer를 구현하려면 attribute reconstruction 전략을 더 신중히 설계해야 한다.

### 4.5 Material evaluation과 bindless

Visibility Buffer는 bindless resource table과 잘 맞는다. Shading pass에서 material id를 읽고, material buffer를 통해 texture index를 찾은 뒤 필요한 texture를 sample한다.

```glsl
MaterialData mat = materialBuffer[materialID];
vec4 baseColor = texture(textures[mat.baseColorTex], uv);
float roughness = texture(textures[mat.roughnessTex], uv).r;
```

이 구조에서는 visibility pass가 material texture를 읽지 않는다. Material texture sampling은 후속 shading pass에서만 수행된다.

즉 rendering pipeline은 다음처럼 분리된다.

1. Visibility pass: 어떤 primitive가 보이는지 결정
2. Shading pass: 보이는 pixel에 대해서만 material evaluation
3. Lighting pass: material 결과와 light를 사용해 최종 color 계산

### 4.6 장점과 비용 모델

Visibility Buffer의 장점은 명확하다.

- G-buffer attachment 수 감소
- memory bandwidth 감소
- material evaluation 지연
- shading pass에서 material system 유연성 증가
- GPU-driven / meshlet pipeline과 결합 용이

하지만 비용도 있다.

- attribute reconstruction 비용 증가
- random buffer fetch 증가
- triangle / primitive id packing 복잡도
- MSAA 처리 복잡도
- texture cache locality 관리 필요
- debugging 난이도 증가

즉 Visibility Buffer는 bandwidth-bound renderer에는 유리할 수 있지만, attribute fetch / ALU / random access가 병목인 장면에서는 항상 이득이 아니다.

### 4.7 Deferred / Forward / Visibility Buffer 비교

세 구조는 다음처럼 볼 수 있다.

| 구조 | 먼저 저장하는 것 | 장점 | 단점 |
|---|---|---|---|
| Forward | 즉시 shading 결과 | 단순, transparency/MSAA 유리 | many-light와 overdraw에 약함 |
| Deferred | material attribute G-buffer | lighting 분리, many-light 유리 | G-buffer bandwidth 큼 |
| Visibility Buffer | primitive/material 참조 | bandwidth 감소, material 지연 평가 | attribute 재구성 복잡 |

Visibility Buffer는 deferred renderer의 “shading 분리” 장점은 유지하면서, G-buffer가 너무 많은 데이터를 저장하는 문제를 줄이려는 방향이다.

### 4.8 GPU-driven rendering과 meshlet 연결

Visibility Buffer는 meshlet / cluster renderer와 잘 맞는다. Visibility pass에서 triangle id 대신 meshlet id와 local primitive id를 저장하면, shading pass에서 meshlet buffer를 통해 attribute를 복원할 수 있다.

GPU-driven pipeline에서는 다음 구조가 가능하다.

1. Compute culling으로 visible meshlet list 생성
2. Meshlet rasterization 또는 indirect draw 수행
3. Visibility Buffer에 meshlet / primitive id 저장
4. Shading pass에서 meshlet data와 material table fetch
5. Lighting / post-process 진행

이 구조는 Nanite-like virtualized geometry, mesh shader pipeline, software rasterization visibility pass와도 연결된다.

## 5. 내 관심 분야와 연결

### CFD / scientific visualization

Scientific visualization에서는 G-buffer에 모든 field attribute를 저장하는 방식이 비효율적일 수 있다. 예를 들어 pressure, velocity, temperature, vorticity, phase field를 모두 attachment로 저장하면 bandwidth가 커진다.

Visibility Buffer 사고를 적용하면 먼저 pixel이 어떤 cell, face, voxel brick, iso-surface primitive를 보는지만 저장하고, 후속 pass에서 필요한 field value를 다시 fetch해 colormap이나 lighting을 계산할 수 있다.

즉 다음 구조가 가능하다.

- visibility: cell id / brick id / primitive id 저장
- shading: field buffer에서 scalar/vector value fetch
- material: transfer function / colormap 적용
- lighting: normal 또는 gradient 기반 shading

대용량 CFD visualization에서는 “값을 미리 다 저장할 것인가”보다 “보이는 pixel에서 필요한 값만 나중에 가져올 것인가”가 중요한 trade-off가 된다.

### Sparse voxel / octree / NanoVDB

Sparse voxel renderer에서는 pixel이 어떤 voxel brick 또는 node를 hit했는지 저장하는 방식으로 Visibility Buffer 사고를 적용할 수 있다.

예를 들어 ray marching 또는 surface extraction 결과로 다음 정보를 저장할 수 있다.

- brick id
- local voxel coordinate
- material id
- hit depth
- gradient / normal reconstruction key

후속 shading pass에서는 brick table과 sparse page table을 참조해 density, SDF, material field를 다시 fetch한다.

이 구조는 memory bandwidth를 줄이고, field evaluation을 필요한 pixel에만 지연시키는 데 유리하다.

### Game engine architecture

Game engine에서는 Visibility Buffer가 modern renderer architecture와 연결된다.

- G-buffer bandwidth 감소
- material shading 지연 평가
- bindless material system
- meshlet / cluster renderer
- GPU-driven visibility
- virtual geometry / Nanite-style pipeline

면접에서 이 개념을 설명하면 deferred shading, GPU memory bandwidth, primitive id packing, material evaluation pipeline을 함께 이해하고 있다는 신호를 줄 수 있다.

## 6. 머릿속에 남길 질문 3개

1. Visibility Buffer가 G-buffer보다 memory bandwidth를 줄일 수 있는 이유는 무엇인가?
2. Visibility Buffer에서 primitive id만 저장했을 때, 후속 shading pass는 normal / uv / material attribute를 어떻게 복원해야 하는가?
3. CFD visualization에서 cell id 또는 brick id 기반 Visibility Buffer를 사용하면 어떤 장점과 비용이 생길 수 있는가?

## 7. graphics engineer 면접 질문 1개와 답변

### Q. Visibility Buffer Rendering은 Deferred Rendering과 무엇이 다르며, 어떤 trade-off가 있나요?

**A.** Deferred Rendering은 geometry pass에서 normal, albedo, roughness, metallic 같은 material attribute를 G-buffer에 저장하고, lighting pass에서 이를 읽어 shading합니다. 반면 Visibility Buffer Rendering은 각 pixel에 보이는 primitive, triangle, instance, material id와 depth 같은 최소 visibility 정보만 저장합니다. 후속 shading pass에서 이 id를 이용해 vertex/index/material buffer를 다시 fetch하고, normal, uv, material attribute를 복원해 shading합니다.

장점은 G-buffer attachment 수와 memory bandwidth를 줄일 수 있고, material evaluation을 보이는 pixel에 대해서만 지연 수행할 수 있다는 점입니다. 또한 bindless material table, meshlet renderer, GPU-driven rendering과 잘 맞습니다. 단점은 attribute reconstruction 비용이 증가하고, random buffer fetch, barycentric interpolation, primitive id packing, MSAA 처리, debugging이 복잡해진다는 점입니다. 따라서 bandwidth-bound renderer에서는 유리할 수 있지만, ALU나 random memory access가 병목인 경우에는 반드시 이득이라고 볼 수 없습니다.

## 8. 포트폴리오 / 커리어 연결

Visibility Buffer Rendering은 포트폴리오에서 다음 메시지를 만든다.

> “나는 deferred renderer의 G-buffer bandwidth 문제를 이해하고, visibility와 material evaluation을 분리하는 modern rendering architecture를 설명할 수 있다.”

네 배경과 연결하면 특히 강한 지점은 다음이다.

- OpenGL deferred renderer의 G-buffer 구조 이해
- 대용량 CFD / VTK visualization에서 field attribute 저장 비용 문제와 연결
- Sparse voxel / octree renderer에서 brick id 기반 deferred field evaluation으로 확장
- WebGPU / Vulkan에서 storage buffer, bindless-like material table, primitive id 기반 shading 구조로 연결

면접에서는 다음처럼 말할 수 있다.

> “Visibility Buffer는 pixel별 material 결과를 저장하는 대신 primitive reference를 저장하고, shading pass에서 필요한 attribute를 재구성하는 방식입니다. 이 구조는 G-buffer bandwidth를 줄이는 대신 attribute fetch와 reconstruction 비용을 지불하는 trade-off입니다.”

## 9. 내일 이어서 볼 개념

**Meshlet Rendering**

Visibility Buffer 다음에는 Meshlet Rendering으로 이어지는 것이 자연스럽다. Visibility Buffer가 primitive id 기반 shading 구조라면, Meshlet은 geometry를 GPU-friendly한 작은 cluster로 나누어 culling, LOD, shading data fetch의 단위로 사용하는 개념이다.

## 10. 참고 키워드

- Visibility Buffer Rendering
- Deferred Shading
- G-buffer Bandwidth
- Primitive ID
- Triangle ID
- Instance ID
- Material ID
- Attribute Reconstruction
- Barycentric Coordinates
- Bindless Material
- Meshlet Rendering
- GPU-Driven Rendering
- Material Evaluation
- Storage Buffer
- Vertex Fetch
- Index Buffer
- Screen-space Shading
- Bandwidth-bound Renderer
- Sparse Voxel Hit Buffer
- Scientific Visualization
