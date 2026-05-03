# Specs — Technical Specification Library

A repository of implementation-ready specifications for use with AI agents (Claude Code, Cursor, Copilot, etc.) and developers. Each spec is a complete technical document describing **how to build** a feature from scratch — architectural decisions, configurations, models, flows, and checklists included.

---

## Objective

Accelerate project development by eliminating time spent on research and repetitive decision-making. Instead of starting from scratch every time you need authentication, payments, or internationalization, you open the corresponding spec and execute — whether you are a human developer or an AI agent.

**Principles:**
- Specs are generic: not tied to any specific project.
- An AI should be able to read the spec and implement the system from scratch.
- Each document covers architectural decisions, code, configuration, and a final checklist.

---

## Available Specs

### Core Agent Architecture

| Spec | What it covers |
|------|---------------|
| [spec-multiagent-pipeline.md](spec-multiagent-pipeline.md) | 5-agent sequential pipeline: Router → Scope Guard → Generator → Assessor → Refiner. Includes orchestration, per-agent model overrides, and quality thresholds. |
| [spec-agent-observability.md](spec-agent-observability.md) | Execution tracing, timeline events, and structured metadata for every pipeline run. Enables debugging UIs and structured logging. |

### Data & Memory

| Spec | What it covers |
|------|---------------|
| [spec-database-memory-rag.md](spec-database-memory-rag.md) | Hybrid memory with Redis (fast, ephemeral) + PostgreSQL (durable). pgvector for semantic RAG. Includes schema, chunking, incremental embeddings, and graceful degradation. |

### Multi-Level RAG Architecture

| Spec | What it covers |
|------|---------------|
| [spec__rag_multinivel_arquitetura.md](spec__rag_multinivel_arquitetura.md) | 5-stage pipeline for document ingestion: Decomposition, Enrichment, Distribution, Vector Indexing, and Orchestrated RAG. |
| [spec__rag_data_structure.md](spec__rag_data_structure.md) | Schema for structured JSON documents, serverless indices (topics/relations), and pgvector database schema for semantic search. |
| [spec__rag_ai_engine.md](spec__rag_ai_engine.md) | LLM-based enrichment (summaries, entities), conversational orchestration (query re-writing, evidence extraction), and hard grounding validation. |

### LLM & Configuration

| Spec | What it covers |
|------|---------------|
| [spec-multi-provider-llm.md](spec-multi-provider-llm.md) | Unified LLM client for Ollama, OpenAI, Claude, and Gemini. Provider abstraction, model resolution priority, and runtime key updates without restart. |
| [spec-declarative-behavior-config.md](spec-declarative-behavior-config.md) | YAML-driven domain/behavior configuration. Externalizes system prompts, guardrails, pedagogy rules, and scope keywords into data files — no code changes for new domains. |
| [spec-runtime-config-api.md](spec-runtime-config-api.md) | REST endpoints for runtime configuration: set API keys, assign models per agent, view memory stats. Enables A/B testing and zero-restart ops changes. |

### Fullstack Implementation (Django)

| Spec | What it covers |
|------|---------------|
| [spec-django-internationalization.md](spec-django-internationalization.md) | Complete guide for multi-language support: session/cookie strategy, i18n_patterns, translation files (.po/.mo), and UI language selectors. |
| [spec-django-cart-payments.md](spec-django-cart-payments.md) | Generic shopping cart and payment system: Order/OrderItem snapshoting, manual (Pix) and gateway payments, and transaction idempotency. |

### Infrastructure

| Spec | What it covers |
|------|---------------|
| [spec-fullstack-docker-network.md](spec-fullstack-docker-network.md) | Docker Compose topology for FastAPI backend + Next.js frontend + PostgreSQL + Redis + Ollama. Internal DNS, CORS, healthchecks, and production considerations. |

---

## Recommended Reading Order

For a new AI application from scratch:

1. [spec-multiagent-pipeline.md](spec-multiagent-pipeline.md) — understand the agent pattern first
2. [spec-multi-provider-llm.md](spec-multi-provider-llm.md) — wire the LLM layer
3. [spec-database-memory-rag.md](spec-database-memory-rag.md) — add memory and RAG
4. [spec__rag_multinivel_arquitetura.md](spec__rag_multinivel_arquitetura.md) — implement advanced document intelligence
5. [spec-declarative-behavior-config.md](spec-declarative-behavior-config.md) — externalize behavior
6. [spec-fullstack-docker-network.md](spec-fullstack-docker-network.md) — containerize
7. [spec-runtime-config-api.md](spec-runtime-config-api.md) — add operator controls
8. [spec-agent-observability.md](spec-agent-observability.md) — add tracing

---

## Roadmap / Planned Specs

### Django
- **Authentication & Authorization** — registration, social login (OAuth), 2FA, group-based permissions.
- **Multi-tenancy** — data isolation per organization with subdomains or separate schemas in PostgreSQL.
- **REST API with DRF** — versioning, JWT auth, throttling, documentation with drf-spectacular.
- **File Uploads** — images, documents, S3/R2 storage, async processing with Celery.
- **Real-time Notifications** — Django Channels, WebSockets, Firebase push notifications.
- **SaaS Subscription System** — plans, trials, recurring billing via Stripe, customer portal.
- **Audit & Activity Logs** — user action tracking, enhanced Django Admin.
- **Background Jobs with Celery** — queues, retries, beat for scheduled tasks, Flower monitoring.

### Next.js
- **Authentication with NextAuth.js** — social providers, sessions, route protection, middleware.
- **SSR / SSG / ISR** — when to use each strategy, caching, revalidation, and fallback.
- **App Router with Server Components** — nested layouts, streaming, suspense boundaries.
- **Forms with React Hook Form + Zod** — client/server validation, file uploads, error feedback.
- **Internationalization with next-intl** — locale routing, translations, date/currency formatting.
- **Django API Integration** — JWT auth in browser/server, cache, and stale-while-revalidate.
- **Design System with Tailwind + shadcn/ui** — themes, base components, accessibility, dark mode.

### General Python
- **CLI with Typer** — subcommands, options, validation, `.env` config, PyPI distribution.
- **Robust Scraping** — Playwright, proxy rotation, rate limit detection, result persistence.
- **ETL with Pandas + SQLAlchemy** — ingestion, transformation, incremental load, and schema migration.
- **Reusable API Client** — retry with backoff, auth, cache, Pydantic typing.
- **Automation with Prefect / Airflow** — DAGs, dependencies, observability, cloud execution.

### Data Science & Machine Learning
- **Data Pipeline with Pandas & Polars** — cleaning, feature engineering, dataset versioning.
- **Model Training & Tracking** — MLflow, experiments, artifact registry, run comparisons.
- **Serving Models with FastAPI** — inference endpoints, input validation, latency, and batching.
- **Standardized EDA** — reproducible exploratory analysis with structured notebooks and automated reports.
- **Lightweight Feature Store** — storage, versioning, and retrieval of features for training and inference.

### Agents & AI
- **Agent with Claude API** — tool use, prompt caching, memory, multi-turn conversation.
- **RAG (Retrieval-Augmented Generation)** — chunking, embedding, vector store (pgvector / Chroma), reranking.
- **Persistent Memory Agent** — short-term, long-term, episodic; DB storage and semantic retrieval.
- **Multi-Agent with Claude Code SDK** — orchestration, specialized sub-agents, handoff, and supervision.
- **LLM Evaluation (LLM-as-judge)** — metrics, test datasets, CI for response quality.
- **Fine-tuning & Lightweight RLHF** — preference collection, LoRA, before/after evaluation.
- **Custom MCP Server** — creating MCP tools to integrate internal systems with Claude Code.

---

## How to Use

### With an AI Agent
```
Read the [spec].md file and implement the described system in my project.
Follow the architectural decisions in section 1 before starting.
```

### As a Technical Reference
Open the spec before starting a feature. Use the **Implementation Checklist** at the end of each document to ensure nothing is missed.

---

## Contributing

1. Create a file following the pattern `YYYY-MM-DD-feature-name-technology-spec.md`.
2. Follow the structure: Overview → Architectural Decisions → Step-by-step Implementation → Checklist.
3. The spec must be generic enough to work on any project within the stack.

---

## Spec Structure

```
# Feature Name — Implementation Spec

## 1. Overview
## 2. Architectural Decisions       ← choices with trade-offs explained
## 3. Configuration / Settings
## 4. Models / Schema
## 5. Business Logic
## 6. Endpoints / Views / API
## 7. Testing
## 8. Implementation Checklist      ← verifiable items at the end
```

---

## Tech Stack Reference

| Layer | Technology |
|-------|-----------|
| Backend | Python 3.12, FastAPI, Pydantic v2, asyncpg, redis-py |
| Frontend | Next.js 15, React 19, TypeScript |
| Database | PostgreSQL 16 + pgvector extension |
| Cache | Redis 7 |
| Local LLM | Ollama |
| Cloud LLMs | OpenAI, Anthropic, Google Gemini |
| Containers | Docker, Docker Compose |

---

## License

MIT — use, adapt, and distribute freely.
