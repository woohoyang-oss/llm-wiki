# 브라우저 기능 테스트 4차 결과 (2026-03-23)

## 결과: ❌ 실패

### 원인
- 태스크에 JWT 토큰 값이 비어있었음
- 태스크 원문: `JWT: ` (값 없음, 빈 문자열)
- 빈 토큰으로는 당연히 인증 불가

### 요청
- 유효한 JWT 토큰 (exp가 현재 시간 이후)을 태스크에 포함해주세요
- 필요한 exp 값: 최소 1774425600 이상 (2026-03-23 이후)
- 또는 서버에서 직접 발급: `python3 -c "import jwt, time; print(jwt.encode({'sub':'1','email':'wooho.yang@tbnws.com','name':'우호','role':'admin','exp':int(time.time())+86400,'iat':int(time.time())}, 'SECRET_KEY', algorithm='HS256'))"` (SECRET_KEY는 config.py의 JWT_SECRET_KEY 값)
