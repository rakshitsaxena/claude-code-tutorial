# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

### Running the Application
```bash
# Quick start (recommended)
./run.sh

# Manual start
cd backend
uv run uvicorn app:app --reload --port 8000
```

### Installing Dependencies
```bash
# Install uv if not already installed
curl -LsSf https://astral.sh/uv/install.sh | sh

# Install Python dependencies
uv sync
```

### Environment Setup
Create a `.env` file in the root directory:
```bash
ANTHROPIC_API_KEY=your_anthropic_api_key_here
```

### Clearing the Vector Database
```bash
rm -rf backend/chroma_db/
# Will regenerate on next server start from docs/*.txt files
```

## Architecture Overview

This is a **full-stack RAG (Retrieval-Augmented Generation) chatbot** that uses Anthropic's **tool-calling feature** as its core RAG mechanism. Unlike traditional RAG systems that always retrieve context first, Claude autonomously decides when to search the vector database based on the query.

### Key Components

**Backend (FastAPI):**
- `app.py` - FastAPI server serving both API endpoints and static frontend
- `rag_system.py` - Main orchestrator coordinating all RAG components
- `ai_generator.py` - Claude API integration with tool execution flow
- `search_tools.py` - Tool-calling implementation for semantic search
- `vector_store.py` - ChromaDB wrapper with dual-collection pattern
- `document_processor.py` - Document parsing and sentence-based chunking
- `session_manager.py` - In-memory conversation history (not persistent)
- `config.py` - Single dataclass for all configuration

**Frontend (Vanilla JS):**
- `frontend/index.html` - Single-page chat interface
- `frontend/script.js` - API communication and markdown rendering
- `frontend/styles.css` - Styling

**Data Storage:**
- `docs/` - Place .txt/.pdf/.docx course materials here (auto-loaded on startup)
- `backend/chroma_db/` - ChromaDB persistent storage (gitignored)

## Critical Design Patterns

### Tool-Calling RAG Pattern

The system does NOT follow the traditional "always retrieve, then generate" RAG pattern. Instead:

1. User query sent to Claude with search tool available
2. Claude decides whether search is needed (tool_choice: "auto")
3. If search needed, Claude calls `search_course_content` tool
4. ToolManager executes search, returns results
5. Claude receives results and synthesizes final answer

**Why this matters:** This allows Claude to answer general questions without searching, and prevents unnecessary vector lookups. The system prompt enforces **one search maximum per query**.

**Key files:**
- `ai_generator.py:74-87` - Tool registration and two-step conversation flow
- `search_tools.py:20-114` - Tool definition and execution
- `ai_generator.py:8-30` - System prompt enforcing single-search rule

### Dual Vector Store Collections

`vector_store.py` uses two ChromaDB collections:

1. **course_catalog** (lines 51, 135-160) - Stores course metadata for fuzzy name matching
   - Documents: Course titles
   - Metadata: Full course info, lessons as JSON string
   - Purpose: Resolve partial course names like "react" → "Introduction to React"

2. **course_content** (lines 52, 162-178) - Stores chunked lesson content
   - Documents: Text chunks with context headers
   - Metadata: `{course_title, lesson_number, chunk_index}`
   - Purpose: Semantic search with metadata filters

**Search workflow** (lines 78-100):
- If course name provided → semantic search in catalog to resolve exact title
- Build metadata filter for course_title and/or lesson_number
- Search content collection with filters and semantic similarity

### Session Management

`session_manager.py` maintains in-memory conversation history:
- **NOT persistent** - sessions reset on server restart
- Stores last `MAX_HISTORY` exchanges (default: 2 in config.py)
- History formatted as plain text and injected into system prompt
- Session ID generated as UUID4

## Document Processing Details

### Expected Document Format

Documents in `docs/` must follow this structure (`document_processor.py:100-104`):

```
Course Title: Introduction to React
Course Link: https://example.com
Course Instructor: John Doe

Lesson 0: Introduction
Lesson Link: https://example.com/lesson-0
[content...]

Lesson 1: Getting Started
Lesson Link: https://example.com/lesson-1
[content...]
```

### Chunking Strategy

**Sentence-based chunking** (not character splits):
- Regex-based sentence splitting that handles abbreviations (`document_processor.py:34`)
- CHUNK_SIZE=800 chars, CHUNK_OVERLAP=100 chars (configurable)
- Overlap calculated in sentences, not characters
- Each chunk gets context header: `"Lesson {N} content: {chunk}"` or `"Course {title} Lesson {N} content: {chunk}"`

**Known inconsistency** (lines 186-188, 234): First chunks vs. last chunks get different context formats.

### Deduplication

`rag_system.py:76-96` prevents re-processing documents:
- Checks existing course titles before processing
- Course title used as unique ID
- Duplicate titles would cause overwrites (no validation)

## API Contract

### POST /api/query

**Request** (`app.py:38-41`):
```json
{
  "query": "string",
  "session_id": "string | null"
}
```

**Response** (`app.py:43-47`):
```json
{
  "answer": "string",
  "sources": ["Course Title - Lesson X", ...],
  "session_id": "string"
}
```

### GET /api/courses

Returns list of all indexed courses with metadata.

## Configuration

All settings in `config.py` Config dataclass:

```python
ANTHROPIC_API_KEY: str         # From .env (required)
ANTHROPIC_MODEL: str           # claude-sonnet-4-20250514
EMBEDDING_MODEL: str           # all-MiniLM-L6-v2 (sentence-transformers)
CHUNK_SIZE: int = 800
CHUNK_OVERLAP: int = 100
MAX_RESULTS: int = 5           # Search results per query
MAX_HISTORY: int = 2           # Conversation exchanges to remember
CHROMA_PATH: str = "./chroma_db"
```

## Common Modification Tasks

### Adding a New Search Tool

1. Create subclass of `Tool` in `search_tools.py`
2. Implement `get_tool_definition()` and `execute()` methods
3. Register in `rag_system.py:23-25` during initialization
4. Tool manager automatically provides to Claude API

### Changing Chunking Strategy

Modify `document_processor.py`:
- `chunk_text()` method (lines 25-91) - Core chunking logic
- Adjust CHUNK_SIZE/CHUNK_OVERLAP in `config.py`
- Delete `backend/chroma_db/` to regenerate with new chunks

### Modifying AI Behavior

Edit `ai_generator.py`:
- SYSTEM_PROMPT (lines 8-30) - Claude's instructions
- Model/temperature/max_tokens (lines 69-73, using config values)
- Tool choice strategy (line 77): "auto" | "required" | specific tool

### Adding Metadata Fields

Requires updates in three places:
1. `models.py` - Add to data classes
2. `document_processor.py` - Extract from documents
3. `vector_store.py` - Store/retrieve from ChromaDB

## Important Gotchas

1. **Sessions are not persistent** - Lost on server restart (in-memory only)

2. **Course titles must be unique** - Used as ChromaDB document IDs, duplicates cause silent overwrites

3. **Document format is strictly expected** - First 3 lines must be Course Title/Link/Instructor or parsing fails

4. **Tool sources must be reset** - `rag_system.py:130-133` shows critical pattern: get sources, use them, then reset to prevent stale data in next query

5. **Lesson metadata serialized as JSON** - ChromaDB limitation requires JSON.dumps() when storing, JSON.loads() when retrieving (`vector_store.py:219-230`)

6. **Conversation history in system prompt** - Not using message history array, plain text injection (`ai_generator.py:61-65`)

7. **Wide-open CORS** - `allow_origins=["*"]` suitable for development only

8. **Startup errors don't halt server** - Document processing failures logged but server continues (`rag_system.py:37-50`)

9. **No input validation** - Course document format not validated, malformed docs may cause silent failures

10. **ChromaDB IDs are brittle** - Uses course title directly as ID, special characters could cause issues

## File Location Reference

When modifying specific functionality:

- **Search behavior** → `search_tools.py` (CourseSearchTool.execute method)
- **Chunking strategy** → `document_processor.py` (chunk_text method)
- **AI prompting** → `ai_generator.py` (SYSTEM_PROMPT constant)
- **API endpoints** → `app.py` (route handlers)
- **Vector search logic** → `vector_store.py` (search method)
- **Configuration values** → `config.py` (Config dataclass)
- **Document parsing** → `document_processor.py` (process_document method)
- **Session logic** → `session_manager.py` (SessionManager class)

## Technology Stack

- **Web Framework:** FastAPI 0.116.1
- **ASGI Server:** Uvicorn 0.35.0
- **Vector Database:** ChromaDB 1.0.15 with persistent storage
- **Embeddings:** sentence-transformers 5.0.0 (all-MiniLM-L6-v2 model)
- **LLM:** Anthropic Claude (via anthropic 0.58.2 SDK)
- **Package Manager:** uv (not pip/poetry)
- **Frontend:** Vanilla JavaScript with marked.js for markdown rendering

## Testing

**No test suite exists** - Manual testing required for all changes. Consider adding pytest tests for:
- Document parsing with various formats
- Chunking logic edge cases
- Vector store search filters
- Tool execution flow
- Session management
