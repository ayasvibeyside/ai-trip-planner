# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A teaching reference implementation of a multi-agent AI system, skinned as a "trip planner." The travel domain is incidental — the point of the repo is to demonstrate three patterns that generalize to any agentic system: parallel multi-agent orchestration (LangGraph), RAG with graceful fallback, and API integration with graceful degradation (search APIs → LLM fallback). Expect the whole backend to live in one file (`backend/main.py`, ~860 lines) by design — it's meant to be read top to bottom.

## Commands

Run everything from the repo root unless noted.

```bash
# Install deps (uv preferred, pip fallback)
cd backend && uv pip install -r requirements.txt   # or: pip install -r requirements.txt

# Configure environment
cp backend/.env.example backend/.env   # then set OPENAI_API_KEY or OPENROUTER_API_KEY

# Run the app (backend on :8000, serves the minimal UI at '/')
./start.sh
# or, for autoreload during development:
cd backend && uvicorn main:app --host 0.0.0.0 --port 8000 --reload

# Synthetic eval / adversarial tool-call generator
python "test scripts/synthetic_data_gen.py" --base-url http://localhost:8000 --count 12

# Smoke test a request directly
curl -X POST http://localhost:8000/plan-trip \
  -H "Content-Type: application/json" \
  -d '{"destination":"Tokyo, Japan","duration":"7 days","budget":"$2000","interests":"food, culture"}'
```

There is no lint/build/test-suite tooling configured (no pytest, no linter config) — validate changes by running the server and hitting `/plan-trip` and `/health`.

Note: the README references `test scripts/test_api.py`, but that file does not currently exist in the repo — don't assume it's there.

## Architecture

### Request flow (`backend/main.py`)

`POST /plan-trip` builds a `TripState` dict and invokes a LangGraph graph (`build_graph()`, compiled fresh per request — no checkpointer):

1. **Parallel fan-out from START**: `research_agent`, `budget_agent`, `local_agent` all run concurrently, each bound to its own small toolset (see below) via `llm.bind_tools(...)`.
2. Each agent follows the same two-step shape: (a) invoke the LLM with tools bound and a system prompt built from an f-string-like template + `using_prompt_template(...)` for tracing, (b) if the LLM requested tool calls, execute them via `ToolNode`, append results to the message list, then make a second untooled LLM call to synthesize a summary. Tool calls made are collected into `state["tool_calls"]` (merged via `operator.add` in `TripState`).
3. **Fan-in**: all three feed into `itinerary_agent`, which truncates each upstream summary to 400 chars, builds a final prompt, and produces `state["final"]` — the itinerary returned to the client.
4. Response is `TripResponse{result, tool_calls}` — `tool_calls` is exposed specifically so traces/evals can inspect which tools each agent invoked and with what args.

When adding a 5th agent or changing the flow, edit `TripState`, add a node function following the existing four agents' shape, and wire it into `build_graph()`.

### Tools and graceful degradation

Every `@tool`-decorated function (in `backend/main.py`) follows the same fallback chain: try `_search_api()` (Tavily first, then SerpAPI, both optional and key-gated) → if no key configured or the call fails, fall back to `_llm_fallback()` which asks the LLM directly and compacts the response to ~200 chars via `_compact()`. Tools never hard-fail from a missing integration. When adding a new tool, follow this same `_search_api → _llm_fallback` pattern rather than calling an external API directly.

Agents and their bound tools:
- `research_agent`: `essential_info`, `weather_brief`, `visa_brief`
- `budget_agent`: `budget_basics`, `attraction_prices`
- `local_agent`: `local_flavor`, `local_customs`, `hidden_gems`

A few tools (`day_plan`, `travel_time`, `packing_list`) are defined but not currently bound to any agent — they exist as extension points for the "add a new tool" learning exercise.

### RAG (opt-in, off by default)

Controlled by `ENABLE_RAG` env var. When on, `LocalGuideRetriever` (constructed once at module load as `GUIDE_RETRIEVER`) loads `backend/data/local_guides.json`, embeds it with `OpenAIEmbeddings` into an `InMemoryVectorStore`, and `local_agent` injects the top-3 retrieved guides into its prompt as curated context before calling the LLM. If embeddings fail to initialize, or the vector store returns nothing, it degrades to `_keyword_fallback()` (simple city/interest substring scoring) — never hard-fails. See `RAG.md` for the full walkthrough.

### LLM initialization

`_init_llm()` (called once at import time, result cached in module-level `llm`) picks a backend in this priority: `TEST_MODE` env var → an in-process fake LLM (no API calls, used for tests) → `OPENAI_API_KEY` → `ChatOpenAI(gpt-3.5-turbo)` → `OPENROUTER_API_KEY` → `ChatOpenAI` pointed at OpenRouter's OpenAI-compatible endpoint (model configurable via `OPENROUTER_MODEL`) → else raises. Exactly one of `OPENAI_API_KEY`/`OPENROUTER_API_KEY` needs to be set.

### Observability (optional)

If `arize-otel`/`openinference` imports succeed and `ARIZE_SPACE_ID`+`ARIZE_API_KEY` are set, tracing is registered once at module load and `LangChainInstrumentor`/`LiteLLMInstrumentor` are attached. If the imports fail, `_TRACING = False` and `using_prompt_template`/`using_metadata`/`using_attributes` become local no-op context managers — so tracing code can be left in place unconditionally throughout the agent functions without needing to guard every call site. `session_id`, `user_id`, and `turn_index` on `TripRequest` are threaded into span attributes in `plan_trip` for multi-turn session tracking in Arize.

### Frontend

`frontend/index.html` is a single static file with no build step, served directly by FastAPI at `GET /`. There is no separate frontend server/toolchain.

### Deployment

`render.yaml` deploys `backend/` as a Render web service (installs via `uv` with pip fallback, runs `uvicorn main:app`). Required env vars are set in the Render dashboard, not committed.

## Extending this codebase

The README frames three "paths" for modification — useful context when a request looks like one of these:
- **New tool**: add a `@tool`-decorated function following the `_search_api → _llm_fallback` pattern, bind it to the relevant agent's tool list.
- **New agent**: add a node function shaped like the existing four, add fields to `TripState` as needed, wire edges in `build_graph()`.
- **New domain entirely** (e.g. PR-description generator, support-ticket triage): the intended approach is to keep the graph/tool/RAG/fallback *shape* and swap out `TripState` fields, prompts, tools, and `backend/data/*.json` for the new domain.
