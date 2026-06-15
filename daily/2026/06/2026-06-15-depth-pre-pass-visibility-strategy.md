---
title: "Depth Pre-Pass vs Forward/Deferred Visibility Strategy"
date: "2026-06-15"
category: "Graphics"
tags: ["GPU", "Rendering Pipeline", "Depth Pre-Pass", "Forward Rendering", "Deferred Rendering", "Visibility", "Hi-Z", "Early-Z", "Overdraw", "GPU-Driven Rendering"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-06-15 - Depth Pre-Pass vs Forward/Deferred Visibility Strategy

## 1. 오늘의 개념

**Depth Pre-Pass**는 본격적인 shading pass 전에 depth만 먼저 렌더링해 depth buffer를 채우는 렌더링 전략이다. 이후 main pass에서는 이미 만들어진 depth buffer를 이용해 가려진 fragment의 shading 비용을 줄이고, Hi-Z occlusion culling, screen-space effect, deferred pass, transparency sorting 같은 후속 단계에서 visibility 정보를 재사용할 수 있다.

하지만 depth pre-pass는 항상 이득이 아니다. 같은 geometry를 최소 한 번 더 그려야 하므로 vertex processing, rasterization, command submission 비용이 추가된다. 따라서 중요한 질문은 다음이다.

> Depth를 먼저 만드는 비용보다, 이후 pass에서 절약되는 shading / overdraw / culling 비용이 더 큰가?

Forward renderer, deferred renderer, GPU-driven renderer는 depth를 사용하는 방식이 다르다. 이 차이를 이해해야 renderer architecture를 설계할 때 depth pre-pass를 넣을지, partial pre-pass를 사용할지, deferred G-buffer pass에 depth 생성을 맡길지 판단할 수 있다.

## 2. 한 줄 핵심

**Depth Pre-Pass는 depth visibility를 먼저 확정해 overdraw와 shading 낭비를 줄이는 전략이지만, forward/deferred/GPU-driven pipeline마다 비용과 이득이 달라지는 trade-off 기반 설계다.**

## 3. 왜 중요한가

Depth buffer는 단순히 픽셀의 깊이를 저장하는 attachment가 아니다. Modern renderer에서 depth는 visibility pipeline의 중심 데이터다.

Depth는 다음 단계에서 재사용된다.

- Early-Z / Late-Z depth test
- Hi-Z occlusion culling
- Screen-space reflection, SSAO, SSGI
- Deferred lighting
- Shadow receiver depth comparison
- Transparency sorting 보조
- Decal / volumetric / fog pass
- TAA reprojection / motion vector와의 결합
- GPU-driven visible list generation

즉 depth를 언제 만들고, 어떤 precision으로 저장하고, 어떤 pass에서 읽고, 어떤 pass에서 write할지 결정하는 것은 renderer 전체 구조에 영향을 준다.

Graphics engineer 관점에서 depth pre-pass는 “성능 옵션”이 아니라 **visibility data ownership을 어느 pass가 가질지 정하는 architecture decision**이다.

## 4. 구현 관점

### 4.1 Depth Pre-Pass 기본 구조

Depth pre-pass는 보통 color output 없이 depth만 기록한다.

```cpp
// Pass 1: Depth only
BindDepthOnlyPipeline();
for (VisibleObject obj : visibleObjects)
{
    BindMesh(obj.mesh);
    DrawIndexed(obj.indexCount);
}

// Pass 2: Main shading
BindMainPipeline();
SetDepthFunc(EqualOrLessEqual);
for (VisibleObject obj : visibleObjects)
{
    BindMaterial(obj.material);
    DrawIndexed(obj.indexCount);
}
```

첫 번째 pass에서 가장 가까운 surface의 depth를 채워두면, 두 번째 pass에서는 뒤쪽 fragment가 depth test에서 빠르게 제거된다. 이때 GPU는 가능한 경우 fragment shader 실행 전에 Early-Z로 많은 fragment를 버릴 수 있다.

### 4.2 Early-Z와 Late-Z

**Early-Z**는 fragment shader 실행 전에 depth test를 수행해 가려진 fragment의 shading을 생략하는 최적화다. 반대로 **Late-Z**는 fragment shader 이후에 depth test가 수행되는 경우다.

Early-Z가 깨질 수 있는 대표 상황은 다음과 같다.

- fragment shader가 depth를 직접 쓰는 경우 (`gl_FragDepth`)
- alpha test / discard 사용
- UAV / image store 등 side effect가 있는 경우
- blending / transparency pass
- depth state가 복잡한 경우

Depth pre-pass는 main shading pass에서 depth가 이미 채워져 있으므로 Early-Z 효율을 높이기 쉽다. 특히 fragment shader가 무겁고 overdraw가 많은 장면에서 이득이 크다.

### 4.3 Forward Rendering에서의 전략

Forward rendering은 geometry pass에서 곧바로 lighting과 material shading을 수행한다. 따라서 overdraw가 많으면 같은 pixel에 대해 비싼 lighting 계산이 여러 번 실행될 수 있다.

Forward renderer에서 depth pre-pass가 유리한 경우는 다음과 같다.

- fragment shader가 비싸다.
- PBR lighting, IBL, multiple lights, clearcoat, subsurface 등 material cost가 높다.
- foliage, character hair, terrain, dense mesh로 overdraw가 많다.
- MSAA를 사용해 shading cost가 커진다.
- Hi-Z occlusion culling용 depth pyramid가 필요하다.

하지만 단점도 있다.

- geometry를 두 번 rasterize한다.
- vertex shader 비용이 증가한다.
- skinned mesh는 pre-pass에서도 skinning 비용이 든다.
- draw call / command recording이 늘어난다.

따라서 forward renderer에서는 full depth pre-pass 대신 **partial depth pre-pass**를 사용하는 경우가 많다. 예를 들어 큰 occluder, opaque static mesh, terrain만 먼저 depth에 기록하고, 작거나 shader가 단순한 object는 pre-pass에서 제외할 수 있다.

### 4.4 Deferred Rendering에서의 전략

Deferred rendering은 G-buffer pass에서 position/depth, normal, albedo, roughness, metallic 같은 material 정보를 먼저 기록하고, lighting은 screen-space pass에서 처리한다.

Deferred renderer에서는 G-buffer pass 자체가 depth를 만든다. 따라서 depth pre-pass를 항상 별도로 넣는 것은 중복 비용일 수 있다.

Deferred에서 depth pre-pass가 유리한 경우는 다음과 같다.

- G-buffer shader가 무겁다.
- overdraw가 매우 많다.
- material texture sampling 비용이 크다.
- G-buffer bandwidth 낭비를 줄이고 싶다.
- Hi-Z를 G-buffer 전에 만들어 GPU occlusion culling에 사용하고 싶다.

그러나 deferred는 이미 첫 geometry pass에서 depth를 생성하므로, 많은 경우 depth pre-pass 없이 G-buffer pass의 depth를 후속 pass에서 재사용한다.

Deferred renderer의 핵심은 다음이다.

> Depth pre-pass를 별도로 둘 것인가, 아니면 G-buffer pass를 depth-producing pass로 사용할 것인가?

이 선택은 scene complexity, material cost, bandwidth, overdraw, occlusion culling 필요성에 따라 달라진다.

### 4.5 Z-Prepass, Depth Priming, Partial Pre-Pass

실무에서는 단순히 “pre-pass on/off”가 아니라 여러 변형이 있다.

#### Full Z-Prepass

모든 opaque geometry를 depth-only로 먼저 그린다.

장점:

- Early-Z 효율 최대화
- 정확한 depth buffer 확보
- Hi-Z pyramid 생성에 좋음

단점:

- geometry pass 비용이 거의 2배에 가까워질 수 있음
- vertex-bound scene에서는 손해

#### Partial Pre-Pass

큰 occluder나 static geometry만 먼저 그린다.

장점:

- 주요 occlusion depth 확보
- 비용 대비 이득이 좋을 수 있음
- foliage, small props, transparent object 제외 가능

단점:

- visibility coverage가 완전하지 않음
- scene classification이 필요함

#### Depth Priming

일부 renderer에서는 depth를 먼저 채운 뒤, main pass에서 depth equal test로 정확한 surface만 shading한다. 이는 overdraw를 줄이지만 depth precision, polygon offset, alpha-tested geometry 처리에 신경 써야 한다.

### 4.6 Hi-Z와의 연결

어제 본 Hi-Z Occlusion Culling은 depth pyramid가 필요하다. 이 depth pyramid의 source가 무엇인지가 중요하다.

가능한 source는 다음과 같다.

- current frame depth pre-pass
- previous frame depth buffer
- G-buffer depth
- main opaque pass 이후 depth

Current frame depth pre-pass를 사용하면 정확도는 높지만 비용이 든다. Previous frame depth를 사용하면 비용은 줄지만 camera motion이나 dynamic object에서 false occlusion이 생길 수 있다. Deferred renderer에서는 G-buffer depth를 pyramid로 만들어 후속 pass에서 사용할 수 있다.

즉 Hi-Z를 어디에 넣을지는 depth production strategy와 직접 연결된다.

### 4.7 GPU-Driven Rendering에서의 전략

GPU-driven renderer에서는 depth pre-pass가 단순 shading 최적화가 아니라 draw list generation과 연결된다.

대표 흐름은 다음과 같다.

1. Previous frame depth pyramid 또는 current depth pre-pass를 준비한다.
2. Compute shader가 frustum + Hi-Z culling을 수행한다.
3. Visible object / meshlet / brick list를 compact한다.
4. Indirect draw command를 생성한다.
5. Main pass가 indirect draw를 수행한다.
6. 생성된 depth를 다음 frame culling source로 재사용한다.

이때 depth는 frame 간 temporal visibility cache처럼 사용된다.

Sparse voxel, octree, large CFD block renderer에서는 이 구조가 특히 중요하다. 모든 node나 block을 CPU가 순회하는 대신, GPU가 depth pyramid와 screen-space bounds를 이용해 visible block list를 만든다.

### 4.8 Memory / Bandwidth 관점

Depth pre-pass는 color target을 쓰지 않아 비교적 가볍지만, 완전히 공짜는 아니다.

비용 요소는 다음과 같다.

- depth attachment write bandwidth
- vertex/index buffer read
- vertex shader execution
- rasterization cost
- command buffer / draw submission
- depth pyramid generation cost
- synchronization barrier

특히 deferred renderer에서는 G-buffer bandwidth가 큰 병목일 수 있다. Depth pre-pass로 overdraw를 줄이면 G-buffer write 낭비를 줄일 수 있지만, depth pre-pass 자체의 geometry 비용과 비교해야 한다.

반대로 forward renderer에서는 heavy shading cost를 줄일 수 있어 depth pre-pass 이득이 더 명확할 수 있다.

## 5. 내 관심 분야와 연결

### CFD / scientific visualization

CFD visualization에서는 volume block, slice mesh, iso-surface, streamline, particle trace 등 다양한 primitive가 섞인다. 이때 depth pre-pass 전략은 단순 mesh renderer보다 복잡하다.

예를 들어 iso-surface mesh가 강한 occluder 역할을 한다면 먼저 depth에 기록하고, 뒤쪽 volume brick ray marching을 줄일 수 있다. 반대로 투명 volume rendering은 depth write를 조심해야 한다.

정리하면 다음 관점이 중요하다.

- Opaque iso-surface는 depth pre-pass 후보
- Transparent volume은 depth test는 하되 depth write는 제한
- Streamline / particle은 depth-aware blending 필요
- Volume brick은 Hi-Z 기반으로 visibility culling 가능

### Sparse voxel / octree / NanoVDB

Sparse voxel renderer에서는 depth pre-pass가 “surface depth를 먼저 확보해 ray marching cost를 줄이는 전략”으로 연결된다.

예를 들어 hybrid renderer에서 mesh surface를 먼저 depth에 기록하고, volume pass에서 depth buffer를 termination depth로 사용하면 ray marching step 수를 줄일 수 있다. 또한 depth pyramid를 사용해 가려진 brick을 dispatch 전에 제거할 수 있다.

즉 sparse voxel에서는 depth가 다음 두 역할을 가진다.

1. Screen-space occlusion culling source
2. Ray marching termination / compositing boundary

### Game engine architecture

Game engine에서는 depth pre-pass 선택이 renderer mode와 직접 연결된다.

- Forward+: depth pre-pass로 light culling과 tile/cluster depth range 계산
- Deferred: G-buffer depth를 후속 lighting / SSAO / SSR에서 재사용
- GPU-driven: depth pyramid로 occlusion culling과 indirect draw list 생성
- Transparent pass: opaque depth를 기준으로 sorting / soft particles / fog 처리

따라서 depth pre-pass를 이해하면 forward, deferred, clustered, GPU-driven renderer가 분리된 개념이 아니라 depth visibility를 중심으로 연결된다는 것을 볼 수 있다.

## 6. 머릿속에 남길 질문 3개

1. Forward renderer에서 depth pre-pass가 deferred renderer보다 더 자주 유리해질 수 있는 이유는 무엇인가?
2. Depth pre-pass가 overdraw를 줄이지만 vertex-bound scene에서는 손해가 될 수 있는 이유는 무엇인가?
3. Sparse voxel volume renderer에서 depth buffer는 occlusion culling 외에 ray marching termination에 어떻게 사용될 수 있는가?

## 7. graphics engineer 면접 질문 1개와 답변

### Q. Depth Pre-Pass는 언제 사용하는 것이 좋고, 언제 오히려 손해가 될 수 있나요?

**A.** Depth Pre-Pass는 main shading 전에 depth-only pass를 수행해 depth buffer를 먼저 채우는 전략입니다. 이후 main pass에서 Early-Z를 통해 가려진 fragment의 fragment shader 실행을 줄일 수 있습니다. Fragment shader가 무겁고 overdraw가 많은 forward renderer, PBR material이 많은 장면, Hi-Z occlusion culling이나 depth pyramid가 필요한 GPU-driven renderer에서는 유리할 수 있습니다.

반대로 vertex processing이 병목인 장면, geometry 수가 많고 shader가 단순한 장면, skinned mesh처럼 pre-pass에서도 vertex cost가 큰 장면에서는 손해가 될 수 있습니다. Deferred renderer에서는 G-buffer pass 자체가 depth를 생성하므로 별도 depth pre-pass가 항상 필요한 것은 아닙니다. 따라서 depth pre-pass는 on/off 옵션이 아니라, scene이 fragment-bound인지 vertex-bound인지, overdraw가 얼마나 큰지, depth pyramid를 어느 시점에 필요한지에 따라 결정해야 합니다.

## 8. 포트폴리오 / 커리어 연결

Depth Pre-Pass 전략은 포트폴리오에서 renderer architecture 판단력을 보여줄 수 있다.

좋은 표현은 다음과 같다.

> “Renderer에서 depth buffer를 단순한 depth test attachment가 아니라, Early-Z, Hi-Z occlusion, screen-space effect, GPU-driven draw list generation에 재사용되는 visibility data로 설계했습니다.”

특히 네 경험과 연결하면 다음 메시지가 강하다.

- CFD / VTK large data visualization에서 block visibility를 depth 기반으로 줄이는 사고
- OpenGL renderer에서 deferred / forward pass의 depth ownership 설계
- Sparse voxel / volume rendering에서 depth buffer를 ray termination과 occlusion source로 활용
- WebGPU / Vulkan으로 넘어갈 때 depth pyramid와 compute culling을 연결할 수 있는 역량

면접에서는 “depth pre-pass는 무조건 성능 향상”이라고 말하기보다, 비용 모델을 함께 말하는 것이 좋다.

> “Depth pre-pass는 fragment shading과 overdraw 비용을 줄이는 대신 geometry를 한 번 더 처리하는 비용을 추가하므로, scene이 fragment-bound인지 vertex-bound인지에 따라 선택해야 합니다.”

## 9. 내일 이어서 볼 개념

**Forward+ / Clustered Shading**

Depth pre-pass를 이해한 다음에는 Forward+ 또는 Clustered Shading으로 이어지는 것이 자연스럽다. 이 기법들은 depth buffer를 이용해 screen tile 또는 3D cluster의 depth range를 만들고, 각 cluster에 영향을 주는 light list를 구성한다. 즉 depth visibility가 lighting optimization으로 확장되는 단계다.

## 10. 참고 키워드

- Depth Pre-Pass
- Z-Prepass
- Depth Priming
- Early-Z
- Late-Z
- Overdraw
- Forward Rendering
- Deferred Rendering
- G-buffer
- Hi-Z Occlusion Culling
- Depth Pyramid
- GPU-Driven Rendering
- Indirect Draw
- Visibility Buffer
- Forward+
- Clustered Shading
- Depth Range
- Reversed-Z
- Depth Precision
- Screen-space Effects
- Ray Marching Termination
