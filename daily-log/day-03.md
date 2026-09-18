# Day 03

## What I did
- Worked through Chapter 6 (framework development), focused on **LangGraph**
  (`code/chapter6/Langgraph/Dialogue_System.py`) — a search assistant modeled as
  a state-machine graph: understand → search → answer.
- Set up an isolated venv `.venv_chp6` (Python 3.14) with `langgraph`,
  `langchain-openai`, `python-dotenv`, `tavily-python`.
- Reused my LLM `.env` (LiteLLM proxy + `anthropic.claude-haiku-4-5`) and added a
  `TAVILY_API_KEY` for real web search.
- Debugged three real issues to get it running (see below).
- Improved the CLI UX: exit hint on every prompt + graceful Ctrl+C / Ctrl+D exit.

## LangGraph — core concepts
- **State** = a shared `TypedDict` passed between nodes. Field
  `messages: Annotated[list, add_messages]` uses a **reducer** so message updates
  *append* instead of overwrite; plain fields get replaced.
- **Nodes** = functions `state -> partial state update` (understand / search / answer).
- **Edges** = wiring; `add_edge(START, ...)` ... `add_edge(..., END)`. This demo is
  linear; the framework's real power is **conditional edges** (router function →
  loops/branching) — the declarative version of ch4's hand-written ReAct loop.
- **Checkpointer** (`InMemorySaver` + `thread_id`) = built-in per-thread
  persistence/memory; lets a graph pause and resume.

## Debugging log (the valuable part)
1. **Wrong venv / not activated** — installs and runs missed the venv (`python`
   not even found). Fix: use `.venv_chp6/bin/python` or `source .../activate` in the
   *same* shell before installing AND running.
2. **`.env` not found → `model=None` crash** — `load_dotenv()` searches the current
   working dir, not the script's dir. Running from repo root found no `.env`, so
   `LLM_MODEL_ID` was None and `ChatOpenAI(model=None)` failed pydantic validation.
   Fix: `load_dotenv(Path(__file__).with_name(".env"))` so it's CWD-independent.
   (Removing the `gpt-4o-mini` default is what surfaced the bug instead of hiding it.)
3. **Bedrock rejects system-only messages** — nodes called
   `llm.invoke([SystemMessage(...)])`. My proxy routes Claude via AWS Bedrock, which
   requires ≥1 **user** message → `400 bedrock requires at least one non-system
   message`. Fix: send the task prompts as `HumanMessage` (all 3 call sites).

## UX improvement — reliable exit
- Problem: startup exit hint scrolled away; after searching there was no visible way
  to quit, and Ctrl+C/Ctrl+D crashed with a traceback.
- Fix: (a) exit hint now in the prompt every turn
  (`🤔 您想了解什么 (输入 quit / exit / 退出 结束):`); (b) wrapped `input()` in
  `try/except (KeyboardInterrupt, EOFError)` for a clean goodbye.
- Verified all 5 exit paths: `quit`, `exit`, `退出`, Ctrl+D (EOF), Ctrl+C (SIGINT,
  exit code 0). All exit cleanly with no traceback.

## Learning focus
- LangGraph = agents as a directed **state machine**: nodes (compute) + edges
  (control flow) + shared state. Loops via conditional edges are the headline feature.
- **"OpenAI-compatible" ≠ "OpenAI-identical."** LiteLLM smooths most differences, but
  provider rules still leak (Bedrock's user-message requirement). Message *roles* are
  a common breakage point when porting example code across providers.
- Env loading should be anchored to the script (`__file__`), not the CWD.

## Next step
- Explore LangGraph conditional edges: add a router so an empty search result loops
  back to re-query — to actually feel the graph's looping capability.
