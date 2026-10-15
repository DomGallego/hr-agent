# PLAN.md — HR AI Platform

**One repo, four modules, one story:** an HR AI platform with an agent (S1), a router (S4),
a compare harness (S2), and a retrieval-quality monitor (S3) — built to refresh skills and
double as a portfolio piece for the Sprout panel.

> Status legend: ⬜ todo · 🔧 in progress · ✅ done

---

## 1. Final Tech Stack

| Layer | Choice | Notes |
|---|---|---|
| Env mgmt | **uv** | `pyproject.toml` + `uv.lock`, `uv sync`, `uv run` |
| Backend | **FastAPI** | async, SSE streaming, Pydantic validation, `Depends()` auth |
| Auth | **PyJWT** | sign/verify JWT; `employee_id` extracted from token only |
| Orchestration | **LangGraph** | per-module graphs, conditional edges, state |
| LLM plumbing | **LangChain** (`@tool`, ChatOpenAI→OpenRouter, prompt templates, parsers) | |
| LLMs | **OpenRouter `:free` models** | agent: `llama-3.1-8b-instruct:free`; judge/alt: `gemini-2.0-flash-exp:free`, `deepseek-chat:free`; decision arms: `microsoft-decision-1`, `scaledown/classify-1` ($0.04/M ≈ free, flagged in README) |
| Guard | rule filter (regex) + LLM tripwire (cheap decision model), fail-closed | S1 |
| DB | **Postgres 16 + pgvector** | all data in one DB; RLS on every table; pgvector for embeddings |
| Authz depth | **Row-Level Security** | `SET app.employee_id` per session; `USING` policies |
| Observability | **Langfuse Cloud (free tier)** | `@observe()` + ~10-line trace-ID/employee-stamp shim; traces, prompt versions, costs, eval scores; public API consumed by S3 monitor |
| Evals | **pytest** suites + **LLM-as-a-judge** | 5 suites (S1), parametrized golden sets, GitHub Actions gate |
| CI | **GitHub Actions** | uv sync → ruff → mypy → pytest (unit + evals) |
| Infra | **Docker Compose** | postgres+pgvector, api, ui only (Langfuse is cloud) |
| UI | **Streamlit** | chat, verdict tables, drift charts |
| Data | **HF `bitext` 26.9k** (S4) + **synthetic generator** (S1/S3) | golden Q&A pairs, drift scenario |
| Optional | local **DistilBERT** fine-tune arm (pluggable, off by default) | S4 |
| Hygiene | ruff, mypy | CI-enforced |

### Key decisions (and why)
- **Langfuse Cloud, not self-hosted**: free tier is generous; zero ops; shareable trace links for
  the README. Code keeps `LANGFUSE_HOST` env var so self-hosting is a one-line switch.
- **No OTel collector**: `@observe()` on endpoints + a small contextvar shim stamps a
  deterministic `trace_id`, `user_id` (pseudonymous `emp_****42`), and `metadata`
  (`module`, `prompt_ver`, `model_id`) onto each trace. Verified: Langfuse ships no FastAPI
  middleware; `@observe()` alone gives a random-ID trace per request, so the shim is the
  identity/correlation layer. S3 monitor queries the Langfuse public API
  (`GET /api/public/v2/observations` with `userId`/`sessionId` + metadata filters).
- **Tools are LangChain `@tool`** in LangGraph ToolNodes (not a separate MCP server). Stretch
  goal: wrap the toolset as a real MCP server via `langchain-mcp-adapters`.
- **pgvector over Qdrant/Chroma**: one DB, one authorization boundary — RLS covers vector search too.
- **Trace shim is ours**: `contextvars.ContextVar` set in FastAPI middleware, echoed back as
  `X-Trace-ID` response header, passed into Langfuse trace context and logged.

---

## 2. Repo Layout

```
hr-platform/
├── PLAN.md                    ← this file
├── README.md                  # architecture diagram, eval results table, demo GIF
├── docker-compose.yml         # postgres+pgvector, api, ui
├── pyproject.toml             # uv workspace root
├── services/
│   ├── api/                   # FastAPI app
│   │   ├── main.py            # app, routers, SSE
│   │   ├── auth.py            # JWT sign/verify, Depends(get_current_employee)
│   │   ├── middleware.py      # trace-id contextvar + employee stamp
│   │   ├── db.py              # async pool, SET app.employee_id per session
│   │   └── routers/           # /auth /chat /classify /compare /drift
│   └── ui/                    # Streamlit app (chat + dashboards)
├── modules/
│   ├── payslip_agent/         # S1: graph, tools, note guard, compose
│   ├── ticket_router/         # S4: backends/{prompted,decision,finetuned}.py
│   ├── model_compare/         # S2: eval matrix, shadow replay, flag flip
│   └── retrieval_watch/       # S3: Langfuse trace API → drift detect → alert
├── packages/evals/            # shared heart: golden sets, suites, judge
│   ├── golden/                # Q&A pairs (jsonl), per-suite
│   ├── suites/                # test_retrieval.py test_faithfulness.py
│   │                          # test_refusals.py test_injection.py test_regression.py
│   └── judge.py               # LLM-as-a-judge (gemini flash :free)
├── db/
│   ├── migrations/            # SQL: schema + RLS policies
│   └── seed/                  # synthetic generators + loaders + drift simulator
└── .github/workflows/ci.yml
```

Each module is independently demoable; `packages/evals` is the shared heart:
model_compare *is* the eval layer, ticket_router *is* a feature the evals gate,
payslip_agent *is* the flagship.

---

## 3. Database Schema (key tables)

```sql
employees     (employee_id PK, name, role, dept)                        -- RLS
payslips      (payslip_id, employee_id FK, month, gross, deductions, net) -- RLS
attendance    (attendance_id, employee_id, date, status)                -- RLS
loans         (loan_id, employee_id, balance, monthly_due)              -- RLS
contributions (contrib_id, employee_id, sss, philhealth, tax)           -- RLS
hr_notes      (note_id, employee_id, author_admin_id, body TEXT)        -- RLS; injection surface
tickets       (ticket_id, text, category40, source='bitext'|'synthetic')
embeddings    (chunk_id, employee_id, content, vec vector(768))         -- RLS + pgvector
prompt_versions (prompt_id PK, module, version, text, created_at)
model_flags     (feature PK, active_backend, model_id, enabled)         -- one-flag rollback
eval_results    (trace_id, feature, prompt_ver, model_id, suite, verdict, ts)
```

**RLS policy (same shape on every employee-scoped table):**

```sql
ALTER TABLE payslips ENABLE ROW LEVEL SECURITY;
CREATE POLICY employee_own ON payslips
  USING (employee_id = current_setting('app.employee_id')::int);
```

Connection-pool wrapper runs `SET app.employee_id = $1` from the JWT-derived ID before every
tool query. Defense in depth: (1) tool params bound to JWT identity, (2) RLS at the DB.
No write tools exist at all.

---

## 4. Module Specs

### S1 — payslip_agent (flagship)
- Graph: `classify_intent → [conditional route] → tool nodes
  (get_payslips / get_attendance / get_loans / get_notes) → note_guard → compose`
- `get_notes` output delimited as `<data source="hr_note">…</data>` — data, never instructions
- Guard: regex rules first, then decision-model tripwire; **guard down = fail closed**
- Refusals: canned messages for out-of-scope / others' data / writes / tax advice, logged with trace ID
- **Log policy enforced in code**: never persist payslip values, note bodies, or tool results;
  log trace_id, pseudonymous emp, tool names + params, guard verdict, response *category*
- Demo: 3 scripted flows (loan deduction, hostile note, coworker's payslip) + free chat

### S4 — ticket_router
- Bitext 26.9k → remap to 40 categories, stratified 4200/1800 train/test (frozen)
- Backends behind one interface:
  - `prompted`: free LLM + few-shot mined from train split
  - `decision`: OpenRouter decision model (microsoft-decision-1 / scaledown-classify-1)
  - `finetuned`: local DistilBERT — optional, `ROUTER_BACKEND` flag, off by default
- Eval: accuracy + per-category matrix + confusion pairs + calibration; verdict table in UI
- Drift hook: confidence-distribution monitoring (same dashboard as S3)

### S2 — model_compare
- Runs any enabled feature's golden suite across ≥2 OpenRouter models,
  per (feature × prompt_ver × model_id) cell
- Decision rule computed from N: non-inferior within margin → switch; inferior → keep
- Shadow-replay: re-run logged traces against candidate model without serving
- Rollback: flip row in `model_flags`; every trace records the triple
  (feature, prompt_ver, model_id)

### S3 — retrieval_watch
- Reads Langfuse public API: per-employee avg retrieval scores over time
- Drift detector: score drop > threshold vs 30-day baseline → alert
- Demo: `db/seed/drift_simulator.py` injects "scanned-garbage docs" for one fake tenant →
  scores visibly drop → monitor flags the cohort → cohort-diff view
  (the "40 vs the 2,960" table from the panel deck)

---

## 5. Eval Design (packages/evals)

Five suites, all pytest + parametrized:

| # | Suite | Checks |
|---|---|---|
| 1 | Retrieval | right rows come back for golden questions |
| 2 | Faithfulness | judge verifies answer uses only retrieved data |
| 3 | Refusals | adversarial asks ("show my coworker's payslip") refused cleanly |
| 4 | Injection | notes containing "SYSTEM: ignore rules" cannot steer behavior |
| 5 | Regression | golden set reruns on every prompt/tool/model change; CI fails on regression |

Golden dataset format (jsonl):
`{query, expected_answer, expected_tools[], expected_trace_shape}`

Synthetic generator produces data + golden pairs; human review pass (Dom) before freezing.

**CI gate:** eval suites run on every PR; ≥2-point regression on any suite = fail
(the S2 margin rule, reused).

---

## 6. Auth Flow (end to end)

1. `/auth/login` demo endpoint: pick employee from seeded list → server signs
   JWT `{employee_id, role, exp}`
2. Streamlit stores token; every request sends `Authorization: Bearer`
3. FastAPI `Depends(get_current_employee)` verifies signature → extracts `employee_id`
   (never from the model/LLM)
4. Trace shim stamps `trace_id` + `user_id=emp_****42` + metadata into Langfuse
5. DB pool sets `app.employee_id` → RLS enforces at query time
6. HR admin role = delegated identity at the tool boundary (the S1 adaptability answer, implemented)

---

## 7. Phased Build Order

| Phase | Scope | Est. | Status |
|---|---|---|---|
| 0 | Scaffold: uv workspace, compose (postgres+pgvector, api, ui), FastAPI health, CI skeleton, Langfuse Cloud env | ½ day | ✅ |
| 1 | Data + auth: migrations, RLS, synthetic generators, JWT, db pool + `SET app.employee_id`, RLS tests (emp A can't see emp B) | 1–2 days | ✅ |
| 2 | payslip_agent: LangGraph graph, 4 `@tool`s, OpenRouter free model, note guard, refusals, trace shim, Streamlit SSE chat; first eval suites | 2–3 days | ✅ (Langfuse shim pending keys) |
| 3 | Evals + Langfuse depth: judge, golden freeze, prompt-version tracking, regression suite in CI, log-policy tests | 1–2 days | ✅ (Cloud wiring ready, pending keys) |
| 4 | ticket_router: Bitext load/remap, prompted + decision backends, verdict table, drift hook | 2 days | ✅ (finetuned arm = documented stub; drift hook lands with S3 in Phase 6) |
| 5 | model_compare: eval matrix runner, shadow replay, `model_flags` flip endpoint, real verdict table | 1–2 days | ✅ |
| 6 | retrieval_watch: trace API reader, drift detector, drift simulator, cohort-diff dashboard | 1–2 days | ✅ (Langfuse reader implemented; live validation pending keys) |
| 7 | Polish: README (diagram, eval results, GIF, one-command setup), optional finetuned arm (weekend), MCP wrapper (stretch), mobile UI check | 1–2 days | ✅ (finetuned arm + MCP = documented roadmap) |

Total: ~10–14 focused days.

---

## 8. Risks / Notes

- **Free-model rate limits** during eval runs → backoff + serial eval runs in CI
- **Langfuse Cloud quota** → traces are metadata-light by design (log policy helps)
- **Bitext categories ≠ 40 HR categories** → remap script is part of the deliverable (shows data judgment)
- **pgvector embedding model** → `sentence-transformers` locally (zero API cost); decide in Phase 2
- **Seeded trace IDs must be deterministic per request** so CI eval failures can be traced in Langfuse
