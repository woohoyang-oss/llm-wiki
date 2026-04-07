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
| G2 | Automatic delay detection | Alert fired within 1 cycle when ETP/ETD slips ≥ N days |
| G3 | Partial shipment awareness | System flags when models within the same PO diverge in schedule |
| G4 | Audit trail | Full history of every status/date change per model per PO |

## 3. Stakeholders

| Role | Party | Responsibility |
|------|-------|---------------|
| **Data Provider** | Keychron HQ | Maintain PO status, expose API |
| **Data Consumer** | ToBe Networks (KR) | Run polling service, act on alerts |
| **Future Consumers** | Other regional partners | Same API, separate credentials |

---

## 4. System Architecture

```
┌──────────────────────────┐
│      Keychron HQ         │
│                          │
│  Internal PO System      │
│  (daily manual update)   │
│          │                │
│          ▼                │
│  ┌────────────────┐      │
│  │  REST API       │◄─── │── Authentication (API Key / OAuth)
│  │  /api/v1/pos    │     │
│  └────────────────┘      │
└──────────┬───────────────┘
           │  HTTPS (polling)
           ▼
┌──────────────────────────────────────────┐
│      Partner Polling Service (ToBe)      │
│                                          │
│  ┌─────────────┐   ┌─────────────────┐   │
│  │  Scheduler   │──▶│  Sync Engine    │   │
│  │  (cron/APSch)│   │                 │   │
│  └─────────────┘   │  1. Fetch POs   │   │
│                     │  2. Diff vs DB  │   │
│                     │  3. Store snap  │   │
│                     │  4. Fire alerts │   │
│                     └────────┬────────┘   │
│                              │            │
│                     ┌────────▼────────┐   │
│                     │   Local DB      │   │
│                     │  (PostgreSQL)   │   │
│                     │                 │   │
│                     │  po_headers     │   │
│                     │  po_line_items  │   │
│                     │  change_log     │   │
│                     └────────┬────────┘   │
│                              │            │
│                     ┌────────▼────────┐   │
│                     │  Notification   │   │
│                     │  Layer          │   │
│                     │                 │   │
│                     │  • Slack        │   │
│                     │  • KakaoTalk    │   │
│                     │  • Notion DB    │   │
│                     │  • Web Dashboard│   │
│                     └─────────────────┘   │
└──────────────────────────────────────────┘
```

---

## 5. Data Model

### 5.1 PO Header

| Field | Type | Description | Example |
|-------|------|-------------|---------|
| `po_number` | string (PK) | Purchase order number | `PO-2026-0042` |
| `po_date` | date | Date PO was placed | `2026-03-15` |
| `overall_status` | enum | Derived: slowest item status | `In Production` |
| `total_models` | int | Count of distinct model lines | `3` |
| `remarks` | text | PO-level notes | — |
| `created_at` | timestamp | First seen by poller | — |
| `updated_at` | timestamp | Last change detected | — |

### 5.2 PO Line Item (per model number)

| Field | Type | Description | Example |
|-------|------|-------------|---------|
| `po_number` | string (FK) | → po_headers | `PO-2026-0042` |
| `line_no` | int | Line sequence | `1` |
| `model_number` | string | Keychron model code | `K10P-RD9` |
| `model_name` | string | Human-readable name | `Keychron K10 Pro Red` |
| `qty_ordered` | int | Units ordered | `500` |
| `qty_produced` | int | Units completed | `320` |
| `status` | enum | Current production status | `In Production` |
| `etp` | date | Est. production completion | `2026-04-20` |
| `etd` | date | Est. departure from origin | `2026-04-28` |
| `eta` | date | Est. arrival at partner | `2026-05-05` |
| `remarks` | text | Model-level notes | `PCB parts pending` |
| `updated_at` | timestamp | HQ last update time | — |

### 5.3 Change Log

| Field | Type | Description |
|-------|------|-------------|
| `id` | serial (PK) | Auto-increment |
| `po_number` | string | Reference PO |
| `model_number` | string | Reference model (nullable for PO-level changes) |
| `field_name` | string | Which field changed (`status`, `etp`, `etd`, etc.) |
| `old_value` | text | Previous value |
| `new_value` | text | New value |
| `changed_at` | timestamp | When the change was detected |

### 5.4 Status Enum (proposed — confirm with HQ)

```
Draft → Confirmed → Waiting Parts → In Production → QC → Packed → Shipped → Delivered
```

---

## 6. System Flow

### 6.1 Polling Cycle (Happy Path)

```
┌─────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│Scheduler │────▶│ Call HQ  │────▶│  Parse   │────▶│  Diff    │────▶│  Store   │
│ triggers │     │  API     │     │ response │     │ vs local │     │ changes  │
└─────────┘     └──────────┘     └──────────┘     └────┬─────┘     └──────────┘
                                                       │
                                                       ▼
                                              ┌─────────────────┐
                                              │  Changes found? │
                                              └────┬───────┬────┘
                                                   │       │
                                                  YES      NO
                                                   │       │
                                              ┌────▼──┐  ┌─▼──────┐
                                              │ Fire  │  │  Log   │
                                              │ Alert │  │ "no    │
                                              │       │  │ change"│
                                              └───────┘  └────────┘
```

### 6.2 Alert Decision Tree (per model, per PO)

```
For each PO line item (model_number):
│
├── Status changed?
│   └── YES → Notify: "[PO-0042] K10P-RD9: In Production → QC"
│
├── ETP moved back ≥ 3 days?
│   └── YES → ⚠️ DELAY ALERT: "[PO-0042] K10P-RD9 ETP slipped 4/15 → 4/25"
│
├── ETD moved back ≥ 3 days?
│   └── YES → ⚠️ DELAY ALERT (same pattern)
│
├── No update for ≥ 3 days?
│   └── YES → 🔔 STALE ALERT: "[PO-0042] K10P-RD9 no update since 3/27"
│
└── Schedule gap vs other models in same PO ≥ 10 days?
    └── YES → 📦 PARTIAL SHIPMENT SUGGESTION:
              "K8P-WH1 ready 4/10, K10P-BL3 not until 4/25.
               Consider shipping K8P-WH1 first?"
```

### 6.3 Partial Shipment Flow

```
PO-2026-0042:
  ├─ K8P-WH1   ETP: 04/10 ✅ Complete  ──┐
  ├─ K10P-RD9  ETP: 04/20 In Production  │  Gap = 15 days
  └─ K10P-BL3  ETP: 04/25 Waiting Parts  │  → System suggests partial shipment
                                           │
                              ┌─────────────▼──────────────┐
                              │  PARTIAL SHIPMENT REVIEW   │
                              │                            │
                              │  Ship now:  K8P-WH1 (200)  │
                              │  Ship later: K10P-RD9 (500)│
                              │              K10P-BL3 (300)│
                              └────────────────────────────┘
```

---

## 7. API Specification (Proposed)

> **This is the API contract we need Keychron HQ to implement or confirm.**

### 7.1 Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v1/pos` | List all POs for the authenticated partner |
| `GET` | `/api/v1/pos/{po_number}` | Get single PO with all line items |
| `GET` | `/api/v1/pos?updated_since={ISO8601}` | Incremental sync — only POs updated after timestamp |

### 7.2 Authentication

- **Option A**: API Key in header (`X-API-Key: <key>`)
- **Option B**: OAuth 2.0 client credentials
- **Option C**: Basic Auth over HTTPS

*Recommendation: Option A for simplicity in MVP.*

### 7.3 Response Format

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
        },
        {
          "line_no": 2,
          "model_number": "K10P-BL3",
          "model_name": "Keychron K10 Pro Blue",
          "qty_ordered": 300,
          "qty_produced": 0,
          "status": "Waiting Parts",
          "etp": "2026-04-25",
          "etd": "2026-05-02",
          "eta": "2026-05-10",
          "remarks": "PCB component delayed from supplier"
        }
      ]
    }
  ]
}
```

### 7.4 Error Responses

| Code | Meaning |
|------|---------|
| `401` | Invalid or missing API key |
| `403` | Partner not authorized for requested PO |
| `404` | PO not found |
| `429` | Rate limit exceeded |
| `500` | Internal server error |

---

## 8. Questions for Keychron HQ

Before development begins, we need answers to the following:

### API & System

| # | Question | Impact |
|---|----------|--------|
| Q1 | Does an API already exist, or does it need to be built? | Timeline & scope |
| Q2 | What system manages PO data internally? (ERP, custom, spreadsheet?) | Integration approach |
| Q3 | What authentication method is preferred? | Security design |
| Q4 | Is there a rate limit on API calls? | Polling frequency |
| Q5 | Can responses be filtered by `updated_since` timestamp? | Bandwidth optimization |

### Data & Business Logic

| # | Question | Impact |
|---|----------|--------|
| Q6 | What are the exact status codes used internally? | Status enum mapping |
| Q7 | Is the model number format standardized? (e.g. `K10P-RD9`) | Data matching with partner ERP |
| Q8 | Are partial shipments (splitting a PO across multiple shipments) supported? | Shipment tracking model |
| Q9 | Can qty_ordered change after PO is confirmed? (defects, adjustments) | Change detection logic |
| Q10 | Will other partners besides ToBe use this system? | Multi-tenancy design |

---

## 9. Tech Stack (Partner Side)

| Component | Choice | Rationale |
|-----------|--------|-----------|
| Language | Python 3.11+ | Fast prototyping, strong HTTP/DB libraries |
| Framework | FastAPI | Async, auto-docs, lightweight |
| Scheduler | APScheduler | In-process, configurable |
| Database | PostgreSQL | Reliable, JSON support, change tracking |
| Notifications | Slack Webhook, Notion API, KakaoTalk (via ToBe MSG) | Multi-channel alerts |
| Deployment | Docker on existing infra (Tailscale network) | No extra cloud cost |

---

## 10. MVP Scope

### Phase 1 — Core Polling (Week 1-2)
- [ ] API client for HQ endpoint
- [ ] Scheduler (configurable interval)
- [ ] Local DB schema + migrations
- [ ] Diff engine (detect changes per model per PO)
- [ ] Change log persistence

### Phase 2 — Alerts (Week 2-3)
- [ ] Slack webhook notifications
- [ ] Delay detection (ETP/ETD slip ≥ N days)
- [ ] Stale PO detection (no update ≥ N days)
- [ ] Partial shipment suggestion

### Phase 3 — Dashboard & Integrations (Week 3-4)
- [ ] Web dashboard (PO list → model detail drill-down)
- [ ] Notion DB sync
- [ ] ETP/ETD timeline (Gantt-style) view
- [ ] Historical delay report per model

---

## 11. Repository Structure

```
keychron-supply-tracker/
├── README.md                  ← This file (EN)
├── README_KR.md               ← Korean version
├── docs/
│   ├── api-spec.md            ← Detailed API contract
│   ├── data-model.md          ← DB schema & ERD
│   └── alert-rules.md         ← Alert configuration
├── src/
│   ├── api_client/            ← HQ API client
│   ├── sync_engine/           ← Diff & change detection
│   ├── scheduler/             ← Polling scheduler
│   ├── notifications/         ← Slack, Notion, KakaoTalk
│   ├── dashboard/             ← Web UI
│   └── db/                    ← Models, migrations
├── tests/
├── docker-compose.yml
├── .env.example
└── LICENSE
```

---

## License

MIT

## Contact

- **Partner (Korea):** ToBe Networks Inc. — Wooho Yang
- **Repository:** [github.com/woohoyang-oss/keychron-supply-tracker](https://github.com/woohoyang-oss/keychron-supply-tracker)
