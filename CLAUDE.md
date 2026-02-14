# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the Application

**Quick Start:**
```bash
./run.sh
```

**Manual Start:**
```bash
cd backend
uv run uvicorn app:app --reload --port 8000
```

**Prerequisites:**
- Requires `.env` file with `ANTHROPIC_API_KEY=your_key_here`
- Python 3.13+ with `uv` package manager
- On startup, the app auto-loads course documents from `docs/` folder

**Access Points:**
- Web Interface: http://localhost:8000
- API Documentation: http://localhost:8000/docs

## Architecture Overview

This is a RAG (Retrieval-Augmented Generation) chatbot system with a **tool-based search pattern** where Claude decides when to search the knowledge base rather than always injecting context.

### Query Flow Pattern

**Critical:** The system uses a **two-stage Claude API interaction**:

1. **First API call**: Claude receives the query with tool definitions and decides whether to use the `search_course_content` tool
2. **Tool execution**: If Claude chooses to search, the system executes vector search and returns formatted results
3. **Second API call**: Claude synthesizes the tool results into a final answer

This differs from traditional RAG where context is always retrieved upfront. Here, Claude has agency to decide when searching is necessary.

### Dual Collection Vector Store Design

**ChromaDB uses TWO separate collections** (not one):

1. **`course_catalog` collection**:
   - Stores course metadata (title, instructor, course_link, lessons_json)
   - Used for **semantic course name resolution**
   - Example: User searches "MCP" → finds "Introduction to Model Context Protocol"
   - IDs are course titles (used as unique identifiers)

2. **`course_content` collection**:
   - Stores chunked course content
   - Each chunk has metadata: `course_title`, `lesson_number`, `chunk_index`
   - IDs format: `{course_title_with_underscores}_{chunk_index}`

**Search flow:**
```
Query → Resolve course name (if provided) via catalog collection
      → Build filter dict
      → Search content collection with filters
      → Return results
```

### Component Relationships

**Data flow:**
```
DocumentProcessor → processes docs → Course + CourseChunks
                                         ↓
VectorStore ← adds metadata to catalog + content to content collection
                                         ↓
CourseSearchTool ← executes search via VectorStore.search()
                                         ↓
ToolManager ← coordinates tool execution, tracks sources
                                         ↓
AIGenerator ← handles Claude API calls with tool use
                                         ↓
RAGSystem ← orchestrates everything, manages sessions
```

## Course Document Format

Documents in `docs/` must follow this structure:

```
Course Title: [Title Here]
Course Link: [URL]
Course Instructor: [Name]

Lesson 1: [Lesson Title]
Lesson Link: [URL]
[lesson content...]

Lesson 2: [Another Lesson]
Lesson Link: [URL]
[lesson content...]
```

**Important:**
- First 3 lines are course metadata
- Lesson markers: `Lesson <number>: <title>` (case-insensitive regex match)
- Lesson links are optional and must appear immediately after lesson marker
- Content is chunked at 800 chars with 100 char overlap (configurable in `config.py`)
- First chunk of each lesson gets prefixed with "Lesson N content:" for context

## Session Management

**Sessions store conversation history in memory** (not persisted):
- Format: `"session_1"`, `"session_2"`, etc.
- Keeps last `MAX_HISTORY * 2` messages (default: 4 messages = 2 exchanges)
- Frontend creates session on first query, maintains `currentSessionId`
- History is formatted as: `"User: question\nAssistant: answer"` and injected into Claude's system prompt

## Configuration

All settings in `backend/config.py`:
- `ANTHROPIC_MODEL`: Currently `"claude-sonnet-4-20250514"`
- `EMBEDDING_MODEL`: `"all-MiniLM-L6-v2"` (SentenceTransformer)
- `CHUNK_SIZE`: 800 chars
- `CHUNK_OVERLAP`: 100 chars
- `MAX_RESULTS`: 5 search results
- `MAX_HISTORY`: 2 conversation exchanges
- `CHROMA_PATH`: `"./chroma_db"` (relative to backend/)

## Key Implementation Details

### Tool Definition Pattern

The `CourseSearchTool` exposes this interface to Claude:
```python
{
  "name": "search_course_content",
  "input_schema": {
    "query": str,           # required
    "course_name": str,     # optional - semantic matching
    "lesson_number": int    # optional - exact filter
  }
}
```

Claude can use partial course names (e.g., "MCP") and the system resolves them via semantic search.

### Source Tracking

Sources flow through the system differently than traditional RAG:
- `CourseSearchTool` stores `last_sources` during search
- `ToolManager.get_last_sources()` retrieves them after AI response
- Sources are then reset for next query
- Frontend displays sources in collapsible `<details>` element

### Chunking Strategy

The `DocumentProcessor.chunk_text()` method:
- Splits on sentence boundaries (regex-based, handles abbreviations)
- Builds chunks by accumulating sentences up to `CHUNK_SIZE`
- Creates overlap by counting back sentences worth `CHUNK_OVERLAP` chars
- **Critical**: First chunk of lessons gets context prefix, but NOT all chunks (see lines 185-188 vs 232-234 inconsistency)

### Filter Building Logic

`VectorStore._build_filter()` creates ChromaDB filters:
- Course only: `{"course_title": title}`
- Lesson only: `{"lesson_number": num}`
- Both: `{"$and": [{"course_title": title}, {"lesson_number": num}]}`
- Neither: `None` (no filter)

## Modifying the System

**To add a new tool:**
1. Create class inheriting from `Tool` in `search_tools.py`
2. Implement `get_tool_definition()` and `execute()`
3. Register with `ToolManager` in `rag_system.py`

**To change chunking:**
- Modify `CHUNK_SIZE` and `CHUNK_OVERLAP` in `config.py`
- Clear existing data: set `clear_existing=True` in startup call
- Or manually delete `backend/chroma_db/` directory

**To change AI behavior:**
- Edit `AIGenerator.SYSTEM_PROMPT` for instructions
- Modify `temperature` (currently 0) or `max_tokens` (currently 800) in `base_params`

## API Endpoints

- `POST /api/query` - Submit query with optional session_id
- `GET /api/courses` - Get course statistics (count + titles)

Both return JSON. See `app.py` for Pydantic models.
