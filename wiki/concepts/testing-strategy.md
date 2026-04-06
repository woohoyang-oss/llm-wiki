---
title: E2E 테스트 전략
tags: [testing, e2e, playwright, crud, jwt, tester]
created: 2026-04-06
updated: 2026-04-06
sources: [tester---테스트-환경-구축--e2e-테스트-실행.md, tester---브라우저-e2e-테스트-실행.md, tester---e2e-테스트-결과.md, tester-crud-전수-점검-결과.md, tester-playwright-삭제-테스트-결과.md]
---
# E2E 테스트 전략

## 개요

[[auto-email]] 프로젝트의 E2E 테스트 전략이다. Playwright 기반 브라우저 테스트, curl 기반 API 테스트, CRUD 전수 점검을 조합하여 프로덕션 환경(mail.tbe.kr)을 검증한다. [[claude-team]]의 Tester 에이전트가 주도적으로 수행한다.

## 테스트 환경 구축

맥스튜디오에서 Playwright chromium을 설치하고 테스트 스크립트를 실행한다.

```
npm init → npx playwright install chromium → python3 tests/e2e_test.py
```

두 가지 테스트 레이어를 운용한다.
- **curl 기반 E2E**: HTTP 상태 코드 검증 (200, 401, 307 등)
- **Playwright 브라우저 E2E**: 실제 브라우저에서 UI 인터랙션 검증

## E2E 테스트 항목

| 테스트 | 검증 내용 | 기대값 |
|--------|----------|--------|
| 프론트엔드 접속 | mail.tbe.kr 응답 | 200 OK |
| API 인증 체크 | /api/admin/dashboard | 401 (미인증 차단) |
| Google 로그인 리다이렉트 | /api/auth/google/login | 307 리다이렉트 |
| 이메일 API 인증 | /api/admin/emails | 401 (미인증 차단) |
| 브라우저 로그인 버튼 | 메인 페이지 버튼 존재 | 버튼 텍스트 확인 |
| Google OAuth 흐름 | 버튼 클릭 후 리다이렉트 | accounts.google.com |

## CRUD 전수 점검

11개 메뉴(담당자, 자동화, 배정규칙, 지식베이스, 메일박스, SLA, 이메일템플릿, 매크로, 서명, 스케줄, 태그)에 대해 LIST/CREATE/UPDATE/DELETE 전체를 API 레벨에서 점검한다.

### 주요 발견 사항
- **PASS 9개** / PARTIAL 1개 / NOT_FOUND 1개
- 스케줄 UPDATE 시 500 에러 발생
- 태그 엔드포인트 미구현 (404)

## Playwright 심층 테스트 -- 삭제 버그 발견

담당자 삭제 테스트에서 중요한 백엔드 버그를 발견했다. API가 HTTP 200 + `{"status":"deleted"}`를 반환하지만, 실제 DB에서 삭제가 이루어지지 않는 현상이다.

### 버그 원인
`delete_agent` 함수가 응답 반환 후 세션 종료 시 commit을 시도하지만, FK 제약 위반(ApprovalQueue, ActivityLog의 CASCADE 누락)으로 commit이 실패하고 자동 rollback된다. FastAPI는 이미 200 응답을 전송한 상태이므로 클라이언트는 성공으로 인식한다.

### 수정 방안
1. 누락된 FK 참조에 NULL 처리 또는 CASCADE 추가
2. return 전 `await db.flush()`로 제약 위반 조기 탐지
3. 명시적 `await db.commit()` + 예외 처리

## JWT 인증 이슈

모든 API 엔드포인트가 JWT 인증을 요구하므로, 테스트 시 Google OAuth 토큰 획득이 선행되어야 한다. 미인증 시 401 응답이 정상 동작함을 확인하였으며, 인증 후 CRUD 테스트는 별도 JWT 토큰으로 수행한다.

## 관련 소스
- [tester---테스트-환경-구축--e2e-테스트-실행.md](../../raw/tester---테스트-환경-구축--e2e-테스트-실행.md)
- [tester---브라우저-e2e-테스트-실행.md](../../raw/tester---브라우저-e2e-테스트-실행.md)
- [tester---e2e-테스트-결과.md](../../raw/tester---e2e-테스트-결과.md)
- [tester-crud-전수-점검-결과.md](../../raw/tester-crud-전수-점검-결과.md)
- [tester-playwright-삭제-테스트-결과.md](../../raw/tester-playwright-삭제-테스트-결과.md)
