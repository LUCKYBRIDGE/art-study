# 놀아트 수묵화 모드 설계안

## 목표

놀아트에 수묵화의 '감성'을 추가하는 것을 **UI 테마 변경**으로 정의하지 않는다.

목표는 학생이 디지털 화면에서도 다음을 경험하는 것이다.

- 붓에 먹을 묻힌다.
- 물의 양을 조절한다.
- 필압과 속도에 따라 획이 달라진다.
- 먹이 줄수록 갈필이 생긴다.
- 젖은 종이에서는 번지고 마르며 형태가 변한다.
- 같은 획도 종이와 먹 상태에 따라 다르게 나타난다.

---

# 1. 놀아트 내 위치

기존 홈의 활동 유형과 병렬로 두는 것을 권장한다.

```
놀아트
├─ 테셀레이션
├─ 협동화
├─ 작품 모으기
├─ 전시
└─ 수묵화  ← 신규
```

테셀레이션 편집기에 브러시 하나를 추가하는 방식은 권장하지 않는다.

수묵화는 interaction model 자체가 다르기 때문이다.

---

# 2. 시작 화면

초등학생 기준으로 처음부터 슬라이더 6개를 보여주지 않는다.

## 기본 화면

```
[새 종이]

         한지 작업 영역

[먹 묻히기] [물 묻히기] [물기 빼기]

[가는 붓] [보통 붓] [큰 붓]

[되돌리기] [저장] [공유]
```

## 펼치기: 자세히
- 먹 농도
- 번짐
- 필압 감도
- 붓 탄력
- 종이 종류
- 붓털 보기

즉 **행위 중심 기본 UI + 수치 중심 고급 UI**.

---

# 3. 렌더링 구조

권장 파이프라인:

```
Pointer / Stylus
      ↓
Stroke Sampler
      ↓
Brush Dynamics
      ↓
Bristle Contact Map
      ↓
Ink + Water Deposit
      ↓
Paper Simulation
      ↓
Composite Renderer
      ↓
Canvas
```

---

# 4. 입력 계층

Pointer Events 사용.

```ts
type BrushInput = {
  x: number
  y: number
  pressure: number
  tiltX: number
  tiltY: number
  speed: number
  timestamp: number
}
```

Stylus가 없는 학교 태블릿에서는:

```ts
effectivePressure =
  hasPressure
    ? event.pressure
    : pressureFromSpeedAndHold(speed, duration)
```

손가락 입력에서도 획이 너무 기계적으로 되지 않도록 속도 기반 pseudo-pressure를 제공한다.

---

# 5. 붓 모델: 놀아트용 간소화

원본 수준의 72×8 입자 대신 기본 모드에서는 24~32 bristles가 적절하다.

```ts
type Bristle = {
  root: Vec2
  tip: Vec2
  velocity: Vec2
  ink: number
  water: number
  stiffness: number
}
```

### 각 프레임

1. 붓 중심을 입력 위치로 이동
2. 이전 tip과 새 목표 위치 사이 탄성 적용
3. 압력에 따라 bristle spread 증가
4. 젖을수록 bristle clustering 증가
5. 각 tip의 접촉 여부 계산
6. 접촉한 셀에 먹/물 deposit

---

# 6. 획 모양

### spread

```ts
spread =
  baseSpread
  * (0.35 + pressure * 0.9)
  * brushSize
```

### drag lag

```ts
tip += (target - tip) * stiffness * dt
```

stiffness가 낮을수록 붓끝이 뒤늦게 따라오며 획이 살아난다.

### bristle clustering

젖은 붓은 털끼리 붙는다.

간단히:

```ts
tip += (clusterCenter - tip)
     * wetness
     * clusterStrength
```

---

# 7. 먹 고갈과 갈필

각 bristle마다 `ink`를 둔다.

```ts
deposit = min(
  bristle.ink,
  contactStrength * pigmentRate
)
```

후:

```ts
bristle.ink -= deposit
```

붓 전체 저장량에서 천천히 재공급:

```ts
bristle.ink +=
  reservoirInk
  * refillRate
  * dt
```

reservoir가 낮아질수록 refillRate를 감소시키면 가닥별 편차가 커지고 갈필이 자연스럽게 발생한다.

---

# 8. 종이 텍스처

단순 이미지 overlay와 시뮬레이션용 fiber field를 분리한다.

## Visual texture
한지 사진/절차적 노이즈를 낮은 대비로 사용.

## Simulation fiber field

각 셀에 방향 벡터:

```ts
fiber = vec2(cos(theta), sin(theta))
```

Perlin/Simplex noise로 theta를 느리게 변화시키면 일정하지 않은 섬유결을 만들 수 있다.

---

# 9. 물 확산

놀아트 Standard 버전에서는 2D grid 기반 ping-pong texture 권장.

텍스처:
- R = water
- G = mobile pigment
- B = fixed pigment
- A = optional absorbency

### diffusion

```
waterNext =
  water
  + Σ(neighborFlow × fiberWeight)
  - evaporation
  - absorption
```

### fiberWeight

```glsl
float alignment =
  abs(dot(flowDirection, fiberDirection));

float diffusion =
  mix(crossFiberRate, alongFiberRate, alignment);
```

이것 하나만 넣어도 원형 blur보다 훨씬 종이 같은 번짐을 만들 수 있다.

---

# 10. 먹 이동

```
pigmentTransport =
  mobilePigment
  × movedWater
  × pigmentMobility
```

물이 이동할 때 일부 pigment도 이웃 셀로 이동한다.

---

# 11. 건조

```ts
water -= evaporationRate * dt
```

물이 일정 값 아래가 되면 mobile pigment 일부를 fixed pigment로 전환한다.

```ts
if (water < dryingThreshold) {
  fixed += mobile * settlingRate
  mobile *= 1 - settlingRate
}
```

---

# 12. 가장자리 농축

수묵의 맛을 크게 좌우한다.

복잡한 fluid solver 없이도 근사 가능하다.

```ts
edgeFactor =
  max(0, localWater - avgNeighborWater)

edgeDeposit =
  mobilePigment
  * edgeFactor
  * edgeDarkening
```

또는 wet-mask의 gradient 크기를 이용한다.

```glsl
edge = length(gradient(water));
fixedPigment += edge * mobilePigment * edgeStrength;
```

---

# 13. 합성

최종 색은 단순 alpha가 아니라 광학 농도 형태가 낫다.

```ts
darkness =
  1 - exp(-pigmentDensity * inkAbsorption)
```

```ts
paperColor = vec3(0.96, 0.945, 0.90)
inkColor   = vec3(0.04, 0.045, 0.04)

final =
  mix(paperColor, inkColor, darkness)
```

---

# 14. 재료 프리셋

## 먹 + 한지
- 높은 흡수
- 중간 확산
- 강한 edge accumulation
- 거친 fiber

## 먹 + 선지
- 높은 확산
- 부드러운 fiber
- 빠른 번짐

## 먹 + 백지
- 낮은 확산
- 높은 경계 선명도

## 주묵
pigment color만 붉게 바꾸지 말고 mobility와 opacity도 별도 profile로 둔다.

---

# 15. 초등 미술 UX

수치보다 감각적 언어를 우선한다.

| 내부 파라미터 | 학생 UI |
|---|---|
| inkLoad | 먹 듬뿍 / 보통 / 거의 없음 |
| water | 촉촉 / 보통 / 마른 붓 |
| stiffness | 부드러운 붓 / 탄탄한 붓 |
| diffusion | 덜 번지는 종이 / 잘 번지는 종이 |

고급 설정에서만 정확한 슬라이더 값을 보여준다.

---

# 16. 수업용 기능

## 붓털 보기
횡이 만들어지는 과정을 확대 표시.

## 번짐 보기
그린 직후 3~5초 동안 물과 먹의 변화 관찰.

## 작품 과정 재생
입력 이벤트 replay.

## 비교
같은 획을:
- 마른 붓
- 물 많은 붓
- 먹 많은 붓

세 조건으로 자동 비교.

이 기능은 단순 그림 앱보다 미술 학습 도구로서 차별성이 크다.

---

# 17. 테셀레이션과 결합 가능성

수묵화 모드를 테셀레이션의 도형 제작 과정과 직접 섞는 것은 초기 버전에서는 피하는 것이 좋다.

다만 이후:

```
테셀레이션 타일 생성
→ 타일 내부 수묵 채색
→ 반복 배치
```

는 가능하다.

이 경우 수묵 texture 결과를 타일의 local texture로 baking한 후 반복한다.

실시간 diffusion을 반복된 모든 타일에서 동시에 계산하면 지나치게 무겁다.

---

# 18. 저장 형식

PNG/JPG 외에 프로젝트 파일을 권장한다.

```ts
type InkProject = {
  version: 1
  paperPreset: string
  brushPreset: string
  strokes: StrokeEvent[]
  stamp?: SealData
  thumbnail: string
}
```

완성 bitmap 대신 stroke events 중심으로 저장하면:
- 파일 용량 감소
- 과정 재생
- 품질 변경 후 재렌더
- 편집 지속

이 가능하다.

---

# 19. 구현 우선순위

### Phase 1 — 감성 검증
- 한지 texture
- 24 bristle
- 먹 고갈
- 간단 diffusion
- edge darkening
- 3개 재료 버튼

### Phase 2 — 물성 강화
- anisotropic fiber
- mobile/fixed pigment
- wet brush clustering
- stylus pressure

### Phase 3 — 교육 기능
- 과정 재생
- 비교 실험
- 붓털 시각화
- 설명 오버레이

### Phase 4 — 고품질
- WebGPU backend
- 큰 종이
- 타임랩스
- 세밀한 material preset

---

# 20. 권장 아키텍처

```
src/features/ink/
├─ InkStudio.tsx
├─ engine/
│  ├─ BrushEngine.ts
│  ├─ BristleSystem.ts
│  ├─ PaperSimulation.ts
│  ├─ PigmentTransport.ts
│  ├─ ReplayEngine.ts
│  └─ types.ts
├─ render/
│  ├─ InkRenderer.ts
│  ├─ Canvas2DRenderer.ts
│  ├─ WebGLRenderer.ts
│  └─ WebGPURenderer.ts
├─ presets/
│  ├─ brushes.ts
│  ├─ papers.ts
│  └─ pigments.ts
├─ shaders/
│  ├─ diffusion.frag
│  ├─ pigment.frag
│  ├─ dry.frag
│  └─ composite.frag
├─ ui/
│  ├─ InkToolbar.tsx
│  ├─ BrushPalette.tsx
│  ├─ WaterTools.tsx
│  ├─ PaperPicker.tsx
│  └─ AdvancedInkSettings.tsx
└─ replay/
   ├─ StrokeRecorder.ts
   └─ TimelapseExporter.ts
```

기존 놀아트 코드에 억지로 섞지 말고 독립 feature module로 시작하는 편이 유지보수에 유리하다.

---

# 21. 가장 중요한 제품 원칙

놀아트의 수묵화 기능은

**'수묵화처럼 보이는 그림을 쉽게 만드는 기능'**

보다

**'먹·물·붓·종이의 관계를 움직이며 배우는 디지털 화실'**

이 되어야 한다.

그렇게 해야 기존 수묵화 웹의 장점을 가져오면서도 놀아트의 교육적 정체성이 생긴다.
