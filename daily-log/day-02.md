# Day 02

## What I did
- Walked through Chapter 4 (agent classic paradigms) module by module with Claude Code.
- Configured the LLM by reusing my Claude Code setup: LiteLLM proxy as an
  OpenAI-compatible endpoint, with `anthropic.claude-haiku-4-5` to save tokens.
  - `.env` mapping: `LLM_BASE_URL` = proxy `/v1`, `LLM_API_KEY` = auth token,
    `LLM_MODEL_ID` = `anthropic.claude-haiku-4-5`.
  - Verified `.env` is git-ignored (root `.gitignore:123`) before adding the key.
- Set up a venv (Python 3.14) and installed `openai`, `python-dotenv`,
  `google-search-results`.
- Ran and tested the three paradigms:
  - `Reflection.py` — generate → critique → refine loop (LLM only).
  - `Plan_and_solve.py` — Planner decomposes into steps, Executor runs them.
  - `ReAct.py` — Thought/Action/Observation loop with the Search tool.
- Modified the demo questions to probe the logic on my own prompts:
  - `Plan_and_solve.py`: swapped the apple word-problem for a Snake-game HTML
    generation task (and a simple two-number system) to see how planning handles
    open-ended / non-math tasks.
  - `ReAct.py`: swapped the Huawei-phone question for "best commercial vs
    open-source LLM in Jun 2026" to watch multi-step search + reasoning.

## Learning focus
- An agent = a control loop around a single LLM call; paradigms differ by loop
  shape (reactive vs plan-first vs self-critique), not by the model.
- Agent↔tool contract is plain-text conventions parsed by regex
  (`Action: Tool[input]`), not native function-calling — intentionally transparent.
- `HelloAgentsLLM` is just the OpenAI SDK pointed at a `base_url`, so any
  OpenAI-compatible endpoint (incl. my proxy) drops in.

## Notes / observations
- Haiku keeps cost low, but ReAct loops up to `max_steps=5` and re-sends the
  growing history each step, so one run = several calls.

## Next step
- Move to Chapter 7: see the same paradigms repackaged as the `hello_agents`
  framework (SimpleAgent / ReActAgent base classes, ToolRegistry).

---

## Chapter 5 review — Low-code platforms (concept-only, no hands-on)

### Purpose
- Chapter 5 is the "other school" of agent building: **flow-driven / low-code**,
  where the LLM is one node in a drag-and-drop workflow graph — as opposed to the
  **AI-native / code** approach in the rest of the book, where the LLM *is* the
  reasoning engine.
- Real value for me is NOT mastering any single platform (the products age fast),
  but learning the transferable **mental model**: nodes, workflow graphs, RAG
  pipelines, triggers, tool/HTTP nodes, agent-behind-an-API.

### The four platforms — pros / cons / production relevance
| Platform | Best at | Hosted / Self-host | Cons | Verdict |
|----------|---------|--------------------|------|---------|
| **n8n** | General workflow automation + AI nodes | Self-host (OSS) | Not LLM-first; agent features bolted on | ⭐ Most durable & widely used |
| **Dify** | LLMOps: RAG + agents + API serving | Self-host (OSS) | Heavier to run; opinionated | ⭐ Genuinely used in production |
| **Coze** | Fast bot prototyping | Hosted (ByteDance) | Vendor lock-in; less used in serious eng orgs | Skim only |
| **FastGPT** | RAG knowledge-base Q&A | Self-host (OSS) | Narrow (KB chatbots) | Read concepts only |

### Durable lessons (the parts that don't go stale)
- **Where low-code breaks down**: great to ~80% of a solution; the last 20%
  (custom logic, state, versioning, testing, latency) is where teams eject to
  code. Knowing that line = senior judgment.
- **RAG is the through-line**: every platform bolts on a knowledge base; same
  concepts I'll build by hand in **Ch8 (Memory & RAG)**.
- **MCP is eating the "tool node"**: in 2026 tool access is standardizing on
  **MCP** (**Ch10**). Mentally map every "tool/HTTP node" → "what MCP standardizes".
- **Hosted vs self-hosted** = a real production decision (vendor lock-in, data
  residency), esp. for regulated data.

### Low-code concept → later code chapter (why this chapter is a preview map)
- RAG knowledge base (FastGPT/Dify) → **Ch8** (Memory & RAG)
- Tool / HTTP nodes → **Ch10** (MCP)
- Workflow orchestration → **Ch6** (LangGraph, in code)

### Decision
- Skipped hands-on for all four (products age fast; doc + screenshots suffice).
- If revisiting later, only two are worth building: **n8n** (workflow/graph model
  clicks fastest) and **Dify** (build an app, then call it via its HTTP API — the
  "agent-as-a-service" pattern is the real production takeaway).
