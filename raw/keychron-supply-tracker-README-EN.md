# keychron-supply-tracker

**Keychron Purchase Order Tracking & Supply Chain Sync Service**

A polling-based system that keeps regional partners (e.g. ToBe Networks, Korea) in sync with Keychron HQ's purchase order statuses — down to the individual model number level.

---

## 1. Problem Statement

Today, Keychron HQ staff update each PO's status daily in an internal system. Regional partners have no automated way to consume those updates. As a result:

- Schedule slips (ETP / ETD delays) are discovered late.
- Inventory flow decisions (expediting, partial shipments) are reactive instead of proactive.
- Manual follow-ups consume time on both sides.

## 2. Goals

| # | Goal | Measure |
|---|------|---------|
| G1 | Near-real-time visibility into PO status per model | Partner sees updates within 1 polling cycle (configurable, default 4 hrs) |
| G2 | Automatic delay detection | Alert fired within 1 cycle when ETP/ETD slips >= N days |
| G3 | Partial shipment awareness | System flags when models within the same PO diverge in schedule |
| G4 | Audit trail | Full history of every status/date change per model per PO |

## 3. Stakeholders

| Role | Party | Responsibility |
|------|-------|---------------|
| Data Provider | Keychron HQ | Maintain PO status, expose API |
| Data Consumer | ToBe Networks (KR) | Run polling service, act on alerts |
| Future Consumers | Other regional partners | Same API, separate credentials |

## 4. System Architecture

```
Keychron HQ
  Internal PO System (daily manual update)
    -> REST API /api/v1/pos (Auth: API Key / OAuth)
       |
       | HTTPS (polling)
       v
Partner Polling Service (ToBe)
  Scheduler (cron/APScheduler)
    -> Sync Engine
       1. Fetch POs
       2. Diff vs DB
       3. Store snapshot
       4. Fire alerts
    -> Local DB (PostgreSQL)
       - po_headers
       - po_line_items
       - change_log
    -> Notification Layer
       - Slack
       - KakaoTalk
       - Notion DB
       - Web Dashboard
```

## 5. Data Model

### 5.1 PO Header
- po_number (PK): Purchase order number, e.g. PO-2026-0042
- po_date: Date PO was placed
- overall_status (enum): Derived from slowest item status
- total_models: Count of distinct model lines
- remarks: PO-level notes
- created_at / updated_at: timestamps

### 5.2 PO Line Item (per model number)
- po_number (FK -> po_headers)
- line_no: Line sequence
- model_number: Keychron model code, e.g. K10P-RD9
- model_name: Human-readable name
- qty_ordered / qty_produced: Units ordered vs completed
- status (enum): Current production status
- etp: Est. production completion date
- etd: Est. departure from origin
- eta: Est. arrival at partner
- remarks: Model-level notes
- updated_at: HQ last update time

### 5.3 Change Log
- id (PK serial)
- po_number, model_number (nullable for PO-level)
- field_name: Which field changed
- old_value / new_value
- changed_at: When detected

### 5.4 Status Enum (proposed - confirm with HQ)
Draft -> Confirmed -> Waiting Parts -> In Production -> QC -> Packed -> Shipped -> Delivered

## 6. System Flow

### 6.1 Polling Cycle
Scheduler triggers -> Call HQ API -> Parse response -> Diff vs local DB -> Store changes
  If changes found -> Fire alerts
  If no changes -> Log "no change"

### 6.2 Alert Decision Tree (per model, per PO)
- Status changed? -> Notify
- ETP moved back >= 3 days? -> DELAY ALERT
- ETD moved back >= 3 days? -> DELAY ALERT
- No update for >= 3 days? -> STALE ALERT
- Schedule gap vs other models in same PO >= 10 days? -> PARTIAL SHIPMENT SUGGESTION

### 6.3 Partial Shipment Flow
When models in same PO have ETP gap >= 10 days:
  System suggests shipping completed models first
  Example: PO-0042 has K8P-WH1 (ETP 4/10 done), K10P-RD9 (ETP 4/20), K10P-BL3 (ETP 4/25)
  -> Suggest shipping K8P-WH1 first

## 7. API Specification (Proposed for HQ)

### 7.1 Endpoints
- GET /api/v1/pos — List all POs for authenticated partner
- GET /api/v1/pos/{po_number} — Single PO with all line items
- GET /api/v1/pos?updated_since={ISO8601} — Incremental sync

### 7.2 Authentication
- Option A (recommended MVP): API Key in header (X-API-Key)
- Option B: OAuth 2.0 client credentials
- Option C: Basic Auth over HTTPS

### 7.3 Response Format (JSON)
```json
{
  "partner_id": "tobe-kr",
  "fetched_at": "2026-03-30T09:00:00Z",
  "purchase_orders": [
    {
      "po_number": "PO-2026-0042",
      "po_date": "2026-03-15",
      "items": [
        {
          "line_no": 1,
          "model_number": "K10P-RD9",
          "model_name": "Keychron K10 Pro Red",
          "qty_ordered": 500,
          "qty_produced": 320,
          "status": "In Production",
          "etp": "2026-04-20",
          "etd": "2026-04-28",
          "eta": "2026-05-05",
          "remarks": ""
        }
      ]
    }
  ]
}
```

### 7.4 Error Responses
- 401: Invalid/missing API key
- 403: Not authorized for requested PO
- 404: PO not found
- 429: Rate limit exceeded
- 500: Internal server error

## 8. Questions for Keychron HQ

### API & System
Q1. Does an API already exist, or needs to be built?
Q2. What system manages PO data? (ERP, custom, spreadsheet?)
Q3. Preferred authentication method?
Q4. Rate limit on API calls?
Q5. Can responses be filtered by updated_since?

### Data & Business Logic
Q6. Exact status codes used internally?
Q7. Model number format standardized? (e.g. K10P-RD9)
Q8. Partial shipments supported?
Q9. Can qty_ordered change after confirmation?
Q10. Will other partners use this system?

## 9. Tech Stack (Partner Side)
- Python 3.11+ / FastAPI
- APScheduler
- PostgreSQL
- Slack Webhook, Notion API, KakaoTalk (ToBe MSG)
- Docker on Tailscale network

## 10. MVP Scope
Phase 1 (Week 1-2): API client, scheduler, DB schema, diff engine, change log
Phase 2 (Week 2-3): Slack alerts, delay detection, stale PO detection, partial shipment suggestion
Phase 3 (Week 3-4): Web dashboard, Notion sync, Gantt timeline, historical reports

## 11. Repository Structure
keychron-supply-tracker/
  README.md (EN), README_KR.md (KR)
  docs/ (api-spec.md, data-model.md, alert-rules.md)
  src/ (api_client/, sync_engine/, scheduler/, notifications/, dashboard/, db/)
  tests/
  docker-compose.yml, .env.example, LICENSE (MIT)

## Contact
Partner (Korea): ToBe Networks Inc. - Wooho Yang
Repo: github.com/woohoyang-oss/keychron-supply-tracker