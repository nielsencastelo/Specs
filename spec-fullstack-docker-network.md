# Spec: Full-Stack Docker Network for AI Applications

## Overview

This spec describes the containerized network topology for a full-stack AI application that includes a Python backend (FastAPI), a Node.js frontend (Next.js), a PostgreSQL database with pgvector, a Redis cache, and an optional local LLM (Ollama). All services communicate through an isolated Docker network.

---

## Service Topology

```
                         ┌──────────────────────────────────────────────┐
                         │               Docker Network (bridge)         │
                         │                                               │
  External               │  ┌──────────┐        ┌──────────────────┐   │
  Browser ─── :3000 ─────►  │ Frontend │        │     Backend      │   │
                         │  │ Next.js  │──:8000─►│    FastAPI       │   │
                         │  └──────────┘        └─────────┬────────┘   │
                         │                                │             │
                         │              ┌─────────────────┼──────────┐  │
                         │              │                 │          │  │
                         │         ┌────▼────┐    ┌───────▼────┐    │  │
                         │         │  Redis  │    │ PostgreSQL │    │  │
                         │         │  :6379  │    │  + pgvector│    │  │
                         │         └─────────┘    └────────────┘    │  │
                         │                                           │  │
                         │         ┌──────────────────────────────┐  │  │
                         │         │  Ollama (optional)   :11434  │  │  │
                         │         └──────────────────────────────┘  │  │
                         └──────────────────────────────────────────────┘

Exposed ports (host):
  3000 → Frontend
  8000 → Backend API (dev only; proxy through frontend in production)
```

---

## Docker Compose Configuration

```yaml
version: "3.9"

services:

  # ── PostgreSQL + pgvector ─────────────────────────────────────────────────
  postgres:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_DB: ${POSTGRES_DB:-appdb}
      POSTGRES_USER: ${POSTGRES_USER:-appuser}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-apppass}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./pgvector/init:/docker-entrypoint-initdb.d   # SQL init scripts run once
    networks:
      - app_network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-appuser} -d ${POSTGRES_DB:-appdb}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 20s

  # ── Redis ─────────────────────────────────────────────────────────────────
  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
    networks:
      - app_network
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

  # ── Backend (FastAPI) ─────────────────────────────────────────────────────
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    environment:
      DATABASE_URL: postgresql://appuser:apppass@postgres:5432/appdb
      REDIS_URL: redis://redis:6379/0
      OLLAMA_BASE_URL: http://ollama:11434
      DEFAULT_PROVIDER: ${DEFAULT_PROVIDER:-ollama}
      DEFAULT_MODEL: ${DEFAULT_MODEL:-phi4}
      OPENAI_API_KEY: ${OPENAI_API_KEY:-}
      ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY:-}
      GEMINI_API_KEY: ${GEMINI_API_KEY:-}
    ports:
      - "8000:8000"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - app_network
    restart: unless-stopped

  # ── Frontend (Next.js) ────────────────────────────────────────────────────
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    environment:
      NEXT_PUBLIC_API_BASE_URL: http://localhost:8000    # browser-side (host)
      API_BASE_URL: http://backend:8000                  # server-side (container)
    ports:
      - "3000:3000"
    depends_on:
      - backend
    networks:
      - app_network
    restart: unless-stopped

  # ── Ollama (optional local LLM) ───────────────────────────────────────────
  ollama:
    image: ollama/ollama:latest
    volumes:
      - ollama_data:/root/.ollama
    networks:
      - app_network
    restart: unless-stopped
    # GPU support (uncomment if NVIDIA GPU available):
    # deploy:
    #   resources:
    #     reservations:
    #       devices:
    #         - driver: nvidia
    #           count: 1
    #           capabilities: [gpu]

networks:
  app_network:
    driver: bridge

volumes:
  postgres_data:
  ollama_data:
```

---

## Backend Dockerfile

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# Install system deps for psycopg2
RUN apt-get update && apt-get install -y \
    libpq-dev gcc \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

For development with auto-reload:
```dockerfile
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]
```

---

## Frontend Dockerfile

```dockerfile
FROM node:20-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production

COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public

EXPOSE 3000
CMD ["node", "server.js"]
```

### next.config.js for standalone output

```js
/** @type {import('next').NextConfig} */
const nextConfig = {
  output: "standalone",
};
module.exports = nextConfig;
```

---

## FastAPI CORS Configuration

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "http://localhost:3000",
        "http://frontend:3000",
        # Add your production domain here
    ],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

In production, restrict `allow_origins` to specific domains only.

---

## Environment Variables (.env)

```env
# Database
POSTGRES_DB=appdb
POSTGRES_USER=appuser
POSTGRES_PASSWORD=change_me_in_production

# LLM Providers
DEFAULT_PROVIDER=ollama
DEFAULT_MODEL=phi4
OPENAI_API_KEY=
ANTHROPIC_API_KEY=
GEMINI_API_KEY=

# Application
ENVIRONMENT=development
SECRET_KEY=change_me_in_production
```

Never commit `.env` with real secrets. Use `.env.example` with placeholder values.

---

## Service Communication (Internal DNS)

Within the Docker network, services communicate by service name:

| From → To | URL |
|-----------|-----|
| Backend → PostgreSQL | `postgresql://appuser:apppass@postgres:5432/appdb` |
| Backend → Redis | `redis://redis:6379/0` |
| Backend → Ollama | `http://ollama:11434` |
| Frontend (server) → Backend | `http://backend:8000` |
| Frontend (browser) → Backend | `http://localhost:8000` |

The frontend has two URLs:
- `API_BASE_URL`: used in Next.js server-side code and API routes (container DNS)
- `NEXT_PUBLIC_API_BASE_URL`: used in browser JS (must be reachable from the user's machine)

---

## FastAPI Application Lifespan

```python
from contextlib import asynccontextmanager
import asyncpg
import redis.asyncio as aioredis

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: establish connection pools
    app.state.db = await asyncpg.create_pool(
        DATABASE_URL,
        min_size=2,
        max_size=10,
        command_timeout=60
    )
    app.state.redis = aioredis.from_url(REDIS_URL, decode_responses=True)

    yield

    # Shutdown: clean up
    await app.state.db.close()
    await app.state.redis.close()

app = FastAPI(lifespan=lifespan)
```

---

## Health Check Endpoint

```python
@app.get("/health")
async def health_check(request: Request):
    db_ok = False
    redis_ok = False

    try:
        async with request.app.state.db.acquire() as conn:
            await conn.fetchval("SELECT 1")
        db_ok = True
    except Exception:
        pass

    try:
        await request.app.state.redis.ping()
        redis_ok = True
    except Exception:
        pass

    return {
        "status": "ok" if db_ok else "degraded",
        "database": "ok" if db_ok else "unavailable",
        "cache": "ok" if redis_ok else "unavailable",
    }
```

---

## Development Workflow

### Start all services

```bash
docker compose up -d
```

### View logs

```bash
docker compose logs -f backend
docker compose logs -f frontend
```

### Rebuild after code changes

```bash
docker compose up -d --build backend
```

### Pull Ollama model after first start

```bash
docker compose exec ollama ollama pull phi4
# Or via the backend API:
curl -X POST http://localhost:8000/config/ollama/pull -d '{"model": "phi4"}'
```

### Reset database (destroys all data)

```bash
docker compose down -v postgres
docker compose up -d postgres
```

---

## Production Considerations

### Secrets Management
- Use Docker secrets or a secrets manager (Vault, AWS Secrets Manager) instead of env vars
- Rotate database passwords using pgBouncer or a connection pooler

### Reverse Proxy
- Add an Nginx or Traefik container in front of frontend and backend
- Terminate TLS at the proxy layer
- Route `/api/` to the backend, `/` to the frontend (no CORS issues)

### Scaling
- Backend is stateless (state in Redis/PostgreSQL) — can run multiple replicas
- Use `deploy.replicas` in Compose or migrate to Kubernetes for horizontal scaling

### Persistent Volumes
- Back up `postgres_data` and `ollama_data` volumes regularly
- Consider mounting external block storage for production databases

---

## Implementation Checklist

- [ ] Create `docker-compose.yml` with postgres, redis, backend, frontend services
- [ ] Add healthchecks to postgres and redis; use `condition: service_healthy` in depends_on
- [ ] Configure `app_network` bridge network; remove all direct `ports` except exposed services
- [ ] Implement FastAPI `lifespan` for pool creation/cleanup
- [ ] Configure CORS in FastAPI with explicit origins (not `*` in production)
- [ ] Use internal service DNS (postgres, redis) in backend env vars
- [ ] Use `NEXT_PUBLIC_API_BASE_URL` (browser) and `API_BASE_URL` (server) in frontend
- [ ] Add `GET /health` endpoint that checks database + cache
- [ ] Create `.env.example` with all required variables (no real values)
- [ ] Add `.env` to `.gitignore`
- [ ] Build frontend with `output: "standalone"` for optimized Docker image
