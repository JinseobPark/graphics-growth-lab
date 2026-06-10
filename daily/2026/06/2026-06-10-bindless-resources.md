---
title: "Bindless Resources"
date: "2026-06-10"
category: "Graphics"
tags: ["GPU", "Rendering Pipeline", "Descriptor Indexing", "Bindless", "Vulkan", "DirectX12", "Metal", "WebGPU", "Resource Binding"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-06-10 - Bindless Resources

## 1. 오늘의 개념

**Bindless Resources**는 렌더링 중 필요한 texture, buffer, sampler 같은 GPU resource를 매 draw call마다 CPU가 하나씩 bind하지 않고, shader가 큰 resource table 또는 descriptor array에서 index로 직접 접근하도록 만드는 렌더링 구조다.

전통적인 OpenGL식 사고에서는 draw call마다 다음과 같은 상태 변경이 반복된다.

- `glBindTexture(...)`
- `glBindBuffer(...)`
- material uniform 갱신
- draw call 제출

반면 bindless 방식에서는 많은 resource를 미리 GPU-visible table에 올려두고, object/material/instance가 **resource handle 또는 descriptor index**만 들고 있다. Shader는 이 index를 사용해 필요한 texture나 buffer를 선택한다.

즉 핵심은 다음 변화다.

> CPU가 매번 resource를 직접 바인딩하는 구조에서, GPU가 index 기반으로 resource를 선택하는 구조로 이동한다.

이 개념은 Vulkan의 **descriptor indexing**, DirectX 12의 **descriptor heap**, OpenGL의 **bindless texture**, modern engine의 **material system**, GPU-driven rendering과 강하게 연결된다.

## 2. 한 줄 핵심

**Bindless Resources는 draw call 중심의 resource binding 비용을 줄이고, shader가 descriptor index로 texture/buffer를 직접 선택하게 만드는 GPU-driven rendering의 핵심 기반이다.**

## 3. 왜 중요한가

복잡한 scene renderer에서는 object 수, material 수, texture 수가 늘어날수록 CPU-side binding 비용이 커진다. 특히 많은 mesh, 많은 material, 많은 texture atlas, voxel brick, sparse volume, particle buffer를 다루는 경우 매 draw마다 resource를 교체하는 구조는 확장성이 낮다.

Bindless 구조는 renderer를 다음 방향으로 바꾼다.

- draw call 수 감소
- material switching 감소
- resource binding state change 감소
- GPU-driven culling / indirect draw와 결합 가능
- large scene, voxel, point cloud, scientific visualization에 유리

Graphics engineer 관점에서 bindless는 단순 최적화 트릭이 아니라, **renderer architecture를 CPU submission 중심에서 GPU data-driven 중심으로 바꾸는 설계 패턴**이다.

## 4. 구현 관점

### 4.1 Traditional binding model

전통적 binding model에서는 CPU가 draw call 직전에 resource를 명시적으로 설정한다.

```cpp
BindPipeline(pipeline);
BindTexture(material.albedo);
BindTexture(material.normal);
BindUniformBuffer(materialUBO);
DrawMesh(mesh);
```

이 구조는 이해하기 쉽지만 object/material 수가 늘어나면 다음 비용이 커진다.

- API call overhead
- state validation cost
- descriptor update cost
- material sorting dependency
- multi-threaded command recording 복잡도

OpenGL 기반 renderer에서는 특히 global state machine 성격 때문에 resource binding이 렌더링 구조 전체를 지배하기 쉽다.

### 4.2 Bindless model

Bindless model에서는 texture와 buffer를 거대한 table에 넣고 material은 index만 가진다.

```cpp
struct MaterialData
{
    uint albedoTextureIndex;
    uint normalTextureIndex;
    uint roughnessTextureIndex;
    uint metallicTextureIndex;
};
```

Shader 쪽에서는 다음과 같은 사고방식이 된다.

```glsl
MaterialData mat = materialBuffer[materialIndex];
vec4 albedo = texture(textures[mat.albedoTextureIndex], uv);
```

중요한 변화는 CPU가 texture object를 매번 bind하는 것이 아니라, GPU-visible descriptor array에 resource를 등록해두고 shader가 index로 접근한다는 점이다.

### 4.3 Vulkan / DirectX 12 / Metal / WebGPU 관점

#### Vulkan

Vulkan에서는 bindless style을 보통 다음 기능과 함께 구성한다.

- `VK_EXT_descriptor_indexing`
- runtime-sized descriptor arrays
- partially bound descriptors
- update-after-bind
- non-uniform indexing

Material buffer에는 texture index를 저장하고, descriptor set에는 많은 sampled image를 배열로 둔다. Shader에서는 `nonuniformEXT(index)`를 통해 wave 내 서로 다른 index 접근을 명시해야 하는 경우가 있다.

#### DirectX 12

DirectX 12에서는 descriptor heap과 root signature가 핵심이다. Shader-visible descriptor heap에 SRV/UAV/CBV descriptor를 배치하고, shader는 descriptor table offset 또는 resource index를 통해 접근한다.

DX12식 사고에서는 bindless가 다음 개념과 연결된다.

- descriptor heap management
- root descriptor table
- bindless SRV/UAV array
- residency management

#### Metal

Metal은 argument buffer를 통해 bindless와 유사한 resource table 구조를 만들 수 있다. Argument buffer는 여러 texture, buffer, sampler reference를 하나의 GPU-visible buffer처럼 묶어 shader에 전달하는 방식이다.

#### WebGPU

WebGPU는 보안성과 portability를 위해 완전한 native bindless를 그대로 제공하지 않는다. 하지만 bind group, texture array, storage buffer indexing, material table 구조를 조합해 제한적인 bindless-like design을 만들 수 있다.

WebGPU에서의 핵심은 native API처럼 모든 resource를 무제한으로 열어두는 것이 아니라, backend별 binding limit 안에서 material batching과 resource table 설계를 균형 있게 잡는 것이다.

### 4.4 Memory layout 관점

Bindless renderer에서 중요한 것은 resource index 자체보다 **material/object data layout**이다.

일반적으로 다음 구조를 사용한다.

```cpp
struct ObjectData
{
    mat4 world;
    uint materialIndex;
    uint meshIndex;
    uint instanceFlags;
};

struct MaterialData
{
    vec4 baseColorFactor;
    float roughness;
    float metallic;
    uint albedoTex;
    uint normalTex;
    uint packedParamsTex;
};
```

이때 고려할 점은 다음과 같다.

- shader에서 cache-friendly하게 읽히는가
- material index가 coherent한가
- texture access divergence가 심하지 않은가
- descriptor table update가 frame마다 과도하지 않은가
- resource lifetime과 index recycling이 안전한가

Bindless라고 해서 자동으로 빨라지는 것은 아니다. Wave 또는 warp 내부에서 서로 다른 texture index를 마구 접근하면 texture cache locality가 나빠질 수 있다. 따라서 GPU-driven renderer에서도 material sorting, batching, cluster grouping은 여전히 중요하다.

### 4.5 Shader divergence 문제

Bindless는 shader flexibility를 높이지만 divergence 위험도 만든다.

예를 들어 하나의 draw call 안에서 instance마다 서로 다른 texture를 참조하면 CPU binding 비용은 줄어들지만, GPU texture sampling coherence가 떨어질 수 있다.

따라서 실무에서는 다음 균형이 필요하다.

- CPU submission overhead를 줄이기 위해 bindless 사용
- GPU cache locality를 위해 material/texture grouping 유지
- 너무 세분화된 resource random access는 피함
- indirect draw와 함께 사용할 때 visibility result를 material-aware하게 정렬

좋은 bindless renderer는 단순히 모든 resource를 배열에 넣는 것이 아니라, **CPU overhead와 GPU memory coherence 사이의 trade-off를 설계하는 renderer**다.

## 5. 내 관심 분야와 연결

### Sparse voxel / NanoVDB / volume rendering

Sparse voxel renderer에서는 brick, tile, node buffer, density texture, material table이 매우 많아질 수 있다. 이때 bindless-like 구조는 각 voxel brick이 자신의 density field, attribute texture, transfer function index를 참조하는 방식으로 확장된다.

예를 들어 semiconductor 3D visualization에서 layer별 material, CD-bias, taper angle, thickness field를 별도 buffer나 texture로 관리한다면, bindless resource table은 다음 구조에 유리하다.

- layer별 parameter buffer
- material별 transfer function
- brick별 distance field / density field
- sparse page table
- ray marching 중 brick resource 선택

### CFD / scientific visualization

CFD 후처리에서는 velocity, pressure, temperature, vorticity, phase field 등 field data가 여러 buffer/texture로 존재한다. Bindless 구조를 사용하면 visualization pass가 field index를 받아 동적으로 scalar/vector field를 선택할 수 있다.

예를 들어 streamline, slice, volume rendering, iso-surface pass가 공통 field table을 참조하면 renderer 구조가 깔끔해진다.

### Game engine architecture

Game engine에서는 bindless가 modern material system과 연결된다.

- material graph output을 texture index table로 compile
- meshlet / cluster renderer와 결합
- GPU culling 후 indirect draw
- instance buffer에서 material index 참조
- virtual texture / streaming texture와 결합

Nintendo, Unity, Unreal 계열의 renderer를 이해하려면 bindless는 단순 API 기능이 아니라 **large-scale renderer data model**로 이해해야 한다.

## 6. 머릿속에 남길 질문 3개

1. Bindless renderer에서 CPU binding 비용을 줄이는 대신 GPU texture cache locality가 나빠질 수 있는 이유는 무엇인가?
2. MaterialData가 texture handle 대신 texture index를 들고 있을 때, resource lifetime과 index recycling은 어떻게 관리해야 하는가?
3. Sparse voxel volume renderer에서 brick마다 서로 다른 resource를 참조해야 한다면, bindless table과 page table은 어떻게 연결될 수 있는가?

## 7. graphics engineer 면접 질문 1개와 답변

### Q. Bindless Resources가 전통적인 resource binding 방식보다 유리한 점은 무엇이며, 어떤 단점이 있을 수 있나요?

**A.** Bindless Resources는 texture나 buffer를 매 draw call마다 CPU에서 bind하지 않고, GPU-visible descriptor table에 등록한 뒤 shader가 index로 접근하게 만드는 방식입니다. 장점은 draw call 제출 비용과 state change를 줄일 수 있고, 많은 material과 instance를 가진 scene에서 GPU-driven rendering, indirect draw, material table 구조와 잘 결합된다는 점입니다. 특히 Vulkan의 descriptor indexing, DX12의 descriptor heap, Metal의 argument buffer 같은 modern explicit API와 잘 맞습니다.

하지만 단점도 있습니다. Shader에서 instance마다 다른 texture index를 접근하면 wave 내부 resource access가 분산되어 cache locality가 나빠질 수 있습니다. 또한 descriptor table 관리, resource lifetime, index recycling, partially bound descriptor 처리, non-uniform indexing 같은 복잡도가 증가합니다. 따라서 bindless는 무조건 빠른 기능이 아니라 CPU overhead와 GPU memory coherence 사이의 trade-off를 관리하는 renderer architecture로 이해해야 합니다.

## 8. 포트폴리오 / 커리어 연결

Bindless Resources는 포트폴리오에서 다음 메시지를 만들 수 있다.

> “나는 단순히 shader effect를 구현하는 개발자가 아니라, 많은 object/material/volume resource를 다루는 renderer data model을 설계할 수 있다.”

특히 다음 경험과 연결하면 강하다.

- OpenGL renderer에서 material/texture binding 구조를 설계한 경험
- VTK/CFD large data visualization에서 field buffer를 관리한 경험
- Sparse voxel, octree, volume rendering에서 brick resource table을 설계한 경험
- WebGPU/Vulkan으로 넘어갈 때 bind group / descriptor set 한계를 고려한 경험

면접이나 포트폴리오 문장으로는 다음처럼 정리할 수 있다.

> “Large-scale scientific visualization renderer에서 object/material/field resource를 index 기반 table로 관리하여, draw submission과 resource switching 비용을 줄이는 GPU-driven rendering 구조를 설계하고자 합니다.”

이 문장은 CFD visualization, semiconductor 3D visualization, game renderer, engine architecture 모두에 연결된다.

## 9. 내일 이어서 볼 개념

**GPU-Driven Rendering**

Bindless Resources를 이해한 다음에는 GPU가 visibility, culling, draw command generation까지 담당하는 GPU-driven rendering으로 이어지는 것이 자연스럽다. Bindless는 GPU-driven renderer에서 material과 resource를 shader가 직접 선택하게 만드는 기반이고, 다음 단계는 indirect draw, compute culling, command buffer generation이다.

## 10. 참고 키워드

- Bindless Resources
- Bindless Texture
- Descriptor Indexing
- Vulkan `VK_EXT_descriptor_indexing`
- DirectX 12 Descriptor Heap
- Metal Argument Buffer
- WebGPU Bind Group Limit
- GPU-Driven Rendering
- Indirect Draw
- Material Buffer
- Resource Table
- Non-uniform Indexing
- Shader Resource View, SRV
- Unordered Access View, UAV
- Sparse Texture
- Virtual Texturing
- Sparse Voxel
- Volume Rendering
- Scientific Visualization
