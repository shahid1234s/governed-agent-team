# Demo Guide

Two scripts: a tight **2-minute** flow for a casual walkthrough, and a **10-minute** technical
deep-dive for a CTO / architect. Both use the default `mock` LLM so the demo is deterministic
and requires no API keys. (To make the agents "think" with a real model, put a key in
`backend/.env` under `LLM_API_KEY` / `LLM_MODEL`.)

> Prerequisite: backend on `http://127.0.0.1:8000` and frontend on `http://localhost:3000`.

---

## 2-Minute Demo (happy path)

**0:00 — Set the stage (15s).** "This is a governed multi-agent ops team. Four agents
research a company and prepare outreach, fully governed and observable."

**0:15 — Mission.** Type the goal (default is pre-filled):
> *Research Helios Robotics and prepare a qualified outreach package.*
Click **Run workflow**.

**0:30 — Manager plans.** "Mission / Goal" shows the objective; "Live Workflow" shows the
plan the ManagerAgent produced and the team starting to execute.

**0:45 — Researcher + failure/recovery.** Watch the Researcher call `get_company_profile`.
Point at **Failure & Recovery**: the tool *fails once* (injected), the system **detects** it,
**retries with backoff**, and **recovers** — the workflow continues without human involvement.

**1:00 — Analyst.** Analyst reads shared memory and writes a qualification. Open **Shared
Memory** to show agents truly read/write the same store.

**1:15 — Writer + governance BLOCK.** Writer drafts outreach, then tries a *premium* paid
action. **Governance** shows it **BLOCKED** (estimated cost $50 exceeds the $5 spend limit).
"Policy is enforced at the tool boundary, not in the prompt."

**1:30 — Human approval.** Writer calls `send_email` → **REQUIRE_APPROVAL** → the workflow
pauses (`AWAITING_APPROVAL`) and the **approval modal** appears with action, recipient, risk,
and reason. Click **Approve**.

**1:45 — Completion + audit.** Workflow flips to **COMPLETED**. Open **Audit Trail**: every
action is logged with timestamp, agent, tool, policy, result, latency. Open **Metrics**:
agent/tool calls, failures, recoveries, approvals, duration, estimated cost.

**1:55 — Recap (optional).** Click **Trigger failure** and re-run to show the failure path on
demand; or click **Reject** in the modal to show the `REJECTED` path.

---

## 10-Minute Technical Demo

1. **Architecture (1 min).** Open `ARCHITECTURE.md` / this file. Walk the component map:
   orchestrator → `AgentContext` → governance → approvals → executor → memory → auditor →
   event bus. Emphasize: no monolith, interfaces everywhere, provider-agnostic LLM.
2. **Orchestration (1.5 min).** Show `orchestrator.py` and `context.py`. One `call_tool` path
   enforces governance + approval + failure/recovery + audit. Agents are pure logic.
3. **Shared memory (1 min).** Show `memory/store.py` (SQLite, typed, upsert-by-key). In the UI,
   watch memory keys appear as agents run. Note Analyst reads what Researcher wrote.
4. **Governance engine (2 min).** Show `governance/engine.py` + `policies.py`: `ALLOWED`,
   `REQUIRE_APPROVAL`, `BLOCKED`; spend limit; least-privilege tool permissions per agent.
   In the UI, show the BLOCK (premium) and REQUIRE_APPROVAL (email) decisions.
5. **Tool permissions (1 min).** Point out `TOOL_PERMISSIONS`: an Analyst calling `send_email`
   is BLOCKED — enforced, not aspirational. (Covered by a unit test.)
6. **Human approval (1 min).** Show `approvals/service.py`: the workflow suspends on a future
   and resumes on resolution; the UI modal drives `POST /api/approvals/{id}/resolve`.
7. **Audit + observability (1 min).** Show `observability/audit.py` (SQLite + secret redaction)
   and the EventBus. The dashboard's Audit Trail and live feed come straight from these.
8. **Failure injection + recovery (1 min).** Show `failures/injector.py` (configurable target)
   and `failures/recovery.py` (retry-with-backoff, recorded recovery). Trigger it live.
9. **Metrics (0.5 min).** Show `observability/metrics.py`: counts, duration, estimated tokens
   and cost (clearly labeled *estimate*).
10. **Tests (0.5 min).** `py -m pytest -q` → 22 passing: governance allow/deny, memory,
    audit+redaction, tools, failure+recovery, and full e2e (approved + rejected + invalid
    input).
11. **Switching the model (0.5 min).** Show `.env.example`: set `LLM_PROVIDER=hy3` + key to use
    a real OpenAI-compatible model with **no agent-code changes**.

---

## Talking points for a Naïve CTO

- "We can own the orchestration and governance layer, not just prompt an LLM."
- "Policy is enforced at the tool boundary; blocks and approvals can't be talked around."
- "Failures are expected and handled — detect, retry, recover, continue, and log it."
- "Everything is observable: a complete audit trail and live metrics out of the box."
- "It's provider-agnostic and swappable — drop in your runtime, memory, or models."

## Known limitations

- Tools are mocks (safe, deterministic). Wiring real tools/LLM is a config + adapter change.
- Cost is an *estimate* from token approximation, not provider billing.
- Single-tenant, file-based SQLite. Swap to Postgres/Redis for production scale.
- In-memory event bus (per process). Use Redis Streams/Kafka for multi-instance.

## Suggested next improvements

- Real vector memory for semantic retrieval; multi-session workspaces.
- Policy DSL / UI editor; streaming token UX; authn + multi-user approvals.
- Durable workflow engine (e.g., temporal) for long-running, resumable workflows.
- Comparative eval harness to measure agent reliability across scenarios.
