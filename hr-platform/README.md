# HR AI Platform

**One repo, four AI systems, one story.** Built to refresh production ML skills and to
defend end-to-end in an interview panel — the four situations from the AI/ML Engineer
assessment (payslip agent, cheaper-model decision, retrieval RCA, fine-tune-or-prompt),
each shipped as working software rather than slideware.

> See [PLAN.md](PLAN.md) for the build plan and [LEARN.md](LEARN.md) for the
> concept deep-dives + 25-question self-test that documents every design decision.

```
                          ┌────────────────────────────┐
                          │      Streamlit UI          │
                          │ chat · classify · compare  │
                          │      · drift dashboard     │
                          └─────────────┬──────────────┘
                                        │ JWT (PyJWT)
                                        ▼
┌────────────────────────────────────────────────────────────────────────────┐
│  FastAPI  —  trace-ID middleware, Depends() auth, SSE streaming            │
│                                                                            │
│   /chat ──────────► LangGraph payslip agent (S1)                           │
│   /classify ──────► ticket router (S4): prompted | decision | finetuned    │
│   /compare/* ─────► model compare (S2): matrix · shadow · one-flag flip    │
│   /drift/* ───────► retrieval watch (S3): alerts · cohort diff · simulate  │
│                                                                            │
│   tools (read-only @tool, no employee_id param)   note guard (fail-closed) │
└──────────┬─────────────────────────────────────┬───────────────────────────┘
           │ SET app.employee_id per txn         │ every LLM call traced
           ▼                                     ▼
┌────────────────────────────┐    ┌────────────────────────────────────────┐
│  Postgres 16 + pgvector    │    │  Langfuse (cloud, free tier)           │
│  RLS on every emp table    │    │  trace IDs · prompt versions · costs   │
│  one auth boundary         │    │  public API ← S3 drift monitor         │
└────────────────────────────┘    └────────────────────────────────────────┘

   OpenRouter :free models (llama-3.1-8b) ── deterministic MockLLM when keyless
   GitHub Actions CI: uv sync → ruff → mypy → pytest (unit + 5 eval suites)
```

## The four systems

| | System | Story |
|---|---|---|
| **S1** | **Payslip agent** | LangGraph agent over read-only tools. Identity from the JWT only — tools take **no** `employee_id` parameter. Postgres **RLS** as layer two. HR notes enter the prompt as `<data>`, guarded by rules + an LLM tripwire that **fails closed**. Four canned refusals. Log policy enforced by tests: no pay values, note bodies, or tool results in any trace. |
| **S4** | **Ticket router** | 40-category HR classification with three swappable arms behind one interface: `prompted` (free LLM), `decision` (calibrated probabilities), `finetuned` (DistilBERT stub, deliberately unimplemented). Low confidence escalates to human review. The evals **detect the decision arm's simulated weakness** as a real confusion pair — and document that keyword mocks collapse on real text (0.20), which is why the LLM arms exist. |
| **S2** | **Model compare** | Per (feature × prompt_ver × model) eval matrix with the **pre-stated margin rule** (non-inferior within 2 pts → switch; hard-metric fail → revisit). Shadow replay re-runs logged requests serving **nobody**. Rollback = one row in `model_flags` (hr_admin-gated endpoint, tested). |
| **S3** | **Retrieval watch** | The monitor the deck wished existed: per-employee daily retrieval scores, drift detector (baseline 30d vs recent 7d, onset dating), and the **cohort-diff table** ("who" vs "when"). A simulator replays the scan-upload incident deterministically. Keyless it reads the local log; keyed it reads the Langfuse public API. |

## Eval results (keyless mock run, deterministic)

| module | metric | value | n / notes |
|---|---|---|---|
| S1 payslip agent · current | faithfulness | **1.000** | n=3 → baseline |
| S1 payslip agent · candidate | faithfulness | **1.000** | n=3 → switch (non-inferior) |
| S4 ticket router · prompted (mock) | exact match, synthetic text | **1.000** | n=480 |
| S4 ticket router · decision (mock) | exact match, synthetic text | **0.992** | n=480, confusion pair detected by evals |
| S4 ticket router · prompted (mock) | exact match, real Bitext | **0.200** | n=400 — mock collapses on real text; the LLM arm exists for this |
| S2 shadow replay · decision | shadow accuracy (serves nobody) | **0.880** | n=100 |

Regenerate anytime: `uv run python -m packages.evals.report` (writes
`packages/evals/EVAL_REPORT.md`; with `OPENROUTER_API_KEY` set it scores the real arms).

## Quickstart

```bash
# 1. deps (uv: https://docs.astral.sh/uv/)
uv sync --dev

# 2. database (Docker Desktop on) — Postgres 16 + pgvector, migrations auto-run
docker compose up -d db

# 3. synthetic data (deterministic, pinned months)
uv run python -m db.seed.load

# 4. run it (two terminals)
uv run uvicorn services.api.main:app --port 8000
uv run streamlit run services/ui/app.py

# 5. tests / evals / report
uv run pytest tests packages/evals/suites -q
uv run python -m packages.evals.report
```

Login is a demo picker (employee 1 = hr_admin, others = employee). All LLM traffic runs
through a deterministic **MockLLM** until `OPENROUTER_API_KEY` is set in `.env` — every
test and demo flow works offline, and CI never touches the network.

Optional `.env` (copy `.env.example`): `OPENROUTER_API_KEY`, `LANGFUSE_PUBLIC_KEY`,
`LANGFUSE_SECRET_KEY`, `LANGFUSE_HOST`.

## Security posture (the panel-grade part)

```
 LAYER 1  JWT  ── identity from the token only; the model never sees or chooses it
 LAYER 2  RLS  ── Postgres policy per table; an unscoped session sees ZERO rows
                  (owner bypass: app connects as hr_app, never as the owner)
 LAYER 3  tools ─ read-only, schema-validated, no employee_id parameter to abuse
 LAYER 4  guard ─ rules + LLM tripwire on HR notes; fail closed when unsure
 LAYER 5  logs ─ pseudonymous IDs + tool names + verdicts; NEVER pay values,
                  note bodies, tool results, or full answers (tested)
```

Rollout safety: shadow (serves nobody) → traffic % → full; any rollback is one
`model_flags` row flip, recorded with `(feature, prompt_ver, model_id)` on every trace.

## Real data

- **Bitext customer-support** (26.9k rows, Hugging Face) — downloaded once, cached, and
  remapped into the 40-category HR taxonomy via an explicit, documented mapping
  (`modules/ticket_router/taxonomy.py::BITEXT_REMAP`). It grounds the classifier arms in
  real phrasing — and honestly demonstrates where the mock arms stop generalizing.
- Payslips/attendance/loans/notes are **synthetic** (pinned dates + seed): the generator
  is part of the deliverable, exactly the "human-verified synthetic golden dataset" the
  S1 answer proposes.

## What's real vs mock

| Real | Mock (keyless) |
|---|---|
| FastAPI, JWT, RLS, pgvector schema, Bitext download/remap, LangGraph wiring, eval machinery, flag flips, drift math | LLM calls (`MockLLM` routes on keywords), note-guard tripwire, embeddings |

Add keys and the same code paths run against OpenRouter `:free` models and Langfuse
Cloud — no rewrites, that was the point of the seams.

## Roadmap

- Add API keys → rerun eval report for the real arms (S2's Friday decision, live)
- Fine-tune the DistilBERT arm (weekend: torch + transformers, must clear ≥5 pts — S4's bar)
- Wrap the S1 toolset as a real **MCP server** (`langchain-mcp-adapters`) — contracts identical
- Nightly shadow-replay of the last day's traces in CI
