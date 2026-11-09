# 놀아트 수묵화 모드 구현 체크리스트

## P0 — 반드시 구현
- [ ] Pointer Events 기반 입력
- [ ] 24~32 bristle bundle
- [ ] bristle별 ink/water 상태
- [ ] pressure → bristle spread
- [ ] 먹 고갈
- [ ] 갈필 자동 발생
- [ ] paper water map
- [ ] pigment map
- [ ] 간단한 diffusion
- [ ] evaporation
- [ ] edge darkening
- [ ] 한지 시각 texture
- [ ] 먹 묻히기 / 물 묻히기 / 물기 빼기 UX
- [ ] undo
- [ ] PNG 저장

## P1 — 표현 품질
- [ ] anisotropic fiber diffusion
- [ ] mobile / fixed pigment 분리
- [ ] bristle wet clustering
- [ ] speed 기반 drag
- [ ] pressure fallback
- [ ] 종이 preset
- [ ] 먹 preset
- [ ] 붓 preset
- [ ] visual / simulation resolution 분리

## P1 — 교육 기능
- [ ] 붓털 보기
- [ ] 번지는 과정 보기
- [ ] 같은 획 조건 비교
- [ ] 작품 과정 replay
- [ ] 타임랩스
- [ ] 교사 시범 replay

## P2 — 고급
- [ ] WebGPU backend
- [ ] WebGL fallback
- [ ] GPU quality detection
- [ ] adaptive simulation resolution
- [ ] stylus tilt
- [ ] seal/낙관
- [ ] 프로젝트 파일 저장
- [ ] replay 데이터 압축

---

## 성능 목표

### 학교용 기준
- 저가형 Android 태블릿에서도 입력 지연이 체감되지 않을 것
- 기본 모드 30fps 이하로 장시간 떨어지지 않을 것
- diffusion simulation이 입력 thread를 막지 않을 것
- WebGPU 미지원 시 기능 자체가 사라지지 않을 것

### 권장 최적화
- simulation texture를 canvas보다 낮은 해상도로 유지
- dirty region만 계산
- pointer sample coalescing
- fixed timestep
- worker/offscreen canvas 검토
- GPU ping-pong textures
- replay는 delta encoding

---

## 검증 장면

### Test 1 — 농묵 획
조건:
- ink 100%
- water 20%
- medium pressure

기대:
- 중심이 진함
- 번짐 적음
- 가장자리 약간 거침

### Test 2 — 담묵
- ink 40%
- water 70%

기대:
- 넓은 번짐
- 농도 감소
- 건조 경계가 보임

### Test 3 — 갈필
- ink reservoir 10%
- water 10%
- 빠른 stroke

기대:
- 여러 bristle이 불연속적으로 흔적 남김
- 단순 dashed line처럼 보이면 실패

### Test 4 — 눌러 그리기
pressure 0.2 → 0.9 → 0.2

기대:
- 획 폭이 연속적으로 넓어졌다 좁아짐
- opacity만 변하면 실패

### Test 5 — 방향 전환
빠르게 ㄱ자 stroke

기대:
- tip lag가 생김
- 각도가 완벽한 벡터 코너처럼 꺾이면 실패

### Test 6 — 종이 비교
동일 stroke를 서로 다른 paper preset에 replay

기대:
- 같은 bitmap tint만 달라지는 것이 아니라 diffusion 형태 자체가 달라짐

---

## 원본과 비교할 때 볼 지표

시각적 픽셀 일치보다 아래를 본다.

1. 획의 다양성
2. 압력 변화 반응
3. 먹 고갈 자연스러움
4. 갈필의 비주기성
5. 젖은 획의 시간 변화
6. 섬유 방향성
7. 건조 경계 농축
8. 학생이 조작 원리를 이해할 수 있는가
9. 학교 태블릿에서 충분히 빠른가

---

## 하지 말아야 할 구현

- [ ] PNG 먹 텍스처를 선 위에 반복 stamp 하는 것만으로 완료 처리
- [ ] Gaussian blur만으로 번짐 구현
- [ ] opacity 감소를 갈필이라고 부르기
- [ ] 종이 배경 이미지만 바꾸고 paper preset이라고 부르기
- [ ] WebGPU 전용으로 만들어 저사양 기기를 배제
- [ ] 미세 조절 슬라이더를 기본 UI에 모두 노출
- [ ] 원본 웹의 그래픽/UI를 그대로 복제

---

## Definition of Done

수묵화 모드 v1은 다음 질문에 모두 YES일 때 완료로 본다.

- 같은 손동작이어도 먹/물 상태에 따라 다른 획이 나오는가?
- 먹이 떨어지면서 갈필이 자연스럽게 생기는가?
- 획을 그은 뒤에도 잠시 번짐이 진행되는가?
- 종이 종류에 따라 번짐 패턴이 달라지는가?
- 학생이 '먹 찍기 → 물 조절 → 그리기'의 관계를 이해할 수 있는가?
- WebGPU가 없는 기기에서도 기본 기능을 사용할 수 있는가?
- 결과뿐 아니라 과정 데이터도 보존할 수 있는가?
