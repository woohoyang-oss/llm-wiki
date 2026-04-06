# 테스터 지시 (커맨드센터)

Playwright로 mail.tbe.kr 브라우저 테스트를 실행해주세요.

## 테스트 스크립트 작성 + 실행

tests/e2e_browser_test.py 생성:

```python
import asyncio
from playwright.async_api import async_playwright

async def test_mail_tbe_kr():
    results = []
    async with async_playwright() as p:
        browser = await p.chromium.launch(headless=True)
        page = await browser.new_page()

        # 1. 메인 페이지 접속
        try:
            resp = await page.goto("https://mail.tbe.kr", wait_until="networkidle", timeout=15000)
            status = resp.status if resp else 0
            title = await page.title()
            results.append(f"✅ 1. 메인 페이지: {status} | 타이틀: {title}")
        except Exception as e:
            results.append(f"❌ 1. 메인 페이지 실패: {e}")

        # 2. 로그인 버튼 존재 확인
        try:
            btn = await page.query_selector("button")
            btn_text = await btn.inner_text() if btn else "없음"
            results.append(f"✅ 2. 로그인 버튼: {btn_text}")
        except Exception as e:
            results.append(f"❌ 2. 로그인 버튼 실패: {e}")

        # 3. Google 로그인 클릭 → 리다이렉트 URL 확인
        try:
            async with page.expect_navigation(timeout=10000) as nav:
                await page.click("button")
            resp = await nav.value
            url = page.url
            if "accounts.google.com" in url:
                results.append(f"✅ 3. Google 리다이렉트 정상: {url[:80]}")
            else:
                results.append(f"⚠️ 3. 리다이렉트 URL: {url[:80]}")
        except Exception as e:
            results.append(f"❌ 3. Google 리다이렉트 실패: {e}")

        # 4. API 직접 호출 테스트
        try:
            api_resp = await page.goto("https://mail.tbe.kr/api/admin/dashboard")
            body = await page.inner_text("body")
            if "인증" in body or "401" in str(api_resp.status):
                results.append(f"✅ 4. API 인증 체크: {api_resp.status}")
            else:
                results.append(f"⚠️ 4. API 응답: {body[:100]}")
        except Exception as e:
            results.append(f"❌ 4. API 테스트 실패: {e}")

        await browser.close()

    return results

if __name__ == "__main__":
    results = asyncio.run(test_mail_tbe_kr())
    print("\n".join(results))
    print(f"\n총 {sum(1 for r in results if r.startswith(chr(9989)))}/{len(results)} 통과")
```

## 실행
```bash
cd ~/auto-email
pip install playwright 2>/dev/null || pip3 install playwright 2>/dev/null
python3 tests/e2e_browser_test.py
```

## 결과 저장
결과를 아카샤에 `Tester - 브라우저 E2E 결과` 키로 저장해주세요.
