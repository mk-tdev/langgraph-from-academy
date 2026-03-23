# Agentic Chat Platform (FastAPI + Next.js, Open-Source LangGraph Only)

Build a monorepo local web app with a FastAPI + LangGraph backend and a Next.js chat frontend to experiment with dynamic agentic workflows (interruptions, tools, subgraphs, parallelization) including PDF/DOCX upload and provider-switchable LLM backends, while explicitly avoiding LangSmith, LangGraph Studio, and `langgraph_sdk`.

## Scope + constraints

- **No LangSmith**: no tracing/project setup, no `LANGSMITH_*` env vars required.
- **No Studio / langgraph CLI dev server**: we will not use `langgraph dev` or any `module-*/studio` graphs.
- **No `langgraph_sdk`**: no remote graph, no SDK client.
- **Backend**: FastAPI service hosting LangGraph runs.
- **Frontend**: Next.js App Router chat UI.
- **Persistence**: SQLite on disk for checkpointing threads.
- **Streaming**: SSE streaming of tokens and (optionally) LangGraph node/tool events (toggle).
- **Provider switching**: per-request `provider` value selects LLM (no secrets stored in DB).

## What we’ll reuse from each module (concept extraction)

### Module 0 (basics)

- Environment setup patterns and minimal LangGraph usage.

### Module 1 (graphs, routing, tools, agents)

- **StateGraph + `MessagesState`** basics.
- **Tool routing** via `ToolNode` + `tools_condition`.
- **ReAct loop** pattern: `assistant -> tools -> assistant`.

### Module 2 (state schemas + reducers)

- Use explicit state schema (`TypedDict`/Pydantic) for non-message state.
- Use **reducers** (e.g., `operator.add`) for keys that accumulate items (e.g., extracted requirements, drafted bullets, critiques).

### Module 3 (persistence, breakpoints, human-in-the-loop, time travel)

- Use a **checkpointer** for persistence.
- Use **interruptions** (`interrupt_before=[...]`) for human review steps.
- Use `graph.get_state(...)` and `graph.update_state(...)` to implement a “review/edit then continue” workflow.
- Optional: **dynamic breakpoints** with `NodeInterrupt` for guardrails (e.g., missing inputs, too little resume detail).

### Module 4 (subgraphs, parallelization, map-reduce)

- Use **subgraphs** for focused tasks (e.g., requirement extraction, bullet rewriting, cover-letter drafting).
- Use **parallelization** for rewriting multiple resume bullets simultaneously.
- Use **map-reduce** style:
  - map: rewrite each selected bullet
  - reduce: assemble final resume section + overall consistency pass

### Module 5 (memory patterns)

- Use persistence to maintain “session” state (thread_id), plus optional lightweight profile preferences (tone, target seniority, constraints).
- Avoid any LangSmith “memory store” flows that require hosted infrastructure; keep local.

### Module 6 (deployment/connecting)

- We will **not** use the SDK connection patterns.
- We will reuse the “productization mindset”: clean entrypoints, config, and a runnable app.

## Target behavior (platform goals)

### Inputs

- Chat messages
- Optional: uploaded files (PDF/DOCX) that become part of session context
- Optional: “agent mode” selection (e.g., job-tailoring, research, support triage)
- Optional: provider selection (OpenAI / Anthropic / Gemini / Ollama local / Ollama remote)

### Outputs

- Assistant responses (streaming)
- Optional: structured artifacts saved to state (summaries, extracted requirements, plans)
- Optional: tool traces/events (for learning/debugging, not LangSmith)

### Human-in-the-loop checkpoints (core)

- Use `interrupt_before=[...]` to pause at review nodes.
- Frontend shows a “Review / Edit state” panel.
- Backend continues via `update_state(...)` + “resume execution”.

### Progress + approvals (learning-focused UX)

- Stream structured status to the UI: current plan, active step, tool calls, intermediate artifacts.
- Require explicit user approval before:
  - executing a generated runtime plan
  - running tools that read files, index documents, or execute code
  - finalizing outputs (optional)

## Proposed architecture

### Monorepo layout

- `backend/`
  - FastAPI app
  - LangGraph graphs (dynamic routing + subgraphs)
  - SQLite checkpointing
  - Upload parsing (PDF/DOCX -> text)
- `frontend/`
  - Next.js chat UI
  - SSE streaming client
  - File upload UI

### Backend API surface (high-level)

- `POST /v1/chat` (non-streaming)
- `GET /v1/chat/stream` (SSE token stream)
- `GET /v1/chat/events` (SSE graph events stream; optional toggle)
- `POST /v1/uploads` (multipart upload)
- `GET /v1/threads/{thread_id}` (fetch current state snapshot)
- `POST /v1/threads/{thread_id}/state` (apply human edits to state)
- `POST /v1/threads/{thread_id}/resume` (continue graph execution from checkpoint)

Additional (approval + progress):

- `POST /v1/threads/{thread_id}/approve` (record an approval decision)
- `GET /v1/threads/{thread_id}/events` (SSE structured events stream for progress UI)

### State schema (high-level)

- `messages: list[AnyMessage]` (use `MessagesState` / `add_messages` reducer)
- `files: list[UploadedFile]` (reducer: list add)
- `file_text: dict[file_id, str]` (extracted content)
- `artifacts: dict[str, Any]` (structured outputs from agent modes)
- `ui: dict` (e.g., last interrupt reason, forms to show)

Additional (dynamic planning + approvals):

- `plan: list[PlanStep]` (runtime-generated, reducer: overwrite)
- `active_step_id: str | None`
- `pending_approval: ApprovalRequest | None`
- `agent_working_set: dict` (lightweight scratchpad for supervisor routing)

### Graph shape (dynamic, not fixed)

- **Entry**: `router` node decides which “mode subgraph” to run based on state (messages + uploaded context + user-selected mode).
- **Modes implemented as subgraphs** (incrementally):
  - `job_tailoring_subgraph`
  - `research_subgraph`
  - `qa_over_uploads_subgraph`
- **Runtime plan-execution (dynamic)**:
  - Supervisor generates a structured plan (steps with type, inputs, assigned agent, tool needs, expected artifacts).
  - Graph interrupts to request approval.
  - If approved, graph executes steps, possibly in parallel, updating `active_step_id` and streaming progress events.
  - Steps can emit follow-up approval requests (e.g., “Run code tool?”, “Index documents?”).
- **Dynamic breakpoints**:
  - Guardrails can raise interruptions (e.g., missing resume, file parsing failure, insufficient detail) and push UI actions.
- **Parallelization**:
  - Map-reduce style steps inside modes (e.g., rewrite multiple bullets; extract multiple sections from documents).

### Tools (minimal, local)

- `parse_pdf(file_path) -> str`
- `parse_docx(file_path) -> str`
- Optional deterministic helpers (scoring, keyword extraction) to demonstrate tool usage without external services.

V1 toolset (selected):

- Document QA/RAG:
  - chunk + embed + retrieve over uploaded files (local index)
- Code execution (sandboxed):
  - restricted Python for transforms/summaries; gated by approval
- Filesystem:
  - read saved uploads; write generated artifacts; gated by approval

## Frontend (Next.js) design

### UI pages/sections

- Chat thread list + active chat view
- Provider/model selector per request
- File upload panel (shows uploaded files + extracted text preview)
- “Graph debug” toggle:
  - Token streaming view
  - Event streaming view (node start/end, tool calls, interrupts)
- “Human review” panel shown when graph interrupts (editable JSON-like state or structured forms)

## Provider switching (LLM adapter layer)

Implement a small provider factory so the graph can call `get_chat_model(provider, model, **kwargs)` and remain provider-agnostic.

Supported providers (target):

- **OpenAI** via `langchain-openai`
- **Anthropic** via `langchain-anthropic`
- **Gemini** via `langchain-google-genai`
- **Ollama local** via `langchain-ollama` (connect to `http://localhost:11434`)
- **Ollama remote/cloud**: same client, configurable base URL

Provider selection is **request-scoped** (passed from frontend), while API keys/base URLs come from env vars.

## Dependencies (expected)

Backend:

- `fastapi`, `uvicorn`
- `langgraph`, `langgraph-prebuilt`
- `langgraph-checkpoint-sqlite`
- `langchain-core`
- Provider libs (as needed):
  - `langchain-openai`
  - `langchain-anthropic`
  - `langchain-google-genai`
  - `langchain-ollama`
- File parsing:
  - PDF: `pypdf` (or `pdfminer.six` if needed)
  - DOCX: `python-docx`

RAG (local):

- Embeddings provider aligned with selected LLM provider where possible, but default to a local embedding model if desired.
- Vector store: start with a lightweight local option.

Frontend:

- Next.js (App Router)
- SSE client (native `EventSource`)

**Not included**:

- `langsmith`
- `langgraph-sdk`
- Studio / `langgraph dev`

## Milestones

1. **Concept mapping complete**
   - Confirm which notebooks/patterns we’re adopting and which we’re excluding (Studio/SDK/LangSmith).

2. **Graph + state design finalized**
   - Define state schema, nodes, reducers, checkpoints, and interruption points.
   - Define runtime plan schema + approval gates + event stream contract.

3. **Monorepo scaffold + end-to-end happy path**
   - Next.js chat -> FastAPI -> LangGraph -> SSE streaming -> persisted thread in SQLite.
   - Supervisor generates plan -> approval -> specialist execution -> progress UI.

4. **Quality & safety hardening**
   - Interruptions for missing info + a clean UI review loop.
   - Event streaming toggle for learning.

## Open questions to confirm before implementation

- Should we build **single-user local first** (no auth) but keep extension points for multi-user later?
- Any must-have models for each provider (defaults)?
- Any tools beyond file parsing you want in v1 (web search, DB query, etc.)?

Agent architecture (v1):

- Supervisor agent (single) orchestrates and owns user-facing messages.
- Specialist agents (multiple) are invoked for isolated tasks with minimal context:
  - `planner` (writes plan)
  - `doc_analyst` (RAG over uploads)
  - `writer` (drafting)
  - `coder` (sandboxed transforms)
