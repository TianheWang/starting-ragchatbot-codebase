# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

**Setup:**
```bash
uv sync                        # Install dependencies
cp .env.example .env           # Then add your ANTHROPIC_API_KEY
```

**Run the app:**
```bash
./run.sh
# or manually:
cd backend && uv run uvicorn app:app --reload --port 8000
```

> Always use `uv run` to execute the server. Never use `pip` directly.

The server runs at `http://localhost:8000` (UI) and `http://localhost:8000/docs` (API docs).

## Architecture

This is a full-stack RAG chatbot: a vanilla JS frontend served by a FastAPI backend, with ChromaDB for vector storage and Claude as the AI backbone.

### Query pipeline (the core flow)

User query → `POST /api/query` → `RAGSystem.query()` → `AIGenerator.generate_response()` → **Claude API (1st call)** → if `stop_reason == "tool_use"` → `CourseSearchTool.execute()` → `VectorStore.search()` → **Claude API (2nd call)** → response + sources back to browser.

Claude drives retrieval via tool use — there is no pre-retrieval step. The AI decides whether to call `search_course_content` and what parameters to pass.

### Key backend modules

| File | Role |
|---|---|
| `app.py` | FastAPI entry point; two endpoints (`/api/query`, `/api/courses`); also serves the frontend as static files |
| `rag_system.py` | Orchestrator — wires together all components and owns the `query()` method |
| `ai_generator.py` | Wraps the Anthropic client; handles the two-call tool-use loop |
| `search_tools.py` | `Tool` ABC + `CourseSearchTool` + `ToolManager`; tracks last sources for UI display |
| `vector_store.py` | ChromaDB wrapper with two collections: `course_catalog` (metadata) and `course_content` (chunks) |
| `document_processor.py` | Parses `.txt`/`.pdf`/`.docx` course files into `Course` + `CourseChunk` objects |
| `session_manager.py` | In-memory conversation history keyed by session ID (not persisted across restarts) |
| `config.py` | Single `Config` dataclass; model, chunk size, overlap, history length, ChromaDB path |
| `models.py` | Pydantic models: `Course`, `Lesson`, `CourseChunk` |

### Course document format

Course files in `docs/` must follow this structure for `DocumentProcessor` to parse them correctly:

```
Course Title: <title>
Course Link: <url>
Course Instructor: <name>

Lesson 1: <title>
Lesson Link: <url>
<lesson content...>

Lesson 2: <title>
...
```

`course_title` is used as the unique ID in ChromaDB — duplicate titles are skipped on re-load.

### Frontend

Single-page app (`frontend/`) with no build step. `script.js` manages session state, sends `POST /api/query` with `{query, session_id}`, and renders Markdown responses via `marked.parse()`. Sources appear in a collapsible `<details>` block.

### Configuration knobs (`backend/config.py`)

- `ANTHROPIC_MODEL` — Claude model in use
- `CHUNK_SIZE` / `CHUNK_OVERLAP` — text chunking for ingestion
- `MAX_RESULTS` — number of ChromaDB hits returned per search
- `MAX_HISTORY` — conversation turns kept in session (in-memory only)
- `CHROMA_PATH` — where ChromaDB persists its data (`./chroma_db` relative to `backend/`)
