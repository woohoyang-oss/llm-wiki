# Email Support Bot — 데이터베이스 현황

> 추출일: 2026-03-17
> DB: PostgreSQL 16 (esbot-db)
> 테이블 수: 35개

---

## 1. 담당자 (agents) — 12명

| ID | 이름 | 이메일 | 역할 | 활성 | 등록일 |
|----|------|--------|------|------|--------|
| 1 | 우호 | wooho.yang@tbnws.com | admin | O | 2026-03-13 |
| 2 | 양우호 | yangwooho@tbnws.com | admin | O | 2026-03-13 |
| 4 | 김기호 | kiho.kim@tbnws.com | agent | O | 2026-03-14 |
| 5 | 문소현 | sohyun.moon@tbnws.com | agent | O | 2026-03-14 |
| 6 | 강은비 | eunbi.kang@tbnws.com | agent | O | 2026-03-14 |
| 7 | 안성태 | sungtae.an@tbnws.com | agent | O | 2026-03-14 |
| 8 | 신희웅 | heewoong.shin@tbnws.com | agent | O | 2026-03-14 |
| 9 | 위다혜 | dahye.wi@tbnws.com | agent | O | 2026-03-14 |
| 10 | 이강혁 | kanghyuk.lee@tbnws.com | agent | O | 2026-03-14 |
| 11 | 박혜경 | hyekyung.park@tbnws.com | agent | O | 2026-03-14 |
| 12 | 신진우 | jinwoo.shin@tbnws.com | agent | O | 2026-03-14 |
| 13 | 전은영 | eunyoung.jeon@tbnws.com | agent | O | 2026-03-16 |

---

## 2. 메일함 (mailbox_config) — 4개

| ID | 주소 | 브랜드 | 카테고리 | 표시명 | 활성 |
|----|------|--------|----------|--------|------|
| 9 | support@keychron.kr | keychron | cs | 키크론 고객지원 | O |
| 10 | support@gtgear.co.kr | gtgear | cs | 지티기어 고객지원 | O |
| 11 | b2b@keychron.kr | keychron | b2b | 키크론 B2B | O |
| 12 | ai.login@tbnws.com | tbnws | cs | TBNWS AI Support | O |

---

## 3. AI 엔진 (llm_engines) — 8개

| ID | 이름 | 타입 | 모델 | 활성 | 우선순위 |
|----|------|------|------|------|----------|
| 4 | Claude CLI (Claude Code) | claude_cli | sonnet | **O** | 1 |
| 5 | qwen3:14b | openai_compat | qwen3:14b | X | 2 |
| 6 | Qwen3 8B Fast (PC) | openai_compat | qwen3:8b | X | 60 |
| 7 | Qwen2.5 14B (PC) | openai_compat | qwen2.5:14b | X | 80 |
| 8 | Qwen3 14B (MacStudio) | openai_compat | qwen3:14b | X | 0 |
| 9 | DeepSeek R1 14B (MacStudio) | openai_compat | deepseek-r1:14b | X | 70 |
| 10 | Qwen3 NoThink 14B (Mini) | openai_compat | qwen3-nothink:14b | X | 30 |
| 11 | Qwen3 14B (Mini) | openai_compat | qwen3:14b | X | 50 |

> 현재 Claude CLI만 활성. 로컬 LLM은 서버 미연결로 비활성 처리.

---

## 4. 이메일 처리 룰 (rules) — 7개

### Rule 6: 스팸/광고 메일 필터 (우선순위: 1)
- **조건**: 제목에 "광고, [AD], unsubscribe, 마케팅" 포함 OR 발신 도메인이 marketing.com, promo.kr
- **액션**: tag_only (스팸 태그)

### Rule 7: POC: 키크론 문의 (통합) (우선순위: 5)
- **조건**: 제목/본문에 "키크론, keychron, 키보드, 스위치, 키캡, 배송, 교환, 반품, AS..." OR 수신 ai.login@tbnws.com
- **액션**: auto_reply, 엔진: qwen3-nothink-mini

### Rule 12: POC: 지티기어/파나텍 문의 (우선순위: 4)
- **조건**: 제목/본문에 "지티기어, gtgear, 파나텍, fanatec, 시뮬레이터, 레이싱..." OR 수신 support@gtgear.co.kr
- **액션**: auto_reply, 엔진: claude-cli, 지식베이스: gtgear-products-md
- **시스템 프롬프트**: "당신은 지티기어(GTGear) 고객지원 AI입니다..."

### Rule 8: CS: 배송 문의 (우선순위: 10)
- **조건**: 제목/본문에 "배송, 택배, 추적, 운송장, 언제 도착"
- **액션**: auto_reply, 엔진: qwen3-14b-mac

### Rule 9: CS: 교환/반품 (우선순위: 20)
- **조건**: 제목/본문에 "교환, 반품, 환불, 취소"
- **액션**: draft_and_wait, 엔진: claude-cli

### Rule 10: AS: 수리/AS 접수 (우선순위: 25)
- **조건**: 제목/본문에 "수리, AS, 고장, 불량, 작동안"
- **액션**: draft_and_wait, 엔진: claude-cli

### Rule 11: B2B: 견적/납품 문의 (우선순위: 30)
- **조건**: 제목/본문에 "견적, 납품, 대량, 단체, enterprise" AND 수신 b2b@
- **액션**: draft_and_wait, 엔진: claude-cli

---

## 5. 자동 배정 규칙 (assignment_rules) — 19개

### 1차 담당자 (우선순위 85~100)

| 우선순위 | 룰 이름 | 조건 | 담당자 ID | 담당자 |
|----------|---------|------|-----------|--------|
| 100 | 키크론 AS문의 → 신희웅 | brand=keychron | 8 | 신희웅 |
| 100 | 지티기어 CS → 강은비 | brand=gtgear | 6 | 강은비 |
| 100 | 에이퍼 CS → 강은비 | brand=aiper | 6 | 강은비 |
| 100 | TBNWS CS → 강은비 | brand=tbnws | 6 | 강은비 |
| 95 | 키크론 B2B → 문소현 | type=b2b | 5 | 문소현 |
| 90 | 키크론 AS → 위다혜 (2차) | brand=keychron | 9 | 위다혜 |
| 90 | 지티기어 CS → 이강혁 (2차) | brand=gtgear | 10 | 이강혁 |
| 85 | 배송문의 → 안성태 | category=배송 | 7 | 안성태 |
| 85 | 반품/환불 → 이강혁 | category=반품 | 10 | 이강혁 |
| 80 | B2B 견적문의 → 박혜경 | category=b2b | 11 | 박혜경 |
| 70 | B2B 견적문의 → 신진우 (2차) | category=b2b | 12 | 신진우 |

### 참조/관리자 (우선순위 50)

| 우선순위 | 룰 이름 | 담당자 |
|----------|---------|--------|
| 50 | 키크론 AS문의 → 양우호 (참조) | 양우호 (ID:2) |
| 50 | 지티기어 CS → 양우호 (참조) | 양우호 (ID:2) |
| 50 | 에이퍼 CS → 양우호 (참조) | 양우호 (ID:2) |
| 50 | TBNWS CS → 양우호 (참조) | 양우호 (ID:2) |
| 50 | B2B → 양우호 (참조) | 양우호 (ID:2) |
| 50 | 배송문의 → 양우호 (참조) | 양우호 (ID:2) |
| 50 | 반품/환불 → 양우호 (참조) | 양우호 (ID:2) |

### Fallback (우선순위 1)

| 우선순위 | 룰 이름 | 담당자 |
|----------|---------|--------|
| 1 | [Fallback] 미분류 → 양우호 | 양우호 (ID:2) |

---

## 6. 자동화 (automations) — 5개

| ID | 이름 | 트리거 | 설명 |
|----|------|--------|------|
| 1 | 미답변 24시간 알림 | time_based | 24시간 미답변 시 텔레그램 알림 + 우선순위 high |
| 2 | 긴급 이메일 자동 에스컬레이션 | time_based | urgent 이메일 1시간 미처리 시 알림 |
| 3 | 해결 → 자동 종료 (48시간) | time_based | solved 상태 48시간 후 closed |
| 4 | 신규 이메일 AI 자동 처리 | event_based | email_received → AI 처리 + 자동 배정 |
| 5 | 승인 완료 시 고객 답변 발송 | event_based | approval_approved → 답변 발송 |

---

## 7. SLA 정책 (sla_policies) — 4개

| ID | 이름 | 우선순위 | 첫 응답(분) | 해결(분) | 영업시간 | 활성 |
|----|------|----------|------------|---------|----------|------|
| 1 | 긴급 SLA | urgent | 60 | 240 | O | O |
| 2 | 높음 SLA | high | 120 | 480 | O | O |
| 3 | 보통 SLA | normal | 240 | 1440 | O | O |
| 4 | 낮음 SLA | low | 480 | 2880 | O | O |

---

## 8. 알림 채널 (notification_channels) — 2개

| ID | 타입 | 이름 | 활성 |
|----|------|------|------|
| 1 | slack | CS Slack 알림 | X |
| 3 | telegram | 이메일봇 텔레그램 | **O** |

### 알림 라우팅 (notification_routing)

| ID | 채널 | 이벤트 | 활성 |
|----|------|--------|------|
| 1 | slack-cs | approval_needed | X |
| 2 | slack-cs | escalation | X |

> 현재 Slack 비활성, 텔레그램만 사용 중

---

## 9. 텔레그램 연동 (telegram_user_links) — 1개

| ID | Agent ID | Chat ID | 활성 | 알림 |
|----|----------|---------|------|------|
| 2 | 2 (양우호) | 8188602191 | O | O |

---

## 10. 지식베이스 (knowledge_repositories) — 9개

| ID | Repo ID | 이름 | 타입 | 태그 | 활성 |
|----|---------|------|------|------|------|
| 10 | gtgear-products-md | 지티기어 상품 문서 | filesystem | gtgear, product, schema | O |
| 11 | keychron-faq-md | 키크론 FAQ 문서 | filesystem | keychron, faq, knowledge | O |
| 12 | email-bot-db | 이메일 봇 내부 DB | postgres | internal, email-bot | O |
| 13 | gtgear-cs | 지티기어 CS 가이드 | filesystem | gtgear, cs | O |
| 14 | aiper-cs | 에이퍼 CS 가이드 | filesystem | aiper, cs | O |
| 15 | pawswing-cs | 포스윙 CS 가이드 | filesystem | pawswing, cs | O |
| 16 | shipping-guide | 배송문의 통합 가이드 | filesystem | shipping, delivery, cs | O |
| 17 | keychron-cs-db | 키크론 CS DB (TBNWS_ADMIN) | custom_mcp | keychron, cs, database | O |
| 21 | internal-hr | 사내 조직/인사 정보 | manual | — | O |

### 지식 문서 (knowledge_documents) — 13개

| ID | Repo | 제목 | 카테고리 |
|----|------|------|----------|
| 1 | keychron-faq-md | A/S 발송 안내 절차 | procedure |
| 2 | keychron-faq-md | 반품/교환 절차 안내 | procedure |
| 3 | keychron-faq-md | 배송 관련 FAQ | faq |
| 4 | gtgear-cs | 지티기어 AS 처리 가이드 | faq |
| 5 | gtgear-cs | 지티기어 교환/반품/환불 정책 | faq |
| 6 | gtgear-cs | 지티기어 FAQ — 고객 응대 가이드 | faq |
| 7 | gtgear-cs | 지티기어 배송 안내 가이드 | faq |
| 8 | aiper-cs | 에이퍼 AS 처리 가이드 | faq |
| 9 | aiper-cs | 에이퍼 FAQ — 고객 응대 가이드 | faq |
| 10 | pawswing-cs | 포스윙 AS 처리 가이드 | faq |
| 11 | pawswing-cs | 포스윙 FAQ — 고객 응대 가이드 | faq |
| 15 | internal-hr | 직원현황 및 조직도 | 조직/인사 |
| 16 | internal-hr | CS/발주 담당자 배정 규칙 | 배정규칙 |

---

## 11. 영업시간 (business_hours) — 1개

| 이름 | 시간대 | 기본 |
|------|--------|------|
| 기본 영업시간 (평일 09:00-18:00) | Asia/Seoul | O |

**스케줄**: 월~금 09:00-18:00 (토/일 휴무)

---

## 12. AI 고지문 (ai_disclaimers) — 1개

| 이름 | 언어 | 기본 |
|------|------|------|
| 기본 한국어 고지문 | ko | O |

> "본 메일은 AI 상담 시스템에 의해 작성되었습니다. 정확한 정보를 제공하기 위해 노력하지만, 일부 내용에 오류가 있을 수 있습니다. 추가 확인이 필요하시면 고객센터로 문의해 주세요."

---

## 13. 이메일 서명 (email_signatures) — 1개

| 이름 | 언어 | 기본 |
|------|------|------|
| 지티기어 기본 서명 | ko | O |

---

## 14. Celery 스케줄 (celery_schedules) — 3개

| ID | 이름 | 태스크 | 주기 | 활성 |
|----|------|--------|------|------|
| 1 | Gmail 수신함 폴링 (3분) | check_new_emails | */3 * * * * | O |
| 2 | 브랜드 메일박스 폴링 (5분) | check_mailbox_emails | */5 * * * * | O |
| 3 | 개인 메일 폴링 (10분) | poll_personal_gmail | */10 * * * * | O |

---

## 15. 운영 통계 (snapshot)

| 항목 | 건수 |
|------|------|
| 전체 이메일 | 134 |
| 승인 대기 | 20 |
| 발송 완료 | 2 |
| 챗봇 알림 | 2 (chatbot_notifications) |

---

## 16. 담당자-브랜드 배정 요약표

| 브랜드 | 1차 담당자 | 2차 담당자 | 관리자(참조) |
|--------|-----------|-----------|-------------|
| 키크론 (CS) | 신희웅 | 위다혜 | 양우호 |
| 키크론 (B2B) | 문소현 | — | 양우호 |
| 지티기어 | 강은비 | 이강혁 | 양우호 |
| 에이퍼 | 강은비 | — | 양우호 |
| TBNWS | 강은비 | — | 양우호 |
| 배송 | 안성태 | — | 양우호 |
| 반품/환불 | 이강혁 | — | 양우호 |
| B2B 견적 | 박혜경 | 신진우 | 양우호 |
| 미분류 (Fallback) | 양우호 | — | — |
