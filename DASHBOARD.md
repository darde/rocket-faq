# Item 1: Rate Limiting, Caching, Security & Cost Optimization

Production-hardening of the backend API. Adds protection against abuse (slowapi rate limiting per endpoint), in-memory caching for repeated questions (TTLCache for RAG responses + embeddings), security headers/input validation/optional API key auth, and token budget enforcement with cost tracking.

## Endpoints

| Method | Path | Description |
|---|---|---|
| `GET` | `/health` | Health check + cache stats |
| `GET` | `/stats` | Token usage summary (daily/monthly) + cache stats |

The existing endpoints were **modified** (not new):

| Endpoint | What changed |
|---|---|
| `POST /api/chat/` | Rate limited (20/min), API key auth, input validation (max 1000 chars, control char sanitization), returns 503 on budget exceeded |
| `POST /api/eval/retrieval` | Rate limited (5/min), API key auth, `k` constrained to 1-20 |
| `POST /api/eval/judge` | Rate limited (5/min), API key auth, question max 1000 chars |
| `POST /api/eval/full` | Rate limited (5/min), API key auth |

## Key Response Codes Added

- **429** — Rate limit exceeded (with `Retry-After` header)
- **401** — Invalid/missing API key (when `API_KEY` is configured)
- **422** — Input validation failure
- **503** — Token budget exhausted

---

# Item 2: Responsible AI and Governance

AI safety layer wrapping the RAG pipeline. Input guardrails screen for PII (SSN, credit card, phone, email, account numbers), prompt injection patterns, and off-topic questions — all before reaching the LLM. Output guardrails check for PII leaks, inject disclaimers (financial/legal/tax), and flag low-confidence responses. Every interaction is audit-logged to a JSONL file. Users can rate responses with thumbs up/down.

## Endpoints

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/chat/feedback` | Submit thumbs up/down feedback for a response (by `request_id`) |
| `GET` | `/api/governance/summary` | Aggregated stats: total queries, flagged counts (PII, injection, off-topic, blocked), disclaimers breakdown, feedback summary |

The chat endpoint response was **expanded**:

| Endpoint | What changed |
|---|---|
| `POST /api/chat/` | Now returns `guardrails` object (input_pii_detected, injection_detected, off_topic, low_confidence, disclaimers_added, blocked) and `request_id` for feedback |

## Guardrails Pipeline

```
Input → PII redaction → Injection check (block if high risk) → Topic check (redirect if off-topic)
  → RAG pipeline (with redacted query)
  → Output PII leak check → Low confidence warning → Disclaimer injection
  → Audit log → Response
```

---

# Item 3: Code Review Bot, Documenter, Tech Debt Analyzer, Multi-Agent System

Three AI-powered agents that analyze the project's own codebase using the existing OpenRouter LLM. A coordinator orchestrates all agents sequentially (budget-aware). Files are processed in configurable batches (default 3 per LLM call). Reports are saved as JSON to `data/agent_reports/`.

**Agents:**
- **Code Reviewer** — quality, security, best practices findings with severity levels
- **Documenter** — generates structured per-module documentation + assembled markdown file
- **Tech Debt Analyzer** — free static analysis (TODOs, large files, missing tests, hardcoded values) + LLM-powered deep analysis (complexity, duplication, testability)

## Endpoints

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/agents/review` | Run code review agent |
| `POST` | `/api/agents/document` | Run documenter agent |
| `POST` | `/api/agents/tech-debt` | Run tech debt analyzer |
| `POST` | `/api/agents/full` | Run all agents via coordinator (combined report) |
| `GET` | `/api/agents/reports` | List all saved report files |
| `GET` | `/api/agents/reports/{filename}` | Get a specific report by filename |

All agent endpoints are rate-limited to **2/minute** (agents are expensive).

## CLI

```bash
python scripts/run_agents.py --all        # Run all agents
python scripts/run_agents.py --review     # Code review only
python scripts/run_agents.py --document   # Documenter only
python scripts/run_agents.py --tech-debt  # Tech debt only
```
