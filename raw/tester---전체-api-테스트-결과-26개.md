# 전체 API 테스트 결과 (2026-03-23 17:25)

## 결과: 18/24 통과 (프론트 전용 2개 제외)

### ✅ 정상 (18개)
dashboard, rules, assignment-rules, automations, schedules, sla-policies,
business-hours, repositories, knowledge-docs, email-templates, macros,
mailboxes, signatures, disclaimers, notifications, personal-settings,
agents, engines, emails, personal-emails

### ❌ 에러 (4개)
| 엔드포인트 | 코드 | 원인 |
|-----------|------|------|
| approvals | 500 | reject_reason 컬럼 미존재 → DB 마이그레이션 필요 |
| statistics | 404 | 엔드포인트 경로 다름 (확인 필요) |
| batch-monitor | 404 | 엔드포인트 경로 다름 |
| system | 404 | 엔드포인트 경로 다름 |

### 🔵 프론트 전용 (2개)
guide, changelog — API 없음, 정상

### 조치 사항
1. [긴급] approvals: alembic migration으로 reject_reason 컬럼 추가
2. statistics/batch-monitor/system: 실제 엔드포인트 경로 확인 후 프론트 수정
