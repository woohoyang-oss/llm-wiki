# ToBe Networks Data API — 인증 가이드

> API Key 발급, 사용법, 스코프 권한 관리 문서
> 최종 업데이트: 2026-03-01

---

## 1. 인증 구조 요약

```
[사용자] ──Google SSO──▶ [키 발급 포털] ──▶ API Key 발급
                         @tbnws.com만              ↓
                                          tbnws_xxxx... (38자)
                                                   ↓
[앱/스크립트] ──X-API-Key 헤더──▶ [Data API] ──▶ 데이터 반환
                                     ↑
                              SQLite에서 해시 검증 + 스코프 확인
```

| 항목 | 내용 |
|------|------|
| 포털 로그인 | Google OAuth 2.0 SSO (`@tbnws.com` 계정만) |
| API 인증 | `X-API-Key` 헤더 |
| Key 저장 | SHA256 해시로만 저장 (원본 복구 불가) |
| Key 형식 | `tbnws_` + 32자 hex = 총 38자 |
| 스코프 | `sales`, `inventory`, `cost`, `cs`, `advertising` |
| MCP | 인증 제외 (로컬 stdio 프로세스) |

---

## 2. API Key 발급 방법

### 2-1. 포털 로그인

1. 브라우저에서 `http://localhost:8100/auth/login` 접속
2. "Google 계정으로 로그인" 클릭
3. `@tbnws.com` Google 계정으로 인증

> **주의**: `@tbnws.com` 이메일만 접근 가능합니다.

### 2-2. 키 발급

1. 로그인 후 대시보드(`/auth/dashboard`)로 자동 이동
2. "새 API Key 발급" 섹션에서 키 용도 입력 (예: `마케팅팀 분석용`)
3. "발급" 클릭
4. **화면에 표시되는 API Key를 반드시 복사하여 안전하게 보관**

> **경고**: 발급된 키는 **이 화면에서 한 번만** 표시됩니다.
> DB에는 해시값만 저장되므로, 키를 잃어버리면 새로 발급해야 합니다.

### 2-3. 키 폐기

- 대시보드에서 해당 키의 "폐기" 버튼 클릭
- 폐기된 키는 즉시 비활성화되며 복구할 수 없음

---

## 3. API Key 사용법

### 3-1. HTTP 헤더

모든 데이터 API 호출 시 `X-API-Key` 헤더를 포함합니다.

**curl 예시:**

```bash
curl -H "X-API-Key: tbnws_your_key_here" \
  "http://localhost:8100/api/v1/sales/overview?account=keychron&start_date=2026-01-01&end_date=2026-02-01"
```

**Python 예시:**

```python
import requests

API_KEY = "tbnws_your_key_here"
BASE_URL = "http://localhost:8100/api/v1"

# 매출 요약
r = requests.get(
    f"{BASE_URL}/sales/overview",
    headers={"X-API-Key": API_KEY},
    params={
        "account": "keychron",
        "start_date": "2026-01-01",
        "end_date": "2026-02-01",
    },
)
print(r.json())
```

**JavaScript (fetch) 예시:**

```javascript
const API_KEY = "tbnws_your_key_here";
const BASE_URL = "http://localhost:8100/api/v1";

const res = await fetch(
  `${BASE_URL}/sales/overview?account=keychron&start_date=2026-01-01&end_date=2026-02-01`,
  { headers: { "X-API-Key": API_KEY } }
);
const data = await res.json();
console.log(data);
```

### 3-2. Swagger UI에서 테스트

1. `http://localhost:8100/docs` 접속
2. 우측 상단 "Authorize" 클릭
3. `X-API-Key` 입력란에 키 붙여넣기
4. 각 엔드포인트에서 "Try it out" → "Execute"

---

## 4. 인증 범위 (스코프)

### 4-1. 스코프 목록

| 스코프 | 도메인 | 접근 엔드포인트 |
|--------|--------|----------------|
| `sales` | 매출 | `/api/v1/sales/*` |
| `inventory` | 재고 | `/api/v1/inventory/*` |
| `cost` | 원가 | `/api/v1/cost/*` |
| `cs` | 고객서비스 | `/api/v1/cs/*` |
| `advertising` | 광고 | `/api/v1/advertising/*` |
| `*` | 전체 | 위 모든 도메인 |

### 4-2. 초기 정책

- 모든 신규 발급 키는 **`*` (전체 권한)** 으로 생성
- 추후 어드민 포털에서 키별 스코프 제한 가능

### 4-3. 스코프 제한 시 동작

| 요청 | 키 스코프 | 결과 |
|------|-----------|------|
| `GET /api/v1/sales/overview` | `["*"]` | ✅ 200 OK |
| `GET /api/v1/sales/overview` | `["sales"]` | ✅ 200 OK |
| `GET /api/v1/sales/overview` | `["inventory"]` | ❌ 403 Forbidden |
| `GET /api/v1/inventory/current` | `["sales", "inventory"]` | ✅ 200 OK |

---

## 5. 인증 불필요 엔드포인트

아래 엔드포인트는 **API Key 없이** 접근 가능합니다.

| 엔드포인트 | 설명 |
|-----------|------|
| `GET /health` | 서버 상태 확인 |
| `GET /docs` | Swagger UI |
| `GET /redoc` | ReDoc |
| `GET /api/v1/context/*` | AI 컨텍스트 (도메인별 시스템 프롬프트) |
| `GET /api/v1/tools/*` | AI Tool 정의 (Claude/GPT 호출 스펙) |
| `GET /auth/login` | 로그인 포털 |

---

## 6. 에러 응답

| HTTP 코드 | 상황 | 응답 |
|-----------|------|------|
| **401** | 키 없음 | `{"detail": "API Key가 필요합니다. /auth/dashboard 에서 발급받으세요."}` |
| **401** | 잘못된 키 | `{"detail": "유효하지 않은 API Key입니다."}` |
| **401** | 비활성 키 | `{"detail": "비활성화된 API Key입니다. 새 키를 발급받으세요."}` |
| **401** | 만료된 키 | `{"detail": "만료된 API Key입니다."}` |
| **403** | 권한 없음 | `{"detail": "이 API Key는 'inventory' 도메인 접근 권한이 없습니다. 허용된 도메인: sales"}` |

---

## 7. 전체 엔드포인트 목록

### 매출 (Sales) — `X-API-Key` + `sales` 스코프

| Method | 엔드포인트 | 설명 |
|--------|-----------|------|
| GET | `/api/v1/sales/overview` | 매출 요약 (총매출, 주문수, AOV) |
| GET | `/api/v1/sales/daily` | 일별 매출 추이 |
| GET | `/api/v1/sales/products` | 상품별 매출 랭킹 |
| GET | `/api/v1/sales/products/price` | 상품 실거래가 조회 |
| GET | `/api/v1/sales/channels` | 채널별 매출 |
| GET | `/api/v1/sales/models` | 모델별 매출 |

### 재고 (Inventory) — `X-API-Key` + `inventory` 스코프

| Method | 엔드포인트 | 설명 |
|--------|-----------|------|
| GET | `/api/v1/inventory/current` | 현재 재고 목록 (GT+CJ 합산) |
| GET | `/api/v1/inventory/current/{product_code}` | 상품 재고 상세 |
| GET | `/api/v1/inventory/reorder-alerts` | 발주 필요 재고 |
| GET | `/api/v1/inventory/smart-reorder` | 스마트 발주 알림 (노이즈 필터링) |
| GET | `/api/v1/inventory/trend-analysis` | 추세 가중 수요예측 |
| GET | `/api/v1/inventory/purchase-orders` | 발주 현황 |

### 원가 (Cost) — `X-API-Key` + `cost` 스코프

| Method | 엔드포인트 | 설명 |
|--------|-----------|------|
| GET | `/api/v1/cost/product/{product_code}` | 상품 원가 조회 (KRW/USD/VAT) |
| GET | `/api/v1/cost/products` | 브랜드 상품 원가 일괄 |
| GET | `/api/v1/cost/margin` | 상품별 마진 분석 |

### CS (고객서비스) — `X-API-Key` + `cs` 스코프

| Method | 엔드포인트 | 설명 |
|--------|-----------|------|
| GET | `/api/v1/cs/overview` | CS 종합 현황 (4대 축) |
| GET | `/api/v1/cs/inquiries/unanswered` | 미답변 문의 목록 |
| GET | `/api/v1/cs/inquiries/answer-rate` | 월별 답변율 |
| GET | `/api/v1/cs/inquiries/categories` | 카테고리별 문의 |
| GET | `/api/v1/cs/claims` | 클레임 현황 |
| GET | `/api/v1/cs/claims/rate` | 클레임 비율 |
| GET | `/api/v1/cs/claims/problem-products` | 반품 다발 상품 |
| GET | `/api/v1/cs/as/status` | AS 처리율 |
| GET | `/api/v1/cs/as/diagnosis` | AS 고장 패턴 |
| GET | `/api/v1/cs/as/products` | AS 다발 상품 |
| GET | `/api/v1/cs/qna/unanswered` | QnA 미답변 |

### 광고 (Advertising) — `X-API-Key` + `advertising` 스코프

| Method | 엔드포인트 | 설명 |
|--------|-----------|------|
| GET | `/api/v1/advertising/overview` | 광고 통합 성과 (SA+GFA) |
| GET | `/api/v1/advertising/sa/performance` | SA 성과 |
| GET | `/api/v1/advertising/sa/daily` | SA 일별 추이 |
| GET | `/api/v1/advertising/gfa/performance` | GFA 성과 |
| GET | `/api/v1/advertising/gfa/funnel` | GFA 퍼널 분석 |
| GET | `/api/v1/advertising/keywords/top` | 효율 키워드 Top |
| GET | `/api/v1/advertising/keywords/waste` | 낭비 키워드 |
| GET | `/api/v1/advertising/bizmoney` | 비즈머니 잔액 |
| GET | `/api/v1/advertising/revenue-correlation` | 매출-광고 상관 |

### AI 컨텍스트 / 도구 — 인증 불필요

| Method | 엔드포인트 | 설명 |
|--------|-----------|------|
| GET | `/api/v1/context/` | 전체 도메인 목록 |
| GET | `/api/v1/context/{domain}` | 도메인별 AI 컨텍스트 |
| GET | `/api/v1/tools/claude` | Claude tool_use 정의 |
| GET | `/api/v1/tools/openai` | GPT function_calling 정의 |
| GET | `/api/v1/tools/generic` | 범용 tool 정의 |
| GET | `/api/v1/tools/domains` | 사용 가능한 도메인 목록 |
| GET | `/api/v1/tools/system-prompt` | AI 시스템 프롬프트 |
| GET | `/api/v1/tools/api-map` | Tool name → API URL 매핑 |

### 인증 포털

| Method | 엔드포인트 | 설명 |
|--------|-----------|------|
| GET | `/auth/login` | 로그인 페이지 |
| GET | `/auth/login/google` | Google OAuth 시작 |
| GET | `/auth/callback` | Google OAuth 콜백 |
| GET | `/auth/logout` | 로그아웃 |
| GET | `/auth/dashboard` | API Key 관리 대시보드 |
| POST | `/auth/keys/create` | API Key 발급 |
| POST | `/auth/keys/{key_id}/revoke` | API Key 폐기 |

---

## 8. 부서별 활용 시나리오

### 마케팅팀
```python
# 키 발급 시 이름: "마케팅팀 분석용"
# 추후 스코프 제한 가능: ["sales", "advertising"]

import requests

API_KEY = "tbnws_xxxx..."
BASE = "http://localhost:8100/api/v1"
HEADERS = {"X-API-Key": API_KEY}

# 이번 달 매출 확인
sales = requests.get(f"{BASE}/sales/overview",
    headers=HEADERS,
    params={"account": "keychron", "start_date": "2026-02-01", "end_date": "2026-03-01"}
).json()

# 광고 ROAS 확인
ads = requests.get(f"{BASE}/advertising/overview",
    headers=HEADERS,
    params={"start_date": "2026-02-01", "end_date": "2026-03-01"}
).json()

print(f"매출: {sales['data']['total_revenue']:,.0f}원")
print(f"광고비: {ads['data']['total_cost']:,.0f}원")
```

### SCM / 물류팀
```python
# 키 발급 시 이름: "SCM 재고관리용"
# 추후 스코프 제한 가능: ["inventory", "cost"]

# 발주 필요 재고 확인
alerts = requests.get(f"{BASE}/inventory/smart-reorder",
    headers=HEADERS,
    params={"brand_code": "KEYCHRON", "days_threshold": 14}
).json()

for item in alerts['data']['items'][:5]:
    print(f"{item['goods_name']} - 잔여일수: {item['days_of_supply']}일, 조치: {item['action']}")
```

### CS팀
```python
# 키 발급 시 이름: "CS팀 대시보드용"
# 추후 스코프 제한 가능: ["cs"]

# 미답변 문의 확인
unanswered = requests.get(f"{BASE}/cs/inquiries/unanswered",
    headers=HEADERS
).json()

print(f"미답변 문의: {unanswered['count']}건")
```

---

## 9. 보안 참고

| 항목 | 설명 |
|------|------|
| 키 저장 | SHA256 해시만 DB에 저장 — 원본 노출 불가 |
| 세션 관리 | 포털 세션은 httponly 쿠키 JWT (1시간 만료) |
| 도메인 제한 | `@tbnws.com` 이메일만 포털 접근 가능 |
| 키 폐기 | 즉시 비활성화, API 호출 불가 |
| 마이그레이션 | `.env`의 `SECRET_KEY`를 비우면 인증 비활성화 (개발용) |
| MCP | 로컬 stdio 프로세스 — 네트워크 인증 불필요 |

---

## 10. FAQ

**Q: 키를 잃어버렸어요**
→ 복구 불가. 대시보드에서 새 키를 발급받고, 기존 키는 폐기하세요.

**Q: 다른 사람에게 키를 공유해도 되나요?**
→ 권장하지 않습니다. 각자 포털에서 본인 키를 발급받아 사용하세요.

**Q: 키 만료 기한이 있나요?**
→ 현재는 무제한. 추후 어드민 포털에서 만료일 설정 가능.

**Q: MCP 서버에서도 키가 필요한가요?**
→ 아니요. MCP는 로컬 stdio 프로세스로 실행되어 인증이 불필요합니다.

**Q: Swagger UI에서 어떻게 테스트하나요?**
→ `/docs` 접속 → 우측 상단 "Authorize" 클릭 → 키 입력 후 사용.
