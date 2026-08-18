DEMO-video = https://mega.nz/file/l2U3USIJ#Vujep4k4lnq8nOWKyQzbadwazeTH4sMxwfqRNuLbuD4
# Governed Multi-Agent Business Team

A small, self-contained demo that proves we can build **reliable, governed, observable
multi-agent systems** — the exact engineering surface Naïve cares about (orchestration,
governance, shared memory, and reliability).

It runs a realistic autonomous business workflow: given a high-level objective, a team of
four AI agents (Manager → Researcher → Analyst → Writer/Action) research a company, evaluate
fit, draft outreach, and request human approval before any external action — with every step
governed, audited, and observable, including a deliberately injected failure that the system
detects and recovers from.

> This is **not** a clone of Naïve. It is a reference implementation of the *outsourceable*
> capability identified during research: **multi-agent orchestration + governance**.

## Highlights

- **Multi-agent orchestration** — a lightweight, owned orchestrator (no framework lock-in).
- **Shared memory** — a SQLite-backed, typed memory store agents read/write.
- **Tool calling** — a permissioned tool registry (all tools are safe mocks).
- **Governance / policy engine** — allow / require-approval / block, spend limits, least-privilege tool permissions.
- **Human-in-the-loop** — real approval queue; workflow pauses until a human decides.
- **Audit logging** — every important action emits a structured, tamper-evident event.
- **Failure detection + recovery** — injected failure, retry-with-backoff, recorded recovery.
- **Cost / task monitoring** — agent calls, tool calls, failures, duration, estimated tokens/cost.
- **Provider-agnostic LLM** — HY3 (OpenAI-compatible) by default, with a deterministic `mock`
  provider for reliable, keyless demos and an `anthropic` provider for switching.

## Architecture

See [ARCHITECTURE.md](./ARCHITECTURE.md). Runtime flow and component responsibilities are there.

## Quick start

### Backend (terminal 1)
```bash
cd backend
py -m venv .venv
.\.venv\Scripts\Activate.ps1        # or: cmd /c ".venv\Scripts\activate.bat"
pip install -r ..\requirements.txt
cp ..\.env.example .env             # optional; defaults work with the mock LLM
py -m uvicorn app.main:app --reload --port 8000
```
Health check: http://127.0.0.1:8000/api/health

### Frontend (terminal 2)
```bash
cd frontend
cmd /c npm install
cmd /c npm run dev
```
Open http://localhost:3000

> PowerShell blocks `npm.ps1`; prefix npm commands with `cmd /c` (or run
> `Set-ExecutionPolicy -Scope Process RemoteSigned`). The backend `py` launcher is Python 3.14.

### Single-process (optional)
```bash
cd frontend && cmd /c npm run build   # outputs frontend/out
# FastAPI serves it automatically at http://127.0.0.1:8000/
```

## Demo

See [DEMO.md](./DEMO.md) for the exact 2-minute and 10-minute scripts.

## Tests

```bash
cd backend
.\.venv\Scripts\Activate.ps1
py -m pytest -q
```
22 tests cover governance (allow/deny/spend-limit/tool-permission), memory, audit + secret
redaction, tool behavior, failure injection + recovery, and a full end-to-end workflow
(approved and rejected paths) plus invalid-input handling.

## Configuration

All behavior is driven by environment variables — see `.env.example`. Key knobs:

| Variable | Default | Purpose |
|---|---|---|
| `LLM_PROVIDER` | `hy3` | `hy3` \| `mock` \| `anthropic` |
| `LLM_BASE_URL` / `LLM_API_KEY` / `LLM_MODEL` | OpenAI-shaped | HY3 / OpenAI-compatible endpoint |
| `SPEND_LIMIT_USD` | `5.00` | governance blocks actions above this estimate |
| `APPROVAL_REQUIRED_ACTIONS` | `send_email,schedule_meeting` | external actions needing a human |
| `FAILURE_INJECT_TOOL` / `FAILURE_INJECT_COUNT` | `get_company_profile` / `1` | deliberate failure for the demo |

If `LLM_PROVIDER` needs a key and none is set, the system **auto-falls back to `mock`** so the
demo never fails to start.

## Security posture

No real email is sent, no money moves, and no external systems are touched — every "action"
tool is a mock. Secrets are redacted from audit logs. Inputs are validated. Tool permissions are
least-privilege. A `mock` mode means the whole demo runs with zero credentials.

## Repository layout

```
governed-multi-agent-team/
├── README.md  ARCHITECTURE.md  DEMO.md  .env.example  requirements.txt
├── backend/app/{llm,agents,memory,governance,tools,observability,failures,approvals,tests}
└── frontend/{app,components,lib}
```
