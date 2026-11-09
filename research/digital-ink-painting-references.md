# Digital Ink / Brush / Paper Simulation References

수묵화 엔진을 독립 구현하기 위한 기술 참고 문헌.

## 1. Position-Based Dynamics

### Müller et al. — Position Based Dynamics
- https://doi.org/10.1016/j.jvcir.2007.01.005
- https://doi.org/10.2312/PE/vriphys/vriphys06/071-080

핵심:
- 위치를 직접 constraint projection
- 실시간 deformable body에서 안정적
- 붓털 chain의 길이/충돌/탄성 제약에 적합

---

## 2. Chinese brush simulation

### Wong & Ip — Virtual brush: a model-based synthesis of Chinese calligraphy
https://doi.org/10.1016/S0097-8493(99)00141-7

핵심:
- brush geometry
- hair property
- ink deposition
- stroke trajectory를 하나의 시스템으로 모델링

### Chu & Tai — Real-time painting with an expressive virtual Chinese brush
https://doi.org/10.1109/MCG.2004.37

핵심:
- deformable brush tuft
- spreading bristles
- real-time brush dynamics
- ink loading/deposition

### Xu et al. — A Solid Model Based Virtual Hairy Brush
https://doi.org/10.1111/1467-8659.00589

핵심:
- hair cluster를 writing primitive로 묶어 계산 비용 감소
- brush behavior와 paper imprint 연결

### Bai et al. — Chinese Hairy Brush: A Physically-Based Model for Calligraphy
https://scholars.cityu.edu.hk/en/publications/chinese-hairy-brush-a-physically-based-model-for-calligraphy/

핵심:
- geometry / dynamic / ink model 분리
- 개별 hair 전체를 계산하지 않는 경량 모델 가능

### Mi et al. — Droplet: A Virtual Brush Model to Simulate Chinese Calligraphy and Painting
https://jcst.ict.ac.cn/article/id/1018

핵심:
- 물리 모델을 지나치게 복잡하게 하지 않고 expressive effect를 유지하는 대안
- dry brush 등의 표현 지원

---

## 3. Ink / Water / Paper

### Small — Simulating watercolor by modeling diffusion, pigment, and paper fibers
https://www.npcglib.org/paper.php?entryid=486

핵심:
- cell 기반 water/pigment movement
- paper fiber
- absorbency
- surface tension
- local rule 반복만으로 복잡한 결과 생성

### Curtis et al. — Computer-Generated Watercolor
https://doi.org/10.1145/258734.258896

PDF mirror:
https://grail.cs.washington.edu/wp-content/uploads/2015/08/curtis-1997-cgw.pdf

핵심:
- paper height/capacity
- water layer
- pigment movement
- deposition
- capillary flow
- edge darkening
- granulation
- optical glazing

### Wang & Rokne — Pressure Based Ink Diffusion Model for Real-Time Simulation of Chinese Calligraphy
https://www.scitepress.org/papers/2007/20719/

핵심:
- multi-layer paper
- capillary network
- Darcy-law-inspired pressure
- carbon pigment deposition

### Ren et al. — Ink Diffusion Simulation in Chinese Calligraphy Using Navier–Stokes Equation
검색 키워드:
Ink Diffusion Simulation in Chinese Calligraphy Using Navier–Stokes Equation

핵심:
- Lattice Boltzmann
- three-layer paper model
- multi-concentration ink

---

## 4. Recent real-time watercolor implementation

### Daeyoung Kim — Artist-Centered Digital Watercolor
https://dubu911.github.io/papers/watercolor_simulation_paper.pdf

핵심:
- water / mobile pigment / static pigment
- double buffering
- GPU compute
- evaporation
- displacement
- pigment transport
- deposition
- Beer-Lambert optical density

이 논문은 수묵 원본의 구현 소스가 아니라 별개 연구다.

---

## 5. Open-source comparative implementation

### Washes
Repository:
https://github.com/castavridis/washes-js

License:
MIT

흥미로운 코드 구조:
- WebGL GPU simulation
- fluid state: velocity / pressure / wetness
- paper height texture
- pigment advection
- deposit layer
- wet diffusion
- edge darkening
- brush stamp shader
- dry brush anisotropy
- bristle-skip-like stochastic contact

특히 `engine/src/shaders/brush_stamp.frag`는 dry brush를 단순 alpha-noise가 아니라:
- paper field
- motion direction
- dryness
- anisotropy
- stochastic skip

의 조합으로 만드는 비교 사례다.

주의:
- 이는 dew978 수묵화 앱의 코드가 아니다.
- 분석 및 놀아트 독립 설계를 위한 비교 자료일 뿐이다.

---

## 6. 수묵 미술 개념 참고

한국민족문화대백과 — 수묵화
https://encykorea.aks.ac.kr/Article/E0031370

수묵화에서 핵심은 먹의 단순 흑백이 아니라 농담과 번짐, 파묵/발묵 등 물과 먹의 운용이다.

서울대학교 동양화 교과 소개에서도:
- 건필 / 윤필
- 발묵 / 파묵
- 선염
- 적묵

등 다양한 필묵법이 별도 표현 영역으로 다뤄진다.

---

# 7. 놀아트에 사용할 연구 우선순위

## 먼저 읽을 것
1. PBD
2. Chu & Tai virtual brush
3. Curtis watercolor
4. Wang & Rokne ink diffusion

## 구현하면서 비교할 것
5. Washes source
6. Artist-Centered Digital Watercolor

## 고급화 단계
7. LBM / Navier-Stokes ink diffusion
8. optical pigment models

