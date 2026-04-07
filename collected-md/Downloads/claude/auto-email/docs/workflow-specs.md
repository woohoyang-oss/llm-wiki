# 워크플로우 상세 스펙 — 데이터 연결 가정 하의 기능 구현

> 작성일: 2026-03-13
> 전제: 모든 데이터는 MCP 레포지토리로 이미 연결되어 있음.
> 이 문서는 "데이터가 있을 때 시스템이 어떻게 동작하는가"에 집중.

---

## 1. 이메일 수신 → 자동 처리 워크플로우

### 1.1 전체 흐름

```
[Gmail 수신]
    ↓
[Celery Beat: 메일박스별 폴링 태스크]
    ↓
[메일 파싱 + DB 저장] (status = "processing")
    ↓
[자동 태그] — KEYWORD_TAG_MAP 기반
    ↓
[룰 엔진 매칭] — priority 순서대로 조건 평가
    ↓ 매칭된 룰의 action 타입에 따라:
    ├─ tag_only → [태그만 부여, 완료]
    ├─ forward → [지정 주소로 전달]
    └─ auto_reply → [AI 처리 진입]
                        ↓
                    [MCP tool_use로 외부 데이터 조회]
                    (고객정보, 주문상태, 재고, 배송 등)
                        ↓
                    [AI 답변 초안 생성 + confidence 산출]
                        ↓
                    ├─ confidence ≥ 0.85 → [자동 발송] + 알림
                    ├─ 0.70 ≤ conf < 0.85 → [승인 큐] + 담당자 알림
                    └─ conf < 0.70 → [에스컬레이션] + 관리자 알림
```

### 1.2 각 단계별 구현 위치

| 단계 | 파일 | 메서드 | 현재 상태 |
|------|------|--------|-----------|
| 폴링 | workers/tasks.py:87 | check_new_emails | ✅ 구현됨 |
| 파싱 | services/pipeline.py:78 | process_message | ✅ 구현됨 |
| 자동 태그 | services/pipeline.py:120 | _auto_tag | ✅ 구현됨 |
| 룰 매칭 | services/rule_engine.py:66 | match | ✅ 구현됨 |
| AI 처리 | services/ai_processor.py:332 | process_email | ✅ 구현됨 |
| MCP 데이터 조회 | services/ai_processor.py:56 | _build_dynamic_mcp_config | ✅ 구현됨 |
| 자동 발송 | services/pipeline.py:281 | _auto_send | ✅ 구현됨 |
| 승인 큐 | services/pipeline.py:341 | _queue_for_approval | ✅ 구현됨 |
| 에스컬레이션 | services/pipeline.py:374 | _escalate | ✅ 구현됨 |
| 알림 | services/notification.py:101 | notify | ✅ 구현됨 |

**결론:** 핵심 파이프라인은 100% 구현됨. Phase 1은 이 파이프라인을 **자동 실행**하는 인프라(스케줄러, 워커)를 구축.

---

## 2. 공용 메일함 워크플로우

### 2.1 담당자가 메일을 처리하는 흐름

```
[담당자 로그인 → 대시보드]
    ↓
[내 할일 확인]
  - SLA 임박 메일 (빨간색)
  - 승인 대기 (노란색)
  - 신규 배정 (파란색)
    ↓
[메일 클릭 → 상세 보기]
    ↓
[AI 초안 확인]
  ├─ AI 초안이 좋으면 → [승인 / 바로 발송]
  ├─ 수정 필요하면 → [수정 후 발송]
  ├─ AI에게 재지시 → [Instruct: 대화형 수정]
  │   "좀 더 공손하게 써줘" / "반품 절차도 안내해줘"
  │     ↓
  │   [AI가 수정된 초안 생성]
  │     ↓
  │   [만족 → 발송] / [불만족 → 재지시 반복]
  └─ 직접 작성 → [수동 답장]
    ↓
[발송 완료]
    ↓
[지식 저장 제안] ← Phase 4
  "이 답변을 지식으로 저장하시겠습니까?"
    ↓
[저장 → 다음 유사 메일에 AI가 참조]
```

### 2.2 Instruct (대화형 AI 수정) — 현재 구현 상세

**엔드포인트:** `POST /admin/emails/{email_id}/instruct`

**요청:**
```json
{
  "messages": [
    {"role": "user", "content": "배송 지연 사유를 좀 더 구체적으로 써줘"}
  ]
}
```

**처리 흐름 (ai_processor.py:400-465):**
1. 원본 메일 정보 로드 (subject, body, from)
2. 활성 레포지토리에서 MCP 설정 빌드
3. conversation 히스토리와 함께 Claude에 전달
4. 응답 파싱: `type="chat"` (추가 질문) 또는 `type="draft"` (최종 초안)

**응답 예시:**
```json
{
  "type": "draft",
  "draft_subject": "Re: 배송 문의",
  "draft_body": "<p>고객님, 현재 CJ대한통운 물량 폭주로 인해...</p>",
  "confidence": 0.88,
  "reasoning": "배송 지연 원인을 구체적으로 명시하여 신뢰도를 높임"
}
```

---

## 3. 개인 메일함 워크플로우

### 3.1 개인 메일 AI 활용 흐름

```
[담당자 → 내 메일함]
    ↓
[개인 Gmail 메일 목록 표시]
    ↓
[메일 선택 → 상세 보기]
    ↓
[AI 자동초안 버튼 클릭]
    ↓ POST /admin/personal-emails/auto-draft
[AI가 개인 지식 + 공용 지식 참조하여 초안 생성]
    ↓
  ├─ 초안 OK → [발송]
  ├─ 수정 필요 → [Instruct 대화]
  │   POST /admin/personal-emails/instruct
  │   {"email_subject": "...", "email_body": "...", "messages": [...]}
  │     ↓
  │   [AI 수정 초안] → [발송]
  └─ 직접 작성 → [수동 답장]
    ↓
[발송 완료]
    ↓ Phase 4 추가
[지식 저장 제안]
  ├─ "개인 지식으로 저장" → 내 개인 레포에 저장
  └─ "공용 지식으로 저장" → admin만 (공용 레포)
```

### 3.2 개인 vs 공용 AI 처리 차이점

| 항목 | 공용 메일함 | 내 메일함 |
|------|------------|-----------|
| 메일 소스 | 공용 계정 (support@) | 개인 Gmail |
| AI 지식 참조 | 공용 레포만 | 개인 레포 + 공용 레포 |
| 룰 매칭 | 공용 룰 | 개인 룰 우선 → 공용 룰 폴백 |
| 자동 발송 | confidence ≥ 0.85 시 자동 | 자동 발송 없음 (항상 확인 후) |
| 서명/고지문 | 메일박스 설정 | 개인 설정 |
| 배정 | 자동배정 엔진 | 해당 없음 (내 메일) |

---

## 4. 승인 처리 워크플로우

```
[AI 초안이 승인 큐에 등록] (confidence 0.70~0.84)
    ↓
[담당자에게 알림] (Telegram/Slack/Email)
    ↓
[담당자 → 승인 대기 페이지]
    ↓
[초안 확인]
  ├─ 승인 → [원문 그대로 발송]
  ├─ 수정 승인 → [수정 후 발송]
  └─ 거부 → [다시 큐에서 제거, 수동 처리]
    ↓
[발송 완료 → 활동 로그 기록]
```

**엔드포인트:** `POST /admin/approvals/{id}/action`
```json
{"action": "approve"}              // 원문 그대로
{"action": "edit", "edited_body": "..."} // 수정 후
{"action": "reject"}               // 거부
```

---

## 5. 자동 배정 워크플로우

```
[메일 수신 → 룰 매칭 → AI 처리 완료]
    ↓
[자동 배정 엔진] ← Phase 2
    ↓
[배정 규칙 매칭] (brand × category → agent/group)
    ↓ 규칙 매칭 시:
    ├─ strategy=fixed → [지정 에이전트에 배정]
    ├─ strategy=round_robin → [그룹 내 순환 배정]
    └─ strategy=least_load → [가장 적은 담당자에 배정]
    ↓ 미매칭 시:
    [미배정 상태 유지 → admin이 수동 배정]
    ↓
[배정된 에이전트에게 알림]
```

---

## 6. 알림 라우팅 워크플로우

```
[이벤트 발생] (auto_sent / approval_needed / escalation / error)
    ↓
[NotificationRouting 조회]
  조건: event_type + brand + category + agent_id
    ↓
[매칭된 라우팅 규칙의 channel로 발송]
  ├─ Slack webhook
  ├─ Telegram bot
  ├─ Webhook (외부 시스템)
  └─ Email (내부 알림)
    ↓
[발송 실패 시 → 로그만 (에러 무시, 파이프라인 중단 안 함)]
```

---

## 7. 지식 축적 워크플로우 (Phase 4)

### 7.1 답변 → 지식 전환

```
[메일 답변 발송 완료]
    ↓
[시스템이 판단]
  - 이 답변에 유용한 정보가 포함되어 있는가?
  - AI instruct 대화가 3턴 이상이었는가?
  - 기존 지식문서에 없는 새로운 내용인가?
    ↓ (하나라도 해당)
[지식 저장 제안 모달 표시]
    ↓
[사용자 선택]
  ├─ "개인 지식으로 저장"
  │   → KnowledgeDocument(scope='personal', owner_id=me) 생성
  │   → KnowledgeEmbedding 자동 생성 (벡터 검색용)
  ├─ "공용 지식으로 저장" (admin만)
  │   → KnowledgeDocument(scope='shared') 생성
  ├─ "건너뛰기" → 아무것도 안 함
  └─ "다시 묻지 않기" → personal_settings.auto_save_knowledge = false
```

### 7.2 지식 → AI 개선 선순환

```
[다음에 유사한 메일 수신]
    ↓
[AI 처리 시 _build_dynamic_mcp_config]
  → 공용 레포 + 개인 레포 (Phase 4) 모두 포함
    ↓
[Claude가 MCP tool_use로 지식문서 검색]
  → 이전에 저장한 답변 지식이 검색됨
    ↓
[더 정확한 답변 생성 → 더 높은 confidence]
    ↓
[자동 발송 비율 증가 → 업무 부담 감소]
```

### 7.3 패턴 감지 → 룰 자동 제안

```
[시스템이 주기적으로 분석] (또는 답변 발송 시)
    ↓
[조건: 같은 사용자가 유사한 메일에 유사한 답변을 3회 이상 발송]
  - 유사 판단: 임베딩 코사인 유사도 ≥ 0.85
    ↓
[룰 생성 제안]
  "이 유형의 메일에 대해 자동 룰을 만드시겠습니까?"
  제안 내용:
    - 조건: subject_or_body contains [추출된 키워드]
    - 액션: auto_reply (knowledge_repos: [관련 레포])
    ↓
[사용자가 승인 → 개인 룰 자동 생성]
```

---

## 8. 자동화 규칙 실행 워크플로우

### 8.1 이벤트 기반 자동화

```
[이벤트 발생]
  예: email.received, email.status_changed, email.sla_warning
    ↓
[Automation 테이블에서 매칭]
  trigger_type='event_based'
  conditions 평가 (brand, category, status 등)
    ↓
[매칭된 자동화의 actions 순차 실행]
  1. assign_agent → 에이전트 배정
  2. set_priority → 우선순위 변경
  3. send_notification → 알림 발송
  4. add_tag → 태그 추가
  ...
    ↓
[run_count 증가, last_run_at 업데이트]
```

### 8.2 시간 기반 자동화

```
[Celery Beat 스케줄에 의해 트리거]
  예: 매일 09:00, 매주 월요일
    ↓
[Automation 조건 평가]
  예: "status=pending AND created_at < 24h ago"
    ↓
[매칭 메일에 액션 실행]
  예: 24시간 미응답 메일 자동 에스컬레이션
```

---

## 9. 역할별 행동 매트릭스

### Admin이 하는 일
| 행동 | 페이지 | 빈도 |
|------|--------|------|
| 전체 운영 현황 확인 | 대시보드 | 매일 |
| 에이전트 성과 확인 | 통계 | 매주 |
| 공용 룰 생성/수정 | 룰 관리 | 가끔 |
| 공용 지식문서 관리 | 지식 문서 | 가끔 |
| 배정 규칙 설정 | 배정 규칙 | 가끔 |
| 에이전트 관리 | 담당자 | 가끔 |
| 메일박스 설정 | 메일박스 | 드물게 |
| 에스컬레이션 메일 처리 | 공용 메일함 | 매일 |

### Agent가 하는 일
| 행동 | 페이지 | 빈도 |
|------|--------|------|
| 내 할일 확인 | 대시보드 | 매일 여러번 |
| 배정된 메일 처리 | 공용 메일함 | 매일 |
| AI 초안 승인/수정 | 승인 대기 | 매일 |
| 개인 메일 처리 | 내 메일함 | 매일 |
| 개인 룰/템플릿 관리 | 룰/템플릿 | 가끔 |
| 개인 지식 축적 | 지식 문서 | 메일 처리 후 |
| 내 성과 확인 | 통계 | 매주 |

### Viewer가 하는 일
| 행동 | 페이지 | 빈도 |
|------|--------|------|
| 운영 현황 모니터링 | 대시보드 | 매일 |
| 메일 이력 조회 | 공용 메일함 | 필요 시 |
| 통계 확인 | 통계 | 매주 |
