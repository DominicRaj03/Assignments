## Plan: Sprint Health & Risk Prediction Agent

**TL;DR**: Build a three-layer system ingesting Jira telemetry (velocity, burndown, blockers) through webhook + polling, materializing daily sprint metrics in a time-series DB, calculating risk via simple weighted signals, and surfacing risk scores + explainability in a lightweight web dashboard. Jira-only MVP with structured tags for retrospective signals. Simple monolith package, no microservices. Each layer testable independently.

---

## **Steps**

### **Phase 1: API Layer (Foundational)**

**1.1 Define HTTP contract**
- Create endpoints:
  - `POST /api/v1/predict/sprint` — synchronous prediction for a single sprint
  - `GET /api/v1/sprint/{sprint_id}/signals` — debug view of contributing signals
  - `POST /api/v1/webhooks/jira/events` — ingest Jira issue events (webhook receiver)
  - `POST /api/v1/admin/backfill` — backfill historical Jira data (admin-only)
  - `GET /api/v1/backtest/calibration` — backtesting metrics for model calibration

**1.2 Signal extractor module** (`signals/extractor.py`)
- **Input**: Sprint ID, list of Jira issue objects
- **Output**: Named dictionary of 7-8 signals (see below)
- **Signals to extract**:
  1. `velocity_trend` — slope of story points closed over last 3 sprints
  2. `burndown_slope` — current sprint: points resolved per day vs ideal
  3. `blocker_density` — count of issues in "Blocked" status
  4. `blocker_age_max` — longest-blocked issue (days)
  5. `scope_creep_rate` — issues added after sprint start / sprint length (%)
  6. `estimate_variance` — std dev of estimate error (actual - estimate) last sprint
  7. `issue_resolution_velocity` — median time to close by story point (days)
  8. `retrospective_blocker_weight` — aggregated tag-based blocker severity from past retros (structured Jira labels/custom field)

**1.3 Prediction engine** (`models/predictor.py`)
- **Input**: Signal dictionary + team-specific weight vector (defaults provided)
- **Output**: Risk score (0–1), component breakdown, confidence
- **Logic**: `risk = ∑(weight_i × normalized_signal_i)` — simple weighted sum
- **Normalization**: Z-score per signal using rolling 3-month historical baseline
- **Confidence**: Proportion of data available (if <50% signals, lower confidence by 20%)

**1.4 API class** (`api/routes.py`)
- Endpoint handlers calling extractor, predictor, and DB layer
- Input validation (sprint_id format, list size limits)
- Response format versioning (`api_version: "1.0"`)
- Error responses: `{error: "...", details: "...", trace_id: "..."}`

**1.5 Webhook receiver** (`api/webhooks.py`)
- Receives Jira `issue.updated` events
- Deduplicates by `event_id + source` (insert-only check in DB)
- Queues to background worker for async processing
- Returns `{received: true, event_id: "..."}` immediately (fire-and-forget)

---

### **Phase 2: DB Layer (Data Foundation)**

**2.1 Core tables**

| Table | Purpose | Key Columns |
|-------|---------|-------------|
| `telemetry_events` | Immutable Jira event log | `event_id`, `source`, `sprint_id`, `issue_key`, `raw_payload`, `received_at`, `dedup_key` |
| `sprint_daily_metrics` | Daily materialized snapshot for fast queries | `sprint_id`, `date`, `velocity_trend`, `burndown_slope`, `blocker_density`, `blocker_age_max`, `scope_creep_rate`, `estimate_variance`, `issue_resolution_velocity`, `retrospective_blocker_weight`, `data_quality_score` (0–1) |
| `predictions_audit` | All predictions + ground truth for backtesting | `prediction_id`, `sprint_id`, `predicted_risk`, `predicted_confidence`, `created_at`, `actual_outcome` (null until sprint closes), `model_version`, `signal_snapshot` (JSON) |
| `team_weights` | Per-team customizable signal weights | `team_id`, `weight_vector` (JSON), `updated_by`, `updated_at` |
| `retrospective_tags` | Structured blocker tags from past sprints (team-tagged in Jira) | `sprint_id`, `issue_key`, `tag` (e.g., "resource_blocker", "dependency_blocker"), `severity` (1–5) |

**2.2 Materialization logic** (`storage/materialized.py`)
- **Trigger**: Daily at 00:00 UTC (scheduled job)
- **Input**: All telemetry events for sprint since last materialization
- **Output**: Single row in `sprint_daily_metrics` per sprint per day
- **Query pattern**: Aggregate events by signal (e.g., SUM blockers, AVG velocity over window)
- **Storage optimization**: Retention policy — keep raw events 90 days (archive to cold storage), keep metrics indefinitely

**2.3 Event deduplication** (`storage/events.py`)
- Unique index on `(event_id, source)` in `telemetry_events`
- Prevents double-counting from webhook retries or polling race conditions

**2.4 Schema versioning**
- Add `schema_version` column to each table (e.g., `1`, `2`)
- Migrations in `storage/migrations/` (Flask-Migrate or Alembic)
- Backward compatibility: New columns nullable, old code ignores unknown fields

---

### **Phase 3: Web Layer (Stakeholder Visibility)**

**3.1 Dashboard layout**
- **Top**: Sprint selector, date range picker
- **Card 1 — Risk Meter**: Risk score (0–100%), gauge, trend indicator (↑/↓/→)
- **Card 2 — Confidence**: (0–100%), explanation if <70%
- **Card 3 — Top Signals**: 3–5 highest contributing signals with sparklines (7-day trend)
- **Card 4 — Weight Tuning** (Admin only): Sliders for each signal weight, "Reset to Defaults" button, "Save" button → updates `team_weights` table
- **Card 5 — Past 5 Sprints**: Bar chart of risk scores over time, colored by outcome (if known)

**3.2 Data refresh**
- Backend exposes `GET /api/v1/sprint/{sprint_id}/dashboard` → returns full dashboard JSON
- Frontend polls every 5 minutes or on manual refresh
- All data read-only except weight sliders (admin UI)

**3.3 Frontend tech** (lightweight SPA)
- Framework: Plain React or Vue (no heavy tooling; <100KB JS for MVP)
- State: Client-side React Hook for sliders; POST weight changes to `/api/v1/admin/weight`
- Styling: Tailwind CSS or Bootstrap

**3.4 Explainability display**
- Each signal row shows:
  - Signal name & icon (e.g., 📉 Velocity Trend)
  - Raw value & unit
  - Contribution to overall risk (e.g., "+15% toward risk")
  - Sparkline trend (7 days)
  - Data quality badge ("5 sprints, reliable" or "low data")

---

## **Architecture & Component Summary**

```
sprint_predictor/ (monolith)
├── api/
│   ├── routes.py           → Prediction & admin endpoints
│   └── webhooks.py         → Jira event receiver
├── signals/
│   ├── extractor.py        → Calculate 8 signal types
│   ├── validators.py       → Data quality checks
│   └── normalizers.py      → Z-score & baseline cache
├── models/
│   ├── predictor.py        → Weighted signal aggregation
│   └── confidence.py       → Confidence scoring
├── storage/
│   ├── events.py           → Telemetry log operations
│   ├── materialized.py     → Daily snapshot aggregation
│   ├── queries.py          → Reusable DB queries
│   └── migrations/         → Schema versions
├── tasks/
│   ├── scheduler.py        → Background job runner
│   └── backfill.py         → Historical data sync from Jira API
├── web/
│   ├── app.py              → Frontend static handler
│   └── public/
│       ├── index.html
│       ├── dashboard.js
│       └── style.css
└── tests/
    ├── fixtures/           → Sample Jira events (JSON)
    ├── test_signals.py     → Signal extraction tests
    ├── test_predictor.py   → Prediction logic tests
    ├── test_api.py         → HTTP endpoint tests
    ├── test_db.py          → Storage layer tests
    └── test_integration.py → Full flow (API → DB → response)
```

---

## **Verification**

**Unit Tests**:
- `test_signals.py`: Extract signals from fixture events, validate output schema
- `test_predictor.py`: Given signals, verify risk score is 0–1 and contributions sum correctly
- `test_api.py`: Mock DB, verify endpoints return correct status codes and JSON shapes

**Integration Tests** (SQLite in-memory):
- Full flow: Webhook → DB → Query → Prediction → HTTP response
- Deduplication: Send same event twice, verify counted once
- Materialization: Run job, verify sprints_daily_metrics has one row per sprint

**Manual verification**:
1. Load fixture Jira events via `/api/v1/webhooks/jira/events`
2. Query `/api/v1/predict/sprint` → verify risk score and signals appear
3. View dashboard at `http://localhost:3000` → verify charts populate
4. Adjust weight slider to 0.5 for one signal, save → re-run prediction, verify impact

**Backtesting**:
- Query `predictions_audit` after N sprints close (once `actual_outcome` filled)
- Compute calibration curve: predicted risk vs actual slip rate
- Verify confidence intervals match observed accuracy

---

## **Decisions Made**

- **Jira only (v1)**: Reduces dependency complexity; GitHub/calendars as v2 additions
- **Weighted signals over ML**: Explainability & tuneability trump black-box accuracy for internal adoption
- **Structured retrospective tags in Jira**: Avoids document parsing NLP complexity; teams tag during retro
- **Daily materialization**: Trades storage (small overhead) for sub-10ms query latency on dashboards
- **Monolith + thread pool**: No Kafka/RabbitMQ; in-process queue sufficient for webhook scale (<1K/sec)
- **Full-stack MVP**: Web dashboard included so PMs can see value immediately without external tools
- **Immutable events + append-only predictions**: Enables auditing, backtesting, and debugging without ETL rewrites

---

**Next step**: Refinements or ready to proceed?
