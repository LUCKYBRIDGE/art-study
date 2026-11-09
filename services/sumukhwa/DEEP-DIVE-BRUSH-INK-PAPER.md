# 수묵화 웹앱 심층 분석 — 실제 먹붓 느낌은 어떻게 만들어지는가

대상: https://dew978.github.io/sumukhwa/  
분석일: 2026-11-10

> 이 문서는 원본 화면과 원본이 화면에서 직접 공개한 기술 설명을 기준으로 작성한 clean-room 분석이다. 원본 소스 저장소는 공개적으로 확인되지 않았으므로 **확인 사실**과 **기술적 추론**을 구분한다.

## 0. 결론부터

이 앱의 먹붓 느낌은 하나의 브러시 shader나 먹 텍스처에서 나오지 않는다.

핵심은 다음 다섯 상태계가 동시에 작동하는 데 있다.

```
손의 이동/필압
    ↓
72가닥 붓털의 지연·변형·갈라짐
    ↓
각 털의 먹/물 잔량과 접촉 여부
    ↓
종이 격자의 수분·안료·섬유 방향
    ↓
확산·건조·경계 농축
    ↓
획이 그어진 뒤에도 계속 변하는 먹 자국
```

따라서 실제감을 만드는 핵심은 **붓의 모양**이 아니라 **상태가 시간에 따라 소모·전달·변형되는 과정**이다.

---

# 1. 증거 수준

문서 전체에서 다음 표기를 사용한다.

- **[확인]** 원본 페이지가 직접 표시하거나 설명한 내용
- **[강한 추론]** 공개 설명과 결과가 서로 잘 맞고 일반적인 그래픽스 모델과도 일치
- **[가능성]** 구현상 자연스럽지만 원본에서 직접 확인되지 않음
- **[놀아트 권장]** 원본 복제가 아니라 놀아트용 독립 구현 제안

---

# 2. 원본이 직접 밝힌 붓 구조

## 2.1 72가닥 × 8입자

**[확인]**

원본은 붓을 다음처럼 설명한다.

- 붓털 72가닥
- 털 한 가닥당 8개 입자의 사슬
- 1/240초마다 위치 기반 동역학(Position-Based Dynamics, PBD)
- 길이 제약
- 원래 모양으로 돌아가려는 탄성
- 젖은 털끼리 모이는 힘
- 종이와의 마찰 및 퍼짐

따라서 전체 붓은 최소:

```
72 bristles × 8 particles = 576 particles
```

의 동적 구조다.

이 숫자는 실시간 상호작용에서는 충분히 작으면서, 붓끝의 미세한 불균일성을 만들기에는 충분히 크다.

---

# 3. 왜 1/240초인가

**[강한 추론]**

화면이 60fps라면 1프레임은 약 1/60초다.

```
1 / 60 sec
= 4 × (1 / 240 sec)
```

즉 렌더 한 프레임 사이에 최대 4회의 붓 물리 substep을 계산할 수 있다.

이 구조의 효과:

1. 빠른 손 이동에서 털이 순간이동하지 않는다.
2. 길이 제약이 안정적으로 유지된다.
3. 종이 충돌 시 털이 깊게 관통하는 현상이 줄어든다.
4. 방향을 급히 바꿀 때 털끝의 늦은 추종이 자연스럽다.
5. 프레임률이 약간 흔들려도 붓 물리가 덜 폭주한다.

PBD 자체가 위치를 직접 제약 조건에 투영하는 방식이라 실시간 변형 물체에서 안정성과 제어성이 높은 편이다.

---

# 4. 한 가닥의 붓털은 어떻게 움직이는가

원본이 정확한 식을 공개하지는 않았지만 공개 설명을 만족하는 가장 자연스러운 구조는 다음과 같다.

```ts
type BristleParticle = {
  pos: Vec2 | Vec3
  prevPos: Vec2 | Vec3
}

type Bristle = {
  p: BristleParticle[8]
  restSegments: number[7]
  restShape: Vec2[]
  ink: number
  water: number
}
```

## 4.1 예측 단계

**[강한 추론]**

```
velocity = pos - prevPos
predicted = pos + velocity + externalMotion
```

Verlet/PBD 계열이면 별도 속도 벡터 없이 이전 위치 차이만으로 관성을 만들 수 있다.

## 4.2 길이 제약

**[확인: 길이 제약 존재 / 식은 추론]**

각 인접 입자 사이 거리:

```
|p[i+1] - p[i]| ≈ restLength[i]
```

가 되도록 위치를 반복 보정한다.

이 때문에 붓털이 고무줄처럼 무한히 늘어나지 않는다.

## 4.3 원형 복원 탄성

**[확인: 원래 모양 복원 힘 존재]**

털이 너무 휘면 초기 붓 모양 쪽으로 돌아가려 한다.

```
correction +=
  (restRelativePosition - currentRelativePosition)
  × stiffness
```

이 항이 없으면 털은 한 번 눌린 뒤 흐물흐물하게 남는다.

## 4.4 종이 접촉

**[확인: 종이 마찰과 퍼짐 존재]**

붓이 종이에 닿으면:

- 수직 방향 이동은 종이 평면에 의해 막히고
- 접선 방향 이동은 마찰 때문에 느려지고
- 필압이 높을수록 여러 털이 바깥으로 퍼진다.

즉 필압은 단순히 선 굵기를 바꾸는 값이 아니라 **실제로 접촉하는 털의 수와 위치**를 바꾼다.

이 차이가 매우 중요하다.

---

# 5. 실제 붓처럼 느껴지는 핵심: tip lag

디지털 원형 브러시는 포인터 중심과 획 중심이 거의 동일하다.

하지만 실제 붓은 손잡이가 방향을 바꿔도 털끝이 즉시 따라오지 않는다.

```
손잡이 ───────→
               \
                \
                 붓끝
```

급격한 방향 전환에서는:

```
handle direction changes
→ root particles move first
→ intermediate particles bend
→ tip follows later
```

이 **시간 지연**이 획의 생동감을 만든다.

결과:

- ㄱ자 획이 벡터 선처럼 날카롭게 꺾이지 않음
- 획 끝에 미세한 꼬리가 생김
- 역방향 전환 시 붓털이 접혔다 펴짐
- 속도 변화에 따라 선 가장자리도 달라짐

---

# 6. 젖은 붓털끼리 뭉치는 힘

**[확인]**

원본은 젖은 털끼리 모이는 힘을 명시한다.

실제 붓에서도 물의 표면장력 때문에 털들이 다수의 작은 묶음으로 뭉친다.

가상 모델에서는 이를 대략:

```
clusterForce_i =
  (localWetClusterCenter - tip_i)
  × water_i
  × clusteringStrength
```

처럼 만들 수 있다.

중요한 점은 모든 털이 하나의 중심으로 완전히 합쳐지는 것이 아니라 **가까운 털끼리 여러 묶음**을 형성해야 한다는 것이다.

그렇지 않으면 붓이 고무 도장처럼 된다.

---

# 7. 실제 먹붓 느낌의 핵심 1 — 털별 먹 잔량

**[확인]**

원본 설명상 먹은 털마다 따로 존재한다.

또한 붓의 '배'에 저장된 먹이 각 털로 공급된다.

```
brush belly reservoir
       ↓
  ┌────┼────┐
hair0 hair1 hair2 ... hair71
```

이 구조가 매우 중요하다.

단일 `inkAmount`만 쓰면 모든 붓털이 동시에 연해지기 때문에 갈필이 생기지 않는다.

각 털이 별도 상태를 가져야:

- 어떤 털은 진하게 남고
- 어떤 털은 거의 마르고
- 어떤 털은 종이에 닿지 않고
- 어떤 털은 다시 조금 보충되는

불균일한 패턴이 생긴다.

---

# 8. 붓 배 → 털 재공급

**[확인: 저장량에서 털로 채워짐 / 상세 수식은 추론]**

가능한 모델:

```ts
available = reservoirInk * refillRate[i] * dt
bristle[i].ink += available
reservoirInk -= available
```

여기에서 `refillRate[i]`에 작은 편차가 있어야 한다.

```ts
refillRate[i] =
  baseRefill
  * randomHairCapacity[i]
  * wetConnectivity[i]
```

원본 설명의 "채움이 느린 털부터 말라"라는 현상과 잘 맞는다.

---

# 9. 실제 먹붓 느낌의 핵심 2 — 먹 전달량

털이 종이에 닿았다고 먹을 전부 찍으면 도장이 된다.

먹 전달량은 최소 다음 네 요인의 함수여야 한다.

```
deposit_i =
  contact_i
× ink_i
× pressure_i
× transferRate
```

여기에 종이 거칠기를 더하면:

```
deposit_i =
  contact_i
× ink_i
× pressure_i
× paperContact_i
× transferRate
```

## paperContact

종이가 완전히 평평하지 않다고 보면:

- 높은 섬유에는 먼저 접촉
- 낮은 홈에는 마른 털이 잘 닿지 않음

이 때문에 마른 붓에서는 종이결 사이사이가 하얗게 남는다.

---

# 10. 갈필은 텍스처가 아니다

**[확인 + 핵심 해석]**

원본은 먹이 줄어들면 털이 갈라져 갈필이 난다고 설명한다.

따라서 갈필 생성 과정은:

```
먹 저장량 감소
→ 일부 털의 잔량 부족
→ 털별 전달량 차이 확대
→ 젖음 감소로 털 묶음 변화
→ 거친 종이의 높은 부분만 일부 접촉
→ 획 내부에 길고 불규칙한 공백 발생
```

이다.

잘못된 구현:

```
stroke
× noiseTexture
```

좋은 구현:

```
72개의 독립적인 contact/deposit 결과
→ 서로 겹쳐 하나의 획 형성
```

### 갈필 품질 판별법

갈필의 빈 부분이:

- 일정 간격이면 실패
- 점 노이즈처럼 보이면 실패
- 획 방향과 무관하면 실패

좋은 갈필은 **붓 진행 방향을 따라 길게 늘어난 불규칙한 섬유성 공백**을 가진다.

---

# 11. 필압이 바꾸는 것은 opacity가 아니다

**[강한 추론]**

필압이 증가하면:

1. 붓끝이 더 눌린다.
2. 털의 접촉 길이가 증가한다.
3. 털 묶음이 옆으로 퍼진다.
4. 접촉하는 털 수가 증가한다.
5. 종이에 전달되는 물/먹 총량이 증가한다.

따라서:

```
pressure
→ geometry
→ contact topology
→ ink transfer
```

로 연결되어야 한다.

```
pressure → opacity
```

하나만 연결하면 먹붓 느낌이 급격히 떨어진다.

---

# 12. 속도가 만드는 차이

**[강한 추론]**

빠른 붓:

- 접촉 시간 짧음
- 털끝 lag 큼
- 단위 거리당 먹 전달량 적음
- 갈필 발생 가능성 증가

느린 붓:

- 접촉 시간 김
- 더 많은 먹과 물 전달
- 종이 흡수 시작 시간이 빠름
- 진하고 안정된 획

권장 식:

```
transferPerDistance
≈ transferPerSecond / max(speed, epsilon)
```

다만 매우 느린 입력에서 폭주하지 않도록 clamp가 필요하다.

---

# 13. 먹과 물은 같은 변수가 아니다

**[확인]**

원본에는:

- 먹 농도
- 물 넣기
- 물 묻히기
- 물기 빼기
- 먹 찍기

가 별도 조작으로 존재한다.

즉 최소한 다음 두 상태는 독립적이다.

```
brush.pigmentMass
brush.waterMass
```

농묵/담묵의 차이는 검정색 alpha 차이가 아니라:

```
pigment concentration
= pigmentMass / waterMass
```

의 차이로 보는 것이 맞다.

---

# 14. 벼루 UX가 내부 상태에 주는 의미

**[확인: UX / 상태 연결은 강한 추론]**

### 물 넣기
벼루의 water 증가 → 먹 농도 감소 가능

### 벼루 문지르기
pigment concentration 증가

### 먹 찍기
벼루 → brush reservoir로 pigment/water transfer

### 물 묻히기
brush water 증가, pigment concentration 희석

### 물기 빼기
brush water 감소

따라서 원본 UX는 장식이 아니라 **시뮬레이션 상태를 현실 행위에 대응시킨 인터페이스**다.

---

# 15. 종이 격자: 2048 × 1152

**[확인]**

원본 설명:

- 종이 시뮬레이션 약 2048×1152칸
- 한 칸 약 0.31mm
- 현재 종이 64×36cm

이 숫자는 정확히 맞는다.

```
640 mm / 2048 = 0.3125 mm
360 mm / 1152 = 0.3125 mm
```

즉 이 격자는 화면 해상도 임의값이 아니라 **실제 종이 물리 크기에 대응하는 공간 단위**로 설계된 것으로 해석하는 것이 자연스럽다.

총 셀 수:

```
2048 × 1152 = 2,359,296 cells
```

약 236만 개다.

---

# 16. 종이 셀은 무엇을 가져야 하는가

원본 설명으로 최소 추정 가능한 상태:

```ts
type PaperCell = {
  water: number
  mobilePigment: number
  fixedPigment: number

  fiberDirection: Vec2
  absorbency: number

  // 가능성 높은 보조 상태
  height?: number
  capacity?: number
}
```

실제 구현에서 모든 정보를 한 texture에 넣을 필요는 없다.

정적 필드:
- fiberDirection
- absorbency
- height

동적 필드:
- water
- mobilePigment
- fixedPigment

로 나누는 것이 효율적이다.

---

# 17. 실제 먹 느낌의 핵심 3 — 문턱을 넘을 때만 흐르는 물

**[확인]**

원본은 물이 "문턱을 넘는 만큼만" 이웃 칸으로 흐른다고 설명한다.

단순 diffusion:

```
flow = k * (waterA - waterB)
```

만 사용하면 모든 방향으로 계속 퍼져 흐릿한 Gaussian blur처럼 된다.

문턱 모델:

```
head =
  waterA
- waterB
- localBarrier

flow =
  max(0, head) * conductivity
```

를 쓰면 작은 수분량은 제자리에 머물고, 충분히 젖었을 때만 인접 셀로 넘어간다.

이것이 실제 종이의 모세관 pinning과 비슷한 시각적 결과를 만든다.

---

# 18. 섬유 방향성 — 원형 blur를 막는 요소

**[확인]**

원본은 물이 종이 섬유결을 따라 더 잘 흐른다고 설명한다.

각 셀에 섬유 벡터 `f`가 있고, 이동 방향 `d`에 대해:

```
alignment = abs(dot(f, d))
```

```
conductivity =
  mix(crossFiberRate, alongFiberRate, alignment)
```

로 계산할 수 있다.

결과:

- 번짐의 외곽이 완벽한 원이 되지 않음
- 가느다란 돌기와 방향성이 생김
- 같은 물 양이어도 위치마다 다른 경계를 생성

수묵 느낌에서 매우 중요한 요소다.

---

# 19. 안료는 물을 따라간다

**[확인]**

원본은 안료가 물을 따라 이동한다고 설명한다.

따라서 안료 자체를 독립적으로 blur하는 구조가 아니다.

```
waterFlow(i → j)
        ↓
pigmentFlow(i → j)
= waterFlow
× localPigmentConcentration
× pigmentMobility
```

이 연결 때문에:

- 물 많은 담묵은 멀리 이동
- 물 적은 농묵은 중심에 남음
- 젖은 곳에 다시 먹을 대면 wet-in-wet 효과 발생

이 가능해진다.

---

# 20. 건조와 고착

**[확인]**

원본은 물이 마르는 자리에 안료가 남는다고 설명한다.

따라서 동적 안료와 고착 안료를 분리하는 것이 자연스럽다.

```
mobilePigment
  ↓ drying / absorption
fixedPigment
```

예:

```
settlingRate =
  baseSettle
× paperAbsorbency
× (1 - wetness)
```

물이 많을 때는 안료가 오래 움직이고,
물이 적을 때 빠르게 고착된다.

---

# 21. 실제 먹 느낌의 핵심 4 — 가장자리 농축

**[확인]**

원본은 안료가 물이 마르는 자리에 남아 가장자리가 짙어진다고 설명한다.

이것은 수채/먹 번짐에서 흔히 보이는 coffee-ring 계열 효과와 유사한 시각 현상이다.

과정:

```
젖은 영역의 외곽이 먼저 마름
→ 내부 수분이 외곽 쪽으로 이동
→ 이동하는 물이 안료를 운반
→ 경계에 안료가 더 많이 고착
→ 테두리가 중심보다 짙어짐
```

이 효과가 없으면 번짐은 단순한 blur처럼 보인다.

---

# 22. 번짐은 '획 이후'에도 진행된다

이 앱에서 붓을 뗀 뒤에도 종이 상태는 계속 계산된다.

이는 디지털 펜 앱과 가장 큰 차이 중 하나다.

```
pointerUp
≠ simulationEnd
```

```
pointerUp
→ water continues flowing
→ pigment continues moving
→ evaporation
→ deposition
→ final dry stroke
```

수묵화에서 매체가 스스로 움직이는 느낌을 만드는 핵심이다.

---

# 23. 붓과 종이가 결합될 때 생기는 6개 대표 효과

## 23.1 농묵
- pigment 높음
- water 낮거나 중간
- 빠른 고착
- 중심 농도 높음
- 번짐 제한

## 23.2 담묵
- pigment 낮음
- water 높음
- 넓은 이동
- 낮은 광학 농도
- 부드러운 경계

## 23.3 윤필
- water 충분
- 털 응집
- 연속 접촉
- 매끈하면서 촉촉한 획

## 23.4 갈필
- water 낮음
- pigment reservoir 낮음
- 털 접촉 불균일
- 종이 높은 부분만 선택적 deposit

## 23.5 발묵
- 매우 높은 water
- 종이 확산 활성
- 안료 이동 범위 큼
- 경계 변화가 시간에 따라 큼

## 23.6 겹침/적묵
- 이미 고착된 pigment 위에 새 water + pigment
- 새 층의 이동과 기존 고착층의 시각 합성
- 동일 black이라도 밀도 중첩으로 깊이 생성

---

# 24. 같은 '검정'인데 왜 깊이가 달라지는가

단순 alpha blending:

```
black rgba(0,0,0,0.5)
```

를 여러 번 덮는 것만으로도 어두워질 수 있지만 실제 먹의 깊이는 충분히 나오지 않는다.

권장 개념:

```
opticalDensity = pigmentMass × absorption
transmittance = exp(-opticalDensity)
darkness = 1 - transmittance
```

즉 안료 질량과 보이는 농도를 비선형으로 연결한다.

이를 사용하면 농묵이 단순히 opacity slider를 올린 검정과 달라진다.

---

# 25. 종이 표면 질감은 두 층으로 나눠야 한다

## A. 보이는 종이
- 한지 색
- 미세 섬유
- 밝기 노이즈
- 작은 얼룩

## B. 물리 종이
- height
- absorbency
- fiber direction
- water capacity

보이는 texture 하나를 배경에 깔고 끝내면 종이질감은 장식이다.

물리 종이 필드가 물 흐름에 영향을 주어야 실제 효과가 난다.

---

# 26. WebGPU가 필요한 이유

**[확인]**

원본은 종이 격자를 GPU compute shader로 계산한다고 설명한다.

236만 셀을 매 step마다 처리한다고 보면 CPU JavaScript에는 부담이 크다.

한 번의 4-neighbor pass만 해도 대략:

```
2.36M cells × 4 neighbors
≈ 9.4M neighbor interactions
```

이다.

여기에:
- water flow
- pigment transport
- drying
- deposition
- edge handling

등 여러 pass가 더해진다.

GPU에서는 셀을 병렬로 처리할 수 있기 때문에 이 구조가 자연스럽다.

---

# 27. GPU 메모리 감각

실제 texture 포맷은 확인되지 않았다.

다만 2048×1152에서 한 texture의 이론적 크기는:

| 포맷 예 | 대략 크기 |
|---|---:|
| R32F | 9 MiB |
| RGBA16F | 18 MiB |
| RGBA32F | 36 MiB |

ping-pong을 하면 두 배가 필요하다.

따라서 원본과 같은 고해상도에서는 **상태를 어떤 channel에 packing하는지가 성능에 중요**하다.

이 값은 원본 포맷을 주장하는 것이 아니라 구현 규모를 이해하기 위한 계산이다.

---

# 28. 고정 timestep + replay

**[확인]**

원본은 최종 그림을 저장해 재생하는 대신 붓 입력을 물리 step마다 기록하고 빈 종이에서 다시 계산한다고 설명한다.

이 방식의 장점:

- timelapse
- 제작 과정 재생
- 동일 입력 재현
- 품질 설정을 바꿔 재렌더 가능
- final bitmap보다 과정 정보를 보존

가능한 이벤트:

```ts
type ReplaySample = {
  tick: number
  x: number
  y: number
  pressure: number
  brushWater: number
  brushInk: number
  brushSize: number
}
```

중요한 점은 replay가 정확하려면 physics step과 random seed도 재현 가능해야 한다는 것이다.

---

# 29. 이 앱이 '실제 붓'처럼 느껴지는 이유를 감각 단위로 해체

| 사용자가 느끼는 현상 | 내부 원인 |
|---|---|
| 붓이 부드럽다 | 8입자 chain + 탄성 |
| 붓끝이 뒤늦게 따라온다 | 물리 substep + tip inertia |
| 눌렀을 때 넓어진다 | bristle spread/contact 증가 |
| 획 끝이 자연스럽다 | 털별 lift timing 차이 |
| 마른 붓이 갈라진다 | 털별 ink/water 잔량 차이 |
| 종이에서 까슬한 공백이 난다 | 접촉 + paper microfield |
| 먹물이 계속 번진다 | water field simulation |
| 가장자리가 짙어진다 | drying-front pigment deposition |
| 같은 검정인데 깊이가 다르다 | pigment mass / optical density |
| 같은 획도 매번 조금 다르다 | 다중 털 + 종이 field + 시간 상태 |

---

# 30. 원본과 유사한 느낌을 만들 때 가장 중요한 순서

정밀 물리의 모든 요소가 같은 중요도를 갖지 않는다.

체감 품질 기여도를 다음처럼 본다.

## Tier S
1. bristle별 독립 접촉
2. bristle별 ink depletion
3. pressure → geometry
4. water / pigment 분리
5. edge deposition

## Tier A
6. tip lag
7. paper fiber anisotropy
8. wet bristle clustering
9. paper height/contact irregularity
10. post-stroke drying

## Tier B
11. 고해상도 grid
12. 완전한 유체 방정식
13. 매우 많은 bristles
14. 정밀 3D brush geometry

놀아트는 Tier S/A부터 구현해야 한다.

---

# 31. 반대로 효과가 적은 '가짜 수묵' 접근

다음 방법만으로는 원본 수준의 느낌이 나오기 어렵다.

- 검정 brush texture stamp
- opacity jitter
- Gaussian blur
- 종이 JPG overlay
- stroke마다 random splatter
- pressure → width만 연결
- dry brush용 별도 PNG mask

이런 효과는 보조 요소로는 쓸 수 있지만 **주 시뮬레이션을 대체하면 안 된다.**

---

# 32. 원본에서 확인되는 fallback 철학

원본은 WebGPU가 불가능하면 별도의 2D 수묵 화실을 안내한다.

중요한 제품 원칙은:

> 고품질 엔진을 못 돌리는 기기에서 고품질처럼 보이는 가짜 화면을 억지로 보여주기보다, 계산량을 낮춘 별도 표현 모델을 제공한다.

놀아트에도 이 철학이 적절하다.

---

# 33. 놀아트에 적용할 때의 권장 축소

원본을 그대로 재현할 필요가 없다.

## 학교 기본
- 24~32 bristles
- bristle당 3~4 control point 또는 간단 spring tip
- 768~1024 simulation width
- water/mobile/fixed pigment
- procedural fiber field
- WebGL2 또는 WebGPU

## 고품질
- 48~64 bristles
- 5~6 particles
- 1536~2048 simulation width
- WebGPU compute
- 더 세밀한 paper field

원본의 핵심 감각은 **72라는 숫자 자체가 아니라 독립 bristle state**에 있다.

---

# 34. 구현 시 반드시 분리할 모듈

```
InkEngine
├─ InputSampler
├─ BrushDynamics
│  ├─ BristleBundle
│  ├─ ConstraintSolver
│  └─ ContactSolver
├─ BrushMaterial
│  ├─ Reservoir
│  ├─ InkTransfer
│  └─ WaterTransfer
├─ PaperModel
│  ├─ FiberField
│  ├─ HeightField
│  └─ AbsorbencyField
├─ WetSimulation
│  ├─ WaterFlow
│  ├─ PigmentAdvection
│  ├─ Deposition
│  └─ Evaporation
├─ Renderer
└─ Replay
```

붓과 종이 시뮬레이션을 한 파일에 섞으면 튜닝하기 매우 어려워진다.

---

# 35. 구현 검증용 최소 실험 10개

1. 같은 경로, 압력만 다르게
2. 같은 경로, 속도만 다르게
3. 같은 경로, water만 다르게
4. 같은 경로, ink reservoir만 다르게
5. 급격한 90도 방향 전환
6. 붓을 천천히 들어 올리기
7. 거의 마른 붓으로 긴 획
8. 젖은 종이 위에 농묵
9. 동일 입력을 paper preset만 바꿔 replay
10. 동일 입력을 두 번 replay해 결정성 확인

---

# 36. 가장 중요한 판단

원본의 뛰어난 점은 "수묵처럼 보이는 shader"가 아니다.

**붓의 상태와 종이의 상태가 서로 물질을 주고받고, 그 상태가 다음 순간의 결과를 바꾸는 폐쇄 루프**가 만들어져 있다는 점이다.

```
brush state
→ paper state
→ visible mark
→ brush depletion
→ next brush state
→ next mark
```

이 feedback loop가 없으면 디지털 붓이고,
이 loop가 있으면 비로소 '재료를 다루는 느낌'에 가까워진다.
