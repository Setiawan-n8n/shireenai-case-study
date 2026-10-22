# Architecture

## Components

```
┌──────────────────────┐   POST (structured request)   ┌──────────────────────────────┐        ┌───────────────────────┐
│ Web app              │ ───────────────────────────▶  │ Agent engine (n8n)           │ ─────▶ │ LLM providers         │
│ Laravel/Inertia/Vue  │                               │  • main writing workflow     │        │ gpt-4o-mini (primary) │
│ UI, state, approvals │ ◀───────────────────────────  │  • style (DNA) retrieval     │        │ Gemini 2.5 Flash-Lite │
└──────────────────────┘   callbacks: ask-approval,    │  • reference (RAG) retrieval │        │ gemini-embedding-001  │
                           updated-blocks, general-chat└──────────────┬───────────────┘        └───────────────────────┘
                                                                      │
                                                                      ▼
                                                     PostgreSQL + pgvector (768-dim)
                                                     style chunks · reference chunks ·
                                                     writing profiles · agent memory
```

| Component | Technology |
|---|---|
| Web app | Laravel + Inertia + Vue |
| Agent engine | n8n (self-hosted), LangChain agent nodes |
| Chat models | OpenAI `gpt-4o-mini` (primary), Gemini 2.5 Flash-Lite via OpenRouter (fallback) |
| Embeddings | `gemini-embedding-001`, fixed at 768 dimensions |
| Web research tool | Online-search model exposed to the agent as a tool |
| Database | PostgreSQL 15 + pgvector (Supabase, self-hosted) |
| PDF processing | Gotenberg (PDF split) |

## Entry points

| Endpoint | Purpose |
|---|---|
| Book processing | Main flow: every writing or chat action from the web app |
| Author DNA upload | Writing samples (PDF, DOCX, TXT) → style analysis → style vector store and writing profile |
| Reference upload | Book reference files → chunks and embeddings → reference vector store |
| Task-list upload | Stores task lists (markdown) used as planning context |

## Main request flow

```
Trigger
 └ Validate payload
   └ Understand & enrich context      detect the action, resolve user_id / book_id from every payload location
     └ Pre-action validation          is the action valid? does it fit the current phase? is a blueprint approved?
       └ Planning                     internal step list for this action
         ├ Style (DNA) retrieval  ─┐  sub-workflows, run in parallel
         └ Reference (RAG) retr.  ─┴ Merge → Build combined context
                                        └ Fetch writing profile and task list
                                          └ Fetch context (final assembly)
                                            └ Route by action (11 actions)
                                              └ Writing agent  (model + fallback model, book memory, research tool)
                                                └ Validate output
                                                  ├ valid   → Prepare payload → route to the right callback
                                                  └ invalid → Error handler → user-facing fallback message
```

### Actions and phases

| Phase | Allowed actions |
|---|---|
| Blueprint | `EXPLORE_BLUEPRINT`, `GENERATE_BLUEPRINT`, `REJECTION_FEEDBACK` |
| Outline | `EXPLORE_OUTLINE`, `GENERATE_OUTLINE`, `REJECTION_FEEDBACK` |
| Writing | `WRITE_CHAPTER`, `EDITOR_REVIEW`, `REVISION_BY_LOCATION`, `REWRITE_CHAPTER` |
| Final polish | `EDITOR_REVIEW`, `REVISION_BY_LOCATION` |
| Any | `GENERAL_CHAT`, `DEEP_RESEARCH` |

### Callback contract

| Callback | Used for | Rule that mattered |
|---|---|---|
| `ask-approval` | Blueprints, outlines, chapters waiting for user approval | `content` must be non-empty, or the backend rejects it with HTTP 422 |
| `updated-blocks` | Targeted revisions | Only the changed blocks are sent |
| `general-chat` | Conversational replies and friendly error messages | Always carries `book_id` and `job_id` so the web app can match the reply to the request |

## Ingestion pipelines

**Author DNA (writing style)**

```
Upload → acknowledge immediately (processing continues asynchronously)
 → normalise input (base64 or binary) → route by file type (PDF, DOCX, TXT) → extract text
 → chunk → loop: style-analysis agent per batch → parse the agent's JSON (tolerant parser)
 → embed (768-dim) → insert into style vector store
 → build and upsert the user's writing profile (markdown + structured fields)
```

**Reference material (RAG)**

```
Upload → acknowledge immediately → split PDF (Gotenberg) → extract text → register document
 → loop: summarisation agent per chunk → embed → safe SQL insert (empty embedding → NULL) → reference vector store
```

## Retrieval

Two sub-workflows share the same pattern:

```
Normalise input → embed the query → vector search (filtered by user) → aggregate → build context
 → has results? → return context directly to the caller, or answer with an LLM chain
```

The style retrieval also loads the user's stored writing profile. Results from both are merged into labelled sections (`WRITING STYLE CONTEXT`, `BOOK KNOWLEDGE CONTEXT`) before the writing agent sees them.

## Data model (simplified)

| Table | Contents | Vector |
|---|---|---|
| `dna_rag` | Writing-style chunks: summary, raw text, keywords, topic | `vector(768)` |
| `knowledge_rag` | Reference-material chunks | `vector(768)` |
| `document_registry` | Uploaded documents | — |
| `user_writing_profiles` | One writing profile per user (markdown + structured fields) | — |
| `task_lists` | Planning task lists | — |
| Agent memory tables | Chat memory for the writing, style, and summarisation agents | — |

## Multi-agent version (in testing)

The single large workflow was then split into an orchestrator and specialised sub-agents:

| Agent | Responsibility |
|---|---|
| Task Manager (orchestrator) | Interprets the request, plans steps, dispatches to sub-agents, assembles the result |
| Writing agent | Blueprint, outline, chapter writing, revision |
| Style (DNA) agent | Builds and serves the author's writing profile |
| Source / RAG agent | Retrieves and summarises reference material |
| Research agent | Web research on demand |
