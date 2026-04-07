# Email Support Bot — 구현 계획서 v2.0

> 작성일: 2026-03-13
> 현재 버전: POC v0.1

---

## 1. 핵심 목적

> 회사 내 다양한 데이터(고객문의, 주문, 배송, 발주, 견적)를 활용하여,
> 개인/공용 이메일의 자동·반자동 처리로 전방위 업무 효율화

---

## 2. 현재 상태 요약

| 영역 | 완성도 | 핵심 갭 |
|------|--------|---------|
| Email Pipeline | 85% | Celery 워커 실행 시 자동 동작하나, 현재 워커 미실행 |
| Batch Scheduling | 40% | Beat 스케줄 1개(5분 폴링) 고정, Admin UI 없음 |
| Agent Assignment | 90% | Rule 액션에 assignee 지원, 자동배정 로직 없음 |
| Knowledge Editing | 95% | Admin UI CRUD 완료 |
| AI Confidence Routing | 95% | ≥0.85 자동발송, 0.70~0.84 승인큐, <0.70 에스컬레이션 |
| Handler Notifications | 70% | Slack/Telegram/Webhook 지원, 에이전트별 라우팅 없음 |
| Role/Permissions | 70% | role 필드 있으나 엔드포인트별 ACL 미적용 |
| Email Sending | 95% | Gmail API Send + Send As 구현 완료 |
| Data Integration (MCP) | 90% | MySQL/FS/MCP 서버 연결 가능, 시나리오별 설정 필요 |
| Personal Workspace | 30% | 개인 메일함 UI 있으나, 개인/공용 엔티티 분리 없음 |
| Dashboard/Statistics | 60% | 기본 차트 있으나 역할별 뷰, 기간 필터 없음 |

---

## 3. Phase 구성 (5 Phases)

| Phase | 내용 | 기간 | 상세 문서 |
|-------|------|------|-----------|
| **Phase 1** | 자동 파이프라인 완성 | 5일 | `phase1-auto-pipeline.md` |
| **Phase 2** | 알림 + 자동배정 | 5일 | `phase2-notification-assignment.md` |
| **Phase 3** | RBAC 권한 + 보안 | 3일 | `phase3-rbac-permissions.md` |
| **Phase 4** | 개인 워크스페이스 + AI 지식 축적 | 5일 | `phase4-personal-workspace.md` |
| **Phase 5** | 역할별 대시보드 + 통계 | 3일 | `phase5-dashboard-analytics.md` |
| — | 데이터 연동 시나리오 | 참조 | `data-integration-scenarios.md` |
| — | 페이지별 기능 스펙 | 참조 | `page-feature-specs.md` |
| — | 통합 마이그레이션 | 참조 | `migration-guide.md` |
| ⏸️ | SLA 정책 / 영업시간 고도화 | 후순위 | 기본 기능 구현 완료, 베타 서비스 |

---

## 4. Phase별 요약

### Phase 1: 자동 파이프라인 (5일)
- Celery Beat DB 스케줄러 (동적 스케줄 관리)
- 메일박스별 독립 폴링 + 개별 주기 설정
- GmailService 멀티 메일박스 지원
- 동기 세션 (sync_session) 추가
- Admin UI: 스케줄 관리 페이지
- 개발환경 시작 스크립트

### Phase 2: 알림 + 자동배정 (5일)
- 에이전트별/브랜드별 알림 라우팅
- 자동 배정 엔진 (fixed/round_robin/least_load)
- Email 알림 채널 추가
- pipeline.py 통합
- Admin UI: 배정 규칙 + 알림 라우팅 관리

### Phase 3: RBAC 권한 (3일)
- 기존 require_role() 활성화 + 65개 엔드포인트 적용
- verify_admin() → require_role() 마이그레이션
- Frontend: 역할별 사이드바 필터링, RoleGuard
- 엔티티별 접근 제어 (admin: 전체, agent: 담당분, viewer: 읽기)

### Phase 4: 개인 워크스페이스 (5일)
- scope + owner_id 패턴 (9개 엔티티)
- 공용/개인 분리: 룰, 레포, 지식문서, 템플릿, 매크로, 서명, 고지문, 알림
- 개인 메일함 설정 (서명, 고지문, AI 자동초안, 알림)
- AI 지식 축적 (답변 → 지식문서 자동 생성 → 다음 AI 개선)
- 반복 패턴 감지 → 룰 자동 제안

### Phase 5: 대시보드 + 통계 (3일)
- Admin 대시보드: 전체 운영 현황
- Agent 대시보드: 내 할일 + 개인 성과 + 개인 메일 요약
- Viewer 대시보드: 읽기 전용 (빠른 실행 숨김)
- 통계: 기간 선택, 브랜드/에이전트 필터, CSV 내보내기
- Agent 개인 성과 뷰, AI 활용 현황

---

## 5. 의존성 + 일정

```
Week 1: Phase 1 (자동 파이프라인)
  Day 1: DB 스키마 + sync_session
  Day 2: DatabaseScheduler + GmailService 리팩토링
  Day 3: 메일박스 폴링 태스크 + 스케줄 서비스
  Day 4: Admin API + celery_app 변경
  Day 5: Admin UI + docker/dev.sh + 테스트

Week 2: Phase 2 (알림 + 자동배정)
  Day 1: DB 스키마 + AssignmentService
  Day 2: notification.py 확장 + EmailAdapter
  Day 3: pipeline.py 통합
  Day 4: Admin API (8 엔드포인트)
  Day 5: Admin UI + 테스트

Week 3: Phase 3 + Phase 5
  Day 1: require_role 수정 + 엔드포인트 적용 (그룹1)
  Day 2: 엔드포인트 적용 (그룹2-3) + Frontend RoleGuard
  Day 3: Phase 3 완료 + Phase 5 대시보드 API 분기
  Day 4: Phase 5 통계 확장 + 필터
  Day 5: Phase 5 Frontend + 테스트

Week 4: Phase 4 (개인 워크스페이스)
  Day 1: scope/owner_id 마이그레이션 (9 테이블)
  Day 2: ScopeFilter 서비스 + API 36개 엔드포인트 확장
  Day 3: 개인설정 API + 지식축적 서비스
  Day 4: 지식 축적 API + 개인 AI 파이프라인
  Day 5: Frontend (공용/개인 탭 + 지식 모달 + 개인 설정)
```

---

## 6. 데이터 연동 활용

MCP 서버 기반으로 다양한 데이터 소스 연결:

| 데이터 | 연결 방식 | 활용 |
|--------|-----------|------|
| TBNWS CRM (고객/AS/주문) | MySQL MCP | 고객 조회, AS 이력, 주문 확인 |
| 사방넷 (멀티채널 주문) | MySQL MCP | 배송 상태, 주문 이력 |
| ERP (재고/발주/원가) | PostgreSQL MCP | 재고 확인, 견적, 발주 현황 |
| 택배사 배송 조회 | Custom MCP 개발 | 실시간 배송 추적 |
| 내부 지식문서 | 파일시스템 MCP | 업무 매뉴얼, FAQ, 정책 |

**시나리오 상세:** `data-integration-scenarios.md` 참조

---

## 7. 자동화 수준 로드맵

```
Level 1 (현재): 분류 + 배정
  메일 수신 → 룰 매칭 → 카테고리 분류 → 에이전트 배정

Level 2 (Phase 1 후): AI 초안 + 승인
  메일 수신 → AI 초안 생성 → 승인 큐 → 확인 → 발송

Level 3 (Phase 1-2 후): 데이터 연동 자동 답변
  메일 수신 → MCP로 실시간 데이터 조회 → AI 답변(데이터 포함) → 자동 발송

Level 4 (Phase 4 후): 지식 축적 + 자동 개선
  답변 → 지식 저장 → 다음 유사 메일에 더 높은 confidence → 자동화 영역 확대

Level 5 (최종): 전방위 자동화
  고객 메일, B2B, 내부 메일, 재고 알림, 발주, 견적 → 모두 AI 기반 처리
```

---

## 8. 즉시 실행 가능한 작업

```bash
# 터미널 1: API 서버
uvicorn app.main:app --reload --port 8000

# 터미널 2: Celery Worker
celery -A app.workers.celery_app worker --loglevel=info -Q email_processing,email_polling

# 터미널 3: Celery Beat (5분마다 폴링)
celery -A app.workers.celery_app beat --loglevel=info
```

---

## 9. 문서 목록

| 문서 | 설명 |
|------|------|
| `implementation-plan.md` | 이 문서 — 전체 개요 |
| `phase1-auto-pipeline.md` | Phase 1 상세 (10 태스크) |
| `phase2-notification-assignment.md` | Phase 2 상세 (6 태스크) |
| `phase3-rbac-permissions.md` | Phase 3 상세 (3 태스크) |
| `phase4-personal-workspace.md` | Phase 4 상세 (10 태스크) |
| `phase5-dashboard-analytics.md` | Phase 5 상세 (3 태스크) |
| `data-integration-scenarios.md` | 데이터 연동 × 자동화 시나리오 |
| `workflow-specs.md` | 9개 워크플로우 상세 (데이터 연결 가정) |
| `shared-infrastructure.md` | 공용 인프라 패턴 (중복 방지) |
| `dev-framework.md` | Claude 토큰 절약형 개발 프레임워크 |
| `ux-ui-guidelines.md` | Gmail 친숙 UX/UI 가이드라인 |
| `parallel-dev-guide.md` | 워크트리 병렬 개발 + 로컬 LLM 활용 |
| `page-feature-specs.md` | 22개 페이지별 기능 스펙 |
| `migration-guide.md` | 통합 마이그레이션 가이드 |
