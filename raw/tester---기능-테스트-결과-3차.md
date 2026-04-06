# 브라우저 기능 테스트 3차 결과 (2026-03-23)

## 새 JWT 토큰 사용

### 결과: 5/5 여전히 로그인 화면 ❌

### 근본 원인 확인
- 새 토큰의 exp: 1774337273 = 2026-03-21 20:47:53 UTC
- 현재: 2026-03-23 → **토큰 여전히 만료됨 (2일 전)**
- SPA 동작: 페이지 로드 → /api/auth/me 호출 → 서버 401 → auth_token 삭제 → 로그인 리다이렉트
- 토큰 주입 후 reload 시 hasToken: false 확인됨 (앱이 삭제함)

### SPA 인증 플로우
1. 앱 시작 → localStorage에서 auth_token 읽기
2. /api/auth/me에 Bearer 토큰으로 요청
3. 서버: JWT 만료 → 401 반환
4. 프론트: 401 수신 → auth_token 삭제 → 로그인 페이지로 이동

### 해결책
- **현재 시간 이후의 exp를 가진 JWT 발급 필요**
- exp를 최소 1774423000 이상으로 설정 (2026-03-23 이후)
- 또는 서버에서 직접 토큰 발급: POST /api/auth/dev-token (있다면)
- 또는 Google OAuth 실제 로그인 사용

### 스크린샷 저장됨
- tests/ss_dashboard.png ~ ss_my-inbox.png
