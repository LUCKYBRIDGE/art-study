# art-study

미술 교육용 웹·앱·도구의 **표현 원리, UI/UX, 기술 구조, 교육적 활용 가능성**을 분석하고, 놀아트(KnolArt)에 재사용 가능한 설계 지식으로 축적하는 저장소입니다.

## 분석 원칙

이 저장소는 `storytold/craft-anything`의 Study → Build → Check 관점을 참고합니다.

1. **Study** — 화면에 보이는 기능, 상호작용, 표현 방식, 기술적 단서를 분리해 관찰
2. **Model** — 핵심 표현을 상태·규칙·데이터 흐름 단위로 재구성
3. **Transfer** — 놀아트에 가져올 수 있는 요소와 가져오면 안 되는 요소를 구분
4. **Prototype** — 실제 구현 가능한 모듈·파라미터·렌더링 파이프라인 수준으로 정리
5. **Check** — 원본 복제가 아니라 독립 구현이 목적이며, 교육성과 성능을 별도로 검증

## 현재 분석

- [수묵화 웹앱 — dew978.github.io/sumukhwa](./services/sumukhwa/README.md)
- [놀아트 수묵화 모드 적용 설계](./knolart/sumukhwa-mode-design.md)
- [구현 체크리스트](./knolart/sumukhwa-implementation-checklist.md)
