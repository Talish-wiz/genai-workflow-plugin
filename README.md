# genai-workflow — Claude Code plugin

A Claude Code plugin that turns Claude Code into a **disciplined GenAI build partner** for the "improve GenAI workflow effectiveness" hackathon problem statement. It locks in an opinionated architecture and stack (**LangSmith** for observability/evals, **NVIDIA NeMo Guardrails** for guardrails, LangChain/LangGraph/deepagents chosen per component), forces a plan-before-code habit, and gives you commands that turn a one-line business idea into a scaffolded, benchmarked, audited project.

---

## What's in the box

```
genai-workflow-plugin/
├── .claude-plugin/plugin.json      # plugin manifest (name, description, keywords)
├── skills/architecture/SKILL.md    # the rulebook — Claude auto-reads this before writing code
├── commands/
│   ├── usecase.md                  # /usecase <scenario> — intake → plan → confirm → scaffold
│   ├── baseline.md                 # /baseline — run the before/after benchmark, emit deck table
│   └── audit.md                    # /audit — scan the repo for architecture-rule violations
├── agents/architecture-auditor.md  # read-only subagent that does the actual audit scanning
├── hooks/hooks.json                # SessionStart hook — reminds Claude the gate is active
└── README.md                       # this file
```

### `skills/architecture/SKILL.md` — the rulebook
This is a Claude Code **skill**: Claude reads it automatically whenever the task looks like "write code," "add a feature," or "here's a new domain use case." It defines:
- A **plan-then-confirm gate** — before touching any non-trivial code, Claude must post a short plan (files touched, agent framework choice, observability/guardrails involved, which metric the change is meant to move) and wait for your yes.
- A **fixed directory layout** (`/prompts/`, `/agents/`, `/tools/`, `/graph/`, `/rag/`, `/observability/`, `/guardrails/`, `/api/`, `/config/`, `/schemas/`, `/tests/`) so nothing gets dumped in one giant file.
- The **non-negotiable stack**: LangSmith owns all tracing/evals, NVIDIA NeMo Guardrails owns all input/output/topical/groundedness rails, Python `logging` owns app logs, and agent code is LangChain (simple chains) / LangGraph (stateful multi-agent) / deepagents (planner-executor autonomy) — chosen per component, not mixed arbitrarily.
- A **before/after contract**: every optimization must map to a hackathon evaluation-lens bucket (prompt efficiency, retrieval quality, reliability, cost, business impact, governance, UX), and no metric may appear in a deck unless it came out of the benchmark harness.

### `commands/usecase.md` — `/usecase <describe the business scenario>`
The entry point for a new domain idea (e.g. *"insurance claims triage assistant"*). Claude:
1. Produces a full intake plan — business framing, baseline definition, per-component framework choice, optimization levers, metric plan, governance plan, file plan, and a mapping to the four evaluation lenses.
2. **Stops and asks for your confirmation** — nothing is created yet.
3. Once you approve, scaffolds the real project: config, a `/guardrails/nemo/` rails config with a starter escalation/fallback flow, `/observability/` (LangSmith client, tracing, evaluators, dataset + benchmark stubs), a baseline pipeline first, then the optimized one.
4. Reports what's built vs. stubbed and what to run next.

### `commands/baseline.md` — `/baseline`
Runs the identical LangSmith dataset through the baseline and optimized pipelines and produces the before/after comparison table (token cost, latency p50/p95, accuracy, hallucination rate, retrieval precision) — the numbers you actually put in the hackathon deck. Refuses to invent numbers that didn't come from a real run.

### `commands/audit.md` + `agents/architecture-auditor.md` — `/audit`
A read-only pass over the repo that flags: prompts written inline instead of in `/prompts/`, stray `print()` instead of `logging`, a second `LLMRails`/LangSmith `Client` instantiated somewhere it shouldn't be, business logic leaking into `/api/` routes, forbidden imports (Langfuse, LLM Guard), and metrics claimed in docs with no benchmark backing them.

### `hooks/hooks.json`
A `SessionStart` hook that prints a short reminder at the start of every Claude Code session in this project, so the workflow gate and stack rules stay top of mind even in a fresh session.

## How it helps you build a GenAI use case, end to end

1. You describe the business scenario in one line via `/usecase`.
2. Claude turns that into an explicit plan instead of guessing — including what the "before" (unoptimized) version looks like, so you have an honest baseline to improve on.
3. You approve, and get a real scaffold: prompts, agents, graph wiring, RAG layer, guardrails config, and an observability layer that's wired for LangSmith from the first commit.
4. As you keep building, the skill keeps every new file in its correct folder and keeps forcing the plan-first habit, so the codebase stays demoable instead of turning into one big script.
5. `/baseline` gives you the quantified "we cut token cost by X%, hallucination rate by Y%" story the hackathon rubric is scored on — backed by actual LangSmith traces, not eyeballed guesses.
6. `/audit` catches drift before your demo — inline prompts, bypassed guardrails, or a stray Langfuse import that snuck back in.

---

## Install locally

### Option A — as a proper Claude Code plugin (recommended)

1. Clone or unzip this repo anywhere, e.g.:
   ```bash
   mkdir -p ~/claude-plugins
   git clone <this-repo-url> ~/claude-plugins/genai-workflow-plugin
   ```
2. Add that folder as a local plugin marketplace and install from it:
   ```bash
   # inside Claude Code
   /plugin marketplace add ~/claude-plugins
   /plugin install genai-workflow
   ```
3. If Claude Code doesn't pick up a bare folder as a marketplace, add a minimal `marketplace.json` next to the plugin folder:
   ```json
   // ~/claude-plugins/.claude-plugin/marketplace.json
   {
     "name": "local-plugins",
     "plugins": [
       { "name": "genai-workflow", "source": "./genai-workflow-plugin" }
     ]
   }
   ```
   then re-run `/plugin marketplace add ~/claude-plugins`.
4. Restart Claude Code / start a new session in your project — the `SessionStart` hook message confirms it's active.

### Option B — skill only (fastest, no plugin machinery)

If you just want the rulebook without slash commands or the hook:
```bash
mkdir -p <your-project>/.claude/skills
cp -r skills/architecture <your-project>/.claude/skills/architecture
```
Claude Code will pick up the skill automatically on the next session in that project. You lose `/usecase`, `/baseline`, `/audit`, and the auditor subagent this way.

### Verify it's working
Open Claude Code in your project and ask something like *"add a new retriever tool"* — Claude should first mention consulting the architecture rules and propose a plan before writing anything. Or just run `/usecase test scenario` if you installed the full plugin.

---

## Stack contract (non-negotiable)

| Concern | Tool | Where it lives |
|---|---|---|
| Observability & evals | **LangSmith** | `/observability/` |
| Guardrails | **NVIDIA NeMo Guardrails** | `/guardrails/nemo/` + `/guardrails/runner.py` |
| App logging | Python `logging` | configured once in `/config/logging_config.py` |
| Agent orchestration | LangChain / LangGraph / deepagents (per component) | `/agents/`, `/graph/` |
| Retrieval | your choice of vector store | `/rag/` |

Forbidden: Langfuse, LLM Guard, bare `print()` logging, prompts inline in code, business logic in `/api/` routes.
