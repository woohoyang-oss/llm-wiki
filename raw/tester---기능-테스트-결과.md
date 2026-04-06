# 브라우저 기능 테스트 결과 (2026-03-23)

## JWT 토큰 주입 방식 테스트

### 결과: 5/5 HTTP 200 — 그러나 모든 페이지가 로그인 화면 표시

| # | 페이지 | HTTP | 실제 렌더링 |
|---|--------|------|------------|
| 1 | /dashboard | 200 | ❌ 로그인 화면 |
| 2 | /emails | 200 | ❌ 로그인 화면 |
| 3 | /rules | 200 | ❌ 로그인 화면 |
| 4 | /agents | 200 | ❌ 로그인 화면 |
| 5 | /my-inbox | 200 | ❌ 로그인 화면 |

### 분석
- HTTP 200은 Next.js SPA가 정상 응답한 것
- localStorage에 JWT 토큰 주입 후 페이지 이동했으나, SPA 라우터가 토큰을 인식하기 전에 로그인 페이지로 리다이렉트
- 가능한 원인:
  1. 토큰 키가 'token'이 아닐 수 있음 (access_token, auth_token 등)
  2. 쿠키 기반 인증일 수 있음 (httpOnly cookie)
  3. JWT 만료 시간 문제
  4. SPA 라우터의 인증 체크 타이밍

### API 직접 호출 확인
- /api/admin/dashboard, /api/admin/emails 등은 이전 테스트에서 401 반환 확인 완료
- 인증 체계 자체는 정상 동작

### 권장사항
- 프론트엔드 코드에서 실제 토큰 저장 키 확인 필요
- Google OAuth 로그인 후 실제 토큰 발급 플로우로 테스트 필요

## 스크린샷
- tests/screenshot_dashboard.png 등 5개 저장됨
