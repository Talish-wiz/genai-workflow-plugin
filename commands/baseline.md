---
description: Run or update the before/after benchmark — baseline vs optimized pipeline on the same LangSmith dataset, producing the comparison table for the hackathon deck. Usage: /baseline [optional: which metric or dataset to focus on]
---

# /baseline — Before/after metrics harness

Focus (optional): **$ARGUMENTS**

Read `skills/architecture/SKILL.md` from this plugin first. Then:

1. **Locate the harness** at `/observability/benchmarks.py` and the dataset code at `/observability/datasets.py`. If either doesn't exist yet, propose creating them (plan-then-confirm applies) rather than benchmarking ad hoc.
2. **Verify preconditions:** a LangSmith dataset of representative queries with reference answers/contexts exists; both baseline and optimized pipelines are runnable; tracing env vars are set. If the dataset is missing, help build one (10–30 examples is enough for a hackathon) before running anything.
3. **Run both variants** through `langsmith.evaluate()` (or the harness's wrapper) with the evaluators from `/observability/evaluators.py`: token cost per query, latency p50/p95, accuracy/task success, hallucination rate (groundedness), retrieval precision.
4. **Emit the comparison table** — baseline vs optimized, absolute values plus % delta per metric — and save it as a markdown/CSV artifact the user can drop into the deck.
5. **Honesty rules:** report regressions as plainly as improvements; never estimate a number the runs didn't produce; if a metric can't be computed yet, say so and list what's needed. Any number in this table must be reproducible from LangSmith runs.
6. **Suggest next lever:** based on which metric moved least, suggest the next optimization (prompt trim, reranker, cache, model routing) — but do not implement it without going through the plan-then-confirm gate.
