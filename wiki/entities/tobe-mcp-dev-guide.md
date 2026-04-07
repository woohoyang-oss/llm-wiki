---
title: ToBe MCP 개발자 가이드
tags: [tobe-mcp, 개발, MCP, Python, aiomysql]
created: 2026-04-07
updated: 2026-04-07
sources: [tobe-mcp-v1-codebase]
---

# ToBe MCP 개발자 가이드

tobe-mcp-v1은 ToBe Networks의 ERP/광고/CS 데이터를 Claude에 노출하는 MCP 도구 서버다. 52개 이상의 도구를 제공하며, [[eywa]]가 이를 직접 임포트하여 통합 MCP 서버로 서빙한다.

---

## 1. 아키텍처

3계층 구조로 관심사를 분리한다.

```
┌─────────────────────┐
│  mcp_server.py      │  ← TOOLS 레지스트리 (튜플 리스트)
│  (엔트리포인트)      │
├─────────────────────┤
│  services/          │  ← 비즈니스 로직 + JSON 직렬화
│  (핸들러 함수)       │
├─────────────────────┤
│  sql/               │  ← 순수 SQL 쿼리 문자열
│  (쿼리 모듈)         │
├─────────────────────┤
│  database.py        │  ← aiomysql 커넥션 풀 (2개 DB)
│  config.py          │  ← pydantic-settings (.env 로드)
└─────────────────────┘
```

- **mcp_server.py**: 도구 정의(이름, 설명, 파라미터, 핸들러)를 TOOLS 리스트로 관리
- **services/**: 각 도메인별 async 핸들러. DB 쿼리 실행, 결과 포맷팅
- **sql/**: 순수 SQL 문자열만 보관. 로직 없음
- **database.py**: TBNWS_ADMIN과 TBNWS_AD 두 개의 aiomysql 풀 관리

---

## 2. 프로젝트 구조

```
tobe-mcp-v1/
├── mcp_server.py              # MCP 엔트리포인트, TOOLS 레지스트리
├── database.py                 # aiomysql 풀 관리 (TBNWS_ADMIN + TBNWS_AD)
├── config.py                   # pydantic-settings (.env)
├── services/
│   ├── sales_service.py        # 매출 도구 8개
│   ├── inventory_service.py    # 재고 도구 11개
│   ├── cost_service.py         # 원가 도구 3개
│   ├── cs_service.py           # CS 도구 11개
│   ├── ad_service.py           # 광고 도구 9개
│   └── gfa_realtime_service.py # GFA 실시간 도구 10개
└── sql/
    ├── sales_queries.py        # 매출 SQL
    ├── inventory_queries.py    # 재고 SQL
    ├── cost_queries.py         # 원가 SQL
    ├── cs_queries.py           # CS SQL
    └── ad_queries.py           # 광고 SQL
```

### 도메인별 도구 수

| 도메인 | 서비스 | 도구 수 | 주요 테이블 |
|--------|--------|---------|------------|
| 매출 | sales_service | 8 | naver_smartstore_orders, erp_sales_daily |
| 재고 | inventory_service | 11 | erp_stock_realtime, erp_product_info |
| 원가 | cost_service | 3 | erp_purchase_order, erp_cost_master |
| CS | cs_service | 11 | cs_tickets, cs_categories |
| 광고 | ad_service | 9 | naver_ad_stats, ad_campaigns |
| GFA | gfa_realtime_service | 10 | gfa_realtime, gfa_campaigns |

---

## 3. 도구 등록 패턴

모든 도구는 `TOOLS` 리스트에 5-튜플로 등록한다.

```python
TOOLS: list[tuple] = [
    (
        "get_daily_sales",                    # name: 도구 이름
        "일별 매출 조회 (기간 지정)",            # description: LLM이 읽는 설명
        {                                      # params: JSON Schema properties
            "start_date": {
                "type": "string",
                "description": "시작일 (YYYY-MM-DD)"
            },
            "end_date": {
                "type": "string",
                "description": "종료일 (YYYY-MM-DD)"
            }
        },
        ["start_date"],                        # required: 필수 파라미터
        sales_service.get_daily_sales          # handler: async callable
    ),
    # ... 나머지 도구들
]
```

**규칙:**
- `name`은 snake_case, 동사로 시작 (get_, search_, list_, count_)
- `description`은 한국어, LLM이 도구 선택에 사용하므로 구체적으로
- `params`는 JSON Schema 형식. type + description 필수
- `required`는 리스트. 빈 리스트 `[]`도 가능 (모든 파라미터가 선택적)
- `handler`는 `async def fn(args: dict) -> str` 시그니처

---

## 4. 서비스 핸들러 패턴

모든 핸들러는 동일한 패턴을 따른다.

```python
async def get_daily_sales(args: dict) -> str:
    """일별 매출 조회"""
    # 1. 파라미터 추출 + 기본값
    start_date, end_date = _default_dates(
        args.get("start_date"),
        args.get("end_date"),
        days=30
    )

    # 2. SQL 실행
    rows = await fetch_all(
        SALES_BY_DATE,          # sql/ 모듈의 쿼리 상수
        (start_date, end_date), # %s 파라미터 바인딩
        db="admin"              # "admin" = TBNWS_ADMIN, "ad" = TBNWS_AD
    )

    # 3. 결과 반환 (JSON 문자열)
    return _json({
        "period": f"{start_date} ~ {end_date}",
        "count": len(rows),
        "data": rows
    })
```

### 헬퍼 함수

| 함수 | 용도 |
|------|------|
| `_default_dates(start, end, days=30)` | start/end가 None이면 오늘 기준 days일 전~오늘 |
| `_json(obj)` | `json.dumps(obj, ensure_ascii=False, default=str)` |
| `fetch_all(query, params, db)` | DictCursor로 전체 행 반환 |
| `fetch_one(query, params, db)` | DictCursor로 단일 행 반환 |

### 주의사항
- 핸들러는 반드시 `str`을 반환해야 한다 (MCP 프로토콜 요구사항)
- 예외 발생 시 `_json({"error": str(e)})`로 감싸서 반환
- datetime, Decimal 등 직렬화 불가 타입은 `_json`의 `default=str`이 처리

---

## 5. SQL 쿼리 규칙

### 파라미터 바인딩

```python
# sql/sales_queries.py
SALES_BY_DATE = """
    SELECT DATE(order_date) as date,
           SUM(payment_amount) as total
    FROM naver_smartstore_orders
    WHERE order_date BETWEEN %s AND %s
    GROUP BY DATE(order_date)
    ORDER BY date
"""
```

- **항상 `%s` 플레이스홀더 사용** (aiomysql/pymysql 규약)
- f-string이나 format으로 SQL을 조합하지 않는다 (SQL injection 방지)
- 쿼리 상수명은 UPPER_SNAKE_CASE

### 데이터베이스 구분

| DB 이름 | 별칭 | 용도 | 주요 테이블 |
|---------|------|------|------------|
| TBNWS_ADMIN | `"admin"` | ERP/주문/재고/CS | erp_product_info, erp_stock_realtime, naver_smartstore_orders |
| TBNWS_AD | `"ad"` | 광고/GFA | naver_ad_stats, gfa_realtime, gfa_campaigns |

### DictCursor

모든 쿼리는 `aiomysql.DictCursor`로 실행되어 결과가 `list[dict]` 형태로 반환된다. 컬럼명이 딕셔너리 키가 되므로 SELECT 절에서 의미 있는 alias를 지정할 것.

---

## 6. 새 도구 추가 절차

### Step 1: SQL 쿼리 작성 (sql/)

```python
# sql/inventory_queries.py에 추가
STOCK_BY_WAREHOUSE = """
    SELECT warehouse_code, product_code,
           SUM(stock_qty) as total_qty
    FROM erp_stock_realtime
    WHERE warehouse_code = %s
    GROUP BY warehouse_code, product_code
"""
```

### Step 2: 서비스 핸들러 작성 (services/)

```python
# services/inventory_service.py에 추가
async def get_stock_by_warehouse(args: dict) -> str:
    warehouse = args.get("warehouse_code", "WH01")
    rows = await fetch_all(STOCK_BY_WAREHOUSE, (warehouse,), db="admin")
    return _json({"warehouse": warehouse, "items": rows, "count": len(rows)})
```

### Step 3: TOOLS 등록 (mcp_server.py)

```python
TOOLS.append((
    "get_stock_by_warehouse",
    "창고별 재고 현황 조회",
    {
        "warehouse_code": {
            "type": "string",
            "description": "창고 코드 (예: WH01)"
        }
    },
    [],
    inventory_service.get_stock_by_warehouse
))
```

### Step 4: eywa 자동 반영

eywa-mcp는 시작 시 `from mcp_server import TOOLS`로 전체 도구를 임포트하므로 별도 배포나 등록 작업 없이 eywa 재시작만 하면 새 도구가 반영된다.

---

## 7. DB 스키마 핵심 테이블

### TBNWS_ADMIN 주요 테이블

| 테이블 | 설명 | 핵심 컬럼 |
|--------|------|----------|
| `erp_product_info` | 상품 마스터 (G-code) | product_code, product_name, brand, category |
| `erp_stock_realtime` | 실시간 재고 | product_code, warehouse_code, stock_qty |
| `naver_smartstore_orders` | 네이버 주문 | order_id, order_date, payment_amount, product_code |
| `erp_purchase_order` | 발주(PO) | po_number, supplier, order_date, status |
| `erp_cost_master` | 원가 마스터 | product_code, purchase_price, selling_price |
| `cs_tickets` | CS 티켓 | ticket_id, category, status, created_at |

### TBNWS_AD 주요 테이블

| 테이블 | 설명 | 핵심 컬럼 |
|--------|------|----------|
| `naver_ad_stats` | 네이버 광고 통계 | campaign_id, date, impressions, clicks, cost |
| `gfa_realtime` | GFA 실시간 데이터 | campaign_id, timestamp, spend, impressions |
| `gfa_campaigns` | GFA 캠페인 마스터 | campaign_id, campaign_name, status, budget |

### G-code 체계

상품 코드는 G-code 체계를 따른다. 자세한 내용은 [[product-code-hierarchy]] 참조.

---

## 8. eywa 연동

[[eywa]]의 MCP 서버(eywa-mcp)는 tobe-mcp-v1을 별도 프로세스가 아닌 **같은 프로세스에서 직접 임포트**한다.

```python
# eywa-mcp/eywa_mcp/server.py
import sys
sys.path.insert(0, settings.TOBE_MCP_PATH)  # /path/to/tobe-mcp-v1
from mcp_server import TOOLS as DATA_TOOLS
```

이 방식의 장점:
- **배포 단순화**: tobe-mcp-v1을 별도로 실행/관리할 필요 없음
- **네트워크 오버헤드 제거**: 프로세스 내 함수 호출
- **도구 자동 동기화**: TOOLS 리스트 변경 시 eywa 재시작만 하면 반영

주의:
- tobe-mcp-v1의 의존성(aiomysql, pydantic-settings 등)이 eywa-mcp 환경에도 설치되어 있어야 함
- `.env` 파일은 eywa-mcp 루트에서 읽힘 (tobe-mcp-v1의 config.py가 참조)

---

## 9. 테스트

### MCP Inspector 사용

```bash
# 1. MCP Inspector 실행
npx @anthropic/mcp-inspector

# 2. 서버 연결 (stdio 모드)
# Command: python mcp_server.py
# Working directory: /path/to/tobe-mcp-v1

# 3. tools/list로 등록된 도구 확인
# → 52개 이상의 도구가 목록에 표시되어야 함

# 4. call_tool로 개별 도구 테스트
# Tool: get_daily_sales
# Arguments: {"start_date": "2026-04-01", "end_date": "2026-04-07"}
```

### 핸들러 단위 테스트

```python
import asyncio
from services.sales_service import get_daily_sales

result = asyncio.run(get_daily_sales({
    "start_date": "2026-04-01",
    "end_date": "2026-04-07"
}))
print(result)  # JSON 문자열
```

---

## 10. 관련 문서

- [[tobe-mcp]] — ToBe MCP 프로젝트 개요 및 도구 목록
- [[eywa]] — Eywa 오케스트레이션 플랫폼
- [[eywa-mcp-guide]] — eywa MCP 사용 가이드
- [[eywa-dev-guide]] — eywa MCP 개발자 가이드
- [[product-code-hierarchy]] — G-code 상품 코드 체계
- [[sales-data-pipeline]] — 매출 데이터 파이프라인
