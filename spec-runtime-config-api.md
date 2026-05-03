# Spec: Runtime Configuration API for AI Applications

## Overview

This spec describes a pattern for exposing runtime configuration of an AI application through API endpoints, allowing operators to change provider API keys, model assignments, and behavioral parameters without restarting the service. This is useful for A/B testing model quality, cost management, and operational flexibility.

---

## What Can Be Configured at Runtime

| Category | Examples |
|----------|---------|
| LLM providers | Enable/disable providers, set API keys |
| Model selection | Change which model each agent uses |
| Agent parameters | Temperature, retry count, quality threshold |
| Session parameters | Context window size, TTL |

---

## Architecture

```
┌──────────────────────────────────────────┐
│              In-Memory Config Store       │
│  _provider_keys: dict[str, str]           │
│  _agent_overrides: dict[str, AgentModel]  │
│  _runtime_params: dict[str, Any]          │
└──────────────────────────────────────────┘
          ▲                    ▲
          │                    │
   POST /config/*       GET /config/*
   (write at runtime)   (read current state)
          │
   Persists to Pydantic Settings
   (with env var fallback)
```

All state lives in memory. On restart, env vars provide the defaults (API keys, models). Runtime changes override but don't persist across restarts unless explicitly written to `.env`.

---

## Config Store Module

```python
# app/core/runtime_config.py
from threading import Lock

_lock = Lock()

# Provider API keys (override env vars)
_provider_keys: dict[str, str] = {}

# Per-agent model assignments
_agent_overrides: dict[str, dict] = {}
# Example: {"generator": {"provider": "openai", "model": "gpt-4.1"}}

# Runtime behavioral parameters
_runtime_params: dict[str, Any] = {}


def set_provider_key(provider: str, api_key: str) -> None:
    with _lock:
        _provider_keys[provider.lower()] = api_key

def get_provider_key(provider: str) -> str | None:
    return _provider_keys.get(provider.lower())

def get_effective_api_key(provider: str, env_fallback: str | None) -> str | None:
    return get_provider_key(provider) or env_fallback

def is_provider_enabled(provider: str, env_key: str | None) -> bool:
    if provider == "ollama":
        return True  # Local provider, no key needed
    return bool(get_effective_api_key(provider, env_key))


def set_agent_override(agent: str, provider: str, model: str) -> None:
    with _lock:
        _agent_overrides[agent] = {"provider": provider, "model": model}

def clear_agent_override(agent: str) -> None:
    with _lock:
        _agent_overrides.pop(agent, None)

def get_agent_overrides() -> dict:
    return dict(_agent_overrides)


def set_param(key: str, value: Any) -> None:
    with _lock:
        _runtime_params[key] = value

def get_param(key: str, default: Any = None) -> Any:
    return _runtime_params.get(key, default)
```

---

## Config API Routes

### Provider Management

```python
from fastapi import APIRouter, HTTPException
from pydantic import BaseModel

router = APIRouter(prefix="/config", tags=["config"])

class ProviderKeyRequest(BaseModel):
    provider: str
    api_key: str

class AgentOverrideRequest(BaseModel):
    agent: str        # "router", "scope_guard", "generator", "assessor", "refiner"
    provider: str
    model: str

@router.post("/providers")
async def set_provider_key(req: ProviderKeyRequest):
    provider = req.provider.lower()
    if provider not in KNOWN_PROVIDERS:
        raise HTTPException(400, f"Unknown provider: {provider}")
    set_provider_key(provider, req.api_key)
    return {"status": "ok", "provider": provider}

@router.get("/providers")
async def list_providers():
    result = {}
    for provider, info in PROVIDER_CATALOG.items():
        enabled = is_provider_enabled(provider, getattr(config, f"{provider}_api_key", None))
        result[provider] = {
            "enabled": enabled,
            "models": info["models"],
            "default_model": info["default_model"],
            "key_set": bool(get_provider_key(provider)),
        }
    return {"providers": result}
```

### Per-Agent Model Overrides

```python
@router.post("/agents")
async def set_agent_overrides(overrides: list[AgentOverrideRequest]):
    for o in overrides:
        if o.agent not in KNOWN_AGENTS:
            raise HTTPException(400, f"Unknown agent: {o.agent}")
        set_agent_override(o.agent, o.provider, o.model)
    return {"status": "ok", "overrides": get_agent_overrides()}

@router.delete("/agents/{agent_name}")
async def clear_agent_override_endpoint(agent_name: str):
    clear_agent_override(agent_name)
    return {"status": "ok", "agent": agent_name}

@router.get("/agents")
async def get_agent_override_state():
    return {"overrides": get_agent_overrides()}
```

### Memory/Cache Stats

```python
@router.get("/memory/stats")
async def memory_stats(request: Request):
    db_ok, redis_ok = False, False
    db_info, redis_info = {}, {}

    try:
        async with request.app.state.db.acquire() as conn:
            sessions = await conn.fetchval("SELECT COUNT(*) FROM app.sessions")
            conversations = await conn.fetchval("SELECT COUNT(*) FROM app.conversations")
        db_ok = True
        db_info = {"sessions": sessions, "conversations": conversations}
    except Exception as e:
        db_info = {"error": str(e)}

    try:
        info = await request.app.state.redis.info("memory")
        keys = await request.app.state.redis.dbsize()
        redis_ok = True
        redis_info = {"used_memory_human": info["used_memory_human"], "keys": keys}
    except Exception as e:
        redis_info = {"error": str(e)}

    return {
        "database": {"ok": db_ok, **db_info},
        "cache":    {"ok": redis_ok, **redis_info},
    }
```

---

## Resolving Effective Configuration

When the pipeline runs a request, it resolves the effective provider/model using this priority:

```
Request body (explicit)
    → Per-agent override (from /config/agents)
        → Config default (from env vars / settings)
```

```python
def resolve_model(
    request_provider: str | None,
    request_model: str | None,
    agent_name: str | None,
) -> tuple[str, str]:
    # 1. Per-agent override
    if agent_name:
        overrides = get_agent_overrides()
        if agent_name in overrides:
            o = overrides[agent_name]
            return o["provider"], o["model"]

    # 2. Request-level
    if request_provider and request_model:
        return request_provider, request_model

    # 3. Config default
    return config.default_provider, config.default_model
```

---

## Local LLM (Ollama) Management

If Ollama is used as the local provider, expose endpoints to manage models:

```python
@router.post("/ollama/pull")
async def pull_model(model: str):
    async with httpx.AsyncClient(timeout=300) as client:
        resp = await client.post(
            f"{OLLAMA_BASE_URL}/api/pull",
            json={"name": model, "stream": False}
        )
        resp.raise_for_status()
    return {"status": "pulled", "model": model}

@router.get("/ollama/models")
async def list_local_models():
    async with httpx.AsyncClient(timeout=10) as client:
        resp = await client.get(f"{OLLAMA_BASE_URL}/api/tags")
        resp.raise_for_status()
    models = [m["name"] for m in resp.json().get("models", [])]
    return {"models": models, "count": len(models)}
```

---

## Frontend: Admin Configuration Page

The configuration API enables a simple admin UI with:

1. **Provider status panel**: shows each provider as enabled/disabled, with a form to paste in an API key
2. **Model assignment panel**: dropdowns to assign provider + model to each agent
3. **Memory stats panel**: shows Redis keys, PostgreSQL record counts, connection health
4. **Ollama model panel**: list local models, pull new models by name

None of this requires a restart — changes take effect on the next request.

---

## Security Considerations

- API keys sent to `/config/providers` are stored in memory only (not logged, not returned)
- `GET /config/providers` should never return the actual key values — only whether they are set
- In production, protect `/config/*` endpoints with authentication middleware
- Consider rate limiting the `/ollama/pull` endpoint (model downloads are large)

```python
# Example: simple token-based protection
from fastapi import Header, HTTPException

async def verify_admin_token(x_admin_token: str = Header(...)):
    if x_admin_token != config.admin_token:
        raise HTTPException(403, "Forbidden")

# Apply to all config routes:
router = APIRouter(prefix="/config", dependencies=[Depends(verify_admin_token)])
```

---

## Implementation Checklist

- [ ] Create `runtime_config.py` module with thread-safe in-memory stores
- [ ] Implement `set/get_provider_key()`, `is_provider_enabled()`
- [ ] Implement `set/get/clear_agent_override()`
- [ ] Build `POST /config/providers` and `GET /config/providers` endpoints
- [ ] Build `POST /config/agents`, `GET /config/agents`, `DELETE /config/agents/{name}` endpoints
- [ ] Build `GET /config/memory/stats` for database + cache health
- [ ] If using Ollama: build `POST /config/ollama/pull` and `GET /config/ollama/models`
- [ ] Implement `resolve_model()` with 3-level priority (request → override → default)
- [ ] Protect `/config/*` routes with auth middleware in production
- [ ] Ensure `GET /providers` never returns actual API key values
