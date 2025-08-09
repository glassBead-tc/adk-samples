### ADK Agent Patterns and Recurring Use Cases

This report synthesizes recurring design patterns, architectural building blocks, and common use cases implemented across the ADK sample agents in this repository (Python and Java). It highlights how examples compose agents, tools, and sub-agents; how they manage state; and how they orchestrate workflows, loops, and evaluations.


## Executive summary
- **Agent composition is modular**: Most samples define a `root_agent` composed from sub-agents and tools, using `Agent`, `LlmAgent`, `SequentialAgent`, and `LoopAgent`.
- **Sub-agents are first-class tools**: Hierarchical orchestration commonly wraps sub-agents via `AgentTool` and low-level actions via `FunctionTool`.
- **Callbacks + shared state power orchestration**: `before_agent_callback`, `after_agent_callback`, and `before_model_callback` prime and persist `CallbackContext.state` for coordination.
- **Prompts and models are explicit**: Instructions are centralized in `prompt.py`; `MODEL` constants and per-agent `model` arguments are used consistently.
- **Evaluation, tests, deployment are standard**: Many agents include `eval/`, `tests/`, and `deployment/` to enable local validation and cloud deployment.
- **Common use cases repeat**: Research/reporting, software support, data/ML workflows, content and marketing, shopping/travel assistants, RAG document QA, and forecasting.


## Core building blocks

- **Root agent convention**
  - Root entry is named `root_agent` to work smoothly with ADK tooling.
  - Examples:
    - Travel Concierge: `root_agent` with sub-agents and a pre-run callback.
    - Software Bug Assistant: `root_agent` with a toolbelt (search, MCP, date, LangChain).

- **Agent types**
  - `Agent` / `LlmAgent`: Conversational or workflow root with tools and sub-agents.
  - `SequentialAgent`: Ordered pipelines of sub-agents; common in ML and image scoring.
  - `LoopAgent`: Repeats sequences until a termination signal; combined with a checker agent or event escalation.
  - Custom `BaseAgent`: Used in advanced cases (e.g., fullstack) for bespoke control.

- **Sub-agents as tools**
  - `AgentTool(agent=...)` exposes a sub-agent as a callable tool under a coordinator (e.g., marketing, finance).
  - Encourages clear role-based decomposition (e.g., data_analyst, trading_analyst, risk_analyst).

- **Function tools**
  - `FunctionTool(func=...)` wraps Python functions for deterministic actions (web search, click, memory, BigQuery operations).
  - Toolchains may include built-ins (e.g., `google_search`) and custom utilities.

- **Callbacks**
  - `before_agent_callback`: Initialize state or instructions (e.g., set schema, load itinerary, set timestamps/IDs).
  - `after_agent_callback`: Persist outputs or logs (e.g., write final state).
  - `before_model_callback`: Cross-cutting concerns (e.g., rate limiting).

- **State and events**
  - `CallbackContext.state` is a shared KV store for coordination across steps.
  - Event-driven control (e.g., `EventActions(escalate=True)`) stops loops when criteria are met.
  - `output_key` captures an agent’s product into session state for later steps.

- **Prompts and structure**
  - Per-agent instructions are centralized in `prompt.py` and imported by agents.
  - `global_instruction` sets shared system behavior for multi-stage workflows.
  - Structured outputs (e.g., Pydantic models) are used where strong schema is needed.

- **Planning**
  - Some agents enable planning via built-in planners with thinking configuration for traceability.


## Representative patterns by example

- **Travel Concierge** (`python/agents/travel-concierge`)
  - Pattern: `Agent` with multiple domain sub-agents (inspiration, planning, booking, pre-trip, in-trip, post-trip).
  - Uses `before_agent_callback` to pre-load itinerary context.
  - Demonstrates long-lived, phase-specific assistants under one coordinator.

- **Software Bug Assistant (Python + Java)**
  - Pattern: Single `Agent` with a rich toolbelt; integrates search, MCP toolbox, and date/time.
  - Emphasizes RAG-like queries across internal/external knowledge sources.

- **Personalized Shopping**
  - Pattern: `Agent` with `FunctionTool` wrappers for deterministic browser-like actions (`search`, `click`).
  - Tests emphasize tool determinism and simple eval scenarios.

- **Marketing Agency**
  - Pattern: `LlmAgent` as coordinator with `AgentTool`-wrapped specialists (domain, website, marketing, logo).
  - Clear orchestration across creative and technical subtasks.

- **Machine Learning Engineering (MLE-STAR)**
  - Pattern: `SequentialAgent` pipeline (initialization → refinement → ensemble → submission), embedded under a root `Agent` front door.
  - Uses `after_agent_callback` to persist pipeline state; `global_instruction` for consistent behavior.

- **LLM Auditor**
  - Pattern: `SequentialAgent` with `critic` then `reviser`, enforcing verify-then-refine loop.
  - Clear separation of concerns between evaluation and synthesis.

- **Image Scoring**
  - Pattern: `SequentialAgent` inside a `LoopAgent`; uses a checker to terminate based on a scoring threshold.
  - `before_agent_callback` seeds run metadata (timestamp, UUID) into state.

- **Gemini Fullstack**
  - Pattern: Multi-agent research system with planner, structured output models, stateful callbacks, and loop termination via a custom checker `BaseAgent`.
  - Implements source collection and citation replacement callbacks for report generation with links.

- **FOMC Research**
  - Pattern: `Agent` with sub-agents for data retrieval, research, and analysis; tool to store intermediate state; `before_model_callback` for rate limiting.

- **Financial Advisor**
  - Pattern: `LlmAgent` coordinator with sub-analyst `AgentTool`s; uses `output_key` for downstream consumption.

- **Data Science**
  - Pattern: Root `Agent` orchestrating BQ(NL2SQL) and DS tools; injects dynamic database schema into instructions via `before_agent_callback`.
  - Uses low-temperature `generate_content_config` for reproducibility.

- **RAG**
  - Pattern: Single-agent retrieval-augmented answering with citations using Vertex AI Retrieval/grounding.

- **Java samples** (Software Bug Assistant, Time Series Forecasting)
  - Mirror Python patterns: single-agent tool orchestration, BigQuery/BQML integration, standard project layout.


## Cross-cutting concerns and best practices

- **Root agent name**: Export the entry as `root_agent` for ADK CLI/Dev UI compatibility.
- **Clear agent roles**: Use meaningful `name` and `description` to aid traceability in logs and UIs.
- **Prompts**: Keep prompts modular in `prompt.py`; use `global_instruction` for global rules.
- **Model selection**: Prefer explicit `model=` configuration (often via env or a `MODEL` constant).
- **Stateful orchestration**: Centralize cross-step data in `CallbackContext.state`; avoid overloading tool parameters.
- **Callbacks**: Use `before_agent_callback` to seed state/instructions, `after_agent_callback` to persist, `before_model_callback` for cross-cutting policies.
- **Sub-agent wrapping**: Use `AgentTool` to hide complexity behind a coordinator; prefer `SequentialAgent` for fixed pipelines.
- **Loop termination**: Use `LoopAgent` plus a checker agent or explicit `EventActions(escalate=True)` to stop conditions-based loops.
- **Structured outputs**: Apply Pydantic schemas or `output_key` where downstream steps expect typed data.
- **Evaluation and tests**:
  - Include `eval/` with `.test.json` inputs and `test_eval.py` harnesses.
  - Unit test critical tools in `tests/` (e.g., browser actions, DB tools).
- **Deployment**: Provide `deployment/` scripts for Vertex AI Agent Engine or cloud environments.
- **Configuration**: Ship `.env.example` and `pyproject.toml`/build files; document required environment variables.
- **Temperature and determinism**: For pipelines and data tasks, lower temperatures improve reproducibility.


## Common use cases observed repeatedly

- **Research and reporting**: Planning, web search, source tracking, and citation-inserted Markdown reports.
- **Software support and diagnostics**: Search across tickets, repos, forums; summarize and propose fixes.
- **Data analytics and ML**: NL2SQL to query data; refinement loops for model building; pipeline orchestration with checkpoints.
- **Content generation and marketing**: Domains, sites, logos, strategies, and media assets orchestrated by a coordinator.
- **Personal shopping and travel**: Guided discovery, browsing actions, booking flows, and phase-specific assistance.
- **RAG document QA**: Answering with citations grounded on retrieved documents.
- **Forecasting**: BQML-backed time series forecasts with explainable outputs.
- **Compliance/policy checking**: Generate and score outputs/images against policy; loop until quality thresholds.


## Minimal code sketches (illustrative)

- Coordinator with sub-agents as tools:
```python
from google.adk.agents import LlmAgent
from google.adk.tools.agent_tool import AgentTool

root_agent = LlmAgent(
    name="coordinator",
    model="gemini-2.5-pro",
    instruction=COORDINATOR_PROMPT,
    tools=[
        AgentTool(agent=sub_agent_a),
        AgentTool(agent=sub_agent_b),
    ],
)
```

- Sequential pipeline with state persistence:
```python
from google.adk.agents import SequentialAgent, Agent

pipeline = SequentialAgent(
    name="pipeline",
    sub_agents=[stage1, stage2, stage3],
    after_agent_callback=save_state,
)

root_agent = Agent(
    model=MODEL,
    name="frontdoor",
    instruction=FRONTDOOR_PROMPT,
    sub_agents=[pipeline],
)
```

- Loop with termination via checker:
```python
from google.adk.agents import LoopAgent

root_agent = LoopAgent(
    name="iterative_runner",
    sub_agents=[sequential_process, checker_agent],
)
# checker_agent triggers stop when criteria met (e.g., EventActions(escalate=True))
```


## Directory and lifecycle conventions

- Directory structure: `agent.py`, `prompt.py`, optional `shared_libraries/`, `sub_agents/`, `tools/`, plus `tests/`, `eval/`, and `deployment/` as needed.
- Lifecycle: Configure env → install deps → run locally (CLI or Dev UI) → evaluate → test → deploy.


## Practical checklist for new agents
- Define `root_agent`, choose agent type(s) and composition pattern.
- Extract prompts to `prompt.py`; set `MODEL` and `global_instruction` if needed.
- Wrap specialists as `AgentTool`; wrap functions as `FunctionTool`.
- Design state schema; initialize via `before_agent_callback`.
- Add eval cases and unit tests for tools; consider low temperature for determinism.
- Provide `.env.example`, `pyproject.toml`/build files, and `deployment/` if applicable.


## Closing note
These patterns are intentionally composable. Start simple with a single `Agent` + a couple of `FunctionTool`s, then evolve to `SequentialAgent` pipelines and `LoopAgent` workflows as your use case matures. The samples here provide practical, working blueprints across research, analytics, content, commerce, and operations domains.