# CRUD 전수 점검 결과 (2026-03-23)

## 요약
- PASS: 9개 / PARTIAL: 1개 / NOT_FOUND: 1개

| 메뉴 | LIST | CREATE | UPDATE | DELETE |
|------|------|--------|--------|--------|
| 담당자 | OK 18건 | OK | OK | OK (삭제 정상) |
| 자동화 | OK 2건 | OK | OK | OK |
| 배정규칙 | OK 23건 | OK | OK | OK |
| 지식베이스 | OK 15건 | OK | OK | OK |
| 메일박스 | OK 4건 | OK | OK | OK |
| SLA | OK 3건 | OK | OK | OK |
| 이메일템플릿 | OK 4건 | OK | OK | OK |
| 매크로 | OK 5건 | OK | OK | OK |
| 서명 | OK 2건 | OK | OK(sig_id) | OK(sig_id) |
| 스케줄 | OK 2건 | OK | FAIL 500 | OK |
| 태그 | 404 | 404 | - | - |

## 이슈
1. 스케줄 UPDATE → 500 Internal Server Error
2. 태그 엔드포인트 미구현 (404)
3. 담당자 삭제는 API레벨 정상 — UI 이슈 가능성
