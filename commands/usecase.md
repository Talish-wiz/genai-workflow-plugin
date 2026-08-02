---
description: Start a new domain/business use case through the full hackathon workflow — intake, plan, confirm, scaffold. Usage: /usecase <describe the business scenario>
---

# /usecase — Domain use-case intake & scaffold

The user is giving you a business scenario to build as an optimized GenAI workflow: **$ARGUMENTS**

First, read `skills/architecture/SKILL.md` from this plugin (the architecture skill) and treat every rule in it as binding. Then run this exact sequence. Do NOT skip the confirmation step.

## Step 1 — Intake analysis (no code yet)

From the scenario, work out and present to the user:

1. **Business framing** — the workflow being optimized, who its users are, and the business objective (productivity, cycle time, quality, risk reduction, adoption, ROI).
2. **Baseline definition** — what the naive "before" version is (single prompt to a big model, no RAG tuning, no caching, no guardrails). This gets built or simulated first.
3. **Agent framework choice per component** — LangChain for linear chains, LangGraph for stateful/branching multi-agent flow, deepagents only for genuine planner-executor autonomy. Justify each choice in one line.
4. **Optimization levers** — pick the concrete ones that fit this domain: prompt optimization, chunking/metadata/reranking/grounding, semantic caching, model routing (cheap model for easy queries), batching, guardrails-as-quality-gate.
5. **Metric plan** — which of these will be measured in the before/after harness: token cost per query, latency p50/p95, accuracy/task success, hallucination rate (groundedness), retrieval precision, user effort/cycle time. State how each is computed and that all of them come from LangSmith runs via `/observability/benchmarks.py`.
6. **Governance plan** — which NeMo Guardrails rails apply (jailbreak/self-check input, PII masking in and out, topical rails, output moderation, groundedness/fact-check rail), where audit logging happens, and where the human-in-the-loop escalation/fallback flow sits.
7. **File plan** — the exact files to be created, each mapped to a folder from the architecture skill's directory layout.
8. **Evaluation-lens mapping** — one line each showing how the plan scores on Performance & Efficiency, Trust & Governance, Value & Operations, and User Experience.
9. **Open assumptions / questions** — anything ambiguous about scope.

## Step 2 — STOP and confirm

Present the plan and ask a concrete question ("Should I go ahead with this?" or offer 2–3 options if there's a real design choice). Do not create files, run bash, or write code until the user explicitly approves.

## Step 3 — Scaffold (only after approval)

Create the project structure exactly per the architecture skill's directory layout, including:
- `/config/settings.py` with `LANGSMITH_TRACING`, `LANGSMITH_API_KEY`, `LANGSMITH_PROJECT` env handling and `/config/logging_config.py` as the single logging setup.
- `/guardrails/nemo/` with `config.yml`, a starter Colang rails file for the domain's topical restriction + escalation/fallback flow, `actions.py` (with an action that writes rail outcomes to the LangSmith trace), and `/guardrails/runner.py` as the ONLY place `LLMRails` is instantiated.
- `/observability/` with `langsmith_client.py`, `tracing.py`, `evaluators.py` (groundedness + task-success at minimum), `datasets.py`, and `benchmarks.py` (baseline-vs-optimized harness stub that emits a comparison table).
- Baseline pipeline first, tagged/`variant=baseline`, then the optimized pipeline.
- Prompts in `/prompts/` only, agents in `/agents/`, wiring in `/graph/`, thin routes in `/api/`.

## Step 4 — Report

End with: files created, what's stubbed vs working, the next command to run, and a reminder that any demo metric must come from `/observability/benchmarks.py` (use `/baseline` to run it).
