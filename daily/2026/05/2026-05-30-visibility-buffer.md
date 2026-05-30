---
title: "Visibility Buffer"
date: "2026-05-30"
category: "Graphics"
tags: ["visibility-buffer", "deferred-rendering", "gpu-driven-rendering", "rendering-pipeline", "g-buffer", "material-system"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-05-30 - Visibility Buffer

## 1. 오늘의 개념

**Visibility Buffer**는 화면의 각 픽셀에 대해 “최종적으로 어떤 primitive가 보였는가”를 기록하는 렌더링 구조다.

전통적인 **Deferred Rendering**에서는 G-buffer에 position, normal, albedo, roughness, metallic, material parameter 같은 shading에 필요한 데이터를 직접 저장한다. 반면 Visibility Buffer는 보통 다음과 같은 최소 정보를 저장한다.

- primitive ID / triangle ID
- instance ID / draw ID
- barycentric coordinate 또는 depth 기반 재구성 정보
- material ID

즉, 픽셀에 “색을 계산하기 위한 결과”를 저장하는 것이 아니라, “어떤 geometry가 이 픽셀을 차지했는지”를 저장한다. 이후 shading pass에서 vertex buffer, index buffer, material buffer, texture descriptor 등을 다시 참조하여 최종 shading을 수행한다.

이 관점은 **visibility determination**과 **material shading**을 분리한다는 점에서 중요하다.

## 2. 한 줄 핵심

**Visibility Buffer는 G-buffer에 모든 shading 속성을 저장하지 않고, 픽셀별 visible primitive 정보를 저장한 뒤 후속 pass에서 geometry/material 데이터를 다시 읽어 shading하는 방식이다.**

## 3. 왜 중요한가

Deferred Rendering은 많은 light를 처리하기 좋지만, G-buffer가 커지기 쉽다. 특히 PBR material이 복잡해질수록 normal, tangent, base color, roughness, metallic, emissive, custom material parameter 등을 여러 render target에 저장해야 한다.

이때 문제는 단순히 VRAM 사용량이 아니라 **memory bandwidth**다. 현대 GPU 렌더링에서 많은 경우 병목은 ALU 연산보다 render target write/read, texture fetch, cache miss, bandwidth pressure에서 발생한다.

Visibility Buffer는 G-buffer에 “완성된 shading 입력 데이터”를 모두 쓰지 않고, primitive/material을 식별할 수 있는 작은 정보를 저장한다. 따라서 render target footprint를 줄일 수 있고, material shading을 더 늦은 단계에서 수행할 수 있다.

또한 modern renderer에서는 geometry가 점점 더 GPU-driven으로 이동한다.

- indirect draw
- meshlet
- mesh shader
- bindless texture/resource
- material table
- GPU culling
- LOD selection

이런 구조에서는 Visibility Buffer가 자연스럽다. 화면에 실제로 보인 primitive만 후속 shading에서 평가할 수 있기 때문이다.

## 4. 구현 관점

### 4.1 기본 pass 구조

Visibility Buffer 기반 렌더러는 보통 다음 흐름을 가진다.

```text
1. Depth / Visibility Pass
   - rasterization 수행
   - 픽셀마다 primitive ID, instance ID, material ID, depth 등을 기록

2. Shading Pass
   - visibility buffer를 읽음
   - primitive ID로 vertex/index buffer 접근
   - barycentric coordinate 또는 depth로 position/normal 재구성
   - material ID로 material parameter/texture 접근
   - lighting 수행

3. Post Processing
   - tone mapping, bloom, TAA, SSR/SSAO 등 수행
```

Deferred Rendering과 유사하게 screen-space shading pass가 존재하지만, G-buffer에 저장된 attribute를 읽는 것이 아니라 geometry/material 원본 데이터를 다시 읽는다는 차이가 있다.

### 4.2 저장되는 데이터

Visibility Buffer에 저장되는 정보는 엔진 구조에 따라 다르지만 대표적으로 다음이 있다.

```text
uint primitiveID;
uint instanceID;
uint materialID;
float depth;
```

또는 packing을 통해 하나의 32-bit / 64-bit value에 여러 ID를 압축할 수도 있다.

예시:

```text
32-bit visibility value
[ drawID | primitiveID | materialLocalIndex ]
```

다만 ID bit width를 정할 때는 scene 규모, draw batching 방식, meshlet 단위, material table 구조를 함께 고려해야 한다.

### 4.3 Shading pass에서 필요한 재구성

Visibility Buffer만으로는 바로 lighting을 할 수 없다. 후속 pass에서 다음 데이터를 재구성해야 한다.

- world position
- geometric normal
- shading normal
- tangent frame
- UV
- material parameters

이를 위해 primitive ID를 기반으로 vertex/index buffer에서 삼각형의 세 vertex를 읽고, barycentric coordinate를 이용해 interpolate한다.

```text
P = b0 * P0 + b1 * P1 + b2 * P2
N = normalize(b0 * N0 + b1 * N1 + b2 * N2)
UV = b0 * UV0 + b1 * UV1 + b2 * UV2
```

핵심은 G-buffer에 미리 저장할 것인가, 아니면 shading 시점에 다시 읽고 계산할 것인가의 trade-off다.

### 4.4 장점

#### G-buffer bandwidth 감소

전통적인 Deferred Rendering은 여러 render target을 사용한다.

```text
RT0: normal + roughness
RT1: albedo + metallic
RT2: emissive + material flags
RT3: motion vector / custom data
Depth: depth
```

반면 Visibility Buffer는 핵심 ID와 depth 중심으로 저장한다. 이론적으로 render target write/read 양이 줄어든다.

#### Material system 유연성

G-buffer 방식은 모든 material을 일정한 attribute layout에 맞춰야 한다. 하지만 실제 AAA/engine renderer에서는 material 종류가 매우 다양하다.

- opaque PBR
- anisotropic material
- clearcoat
- cloth
- skin
- hair
- decal
- terrain
- procedural material

Visibility Buffer는 material ID를 통해 후속 shading pass에서 material별 branch 또는 shader table을 사용할 수 있다. 즉, G-buffer layout에 material 표현력을 강제로 맞추는 부담이 줄어든다.

#### GPU-driven rendering과 잘 맞음

GPU culling, meshlet rendering, indirect draw에서는 CPU가 모든 draw call/material binding을 직접 관리하지 않는다. Visibility Buffer는 drawID, instanceID, primitiveID를 통해 GPU 내부의 compact data table을 참조하는 구조와 잘 맞는다.

### 4.5 단점과 주의점

#### Shading pass의 random access 증가

Visibility Buffer는 G-buffer bandwidth를 줄일 수 있지만, shading pass에서 vertex/index/material buffer를 다시 읽어야 한다. 이 접근은 screen-space 순서로 발생하기 때문에 primitive-locality가 좋지 않을 수 있다.

즉, 이 방식은 다음 trade-off를 가진다.

```text
G-buffer 방식:
- 큰 render target write/read
- screen-space locality 좋음
- shading 입력이 이미 정리되어 있음

Visibility Buffer 방식:
- 작은 visibility target
- 후속 pass에서 geometry/material random access 증가
- material system 유연성 증가
```

#### Alpha blending / transparency 처리

Visibility Buffer는 기본적으로 “픽셀당 가장 앞에 보이는 primitive 하나”를 기록하기 쉽다. 따라서 order-dependent transparency, hair, particle, volume처럼 여러 fragment가 누적되어야 하는 경우에는 별도 경로가 필요하다.

일반적으로 다음과 같이 분리한다.

```text
Opaque geometry: Visibility Buffer
Transparent geometry: Forward / Weighted Blended OIT / per-pixel linked list 등 별도 처리
```

#### MSAA와 edge 처리

픽셀 하나에 여러 triangle이 걸치는 edge에서는 sample 단위 visibility가 필요할 수 있다. Visibility Buffer를 pixel 단위로만 저장하면 geometry edge aliasing 또는 shading mismatch가 생길 수 있다.

따라서 MSAA를 고려하면 sample-level visibility buffer, depth resolve 전략, TAA와의 결합을 함께 설계해야 한다.

## 5. 내 관심 분야와 연결

### Real-time Rendering / Game Engine

Visibility Buffer는 modern game renderer에서 Deferred Rendering의 bandwidth 문제와 material flexibility 문제를 해결하기 위한 선택지다. 특히 많은 material variation, GPU culling, mesh shader 기반 renderer를 설계할 때 중요한 개념이다.

### WebGPU / Vulkan / DirectX

WebGPU, Vulkan, DirectX 12 같은 explicit graphics API에서는 buffer/resource binding 구조를 직접 설계해야 한다. Visibility Buffer 방식은 다음 개념들과 연결된다.

- storage buffer 기반 scene data
- bindless-like material indexing
- indirect draw command buffer
- depth pre-pass
- compute shading pass
- render target format packing

WebGPU에서도 완전한 bindless는 제한적이지만, material table과 texture array / bind group indexing 구조를 잘 잡으면 Visibility Buffer에 가까운 구조를 실험적으로 설계할 수 있다.

### CFD / Scientific Visualization

대규모 CFD/VTK/volume/particle visualization에서는 모든 attribute를 화면 버퍼에 직접 저장하기 어렵다. 예를 들어 cell ID, block ID, scalar field ID를 visibility 단계에서 저장하고, 후속 pass에서 실제 field value를 lookup하는 구조를 생각할 수 있다.

이 관점은 다음과 연결된다.

- cell picking
- slice/clip 후 visible primitive 식별
- sparse voxel / octree block ID 기반 shading
- large dataset attribute lookup
- GPU-side filtering 결과의 후속 visualization

즉, Visibility Buffer는 게임 렌더링뿐 아니라 scientific visualization에서도 “화면에 보인 데이터 엔티티를 식별하고 나중에 해석한다”는 설계 패턴으로 확장할 수 있다.

### Semiconductor 3D Visualization

반도체 3D 구조에서는 layer, material, process region, mask region, CD-bias/taper parameter 등이 복잡하게 얽힌다. 모든 material 속성을 G-buffer에 고정 layout으로 넣기보다, 픽셀마다 보이는 surface/entity ID를 저장한 뒤 후속 shading에서 layer/material/process metadata를 참조하는 방식이 더 유연할 수 있다.

예를 들어:

```text
Visibility Buffer value:
- surfaceID
- layerID
- materialID
- processRegionID

Shading / Inspection pass:
- layer thickness lookup
- material color lookup
- taper angle visualization
- selected process region highlight
```

이는 단순 렌더링뿐 아니라 inspection, picking, measurement UI와도 연결된다.

## 6. 머릿속에 남길 질문 3개

1. **G-buffer에 normal/albedo/material parameter를 직접 저장하는 방식과 primitive/material ID만 저장하는 방식은 bandwidth와 flexibility 관점에서 어떻게 다른가?**
2. **Visibility Buffer의 shading pass에서 vertex/index/material buffer를 다시 읽을 때 cache locality 문제는 어떻게 발생하는가?**
3. **GPU-driven rendering, mesh shader, bindless resource 구조와 결합하면 Visibility Buffer의 장점은 어디서 가장 커지는가?**

## 7. Graphics Engineer 면접 질문 1개와 답변

### Q. Visibility Buffer는 Deferred Rendering의 G-buffer와 무엇이 다르며, 어떤 상황에서 유리한가?

**A.** Deferred Rendering의 G-buffer는 픽셀별 normal, albedo, roughness, metallic 같은 shading attribute를 여러 render target에 직접 저장한다. 반면 Visibility Buffer는 픽셀별로 visible primitive, instance, material ID 같은 식별 정보를 저장하고, 후속 shading pass에서 vertex/index/material buffer를 다시 참조해 필요한 attribute를 재구성한다.

이 방식은 G-buffer bandwidth와 memory footprint를 줄일 수 있고, material 종류가 다양하거나 GPU-driven rendering 구조를 사용하는 엔진에서 유리하다. 특히 material table, bindless resource, indirect draw, meshlet 기반 renderer와 잘 맞는다.

하지만 shading pass에서 geometry/material buffer에 대한 random access가 증가할 수 있고, transparency나 MSAA edge 처리는 별도 설계가 필요하다. 따라서 Visibility Buffer는 모든 상황에서 Deferred Rendering을 대체한다기보다, 복잡한 material system과 bandwidth 제약이 큰 renderer에서 고려할 수 있는 구조다.

## 8. 포트폴리오 / 커리어 연결

Visibility Buffer를 이해하고 있다는 것은 단순히 “Deferred Rendering을 안다”를 넘어선다. 렌더링 파이프라인을 memory layout, bandwidth, material system, GPU-driven architecture 관점에서 설계할 수 있다는 신호다.

포트폴리오에서는 다음 식으로 표현할 수 있다.

```text
Designed a visibility-oriented rendering pipeline concept that separates visibility determination from material shading, reducing G-buffer bandwidth pressure and enabling flexible material lookup for large-scale visualization datasets.
```

또는 반도체/CFD visualization 맥락에서는 다음과 같이 연결할 수 있다.

```text
Explored visibility-buffer-style entity ID rendering for large scientific datasets, enabling deferred attribute lookup, picking, inspection, and material-aware visualization without storing all attributes in screen-space buffers.
```

면접에서는 이 개념을 다음 키워드와 함께 말하면 좋다.

- G-buffer bandwidth
- render target format packing
- primitive ID / material ID
- shader resource buffer lookup
- cache locality
- GPU-driven rendering
- bindless resource
- transparency limitation

## 9. 내일 이어서 볼 개념

**Clustered Shading**

Visibility Buffer가 “픽셀에서 무엇이 보였는가”를 다루는 구조라면, Clustered Shading은 “화면/공간을 cluster로 나누고 어떤 light가 영향을 주는가”를 다루는 구조다.

내일은 Deferred / Forward+ / Clustered Shading이 light culling, depth slicing, compute shader, tiled memory access와 어떻게 연결되는지 살펴본다.

## 10. 참고 키워드

- Visibility Buffer
- Deferred Rendering
- Deferred Shading
- G-buffer
- GPU-driven Rendering
- Primitive ID
- Material ID
- Draw ID
- Bindless Resource
- Meshlet
- Mesh Shader
- Indirect Draw
- Render Target Bandwidth
- Cache Locality
- Forward+
- Clustered Shading
- Opaque Pass
- Transparency Rendering
- MSAA Resolve
- Scientific Visualization
- Entity ID Buffer
- Picking Buffer
