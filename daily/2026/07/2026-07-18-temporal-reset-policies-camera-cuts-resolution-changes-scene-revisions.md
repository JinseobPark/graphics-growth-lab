---
title: "Temporal Reset Policies: Camera Cuts, Resolution Changes, and Scene Revisions"
date: "2026-07-18"
category: "Graphics"
tags: ["GPU", "Temporal Rendering", "Temporal Anti-Aliasing", "Temporal Upscaling", "History Invalidation", "Camera Cut", "Dynamic Resolution", "Scene Revision", "Compute Shader", "Memory Layout", "Scientific Visualization"]
level: "intermediate"
---

# [Daily Graphics Growth] 2026-07-18 - Temporal Reset Policies: Camera Cuts, Resolution Changes, and Scene Revisions

## 1. 오늘의 개념

Temporal Anti-Aliasing(TAA), temporal upscaling, ray-tracing denoising, temporal volume filtering처럼 이전 프레임의 결과를 재사용하는 시스템은 **프레임 사이에 장면의 의미가 연속적이다**라는 가정 위에서 동작한다.

하지만 다음과 같은 사건이 발생하면 이전 history의 좌표와 의미가 현재 프레임에 더 이상 대응하지 않는다.

- 카메라가 연속적으로 이동하지 않고 다른 위치로 순간 전환되는 **Camera Cut / Jump Cut**
- render resolution, presentation resolution, viewport가 달라지는 **Resolution Change**
- object가 teleport하거나 mesh topology, material, simulation field가 교체되는 **Scene Revision**
- jitter sequence, projection mode, exposure convention, depth convention이 바뀌는 **Pipeline Revision**

이때 필요한 것이 **Temporal Reset Policy**, 즉 어떤 변화에서 어느 history resource를 완전히 초기화하고, 어느 변화에서는 부분적으로 감쇠하거나 재투영할지를 정하는 정책이다.

전날의 주제였던 **History Length, Lock Status, and Temporal Stability State**가 pixel별 과거의 신뢰도를 저장하는 방법이었다면, 오늘의 주제는 그 상태가 더 이상 의미가 없어진 순간을 renderer가 어떻게 감지하고 정리하는가에 해당한다.

```text
Temporal reuse가 성립하려면

Previous state
  + 현재 frame으로의 유효한 좌표 대응
  + 동일한 signal 의미
  + 호환되는 resolution / projection / exposure
  + 추적 가능한 scene identity

가 모두 유지되어야 한다.
```

Temporal reset은 단순히 texture를 검은색으로 지우는 작업이 아니다. **history validity contract가 깨졌음을 시스템 전체에 전파하는 상태 전이**다.

---

## 2. 한 줄 핵심

**Temporal reset policy는 이전 프레임의 데이터가 현재 프레임과 더 이상 같은 좌표·같은 장면·같은 신호를 나타내지 않을 때, history를 완전 초기화하거나 선택적으로 무효화하여 ghosting보다 재수렴 비용을 선택하는 안전장치다.**

---

## 3. 왜 중요한가

### 3.1 잘못된 history는 noise보다 위험하다

Temporal accumulation은 정상 상태에서 강력하다.

- 여러 frame의 sample을 결합해 aliasing과 noise를 줄인다.
- 저해상도 입력으로부터 sub-pixel detail을 복원한다.
- expensive lighting 또는 volume integration 결과를 시간축에서 재사용한다.

그러나 history가 현재 장면과 무관한 값이 되면 동일한 누적 구조가 오히려 큰 artifact를 만든다.

```text
카메라 cut 이전
- 화면 중앙: 밝은 emissive object

카메라 cut 이후
- 화면 중앙: 어두운 벽

old history를 그대로 재사용
→ 밝은 object의 잔상이 벽 위에 누적
→ neighborhood clamp가 완전히 제거하지 못함
→ 수 frame 동안 강한 ghosting
```

현재 frame만 사용하면 한두 frame 동안 noisy하거나 jagged해질 수 있다. 반면 잘못된 history를 사용하면 **존재하지 않는 구조가 화면에 남는다**. Temporal system에서는 보통 짧은 재수렴을 감수하고 의미가 깨진 history를 버리는 편이 안전하다.

### 3.2 Camera Cut은 motion vector로 복구할 수 없다

일반적인 camera motion에서는 이전 위치의 pixel을 현재 frame으로 reproject할 수 있다.

```text
prevUV = currentUV + motionVector
```

Camera cut은 연속적인 이동이 아니라 viewpoint의 불연속이다. 두 카메라 사이의 matrix 차이가 계산 가능하더라도, 다음 이유로 history reuse가 실질적으로 성립하지 않는다.

- 이전 frame에 없던 surface가 화면 대부분에 나타난다.
- motion vector가 viewport 전체를 가로지르거나 범위를 벗어난다.
- 이전 depth와 현재 depth의 neighborhood가 거의 대응하지 않는다.
- visibility와 occlusion 관계가 대규모로 재구성된다.
- exposure, focus, cinematic post-process가 동시에 바뀔 수 있다.

따라서 AMD FidelityFX Super Resolution 계열도 camera jump cut이 발생한 첫 frame에 reset signal을 전달하도록 명시한다. 이 경우 내부 temporal resources를 정리하며, 평상시보다 추가 clear 비용이 발생할 수 있다.

### 3.3 Resolution Change는 크기 문제가 아니라 좌표계 문제다

Dynamic Resolution Scaling(DRS)에서는 render resolution이 frame마다 달라질 수 있다.

예를 들어 presentation resolution은 `2560×1440`으로 유지하면서 내부 render resolution이 다음처럼 변경될 수 있다.

```text
Frame N     : 1920 × 1080
Frame N + 1 : 1706 × 960
```

Temporal upscaler의 output history가 presentation resolution에 저장된다면 output texture 크기는 유지될 수 있다. 그렇다고 모든 history가 자동으로 유효한 것은 아니다.

다음 요소가 함께 바뀔 수 있다.

- input pixel footprint
- jitter의 normalized amplitude
- render-resolution motion vector scale
- depth buffer texel layout
- reactive mask와 transparency mask 해상도
- lock 또는 stability metadata의 indexing
- mip selection과 texture detail frequency

즉, resolution change는 texture dimension의 변화뿐 아니라 **현재 sample이 화면에서 차지하는 footprint의 변화**다.

Resolution 변화 폭이 작고 API가 dynamic resolution을 명시적으로 지원한다면 history를 유지하며 coordinate scaling을 적용할 수 있다. 반면 viewport topology나 output resolution 자체가 바뀌면 resource 재할당과 full reset이 더 안전하다.

### 3.4 Scene Revision은 renderer 밖에서 발생한다

Camera cut은 renderer가 쉽게 감지할 수 있지만 scene revision은 application, simulation, asset streaming 계층에서 발생한다.

대표적인 예는 다음과 같다.

- object transform이 animation이 아니라 teleport로 변경
- entity ID가 재사용됨
- mesh LOD가 silhouette 수준으로 교체됨
- material shader 또는 opacity mode가 변경됨
- terrain chunk, voxel brick, octree node가 새 데이터로 대체됨
- marching cubes topology가 크게 갱신됨
- CFD timestep을 건너뛰거나 다른 dataset을 로드함
- semiconductor process step이 변경되어 전체 구조가 재생성됨

이 사건들은 GPU shader가 color, depth, motion vector만 보고 완벽하게 구분하기 어렵다.

예를 들어 같은 object ID와 비슷한 depth를 가진 surface라도, underlying scalar field가 교체되면 이전 volume history는 현재 data와 의미적으로 무관하다. 따라서 temporal system은 renderer 내부 heuristic뿐 아니라 application이 전달하는 **scene revision signal**을 받아야 한다.

### 3.5 Reset이 너무 적어도, 너무 많아도 문제다

#### Reset이 부족한 경우

- ghosting
- stale lighting
- 잘못된 thin-feature lock
- 이전 topology의 silhouette 잔상
- history confidence가 실제보다 높게 유지
- object ID 재사용으로 다른 물체의 history가 연결

#### Reset이 과도한 경우

- TAA shimmer 증가
- denoiser noise 재등장
- temporal upscaler의 detail reconstruction 손실
- volume rendering flicker
- camera가 조금 움직일 때마다 accumulation이 재시작
- history buffer clear와 resource transition 비용 증가

따라서 reset policy의 목표는 “변화가 있으면 항상 reset”이 아니다.

> **Temporal coherence가 유지되는 변화와 signal identity가 깨지는 변화를 구분하는 것**이 핵심이다.

---

## 4. 구현 관점

### 4.1 Reset을 단일 boolean이 아니라 reason mask로 관리한다

간단한 renderer는 `resetHistory = true` 하나로 충분할 수 있다. 규모가 커지면 reset 이유에 따라 대상 resource와 범위가 달라진다.

개념적으로 다음과 같은 bit mask를 둘 수 있다.

```cpp
enum class TemporalResetReason : uint32_t
{
    None               = 0,
    CameraCut          = 1u << 0,
    ProjectionChanged  = 1u << 1,
    RenderSizeChanged  = 1u << 2,
    OutputSizeChanged  = 1u << 3,
    JitterChanged      = 1u << 4,
    ExposureDiscontinuity = 1u << 5,
    SceneRevision      = 1u << 6,
    SimulationJump     = 1u << 7,
    ResourceRecreated  = 1u << 8
};
```

이 구조의 장점은 다음과 같다.

- debug HUD에서 reset 원인을 추적할 수 있다.
- pass별로 필요한 이유만 선택할 수 있다.
- full reset과 partial invalidation을 구분할 수 있다.
- CPU application event와 GPU temporal pass를 연결하기 쉽다.

예를 들어 TAA color history는 exposure discontinuity에 민감하지만, object-ID history는 동일한 scene이라면 유지할 수 있다. 반대로 mesh topology revision은 color, depth, normal, lock metadata를 함께 무효화해야 한다.

### 4.2 Frame-level reset과 pixel-level invalidation을 분리한다

Temporal invalidation은 두 레벨로 생각하는 것이 좋다.

#### Frame-level global reset

전체 화면의 history가 의미를 잃는 사건이다.

- camera cut
- output resolution 변경
- projection type 변경
- 다른 scene 또는 dataset 로드
- temporal algorithm 설정의 비호환 변경

```text
Global reset
→ previous history sampled 안 함
→ history length = 0
→ lock status = unlocked
→ stability = invalid
→ current frame으로 history 재시작
```

#### Pixel-level local invalidation

일부 영역만 history가 깨지는 사건이다.

- disocclusion
- teleport한 object의 screen region
- 새로 stream된 voxel brick 영역
- topology가 변경된 marching-cubes surface 주변
- transparent particle burst
- local lighting discontinuity

```text
Local invalidation mask
0: history reuse 가능
1: history 폐기 또는 강한 감쇠
```

Global reset만 제공하면 작은 변화에도 전체 화면이 재수렴한다. Pixel-level invalidation만 믿으면 camera cut 같은 대규모 불연속을 shader heuristic이 놓칠 수 있다. 두 방식을 함께 운영해야 한다.

### 4.3 Reset policy를 Hard, Soft, Remap으로 나눈다

#### Hard Reset

이전 history를 전혀 사용하지 않는다.

```text
historyWeight = 0
historyLength = 0
lockLifetime  = 0
stability     = 0
```

적합한 사건:

- camera jump cut
- 완전히 다른 scene 로드
- output resolution 또는 projection mode의 비호환 변경
- history resource 재생성
- simulation dataset 교체

#### Soft Reset / Decay

history를 즉시 제거하지 않고 신뢰도와 길이를 감소시킨다.

```text
historyLength *= 0.25
stability     *= 0.2
historyWeight  = min(historyWeight, 0.3)
```

적합한 사건:

- 작은 exposure 변화
- moderate dynamic-resolution change
- material parameter의 점진적 변경
- local topology update
- shading rate 변경

Soft reset은 ghosting 위험보다 flicker 억제가 더 중요한 경우에 유용하다.

#### History Remap

history coordinate 또는 signal scale을 변환한 뒤 재사용한다.

```text
prevUV = RemapViewport(currentUV, oldViewport, newViewport)
historyRadiance *= prevPreExposure / currentPreExposure
```

적합한 사건:

- 호환되는 render-size 변경
- viewport rectangle 이동
- pre-exposure convention 내의 연속적 변화
- atlas region 재배치가 명확히 추적되는 경우

Remap은 가장 효율적이지만, 변환이 정확하다는 계약이 필요하다. 불완전한 remap은 조용히 잘못된 history를 만든다.

### 4.4 Camera cut 검출은 explicit signal이 우선이다

Camera matrix 차이로 자동 검출할 수 있다.

```text
translationDelta = length(currCameraPos - prevCameraPos)
rotationDelta    = angle(currForward, prevForward)
```

하지만 threshold 방식은 장면 scale과 카메라 종류에 의존한다.

- 우주 규모 장면의 큰 translation은 정상 이동일 수 있다.
- 작은 방에서 2 m 이동은 사실상 cut에 가까울 수 있다.
- FOV 변경이나 orthographic 전환은 position delta가 없어도 불연속이다.
- cinematic camera system은 cut 여부를 이미 알고 있다.

따라서 우선순위는 다음이 적절하다.

```text
1. Application / camera system의 explicit camera-cut flag
2. projection type, FOV, near/far plane의 비호환 변경 검사
3. matrix delta 기반 safety heuristic
4. shader-side viewport / depth validation
```

명시적 signal은 의미를 알고 있는 상위 계층에서 전달하고, heuristic은 누락에 대비한 방어선으로 사용한다.

### 4.5 Resolution 변화에서 확인할 좌표와 단위

Temporal pass integration에서 자주 발생하는 문제는 동일한 motion vector가 서로 다른 단위로 해석되는 것이다.

- render pixel 단위
- output pixel 단위
- normalized device coordinate(NDC)
- normalized UV
- jitter 포함/제외 motion

Resolution이 바뀌면 다음 계약을 명확히 해야 한다.

```text
motionVectorScale
jitterCurrent
jitterPrevious
renderSizeCurrent
renderSizePrevious
outputSize
viewportOffset
```

예를 들어 motion vector가 render pixel 단위라면 `1920×1080`에서 기록된 `(2, 1)`과 `1280×720`에서 기록된 `(2, 1)`은 화면상 이동량이 다르다. Normalized UV로 변환하는 단계에서 frame별 render size가 정확히 반영되어야 한다.

### 4.6 Scene Revision ID를 GPU-visible state로 전달한다

Application에서 scene의 의미가 크게 바뀔 때 monotonic revision ID를 증가시킬 수 있다.

```cpp
struct TemporalFrameConstants
{
    uint32_t frameIndex;
    uint32_t sceneRevision;
    uint32_t simulationRevision;
    uint32_t resetReasonMask;
};
```

이전 frame metadata에도 revision을 저장한다.

```text
if (current.sceneRevision != history.sceneRevision)
    globalSceneHistoryValid = false;
```

Local object 단위로는 generation counter를 object ID와 결합할 수 있다.

```text
Temporal identity = object index + generation
```

Entity slot이 재사용되더라도 generation이 달라지면 다른 object로 판단할 수 있다. 이는 C++ ECS(Entity Component System)의 stale handle 방지 방식과 유사하다.

### 4.7 GPU resource와 memory layout

Temporal pipeline은 보통 여러 history resource를 함께 가진다.

| Resource | 예시 format | Reset 시 처리 |
|---|---:|---|
| History Color | `RGBA16F` | clear 또는 current color로 seed |
| History Moments | `RG16F` / `RG32F` | 0 또는 current moments |
| History Depth | `R32F` | invalid depth sentinel |
| History Normal | packed `RGBA8` / `RGB10A2` | invalid marker |
| History Length | `R8_UINT` / `R16F` | 0 |
| Lock Status | `R8_UNORM` | 0 |
| Stability / Confidence | `R8_UNORM` / `R16F` | 0 |
| Object Identity | `R32_UINT` | invalid ID |
| Revision Metadata | constant buffer / structured buffer | current revision 기록 |

모든 texture를 실제 clear할 필요는 없다. **Epoch 또는 generation 방식**을 사용할 수 있다.

```text
historyEpoch texture/metadata == currentEpoch
    → history valid
다름
    → history invalid
```

Global reset마다 큰 texture를 clear하는 대신 frame constant의 epoch를 증가시키고, history sample 시 epoch mismatch를 invalid로 처리할 수 있다. 다만 epoch를 pixel별로 저장하면 추가 memory bandwidth가 생긴다.

또 다른 방식은 reset frame에서 compute shader가 current frame을 history target에 직접 seed하는 것이다.

```text
Reset frame
Current color ──────────→ History color
Current depth ──────────→ History depth
0 ─────────────────────→ History length / lock
```

이 방식은 다음 frame부터 history resource가 명확한 초기 상태를 가진다는 장점이 있다.

### 4.8 Render graph에서 reset은 resource dependency다

Render graph 관점에서 reset pass는 단순 상태 변수 이상의 의미를 가진다.

```text
Camera / Scene Update
        ↓
Temporal Reset Classification
        ↓
History Clear or Seed Pass
        ↓
Temporal Resolve / Denoise / Upscale
        ↓
Write New History
```

Reset이 필요한데 clear pass가 누락되거나, clear 이전 history를 temporal resolve가 읽으면 read-after-write ordering 문제가 아니라 **semantic dependency violation**이 된다.

Vulkan, DirectX 12, WebGPU 같은 explicit API에서는 다음을 함께 관리해야 한다.

- history resource의 read/write transition
- ping-pong index 교체 시점
- reset frame의 source와 destination alias 여부
- async compute queue 사용 시 synchronization
- resize 후 old resource lifetime
- command buffer recording 시 reset flag의 frame ownership

특히 resize와 reset을 서로 다른 thread에서 처리하면 CPU에서는 새 dimension을 사용하지만 GPU command는 old history를 참조하는 frame-lag 문제가 생길 수 있다.

### 4.9 Debug visualization이 정책 품질을 결정한다

Temporal reset은 artifact가 발생한 뒤 원인을 추적하기 어렵다. 다음 debug view가 유용하다.

- reset reason mask
- global reset frame marker
- pixel-level invalidation mask
- history length
- history confidence
- object generation mismatch
- scene / simulation revision ID
- previous UV out-of-bounds 영역
- resolution remap error

예를 들어 화면 전체가 한 frame 동안 빨간색이면 global reset, 특정 object만 노란색이면 local scene revision으로 표시할 수 있다. 이러한 visualization은 “왜 ghosting이 생겼는가”뿐 아니라 “왜 매번 history가 사라지는가”도 보여준다.

---

## 5. 내 관심 분야와 연결

### 5.1 Real-Time Rendering과 Game Engine Architecture

게임 엔진에서는 temporal reset의 주체가 한 subsystem이 아니다.

- camera system: cut, FOV, projection 변경
- animation system: teleport, pose discontinuity
- world partition: level streaming, object generation 교체
- material system: shader permutation, blend mode 변경
- renderer: resolution, jitter, exposure, temporal algorithm 설정 변경
- post-process: depth of field, motion blur, cinematic transitions

따라서 renderer가 모든 변화를 추론하기보다 각 subsystem이 temporal invalidation event를 발행하고, render frame 시작 시 하나의 reset reason mask로 집계하는 구조가 확장성이 높다.

### 5.2 CFD / Scientific Visualization

CFD visualization에서는 camera가 고정되어 있어도 data 의미가 불연속적으로 바뀔 수 있다.

- timestep을 `t=100`에서 `t=500`으로 건너뜀
- 다른 case 또는 boundary condition 결과로 교체
- scalar field를 pressure에서 temperature로 변경
- streamline seed가 전체 교체
- clip plane이나 threshold 조건이 크게 변경

이때 color buffer의 geometry가 비슷하다는 이유로 history를 유지하면 이전 scalar field의 색과 구조가 남을 수 있다. Scientific visualization에서는 **simulation revision과 visualization mapping revision**을 별도로 관리하는 것이 좋다.

```text
Simulation revision
- 원본 field / timestep / topology 변화

Visualization revision
- transfer function / threshold / clipping / seed 변화
```

두 revision의 영향 범위가 다르므로 volume history, surface history, UI overlay를 선택적으로 reset할 수 있다.

### 5.3 Marching Cubes, Sparse Voxel, Octree

Marching cubes surface는 scalar field 변화에 따라 vertex count와 topology가 매 frame 달라질 수 있다. 단순 vertex index 기반 motion vector는 temporal identity를 유지하지 못한다.

- scalar field가 천천히 변하면 local invalidation 또는 confidence decay
- iso-value가 크게 바뀌면 surface history reset
- chunk가 새로 load되면 해당 brick의 screen projection 영역 invalidation
- octree LOD가 변경되면 silhouette 주변 history 완화

Sparse voxel이나 NanoVDB 기반 volume rendering에서는 brick residency 변화도 scene revision의 일부다. 이전 frame에는 비어 있던 공간에 새 brick이 들어오면 depth가 없어도 radiance history가 남아 있을 수 있다. Residency mask 또는 brick generation을 temporal validity에 연결할 수 있다.

### 5.4 Semiconductor 3D Visualization

공정 step이 Lithography → Etch → Deposition → CMP로 전환되면 geometry가 점진적 animation이 아니라 구조적으로 교체될 수 있다.

- material ID 변화
- surface topology 변화
- layer thickness 변화
- void 또는 trench 생성
- mesh / voxel representation 교체

이 경우 공정 revision을 증가시키고, geometry-dependent history를 reset하는 편이 안전하다. 반면 camera와 UI overlay history는 유지할 수 있다. Reset policy를 resource별로 나누면 대규모 scene에서도 불필요한 전체 초기화를 줄일 수 있다.

### 5.5 WebGPU / Vulkan / DirectX / Metal

API가 달라도 핵심은 같다.

- **Vulkan / DirectX 12**: explicit resource state와 queue synchronization
- **WebGPU**: resize 시 texture recreation, bind group 재구성, frame-consistent uniform 관리
- **Metal**: drawable size 변경과 history texture lifetime, command buffer ordering
- **OpenGL**: implicit state 환경에서도 framebuffer resize와 ping-pong texture 교체 순서 관리

Temporal reset은 shader 알고리즘만의 문제가 아니라 resource lifetime, frame ownership, API synchronization을 연결하는 엔진 설계 문제다.

---

## 6. 머릿속에 남길 질문 3개

1. **현재 발생한 변화는 좌표만 바뀐 것인가, 아니면 history가 나타내는 signal의 의미 자체가 바뀐 것인가?**

2. **이 사건은 전체 화면의 temporal coherence를 깨는가, 특정 object·voxel brick·simulation region만 무효화하면 되는가?**

3. **Hard reset, soft decay, coordinate remap 중 어떤 정책이 ghosting과 재수렴 비용 사이에서 가장 안전한가?**

---

## 7. Graphics Engineer 면접 질문 1개와 답변

### 질문

**Temporal AA 또는 temporal upscaler에서 camera cut과 dynamic resolution change가 발생했을 때 history buffer를 어떻게 처리하겠습니까?**

### 답변

먼저 두 사건을 같은 reset으로 처리하지 않고 **temporal continuity가 유지되는지**를 구분하겠습니다.

Camera cut은 이전 frame과 현재 frame의 visibility 대응이 대규모로 깨지는 불연속이므로 application의 explicit camera-cut flag를 우선 사용해 full history reset을 수행합니다. Color, depth, moments, history length, lock status, stability metadata를 invalid 상태로 만들고 current frame으로 새 history를 seed합니다. Matrix delta heuristic은 flag 누락에 대비한 보조 수단으로 사용합니다.

Dynamic resolution change는 output resolution과 render resolution 중 무엇이 바뀌었는지 확인합니다. Temporal algorithm이 dynamic resolution을 지원하고 output history가 유지되며 motion vector, jitter, viewport scaling 계약이 정확하다면 history UV를 remap하고 confidence를 일시적으로 낮춰 재사용할 수 있습니다. 반면 output size, projection, viewport topology 또는 history resource layout이 바뀌면 resource를 재생성하고 full reset합니다.

엔진 구조에서는 reset 원인을 bit mask로 관리하고, global reset과 pixel-level invalidation을 분리합니다. 또한 scene revision과 resource generation을 GPU-visible metadata로 전달해 object teleport, streaming, simulation jump 같은 renderer 외부의 불연속도 temporal validity에 반영합니다.

---

## 8. 포트폴리오 / 커리어 연결

Temporal rendering 포트폴리오에서 단순히 “TAA를 구현했다”보다 다음 설계를 설명할 수 있으면 graphics engineer 역량이 더 잘 드러난다.

- history validity contract를 정의한 방식
- camera cut, resize, teleport, scene streaming 이벤트를 수집하는 구조
- full reset과 local invalidation의 구분
- render-size 변화에서 motion vector와 jitter 단위를 맞춘 방식
- history color뿐 아니라 moments, depth, lock, confidence를 함께 초기화한 이유
- `RGBA16F`, `R8_UINT`, `R32_UINT` 등 metadata format 선택 근거
- ping-pong resource와 render graph synchronization
- reset reason, invalidation mask, history length debug view
- ghosting과 excessive reset을 구분해 분석한 사례

면접에서는 다음 문장이 핵심이다.

> “Temporal artifact는 blending formula만의 문제가 아니라, 이전 frame 데이터의 identity와 lifetime을 엔진 전체에서 관리하는 문제라고 보았습니다.”

이 관점은 game renderer뿐 아니라 CFD post-processing, volume rendering, semiconductor process visualization처럼 장면 데이터가 frame 사이에 교체되는 시스템에도 직접 적용된다.

---

## 9. 내일 이어서 볼 개념

**Disocclusion Detection and History Rejection**

Global camera cut과 scene revision을 처리한 다음에는, 정상적인 camera/object motion 중에서도 이전 frame에 가려져 있던 surface가 새로 나타나는 **disocclusion**을 pixel 단위로 검출해야 한다.

내일은 다음 흐름으로 이어진다.

- previous depth와 current depth를 어떤 공간에서 비교하는가
- motion-vector dilation이 disocclusion 판정에 미치는 영향
- depth threshold를 거리와 slope에 따라 조절하는 이유
- normal, object ID, material ID를 함께 사용하는 history rejection
- thin geometry에서 false rejection을 줄이는 방법
- scientific visualization에서 topology 변화와 disocclusion을 구분하는 관점

---

## 10. 참고 키워드

- Temporal Reset Policy
- History Invalidation
- Camera Cut / Camera Jump Cut
- Dynamic Resolution Scaling (DRS)
- History Remapping
- Hard Reset / Soft Reset
- Temporal Epoch / Generation Counter
- Scene Revision ID
- Simulation Revision
- Resource Generation
- Global Reset
- Pixel-Level Invalidation
- History Seed Pass
- Temporal Resource Ping-Pong
- Motion Vector Scale
- Jitter Sequence Reset
- Exposure Discontinuity
- Viewport Remapping
- Object Identity / Object Generation
- AMD FidelityFX Super Resolution – Camera Jump Cuts
- NVIDIA NRD – History Confidence
- Temporal Anti-Aliasing
- Temporal Upscaling
- Spatiotemporal Denoising
- Disocclusion Detection
