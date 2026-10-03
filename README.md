# Policy-Compliance Verification Agent

An agentic workflow that checks dashboard requests (expense reports, procurement requests, access changes, time off) against company policy. It flags violations, cites the exact policy text behind each verdict, and suggests concrete fixes.

Built for ETH course 275-0005-00L *From Data to Solutions* (Weekend 6 project). It reuses the RAG system from the previous project (`rag/`) as a retrieval tool.

## How it works

Each request passes through a pipeline of small steps. Every step hands a typed object to the next one, so each step can be tested on its own and every decision traces back to a policy sentence.

```
Action → 1. Validate → 2. Build context → 3. Retrieve policies → 4. Verifier → 5. Solution → 6. Display
                                                                                            ↓
                                                                                      UI feedback
7. Pipeline: wires steps 1–6 into a single verify(action) call
```

| Step | Module | What it does |
|---|---|---|
| 1 | `agentic/action_validation.py` | Rejects malformed requests deterministically, before any LLM call |
| 2 | `agentic/context_builder.py` | Turns the action into a RAG query, filtered to the governing policy document |
| 3 | `agentic/policy_tool.py` | Retrieves relevant policy chunks (hybrid search over Qdrant) |
| 4 | `agentic/verifier_agent.py` | LLM decides compliant / non-compliant, citing retrieved text |
| 5 | `agentic/solution_agent.py` | LLM proposes one grounded fix per problem |
| 6 | `agentic/display_agent.py` | Maps results to a fixed set of UI tools (highlight, warn, cite, suggest, mark OK) |
| 7 | `agentic/pipeline.py` | `VerifierPipeline.verify()` orchestrates everything |

### Supported action types

| Action type | Governing policy |
|---|---|
| `expense_report` | `expense_reimbursement_policy.md` |
| `procurement_request` | `procurement_policy.md` |
| `access_change` | `information_security_policy.md` |
| `time_off_request` | `leave_and_absence_policy.md` |

Field schemas are defined in `agentic/action_types.py`. The policy corpus is in `rag/data/`.

## Project layout

```
agentic/     Pipeline steps, data models, prompts
rag/         RAG system from the previous project (ingestion, retrieval, Qdrant, clients)
solutions/   Notebook-exported implementations, loaded over the reference code
server.py    FastAPI backend
clients.py   Builds LLM + embedding clients from an OpenRouter key
frontend/    React + Vite dashboard
examples/    Sample action payloads (compliant and non-compliant)
notebook/    Course notebook, project brief, documentation, defense notes
tests/       Test harness and mocks
```

## Getting started

### Prerequisites

- Python 3.12+ and [uv](https://docs.astral.sh/uv/)
- Node.js (for the frontend)
- An [OpenRouter](https://openrouter.ai/) API key

### Backend

```bash
uv sync
uv run uvicorn server:app --reload
```

The API runs at `http://localhost:8000`. The server has no API key of its own: the caller sends their OpenRouter key in the `X-OpenRouter-Key` header on every request, and the key is never stored or logged.

The first `/verify` request ingests the policy documents into the local Qdrant store (`./qdrant_data`) using the caller's key. Later requests reuse the store.

### Frontend

```bash
cd frontend
npm install
npm run dev
```

Open the URL Vite prints, then enter your OpenRouter key in the dashboard. Set `VITE_API_BASE` if the backend is not on `localhost:8000`.

### Try it from the command line

```bash
curl -X POST http://localhost:8000/verify \
  -H "Content-Type: application/json" \
  -H "X-OpenRouter-Key: $OPENROUTER_API_KEY" \
  -d @examples/expense_too_expensive_no_receipt.json
```

## API

| Method | Path | Description |
|---|---|---|
| `GET` | `/health` | Liveness check |
| `GET` | `/status` | Which notebook-exported functions are implemented |
| `GET` | `/action-types` | Action catalogue and field schemas |
| `GET` | `/policies` | Policy documents currently in the store |
| `POST` | `/verify` | Verify an action. Returns 400 for invalid input, 401 for a bad key |

## Configuration

| Variable | Default | Purpose |
|---|---|---|
| `AGENT_IMPL` | `auto` | `auto`: use exported solutions where present, else reference code. Other options: `reference`, `student` |
| `AGENT_DB_PATH` | `./qdrant_data` | Qdrant storage path |
| `AGENT_COLLECTION` | `documents` | Qdrant collection name |
| `EMBEDDING_MODEL` | `google/gemini-embedding-001` | Embedding model on OpenRouter. Try `openai/text-embedding-3-small` if embeddings fail intermittently |

The default chat model is `google/gemini-3-flash-preview` (set in `rag/clients/llm.py`).

## Documentation

- `notebook/DOCUMENTATION.md`: full walkthrough of the project and design decisions
- `notebook/DEFENSE.md`: condensed presentation notes
- `notebook/FDD26-W6-Project.pdf`: original project brief
