# Email Support Bot — 업데이트 히스토리

## v2.4.0 — V3 개선안 구현 (2026-03-14 ~ 03-18)

### Phase 1: 프로액티브 알림 인프라
- `ChatbotNotification` 모델 추가 (chatbot_notifications 테이블)
- Celery beat 2분 주기 `generate_proactive_notifications` 태스크
- 승인 대기/미배정 이메일/SLA 위반 자동 감지
- 에이전트별 알림 중복 제거 (10분 윈도우)

### Phase 2: 챗봇 배지 + 프로액티브 메시지
- 30초 폴링으로 읽지 않은 알림 수 표시 (숫자 배지)
- 챗봇 열 때 미읽음 알림 자동 로드 + 자동 읽음 처리
- 우선순위별 스타일링 (amber: normal, red: high/urgent)
- 알림 API admin scope 수정 — admin은 모든 알림 조회 가능

### Phase 3: 셋업 위자드
- `GET /setup/status` — 5개 스텝 실시간 DB 상태 체크
  - 메일함 설정, AI 엔진, 자동 응답 규칙, 텔레그램 연동, 테스트 이메일
- `POST /setup/complete` — 셋업 완료 처리
- `agents.has_completed_setup` 컬럼 추가
- 챗봇에 체크리스트 UI + "나중에 하기" 버튼

### Phase 4: 텔레그램 풀 워크플로우
- 텔레그램 인라인 버튼 (승인/거부/상세보기)
- 텔레그램 명령어 (/status, /pending 등)
- 알림 포맷 통일 (`NotificationService.format_approval_message()`)

### Phase 5: AI 피드백 학습
- `GET /ai/feedback-stats` — 승인/거부 패턴 분석, 룰별 통계
- `POST /ai/update-thresholds` — confidence 임계값 실시간 조정
- Pipeline 임계값을 mutable dict로 변경 (`_confidence_thresholds`)

### Phase 6: 챗봇 커맨드 센터
- 자연어 명령 감지 + 즉시 실행 (AI 호출 없이 DB 직접 조회)
- 명령어: 현황, 승인 대기, 최근 이메일, 오늘 통계, 도움말
- 사용자 관리: 등록, 수정, 목록 조회 (admin only)
- 퀵 커맨드 칩 버튼 (첫 화면에 표시)

### 기타 개선
- 이메일 상세 페이지 액션 버튼 4개 추가 (보관/별표/우선순위/삭제)
- 텔레그램 HTML 파싱 에러 수정 (이메일 주소 이스케이프)
- 데이터베이스 전체 덤프 문서 (`docs/database-dump.md`)

---

## v2.3.0 — Gmail 스타일 웹메일 고도화 (2026-03-12 ~ 03-13)

### 주요 기능
- AI 초안 발송 시 수신자/참조/숨은참조 편집 가능
- 참조에 수신 메일함 디폴트 추가
- Docker에 Claude CLI 설치 + 인증 토큰 볼륨 공유
- 내 메일함에도 회신/전체회신/전달 + 룰 만들기 버튼 추가
- Gmail 스타일 메일쓰기 compose 창 구현

### 버그 수정
- 사이드바 숫자 포맷 (천단위 콤마)
- AI 어시스턴트 채팅 사용자 변경 시 리셋
- APP_VERSION 클라이언트 에러 해결
- 사이드바 클라이언트 에러 수정

---

## v2.2.0 — Gmail-like 웹메일 기본 (2026-03-10 ~ 03-11)

### 주요 기능
- 사이드바 이모지 + 홈링크 + 라벨 미읽은 수 뱃지
- 메일쓰기 버튼 동작 구현
- Gmail 스타일 UI/UX 적용
- 이메일 상세 페이지 전면 개편

---

## v2.1.0 — 멀티 브랜드 + 개인 워크스페이스 (2026-03-08 ~ 03-09)

### 주요 기능
- 멀티 브랜드 메일함 (keychron, gtgear, aiper, tbnws)
- 개인 워크스페이스 (내 메일함, 개인 설정)
- RBAC 권한 체계 (admin/agent 역할)
- ScopeMixin 기반 공유/개인 데이터 분리

---

## v2.0.0 — 기본 자동화 파이프라인 (2026-03-05 ~ 03-07)

### 핵심 아키텍처
- Gmail API → 이메일 수신 → 룰 매칭 → AI 처리 → 승인 큐 → 발송
- AIRouter: 룰별 engine_id → 해당 엔진 디스패치
- ClaudeCLIProcessor: Claude API + MCP tool_use
- LocalLLMProcessor: Ollama/OpenAI compat
- Confidence 라우팅: ≥0.85 자동발송, 0.70~0.84 승인큐, <0.70 에스컬레이션
- Celery 비동기 태스크 (이메일 폴링, AI 처리)
- 텔레그램 알림 연동
