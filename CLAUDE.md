# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the Application

```bash
# Install dependencies
uv sync

# Start the server (from repo root, using Git Bash on Windows)
./run.sh

# Or manually
cd backend && uv run uvicorn app:app --reload --port 8000
```

Requires a `.env` file at the repo root with `ANTHROPIC_API_KEY=...` (copy from `.env.example`).

The server runs at `http://localhost:8000`. On startup it auto-ingests any `.txt`/`.pdf`/`.docx` files in `docs/` into ChromaDB, skipping already-indexed courses.

There are no tests or linters configured in this project.

## Architecture

This is a full-stack RAG chatbot. The backend is a FastAPI app (`backend/`) that serves the frontend (`frontend/`) as static files. All backend modules are imported as flat local modules (no package prefix) since uvicorn runs from inside `backend/`.

### RAG Pipeline

`RAGSystem` (`rag_system.py`) is the central orchestrator. A query goes through:

1. **`SessionManager`** — retrieves prior conversation history (last 2 exchanges, stored in-memory as formatted strings injected into the Claude system prompt)
2. **`AIGenerator`** — calls Claude with a `search_course_content` tool definition; Claude decides whether to call it
3. If tool use: **`ToolManager` → `CourseSearchTool` → `VectorStore.search()`** — runs semantic similarity search against ChromaDB, returns formatted text chunks
4. Tool results are sent back to Claude in a second API call (no tools this time) to produce the final answer
5. Sources and response saved back to `SessionManager`

### Vector Store (ChromaDB)

Two collections in `./chroma_db` (relative to `backend/`):
- `course_catalog` — one document per course (title + metadata); used for **semantic course name resolution** when a `course_name` filter is passed to `search()`
- `course_content` — chunked lesson text; queried for actual RAG retrieval

Embeddings use `all-MiniLM-L6-v2` via `sentence-transformers`. ChromaDB is persistent on disk.

### Document Format

Course files must follow this structure:
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

`DocumentProcessor` parses this format, chunks lesson text at 800 chars with 100-char overlap, and prepends lesson/course context to the first chunk of each lesson.

### Key Config (`backend/config.py`)

| Setting | Default |
|---|---|
| `ANTHROPIC_MODEL` | `claude-sonnet-4-20250514` |
| `EMBEDDING_MODEL` | `all-MiniLM-L6-v2` |
| `CHUNK_SIZE` | 800 |
| `CHUNK_OVERLAP` | 100 |
| `MAX_RESULTS` | 5 |
| `MAX_HISTORY` | 2 exchanges |
| `CHROMA_PATH` | `./chroma_db` |