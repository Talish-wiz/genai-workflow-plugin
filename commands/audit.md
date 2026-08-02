---
description: Audit the codebase for violations of the architecture skill — inline prompts, stray print(), guardrails or LangSmith clients instantiated outside their homes, fat routes, missing traces. Usage: /audit [optional: folder to focus on]
---

# /audit — Architecture compliance check

Scope (optional): **$ARGUMENTS**

Read `skills/architecture/SKILL.md` from this plugin, then use the `architecture-auditor` agent (or do it inline if agents are unavailable) to check the codebase for:

1. **Inline prompts** — multi-line prompt strings outside `/prompts/` (NeMo's `/guardrails/nemo/` content is exempt).
2. **Bare `print()`** in committed code instead of `logging.getLogger(__name__)`.
3. **Stray instantiation** — `LLMRails`/`RailsConfig` created anywhere other than `/guardrails/runner.py`; LangSmith `Client` created anywhere other than `/observability/langsmith_client.py`; logging handlers configured outside `/config/logging_config.py`.
4. **Fat routes** — business logic, prompt construction, or direct LLM calls inside `/api/`.
5. **Missing observability** — agent nodes/tools that won't appear as their own span (no auto-trace and no `@traceable`); traces missing session/user metadata.
6. **Guardrail bypass** — any path from `/api/` to an agent that doesn't pass through `/guardrails/runner.py`; rail outcomes not recorded on the LangSmith trace.
7. **Schema duplication** — the same data shape defined in more than one place instead of `/schemas/`.
8. **Forbidden dependencies** — imports of Langfuse, LLM Guard, or any alternative observability/guardrails library.
9. **Unbacked claims** — metrics referenced in docs/deck files that have no corresponding harness output.

Report as a table: file → rule violated → suggested fix. Then ask which fixes to apply (plan-then-confirm applies to multi-file fixes).
