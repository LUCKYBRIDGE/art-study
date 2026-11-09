# 놀아트 Ink Engine v1 — 기술 사양

목표: 원본 수묵화 웹의 UI/코드를 복제하지 않고 **먹붓의 핵심 물성**만 독립 구현한다.

---

# 1. v1의 품질 목표

v1에서 반드시 보여야 하는 현상:

1. 눌렀을 때 단순 굵기 변화가 아니라 붓털 접촉이 늘어남
2. 방향 전환 때 붓끝이 늦게 따라옴
3. 먹이 줄면서 일부 붓털부터 끊김
4. 갈필 공백이 진행 방향을 따름
5. 물 많은 획은 그은 뒤에도 퍼짐
6. 종이결 때문에 번짐 경계가 원형이 아님
7. 젖은 영역의 가장자리가 중심보다 일부 진해짐
8. 같은 먹색도 pigment mass에 따라 깊이가 비선형적으로 변함

---

# 2. 권장 아키텍처

```
src/features/ink/
├─ engine/
│  ├─ InkEngine.ts
│  ├─ fixedStep.ts
│  ├─ input/
│  │  ├─ PointerSampler.ts
│  │  └─ StrokeResampler.ts
│  ├─ brush/
│  │  ├─ BrushBundle.ts
│  │  ├─ Bristle.ts
│  │  ├─ BristleSolver.ts
│  │  ├─ ContactSolver.ts
│  │  └─ Reservoir.ts
│  ├─ paper/
│  │  ├─ PaperState.ts
│  │  ├─ FiberField.ts
│  │  └─ PaperPresets.ts
│  ├─ fluid/
│  │  ├─ WaterPass.ts
│  │  ├─ PigmentPass.ts
│  │  ├─ DryingPass.ts
│  │  └─ EdgeDepositPass.ts
│  ├─ replay/
│  │  ├─ Recorder.ts
│  │  └─ Player.ts
│  └─ types.ts
├─ gpu/
│  ├─ webgpu/
│  └─ webgl/
├─ render/
├─ presets/
└─ ui/
```

---

# 3. fixed timestep

렌더링과 brush physics를 분리한다.

권장:

```ts
const BRUSH_DT = 1 / 240
const MAX_SUBSTEPS = 8
```

```ts
accumulator += frameDt

let steps = 0
while (accumulator >= BRUSH_DT && steps < MAX_SUBSTEPS) {
  brush.step(BRUSH_DT)
  accumulator -= BRUSH_DT
  steps++
}
```

frame drop 시 무한 catch-up을 막기 위해 max substep을 둔다.

paper simulation은 처음부터 240Hz일 필요가 없다.

권장:
- brush: 120~240Hz
- paper: 30~60Hz
- render: display refresh

각 subsystem의 update frequency를 분리한다.

---

# 4. Brush data

v1 권장:

```ts
const BRISTLE_COUNT = 28
const PARTICLES_PER_BRISTLE = 4

type Particle = {
  x: number
  y: number
  px: number
  py: number
}

type Bristle = {
  particles: Particle[]
  restLength: number

  ink: number
  water: number

  capacity: number
  refillRate: number
  transferRate: number

  contact: number
}
```

28×4 = 112 dynamic particles로 시작한다.

모바일에서도 충분히 가볍고 독립 bristle 효과를 볼 수 있다.

---

# 5. bristle 초기 분포

완벽한 원형 균일 배치 금지.

```ts
angle_i =
  goldenAngle * i
  + seededNoise(i) * angularJitter

radius_i =
  sqrt(i / N)
  * bundleRadius
  * radialJitter(i)
```

각 털의:
- capacity
- stiffness
- refillRate
- transferRate

에도 ±5~15% 정도 seeded variation을 둔다.

이 편차가 갈필과 미세 질감을 만든다.

---

# 6. motion target

포인터 raw event를 그대로 physics target으로 쓰지 않는다.

```
Pointer events
→ coalesced events
→ spatial resampling
→ smoothed handle path
→ brush root target
```

권장 sample spacing:

```
0.3~1.0 simulation cell
```

속도에 따라 adaptive resampling 가능.

---

# 7. PBD-lite solver

## Integrate

```ts
vx = p.x - p.px
vy = p.y - p.py

p.px = p.x
p.py = p.y

p.x += vx * damping
p.y += vy * damping
```

root particle는 handle target에 고정하거나 높은 stiffness로 추종시킨다.

## Distance constraint

```ts
const dx = b.x - a.x
const dy = b.y - a.y
const len = hypot(dx, dy)

const error = (len - restLen) / len

a += correction * 0.5
b -= correction * 0.5
```

## Shape memory

각 particle이 rest curve에서 너무 벗어나지 않도록 약한 복원력을 추가한다.

## Constraint iterations

v1:
- desktop 4~6
- low power 2~3

---

# 8. pressure model

pressure는 최소 세 곳에 영향을 줘야 한다.

```ts
spread = lerp(minSpread, maxSpread, pressureCurve(p))
contactDepth = pressure * maxDepth
transferBoost = lerp(0.6, 1.3, p)
```

pressure curve:

```ts
p2 = smoothstep(0, 1, p)
```

단순 선형보다 자연스럽다.

---

# 9. touch fallback pressure

실제 pressure가 없는 손가락 입력:

```ts
pseudoPressure =
  clamp(
    base
    + slowSpeedBoost
    + holdDurationBoost,
    0.15,
    0.85
  )
```

속도가 느리고 오래 머물수록 조금 더 눌린 것으로 간주한다.

마우스/손가락에서도 획 변화가 생긴다.

---

# 10. wet clustering

털 전체를 한 점으로 끌지 않는다.

가까운 털을 3~6개 작은 cluster로 묶는다.

```ts
clusterStrength =
  brushWetness
  * preset.clusterStrength
```

cluster center 방향으로 tip particle만 약하게 이동시킨다.

마른 붓에서는 clusterStrength를 줄여 털이 갈라지게 한다.

---

# 11. reservoir

```ts
type BrushReservoir = {
  ink: number
  water: number
}
```

각 털 재공급:

```ts
const need = bristle.capacity - bristle.ink
const refill =
  min(
    need,
    reservoir.ink
      * bristle.refillRate
      * connectivity
      * dt
  )
```

```connectivity`는 water가 적어지면 낮춘다.

따라서 마른 붓에서 일부 털의 먹 보충이 늦어진다.

---

# 12. contact/deposit

각 bristle tip마다 종이 접촉점을 계산한다.

```ts
contact =
  pressureContact
  * paperContact
  * bristleContact
```

```ts
inkDeposit =
  min(
    bristle.ink,
    contact
      * bristle.transferRate
      * dt
  )
```

```ts
waterDeposit =
  min(
    bristle.water,
    contact
      * waterTransferRate
      * dt
  )
```

deposition 후 bristle state에서 실제로 차감한다.

이 차감이 다음 획에 영향을 미쳐야 한다.

---

# 13. dry brush / 갈필

갈필을 별도 mode로 만들지 않는 것이 기본 원칙이다.

조건:

```
low brush water
+ low reservoir
+ bristle refill variance
+ paper micro height
+ higher speed
```

→ 자연스럽게 발생하도록 한다.

추가 안정화용 보조:

```ts
contactThreshold_i =
  paperHeight(x,y)
  + seededBristleOffset[i]
```

마른 상태일수록 threshold 영향도를 높인다.

---

# 14. paper state

v1 권장 grid:

- desktop: 1024×576
- low: 768×432
- high: 1536×864

실제 화면 비율이 16:9가 아니라면 canvas aspect에 맞게 결정한다.

상태:

```ts
type DynamicPaper = {
  water: Texture
  mobilePigment: Texture
  fixedPigment: Texture
}

type StaticPaper = {
  fiber: Texture
  height: Texture
  absorbency: Texture
}
```

---

# 15. fiber field 생성

단순 white noise 금지.

낮은 주파수 방향장 + 작은 왜곡:

```ts
theta =
  lowFreqNoise(x * 0.01, y * 0.01) * PI

fiber =
  vec2(cos(theta), sin(theta))
```

한지 preset은 방향성에 너무 강한 규칙성을 주지 않는다.

권장:
- large-scale orientation
- local perturbation
- occasional cross fiber

---

# 16. water flow

v1에서는 완전한 Navier-Stokes가 필요 없다.

thresholded anisotropic diffusion이면 충분하다.

각 neighbor에 대해:

```ts
delta = waterHere - waterThere - barrier

if (delta > 0) {
  alignment = abs(dot(fiberDir, neighborDir))

  k = mix(
    crossFiberConductivity,
    alongFiberConductivity,
    alignment
  )

  flow = delta * k * dt
}
```

총 outflow가 현재 water를 넘지 않게 normalize한다.

---

# 17. pigment transport

```ts
concentration =
  mobilePigment / max(water, EPS)

pigmentFlow =
  waterFlow
  * concentration
  * mobility
```

water transport와 같은 방향으로 pigment가 이동한다.

pigment 자체에 약한 diffusion을 추가할 수 있지만 water와 분리된 강한 blur는 금지한다.

---

# 18. drying

```ts
evaporation =
  baseEvaporation
  * exposure
  * paperPreset.evaporation
```

```ts
waterNext =
  max(0, water - evaporation * dt)
```

edge는 neighbor가 dry할수록 exposure를 조금 더 높인다.

---

# 19. deposition

```ts
dryness = 1 - saturate(water / wetScale)

rate =
  baseDeposit
  * absorbency
  * dryness * dryness
```

```ts
amount =
  mobilePigment
  * (1 - exp(-rate * dt))
```

```
mobile -= amount
fixed += amount
```

---

# 20. edge concentration

v1 방식:

1. wet mask 생성
2. local wet gradient 계산
3. dry front 쪽으로 이동하는 pigment의 settling 증가

```ts
edge =
  length(gradient(water))

edgeDeposit =
  edge
  * mobilePigment
  * edgeStrength
  * dryingFactor
```

edgeDarkening을 단순 Sobel 후 검정 outline으로 넣지 않는다.

**pigment state를 실제로 바꿔야** 겹침과 재습윤에서 일관성이 유지된다.

---

# 21. optical density

먹 black:

```ts
density =
  fixedPigment
  + mobilePigment * wetVisibility

darkness =
  1 - exp(-absorptionK * density)
```

paper와:

```ts
finalColor =
  mix(paperColor, inkColor, darkness)
```

초기에는 이것만으로 충분하다.

---

# 22. paper visual

시뮬레이션 field를 그대로 화면 noise로 보이게 하지 않는다.

시각 texture:
- low contrast
- multi-scale fibers
- 약한 warm/cool variation
- normal-like micro shading는 매우 약하게

먹이 없는 상태에서도 texture가 지나치게 보이면 실제 한지보다 벽지처럼 보인다.

---

# 23. replay

모든 pointer event를 그대로 저장하지 않는다.

physics input sample로 정규화해 저장한다.

```ts
type InkInputFrame = {
  tick: number
  x: number
  y: number
  pressure: number
  tiltX?: number
  tiltY?: number

  action?:
    | "dipInk"
    | "dipWater"
    | "blot"
    | "setBrush"
    | "setPaper"

  value?: number
}
```

random seed + preset version도 프로젝트에 저장한다.

---

# 24. backend

```
InkEngine API
      ↓
┌───────────────┬─────────────┐
WebGPU backend  WebGL2 backend
High/Standard   Standard/Lite
```

동일한 state semantics를 유지하고 계산 정밀도만 낮춘다.

WebGPU 미지원 시 수묵화 기능 자체를 막지 않는다.

---

# 25. 품질 단계

## Lite
- 16 bristles
- 3 particles
- 512~768 grid width
- 30Hz paper sim
- simple fiber flow

## Standard
- 28~32 bristles
- 4 particles
- 768~1024
- 60Hz paper sim
- edge deposition

## High
- 48~64 bristles
- 5~6 particles
- 1536~2048
- WebGPU
- richer fiber/contact

---

# 26. 성능 측정

매 프레임 기록:

```ts
type InkPerf = {
  brushMs: number
  paperMs: number
  renderMs: number
  droppedSteps: number
}
```

자동 quality downgrade 조건을 둘 수 있다.

예:
- 2초 이상 24fps 미만 → simulation resolution 한 단계 감소

---

# 27. 반드시 분리 테스트

## Brush-only debug
흰 배경 위에 bristle skeleton과 contact point 표시.

## Ink-only debug
paper flow off, deposit만 보기.

## Water-only debug
pigment 없이 water map만 표시.

## Fiber debug
flow vector field 표시.

## Deposit debug
mobile / fixed pigment를 다른 색으로 임시 표시.

이 debug view가 없으면 물성 tuning이 거의 불가능하다.

---

# 28. v1 Definition of Done

다음 모두 만족:

- [ ] pressure 변화가 geometry에 반영
- [ ] 90도 turn에서 tip lag
- [ ] reservoir 소모
- [ ] bristle별 잔량 차이
- [ ] 갈필이 procedural result로 발생
- [ ] wet stroke가 pointer up 후 변화
- [ ] paper fiber가 diffusion shape에 영향
- [ ] edge pigment accumulation
- [ ] replay 가능
- [ ] low-end fallback 존재
- [ ] 원본 asset/code/UI 직접 복제 없음

