---
title: eywa MCP 사용 가이드
tags: [eywa, mcp, 가이드, 위키, 도구]
created: 2026-04-07
updated: 2026-04-07
sources: []
---

# eywa MCP 사용 가이드

eywa는 TBNWS 전 직원이 사용하는 통합 MCP 서버입니다.
위키(업무 지식) + 데이터(매출/광고/재고/CS) + 스킬 가이드(워크플로우)를 하나의 MCP로 제공합니다.

## 연결 정보

- **서버 주소**: `https://192.168.1.59:8090/mcp` (사내 Wi-Fi)
- **Tailscale**: `https://100.126.144.76:8090/mcp`
- **도구 수**: 53개 (eywa 9 + 데이터 44)

### Claude Desktop 설정

`claude_desktop_config.json`에 추가:

```json
{
  "mcpServers": {
    "eywa": {
      "url": "https://192.168.1.59:8090/mcp"
    }
  }
}
```

### Claude Code 설정

```bash
claude mcp add eywa --transport streamable-http https://192.168.1.59:8090/mcp
```

> HTTPS 인증서는 사내 CA(mkcert)로 발급. 신뢰하려면 관리자에게 rootCA.pem 요청.

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

- `page_name`: 페이지 이름 (kebab-case, 확장자 제외)

### 위키 목록

```
wiki_list(type="entities")
```

- `type`: 페이지 타입 필터 (선택). 생략 시 전체 목록.

### 위키 그래프

```
wiki_graph(page_name="tobe-mcp")
```

페이지의 outgoing 링크(이 페이지가 참조하는 페이지)와 incoming 링크(이 페이지를 참조하는 페이지)를 반환합니다.

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

- `title`: 문서 제목 (필수)
- `content`: 마크다운 본문 (필수)
- `type`: 페이지 타입 — entities, concepts, summaries 등 (기본: concepts)
- `tags`: 태그 목록 (선택, 배열)

**동작:**
1. `raw/contributions/` 폴더에 `YYYY-MM-DD--제목.md` 파일 생성
2. status: pending 상태로 저장
3. 관리자 승인 후 wiki/ 폴더로 이동하여 검색 가능

> 누구나 기여할 수 있습니다. 직접 wiki/ 폴더를 수정하지 않으므로 안전합니다.

---

## 3. 수정 (관리자 전용)

### 위키 수정

```
wiki_edit(
  page_name="auto-email",
  content="---\ntitle: Auto-Email\ntags: [이메일, AI]\n...\n---\n\n수정된 본문",
  admin_email="wooho.yang@tbnws.com"
)
```

- `page_name`: 수정할 페이지 이름 (필수)
- `content`: 수정된 전체 마크다운 (frontmatter 포함, 필수)
- `admin_email`: 관리자 이메일 (필수)

**권한:**
- 현재 관리자: `wooho.yang@tbnws.com`
- 관리자가 아닌 이메일로 호출 시 거부됨

**권장 워크플로우:**
1. `wiki_read(page_name="대상페이지")` 로 현재 내용 확인
2. 내용 수정
3. `wiki_edit(...)` 으로 저장
4. 수정 즉시 인덱스 리빌드 → 검색에 반영

---

## 4. 삭제 (관리자 전용)

### 위키 삭제

```
wiki_delete(
  page_name="삭제할-페이지",
  admin_email="wooho.yang@tbnws.com"
)
```

- `page_name`: 삭제할 페이지 이름 (필수)
- `admin_email`: 관리자 이메일 (필수)

**동작:**
1. 해당 .md 파일 삭제
2. 인덱스 리빌드
3. 다른 페이지에서 `[[삭제된-페이지]]` 링크가 있으면 깨진 링크가 됨 → 관리자가 정리 필요

---

## 5. 스킬 가이드 (복합 워크플로우)

eywa는 7개의 스킬 가이드를 제공합니다. 복합 질문을 받으면 가이드를 먼저 확인하세요.

### 가이드 목록 조회

```
guide_list()
```

### 가이드 상세 읽기

```
guide_read(skill="morning-brief")
```

**사용 가능한 스킬:**

| 스킬 | 설명 |
|------|------|
| morning-brief | 오전 브리핑 (매출+광고+CS 종합) |
| cs-urgency | CS 긴급도 분류 + 미답변 현황 |
| ad-daily | 광고 일일 리포트 (SA+GFA) |
| reorder-check | 재고/발주 점검 |
| product-health | 상품 건강도 분석 |
| weekly-report | 주간 종합 리포트 |
| budget-optimize | 광고 예산 최적화 |

가이드에는 **어떤 데이터 도구를 어떤 순서로 호출할지** 상세 지침이 포함되어 있습니다.

---

## 6. 데이터 도구 (44개)

eywa 뒤에 숨겨진 tobe-mcp-v1의 데이터 도구입니다. 직접 호출할 수 있습니다.

### 매출 (7개)
`SALES_OVERVIEW`, `SALES_DAILY`, `SALES_BY_PRODUCT`, `SALES_BY_CHANNEL`, `ORDER_DETAIL`, `CLAIMS_STATUS`, `REFUND_ANALYSIS`

### 광고 (9개)
`SA_CAMPAIGN_DAILY`, `SA_KEYWORD_DAILY`, `GFA_CAMPAIGN_DAILY`, `AD_SPEND_SUMMARY`, `BIZMONEY_BALANCE`, `AD_PERFORMANCE_COMPARE`, `WASTE_KEYWORDS`, `TOP_KEYWORDS`, `AD_SALES_CORRELATION`

### GFA 실시간 (10개)
`GFA_REALTIME_KPI`, `GFA_HOURLY_TREND`, `GFA_CAMPAIGN_STATUS`, `GFA_GOAL_PROGRESS`, `GFA_FUNNEL_ANALYSIS`, `GFA_7DAY_HISTORY`, `GFA_DAILY_BUDGET_PACE`, `GFA_ANOMALY_CHECK`, `GFA_ACCOUNT_COMPARE`, `GFA_TIME_SLOT_ANALYSIS`

### 재고 (5개)
`CURRENT_STOCK`, `STOCK_MOVEMENT`, `REORDER_ALERT`, `PURCHASE_ORDERS`, `EXPORT_SCHEDULE`

### 원가 (3개)
`PRODUCT_COST`, `MARGIN_ANALYSIS`, `COST_TREND`

### CS (10개)
`UNANSWERED_INQUIRIES`, `UNANSWERED_QNA`, `MONTHLY_RESPONSE_RATE`, `INQUIRY_TREND`, `CS_KEYWORD_ANALYSIS`, `CS_PRODUCT_ISSUES`, `AS_STATUS`, `AS_BACKLOG`, `AS_DIAGNOSIS_PATTERN`, `AS_PRODUCT_RANKING`

---

## 권한 요약

| 작업 | 일반 사용자 | 관리자 (wooho.yang@tbnws.com) |
|------|:-----------:|:----------------------------:|
| 위키 검색/읽기/목록/그래프 | O | O |
| 위키 기여 (새 문서 제출) | O | O |
| 위키 수정 | X | O |
| 위키 삭제 | X | O |
| 스킬 가이드 조회 | O | O |
| 데이터 도구 (44개) | O | O |

---

## 관련 시스템

- [[eywa]] — eywa 오케스트레이션 플랫폼 (오픈소스)
- [[tobe-mcp]] — ToBe MCP 스킬 서버 (데이터 44개 도구)
- [[skills-overview]] — 스킬 가이드 요약
