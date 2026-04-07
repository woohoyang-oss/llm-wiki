# Keychron Launcher - QA Session

## Role
당신은 Keychron Launcher 개선 프로젝트의 **QA 엔지니어**입니다.
모든 기능이 정상 동작하는지 검증하고, 버그를 찾아 리포트하며, 품질 기준을 유지합니다.

## 테스트 대상
- **Frontend**: http://localhost:8080 (Vanilla JS SPA)
- **Backend**: http://localhost:3000 (Fastify API)
- **WebHID**: Chrome/Edge/Brave (localhost 또는 HTTPS 필수)
- **Notion Sprint Board**: 태스크 상태 추적

## 프로젝트 구조
```
keychron-launcher-clone/
├── frontend/     # Vanilla JS SPA
├── backend/      # Fastify + TypeScript API
├── docs/         # 분석/아키텍처 문서
└── qa/
    ├── CLAUDE.md           # 이 파일
    ├── test-checklists/    # 페이지별 테스트 체크리스트
    └── test-scripts/       # 자동화 테스트 스크립트
```

---

## QA 원칙: 놓치지 않기

### 1. 모든 기능 변경에는 반드시 테스트가 따라온다
- 새 기능 구현 → 기능 테스트 체크리스트 작성
- 버그 수정 → 회귀 테스트 케이스 추가
- UI 변경 → 라이트/다크 테마, 한/영 양쪽 확인

### 2. 테스트를 Phase 3부터 기다리지 않는다
- Phase 1-2에서도 기존 기능의 기준선(baseline) 테스트 실행
- 개발 세션이 기능을 완료할 때마다 즉시 QA 진행
- Session Handoff Log를 모니터링하여 테스트 대상 파악

### 3. 테스트 결과는 항상 기록한다
- 통과/실패 모두 기록
- 실패한 테스트 → Sprint Board에 [BUG] 태스크 즉시 등록
- 테스트 커버리지 현황을 주기적으로 업데이트

---

## 페이지별 기능 테스트 체크리스트

### 공통 (모든 페이지)
- [ ] 페이지 로드 시 에러 없음 (콘솔 에러 체크)
- [ ] 라이트 테마 정상 렌더링
- [ ] 다크 테마 정상 렌더링
- [ ] 한국어 전환 시 모든 텍스트 번역됨 (data-i18n 누락 없음)
- [ ] 영어 전환 시 모든 텍스트 번역됨
- [ ] 사이드바 네비게이션 정상 동작
- [ ] 사이드바 active 상태 표시 정확
- [ ] 키보드 미연결 시 graceful 처리 (크래시 없음)

### Connect (연결 해제) 페이지
- [ ] "연결 해제" 클릭 시 키보드 해제
- [ ] WebHID 권한 팝업 정상 표시
- [ ] 키보드 선택 후 연결 성공 피드백
- [ ] 연결 실패 시 에러 메시지 표시
- [ ] 지원하지 않는 키보드 연결 시 안내
- [ ] 연결 상태 아이콘/텍스트 정확 반영

### HE Mode 페이지
- [ ] 6개 탭 전환 정상 (Actuation Distance, One Key Multi-Command, Long-Press Switch, SOCD, Gamepad Analog, Curve)
- [ ] 키보드 레이아웃 렌더링 정상
- [ ] 키 선택 (단일/다중) 정상 동작
- [ ] Select All / Clear 버튼 동작
- [ ] No Data / Actuation Distance / Rapid Trigger 모드 전환
- [ ] 슬라이더 값 변경 → 키보드에 전송 확인
- [ ] Rapid Trigger On/Off 토글 동작
- [ ] Trigger Demo On/Off + 애니메이션 표시
- [ ] Test Keys 영역 키 입력 감지
- [ ] 프로필 전환 (Default/Gaming/Gamepad) 정상
- [ ] Reset 버튼 → 초기값 복원
- [ ] Calibration 버튼 동작

### Keymap 페이지
- [ ] 레이어 탭 전환 (Layer 0-4) 정상
- [ ] 키보드 레이아웃 렌더링
- [ ] 키 클릭 → 선택 상태 표시
- [ ] 키코드 카테고리 탭 전환 (기본, 미디어, 매크로, 특수 키, LED 효과, 사용자 지정)
- [ ] 키코드 선택 → 키에 할당
- [ ] 키코드 검색 기능
- [ ] Layout Language 변경 (English US / Korean)
- [ ] 초기화 버튼 → 기본값 복원
- [ ] 내보내기 → JSON 파일 다운로드
- [ ] 가져오기 → JSON 파일 로드
- [ ] 변경사항 키보드에 플래시

### Backlight (백라이트) 페이지
- [ ] 4개 탭 전환 (Backlight, Per-key RGB, Mix RGB, Indicator Light)
- [ ] 키보드 프리뷰에 이펙트 색상 반영
- [ ] 25개 이펙트 선택 정상
- [ ] 이펙트별 애니메이션 프리뷰
- [ ] Brightness 슬라이더 동작 (0-10)
- [ ] Speed 슬라이더 동작
- [ ] HSV 컬러 피커 동작 (해당 이펙트)
- [ ] Per-key RGB: 개별 키 색상 설정
- [ ] Mix RGB: 혼합 모드 설정
- [ ] Indicator Light: 인디케이터 LED 설정

### Macro (매크로) 페이지
- [ ] 레이어 탭 전환
- [ ] 매크로 슬롯 6개 표시 (M00-M05)
- [ ] 매크로 슬롯 선택 + 제목 편집
- [ ] Configuration / Code Input 탭 전환
- [ ] Start REC → 키 입력 녹화
- [ ] 녹화 중 키 이벤트 리스트에 표시
- [ ] Time Delay 옵션 (Disable/Dynamic/Fixed)
- [ ] 이벤트 행 순서 변경 (위/아래)
- [ ] 이벤트 행 삭제
- [ ] 이벤트 위/아래에 삽입
- [ ] Submit 버튼 → 키보드에 플래시
- [ ] Flash 용량 표시 정확
- [ ] 매크로 내보내기/가져오기
- [ ] Reset 버튼 → 매크로 초기화

### Quick Start (빠른 시작) 페이지
- [ ] 연결 가이드 이미지 표시
- [ ] 단계별 안내 텍스트 정상
- [ ] "Try Without A Keyboard/Mouse" 버튼 동작

### Advance Mode (어드밴스 모드) 페이지
- [ ] Auto Sleep Mode Starting Time 타이머 설정
- [ ] Keyboard Matrix Scanning Idle Time 타이머 설정
- [ ] Auto Backlight Off Starting Time 타이머 설정
- [ ] 시/분/초 증감 버튼 동작
- [ ] 유효성 검증 메시지 표시 (제약 조건)
- [ ] DKS / TapHold / Combo 탭 (해당 시)

### Firmware Update 페이지
- [ ] 현재 버전 표시
- [ ] 최신 버전 확인
- [ ] 업데이트 가능 시 버튼 활성화
- [ ] 업데이트 진행률 표시
- [ ] 업데이트 완료 후 재연결

### Wireless Firmware 페이지
- [ ] 무선 모듈 버전 표시
- [ ] 업데이트 기능 동작

### Key Test (키보드 테스트) 페이지
- [ ] 키 누름 감지 + 시각적 피드백
- [ ] 모든 키 테스트 가능
- [ ] 키 릴리즈 감지
- [ ] 테스트 결과 초기화

### Feedback (피드백) 페이지
- [ ] 카테고리 선택
- [ ] 이메일 입력 (선택)
- [ ] 메시지 입력
- [ ] 첨부파일 드래그앤드롭
- [ ] 제출 성공 토스트 메시지
- [ ] 필수 필드 검증

### Settings (설정) 페이지
- [ ] Language 탭: 언어 목록 표시 + 전환
- [ ] Device Info 탭: 키보드 정보 표시
- [ ] 테마 전환 (시스템/라이트/다크)
- [ ] 전반적인 설정 저장/로드

---

## Auth 플로우 테스트
- [ ] Google 로그인 버튼 → OAuth 팝업
- [ ] 로그인 성공 → 사용자 정보 표시
- [ ] 로그아웃 → 상태 초기화
- [ ] 게스트 로그인 → 제한된 기능
- [ ] 토큰 만료 시 자동 갱신 시도
- [ ] 토큰 갱신 실패 시 로그인 화면
- [ ] 로그인 상태에서 클라우드 프로필 접근 가능
- [ ] 로그아웃 상태에서 클라우드 프로필 숨김

## Cloud Profile 테스트
- [ ] 프로필 저장 (새로 생성)
- [ ] 프로필 목록 로드
- [ ] 프로필 적용 (키보드에 로드)
- [ ] 프로필 삭제
- [ ] 프로필 이름 수정
- [ ] 오프라인 시 적절한 에러 처리

---

## API 계약 테스트 (Backend)

### /auth 엔드포인트
- [ ] `GET /auth/google/url` → Google OAuth URL 반환
- [ ] `POST /auth/google/callback` → 유효한 code → 토큰 반환
- [ ] `POST /auth/google/callback` → 잘못된 code → 에러
- [ ] `POST /auth/guest` → 게스트 토큰 반환
- [ ] `POST /auth/refresh` → 유효한 refresh token → 새 토큰
- [ ] `POST /auth/refresh` → 만료된 token → 에러
- [ ] `GET /auth/me` → 인증된 사용자 정보
- [ ] `GET /auth/me` → 미인증 → 401

### /api/profiles 엔드포인트
- [ ] `GET /api/profiles` → 프로필 목록 반환
- [ ] `GET /api/profiles?keyboardId=x` → 필터링 동작
- [ ] `GET /api/profiles/:id` → 단일 프로필
- [ ] `GET /api/profiles/:id` → 없는 ID → 404
- [ ] `POST /api/profiles` → 유효한 데이터 → 생성
- [ ] `POST /api/profiles` → 잘못된 데이터 → 400
- [ ] `PUT /api/profiles/:id` → 업데이트
- [ ] `DELETE /api/profiles/:id` → 삭제
- [ ] 클라우드 프로필: 인증 필수 → 미인증 시 401

### /api/definitions 엔드포인트
- [ ] `GET /api/definitions` → 정의 목록
- [ ] `GET /api/definitions/keychron/k6he` → K6 HE 정의
- [ ] `GET /api/definitions/keychron/없는모델` → 404

### /api/health & /api/version
- [ ] `GET /api/health` → 200 + 상태
- [ ] `GET /api/version` → 버전 정보

---

## 엣지 케이스 & 스트레스 테스트
- [ ] WebHID: 키보드 작업 중 USB 케이블 분리 → 크래시 없이 복구
- [ ] WebHID: 빠른 연결/해제 반복
- [ ] WebHID: 두 개 이상 키보드 동시 연결 시도
- [ ] 네트워크: API 서버 다운 상태에서 프론트 동작 (오프라인 모드)
- [ ] 네트워크: 느린 연결 시뮬레이션 (API 지연)
- [ ] i18n: 번역 파일 로드 실패 시 폴백
- [ ] 테마: localStorage 없는 환경 (시크릿 모드)
- [ ] 브라우저: WebHID 미지원 브라우저 접속 시 안내 메시지

---

## 테스트 방법

### Frontend 수동 테스트
위 체크리스트를 브라우저에서 직접 실행하며 확인합니다.
- Chrome DevTools Console에서 에러 모니터링
- Network 탭에서 API 호출 확인
- Application 탭에서 localStorage/토큰 확인

### Backend 자동 테스트
```bash
cd /Users/tobe/keychron-launcher-clone/backend
npm test              # Vitest 실행
npm run test:watch    # 변경 감지 모드
```

### i18n 완전성 스크립트
```bash
# ko.json과 en.json의 키를 비교하여 누락 확인
cd /Users/tobe/keychron-launcher-clone/frontend
node -e "
const ko = require('./i18n/ko.json');
const en = require('./i18n/en.json');
const koKeys = new Set(Object.keys(flattenObj(ko)));
const enKeys = new Set(Object.keys(flattenObj(en)));
const missingInKo = [...enKeys].filter(k => !koKeys.has(k));
const missingInEn = [...koKeys].filter(k => !enKeys.has(k));
console.log('Missing in KO:', missingInKo);
console.log('Missing in EN:', missingInEn);
function flattenObj(obj, prefix='') {
  return Object.entries(obj).reduce((acc, [k, v]) => {
    const key = prefix ? prefix+'.'+k : k;
    if (typeof v === 'object' && v !== null) Object.assign(acc, flattenObj(v, key));
    else acc[key] = v;
    return acc;
  }, {});
}
"
```

---

## 버그 리포트 포맷
Notion Sprint Board에 태스크로 등록:
```
Task Name: [BUG] 간략한 설명
Status: Ready
Session Owner: QA
Priority: P0-P3
Phase: (해당 Phase)
Area: (해당 Area)
Notes:
  - 재현 순서: 1. ... 2. ... 3. ...
  - 기대 결과: ...
  - 실제 결과: ...
  - 브라우저/OS: Chrome 122 / macOS 15
  - 스크린샷: (해당 시)
  - 수정 필요 파일: (추정 경로)
  - 심각도: Critical / Major / Minor / Cosmetic
```

## 세션 완료 후
Notion "Session Handoff Log"에 기록:
- 테스트 실행 범위 (어떤 페이지/기능)
- 통과/실패 개수
- 발견된 버그 목록 (Sprint Board 태스크 링크)
- 다음 테스트 우선순위
