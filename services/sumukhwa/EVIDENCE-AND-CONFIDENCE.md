# 수묵화 분석 증거 지도와 신뢰도

## 목적

수묵화 웹앱의 공개 UI 설명은 매우 구체적이지만 공개 소스코드는 확인되지 않았다.

따라서 이 저장소에서는 **관찰 사실 / 학술적으로 타당한 해석 / 놀아트용 제안**을 섞지 않는다.

---

## A. 원본에서 직접 확인된 사실

대상: https://dew978.github.io/sumukhwa/

### Brush
- 붓털 72가닥
- 한 가닥을 8입자 chain으로 계산
- 1/240초 PBD
- 길이 제약
- shape-restoring elasticity
- wet bristle clustering
- paper friction/spread
- 털별 먹 양
- brush belly reservoir에서 재공급
- 먹이 줄면서 일부 털부터 마르는 구조
- 갈필 발생

### Paper
- 2048×1152 simulation grid
- 약 0.31mm/cell
- water threshold 기반 이웃 이동
- fiber direction을 따라 더 잘 흐름
- pigment가 water를 따라감
- drying 위치에서 pigment가 남음
- edge가 짙어지는 현상

### UI/state
- 먹 농도
- 붓 크기
- 필압
- 붓털 탄력
- 번짐
- 먹 찍기
- 물 묻히기
- 물기 빼기
- 물 넣기
- 벼루 문지르기
- 종이/먹 조합 preset
- brush skeleton 보기
- 고품질/기본 graphics preset
- replay
- timelapse
- PNG/JPG 저장
- 낙관
- WebGPU 사용
- 별도 2D fallback 안내

### replay
- 최종 이미지를 재생하는 방식이 아니라 physics step 단위 brush input 기록 후 빈 종이에서 재계산한다고 설명

---

## B. 강한 기술적 추론

아래는 원본이 정확한 수식을 공개하지 않았지만 공개 구조를 구현하려면 매우 자연스러운 해석이다.

- PBD가 fixed timestep substeps로 실행
- bristle contact count가 pressure에 따라 증가
- tip lag가 chain deformation에서 발생
- bristle별 refill rate/capacity에 편차가 존재하거나 동등한 효과가 있음
- water와 pigment가 별도 state field
- mobile/fixed pigment에 해당하는 상태 분리
- paper static field와 dynamic wet field 분리
- GPU ping-pong 또는 이에 준하는 read/write buffer 구조
- replay 결정성을 위해 fixed dt/random seed 관리

이 항목들은 원본 구현을 주장하는 것이 아니다.

---

## C. 원본에서 확인되지 않은 것

다음은 현재 증거로 단정하지 않는다.

- 정확한 WGSL shader 코드
- texture format (R16F/R32F/RGBA16F/RGBA32F 등)
- compute workgroup 크기
- paper simulation update frequency
- PBD constraint iteration count
- CPU/GPU 중 어느 쪽에서 brush PBD를 계산하는지
- 각 bristle의 refill 공식
- paper height field 생성 알고리즘
- fiber field 생성 방식
- edge-darkening 공식
- optical compositing 공식
- random seed 구조
- undo snapshot 구조

---

# Clean-room / 저작권 경계

원본 페이지에는 교실 내 비영리 사용 외 복제·수정·상업적 이용을 제한하는 저작권 안내가 표시된다.

따라서 놀아트에서는 다음 원칙을 유지한다.

## 하지 않는 것
- 원본 HTML/CSS/JS 복사
- 원본 shader 추출/복사
- 원본 아이콘/그래픽/종이 texture 복사
- UI 배치와 시각 디자인의 직접 모사
- 원본 고유 문구의 대량 전재

## 하는 것
- 공개된 동작과 기술 설명을 연구
- PBD, fluid/pigment simulation 등 일반적인 컴퓨터 그래픽스 기법 연구
- 학술 문헌과 오픈소스 구현을 별도 참고
- 놀아트의 교육 목적에 맞게 독립 설계
- 자체 코드/texture/preset 제작

이 분석의 목적은 원본 복제가 아니라 **표현 원리를 이해해 독립 구현하는 것**이다.

---

# 참고 연구와의 대조

## Position-Based Dynamics

Matthias Müller et al.의 PBD는 속도/힘 적분 중심 방법과 달리 위치 제약을 직접 해결해 실시간 변형 모델에서 안정적 제어가 가능하도록 한다.

- https://doi.org/10.1016/j.jvcir.2007.01.005
- https://doi.org/10.2312/PE/vriphys/vriphys06/071-080

원본이 붓털 chain에 PBD를 사용한다는 설명과 잘 부합한다.

## Virtual Chinese Brush

Chinese brush simulation 연구에서는 오래전부터:
- brush geometry
- brush dynamics
- ink loading/deposition

을 분리해 모델링해 왔다.

- Wong & Ip, Virtual brush: a model-based synthesis of Chinese calligraphy  
  https://doi.org/10.1016/S0097-8493(99)00141-7
- Chu & Tai, Real-time painting with an expressive virtual Chinese brush  
  https://doi.org/10.1109/MCG.2004.37
- Bai et al., Chinese Hairy Brush  
  https://scholars.cityu.edu.hk/en/publications/chinese-hairy-brush-a-physically-based-model-for-calligraphy/

## Water/pigment/paper

- Small, Simulating watercolor by modeling diffusion, pigment, and paper fibers  
  https://www.npcglib.org/paper.php?entryid=486
- Curtis et al., Computer-Generated Watercolor  
  https://doi.org/10.1145/258734.258896
- Wang & Rokne, Pressure Based Ink Diffusion Model for Real-Time Simulation of Chinese Calligraphy  
  https://www.scitepress.org/papers/2007/20719/

이 연구들은 water transport, pigment transport, paper absorbency/fiber, deposition을 분리하는 접근의 타당성을 뒷받침한다.

---

# 비교용 오픈소스 구현

## Washes

Repository:
https://github.com/castavridis/washes-js

License:
MIT

이 프로젝트는 원본 수묵화 앱의 소스가 아니다. 별개의 watercolor engine이다.

분석 가치:
- GPU ping-pong state
- wet diffusion
- pressure/velocity field
- pigment advection
- deposit layer
- paper height
- edge darkening
- dry brush용 anisotropy / bristle skip

특히 dry brush를 만들 때 단일 opacity noise보다:
- paper field
- motion direction
- anisotropy
- bristle skip

을 결합하는 실제 코드 사례를 제공한다.

놀아트에서 참고할 때에도 원본 수묵화 앱과 동일 구현이라고 오인하면 안 된다.

---

# 신뢰도 요약

| 항목 | 신뢰도 |
|---|---|
| 72 bristles / 8 particles | 매우 높음 — 원본 직접 설명 |
| PBD 1/240 sec | 매우 높음 |
| bristle-level ink state | 매우 높음 |
| brush reservoir | 매우 높음 |
| 2048×1152 paper grid | 매우 높음 |
| fiber-direction flow | 매우 높음 |
| pigment follows water | 매우 높음 |
| edge deposition | 매우 높음 |
| pressure → geometry/contact | 높음 — 물리 모델상 강한 추론 |
| ping-pong GPU textures | 중간 — 일반적 구현이지만 원본 미확인 |
| exact shader equations | 낮음/미확인 |
| exact texture formats | 미확인 |

