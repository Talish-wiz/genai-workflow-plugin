---
name: architecture
description: Enforces modular project structure AND the required tech stack for this GenAI-effectiveness hackathon codebase — LangSmith for all observability/evals, NVIDIA NeMo Guardrails for all guardrails, LangChain/LangGraph/deepagents chosen per use case, a mandatory plan-then-confirm workflow before any code is written, and a mandatory before/after metrics harness for every optimization. MUST be consulted before creating any new file, adding any function, writing any prompt/agent/tool, or claiming any improvement metric. Also triggers whenever the user describes a new domain/business use case to build.
---

# Project Architecture — Modular Structure Rules (Hackathon Edition)

This project targets the hackathon problem: **demonstrably improve the effectiveness of a GenAI-enabled workflow** through optimization, governance, efficiency, and responsible adoption. Every architectural decision below exists to serve one of the four evaluation lenses: **Performance & Efficiency, Trust & Governance, Value & Operations, User Experience.**

This project separates concerns strictly by folder. Before writing any code, decide which folder it belongs in. Never mix layers inside a single file.

## Workflow gate — plan first, wait for approval, then build

Before writing or editing any code for a new feature, task, or non-trivial change:

1. **Write a short plan first, in chat, not in a file.** Cover: which folders/files will be touched or created (per the layout below), which agent framework applies (LangChain/LangGraph/deepagents) and why, whether LangSmith tracing/evals or NeMo Guardrails rails are involved, **which before/after metric this change is expected to move** (token cost, latency, accuracy, hallucination rate, retrieval precision, user effort, cycle time), and any open assumptions.
2. **Stop and ask for confirmation.** Do not create files, run bash, or edit code yet. Ask something concrete like "Should I go ahead with this?" or offer 2–3 options if there's a real design choice — don't just narrate the plan and continue unprompted.
3. **Only after the user explicitly confirms** (a yes, a pick between options, or edits to the plan followed by approval), start implementation, following the directory layout and rules below exactly.
4. **Exceptions:** trivial one-line fixes, typo corrections, or a change the user already specified in full detail (exact file + exact code) don't need a fresh plan-and-confirm cycle — use judgment, but default to confirming when a plan would touch more than one file or introduce a new dependency/pattern.
5. If the user's request is ambiguous about scope, the plan step doubles as the clarifying question — don't guess silently on anything structurally significant.

### Domain use-case intake (hackathon-specific)

When the user gives a **new domain/business use case** (e.g., "insurance claims triage", "HR policy assistant", "customer escalation prediction"), the plan in step 1 must additionally include:

- **Business scenario framing:** the workflow being optimized, its users, and the business objective it aligns to.
- **Baseline definition:** what the "before" (naive/unoptimized) version looks like — this gets built or simulated first so the before/after comparison is real, not eyeballed.
- **Optimization levers to apply:** at least pick from — prompt optimization, retrieval improvements (chunking/metadata/reranking/grounding), caching, model routing, batching, guardrails, evaluation-driven iteration.
- **Metric plan:** which metrics will be logged to LangSmith for both baseline and optimized runs, and how they map to the four evaluation lenses.
- **Governance plan:** which NeMo Guardrails rails apply, what gets audit-logged, where human-in-the-loop review or escalation/fallback sits.

## Required stack — non-negotiable

- **Observability & evaluation = LangSmith.** No other tracing/observability tool (no raw print-logging, no custom dashboard, no other APM, no Langfuse) is used for tracing LLM calls, agent runs, or tool calls. If it's not going through LangSmith, it doesn't count as observability in this project.
- **Application logging = Python's built-in `logging` module.** Separate from LangSmith. LangSmith traces LLM/agent behavior for evaluation and demo purposes; `logging` covers ordinary application concerns — startup/shutdown, request lifecycle, errors, exceptions, guardrail blocks, warnings. Never use bare `print()` for anything beyond quick local debugging you intend to remove.
- **Guardrails = NVIDIA NeMo Guardrails.** All input/output/dialog/retrieval rails (jailbreak & prompt-injection checks, PII masking, topical restriction, self-check output moderation, fact-checking/groundedness rails) go through NeMo Guardrails config. Do not hand-roll regex-based PII detection or bring in a different guardrails library (no LLM Guard, no Guardrails-AI) unless explicitly told to add one alongside it.
- **Agent code = LangChain / LangGraph / deepagents**, chosen per component based on the rule below — not mixed arbitrarily, and not substituted with a different agent framework.

### Which agent framework for which use case

- **LangChain** — use for a single, mostly-linear chain: one prompt → one or a few tool calls → one output. No branching state machine needed. Good for simple retrieval-then-answer flows, single-purpose tools, or utility chains called by an agent.
- **LangGraph** — use whenever the flow needs explicit state, branching, loops, or multiple cooperating nodes (e.g., a router that hands off between a retrieval agent, a validation agent, and a response agent; anything with conditional edges or retries). This is the default for anything you'd call "the agent system" in your presentation — it belongs in `/graph/`.
- **deepagents** — use for a task that needs a planner-executor pattern with its own sub-task decomposition, longer-horizon autonomy, or file-system/workspace-style state across many steps (e.g., a research or multi-step build agent that plans, executes, and revises its own plan). Don't reach for deepagents for something a single LangGraph node already handles — only bring it in when the task genuinely needs autonomous multi-step planning.
- If unsure which of the three a new feature needs, default to the simplest (LangChain first, then LangGraph, then deepagents) rather than over-engineering.

## Directory layout

```
/prompts/           # ALL prompt text lives here. Nothing else.
    system/         # system prompts, one .md or .txt per agent
    templates/      # reusable prompt fragments / few-shot examples
    __init__.py     # loader functions only (load_prompt(name) -> str)

/agents/            # one file per agent/node. No prompt text, no tool logic.
    base.py         # shared agent base class/interface
    <agent_name>.py # imports its prompt from /prompts, its tools from /tools

/tools/             # one file per tool (or tightly related tool group)
    <tool_name>.py  # pure function/class, no agent logic, no prompt text

/graph/             # LangGraph assembly ONLY — wiring, not logic
    state.py        # shared state schema (TypedDict/Pydantic)
    build_graph.py  # nodes + edges assembly, imports from /agents
    routing.py      # conditional edge / router logic if complex enough to isolate

/rag/               # retrieval-specific code, kept separate from agents
    ingest.py       # chunking strategy + metadata enrichment live here
    retriever.py    # incl. reranking / hybrid search if used
    vectorstore.py

/observability/     # ALL LangSmith tracing, metrics, and evaluation code lives here
    langsmith_client.py  # single shared langsmith.Client instance, read from /config
    tracing.py           # @traceable wrappers + run-tree helpers; LangChain/LangGraph
                         # auto-tracing is enabled via env (LANGSMITH_TRACING) in /config,
                         # but any non-LangChain function that must appear as its own span
                         # gets wrapped here
    evaluators.py        # LLM-as-judge + deterministic evaluators, run via
                         # langsmith.evaluate() against LangSmith datasets
    datasets.py          # dataset creation/upload for baseline & optimized example sets
    benchmarks.py        # BEFORE/AFTER harness — runs the same dataset through baseline
                         # and optimized pipelines, aggregates token cost, latency,
                         # accuracy, hallucination rate, retrieval precision
    metrics.py           # KPI aggregation read by the dashboard / demo

/guardrails/        # Guardrails layer — built on NVIDIA NeMo Guardrails
    nemo/                # the NeMo Guardrails config directory (loaded as one unit)
        config.yml       # models, rails: input/output/retrieval flows, exceptions
        prompts.yml      # self-check input/output prompt overrides (if customized)
        rails/           # Colang flows (.co) — topical rails, refusals, escalation flow
        actions.py       # custom actions (e.g., RBAC check, audit-log emit) registered
                         # with the rails runtime
    runner.py            # single wrapper: loads RailsConfig.from_path("guardrails/nemo"),
                         # exposes guarded generate()/stream(); the ONLY place LLMRails
                         # is instantiated
    rbac.py              # role-based access checks, independent of NeMo (called from
                         # actions.py and /api middleware)
    validators.py        # custom schema/regex checks not covered by NeMo rails

/api/               # FastAPI/Node routes — THIN. No business logic here.
    routes/
    dependencies.py
    middleware.py

/config/            # settings, env loading, constants, logging setup
    settings.py         # incl. LANGSMITH_TRACING, LANGSMITH_API_KEY, LANGSMITH_PROJECT
    logging_config.py   # single place that configures Python's logging module

/schemas/           # Pydantic/Zod models shared across layers
    <domain>.py

/tests/             # mirrors the top-level structure 1:1
    test_agents/
    test_tools/
    test_graph/
    test_api/
```

## Observability, evaluations & guardrails — mandatory for this project

**Observability (LangSmith):**
- Tracing is enabled globally via env in `/config/settings.py` (`LANGSMITH_TRACING=true`, `LANGSMITH_API_KEY`, `LANGSMITH_PROJECT`). LangChain/LangGraph runs are then traced automatically — never instantiate a LangSmith client inline inside an agent or route; the shared client lives in `/observability/langsmith_client.py`.
- Every tool call and agent node should appear as its own child run in the trace, not one big blob per request. LangGraph nodes get this for free; any plain-Python function that matters (guardrail checks, rerankers, cache lookups) gets `@traceable` from `/observability/tracing.py`.
- **Session IDs and user IDs must be attached to every run as metadata/tags** (`metadata={"session_id": ..., "user_id": ...}`) so you can filter by session in the LangSmith UI during the demo.
- Use two LangSmith **projects** (or a `variant: baseline|optimized` tag) so baseline and optimized traces are separable at a glance — this is what powers the before/after story.

**Application logging (Python `logging` module):**
- Logging is configured once, in `/config/logging_config.py` (handlers, formatter, level from an env var like `LOG_LEVEL`). Every other file calls `logging.getLogger(__name__)` — never configures handlers itself.
- Use levels correctly: `DEBUG` for verbose dev detail, `INFO` for normal lifecycle events, `WARNING` for recoverable issues (a rail flagged but didn't block, a retry), `ERROR`/`EXCEPTION` for failures — use `logger.exception(...)` inside `except` blocks.
- `/api/` middleware logs every request in and response out (status, latency, route) at `INFO`, and unhandled exceptions at `ERROR` before returning a clean error response.
- Guardrail blocks and evaluation failures get **both**: LangSmith feedback/metadata on the run (for the demo dashboard) and a `logging` line (for debugging/ops) — complementary, not interchangeable.
- Never log raw PII, secrets, or full prompts containing user data at `INFO` or above — if needed for debugging, log at `DEBUG` and ensure `DEBUG` isn't enabled in anything you'd screen-share.

**Evaluations (LangSmith):**
- `/observability/evaluators.py` holds both deterministic evaluators (schema validity, required-field checks, exact/fuzzy match where applicable) and LLM-as-judge evaluators for open-ended quality (helpfulness, groundedness against retrieved context, hallucination detection).
- Evaluations run via `langsmith.evaluate()` against **LangSmith datasets** (built in `/observability/datasets.py`), not ad hoc — this is what lets you show KPI dashboard numbers backed by logged runs instead of eyeballed examples.
- At minimum: a **groundedness score** for every RAG-backed response, a **task-success score** for every agent run, and **retrieval precision** on the eval dataset.
- Online feedback: attach scores to production traces with `client.create_feedback(run_id, key, score)` so guardrail outcomes, user thumbs-up/down, and judge scores all live on the same trace.
- **Before/after harness:** `/observability/benchmarks.py` runs the identical dataset through the baseline and optimized pipelines and emits a comparison table (token cost, latency p50/p95, accuracy, hallucination rate, retrieval precision). Any improvement claimed in the deck must come from this harness.

**Guardrails (NVIDIA NeMo Guardrails):**
- The rails config lives entirely in `/guardrails/nemo/` and is loaded exactly once via `/guardrails/runner.py` (`RailsConfig.from_path` → `LLMRails`). Never instantiate `LLMRails` or load rails config anywhere else.
- **Input rails** (checked before anything reaches an agent): jailbreak/prompt-injection detection (self-check input), PII detection/masking (Presidio-backed `mask sensitive data` or `detect_pii` action), and topical rails restricting the assistant to the business domain.
- **Output rails** (checked before a response leaves `/api/`): self-check output moderation (toxicity/unsafe content), PII masking on the way out (in case a model echoes retrieved PII), and — for RAG answers — the fact-checking/groundedness rail against retrieved chunks.
- **Escalation/fallback is a rail, not an afterthought:** define a Colang flow in `/guardrails/nemo/rails/` for "cannot answer / low confidence / blocked" that returns a graceful fallback message and flags the request for human review. This is your hackathon "human-in-the-loop + fallback UX" evidence.
- Guardrail outcomes (blocked/flagged/passed + which rail triggered) are recorded on the LangSmith trace as feedback/metadata via a custom action in `/guardrails/nemo/actions.py`, so a blocked request shows up in the same dashboard as everything else — don't let rail events disappear into a separate log file.
- Never call rails inline inside an agent or tool file — always through `/guardrails/runner.py` (or as `/api/` middleware) so rail config stays in one place.
- RBAC stays in `/guardrails/rbac.py` and is enforced at the `/api/` boundary; NeMo actions may *consult* it but do not own it.

## Hackathon alignment — the before/after contract

Every optimization feature must map to at least one expected-outcome bucket, and the mapping goes in the plan (workflow gate step 1):

| Expected outcome (problem statement) | Where it lives in this repo |
|---|---|
| Prompt effectiveness, lower token spend | `/prompts/` versioning + `/observability/benchmarks.py` token deltas |
| Retrieval quality (chunking, metadata, ranking, grounding) | `/rag/` + retrieval-precision evaluator |
| Reliability, fewer hallucinations, citations | groundedness rail + groundedness evaluator + citation formatting in agents |
| Cost reduction (caching, routing, batching, model selection) | `/tools/` or `/graph/routing.py` (model router), cost per run from LangSmith |
| Measurable business impact | `/observability/metrics.py` → demo dashboard (cycle time, adoption proxies, ROI calc) |
| Responsible AI & governance | `/guardrails/nemo/` + audit logging + human-in-the-loop rail + LangSmith auditability |
| User experience | thin `/api/` + frontend; fallback/escalation rail; latency budget in benchmarks |

Rules of the contract:
1. **No metric without a trace.** A number goes in the deck only if it can be reproduced from LangSmith runs via `/observability/benchmarks.py`.
2. **Baseline first.** When building a new use case, the naive baseline pipeline is built (or a frozen "v0" tag is kept) before optimizations land, so the comparison is honest.
3. **Only claim what's implemented.** Never present a feature, rail, or metric in demo material that isn't actually wired up in this repo.

## Hard rules

1. **Prompts never live inline in code.** If you're about to write a multi-line string that's a system prompt, few-shot example, or instruction block — stop, put it in `/prompts/`, and import it with a loader function instead. (NeMo's own `config.yml`/`prompts.yml`/Colang content is the one exception — it lives in `/guardrails/nemo/` because NeMo loads that directory as a unit.)
2. **Route handlers in `/api/` may only:** parse the request, call into `/agents` or `/graph` (through the guardrails runner), and return the response. No prompt construction, no LLM calls, no business logic directly in a route.
3. **`/graph/build_graph.py` only wires nodes and edges.** It should read like a table of contents. If a routing function is more than ~10 lines, move it to `/graph/routing.py`.
4. **Each agent file in `/agents/` imports its prompt and tools — it does not define them.** An agent file should mostly be: load prompt → bind tools → define node function.
5. **Tools are single-purpose.** If a tool file is doing more than one distinct capability, split it.
6. **Observability code never gets duplicated per-agent.** All LangSmith client usage, `@traceable` wrappers, dataset code, and evaluators live in `/observability/` and get imported wherever needed.
7. **Guardrails are applied at the `/api/` or `/graph/` boundary** via `/guardrails/runner.py` — never scattered inline inside agent logic.
8. **No bare `print()` statements in committed code.** Use `logging.getLogger(__name__)`; configuration lives only in `/config/logging_config.py`.
9. **Schemas are defined once in `/schemas/` and imported everywhere** — never redefine the same shape in two files.
10. **One new capability = one new file**, not a new function bolted onto an existing unrelated file. If unsure where something goes, ask which folder above it fits before writing it.
11. **Every optimization PR-sized change states its target metric** and, once merged, gets a benchmark run to confirm (or honestly report) the delta.

## Self-check before writing any file

Ask in this order:
- Is this prompt text? → `/prompts/`
- Is this NeMo rails config/Colang/custom action? → `/guardrails/nemo/`
- Is this a tool (does one thing, callable)? → `/tools/`
- Is this an agent/node definition? → `/agents/`
- Is this graph wiring (nodes/edges/state)? → `/graph/`
- Is this retrieval-specific? → `/rag/`
- Is this tracing/eval/benchmark? → `/observability/`
- Is this validation/RBAC/rail wrapping? → `/guardrails/`
- Is this a route? → `/api/` (thin only)
- Is this a shared data shape? → `/schemas/`

If code doesn't cleanly fit one of these, flag it to the user rather than guessing a location.
