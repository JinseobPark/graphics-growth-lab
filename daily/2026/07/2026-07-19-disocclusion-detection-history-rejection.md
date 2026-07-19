---
title: "Disocclusion Detection and History Rejection"
date: "2026-07-19"
category: "Graphics"
tags: ["GPU", "Temporal Rendering", "TAA", "Disocclusion", "History Rejection", "Depth Buffer", "Motion Vectors", "Compute Shader", "Scientific Visualization"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-07-19 - Disocclusion Detection and History Rejection

## 1. 오늘의 개념

**Disocclusion Detection**은 현재 프레임에서 보이는 픽셀이 이전 프레임에서는 다른 표면에 가려져 있었거나 화면 밖에 있어, 재사용 가능한 history가 없는 영역을 판별하는 과정이다.

TAA, temporal upscaling, denoising은 motion vector로 이전 좌표를 찾는다. 하지만 좌표를 찾았다는 사실만으로 같은 표면의 과거 값임을 보장할 수 없다. 카메라나 물체가 움직여 배경이 새로 드러났다면 이전 좌표에는 foreground의 색과 깊이가 남아 있을 수 있다.

따라서 temporal pipeline은 **Reprojection**과 **History Validation**을 분리해야 한다. 전날의 Temporal Reset Policy가 camera cut 같은 frame-level invalidation을 다뤘다면, 오늘은 정상적인 motion 안에서 발생하는 pixel-level invalidation을 다룬다.

## 2. 한 줄 핵심

**Disocclusion detection은 motion vector가 가리킨 이전 픽셀의 depth와 surface identity를 검사해, 현재 표면과 대응하지 않는 history를 누적 전에 거부하는 temporal validity test다.**

## 3. 왜 중요한가

Motion vector는 history 후보 위치만 제공한다. Foreground 뒤에서 배경이 드러나거나, thin geometry가 사라지거나, marching-cubes topology와 streamed voxel data가 변경되면 이전 좌표에는 다른 signal이 저장되어 있을 수 있다.

잘못된 history는 단순 blur보다 위험하다. 존재하지 않는 silhouette, 과거 density structure, 이전 timestep의 contour가 현재 영상에 남기 때문이다. Temporal rendering에서는 대체로 **잘못된 안정성보다 잠시 증가한 noise가 낫다.**

가장 중요한 검사는 depth correspondence다. Current depth로 현재 surface position을 복원하고 previous camera로 투영해 expected previous depth를 계산한 뒤, motion vector가 가리킨 위치의 previous depth와 비교한다. 차이가 크면 현재 surface가 이전 프레임에서 보이지 않았거나 다른 surface가 해당 위치를 차지했던 것으로 판단한다.

Perspective depth는 비선형이므로 고정 epsilon은 near/far distance에서 의미가 달라진다. Linear view-space depth, world-space separation, local depth range, depth precision을 반영한 threshold가 더 안정적이다. AMD FSR2의 Depth Clip 단계도 previous depth와 현재 표면의 대응을 검사해 disocclusion mask를 생성한다.

## 4. 구현 관점

기본 입력은 current depth, previous depth, motion vector, current/previous camera matrix다. 필요하면 normal과 object ID를 추가한다.

처리 흐름은 다음과 같다.

1. motion vector로 previous UV를 계산한다.
2. previous UV가 viewport 내부인지 검사한다.
3. current depth로 surface position을 복원한다.
4. previous camera 공간의 expected depth를 계산한다.
5. previous depth와 비교한다.
6. depth, normal, ID consistency로 validity를 결정한다.
7. rejection mask 또는 confidence를 temporal weight에 반영한다.

Depth는 color처럼 bilinear filtering하면 서로 다른 두 표면이 섞여 존재하지 않는 중간값이 생길 수 있다. Point sample은 표면 혼합을 막지만 sub-pixel motion에 민감하다. 2×2 또는 3×3 conservative neighborhood는 reprojection 오차에 강하지만 다른 표면을 허용할 위험이 있다. Reconstructed previous depth는 대응을 강화하지만 별도 pass와 resource 비용이 든다.

Silhouette에서는 foreground와 background coverage가 섞인다. **Velocity Dilation**은 camera에 가까운 surface의 motion을 확장해 edge의 reprojection candidate를 안정화한다. 그러나 dilation은 후보 좌표를 보정할 뿐, history validity는 depth validation이 결정한다.

명확한 disocclusion에서는 history color뿐 아니라 history length, lock status, variance, sample count도 함께 reset해야 한다. Metadata를 유지하면 새 표면을 이미 안정된 영역으로 오판할 수 있다.

경계가 불확실할 때는 binary reject 대신 depth confidence, normal confidence, identity confidence를 결합해 연속적인 history confidence를 만들 수 있다. 결과는 `R8_UNORM` mask로 저장하면 bandwidth가 작고 debug가 쉽다. 더 정밀한 누적에는 `R16_FLOAT`가 가능하며, 기존 temporal metadata에 packing하면 resource 수는 줄지만 lifetime과 synchronization이 복잡해진다.

Compute shader는 보통 render resolution에서 8×8 또는 16×16 group으로 실행한다. 비용은 arithmetic보다 current depth, velocity, previous depth neighborhood, optional normal/ID texture access가 지배한다. 여러 temporal pass가 mask를 공유하면 독립 pass가 유리하고, accumulation에서만 사용하면 pass fusion으로 bandwidth를 줄일 수 있다.

Debug view에는 raw/dilated velocity, expected/sample depth, depth delta, rejection mask, history confidence, history length, ID mismatch가 필요하다. False rejection은 shimmer와 재수렴 저하를 만들고, false acceptance는 ghosting과 stale geometry를 만든다. Silhouette에서는 false acceptance를 더 보수적으로 막는 편이 일반적이다.

## 5. 내 관심 분야와 연결

CFD와 scientific visualization에서는 camera motion뿐 아니라 timestep jump, dataset revision, clip plane, transfer function 변경도 history 의미를 깨뜨린다. Depth mask에 simulation revision ID와 dataset generation counter를 결합해야 한다.

Volume rendering은 단일 surface depth가 없을 수 있으므로 representative depth, opacity-weighted depth, ray termination depth, transmittance 변화, density/gradient consistency가 validation 신호가 된다.

SDF, level-set, marching cubes에서는 표면 위치가 비슷해도 topology identity가 유지되지 않을 수 있다. Iso-value revision, local field 변화량, normal consistency, topology-change mask가 필요하다.

Sparse voxel과 octree streaming에서는 새 brick의 world-space AABB를 screen-space로 투영해 local invalidation mask를 만들면 전체 reset 없이 stale history만 제거할 수 있다.

Vulkan, DirectX, Metal, WebGPU에서는 알고리즘보다 current/previous resource lifetime, ping-pong indexing, texture state transition, reversed-Z와 jitter matrix convention이 자주 문제를 만든다.

## 6. 머릿속에 남길 질문 3개

1. Motion vector가 정확해도 previous history가 현재 surface와 대응하지 않을 수 있는 이유는 무엇인가?
2. Depth validation에서 고정 epsilon 대신 projection과 depth precision을 반영해야 하는 이유는 무엇인가?
3. CFD volume, marching-cubes topology, streamed voxel brick에는 어떤 revision signal이 필요한가?

## 7. Graphics Engineer 면접 질문 1개와 답변

**질문: TAA에서 disocclusion을 어떻게 검출하고 잘못된 history 누적을 어떻게 방지하겠습니까?**

**답변:** Motion vector로 previous UV를 구한 뒤 current surface를 previous camera로 투영해 expected previous depth를 계산하고, 해당 위치의 previous depth와 비교합니다. Projection과 depth precision을 고려한 threshold보다 차이가 크면 history를 reject합니다. Silhouette에서는 point sample, conservative neighborhood, reconstructed previous depth를 검토하고 필요하면 normal과 object ID를 추가합니다. Invalid pixel에서는 color뿐 아니라 history length, lock, variance, sample count를 함께 reset하며, 경계는 confidence 기반으로 감쇠합니다. 마지막으로 depth delta와 rejection mask를 debug view로 제공해 false acceptance와 false rejection을 구분합니다.

## 8. 포트폴리오 / 커리어 연결

포트폴리오에서는 단순히 “TAA를 구현했다”보다 jittered sampling, motion-vector reprojection, velocity dilation, disocclusion detection, history clipping, adaptive weight, temporal reset을 하나의 **history validity architecture**로 연결했다고 설명하는 것이 강하다.

Depth validation 전후, binary와 confidence rejection, depth-only와 depth+normal+ID, static mesh와 dynamic topology를 비교하면 GPU memory와 engine event까지 고려한 설계 역량을 보여줄 수 있다.

## 9. 내일 이어서 볼 개념

**Depth-Aware Reprojection Confidence and Surface Identity Validation**

Depth difference를 연속적인 confidence로 변환하고 normal, object ID, material class, motion divergence를 결합해 surface identity confidence를 구성하는 방법으로 이어간다.

## 10. 참고 키워드

Disocclusion Detection, History Rejection, Temporal Reprojection, Previous Depth Reconstruction, Depth Clip, Akeley Separation, Motion Vector Dilation, Surface Identity, Conservative Depth Test, Reversed-Z, Temporal Confidence, FSR2 Depth Clip, SVGF Temporal Accumulation

참고 자료: AMD GPUOpen FidelityFX Super Resolution 2.3.3 문서, NVIDIA Research SVGF 논문 페이지
