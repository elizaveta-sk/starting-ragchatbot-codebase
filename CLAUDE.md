# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

A course-materials RAG chatbot: FastAPI backend + ChromaDB vector store + Claude (via the `anthropic` SDK) with a vanilla HTML/JS frontend. Python 3.13, managed with `uv`.

## Commands

```bash
uv sync                                              # install dependencies
./run.sh                                             # start server (http://localhost:8000, API docs at /docs)
cd backend && uv run uvicorn app:app --reload --port 8000   # same, manually
```

Requires `ANTHROPIC_API_KEY` in a `.env` file at the repo root (see `.env.example`). There is no test suite, linter, or build step configured.

The server must be started from `backend/`: paths in `app.py` (`../docs`, `../frontend`) and `config.CHROMA_PATH` (`./chroma_db`) are relative to that directory.

## Architecture

Request flow for `POST /api/query`:

1. `app.py` creates a session if none was given, then calls `RAGSystem.query()`.
2. `rag_system.py` (the orchestrator wiring all components together) pulls formatted history from `SessionManager` and calls `AIGenerator.generate_response()` with the tool definitions from `ToolManager`.
3. `ai_generator.py` makes a Claude call with `tool_choice: auto`. Retrieval is **tool-based**, not pre-fetched: Claude decides whether to call `search_course_content`. If it does, the tool is executed and a second call is made with `tool_choice: none` (one tool round max). The same `tools` list is kept on the follow-up call because changing it would invalidate replayed thinking blocks; text is extracted by filtering out non-text (thinking) blocks.
4. `search_tools.py`: `CourseSearchTool.execute()` calls `VectorStore.search()` and records `last_sources`, which `RAGSystem` reads via `ToolManager.get_last_sources()` and then resets. New tools subclass `Tool` and are registered on the `ToolManager` in `RAGSystem.__init__`.
5. `vector_store.py` uses two Chroma collections:
   - `course_catalog`: one doc per course (id = course title), metadata includes instructor, links, and lessons serialized as JSON. Used to fuzzy-resolve a partial `course_name` to an exact title via semantic search.
   - `course_content`: text chunks with `course_title`, `lesson_number`, `chunk_index` metadata; searched with a `where` filter built from the resolved title/lesson.

Conversation history (`session_manager.py`) is in-memory only, capped at `MAX_HISTORY` exchanges, and injected into the **system prompt** as plain text rather than as message turns.

### Document ingestion

On startup, `app.py` loads every `.txt/.pdf/.docx` in `docs/` via `RAGSystem.add_course_folder()`, skipping courses whose title already exists in Chroma (the DB persists in `backend/chroma_db/`). Delete that directory or use `clear_existing=True` to force re-indexing after changing chunking or the embedding model.

`document_processor.py` expects this file format:

```
Course Title: <title>
Course Link: <url>
Course Instructor: <name>

Lesson 0: <lesson title>
Lesson Link: <url>
<content...>
Lesson 1: ...
```

Content is split into sentence-based chunks (`CHUNK_SIZE`/`CHUNK_OVERLAP` in characters) and prefixed with course/lesson context before embedding.

### Configuration

All tunables (model, embedding model `all-MiniLM-L6-v2`, chunk sizes, `MAX_RESULTS`, `MAX_HISTORY`, Chroma path) live in the `Config` dataclass in `backend/config.py`.

### Frontend

`frontend/` is static files served by FastAPI at `/` (mounted last in `app.py`, so API routes must be registered before it). `script.js` calls `/api/query` and `/api/courses` and keeps `session_id` across messages.
