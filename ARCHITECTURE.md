# Architecture

A clean, modular, dependency-injected design. No monolith, no global mutable state, no
provider lock-in. Every component has one job and talks through interfaces.

## Component map

```
                         ┌─────────────────────────────────────────────┐
   Browser (Next.js)     │                  FastAPI (backend)          │
   ───────────────       │                                              │
   Dashboard  ─── REST/SSE ──►  main.py (routes, CORS, SSE)             │
                              │                                         │
                              │   WorkflowOrchestrator                  │
                              │      │ plans & sequences agents         │
                              │      ▼                                   │
                              │   AgentContext (DI surface per workflow) │
                              │      ├─ call_tool(agent, tool, action)   │
                              │      │    1. GovernanceEngine.evaluate   │
                              │      │    2. ApprovalService (HITL)       │
                              │      │    3. ToolExecutor (fail+recover)  │
                              │      ├─ write_memory / read_memory       │
                              │      └─ llm.structured (LLMProvider)      │
                              │                                          │
                              │   System (container)                      │
                              │   ├─ LLMProvider  (hy3|mock|anthropic)    │
                              │   ├─ MemoryStore  (SQLite)               │
                              │   ├─ GovernanceEngine + policies         │
                              │   ├─ ToolRegistry  (7 mock tools)        │
                              │   ├─ Auditor      (SQLite + EventBus)     │
                              │   ├─ ApprovalService                      │
                              │   ├─ FailureInjector                      │
                              │   ├─ MetricsCollector                     │
                              │   └─ EventBus     (SSE / polling feed)    │
                              └─────────────────────────────────────────────┘
```

## Data flow (one workflow)

1. **Goal** arrives at `POST /api/workflow/start`.
2. **ManagerAgent** calls the LLM and returns a `Plan` (delegated steps).
3. **ResearcherAgent** calls `search_company` then `get_company_profile`. The second is the
   injected-failure target: the `ToolExecutor` detects the failure, records it, retries with
   backoff, and recovers. Findings are written to **shared memory**.
4. **AnalystAgent** reads memory and produces a qualification/evaluation.
5. **WriterAgent** drafts outreach, attempts a *premium* action that the **governance engine
   BLOCKS** (spend limit), then calls `send_email` which **requires human approval**. The
   orchestrator pauses (`AWAITING_APPROVAL`); the UI shows a modal; on **Approve** the action
   executes, on **Reject** the workflow ends `REJECTED`.
6. Every tool call, memory write, governance decision, approval, failure and recovery is
   recorded by the **Auditor** (SQLite) and pushed to the **EventBus** (dashboard feed).
7. **MetricsCollector** tallies agent/tool calls, failures, recoveries, approvals, duration and
   estimated cost.

## Module responsibilities

| Module | Responsibility | Key types |
|---|---|---|
| `llm/` | Provider abstraction; structured JSON output, retries, timeouts | `LLMProvider`, `MockProvider`, `OpenAICompatibleProvider`, `AnthropicProvider` |
| `agents/` | Four role agents; pure logic, no infra | `ManagerAgent`, `ResearcherAgent`, `AnalystAgent`, `WriterAgent` |
| `memory/` | Typed, persistent shared state | `MemoryStore` |
| `governance/` | Policy evaluation: allow / approve / block | `GovernanceEngine`, `policies` |
| `tools/` | Permissioned tool registry (safe mocks) | `ToolRegistry` + 7 tools |
| `observability/` | Audit log, event bus, metrics | `Auditor`, `EventBus`, `MetricsCollector` |
| `failures/` | Injection + recovery | `FailureInjector`, `ToolExecutor` |
| `approvals/` | Human-in-the-loop queue | `ApprovalService` |
| `orchestrator.py` | Sequences agents, owns status transitions | `WorkflowOrchestrator` |
| `context.py` | Per-workflow DI surface agents use | `AgentContext` |
| `system.py` | Wires everything together | `System` |

## Design decisions (why)

- **Custom orchestrator, not LangGraph** — full ownership, no hidden control flow, easy to
  explain to a CTO, and demonstrates real engineering rather than prompt-chat.
- **SQLite + typed models** — zero-infra persistence that is inspectable and reproducible.
  No vector DB (unnecessary here); the `MemoryStore` interface can be swapped for one later.
- **`mock` LLM by default** — deterministic, keyless, reliable demos; the same `LLMProvider`
  interface drives HY3/anthropic with no agent-code changes.
- **Governance before execution** — every tool call is evaluated *before* it runs, so blocks
  and approvals are enforced at the boundary, not hoped for in prompts.
- **Audit + events decoupled** — the `Auditor` persists; the `EventBus` streams. The UI polls
  `/api/events` (no fragile SSE setup required), and an SSE endpoint also exists.

## Reliability mechanisms

- LLM calls: low temperature, JSON-schema structured output, retries with exponential backoff,
  timeouts.
- Tool calls: failure injection, retry-with-backoff, recorded recovery, then continue.
- Human approval: workflow suspends on a `asyncio.Future` and resumes on resolution; no polling
  loop inside the backend.
- Secrets: redacted from all audit records (`api_key`, `token`, `secret`, `password`,
  `authorization`).

## Request lifecycle (tool call with governance)

```
Agent ──call_tool──► GovernanceEngine.evaluate
                        ├─ BLOCKED      → return, no execution, audited
                        ├─ REQUIRE_APPROVAL → ApprovalService (pause) → on approve, execute
                        └─ ALLOWED      → ToolExecutor.execute (fail+recover) → audited
```
