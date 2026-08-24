# 🔍 Smart Log Analyzer & Anomaly Detector

![Status](https://img.shields.io/badge/status-MVP-yellow.svg)
![Python](https://img.shields.io/badge/Python-FastAPI-009688?style=flat&logo=fastapi&logoColor=white)
![SQLite](https://img.shields.io/badge/DB-SQLite-07405E?style=flat&logo=sqlite&logoColor=white)
![Claude](https://img.shields.io/badge/AI-Claude_Sonnet-blueviolet)
![License](https://img.shields.io/badge/license-MIT-blue.svg)

**Live demo:** [digi-plusit.vercel.app](https://digi-plusit.vercel.app/)

---

## 📖 Project Overview

Smart Log Analyzer & Anomaly Detector is an MVP that ingests log entries, flags anomalies using a rule/statistics-based detector, and then uses Claude to turn each flagged entry into a plain-English explanation with a likely root cause and next step. It includes a browser UI to view the log timeline, see flagged entries highlighted, and drill into anomaly detail.

The core design decision: **detection is deterministic and AI-free.** A hand-written scoring engine decides what's anomalous. Claude is only called afterward, on entries the engine has already flagged, and only to explain — not to re-judge. This keeps the detection logic auditable and testable, and stops the AI from doing the one job it wasn't meant to do.

---

## ✨ Features

- **CSV log ingestion** with header validation and per-row error handling — bad rows are skipped and logged, not silently dropped.
- **Demo dataset generator** — one click loads a synthetic dataset (mostly normal traffic plus 4 injected anomaly types) to see the pipeline run end to end.
- **Rule-based anomaly detection** using additive scoring across four independent checks (server errors, rare event types, repeated auth failures, traffic bursts).
- **AI-generated explanations** — Claude turns each flagged entry into a plain-English explanation and a suggested root cause / next step.
- **Graceful degradation** — if no Claude API key is set, ingestion and detection still run normally; the AI fields just fall back to a placeholder instead of crashing the pipeline.
- **Timeline UI** — single-page frontend (vanilla JS, no build step) to browse logs and drill into flagged entries.
- **Ingestion issue log** — a dedicated table records rows that failed validation, so bad data is visible instead of disappearing.

---

## 🌍 Real World Use Cases

1. **Server monitoring** — catch 5xx errors as they land instead of digging through raw logs after the fact.
2. **Brute-force detection** — flag repeated 401/403s from the same source within a short window.
3. **Traffic anomaly spotting** — catch bursts of requests from a single source that look like abuse or a misbehaving client.
4. **Rare-event surfacing** — flag event types that make up a tiny fraction of overall traffic, which are easy to miss by eye.
5. **Fast triage for small teams** — a lightweight, no-infra way to get plain-English incident summaries without standing up a full observability stack.

---

## 🏗 System Architecture

```text
       [ User ]
          │
          ▼
   (Frontend: index.html) ──────> Timeline UI / Anomaly detail view
          │
          ▼
   (FastAPI backend: app.py) ───> Routes for CSV upload, demo data, logs, anomalies
          │
          ▼
     (ingest.py) ───────────────> CSV parsing + validation
          │
          ▼
 (anomaly_detector.py) ─────────> Rule-based scoring engine (own logic, no AI)
          │
          ├──────────────────────────┐
          ▼                          ▼
   (database.py: SQLite)      (ai_explainer.py)
   logs / anomalies /               │
   ingest_issues tables             ▼
                              (Claude API)
                                     │
                                     ▼
                        Plain-English explanation +
                           suggested root cause
```

1. The user uploads a CSV or clicks **Load demo dataset** in the frontend.
2. The FastAPI backend routes the request to `ingest.py`, which parses and validates each row.
3. Valid rows are scored by `anomaly_detector.py`. Rows crossing the threshold are flagged.
4. Every row is persisted to SQLite; flagged rows also get a row in the `anomalies` table.
5. For each flagged row, `ai_explainer.py` calls Claude with the entry and the specific rule(s) it tripped, and stores the returned explanation.
6. The frontend polls/fetches this data to render the timeline, with flagged entries highlighted.

---

## 🛠 Tech Stack

### Backend
- **Python / FastAPI** — REST API and app entry point (`app.py`), run via `uvicorn`.
- **SQLite** — lightweight file-based persistence, no external DB server needed.

### Frontend
- **Vanilla JavaScript, single `index.html`** — no build step, no framework.

### AI Integration
- **Anthropic Claude API** — used only for explaining already-flagged entries (default model: `claude-sonnet-4-5`, configurable via env var).

### Deployment
- **Vercel** — live demo hosted at `digi-plusit.vercel.app`.

---

## 📁 Project Structure

```
backend/
  app.py               FastAPI routes
  database.py           SQLite schema + persistence helpers
  ingest.py              CSV parsing + validation
  anomaly_detector.py    Rule-based detection engine (the "own algorithm")
  ai_explainer.py         Claude API calls, scoped to explanation only
  seed_data.py            Synthetic demo dataset generator
frontend/
  index.html             Single-page UI (vanilla JS, no build step)
```

`ingest.py` is deliberately isolated so another log format (JSON, plain text) could be added later without touching detection, storage, or AI logic.

---

## 🗄 Database Design

SQLite, three tables:

| Table | Purpose |
|---|---|
| `logs` | Every ingested entry, valid or flagged. |
| `anomalies` | One row per flagged log — includes score, which rule(s) it tripped, and the Claude-generated explanation. |
| `ingest_issues` | Rows that failed validation during CSV parsing (missing timestamp, bad status code, etc.), so bad data stays visible instead of being silently dropped. |

### Expected CSV format for upload

Header row required, case-insensitive columns:

```
timestamp,source,event_type,status_code,message
2026-08-20T09:14:02,192.168.1.14,GET /api/users,200,success
```

- `severity` is optional — derived from `status_code` if not supplied (2xx/3xx → INFO, 4xx → WARNING, 5xx → ERROR).
- `source` and `event_type` are required.
- `status_code` and `message` are optional.

---

## 🚨 Anomaly Detection Engine

Detection is **additive scoring**, not a single hard-coded if/else. Each entry accumulates points from independent checks; if the total crosses a threshold of 30, it's flagged.

| Rule | Score | Logic |
|---|---|---|
| Server error | +40 | Status code ≥ 500 |
| Rare event type | +30 | Event type makes up less than 2% of the dataset |
| Repeated auth failure | +50 | 3+ 401/403s from the same source within 5 minutes (brute-force pattern) |
| Traffic burst | +35 | 10+ requests from the same source within 60 seconds |

The additive approach means an entry that trips two weak signals gets flagged even if neither alone would justify it — and the score gives a rough severity ranking for triage.

---

## 🤖 AI Integration

Claude's role is intentionally narrow. It:

- Only runs on entries the rule engine has **already** flagged.
- Receives the entry plus the specific rule(s) it tripped.
- Is explicitly told not to re-judge whether the entry is anomalous — only to explain it and suggest a root cause / next step.

This keeps detection deterministic and auditable, and stops the AI from quietly taking over the one decision the project is built to keep rule-based.

If `ANTHROPIC_API_KEY` isn't set, the app still runs end to end — flagging and persistence work normally, and the AI fields just fall back to "AI explanation unavailable" plus the raw rule-based reasons.

---

## 🚀 Installation Guide

### Prerequisites
- Python 3.x
- pip
- An Anthropic API key (optional — only needed for AI explanations)

### Setup

```bash
cd backend
pip install -r requirements.txt
export ANTHROPIC_API_KEY=sk-ant-...        # required for AI explanations
export ANTHROPIC_MODEL=claude-sonnet-4-5   # optional, defaults to this
uvicorn app:app --reload --port 8000
```

Open `http://localhost:8000` in a browser. Click **Load demo dataset** to generate a synthetic dataset and see the full pipeline run, or **Upload CSV** to bring your own data.

---

## 📌 Assumptions

- The log schema (`timestamp, source, event_type, status_code, message`) is a generalized superset of a typical HTTP access log, so it can also cover non-HTTP sources — `source` doesn't have to be an IP.
- Severity is auto-derived from status code when not supplied, since most raw logs don't carry an explicit severity field.
- Rule thresholds (30-point flag bar, 60s/10-request burst, 5-min/3-attempt brute force) are reasonable defaults for a demo-scale dataset — not tuned against real production traffic.
- Single-tenant, single-process app with no API authentication — out of scope for an MVP.

---

## ⚠️ Limitations

- **Thresholds are hand-tuned, not learned.** They work on the synthetic demo dataset but need calibration (or a feedback loop from analyst overrides) before production use.
- **No cross-run baseline.** Detection only looks at the current batch — a source that's unusually active in every upload won't be flagged relative to its own history, since there's no persisted per-source baseline.
- **Traffic-burst rule is a blunt absolute count**, not an adaptive statistical one. A per-source z-score was tried and dropped — it caused both false negatives (too little history to build a baseline) and false positives (naturally bursty normal sources).
- **AI calls are synchronous and sequential** — one per flagged entry. Fine for a handful of anomalies; a large flagged batch would need batching or async calls.
- **No auth or rate limiting** on the API — not safe to expose publicly as-is.
- **CSV only.** JSON or plain-text log ingestion isn't implemented yet, though `ingest.py` is isolated specifically so another parser could be added without touching detection, storage, or AI logic.

---

## 🔮 Future Enhancements

- Persisted per-source historical baseline for cross-run anomaly detection.
- Adaptive/statistical traffic-burst detection (revisit the z-score approach with a warm-up period).
- Async / batched Claude calls for large flagged batches.
- API authentication and rate limiting.
- JSON and plain-text log ingestion, reusing the isolated `ingest.py` boundary.
- Feedback loop where analyst overrides retrain or adjust rule thresholds over time.

---

## 📄 Resume Description

* **Built a log anomaly detection MVP** in Python/FastAPI with a deterministic, rule-based scoring engine — kept fully separate from AI, so detection stays auditable.
* **Integrated the Claude API** to generate plain-English explanations and root-cause suggestions for flagged log entries, with graceful fallback when no API key is present.
* **Designed a validation-first CSV ingestion pipeline** that logs malformed rows instead of dropping or crashing on them.
* **Documented and shipped known limitations honestly** (fixed thresholds, no cross-run baseline, synchronous AI calls) rather than overstating the MVP's readiness for production.

---

## 🎤 Interview Explanation

### ⏱ 30-Second Pitch
"I built a log anomaly detector where a rule-based scoring engine does all the actual flagging — no AI involved in that decision. Once something's flagged, I hand it to Claude with the specific rule it tripped, and Claude's only job is to explain it in plain English and suggest a fix. Keeping detection and explanation separate means the flagging logic stays deterministic and testable."

### ⏱ 1-Minute Explanation
"The backend is FastAPI with SQLite. Logs come in as CSV, get validated, and any row that fails validation is logged rather than dropped. Each valid row runs through an additive scoring engine — server errors, rare event types, repeated auth failures, and traffic bursts each add points, and crossing a threshold flags the row. Only flagged rows get sent to Claude, along with the specific rule they tripped, and Claude is explicitly told not to re-judge whether it's anomalous — just to explain it. If there's no API key set, the app still works end to end; the AI fields just fall back to a placeholder instead of breaking ingestion."