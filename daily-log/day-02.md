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
