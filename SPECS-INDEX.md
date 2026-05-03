# AI Application Specs — Index

A collection of reusable architectural specs for building AI-powered applications. Each spec is self-contained and can be applied independently or combined.

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

### LLM & Configuration

| Spec | What it covers |
|------|---------------|
| [spec-multi-provider-llm.md](spec-multi-provider-llm.md) | Unified LLM client for Ollama, OpenAI, Claude, and Gemini. Provider abstraction, model resolution priority, and runtime key updates without restart. |
| [spec-declarative-behavior-config.md](spec-declarative-behavior-config.md) | YAML-driven domain/behavior configuration. Externalizes system prompts, guardrails, pedagogy rules, and scope keywords into data files — no code changes for new domains. |
| [spec-runtime-config-api.md](spec-runtime-config-api.md) | REST endpoints for runtime configuration: set API keys, assign models per agent, view memory stats. Enables A/B testing and zero-restart ops changes. |

### Infrastructure

| Spec | What it covers |
|------|---------------|
| [spec-fullstack-docker-network.md](spec-fullstack-docker-network.md) | Docker Compose topology for FastAPI backend + Next.js frontend + PostgreSQL + Redis + Ollama. Internal DNS, CORS, healthchecks, and production considerations. |

---

## How to Use These Specs

Each spec is written as an implementation guide for an AI coding assistant or developer. They are:
- **Self-contained**: each spec has enough context to implement independently
- **Generic**: no domain-specific references; applicable to any AI app
- **Opinionated**: include specific design decisions with rationale

### Recommended Reading Order

For a new AI application from scratch:

1. [spec-multiagent-pipeline.md](spec-multiagent-pipeline.md) — understand the agent pattern first
2. [spec-multi-provider-llm.md](spec-multi-provider-llm.md) — wire the LLM layer
3. [spec-database-memory-rag.md](spec-database-memory-rag.md) — add memory and RAG
4. [spec-declarative-behavior-config.md](spec-declarative-behavior-config.md) — externalize behavior
5. [spec-fullstack-docker-network.md](spec-fullstack-docker-network.md) — containerize
6. [spec-runtime-config-api.md](spec-runtime-config-api.md) — add operator controls
7. [spec-agent-observability.md](spec-agent-observability.md) — add tracing

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
