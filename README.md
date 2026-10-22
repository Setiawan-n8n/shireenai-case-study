# ShireenAI — Engineering Case Study

ShireenAI is an AI book-writing assistant ("AI ghost writer"). Users don't write by hand: they talk to an AI agent that learns their writing style (the **Author DNA**), proposes a book blueprint, writes chapter by chapter, critiques drafts, and revises specific passages on request.

This repository is a **public case study** of the AI agent engine I built for it: the architecture, how the agent is constrained, and a log of the real problems I hit and how I solved them. The workflow source and infrastructure details are kept in a private repository.

## At a glance

| | |
|---|---|
| **Role** | AI / backend engineer: designed and built the agent engine (n8n + LLMs + RAG) |
| **Stack** | n8n (self-hosted), LangChain agent nodes, OpenAI `gpt-4o-mini`, Google Gemini 2.5 Flash-Lite (via OpenRouter), `gemini-embedding-001` embeddings, PostgreSQL + pgvector (Supabase, self-hosted), Gotenberg for PDF processing; web app in Laravel + Inertia + Vue |
| **Scale of iteration** | 25+ major versions (about 340 workflow revisions in total) |
| **Retrieval** | Two vector stores in pgvector (768-dim): one for the author's writing-style samples, one for book reference material |
| **Status** | Main engine migrated to a self-hosted stack and under end-to-end testing; multi-agent orchestration (task manager + writing, style-profiling, RAG/source, and research agents) built and in testing |

## Design principles

1. **The web app holds no AI logic.** Prompts, phases, and rules all live in the agent engine. The web app sends a structured request and receives structured callbacks.
2. **The agent is a state machine, not a free chatbot.** Each phase (Author DNA → understanding summary → blueprint → outline → chapter writing → editor review → revision by location) has a goal, an allowed output format, and explicit prohibitions. A pre-action validator rejects actions that don't fit the current phase.
3. **The user approves before the agent moves on.** Blueprints and chapters go back to the user for approval through a callback, never straight into the next phase.
4. **Structured output beats prompt features.** Every action returns a defined JSON shape, and that JSON is validated before anything moves downstream.

## Documents

- [Architecture](docs/architecture.md): components, request flow, ingestion pipelines, retrieval, data model
- [Problems & solutions](docs/problems-and-solutions.md): about 40 real issues across RAG, API errors, agent behaviour, runtime, and infrastructure, with root causes and fixes

## Key lessons

- **RAG failures are usually silent.** A wrong `user_id`, a vector-dimension mismatch, or a swallowed SQL error all look like "the model just didn't use the context". Check every write explicitly.
- **Don't trust data that has passed through an agent.** LLM agent nodes don't reliably forward every input field. Re-read identifiers from the original trigger.
- **One hard prompt rule can break the whole output.** Making a "proactive follow-up question" mandatory caused the model to corrupt its JSON, and the corrupted JSON cascaded into HTTP 422 errors downstream.
- **Validate at the boundary.** Routing invalid agent output to an error handler stopped one bad step from breaking every step after it.
- **Never hard-code secrets in workflow nodes.** They travel with every export.

## What I'd do next

- Finish end-to-end tests against a fixed set of reference books and score output quality (style consistency, structure adherence).
- Move prompt construction into a dedicated, testable step instead of a long inline expression.
- Add an evaluation harness for retrieval quality (recall at k on known passages) before tuning chunking or embeddings.
