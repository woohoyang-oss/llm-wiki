# Keychron Launcher 원본 사이트 분석

## 기본 정보
- **URL**: https://launcher.keychron.com
- **기술**: Angular 16.2.12 + WebHID API
- **버전**: Launcher V1.2.4
- **프로토콜**: VIA V3 (33-byte HID packets, usagePage 0xFF60, usage 0x61)
- **HE 확장**: Keychron HE Channel ID 0x03 (actuation, rapid trigger, SOCD, gamepad analog, calibration)
- **대상 키보드**: K6 HE (Hall Effect, 68키 레이아웃)

## 글로벌 레이아웃

### 사이드바 네비게이션
- **구조**: 고정 좌측 사이드바 + 우측 메인 콘텐츠 영역
- **섹션 구분**:
  - **Device Card**: 상단 - 연결된 키보드 이름(K6 HE), 연결 상태, 드롭다운(연결 해제)
  - **Config 섹션**: HE 모드, 키맵, 백라이트, 매크로
  - **Tools 섹션**: 빠른 시작, 어드밴스 모드, 펌웨어 업데이트, 무선 펌웨어, 키보드 테스트, 피드백
  - **Footer**: 설정, 테마 스위처(시스템/라이트/다크), 언어 스위처(KO/EN), 사용자 인증 버튼
- **라우팅**: Hash-based SPA (`#/pageId`), `pages/*.html` 파일을 fetch하여 `#page-content`에 삽입
- **인증**: Google OAuth 2.0 PKCE + 게스트 로그인, 라운드 프로필 버튼 + 팝업 메뉴

### 공통 컴포넌트
- **키보드 렌더러**: `Components.renderK6HEKeyboard()` - K6 HE 68키 SVG 키보드
- **디자인 토큰**: CSS Custom Properties 기반 (`tokens.css`)
- **i18n**: `data-i18n` 속성 기반, ko.json/en.json (12개 언어 UI 제공, 실제 번역은 ko/en만)
- **테마**: 시스템/라이트/다크 3-way 스위처

---

## 1. HE Mode (#/he-mode)

### UI 구성
- **헤더**: "HE 모드" 타이틀 + 초기화/캘리브레이션 버튼
- **메인 영역**: 2-column flex 레이아웃
  - **좌측 (flex:1)**: K6 HE 키보드 시각화 (히트맵 모드, 멀티셀렉트 가능)
  - **우측**: 프로필 사이드바 (기본 모드, 게임 모드, 게임패드)
- **클라우드 액션**: 로그인 시 "클라우드에 저장" / "클라우드에서 불러오기" 버튼 표시
- **필터 바**: 전체 선택/해제 + 선택 카운트 + 필터 칩(입력 지점 거리 / 래피드 트리거)
- **탭 바**: 6개 탭

### 서브탭 분석

#### 1.1 Actuation Distance (입력 지점 거리) - 기본 탭
- **Card 1 - 입력 지점 거리**: 슬라이더(0.1~4.0mm, step 0.1) + 숫자 입력 + "선택한 키에 적용" 버튼
- **Card 2 - 래피드 트리거**: 토글 on/off + 기본/고급 세그먼트 컨트롤 + 프레스 감도/릴리즈 감도 슬라이더(0.1~4.0mm)
- **Card 3 - 트리거 데모**: 토글 on/off + 스위치 단면도 시각화 (stem + spring + 깊이 바)
- **Card 4 - 키 테스트**: 토글 on/off + 현재 키/키 값 표시
- **레이아웃**: 4-card 그리드

#### 1.2 Multi Command (멀티 커맨드)
- **상태**: 탭 UI만 존재, 콘텐츠 미구현
- **기능**: 하나의 키에 여러 커맨드 할당

#### 1.3 Long Press (길게 누르기)
- **상태**: 탭 UI만 존재, 콘텐츠 미구현
- **기능**: 길게 눌렀을 때 다른 기능 실행

#### 1.4 SOCD
- **상태**: 탭 UI만 존재, 콘텐츠 미구현
- **프로토콜**: HE_CMD.SOCD (0x03) - 3가지 모드 (Neutral, Last Wins, First Wins)
- **기능**: 동시 반대 방향 입력 처리 (격투 게임 등에서 중요)

#### 1.5 Gamepad Analog (게임패드 아날로그)
- **상태**: 탭 UI만 존재, 콘텐츠 미구현
- **프로토콜**: HE_CMD.GAMEPAD_ANALOG (0x04)
- **기능**: 아날로그 스틱 입력 시뮬레이션

#### 1.6 Curve (커브)
- **상태**: 탭 UI만 존재, 콘텐츠 미구현
- **기능**: 입력 깊이 응답 커브 설정

### 기능 상세
- **프로파일 시스템**: 3개 프로파일 (기본/게이밍/게임패드), 각각 status dot으로 활성 상태 표시
- **키 선택**: 멀티셀렉트 지원, 전체 선택/해제, 선택 카운트 표시
- **히트맵**: 키별 actuation 값에 따른 색상 그라데이션
- **적용 애니메이션**: 값 적용 시 플래시 애니메이션
- **VIA 프로토콜**: HE_CHANNEL_ID=0x03, HE_CMD.ACTUATION_POINT=0x01, mmToRaw(mm * 10)

### 개선 기회
- **서브탭 미구현**: 6개 중 5개 탭(Multi Command, Long Press, SOCD, Gamepad Analog, Curve)의 콘텐츠가 비어있음
- **프로파일 관리**: 프로파일 이름 편집, 프로파일 복사/삭제 기능 부재
- **히트맵 범례**: 색상이 어떤 값을 의미하는지 범례가 없음
- **실시간 프리뷰**: 트리거 데모 카드가 실제 키보드 입력과 연동되지 않음
- **배치 적용**: 여러 키에 다른 값을 한 번에 설정하는 프리셋 기능 부재
- **Undo/Redo**: 변경 사항 취소/재실행 기능 없음

---

## 2. Keymap (#/keymap)

### UI 구성
- **헤더**: "키맵" 타이틀 + 초기화/내보내기/가져오기 버튼
- **레이어 선택**: 5개 레이어 세그먼트 컨트롤 (각각 색상 도트 포함)
  - Layer 0: 기본 (Base) - 파란색 #0A84FF
  - Layer 1: Fn - 주황색 #FF9F0A
  - Layer 2: 레이어 2 - 초록색 #32D74B
  - Layer 3: 레이어 3 - 보라색 #BF5AF2
  - Layer 4: 게이밍 - 빨간색 #FF453A
- **레이아웃 언어**: English (US) / Korean 세그먼트 컨트롤
- **키보드 시각화**: K6 HE 68키 레이아웃 (SVG 히어로 배경)
- **키 할당 패널**: 선택된 키 정보 + 현재 바인딩 표시 (숨김/표시 토글)
- **키 선택기**: 검색 + 6개 카테고리 탭

### 키코드 카테고리
1. **기본 (Basic)**: Esc, F1-F12, A-Z, 0-9, Tab, Enter, Space, Backspace, Delete, Caps, Shift, Ctrl, Alt, Cmd
2. **미디어 (Media)**: Play, Pause, Next, Prev, Vol+, Vol-, Mute, Bright+, Bright-
3. **매크로 (Macro)**: M0-M5 (6개 슬롯)
4. **특수 키 (Special)**: Print, Scroll, Pause, Ins, Home, End, PgUp, PgDn, N.Lock
5. **LED**: RGB+, RGB-, RGB Toggle, Mode+, Mode-, Hue+, Hue-, Speed+, Speed-
6. **사용자 지정 (Custom)**: 빈 상태 - "사용자 지정 키를 추가하세요"

### 기능 상세
- **키 선택**: 키보드에서 키 클릭 -> 할당 패널 표시 + 현재 바인딩 표시
- **키 할당**: 키 선택기에서 아이템 클릭 -> 선택된 키에 할당 + diff dot(파란색 5px 원) 추가
- **검색**: 키 선택기 내 실시간 필터링
- **리맵 표시**: 기본값과 다르게 매핑된 키에 파란색 도트 표시
- **VIA 프로토콜**: VIA_CMD.GET_KEYCODE=0x04, SET_KEYCODE=0x05, GET_LAYER_COUNT=0x11

### 개선 기회
- **시각적 피드백 부족**: 어떤 레이어가 수정되었는지 한눈에 알기 어려움
- **Drag & Drop**: 키 재배치를 드래그 앤 드롭으로 할 수 없음
- **레이어 비교**: 두 레이어를 나란히 비교하는 기능 없음
- **QMK/VIA 키코드 완전성**: 현재 키코드 목록이 매우 제한적 (실제 VIA는 수백 개)
- **Tap Dance / Mod-Tap**: 복합 키코드 설정 UI 미제공
- **JSON Import/Export**: 기능 버튼은 있지만 실제 동작 미구현

---

## 3. Backlight (#/backlight)

### UI 구성
- **헤더**: "백라이트" 타이틀
- **4개 탭**: 백라이트, Per-key RGB, Mix RGB, 인디케이터 라이트

### 3.1 Backlight 탭 (기본)
- **키보드 프리뷰**: K6 HE 키보드에 실시간 RGB 애니메이션 적용
- **2-column 레이아웃**:
  - **좌측 - 이펙트 칩 그리드**: 23개 이펙트 (0~22번)
  - **우측 - 설정 패널 (sticky)**: 현재 효과명 + 밝기 슬라이더(0~10) + 속도 슬라이더(0~255) + HSV 색상 피커

#### 23개 이펙트 목록
| # | 이름 | 색상 모드 | 애니메이션 |
|---|------|-----------|------------|
| 0 | 끄기 (Off) | none | none |
| 1 | 단색 (Solid) | 단색 | none |
| 2 | 브리딩 (Breathing) | 단색 | breathing |
| 3 | 밴드 스파이럴 (Band Spiral) | rainbow | spiral |
| 4 | 스펙트럼 사이클 (Spectrum Cycle) | rainbow | spectrum |
| 5 | 좌우 이동 (Left-Right) | rainbow | leftright |
| 6 | 상하 사이클 (Cycle Up Down) | rainbow | updown |
| 7 | 레인보우 무빙 (Rainbow Moving) | rainbow | leftright |
| 8 | 내외 사이클 (Cycle Out In) | rainbow | outin |
| 9 | 듀얼 사이클 (Dual Cycle) | rainbow | outin |
| 10 | 핀휠 (Pinwheel) | rainbow | pinwheel |
| 11 | 스타필드 (Starfield) | rainbow | twinkle |
| 12 | 듀얼 비콘 (Dual Beacon) | rainbow | beacon |
| 13 | 레인보우 비콘 (Rainbow Beacon) | rainbow | beacon |
| 14 | 젤리빈 레인드롭 (Jellybean Raindrops) | rainbow | twinkle |
| 15 | 픽셀 레인 (Pixel Rain) | rainbow | rain |
| 16 | 타이핑 히트맵 (Typing Heatmap) | 단색 | reactive |
| 17 | 디지털 레인 (Digital Rain) | 단색(#00ff44) | rain |
| 18 | 리액티브 심플 (Reactive Simple) | 단색 | reactive |
| 19 | 리액티브 멀티와이드 (Reactive Multiwide) | rainbow | reactive |
| 20 | 리액티브 멀티넥서스 (Reactive Multinexus) | rainbow | reactive |
| 21 | 스플래시 (Splash) | rainbow | splash |
| 22 | 솔리드 스플래시 (Solid Splash) | 단색 | splash |

#### 색상 피커 (HSV)
- **SatVal 박스**: 2D 그라데이션 (가로=채도, 세로=명도), 마우스 드래그
- **Hue 스트립**: 세로 무지개 바, 마우스 드래그
- **입력**: HEX 입력 + R/G/B 개별 숫자 입력 + 프리뷰 색상 사각형
- **실시간 연동**: 색상 변경 시 키보드 프리뷰 + 이펙트 칩 + 설정 도트 즉시 반영

### 3.2 Per-key RGB 탭
- **키보드**: 별도 K6 HE 키보드 인스턴스
- **토글**: Per-key RGB on/off
- **프리셋 팔레트**: 4개 (Gaming/Ocean/Sunset/Forest) - 각각 4색 도트 프리뷰
- **색상 선택**: 9색 스워치 + 커스텀 color picker
- **인터랙션**: 스워치 선택 후 키보드 키 클릭하면 해당 색상 적용

### 3.3 Mix RGB 탭
- **토글**: Mix RGB on/off
- **존 설정**: 2개 존 (Zone 1: F1~F6, Zone 2: F7~F12)
  - 각 존별 이펙트 선택 (드롭다운) + 속도 슬라이더
- **키보드 프리뷰**: 축소된(scale 0.6) 키보드 프리뷰

### 3.4 Indicator Light 탭
- **키 토글 리스트**: Caps Lock 토글 on/off + 미니 키 프리뷰
- **색상 피커**: HSV 피커 (Backlight 탭과 동일 구조) + HEX/RGB 입력
- **용도**: Caps Lock, Num Lock 등의 인디케이터 LED 색상 설정

### 개선 기회
- **이펙트 프리뷰 실 키보드 연동**: 프리뷰 애니메이션은 JS로 시뮬레이션만 제공, 실제 키보드에는 적용되지 않음
- **Mix RGB 확장**: 현재 2존만 지원, 임의 존 정의 불가
- **Per-key 일괄 적용**: 영역 선택 후 한 번에 색상 적용 기능 없음
- **이펙트 검색/필터**: 23개 이펙트를 스크롤해야 찾아야 함
- **인디케이터 키 추가**: 현재 Caps Lock만, Num Lock/Scroll Lock 등 미지원
- **색상 프리셋 저장**: 사용자가 만든 색상 조합을 저장/불러오기 불가

---

## 4. Macro (#/macro)

### UI 구성
- **녹화 배너**: 화면 상단 고정 - 녹화 중 표시 (빨간 점 + "녹화 중" + 타이머 + 라이브 키 표시)
- **키보드 시각화**: K6 HE 68키 (매크로 할당 대상 키 선택용)
- **2-column 레이아웃**:
  - **좌측 - 매크로 슬롯 패널**: 6개 슬롯 (M00~M05) + Flash 사용량 표시
  - **우측 - 매크로 에디터**: 2개 탭 (구성/코드 입력)

### 매크로 슬롯 (6개)
- **구조**: 슬롯 ID (M00~M05) + 제목 입력 + 키 수 배지 + 삭제 버튼
- **상태**: 데이터 있음(has-content) / 비어있음(empty)
- **Flash 표시**: 사용량 바 + 퍼센트 (예: Flash(2.37KB) 12%)

### 에디터 - Configuration 탭
- **녹화 컨트롤**: 녹화 시작/중지 버튼 (빨간 점 + 애니메이션)
- **이벤트 리스트**: 각 이벤트 행에 체크박스 + 번호 + 타입 아이콘 + 값 입력 + 위/아래 이동 + 더보기 메뉴
- **이벤트 타입 4가지**:
  1. **Press** (누르기): 키 이름 입력
  2. **Release** (릴리즈): 키 이름 입력
  3. **Delay** (딜레이): ms 단위 숫자 입력 (1~9999)
  4. **Text** (텍스트 삽입): 텍스트 입력
- **추가 버튼**: + 버튼 -> 컨텍스트 메뉴 (누르기/릴리즈/딜레이/텍스트 삽입)
- **행 메뉴**: 위에 삽입/아래에 삽입/삭제
- **일괄 작업**: 전체 선택/삭제

### 에디터 - Code Input 탭
- **텍스트 영역**: 매크로를 VIA 코드 형식으로 직접 편집
- **코드 형식**: `{+KC_CTRL}{+KC_C}{-KC_C}{-KC_CTRL}{W200}{+KC_CTRL}{+KC_V}{-KC_V}{-KC_CTRL}`
- **양방향 동기화**: Configuration 변경 -> Code 자동 업데이트

### 기능 상세
- **VIA 프로토콜**: VIA_CMD.MACRO_GET_COUNT=0x0C, MACRO_GET_BUFFER_SIZE=0x0D, MACRO_GET_BUFFER=0x0E, MACRO_SET_BUFFER=0x0F
- **데모 데이터**: M00="Copy & Paste" (Ctrl+C, delay 200ms, Ctrl+V), M01="Screenshot" (Cmd+Shift+4)
- **녹화 시뮬레이션**: 키 입력 시뮬레이션 (H, e, l, l, o를 300ms 간격으로)

### 개선 기회
- **실제 키보드 녹화**: 현재 시뮬레이션만, 실제 키 입력 캡처 미구현
- **매크로 루프**: 반복 실행(루프) 설정 불가
- **조건부 매크로**: if/else 로직 없음
- **매크로 테스트**: 녹화 후 즉시 테스트 기능 없음
- **코드 -> Configuration 역변환**: Code Input에서 수정한 내용이 Configuration으로 역파싱되지 않음
- **슬롯 수 제한**: 6개 고정, 사용자 확장 불가

---

## 5. Quick Start (#/assist)

### UI 구성
- **헤더**: "빠른 시작" 타이틀
- **키보드 시각화**: K6 HE 68키
- **설명 텍스트**: "Keychron Assistant는 고급 키 매핑과 시스템 수준 바로가기를 지원합니다."
- **2-card 레이아웃**:
  - **Card 1 - 상태 확인**: Keychron Assistant 앱 실행 상태 (빨간/초록 dot) + 상태 확인 버튼 + 켜기 버튼
  - **Card 2 - 다운로드**: 플랫폼별 다운로드 버튼 (macOS/Windows, 자동 감지하여 Recommended 표시)
- **설치 가이드 타임라인**: 3단계 (다운로드 -> 설치 및 실행 -> 런처로 돌아오기)

### 기능 상세
- **플랫폼 자동감지**: `navigator.platform` + `navigator.userAgent`로 macOS/Windows 판별
- **상태 확인 시뮬레이션**: 버튼 클릭 -> 스피너 2초 -> "실행 중 아님" 표시

### 개선 기회
- **실제 앱 연동**: Keychron Assistant 앱과의 WebSocket/IPC 통신 미구현
- **설치 상태 자동 감지**: 페이지 로드 시 자동 확인 없음
- **다운로드 링크 부재**: 실제 다운로드 URL 없음
- **Linux 지원**: macOS/Windows만 표시

---

## 6. Advance Mode (#/advance)

### UI 구성
- **헤더**: "어드밴스 모드" 타이틀 + 적용 버튼
- **키보드 시각화**: K6 HE 68키 (SVG 히어로 배경)
- **4-card 그리드** (2x2): 접이식(disclosure) 패턴, 한 번에 하나만 열림

### 4개 기능 카드

#### 6.1 DKS (Dynamic Keystroke)
- **설명**: 하나의 키에 최대 4개의 동작 할당
- **시각화**: Press -> Hold -> Tap -> Release 4단계 플로우
- **확장 패널**: 깊이 단계 바 차트 (1~4단계, 각 높이 30%/60%/90%/100%) + Configure Key/Reset 버튼

#### 6.2 Toggle Key (토글 키)
- **설명**: 키를 누를 때마다 두 기능 간 전환
- **시각화**: A <-> B 토글 화살표
- **확장 패널**: State A / State B 각각 "Click to assign..." + Assign/Reset 버튼

#### 6.3 Tap/Hold (탭/홀드)
- **설명**: 짧게 누르기와 길게 누르기에 다른 기능 할당
- **시각화**: Tap=Esc, Hold=Ctrl 예시
- **확장 패널**: Hold Threshold 슬라이더(100~500ms, 기본 200ms) + Tap Action/Hold Action 표시 + Apply/Reset

#### 6.4 Combo Key (콤보 키)
- **설명**: 하나의 키로 여러 키 조합 동시 입력
- **시각화**: Ctrl + Shift + V 예시
- **확장 패널**: Key Combination 입력 영역 (태그 형태) + "Press keys to add..." + Save Combo/Clear

### 개선 기회
- **키 선택 연동**: 키보드에서 키 선택 -> 해당 키에 기능 할당하는 플로우가 미구현
- **기능 적용**: 모든 확장 패널의 버튼이 시뮬레이션만
- **DKS 깊이 설정**: 각 단계별 키코드 할당 UI 없음
- **프리셋**: 자주 사용하는 조합 프리셋 없음
- **시각적 피드백**: 어떤 키에 어떤 고급 기능이 할당되었는지 키보드 위에서 확인 불가

---

## 7. Firmware Update (#/firmware)

### UI 구성
- **헤더**: "펌웨어 업데이트" 타이틀
- **히어로 카드**:
  - **디바이스 아이콘**: 키보드 SVG
  - **디바이스 정보**: K6 HE / Hall Effect Keyboard
  - **버전 표시**: 현재 버전(V1.0.8) + 최신 버전(V1.0.8)
  - **상태**: 최신 상태 체크마크 또는 업데이트 가능 배너
- **업데이트 배너** (숨김): "NEW V1.0.9 Available" + Install Update 버튼
- **프로그레스 바** (숨김): Downloading.../Installing... + 퍼센트
- **업데이트 확인 버튼**: 새로고침 아이콘 + "업데이트 확인"
- **경고**: "업데이트 중 키보드를 분리하지 마세요."
- **변경 사항 (Changelog)**: 접이식 카드 (V1.0.8 Current, V1.0.7, V1.0.6)

### 기능 상세
- **업데이트 확인 시뮬레이션**: 버튼 -> 스피너 2.5초 -> V1.0.9 업데이트 가능 표시
- **설치 시뮬레이션**: Install -> 프로그레스 바 (랜덤 증가) -> Complete -> V1.0.9로 업데이트

### 개선 기회
- **실제 펌웨어 다운로드**: 파일 다운로드/적용 미구현
- **DFU 모드**: 부트로더 진입(VIA_CMD.BOOTLOADER_JUMP) 미구현
- **릴리즈 노트 상세**: 변경 사항이 매우 간략함
- **롤백 기능**: 이전 버전으로 되돌리기 불가
- **자동 업데이트 확인**: 페이지 접근 시 자동 확인 없음

---

## 8. Wireless Firmware (#/wireless-firmware)

### UI 구성
- **헤더**: "무선 펌웨어" 타이틀
- **동기화 상태 시각화**: Keyboard <-> Synced <-> Dongle (화살표 + Synced 텍스트)
- **2-card 그리드**:
  - **Bluetooth 펌웨어 카드**: BT 아이콘 + 신호 강도(Strong, 4-bar) + 현재/최신 버전(V2.1.3) + "최신 상태" + 업데이트 버튼
  - **2.4G 동글 펌웨어 카드**: WiFi 아이콘 + 신호 강도(Medium, 3-bar) + 현재 V1.3.0/최신 V1.3.1 + "업데이트 가능" 배지 + 업데이트 버튼 + 프로그레스 바

### 기능 상세
- **동글 업데이트 시뮬레이션**: 버튼 -> 프로그레스 바 -> Complete -> 상태 업데이트
- **신호 강도 시각화**: SVG 막대 4개 (높이별 차등, 색상: success/warning)

### 개선 기회
- **실제 무선 모듈 통신**: BLE/2.4G 모듈과의 실제 통신 미구현
- **연결 상태 실시간 표시**: 현재 정적 표시
- **페어링 관리**: BT 페어링 기기 목록/해제 기능 없음
- **배터리 상태**: 무선 키보드 배터리 잔량 표시 없음

---

## 9. Key Test (#/test)

### UI 구성
- **헤더**: "키보드 테스트" 타이틀 + 매트릭스 테스트 토글 + 초기화 버튼
- **진행률**: 테스트된 키 수 / 68 + 퍼센트 + 프로그레스 바
- **키보드 시각화**: K6 HE 68키 (키 누를 때 하이라이트)
- **음향 효과 섹션**: 토글 on/off + 14개 사운드 카드 그리드
  - Default, Balloon, CarHorn, Cartoon, Coin Collecting, Dog Barking, Donkey, Glass, Phone Dial, Party Horn, Punch, Rubber Toy, Sneeze, Rubber Chicken
- **키 입력 기록**: 타임스탬프 + 키 이름 칩 목록 (스크롤, 최대 160px 높이)

### 기능 상세
- **키 추적**: `Set`으로 `e.code` 기반 중복 제거
- **실시간 하이라이트**: keydown -> 가상 키보드에서 해당 키 배경색 변경 (0.3초 후 복원)
- **100% 완료**: 프로그레스 바 색상이 accent -> success로 변경

### 개선 기회
- **매트릭스 테스트 미구현**: 토글 UI만 있고 실제 매트릭스(VIA switch_matrix_state) 테스트 없음
- **키 누름 깊이**: HE 키보드의 아날로그 깊이 값 표시 없음
- **사운드 재생**: 사운드 카드 선택은 되지만 실제 오디오 재생 미구현
- **결과 저장/공유**: 테스트 결과를 이미지/PDF로 내보내기 불가
- **불량 키 마킹**: 응답하지 않는 키를 별도 표시하는 기능 없음

---

## 10. Feedback (#/feedback)

### UI 구성
- **헤더**: "피드백" 타이틀
- **카테고리 선택**: 4개 칩 (버그 리포트, 기능 요청, 개선 제안, 기타)
- **이메일**: 선택적 입력
- **메시지**: 텍스트 영역 + 글자 수 카운트 (0/2000) + 카테고리별 placeholder 변경
- **첨부파일**: 드래그 앤 드롭 영역 (PNG, JPG 최대 5MB)
- **제출 버튼**: "보내기"
- **디바이스 정보 자동 첨부**: "K6 HE / FW V1.0.8 / Launcher V1.2.4"
- **토스트 알림**: 성공 시 하단 토스트 (3초 후 자동 닫힘)

### 기능 상세
- **카테고리 연동 placeholder**: 카테고리 선택에 따라 placeholder 텍스트 변경
- **글자 수 경고**: 1800자 이상 warning 색상, 2000자 이상 error 색상

### 개선 기회
- **실제 전송**: 백엔드 API 연동 미구현 (feedback 라우트 미완성)
- **파일 업로드**: 드래그 앤 드롭 UI만, 실제 업로드 미구현
- **피드백 히스토리**: 이전 제출 피드백 확인 불가
- **스크린샷 캡처**: 런처 화면 직접 캡처 기능 없음

---

## 11. Settings (#/setting)

### UI 구성
- **헤더**: "설정" 타이틀
- **5개 탭**: 언어, 장치 정보, 실행 로그, 버전 기록, 계정

### 11.1 Language 탭 (기본)
- **테마 설정**: 시스템/라이트/다크 3개 카드 (SVG 아이콘 + 라벨)
- **언어 선택**: 12개 언어 카드 그리드 (국기 이모지 + 언어명)
  - English, 한국어, 简体中文, Deutsch, 日本語, Bahasa, ภาษาไทย, Polski, Italiano, Francais, Espanol, Portugues

### 11.2 Device Info 탭
- **디바이스 카드**: 키보드 미니 일러스트(SVG) + 상세 정보
  - 모델: K6 HE
  - 펌웨어: V1.0.8
  - 연결: USB (badge)
  - 런처 버전: V1.2.4

### 11.3 Activity Log 탭
- **로그 액션**: 로그 복사 / 지우기 버튼
- **로그 내용**: 타임스탬프 + 이벤트 형식
  - 예: `[2026-02-13 10:23:45] Device connected: K6 HE`
- **클립보드 복사**: `navigator.clipboard.writeText()` + 토스트 알림

### 11.4 Version Record 탭
- **버전 카드 목록**: V1.2.4 (Current), V1.2.3, V1.2.2
  - 각 카드: 버전 번호 + 날짜/Current 배지 + 변경 설명

### 11.5 Account 탭
- **로그인 상태**: 아바타 + 이름 + 이메일 + Google 로그인 표시 + 저장된 파일 수 + 로그아웃 버튼
- **로그아웃 상태**: 클라우드 아이콘 + 로그인 유도 메시지 + Google 로그인/게스트 로그인 버튼

### 개선 기회
- **번역 완성도**: 12개 언어 UI 제공하지만 실제 ko/en만 번역 파일 존재
- **로그 필터/검색**: 로그 양이 많아지면 필터링 필요
- **디바이스 진단**: 키보드 상태 진단(키 응답 시간, 채터링 등) 기능 없음
- **설정 내보내기/가져오기**: 전체 설정 백업/복원 없음
- **디스플레이 설정**: 키보드 시각화 크기, 색상 테마 등 세부 설정 없음

---

## 전체 기능 갭 매트릭스

### 구현 상태 요약

| 페이지 | 기능 | UI | WebHID 통신 | 백엔드 API | 상태 |
|--------|------|-----|------------|-----------|------|
| Connect | WebHID 연결 | Done | Partial | N/A | Partial |
| HE Mode - Actuation | 입력 지점 설정 | Done | Not Connected | N/A | UI Only |
| HE Mode - Rapid Trigger | 래피드 트리거 | Done | Not Connected | N/A | UI Only |
| HE Mode - Multi Command | 멀티 커맨드 | Tab Only | Missing | N/A | Missing |
| HE Mode - Long Press | 길게 누르기 | Tab Only | Missing | N/A | Missing |
| HE Mode - SOCD | SOCD | Tab Only | Missing | N/A | Missing |
| HE Mode - Gamepad Analog | 게임패드 아날로그 | Tab Only | Missing | N/A | Missing |
| HE Mode - Curve | 커브 | Tab Only | Missing | N/A | Missing |
| Keymap | 키 매핑 | Done | Not Connected | N/A | UI Only |
| Backlight - Effects | RGB 이펙트 | Done (23개) | Not Connected | N/A | UI Only |
| Backlight - Per-key | 개별 키 색상 | Done | Not Connected | N/A | UI Only |
| Backlight - Mix RGB | 존별 이펙트 | Done | Not Connected | N/A | UI Only |
| Backlight - Indicator | 인디케이터 LED | Done | Not Connected | N/A | UI Only |
| Macro | 매크로 에디터 | Done | Not Connected | N/A | UI Only |
| Quick Start | Keychron Assistant | Done | N/A | N/A | UI Only |
| Advance - DKS | Dynamic Keystroke | Done | Not Connected | N/A | UI Only |
| Advance - Toggle Key | 토글 키 | Done | Not Connected | N/A | UI Only |
| Advance - Tap/Hold | 탭/홀드 | Done | Not Connected | N/A | UI Only |
| Advance - Combo Key | 콤보 키 | Done | Not Connected | N/A | UI Only |
| Firmware | 펌웨어 업데이트 | Done | Not Connected | N/A | Simulated |
| Wireless Firmware | 무선 펌웨어 | Done | Not Connected | N/A | Simulated |
| Key Test | 키보드 테스트 | Done | Partial (keydown) | N/A | Partial |
| Feedback | 피드백 제출 | Done | N/A | Missing | UI Only |
| Settings | 설정 관리 | Done | N/A | Partial | Partial |
| Cloud Profiles | 클라우드 저장 | Done | N/A | Done | Working |
| Auth | Google OAuth | Done | N/A | Done | Working |

### 원본 사이트 대비 주요 차이점

1. **WebHID 실제 통신**: 프로토콜 상수/헬퍼는 정의되어 있지만 (`protocol.ts`, `he-protocol.ts`, `keyboard-api.js`) 실제 키보드와의 HID 패킷 교환이 연결되어 있지 않음
2. **키보드 모델 하드코딩**: K6 HE 68키만 하드코딩, 동적 키보드 정의 로딩 필요 (`definition-parser.ts` 존재하지만 미연동)
3. **HE 서브기능 5개 미구현**: Multi Command, Long Press, SOCD, Gamepad Analog, Curve
4. **매트릭스 테스트**: VIA의 `SWITCH_MATRIX_STATE` 읽기 미구현
5. **펌웨어 업데이트**: DFU 모드 진입 + 바이너리 전송 미구현

### 원본 UX 문제점 종합

1. **정보 과부하**: HE Mode의 6개 탭 + 4개 카드가 한 화면에 너무 많은 정보 제공
2. **일관성 부족**: 페이지마다 키보드 시각화의 상호작용 방식이 다름 (멀티셀렉트 vs 단일 셀렉트 vs 읽기전용)
3. **피드백 부재**: 설정 변경 후 실제 키보드에 적용되었는지 확인 수단 없음
4. **온보딩 없음**: 처음 사용하는 사용자를 위한 투어/가이드 없음
5. **접근성**: 키보드 네비게이션, 스크린 리더 지원 미흡
6. **반응형**: 고정 레이아웃, 모바일/태블릿 미지원
7. **에러 처리**: 연결 실패, 통신 오류 시 사용자 피드백 불충분
8. **Undo/Redo**: 모든 설정 페이지에서 변경 취소 기능 없음
9. **비교 기능**: 설정 변경 전후 비교, 레이어간 비교 불가
10. **프리셋/템플릿**: 게임별 프리셋, 커뮤니티 프로파일 공유 기능 부재
