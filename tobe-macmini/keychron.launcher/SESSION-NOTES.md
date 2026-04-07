# Keychron Launcher Prototype - 세션 노트

## 마지막 업데이트: 백라이트 페이지 대규모 개편 (2026-02-14)

---

## 현재 진행 상황

### 5-Phase 파인튜닝 플랜

| Phase | 상태 | 내용 |
|-------|------|------|
| **Phase 1** | ✅ 완료 | i18n 완성 — settings, feedback, he-mode |
| **Phase 2** | ✅ 완료 | i18n 완성 — backlight 이펙트명 + advance 페이지 |
| **Phase 3** | ✅ 완료 | 인라인 스타일 → CSS — settings + feedback |
| **Phase 4** | ⬜ 대기 | 인라인 스타일 → CSS — backlight + keymap |
| **Phase 5** | ⬜ 대기 | connect `<style>` 블록 이동 + advance 인라인 정리 |

### 추가 완료 작업 (플랜 외)

| 작업 | 상태 | 내용 |
|------|------|------|
| **Assist i18n** | ✅ 완료 | 빠른시작 페이지 셋업 가이드 + 상태 텍스트 i18n 처리 |
| **Macro 버튼** | ✅ 완료 | 적용 버튼 width:100% → min-width:160px, 우측 정렬 |
| **Macro 리스트 에디터** | ✅ 완료 | 칩 타임라인 → 리스트 기반 에디터로 전면 교체 (실제 키크론 런처 참고) |
| **네비 그라디언트 효과** | ✅ 완료 | 사이드바 nav-item에 애니메이팅 그라디언트 호버/활성 효과 (실제 런처 참고) |
| **백라이트 대규모 개편** | ✅ 완료 | 23개 이펙트, 동적 키보드 애니메이션, HSV 피커, Per-key/Mix/Indicator 탭 완성 |
| **CSS 캐시 방지** | ✅ 완료 | index.html에 개발용 cache-busting 스크립트 추가 |
| **토글 마크업 수정** | ✅ 완료 | backlight.html의 toggle-slider → toggle-track + toggle-thumb 수정 |

---

## 완료된 작업 히스토리

### 네비게이션 그라디언트 효과 (커밋: cd4e234)

**배경:**
- 기존 사이드바 nav-item은 단색 배경(hover/active)으로 단조로웠음
- 실제 Keychron Launcher 분석 → `::after` pseudo-element에 `linear-gradient` + `@keyframes` 애니메이션 사용
- 실제 런처는 청록+보라+파랑+핑크 다색 그라디언트, 우리는 키크론 오렌지 기반으로 차별화

**주요 변경:**
- `css/layout.css` — nav-item에 animated gradient 효과 추가
  - `@keyframes nav-gradient-shift` — background-position 0%→100%→0% (8초 무한 루프)
  - `::after` pseudo-element: 6색 오렌지 계열 그라디언트, `background-size: 500%`
  - 비활성: `opacity: 0`, 호버: `opacity: 0.55`, 활성: `opacity: 1`
  - `transition: opacity 0.5s ease` — 부드러운 페이드인/아웃
  - `.nav-item > *`에 `z-index: 1`로 텍스트/아이콘을 그라디언트 위에 표시
  - `.nav-item.active::before`(좌측 바)에 `z-index: 2` 추가
  - 라이트 테마용 별도 그라디언트 (opacity 낮춤)


### Macro 리스트 에디터 교체 (커밋: 912a421)

**배경:**
- 기존 매크로 에디터는 칩(chip) 기반 타임라인 디자인이었음
- 실제 Keychron Launcher (`launcher.keychron.com`) 분석 결과, 리스트 기반 에디터가 훨씬 직관적
- V2 프로토타입을 별도 페이지(`macro-v2.html`)에서 개발 후 원본에 반영

**주요 변경:**
- `pages/macro.html` — 칩 타임라인 에디터 → 리스트 기반 이벤트 에디터로 전면 교체
  - 2-칼럼 레이아웃: 녹화 컨트롤 (140px) | 이벤트 리스트 (max-width 560px)
  - 카드 스타일 이벤트 행: 컬러 좌측 보더 (녹=누르기, 빨=릴리즈, 회=딜레이, 파=텍스트)
  - 커스텀 녹화 버튼: 중립 회색 pill + 빨간 점 (녹화중 빨간 배경 + 흰 점 깜박임)
  - 항상 보이는 행 액션 버튼 (위/아래/더보기)
  - 컨텍스트 메뉴: 위에 삽입 / 아래에 삽입 / 삭제
  - QMK 코드 동기화 (코드 입력 탭)
- `css/pages.css` — 새 `ml-*` 클래스 추가 + 기존 칩 CSS (`mv2-toolbar`, `mv2-evt-*`, `mv2-timeline-*`) 삭제
- `macro-v2.html` 삭제, `app.js`에서 macro-v2 라우트 제거

**새로 추가된 CSS 클래스:**
- `ml-editor-split`, `ml-record-col`, `ml-rec-btn`, `ml-rec-dot`
- `ml-list-col`, `ml-list-header`, `ml-list-title`, `ml-list-actions`
- `ml-event-list`, `ml-row`, `ml-row--press/release/delay/text`
- `ml-row-check`, `ml-row-num`, `ml-row-type`, `ml-row-value`
- `ml-key-input`, `ml-delay-input`, `ml-delay-unit`, `ml-text-input`
- `ml-row-actions`, `ml-row-btn`, `ml-add-btn`
- `ml-context-menu`, `ml-ctx-item`, `ml-ctx-danger`, `ml-ctx-divider`
- `ml-code-textarea`, `ml-empty-state`, `ml-empty-hint`

### Phase 1: i18n 완성 — settings, feedback, he-mode (커밋: c6afd26)

**수정된 파일:**
- `i18n/ko.json` — ~25개 새 번역 키 추가
- `i18n/en.json` — 위와 동일한 영어 키
- `pages/settings.html` — data-i18n 속성 추가 (테마, 장치정보, 로그 버튼 등)
- `pages/feedback.html` — data-i18n 속성 추가 (이메일, 첨부파일, 드롭존, 토스트 등)
- `pages/he-mode.html` — JS 내 "No Data", "keys" 텍스트 → I18n.t() 호출
- `js/i18n.js` — JSON fetch에 cache-busting 추가 (`{ cache: 'no-store' }`)
- `js/app.js` — 페이지 템플릿 fetch에 cache-busting 추가

### Phase 2: i18n 완성 — backlight + advance (커밋: e522859)

**수정된 파일:**
- `i18n/ko.json` — 효과명 13개 + 팔레트 4개 + advance 18개 키 추가
- `i18n/en.json` — 동일
- `pages/backlight.html` — 13개 이펙트 칩 + 4개 팔레트명에 data-i18n 추가
- `pages/advance.html` — DKS/Toggle/TapHold/Combo 패널 텍스트에 data-i18n 추가

### Phase 3: 인라인 스타일 → CSS — settings + feedback (커밋: b6524e1)

**수정된 파일:**
- `css/pages.css` — settings/feedback용 CSS 클래스 ~40개 추가 + `.toast`에 flex 속성 보강
- `pages/settings.html` — 인라인 스타일 전면 제거, CSS 클래스 교체
- `pages/feedback.html` — 인라인 스타일 ~15개 제거, CSS 클래스 교체

**핵심 변경 패턴:**
- HTML `style="..."` → CSS 클래스
- JS `el.style.xxx = '...'` → `el.classList.toggle('className')`
- JS `onfocus`/`onblur`/`mouseenter`/`mouseleave` → CSS `:focus`/`:hover`

### Assist 페이지 i18n (커밋: b6524e1에 포함)

- `pages/assist.html` — 14개 하드코딩 텍스트에 `data-i18n` 속성 추가, JS 내 3개 문자열 `I18n.t()` 호출

### Macro 적용 버튼 수정 (커밋: c498416)

- `css/pages.css` — `.macro-submit-btn` width:100% → min-width:160px
- `css/pages.css` — `.macro-submit-bar` justify-content: center → flex-end

### 이전 세션 작업 (이 플랜 이전)

- **매크로 페이지 리디자인**: 2-서브-칼럼 에디터 레이아웃 (키크론 런처 매칭)
- **백라이트 2-칼럼 레이아웃**: 이펙트 칩 + 세팅 패널 나란히 배치
- **연결 페이지**: 3-스텝 카드 + 데모 모드 버튼
- **12개 전체 페이지 프로토타입 완성**

---

## 다음 작업 (Phase 4) 상세

### Phase 4: 인라인 스타일 → CSS — backlight + keymap

**`pages/backlight.html` 변경:**
- 인라인 스타일 → CSS 클래스 교체 (Phase 3과 동일한 패턴)

**`pages/keymap.html` 변경:**
- 인라인 스타일 → CSS 클래스 교체

---

## 기술 참고사항

### 캐시 이슈 해결
- Python http.server가 JSON/HTML 파일 캐시 → `{ cache: 'no-store' }` + `?v=${Date.now()}` 로 해결
- i18n.js의 fetch와 app.js의 페이지 템플릿 fetch 모두 적용

### SPA 라우터
- `app.js`가 `pages/{pageId}.html`을 fetch → innerHTML에 삽입 → 스크립트 별도 실행
- `I18n.apply()`는 페이지 로드 후 app.js line ~104에서 호출

### ThemeManager
- `ThemeManager.getCurrent()` 메서드 사용 (`.current` 프로퍼티 없음)
- `ThemeManager.set()`, `ThemeManager._updateSwitcherUI()` 사용
- localStorage 기반

### 로컬 개발
```bash
cd prototype
python3 -m http.server 8080
# http://localhost:8080 에서 확인
```

### 배포
- `git push origin main` → GitHub Actions `deploy.yml` → SCP to EC2 → 자동 배포 (포트 8083)
- 약 20~30초 소요

---

## 프로젝트 구조 요약

```
prototype/
├── index.html          # SPA 진입점 (사이드바)
├── css/
│   ├── tokens.css      # 디자인 토큰 (Light/Dark)
│   ├── layout.css      # 사이드바 + 디테일 패인
│   ├── components.css  # 재사용 컴포넌트
│   └── pages.css       # 페이지별 스타일
├── js/
│   ├── theme.js        # 테마 전환 (System/Light/Dark)
│   ├── i18n.js         # 번역 엔진 (KO/EN)
│   ├── app.js          # SPA 라우터
│   └── components.js   # 키보드 렌더러 등
├── pages/              # 12개 페이지 템플릿
│   ├── connect.html, he-mode.html, keymap.html
│   ├── backlight.html, macro.html, assist.html
│   ├── advance.html, firmware.html, wireless-firmware.html
│   ├── test.html, feedback.html, settings.html
├── i18n/
│   ├── ko.json         # 한국어 번역
│   └── en.json         # 영어 번역
└── assets/
    └── keyboard-k6he.svg
```
