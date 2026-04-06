# 챗봇 코드 분석 결과 (2026-03-24)

## 1. 프론트엔드 구조

### 컴포넌트: `admin/src/components/ai-chatbot.tsx` (957줄)
- 우측 하단 플로팅 버튼 (Ctrl+/ 토글)
- 세션 관리 (CRUD): /admin/chat/sessions
- 알림 통합: /admin/notifications (unread-count, read-all)
- 초기 설정 마법사: /admin/setup/status
- Quick Commands: 현황, 승인대기, 최근이메일, 지식목록, 레포지토리, 도움말
- 현재 이메일 ID 자동 전달 (`selected-email-store`)
- 페이지 컨텍스트 자동 전달 (`pathname`)

### 호출 API
- POST /admin/chat — 메시지 전송 (messages, context, session_id, email_id)
- GET/POST/DELETE /admin/chat/sessions — 세션 CRUD
- GET /admin/notifications?unread_only=true — 알림
- POST /admin/notifications/read-all — 알림 읽음
- GET /admin/setup/status — 초기 설정 상태
- POST /admin/setup/complete — 설정 건너뛰기

## 2. 백엔드 챗봇 엔드포인트

### POST /admin/chat (admin.py:8464)
- 시스템 프롬프트 기반 Claude AI 대화
- [action] 태그 파싱 → `_execute_chatbot_action()` 자동 실행
- 컨텍스트: 담당자 현황, 텔레그램 상태, 설정 등 자동 주입

## 3. 챗봇 실행 가능 액션 (36종)

### 담당자 관리 (admin only)
- assign_responsibility — 담당 배정
- remove_responsibility — 담당 해제

### 텔레그램 연동
- telegram_set_bot — 봇 토큰 등록
- telegram_status — 연동 상태 확인
- telegram_generate_link — 연동 링크 생성
- telegram_settings — 알림 설정
- telegram_disconnect — 연동 해제
- send_telegram_test — 테스트 메시지

### 노션 연동
- notion_set_token — 토큰 등록
- notion_test — 연결 테스트
- notion_list_databases — DB 목록
- notion_set_default_db — 기본 DB 설정

### 개인 설정
- get_personal_settings — 설정 조회
- update_personal_settings — 설정 변경

### 알림 채널
- list_notification_channels — 채널 목록
- create_notification_channel — 채널 생성 (admin)
- delete_notification_channel — 채널 삭제 (admin)

### 지식 베이스 (READ + CREATE)
- list_repositories — 레포지토리 목록
- list_knowledge_docs — 지식문서 목록
- create_knowledge_doc — 지식문서 생성
- list_templates — 템플릿 목록
- create_template — 템플릿 생성
- list_macros — 매크로 목록
- create_macro — 매크로 생성

### 규칙/정책 (READ + TOGGLE)
- list_rules — 규칙 목록
- toggle_rule — 규칙 활성/비활성
- create_rule — 규칙 생성
- list_assignment_rules — 배정규칙 목록
- list_sla_policies — SLA 정책 목록
- list_business_hours — 영업시간 목록
- list_schedules — 스케줄 목록
- toggle_schedule — 스케줄 활성/비활성

### 기타
- list_signatures — 서명 목록
- list_disclaimers — 면책조항 목록
- list_mailboxes — 메일함 목록
- search_emails — 이메일 검색
- list_agents — 담당자 목록
- get_email_detail — 이메일 상세 조회

## 4. 챗봇에서 불가능한 CRUD 작업

| 엔티티 | LIST | CREATE | UPDATE | DELETE |
|--------|------|--------|--------|--------|
| 지식문서 | O | O | X | X |
| 템플릿 | O | O | X | X |
| 매크로 | O | O | X | X |
| 규칙 | O | O | toggle만 | X |
| 배정규칙 | O | X | X | X |
| SLA 정책 | O | X | X | X |
| 영업시간 | O | X | X | X |
| 스케줄 | O | X | toggle만 | X |
| 서명 | O | X | X | X |
| 면책조항 | O | X | X | X |
| 메일함 | O | X | X | X |
| 에이전트 | O | X | X | X |
| 태그 | X | X | X | X |
| 자동화 | X | X | X | X |

## 5. 주요 발견사항
- 챗봇은 **조회(LIST)** 는 대부분 가능하나, **수정/삭제**는 거의 불가
- **태그, 자동화(automations)** 는 챗봇에서 아예 접근 불가
- 챗봇은 Claude API를 통한 자연어 대화 + [action] 태그 기반 실행 구조
- 세션 영속성 있음 (DB 저장, 세션 전환 가능)
- 알림 시스템과 통합되어 있음 (unread count polling 30초)
