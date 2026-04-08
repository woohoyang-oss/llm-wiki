---
title: eywa MCP 사용 가이드
tags: [eywa, mcp, 가이드, 위키, 도구]
created: 2026-04-07
updated: 2026-04-08
sources: []
---

# eywa MCP 사용 가이드

eywa는 TBNWS 전 직원이 사용하는 통합 MCP 서버입니다.
위키(업무 지식) + 데이터(매출/광고/재고/CS/원가) + 스킬 가이드(워크플로우)를 하나의 MCP로 제공합니다.

## 연결 정보

| 항목 | 주소 |
|------|------|
| HTTP (사내 Wi-Fi) | `http://192.168.1.59:8080/mcp` |
| HTTPS (사내 Wi-Fi) | `https://192.168.1.59:8090/mcp` |
| Tailscale | `https://100.126.144.76:8090/mcp` |
| 도구 수 | **75개** (eywa 9 + 데이터 66) |
| 가이드 수 | **13개** |

---

## 설정 방법

### Claude Desktop (Mac)

설정 파일 위치: `~/Library/Application Support/Claude/claude_desktop_config.json`

```json
{
  "mcpServers": {
    "eywa": {
      "url": "http://192.168.1.59:8080/mcp"
    }
  }
}
```

### Claude Desktop (Windows)

설정 파일 위치: `%APPDATA%\Claude\claude_desktop_config.json`

1. `Win + R` → `%APPDATA%\Claude` 입력 → Enter
2. `claude_desktop_config.json` 파일을 메모장으로 열기 (없으면 새로 생성)
3. 아래 내용 붙여넣기:

```json
{
  "mcpServers": {
    "eywa": {
      "url": "http://192.168.1.59:8080/mcp"
    }
  }
}
```

4. 저장 후 Claude Desktop 재시작

> Windows에서 HTTPS(8090)를 사용하려면 mkcert rootCA.pem 설치가 필요합니다.
> HTTP(8080)는 인증서 없이 바로 사용 가능하므로 사내 네트워크에서는 HTTP를 권장합니다.

### Claude Code (Mac/Windows/Linux 공통)

```bash
claude mcp add --transport http -s user eywa http://192.168.1.59:8080/mcp
```

> `-s user` 옵션은 모든 프로젝트에서 eywa를 사용할 수 있게 합니다.
> 특정 프로젝트에서만 사용하려면 `-s project`로 변경하세요.

### 연결 확인

Claude에게 아래 중 하나를 물어보세요:

- "오늘 매출 알려줘" → `get_sales_overview` 호출되면 정상
- "키크론 위키 검색" → `wiki_search` 호출되면 정상
- "가이드 목록" → `guide_list` 호출되면 정상

---

## 문제 해결 (Windows)

### "서버에 연결할 수 없습니다"
1. 사내 Wi-Fi에 연결되어 있는지 확인 (192.168.1.x 대역)
2. 브라우저에서 `http://192.168.1.59:8080/mcp` 접속 → 응답이 오면 서버 정상
3. Windows 방화벽이 Claude Desktop의 네트워크 접근을 차단하는 경우 → 방화벽 예외 추가

### "도구를 찾을 수 없습니다"
1. Claude Desktop을 완전히 종료 후 재시작 (트레이 아이콘 → 종료)
2. 설정 파일의 JSON 문법 오류 확인 (쉼표, 중괄호)

### HTTPS 사용 시 인증서 오류
사내 CA 인증서를 Windows에 설치해야 합니다:
1. 관리자에게 `rootCA.pem` 파일 요청
2. 파일 더블클릭 → "인증서 설치" → "로컬 컴퓨터" → "신뢰할 수 있는 루트 인증 기관"
3. Claude Desktop 재시작

> 인증서 설치가 번거로우면 HTTP(8080)를 사용하세요. 사내 네트워크이므로 보안 문제 없습니다.

---

## 1. 읽기 (모든 사용자)

### 위키 검색

```
wiki_search(query="키크론", tags=["재고"], type="entities", limit=10)
```

- `query`: 검색어 (필수)
- `tags`: 태그 필터 (선택, 배열)
- `type`: 페이지 타입 — entities, concepts, summaries, org/cs, org/sales 등 (선택)
- `limit`: 결과 수 (기본 20)

**사용 예시:**
- "auto-email 관련 문서 찾아줘" → `wiki_search(query="auto-email")`
- "마케팅 관련 개념 문서" → `wiki_search(query="마케팅", type="concepts")`
- "CS팀 문서 전부" → `wiki_search(query="", type="org/cs")`

### 위키 읽기

```
wiki_read(page_name="auto-email")
```

### 위키 목록

```
wiki_list(type="entities")
```

### 위키 그래프

```
wiki_graph(page_name="tobe-mcp")
```

---

## 2. 쓰기 (모든 사용자)

### 위키 기여 (새 문서 제출)

```
wiki_contribute(
  title="신규 프로젝트 소개",
  content="# 프로젝트 설명\n\n내용...",
  type="entities",
  tags=["프로젝트", "신규"]
)
```

**동작:** `raw/contributions/`에 pending 상태로 저장 → 관리자 승인 후 wiki/로 이동

---

## 3. 수정/삭제 (관리자 전용)

```
wiki_edit(page_name="대상", content="수정된 마크다운", admin_email="wooho.yang@tbnws.com")
wiki_delete(page_name="삭제대상", admin_email="wooho.yang@tbnws.com")
```

---

## 4. 스킬 가이드 (13개 워크플로우)

복합 질문을 받으면 가이드를 먼저 확인하세요.

```
guide_list()              # 전체 가이드 목록
guide_read(skill="morning-brief")  # 상세 읽기
```

| 가이드 | 설명 |
|--------|------|
| `/morning-brief` | 아침 운영 브리핑 |
| `/cs-urgency` | CS 긴급 대응 리스트 |
| `/ad-daily` | 광고 일일 리포트 |
| `/reorder-check` | 발주 의사결정 |
| `/product-health` | 상품 종합 진단 |
| `/weekly-report` | 주간 경영 리포트 |
| `/budget-optimize` | 광고 예산 최적화 |
| `/daily-checkin` | 매니저 업무 체크인 |
| `/pi-risk-check` | PI(발주) 리스크 점검 |
| `/inventory-health` | 재고 건강 종합 점검 |
| `/cash-health` | 현금 흐름 건강 점검 |
| `/brand-scorecard` | 브랜드 성적표 |
| `/period-review` | 기간 비교 리뷰 |

---

## 5. 데이터 도구 (66개)

### 매출 (12개)
| 도구 | 설명 |
|------|------|
| `get_sales_overview` | 전 채널 매출 요약 ★ERP |
| `get_sales_daily` | 일별 매출 추이 |
| `get_sales_by_product` | 상품별 매출 랭킹 |
| `get_sales_by_channel` | 채널별 매출 비교 |
| `get_sales_by_model` | 모델별 매출 집계 |
| `get_sales_by_brand` | 브랜드별 매출 ★ERP |
| `get_product_real_price` | 실거래가 조회 |
| `get_product_daily_sales` | 네이버 상품 일별 판매 |
| `get_switch_preference` | 스위치 선호도 |
| `get_period_comparison` | 전주/전월 비교 ★ERP |
| `get_sales_target_progress` | 목표 달성률 ★ERP |
| `get_channel_sales` | 키워드별 채널 판매 |

### 재고 (14개)
| 도구 | 설명 |
|------|------|
| `get_inventory_current` | 현재 재고 (GT+CJ) |
| `get_inventory_detail` | 개별 상품 재고 상세 |
| `get_inventory_capital` | 재고 자본 (브랜드별) |
| `get_inventory_turnover` | 재고 회전율 |
| `get_daily_export` | G-code 일별 출고 ★NEW |
| `get_reorder_alerts` | 발주 필요 알림 |
| `get_trend_analysis` | 추세 수요예측 ★실출고보강 |
| `get_smart_reorder` | 스마트 발주 제안 |
| `get_purchase_orders` | 발주 현황 |
| `get_sibling_analysis` | 옵션 그룹 불균형 |
| `get_shipment_expedition` | 출하촉진 판단 |
| `get_order_recommendation` | 발주 추천 |
| `get_stale_inventory` | 체류 재고 |
| `get_trend_movers` | 판매 추세 변동 상품 |

### 원가 (5개)
| 도구 | 설명 |
|------|------|
| `get_product_cost` | G-code 원가 조회 |
| `get_products_cost_batch` | 브랜드 전체 원가 |
| `get_product_cost_by_keyword` | 키워드 원가 검색 ★실거래가매칭 |
| `get_margin_analysis` | 마진 분석 |
| `get_fx_sensitivity` | 환율 민감도 |

### 상품분석 (2개)
| 도구 | 설명 |
|------|------|
| `get_product_lifecycle` | 상품 수명주기 (PLC) |
| `get_cannibalization_check` | 카니발라이제이션 감지 |

### CS (12개)
| 도구 | 설명 |
|------|------|
| `get_cs_overview` | CS 종합 현황 |
| `get_unanswered_inquiries` | 미답변 문의 목록 |
| `get_claims_summary` | 클레임 현황 |
| `get_as_status` | AS 처리 현황 |
| `get_as_diagnosis` | AS 고장 패턴 |
| `get_as_products` | AS 다발 상품 |
| `get_inquiry_answer_rate` | 답변율 추이 |
| `get_inquiry_categories` | 카테고리별 문의 |
| `get_claims_rate` | 클레임 비율 |
| `get_problem_products` | 반품 다발 상품 |
| `get_qna_unanswered` | QnA 미답변 |
| `get_cs_workload_forecast` | CS 부하 예측 |

### 광고 SA (4개)
| 도구 | 설명 |
|------|------|
| `get_ad_overview` | SA+GFA 통합 요약 |
| `get_sa_performance` | SA 캠페인 성과 |
| `get_sa_daily` | SA 일별 추이 |
| `get_bizmoney` | 비즈머니 잔액 |

### 광고 GFA 기간 (5개)
| 도구 | 설명 |
|------|------|
| `get_gfa_performance` | GFA 기간 성과 |
| `get_gfa_funnel` | GFA 퍼널 분석 |
| `get_waste_keywords` | 낭비 키워드 |
| `get_top_keywords` | ROAS 상위 키워드 |
| `get_revenue_ad_correlation` | 매출-광고 상관관계 |

### GFA 실시간 (4개)
| 도구 | 설명 |
|------|------|
| `get_gfa_realtime_kpi` | 실시간 KPI |
| `get_gfa_realtime_hourly` | 시간별 누적 |
| `get_gfa_realtime_campaigns` | 캠페인별 실시간 |
| `get_gfa_realtime_alerts` | 실시간 알림 |

### GFA OPS AI (3개) — EC2 전용
| 도구 | 설명 |
|------|------|
| `get_gfa_ops_status` | OPS AI 상태 |
| `get_gfa_ops_scores` | 효율 스코어 |
| `get_gfa_ops_proposals` | 예산 재배분 제안 |

### GFA 대시보드 (3개) — EC2 전용
| 도구 | 설명 |
|------|------|
| `get_gfa_budget_insight` | 예산 조정 인사이트 |
| `get_gfa_fatigue` | 소재 피로도 |
| `get_gfa_waste_creatives` | 낭비 소재 |

### 발주 리스크 (2개)
| 도구 | 설명 |
|------|------|
| `get_po_risk_check` | 발주 리스크 점검 |
| `get_po_brand_summary` | 브랜드별 PO 요약 |

---

## 권한 요약

| 작업 | 일반 사용자 | 관리자 |
|------|:-----------:|:------:|
| 위키 검색/읽기/목록/그래프 | O | O |
| 위키 기여 (새 문서 제출) | O | O |
| 위키 수정/삭제 | X | O |
| 스킬 가이드 조회 | O | O |
| 데이터 도구 (66개) | O | O |

---

## 관련

- [[eywa]] — eywa 오케스트레이션 플랫폼
- [[tobe-mcp]] — ToBe MCP 스킬 서버 (75개 도구)
- [[eywa-dev-guide]] — eywa MCP 개발자 가이드
