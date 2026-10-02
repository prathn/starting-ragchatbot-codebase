# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

Dependencies are managed with `uv` (Python 3.13+). Requires `ANTHROPIC_API_KEY` in a root `.env` (see `.env.example`).

```bash
uv sync                                            # install dependencies
./run.sh                                           # start the app on :8000
cd backend && uv run uvicorn app:app --reload --port 8000   # manual start
```

- Web UI: http://localhost:8000 — API docs: http://localhost:8000/docs
- The server **must be started from `backend/`**: `app.py` uses relative paths `../docs`, `../frontend`, and `./chroma_db`.
- There is no test suite, linter, or build step configured. `main.py` at the root is an unused placeholder.

## Architecture

A course-materials RAG chatbot: FastAPI backend (`backend/`) + vanilla JS frontend (`frontend/`, served as static files by FastAPI at `/`).

**Retrieval is tool-driven, not pre-fetched.** `RAGSystem.query()` does not search the vector store itself. It passes the user query to Claude together with the `search_course_content` tool definition; Claude decides whether to search. Flow:

1. `app.py` `POST /api/query` → `RAGSystem.query()` (`rag_system.py`), creating a session if none is given.
2. `AIGenerator.generate_response()` (`ai_generator.py`) calls Claude with tools. If `stop_reason == "tool_use"`, `_handle_tool_execution()` runs the tool(s) via `ToolManager` and makes **one** follow-up call *without* tools — so only a single round of tool use is possible. The system prompt also limits Claude to one search per query.
3. `CourseSearchTool.execute()` (`search_tools.py`) → `VectorStore.search()`.
4. Sources for the UI are passed out-of-band: `CourseSearchTool._format_results()` stores them in `self.last_sources`; `RAGSystem` reads them via `ToolManager.get_last_sources()` and then calls `reset_sources()`. New tools that should surface sources must expose a `last_sources` attribute.

New tools implement the `Tool` ABC (`get_tool_definition()` returning an Anthropic tool schema, and `execute(**kwargs)`) and are registered in `RAGSystem.__init__`.

**Vector store (`vector_store.py`)** — persistent ChromaDB with `all-MiniLM-L6-v2` sentence-transformer embeddings and two collections:
- `course_catalog`: one doc per course, ID = course title; metadata holds instructor, course link, and lessons (serialized as JSON). Used to fuzzy-resolve a user-supplied `course_name` to an exact title via semantic search.
- `course_content`: text chunks with `course_title`, `lesson_number`, `chunk_index` metadata; searched with an optional `where` filter built from the resolved title and/or lesson number.

**Ingestion** — on startup, `app.py` calls `add_course_folder("../docs")`, which skips courses whose title already exists in the catalog (so editing an already-loaded doc won't re-index it; delete `backend/chroma_db` to rebuild). `document_processor.py` expects this file format:

```
Course Title: <title>
Course Link: <url>
Course Instructor: <name>

Lesson 0: <lesson title>
Lesson Link: <url>
<lesson text...>
Lesson 1: ...
```

Lesson text is split into sentence-based chunks (`CHUNK_SIZE=800`, `CHUNK_OVERLAP=100` chars). Note the chunk-prefixing is inconsistent: for lessons inside the loop only the first chunk gets a `Lesson N content:` prefix, while the final lesson's chunks all get `Course <title> Lesson N content:`.

**Sessions** — `session_manager.py` keeps history in memory only (lost on restart), trimmed to `MAX_HISTORY * 2` messages, and injected into the **system prompt** as plain text rather than as message turns.

**Config** — all tunables (model, embedding model, chunk sizes, `MAX_RESULTS`, `MAX_HISTORY`, Chroma path) live in the `Config` dataclass in `backend/config.py`.
