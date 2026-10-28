# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is **crewAI**, the open-source Python framework for orchestrating role-playing, autonomous multi-agent
systems (`crewAIInc/crewAI`). It is standalone — built from scratch, independent of LangChain or other agent
frameworks. It ships both as a library (`import crewai`) and as a CLI (`crewai ...`, entry point
`crewai.cli.cli:crewai`) for scaffolding and running crew/flow projects.

Two complementary core concepts:
- **Crews** — teams of autonomous `Agent`s executing `Task`s collaboratively (sequential or hierarchical process),
  optimized for autonomy and role-based collaboration.
- **Flows** — event-driven, production-grade orchestration (`@start`, `@listen`, `@router`) for precise control
  over execution paths; Flows can natively embed Crews.

Requires Python `>=3.10,<3.14`. Dependency/build management is via `uv` (hatchling backend, dynamic versioning
from `src/crewai/__init__.py`).

## Commands

```bash
# Install dependencies (locks + syncs from pyproject.toml/uv.lock)
uv lock
uv sync

# Create/activate the virtual env
uv venv

# Install git hooks (ruff + ruff-format on commit, per .pre-commit-config.yaml)
pre-commit install

# Run the full test suite
uv run pytest .

# Run a single test file / test
uv run pytest tests/agent_test.py
uv run pytest tests/agent_test.py::test_agent_execution

# Lint (ruff; CI only lints files changed vs. target branch, excludes src/crewai/cli/templates/)
ruff check .
ruff check --fix .

# Static type checks
uvx mypy src

# Build a distributable package
uv build
```

CI runs tests across Python 3.10–3.13 with `uv run pytest --block-network --timeout=60`, i.e. tests must not hit
the network — LLM calls are pre-recorded and replayed via VCR cassettes in `tests/cassettes/` (see
`tests/conftest.py`'s `vcr_config` fixture and the `OPENAI_API_KEY=fake-api-key` env used in CI). The `conftest.py`
`setup_test_environment` autouse fixture redirects `CREWAI_STORAGE_DIR` to a temp dir for every test. Security
scanning uses `bandit -c pyproject.toml -r src/ -ll` (excludes `src/crewai/cli/templates`).

## Architecture

### Package layout (`src/crewai/`)

| Area | Path | Role |
|------|------|------|
| Core objects | `agent.py`, `crew.py`, `task.py`, `lite_agent.py`, `llm.py`, `process.py` | Top-level classes re-exported from `crewai/__init__.py` |
| Agent internals | `agents/` | `agent_builder/base_agent.py` (abstract base all agent types extend), `crew_agent_executor.py` (the ReAct-style execution loop), `cache/`, `agent_adapters/` (LangGraph, OpenAI Agents SDK adapters) |
| Task variants | `tasks/` | `conditional_task.py`, `task_output.py`, `llm_guardrail.py`, guardrail results |
| Crew helpers | `crews/` | `crew_output.py` |
| Flows | `flow/` | `flow.py` (the `Flow` class + `@start`/`@listen`/`@router`/`@and_`/`@or_` decorators, `FlowState`), `persistence/` (flow state persistence backends), `flow_visualizer.py`/`plot_flow.py` (flow diagrams) |
| Project scaffolding decorators | `project/` | `crew_base.py` (`@CrewBase` class decorator), `annotations.py` (`@agent`, `@task`, `@crew`, `@llm`, `@tool`, `@before_kickoff`, `@after_kickoff`, `@output_json`, `@output_pydantic`, `@cache_handler`, `@callback` — used in the YAML-driven crew project structure created by `crewai create crew`) |
| Tools | `tools/` | `base_tool.py` (`BaseTool`/`Tool`), `structured_tool.py`, `tool_usage.py`, `agent_tools/` (agent-to-agent delegation tools), `cache_tools/` |
| Memory | `memory/` | `short_term/`, `long_term/`, `entity/`, `external/`, `user/`, `contextual/` (assembles the different memory types into executor context), `storage/` (backing stores, e.g. SQLite/Chroma-based) |
| Knowledge | `knowledge/` | `knowledge.py` (RAG-style knowledge base attachable to agents/crews), `source/` (file/string/URL loaders), `embedder/`, `storage/` |
| Events | `utilities/events/` | Pub/sub event bus (`crewai_event_bus.py`) with typed events per domain: `agent_events.py`, `crew_events.py`, `task_events.py`, `flow_events.py`, `llm_events.py`, `llm_guardrail_events.py`, `knowledge_events.py`, `tool_usage_events.py`, `reasoning_events.py`; `event_listener.py` is the default listener (console/telemetry output) |
| Security | `security/` | `Fingerprint`, `SecurityConfig` — identity/fingerprinting attached to agents and crews |
| Telemetry | `telemetry/` | Anonymous, opt-out (`OTEL_SDK_DISABLED=true`) usage telemetry via OpenTelemetry — never sends prompts, task content, or secrets unless `share_crew` is explicitly enabled |
| CLI | `cli/` | `cli.py` is the click `crewai` command group; see below |
| i18n | `translations/` | Prompt/message localization |

### Execution flow (Crew)

`Crew(agents=[...], tasks=[...], process=Process.sequential|hierarchical)` → `crew.kickoff(inputs=...)`:
- `Process.sequential`: tasks run in order, each `Agent` (via `CrewAgentExecutor`) reasons over its `role`/`goal`/
  `backstory`, invokes `tools`, consults `memory`/`knowledge`, and produces a `TaskOutput`.
- `Process.hierarchical`: a manager agent is auto-assigned to plan, delegate, and validate task execution across
  the crew.
- Agents can delegate to one another via `AgentTools` (`tools/agent_tools/`).
- Every stage emits events on `crewai_event_bus` (`CrewKickoffStartedEvent`, `AgentExecutionStartedEvent`,
  `MethodExecutionFinishedEvent` for flows, etc.) that `EventListener` and telemetry consume.

### Project-generated structure (`crewai create crew <name>`)

Scaffolds (templates live in `src/crewai/cli/templates/`, excluded from lint/mypy/bandit):
```
my_project/src/my_project/
├── main.py            # entry point, builds inputs, calls crew().kickoff()
├── crew.py            # @CrewBase class; methods decorated @agent/@task/@crew build the Crew
├── tools/custom_tool.py
└── config/
    ├── agents.yaml     # role/goal/backstory per agent, referenced via self.agents_config[...]
    └── tasks.yaml      # description/expected_output/agent per task, via self.tasks_config[...]
```
`@CrewBase` + the `agents_config`/`tasks_config` YAML pattern is the idiomatic way crewAI projects are authored;
`crew.py`'s `@agent`/`@task` methods are memoized (`project/utils.py`) so they're only constructed once per
instance.

### CLI surface (`crewai <cmd>`, defined in `cli/cli.py`)

`create`, `run`, `train`, `replay`, `test`, `reset-memories`, `install`, `update`, `signup`/`login`, `chat`,
`log-tasks-outputs`, plus subgroups `tool` (`create`/`install`/`publish`), `deploy` (`create`/`push`/`status`/
`logs`/`remove`/`list`), and `flow` (`kickoff`/`plot`/`add-crew`) for CrewAI Enterprise / Crew Control Plane
integration (`plus_api.py`, `authentication/`, `organization/`).

## Conventions

- `.editorconfig`: LF line endings, UTF-8, trailing newline; Python files use **2-space indentation**.
- Ruff is the linter/formatter (`.ruff.toml` excludes `templates` and `__init__.py`); mypy runs with
  `ignore_missing_imports = true` and excludes `cli/templates`.
- Pydantic v2 (`BaseModel`, `Field`, `model_validator`, etc.) is used pervasively for core objects — `Agent`,
  `Crew`, `Task`, `Flow` states are all Pydantic models.
- New tests generally need a VCR cassette under `tests/cassettes/` if they exercise an LLM call, since CI runs
  with `--block-network`.
