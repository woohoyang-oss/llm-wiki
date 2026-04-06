# 테스터 지시 (커맨드센터 → 테스터)

## 1단계: Playwright 설치
```bash
cd ~/auto-email
npm init -y 2>/dev/null
npx playwright install chromium
```

## 2단계: E2E 테스트 스크립트 작성
tests/e2e_test.py 파일 생성:
```python
import subprocess, json, time

BASE = "https://mail.tbe.kr"

def test_frontend():
    """프론트엔드 접속 확인"""
    r = subprocess.run(["curl", "-s", "-o", "/dev/null", "-w", "%{http_code}", BASE], capture_output=True, text=True)
    assert r.stdout == "200", f"프론트 접속 실패: {r.stdout}"
    print("✅ 프론트엔드 200 OK")

def test_api_auth():
    """API 인증 체크"""
    r = subprocess.run(["curl", "-s", "-o", "/dev/null", "-w", "%{http_code}", f"{BASE}/api/admin/dashboard"], capture_output=True, text=True)
    assert r.stdout == "401", f"API 인증 체크 실패: {r.stdout}"
    print("✅ API 인증 401 정상")

def test_google_login_redirect():
    """Google 로그인 리다이렉트"""
    r = subprocess.run(["curl", "-s", "-o", "/dev/null", "-w", "%{http_code}", f"{BASE}/api/auth/google/login"], capture_output=True, text=True)
    assert r.stdout in ("307", "302"), f"Google 로그인 리다이렉트 실패: {r.stdout}"
    print("✅ Google 로그인 307 리다이렉트 정상")

def test_api_emails():
    """이메일 API 인증 체크"""
    r = subprocess.run(["curl", "-s", "-o", "/dev/null", "-w", "%{http_code}", f"{BASE}/api/admin/emails"], capture_output=True, text=True)
    assert r.stdout == "401", f"이메일 API 실패: {r.stdout}"
    print("✅ 이메일 API 401 정상")

if __name__ == "__main__":
    tests = [test_frontend, test_api_auth, test_google_login_redirect, test_api_emails]
    passed = 0
    for t in tests:
        try:
            t()
            passed += 1
        except AssertionError as e:
            print(f"❌ {t.__name__}: {e}")
    print(f"\n결과: {passed}/{len(tests)} 통과")
```

## 3단계: 실행
```bash
python3 tests/e2e_test.py
```

## 4단계: 결과를 아카샤에 저장
결과를 아카샤에 `Tester - E2E 테스트 결과` 키로 저장해주세요.
