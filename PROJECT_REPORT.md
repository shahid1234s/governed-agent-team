# A Governed Multi-Agent Business Team

### Autonomous Research, Analysis & Outreach — with Human-in-the-Loop Safety

*Prepared for: Prospective Partner / [Company Name]*

---

## 1. Executive Summary

This project demonstrates a **production-minded autonomous agent system** that takes a single business objective — *"research a target company and prepare a qualified outreach package"* — and executes it end-to-end through a coordinated team of specialised AI agents.

Unlike typical "chatbot" demos, this system is built around **governance first**: every action an agent takes is evaluated by a policy engine before it runs, external communications require human approval, and a hard spend limit prevents runaway cost. Every step is recorded in an immutable audit trail.

**What it does, in one line:**

> A Manager plans the work → a Researcher gathers intelligence → an Analyst qualifies the opportunity → a Writer drafts and (with your approval) sends outreach — all under continuous policy control, producing a polished, downloadable deliverable.

**Headline capabilities**

- A **4-agent team** that plans, researches, analyses, and acts autonomously.
- A **governance engine** that allows, blocks, or requires approval for every action (least-privilege, spend limit, forbidden tools).
- **Human-in-the-loop**: any external communication (email, meeting) pauses for your sign-off.
- A **reliability layer**: simulated failures auto-recover via retry-with-backoff, fully logged.
- A **live dashboard** showing agent activity, metrics, and a final deliverable you can download as Text, Markdown, HTML, JSON, Word, or PDF — or copy in one click.
- **Reproducible by design**: a deterministic mock mode runs the entire demo with zero external cost; a Gemini-powered mode uses a real LLM brain with Google Search Grounding.

---

## 2. The Problem / Why Now

Enterprises are racing to deploy autonomous agents, but ungoverned agents create real risk:

- **Unsafe actions** — an agent could send emails, spend money, or call forbidden systems without oversight.
- **No auditability** — when something goes wrong, there is no record of *who decided what*.
- **Runaway cost** — unbounded LLM and tool usage with no hard limits.
- **Brittle reliability** — a single tool outage stalls the whole workflow.

This project answers each of those risks directly, and proves the pattern with a working, demonstrable system.

---

## 3. The Solution

A thin **orchestrator** conducts a team of role-specialised agents through a pipeline. Agents share a common **memory**, call a curated set of **governed tools**, and are supervised by a **governance engine** on every action. The result is assembled into a human-readable deliverable and surfaced in a clean dashboard.

**Core features**

- **Planner-led decomposition** — the Manager turns a vague goal into an ordered plan.
- **Shared memory** — research and analysis are written once and read by downstream agents (no re-work, no drift).
- **Structured outputs** — agents return validated, schema-checked results (JSON mode).
- **Observable by default** — a live event stream, audit log, and cost/risk metrics.
- **Deliverable-first** — the system's output is a complete, branded document, not a chat transcript.

---

## 4. Architecture

The system is a single FastAPI backend that also serves a statically-built Next.js dashboard (same-origin, no CORS friction). Every agent action is funnelled through a shared **Agent Context** that enforces tools, memory, governance, approvals, audit, and metrics.

```text
                          +-----------------------+
                          |     USER / OPERATOR   |
                          |  set goal, approve    |
                          +-----------+-----------+
                                      |
                                      v
                          +-----------------------+
                          |    NEXT.JS DASHBOARD   |
                          | goal / activity /      |
                          | metrics / approvals /  |
                          | OUTPUT panel           |
                          +-----------+-----------+
                                      |
                                      v   HTTPS (same origin)
          +-----------------------------------------------+
          |            FASTAPI BACKEND (Python)           |
          +-----------------------------------------------+
                                      |
                                      v
   +--------------------+     +-----------------------------+
   |     API LAYER      | --> |    WORKFLOW ORCHESTRATOR    |
   |  /start /output    |     |    Manager -> Researcher    |
   |  /stream /approvals|     |    -> Analyst -> Writer     |
   +--------------------+     +-------------+---------------+
                                       |
                                       v   every agent action
   +-----------------------------------------------------+
   |                  AGENT CONTEXT                     |
   |  Tools | Memory | Governance | LLM (Gemini) | Observe |
   +-----------------------------------------------------+
                                       |
                    +------------------+-------------------+
                    |                                      |
                    v                                      v
   +-----------------------------+      +-----------------------------+
   |      GOVERNANCE ENGINE      |      |       OUTPUT ASSEMBLY       |
   |  ALLOW / REQUIRE_APPROVAL / |      |  6-section deliverable ->   |
   |  BLOCKED                    |      |  Dashboard Output Panel     |
   +-----------------------------+      +-----------------------------+
```

---

## 5. The Agent Team

| Agent | Role | Responsibility | Allowed Tools (least-privilege) |
|-------|------|----------------|---------------------------------|
| **ManagerAgent** | Team Lead / Orchestrator | Decomposes the objective into an ordered, delegated plan | `search_company`, `summarize_information`, `draft_message`, `cost_estimator` |
| **ResearcherAgent** | Researcher | Gathers and structures company intelligence into shared memory | `search_company`, `get_company_profile`, `summarize_information` |
| **AnalystAgent** | Analyst | Reads shared research; scores fit and qualification; recommends next step | `search_company`, `summarize_information` |
| **WriterAgent** | Writer / Action | Drafts outreach, demonstrates the blocked premium action, and requests approval before any external send | `draft_message`, `send_email`, `schedule_meeting`, `cost_estimator`, `summarize_information` |

Each agent emits lifecycle events (`AGENT_STARTED`, `AGENT_COMPLETED`, `PLAN_CREATED`) that drive the live dashboard.

---

## 6. The Toolset

| Tool | Class | Purpose |
|------|-------|---------|
| `search_company` | Read | Find a target company by query |
| `get_company_profile` | Read | Retrieve the structured company profile (also the deliberate failure-injection target) |
| `summarize_information` | Read | Condense findings into concise facts |
| `draft_message` | Draft | Compose a personalised outreach message |
| `send_email` | **External** | Send outbound email — requires human approval |
| `schedule_meeting` | **External** | Schedule a meeting — requires human approval |
| `cost_estimator` | Guard | Estimate action cost; enforces the spend limit |
| `web_search` | Research | Google Search Grounding via the Gemini provider (live web facts + cited sources) |
| `government_data` | Research | Retrieve official/regulatory data from governed `.gov` / `europa.eu` / etc. domains |

All tools are registered centrally and are the *only* capabilities agents can invoke — there is no unconstrained "run anything" escape hatch.

---

## 7. Governance & Safety

The governance engine evaluates **every** action *before* execution and returns one of three outcomes: `ALLOWED`, `REQUIRE_APPROVAL`, or `BLOCKED`. Policies are declarative and centralised — they can be reviewed and changed without touching agent code.

| Policy ID | Trigger | Outcome |
|-----------|---------|---------|
| `POL-TOOL-FORBIDDEN` | Tool is unknown or explicitly forbidden (e.g. `move_money`, `execute_dangerous_action`) | **BLOCKED** |
| `POL-TOOL-PERMISSION` | Agent tries a tool outside its least-privilege allow-list | **BLOCKED** |
| `POL-SPEND-LIMIT` | Estimated cost exceeds the configured limit (default **$5.00**) — e.g. a $50 premium outreach | **BLOCKED** |
| `POL-APPROVAL-EXTERNAL` | Action is external communication (`send_email`, `schedule_meeting`) | **REQUIRE_APPROVAL** |
| `POL-ALLOW` | Action is within policy for the agent and tool | **ALLOWED** |

**Why this matters:** a paid/premium action is stopped *automatically* by the spend guard, and no email ever leaves the system without a human clicking "Approve". Forbidden and out-of-scope tools are rejected outright. Every decision is recorded with its policy ID, risk level, and rationale.

**Governance decision flow (every action):**

```text
                    agent action
                         |
                         v
              +-----------------------+
              |   GOVERNANCE ENGINE   |
              |       evaluate()      |
              +-----------+-----------+
                          |
              +-----------+-----------+
              |           |           |
              v           v           v
        +----------+ +-----------+ +----------+
        |  ALLOW   | | APPROVE   | | BLOCKED  |
        | proceed  | | wait for  | | stop,    |
        |          | | human     | | log why  |
        +----------+ +-----------+ +----------+
```

---

## 8. Reliability

Autonomous systems must survive tool failures. The system includes a **failure injector** and a **recovery executor**:

- A configured tool call (e.g. `get_company_profile`) is forced to fail a set number of times.
- The executor retries with **exponential backoff** (up to 4 attempts) and, on success, records a `RECOVERY_SUCCEEDED` event.
- Every failure and recovery is published to the event bus and written to the **audit log** — so resilience is observable, not invisible.

This turns a brittle single-shot call into a demonstrably robust pipeline.

---

## 9. Observability

| Component | What it provides |
|-----------|------------------|
| **EventBus** | Real-time stream of agent/tool/workflow events powering the live dashboard |
| **Auditor** | Immutable, queryable audit log of every action, decision, failure, and recovery |
| **MetricsCollector** | Tracks estimated cost (USD), tool calls, failures, recoveries, and approvals resolved/required |

The result is a system you can **watch, audit, and trust** — not a black box.

---

## 10. Technology Stack

| Layer | Technology |
|-------|------------|
| **Backend** | Python 3.14, FastAPI, Uvicorn, Pydantic v2, httpx, python-dotenv |
| **LLM Brain** | Google Gemini (`gemini-flash-lite-latest`) — `generate`, `structured` (JSON mode), and `grounded_generate` (Google Search Grounding); a `mock` provider enables zero-cost reproducible demos |
| **Document generation** | `python-docx` (Word), `fpdf2` (PDF), plus Text/Markdown/HTML/JSON renderers |
| **Frontend** | Next.js 14, TypeScript, Tailwind CSS (statically exported and served by the backend) |
| **Delivery** | Single-origin deployment — backend serves both API and dashboard at one port |

---

## 11. Demo Walkthrough (the built-in demonstration dataset)

The bundled demonstration uses a realistic, fully-seeded scenario — **"Helios Robotics"**, a 145-person industrial-automation company that recently launched a collaborative robot arm.

1. **Objective** — *"Research Helios Robotics and prepare a qualified outreach package."*
2. **Manager** produces an ordered plan delegating to Researcher → Analyst → Writer.
3. **Researcher** searches the company and retrieves its profile (the step that deliberately fails once, then auto-recovers), then writes structured findings to shared memory.
4. **Analyst** reads the research and returns a **fit score of 0.86**, qualification *"Qualified – High intent"*, and a recommendation to proceed.
5. **Writer** drafts a personalised message to the VP of Operations, then:
   - Attempts a **$50 premium outreach** → **blocked** by the $5 spend limit (`POL-SPEND-LIMIT`).
   - Attempts the **external email send** → **paused for human approval** (`POL-APPROVAL-EXTERNAL`).
6. Once approved, the workflow completes and the **Output Deliverable** is assembled.

This single run showcases planning, memory sharing, failure recovery, spend control, and human-in-the-loop — all in under a minute.

---

## 12. The Output Deliverable

The system's product is not a chat log — it is a clean, sectioned business document assembled server-side:

| # | Section | Contents |
|---|---------|----------|
| 1 | **Objective** | The original goal |
| 2 | **Company Snapshot** | Name, industry, key pain points |
| 3 | **Research Summary** | Condensed intelligence |
| 4 | **Analyst Assessment** | Fit score (badged), qualification, recommendation |
| 5 | **Outreach Package** | Subject, message, delivery status, any governance notes (e.g. premium blocked) |
| 6 | **Governance & Cost Summary** | Status, approvals resolved, estimated cost |

**Download in one click** as **Text, Markdown, HTML, JSON, Word (.docx), or PDF** — or use the **Copy** button to drop the full text onto your clipboard instantly.

---

## 13. Business Value — for [Company Name]

Because the governance, memory, and tool layers are decoupled from the demo scenario, the same system adapts to **any** objective your team cares about:

- **Controlled autonomy** — agents act freely *within* guardrails, and escalate only the decisions that matter to a human.
- **Cost predictability** — hard spend limits and live cost metrics eliminate bill shock.
- **Trust & compliance** — every action is policy-checked and audited, satisfying risk and compliance reviews.
- **Resilience** — built-in retry/recovery keeps operations running through transient failures.
- **Reusable deliverables** — each run produces a shareable, branded document your team can act on immediately.

In short: **you get the leverage of autonomous agents without surrendering oversight.**

---

## 14. How to Run / Try It

```bash
# Backend (serves API + dashboard on http://localhost:8000)
cd backend
$env:PYTHONPATH="."
.\.venv\Scripts\python.exe -m uvicorn app.main:app --port 8000 --reload

# Open the dashboard
#   http://localhost:8000/
```

- **Zero-cost demo:** set `LLM_PROVIDER=mock` (the deterministic provider runs the full scenario with no external calls).
- **Live brain:** keep `LLM_PROVIDER=gemini` with a valid `GEMINI_API_KEY` in `backend/.env` to use the real Gemini model with Google Search Grounding.
- **Verify:** backend test suite (`pytest`) covers orchestration, governance, failure recovery, and all six output formats (27 passing).

---

*This report describes the system as built. The demonstration dataset ("Helios Robotics") is illustrative seed data and can be replaced with any target without code changes.*
