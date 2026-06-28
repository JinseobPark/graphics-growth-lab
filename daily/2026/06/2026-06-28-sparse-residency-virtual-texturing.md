---
title: "Sparse Residency and Virtual Texturing"
date: "2026-06-28"
category: "Graphics"
tags: ["GPU", "Rendering Pipeline", "Sparse Residency", "Virtual Texturing", "Tiled Resources", "Texture Streaming", "Page Table", "Sparse Voxel", "Scientific Visualization"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-06-28 - Sparse Residency and Virtual Texturing

## 1. 오늘의 개념

**Sparse Residency**와 **Virtual Texturing**은 매우 큰 texture, volume, field, voxel, material data를 한 번에 GPU memory에 모두 올리지 않고, 실제로 필요한 tile 또는 page만 물리 GPU memory에 resident하게 유지하는 resource management 구조다.

전통적인 texture 사용 방식에서는 texture 전체를 GPU memory에 올린다. 예를 들어 16K x 16K terrain texture, huge material atlas, 3D volume texture, sparse voxel field를 모두 dense texture로 올리면 memory가 빠르게 한계에 도달한다.

Virtual Texturing은 이 문제를 page 단위로 나눈다.

- Virtual address space: 매우 큰 논리 texture 공간
- Physical page pool: 실제 GPU memory에 올라온 tile/page 저장소
- Page table: virtual page가 physical page pool의 어디에 있는지 매핑
- Feedback pass: 현재 frame에서 어떤 page가 필요한지 기록
- Streaming system: 필요한 page를 disk/CPU/GPU로 로드하고 mapping 갱신

핵심 변화는 다음이다.

> Resource 전체를 resident하게 유지하는 구조에서, 현재 view와 shading에 필요한 page만 resident하게 유지하는 구조로 이동한다.

Clipmap이 카메라 주변의 계층적 LOD window를 유지하는 구조라면, sparse residency는 더 일반적으로 “필요한 page만 실제 memory에 매핑하는 구조”다.

## 2. 한 줄 핵심

**Sparse Residency and Virtual Texturing은 거대한 texture/volume/field resource를 page table과 physical page pool로 나누고, 화면에서 필요한 page만 GPU memory에 올려 large-scale rendering의 memory pressure를 줄이는 구조다.**

## 3. 왜 중요한가

현대 renderer는 더 이상 작은 texture 몇 개만 다루지 않는다. Large terrain, photogrammetry scan, open-world material, volume simulation, sparse voxel, semiconductor process field, CFD scalar/vector field는 모두 GPU memory보다 훨씬 큰 data를 요구할 수 있다.

모든 데이터를 dense resource로 올리는 방식은 다음 문제를 만든다.

- GPU memory usage 폭증
- texture upload bandwidth 증가
- loading time 증가
- cache locality 저하
- high-resolution data 사용 제약
- large-scale visualization에서 domain size 제한

Sparse residency와 virtual texturing은 이 문제를 해결하기 위해 resource를 page 단위로 관리한다. 화면에서 보이지 않는 page, 너무 멀어 낮은 mip만 필요한 page, 아직 접근하지 않은 page는 물리 memory에 올리지 않는다.

Graphics engineer 관점에서는 이 개념이 단순 texture streaming이 아니라, **GPU address space와 memory residency를 분리하는 architecture**라는 점이 중요하다.

## 4. 구현 관점

### 4.1 Virtual texture의 기본 구조

Virtual texture는 거대한 논리 texture를 page 단위로 나눈다. 예를 들어 65536 x 65536 texture를 128 x 128 tile로 나누면, 실제로 필요한 tile만 physical page pool에 올릴 수 있다.

구조는 다음과 같다.

```text
Virtual Texture Space
  ├─ virtual page (x, y, mip)
  ├─ virtual page (x, y, mip)
  └─ ...

Physical Page Pool
  ├─ physical tile 0
  ├─ physical tile 1
  ├─ physical tile 2
  └─ ...

Page Table
  virtual page id → physical tile id
```

Shader는 texture coordinate를 바로 physical texture coordinate로 사용하지 않는다. 먼저 page table을 조회해 해당 virtual page가 physical pool의 어디에 있는지 찾고, 그 위치에서 sample한다.

### 4.2 Page table lookup

Virtual texturing shader의 개념적 흐름은 다음과 같다.

1. UV와 mip level을 이용해 virtual page coordinate를 계산한다.
2. Page table texture/buffer에서 physical page id를 읽는다.
3. Page 내부 local coordinate를 계산한다.
4. Physical page pool texture에서 실제 texel을 sample한다.

단순화하면 다음과 같다.

```glsl
VirtualPage page = ComputeVirtualPage(uv, mip);
PhysicalPage phys = pageTable[page.id];
vec2 localUV = ComputeLocalPageUV(uv);
vec4 value = SamplePhysicalPagePool(phys, localUV);
```

실제 구현에서는 filtering, mip transition, page border, anisotropic filtering, missing page fallback 때문에 더 복잡하다.

### 4.3 Feedback pass

Virtual texturing은 어떤 page가 필요한지 알아야 한다. 이를 위해 feedback pass를 사용한다.

Feedback pass에서는 shading 중 접근한 virtual page id를 별도 buffer나 texture에 기록한다. CPU 또는 GPU streaming system은 이 feedback을 읽고 다음 frame에 필요한 page를 요청한다.

흐름은 다음과 같다.

1. Frame N에서 shader가 필요한 virtual page id를 기록한다.
2. Feedback buffer를 분석한다.
3. 필요한 page가 physical pool에 없으면 streaming request를 만든다.
4. Disk/CPU memory에서 page data를 준비한다.
5. GPU physical page pool에 upload한다.
6. Page table mapping을 갱신한다.
7. Frame N+1 이후 더 높은 resolution page를 사용할 수 있다.

이 구조는 즉시 완벽한 high-res data가 보이는 것이 아니라, 점진적으로 page가 채워지는 streaming system이다.

### 4.4 Missing page fallback

필요한 page가 아직 resident하지 않을 수 있다. 이때 renderer는 fallback을 제공해야 한다.

대표 전략은 다음과 같다.

- 더 낮은 mip level page 사용
- placeholder color 사용
- 이전 frame page 유지
- parent page sample
- streaming priority 증가

좋은 virtual texturing system은 missing page를 visual artifact로 크게 드러내지 않는다. 보통 낮은 mip page는 미리 resident하게 유지하고, high-resolution page가 늦게 들어와도 coarse한 결과는 먼저 보이게 한다.

### 4.5 Page border와 filtering 문제

Tile 단위 texture sampling에서 가장 까다로운 문제 중 하나는 filtering이다. Bilinear / trilinear / anisotropic filtering은 주변 texel과 mip level을 참조한다. 그런데 tile 경계에서 이웃 page가 physical pool의 다른 위치에 있거나 아직 resident하지 않을 수 있다.

이를 해결하기 위해 page마다 border texel을 둔다.

예를 들어 128x128 logical tile을 저장할 때 실제 physical page는 132x132처럼 border를 포함할 수 있다. Border에는 이웃 page의 texel을 복사해 filter seam을 줄인다.

고려할 점은 다음이다.

- bilinear filtering border
- mip transition seam
- anisotropic filtering footprint
- page table LOD 선택
- parent fallback과 child page transition

Virtual texturing은 page table만 만들면 끝나는 구조가 아니라, texture filtering과 LOD continuity까지 관리해야 한다.

### 4.6 Sparse residency API 관점

Sparse residency는 API 또는 hardware가 resource의 일부 page만 physical memory에 bind할 수 있게 해주는 기능이다.

대표 개념은 다음과 같다.

- Vulkan sparse resources
- DirectX 12 tiled resources
- Metal sparse textures
- OpenGL sparse texture extension

이 기능을 사용하면 거대한 virtual texture object를 만들고, 일부 tile만 실제 memory에 bind할 수 있다. Manual virtual texturing처럼 shader에서 page table lookup을 직접 할 수도 있고, hardware sparse texture 기능을 활용할 수도 있다.

WebGPU는 portability와 safety 때문에 native sparse residency를 직접 노출하지 않는 경우가 많다. 하지만 storage texture, texture array, atlas, page table buffer를 조합해 virtual texturing-like 구조를 학습할 수 있다.

### 4.7 Sparse voxel / volume data와 연결

Virtual texturing 사고는 2D texture에만 적용되지 않는다. 3D volume과 voxel field에도 적용할 수 있다.

3D virtual volume 구조는 다음과 같다.

- virtual 3D volume space
- brick/page 단위 physical pool
- page table 또는 sparse tree
- ray marching 중 brick lookup
- missing brick fallback
- LOD level별 brick resolution

Sparse voxel renderer에서는 voxel brick이 virtual texture page와 비슷한 역할을 한다. Ray가 어떤 region을 통과할 때 필요한 brick만 resident하면 된다.

CFD field visualization에서도 pressure, velocity, temperature field를 block/page 단위로 관리할 수 있다. 보이는 region이나 관심 영역만 high-resolution으로 resident하게 유지하고, 나머지는 coarse field로 대체할 수 있다.

### 4.8 성능과 메모리 trade-off

Sparse residency와 virtual texturing은 memory를 줄이지만 복잡도를 증가시킨다.

장점:

- 매우 큰 resource를 다룰 수 있다.
- GPU memory pressure를 줄인다.
- high-resolution data를 필요할 때만 로드한다.
- clipmap / LOD / streaming과 결합하기 좋다.

비용:

- page table lookup 비용
- feedback pass 비용
- streaming latency
- page upload bandwidth
- filtering seam 처리
- page eviction policy 설계
- debugging 복잡도

즉 virtual texturing은 “무조건 빠른 texture”가 아니라, **memory capacity 문제를 streaming과 indirection 문제로 바꾸는 구조**다.

## 5. 내 관심 분야와 연결

### CFD / scientific visualization

CFD 대용량 field에서는 전체 domain을 dense GPU texture나 buffer로 올리기 어려울 수 있다. Virtual texturing 구조를 사용하면 field block을 page 단위로 나누고, 현재 camera와 visualization mode에 필요한 block만 resident하게 유지할 수 있다.

가능한 구조는 다음과 같다.

- pressure / velocity / temperature field page table
- visible block feedback
- ROI 중심 high-resolution page streaming
- volume ray marching 중 brick lookup
- missing page는 coarse parent block 사용
- scalar gradient가 큰 영역에 streaming priority 부여

Scientific visualization에서는 단순히 보이는 page뿐 아니라 data importance도 streaming priority에 포함해야 한다.

### Sparse voxel / octree / NanoVDB

Sparse voxel에서는 brick residency가 핵심이다. 모든 voxel brick을 GPU에 올리는 대신, visible brick이나 ray traversal에 필요한 brick만 resident하게 유지한다.

이 구조는 다음과 연결된다.

- sparse page table
- voxel brick pool
- clipmap level
- SDF / density brick streaming
- ray marching missing brick fallback
- marching cubes active brick loading

NanoVDB 같은 sparse volume representation을 GPU renderer와 연결할 때도 page/brick residency 사고가 중요하다.

### Game engine architecture

Game engine에서는 virtual texturing이 open-world material과 terrain texture, megatexture, virtual shadow map, sparse voxel GI와 연결된다.

- virtual texture material streaming
- tiled resources
- sparse texture residency
- virtual shadow map page caching
- terrain texture clipmap
- decal / material atlas paging

면접에서는 “texture streaming”이라고만 말하기보다, page table, physical pool, feedback, residency, fallback을 함께 설명하는 것이 좋다.

## 6. 머릿속에 남길 질문 3개

1. Virtual Texturing이 큰 texture를 전부 GPU memory에 올리지 않고도 sampling할 수 있는 이유는 무엇인가?
2. Page table, physical page pool, feedback pass는 각각 어떤 역할을 하는가?
3. CFD volume field나 sparse voxel brick을 virtual texture page처럼 관리하면 어떤 장점과 복잡도가 생기는가?

## 7. graphics engineer 면접 질문 1개와 답변

### Q. Virtual Texturing과 Sparse Residency는 무엇이며, large-scale rendering에서 어떤 trade-off가 있나요?

**A.** Virtual Texturing은 매우 큰 texture나 volume resource를 page 단위로 나누고, 실제로 필요한 page만 physical GPU memory에 올려 사용하는 구조입니다. Shader는 virtual coordinate로 접근하지만, page table을 통해 해당 virtual page가 physical page pool의 어디에 있는지 찾고 실제 data를 sample합니다. Sparse Residency는 API나 hardware가 resource의 일부 page만 실제 memory에 bind할 수 있게 해주는 기능입니다.

장점은 전체 texture나 volume을 GPU memory에 올리지 않고도 large-scale data를 다룰 수 있다는 점입니다. Terrain, open-world material, sparse voxel, CFD volume field처럼 resource가 큰 경우에 유리합니다. 단점은 page table lookup, feedback pass, streaming latency, page upload bandwidth, filtering seam, missing page fallback, eviction policy 같은 복잡도가 증가한다는 점입니다. 따라서 이 구조는 단순 성능 최적화라기보다 memory capacity 문제를 streaming과 indirection 문제로 바꾸는 architecture로 이해해야 합니다.

## 8. 포트폴리오 / 커리어 연결

Sparse Residency and Virtual Texturing은 포트폴리오에서 다음 메시지를 만든다.

> “나는 large-scale texture/volume/field data를 dense resource로만 다루지 않고, page table과 residency 기반으로 GPU memory를 설계하는 방식을 이해한다.”

네 배경과 연결하면 다음 표현이 좋다.

- CFD / VTK 대용량 field를 block/page 단위로 streaming하는 사고
- Sparse voxel / NanoVDB renderer에서 brick pool과 page table을 설계하는 관점
- WebGPU에서는 manual page table + texture atlas 방식으로 개념을 학습 가능
- Vulkan / DX12에서는 sparse resource / tiled resource로 확장 가능
- Semiconductor 3D visualization에서 layer/material/field data를 page 단위로 관리하는 구조

면접에서는 다음처럼 말할 수 있다.

> “Virtual Texturing은 큰 resource 전체를 올리는 대신 page table을 통해 필요한 tile만 physical memory에 resident하게 유지하는 방식입니다. 이 사고는 texture뿐 아니라 volume brick, sparse voxel, CFD field block streaming에도 확장할 수 있습니다.”

## 9. 내일 이어서 볼 개념

**Page Table and Indirection in GPU Rendering**

Sparse residency와 virtual texturing 다음에는 page table과 indirection 구조를 더 깊게 보는 것이 자연스럽다. Texture page, voxel brick, material table, bindless resource table은 모두 shader가 index 또는 address를 통해 실제 data를 찾아가는 indirection 구조를 공유한다.

## 10. 참고 키워드

- Sparse Residency
- Virtual Texturing
- Tiled Resources
- Sparse Texture
- Page Table
- Physical Page Pool
- Residency
- Feedback Pass
- Texture Streaming
- Page Fault
- Missing Page Fallback
- Tile Border
- Anisotropic Filtering
- Virtual Shadow Map
- Sparse Voxel Brick Pool
- Volume Streaming
- Clipmap
- GPU Memory Management
- Vulkan Sparse Resources
- DirectX 12 Tiled Resources
- Metal Sparse Texture
