---
title: "Depth-Aware Reprojection Confidence and Surface Identity Validation"
date: "2026-07-20"
category: "Graphics"
tags: ["GPU", "Temporal Rendering", "TAA", "Reprojection", "History Validation", "Surface Identity", "Compute Shader", "Scientific Visualization"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-07-20 - Depth-Aware Reprojection Confidence and Surface Identity Validation

## 1. 오늘의 개념

**Depth-Aware Reprojection Confidence**는 motion vector로 찾은 이전 프레임의 history가 현재 표면과 얼마나 잘 대응하는지를 0~1 값으로 나타내는 방식이다. 전날의 disocclusion detection이 history를 사용할지 말지를 판정했다면, 오늘의 주제는 경계와 불확실성을 포함해 history를 얼마나 믿을지 결정한다.

**Surface Identity Validation**은 depth뿐 아니라 normal, object/instance ID, material class, motion consistency, geometry revision을 검사해 재투영된 값이 같은 표면에서 왔는지 확인한다. 화면 좌표와 깊이가 비슷하더라도 실제 표면의 정체성은 다를 수 있다.

## 2. 한 줄 핵심

**Reprojection confidence는 depth·normal·identity·motion 신호를 결합해 history의 표면 대응 품질을 수치화하고, 그 값을 temporal weight와 metadata 수명에 반영하는 장치다.**

## 3. 왜 중요한가

Binary rejection은 명확한 disocclusion에서는 강하지만 silhouette, sub-pixel motion, depth quantization, camera jitter가 섞인 영역에서는 결과가 프레임마다 흔들릴 수 있다. 이때 ghosting을 막는 대신 shimmer와 재수렴 지연이 발생한다.

연속적인 confidence를 사용하면 작은 오차에서는 history를 유지하고, 불확실한 구간에서는 history length를 줄이며, 명확한 mismatch에서만 완전히 reset할 수 있다. NVIDIA NRD도 application-provided history confidence를 temporal lag를 줄이는 핵심 입력으로 사용한다.

Depth-only validation에는 한계가 있다. 서로 가까운 평행 표면, thin shell, deforming mesh, animated normal은 깊이가 유사해도 history를 공유하면 안 될 수 있다. 따라서 validation을 다음처럼 나누는 것이 유리하다.

- **Geometric correspondence:** depth, position, normal, motion
- **Semantic identity:** object/instance ID, material class, geometry revision

## 4. 구현 관점

### Reprojection과 depth confidence

현재 픽셀의 motion vector로 이전 좌표를 구한다.

`prevUV = currUV + motionVector`

Current depth로 현재 surface position을 복원한 뒤 previous camera로 투영하면 expected previous depth를 얻을 수 있다. 이를 reprojected 위치의 previous depth와 비교한다.

`d = abs(zPrevSample - zPrevExpected) / (epsilonAbs + epsilonRel * abs(zPrevExpected))`

`Cdepth = 1 - smoothstep(t0, t1, d)`

고정 depth epsilon은 perspective projection과 depth precision 변화에 취약하다. Linear view-space depth, world-space separation, reversed-Z와 depth format을 반영해야 한다. AMD FSR2의 Depth Clip도 reconstructed previous depth와 현재 표면의 대응을 비교해 disocclusion 값을 만든다.

Depth texture를 bilinear filtering하면 foreground와 background가 섞인 가짜 중간값이 생길 수 있다. Point sample은 혼합을 막지만 sub-pixel motion에 민감하고, 2×2 또는 3×3 search는 reprojection 오차에 강한 대신 다른 표면을 허용할 위험이 있다.

### Normal과 identity validation

Normal confidence는 다음처럼 구성할 수 있다.

`Cnormal = smoothstep(n0, n1, dot(Ncurrent, Nprevious))`

Normal은 동일한 좌표계에서 비교해야 한다. Packed normal의 정밀도보다 threshold를 지나치게 엄격하게 잡으면 false rejection이 증가한다.

Object/instance ID는 강한 hard gate다.

`Cidentity = currentID == previousID ? 1 : 0`

단, ID는 object lifetime 동안 안정적이어야 한다. Draw order나 triangle index처럼 매 프레임 바뀌는 값은 temporal identity로 사용할 수 없다. Marching Cubes에서는 triangle ID보다 source voxel cell, brick ID, simulation region과 generation 값을 사용하는 편이 낫다.

### Motion consistency와 confidence 결합

주변 velocity와 중심 velocity의 차이가 크면 reprojection 후보가 불안정할 수 있다. 다만 deformation과 rotation도 motion divergence를 만들기 때문에 보통 hard reject가 아니라 soft confidence로 사용한다.

치명적인 mismatch가 평균에 가려지지 않도록 hard gate와 soft confidence를 분리한다.

`Chard = validViewport * validObjectID * validRevision`

`Csoft = pow(Cdepth, wd) * pow(Cnormal, wn) * pow(Cmotion, wm)`

`Chistory = Chard * Csoft`

보수적인 pipeline은 soft 값에 product 대신 minimum을 사용할 수 있다. Product는 작은 불확실성이 누적되어 빠르게 confidence를 낮추고, minimum은 가장 약한 신호가 전체를 지배한다.

최종 confidence는 color blending뿐 아니라 metadata에도 반영해야 한다.

`historyLength = min(historyLength, maxHistoryLength * Chistory)`

`Chistory = 0`이면 color, moments, variance, lock status, sample count를 함께 reset한다. Color만 교체하고 metadata를 유지하면 새 표면을 이미 수렴한 영역으로 오판한다.

### GPU memory와 pass 구성

- Confidence: `R8_UNORM`이면 bandwidth와 debug 편의의 균형이 좋다.
- Normal: 기존 G-buffer의 packed normal을 재사용한다.
- Object ID: `R16_UINT` 또는 `R32_UINT`를 exact load한다.
- Revision: per-object buffer 또는 compact integer texture로 관리한다.

Integer ID texture는 filtered sampling이 아니라 `textureLoad` 계열의 exact access가 필요하다. Vulkan, DirectX, Metal, WebGPU 모두 resource format과 shader sample type을 정확히 맞춰야 한다.

Validation 결과를 여러 pass가 공유하면 독립 compute pass가 유리하고, accumulation에서만 사용하면 pass fusion으로 intermediate bandwidth를 줄일 수 있다. 실제 병목은 ALU보다 previous depth/normal/ID neighborhood texture fetch일 가능성이 높다.

Debug view는 expected/sample depth, normal dot, ID mismatch, motion divergence, 각 confidence channel, final history confidence, history length를 분리해 보여줘야 한다. Ghosting은 false acceptance, shimmer와 noise 증가는 false rejection을 우선 의심한다.

## 5. 내 관심 분야와 연결

CFD와 scientific visualization에서는 geometry가 같더라도 timestep과 scalar/vector field가 변할 수 있다. Geometry confidence와 signal confidence를 분리하면 mesh history는 유지하면서 pressure, velocity, temperature colormap history만 빠르게 줄일 수 있다.

Clip plane, iso-value, transfer function 변경은 화면상 depth가 비슷해도 의미를 바꾼다. `visualizationRevision`을 hard gate에 포함하면 과거 contour와 heatmap이 남는 것을 막을 수 있다.

Volume rendering은 단일 surface depth가 없으므로 representative depth, opacity-weighted depth, ray termination depth와 함께 transmittance, density gradient, transfer-function revision을 비교해야 한다.

SDF, level-set, Marching Cubes는 topology가 변할 수 있으므로 triangle ID보다 source cell, brick, field generation, iso-value revision이 안정적이다. Sparse voxel과 octree streaming에서는 `brick coordinate + generation`을 identity로 사용하고, 변경된 brick AABB만 screen-space mask로 투영해 local history를 reset할 수 있다.

## 6. 머릿속에 남길 질문 3개

1. Depth가 비슷하더라도 history가 다른 표면에서 왔다고 판단해야 하는 사례는 무엇인가?
2. Hard identity gate와 soft geometric confidence를 분리해야 하는 이유는 무엇인가?
3. Marching Cubes, volume rendering, streamed voxel에서 triangle/object ID 대신 어떤 stable identity를 사용할 수 있는가?

## 7. Graphics Engineer 면접 질문 1개와 답변

**질문: Temporal accumulation에서 reprojection confidence를 어떻게 설계하고 depth-only validation의 한계를 어떻게 보완하겠습니까?**

**답변:** Current surface를 previous camera로 투영해 expected previous depth를 구하고, sampled previous depth와 비교해 거리와 depth precision을 반영한 연속적인 depth confidence를 계산합니다. 이후 normal dot, stable object/instance ID, material class, motion divergence, geometry revision을 추가합니다. ID와 revision mismatch는 hard reject로 처리하고 depth·normal·motion은 soft confidence로 결합합니다. 최종 confidence는 blending weight뿐 아니라 history length, moments, variance, lock status에도 적용합니다. 구현에서는 integer ID exact load, packed normal precision, neighborhood bandwidth, ping-pong resource lifetime을 고려하고 각 confidence channel을 별도 debug view로 제공합니다.

## 8. 포트폴리오 / 커리어 연결

포트폴리오에서는 단순히 motion vector로 history를 가져왔다고 설명하기보다 다음의 **history validity architecture**를 보여주는 것이 강하다.

`Motion Reprojection → Surface Reconstruction → Depth/Normal/Identity Validation → History Confidence → Metadata Update → Temporal Accumulation`

Binary와 continuous confidence, depth-only와 depth+normal+stable ID, static mesh와 generated topology, separate pass와 fused pass를 비교하면 알고리즘뿐 아니라 GPU resource와 engine event까지 고려한 설계 역량을 드러낼 수 있다.

이 구조는 TAA, temporal upscaling, ray-tracing denoising, CFD visualization, volume rendering을 하나의 공통 관점으로 설명할 수 있어 게임 엔진 및 실시간 렌더링 직무 포트폴리오에 활용도가 높다.

## 9. 내일 이어서 볼 개념

**Temporal Moments and Variance-Guided History Confidence**

Geometry가 같더라도 lighting, radiance, density, transfer function이 빠르게 바뀌는 경우를 다룬다. First/second moment와 variance를 사용해 signal 변화와 noise를 구분하고 temporal accumulation과 spatial filtering radius를 조절하는 방법으로 이어간다.

## 10. 참고 키워드

Depth-Aware Reprojection, Surface Identity Validation, History Confidence, Temporal Accumulation, Previous Depth Reconstruction, Depth Clip, Akeley Separation, Normal Consistency, Stable Object ID, Instance ID, Material Class, Motion Divergence, Geometry Revision, Generation Counter, R8_UNORM, FSR2 Depth Clip, NVIDIA NRD History Confidence, SVGF

참고 자료: AMD GPUOpen FidelityFX Super Resolution 2.3.3 문서, NVIDIA RTX NRD 문서, NVIDIA Research SVGF 논문
