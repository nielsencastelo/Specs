# Spec: Multi-Provider LLM Abstraction Layer

## Overview

This spec describes a provider abstraction layer that allows an AI application to switch between multiple LLM providers (local and cloud) through a single unified interface. Providers can be changed at runtime without restarting the application, and individual agents can use different providers for cost/quality optimization.

Supported providers in this pattern:
- **Ollama** (local self-hosted, no cost, privacy-preserving)
- **OpenAI** (gpt-4.1, gpt-4o-mini, etc.)
- **Anthropic / Claude** (claude-sonnet, claude-haiku, etc.)
- **Google Gemini** (gemini-2.5-flash, gemini-2.5-pro)

---

## Architecture

```
Application / Agent
        │
        │  llm_chat(prompt, provider, model, temperature)
        ▼
┌─────────────────────────────┐
│   LLM Client (Unified API)  │
│  resolve_model_selection()  │
│  llm_chat()                 │
└────────────┬────────────────┘
             │
    ┌────────┼────────┬──────────┐
    ▼        ▼        ▼          ▼
 Ollama   OpenAI  Anthropic  Gemini
(local)  (cloud)  (cloud)   (cloud)
```

---

## Provider Catalog

Define all providers and their models in a configuration object:

```python
PROVIDER_CATALOG = {
    "ollama": {
        "base_url": "http://localhost:11434",
        "models": ["phi4", "llama3.2", "mistral", "qwen2.5"],
        "default_model": "phi4",
        "requires_api_key": False,
    },
    "openai": {
        "base_url": "https://api.openai.com/v1",
        "models": ["gpt-4.1-mini", "gpt-4.1", "gpt-4o-mini"],
        "default_model": "gpt-4.1-mini",
        "requires_api_key": True,
    },
    "claude": {
        "base_url": "https://api.anthropic.com",
        "models": ["claude-sonnet-4-6", "claude-haiku-4-5"],
        "default_model": "claude-sonnet-4-6",
        "requires_api_key": True,
    },
    "gemini": {
        "base_url": "https://generativelanguage.googleapis.com",
        "models": ["gemini-2.5-flash", "gemini-2.5-pro"],
        "default_model": "gemini-2.5-flash",
        "requires_api_key": True,
    },
}
```

---

## Configuration

### Settings (Pydantic)

```python
from pydantic_settings import BaseSettings

class LLMConfig(BaseSettings):
    # Default provider + model
    default_provider: str = "ollama"
    default_model: str = "phi4"
    default_temperature: float = 0.2

    # API keys (loaded from env or set at runtime)
    openai_api_key: str | None = None
    anthropic_api_key: str | None = None
    gemini_api_key: str | None = None

    # Timeouts
    llm_timeout_s: int = 180

    class Config:
        env_file = ".env"
```

### Runtime API Key Updates

Expose an endpoint to set provider keys without restart:

```python
# In-memory runtime overrides
_runtime_keys: dict[str, str] = {}

def set_provider_key(provider: str, api_key: str) -> None:
    _runtime_keys[provider] = api_key

def get_api_key(provider: str) -> str | None:
    return _runtime_keys.get(provider) or getattr(config, f"{provider}_api_key", None)
```

---

## Unified LLM Interface

### Main Function

```python
async def llm_chat(
    prompt: str,
    provider: str,
    model: str,
    temperature: float = 0.2,
    system_prompt: str | None = None,
    json_schema: dict | None = None,   # For Ollama native JSON mode
    max_tokens: int = 4096,
    timeout_s: int = 180,
) -> str:
    """
    Unified LLM chat call. Returns the text content of the response.
    Raises LLMError on failure.
    """
    provider = provider.lower()

    if provider == "ollama":
        return await _call_ollama(prompt, model, temperature, system_prompt, json_schema, timeout_s)
    elif provider == "openai":
        return await _call_openai(prompt, model, temperature, system_prompt, max_tokens, timeout_s)
    elif provider == "claude":
        return await _call_claude(prompt, model, temperature, system_prompt, max_tokens, timeout_s)
    elif provider == "gemini":
        return await _call_gemini(prompt, model, temperature, system_prompt, max_tokens, timeout_s)
    else:
        raise LLMError(f"Unknown provider: {provider}")
```

---

## Provider Implementations

### Ollama (Local)

```python
async def _call_ollama(prompt, model, temperature, system_prompt, json_schema, timeout_s):
    url = f"{OLLAMA_BASE_URL}/api/chat"

    messages = []
    if system_prompt:
        messages.append({"role": "system", "content": system_prompt})
    messages.append({"role": "user", "content": prompt})

    body = {
        "model": model,
        "messages": messages,
        "stream": False,
        "options": {"temperature": temperature},
    }

    # Native JSON mode: forces structured output
    if json_schema:
        body["format"] = json_schema

    async with httpx.AsyncClient(timeout=timeout_s) as client:
        resp = await client.post(url, json=body)
        resp.raise_for_status()
        return resp.json()["message"]["content"]
```

**Ollama advantage**: pass the Pydantic JSON schema via `format` to get guaranteed valid JSON output without extra prompting.

### OpenAI

```python
async def _call_openai(prompt, model, temperature, system_prompt, max_tokens, timeout_s):
    api_key = get_api_key("openai")
    if not api_key:
        raise LLMError("OpenAI API key not configured")

    messages = []
    if system_prompt:
        messages.append({"role": "system", "content": system_prompt})
    messages.append({"role": "user", "content": prompt})

    async with httpx.AsyncClient(timeout=timeout_s) as client:
        resp = await client.post(
            "https://api.openai.com/v1/chat/completions",
            headers={"Authorization": f"Bearer {api_key}"},
            json={
                "model": model,
                "messages": messages,
                "temperature": temperature,
                "max_tokens": max_tokens,
            },
        )
        resp.raise_for_status()
        data = resp.json()
        return _extract_openai_text(data["choices"][0]["message"]["content"])

def _extract_openai_text(content) -> str:
    # Handle both string and list-of-blocks (vision responses)
    if isinstance(content, str):
        return content
    if isinstance(content, list):
        return " ".join(block.get("text", "") for block in content if block.get("type") == "text")
    return str(content)
```

### Anthropic / Claude

```python
async def _call_claude(prompt, model, temperature, system_prompt, max_tokens, timeout_s):
    api_key = get_api_key("claude")
    if not api_key:
        raise LLMError("Anthropic API key not configured")

    body = {
        "model": model,
        "max_tokens": max_tokens,
        "temperature": temperature,
        "messages": [{"role": "user", "content": prompt}],
    }

    # Claude requires system as top-level key, not inside messages
    if system_prompt:
        body["system"] = system_prompt

    async with httpx.AsyncClient(timeout=timeout_s) as client:
        resp = await client.post(
            "https://api.anthropic.com/v1/messages",
            headers={
                "x-api-key": api_key,
                "anthropic-version": "2023-06-01",
                "content-type": "application/json",
            },
            json=body,
        )
        resp.raise_for_status()
        return resp.json()["content"][0]["text"]
```

### Google Gemini

```python
async def _call_gemini(prompt, model, temperature, system_prompt, max_tokens, timeout_s):
    api_key = get_api_key("gemini")
    if not api_key:
        raise LLMError("Gemini API key not configured")

    # Gemini: combine system + user into single contents array
    combined = f"{system_prompt}\n\n{prompt}" if system_prompt else prompt

    url = f"https://generativelanguage.googleapis.com/v1beta/models/{model}:generateContent"

    async with httpx.AsyncClient(timeout=timeout_s) as client:
        resp = await client.post(
            url,
            params={"key": api_key},
            json={
                "contents": [{"parts": [{"text": combined}]}],
                "generationConfig": {
                    "temperature": temperature,
                    "maxOutputTokens": max_tokens,
                },
            },
        )
        resp.raise_for_status()
        return resp.json()["candidates"][0]["content"]["parts"][0]["text"]
```

---

## Model Resolution

Resolve provider + model from request, config, or per-agent overrides:

```python
def resolve_model_selection(
    request_provider: str | None,
    request_model: str | None,
    agent_name: str | None,
    agent_overrides: dict,      # {"router": {"provider": "ollama", "model": "phi4"}, ...}
    config: LLMConfig,
) -> tuple[str, str]:
    """
    Priority order:
    1. Per-agent override (if agent_name is set)
    2. Request-level override
    3. Config default
    """
    if agent_name and agent_name in agent_overrides:
        ov = agent_overrides[agent_name]
        return ov["provider"], ov["model"]

    if request_provider and request_model:
        return request_provider, request_model

    return config.default_provider, config.default_model
```

---

## Provider Status Endpoint

```python
@router.get("/providers")
async def list_providers() -> dict:
    catalog = {}
    for name, info in PROVIDER_CATALOG.items():
        has_key = bool(get_api_key(name)) if info["requires_api_key"] else True
        catalog[name] = {
            "enabled": has_key or name == "ollama",
            "models": info["models"],
            "default_model": info["default_model"],
            "requires_api_key": info["requires_api_key"],
        }
    return {"providers": catalog}

@router.post("/providers")
async def set_provider_key_endpoint(provider: str, api_key: str) -> dict:
    if provider not in PROVIDER_CATALOG:
        raise HTTPException(400, f"Unknown provider: {provider}")
    set_provider_key(provider, api_key)
    return {"status": "ok", "provider": provider}
```

---

## Ollama Model Management

If Ollama is used as the local fallback provider, expose model pull and list endpoints:

```python
@router.post("/ollama/pull")
async def pull_ollama_model(model: str) -> dict:
    async with httpx.AsyncClient(timeout=300) as client:
        resp = await client.post(
            f"{OLLAMA_BASE_URL}/api/pull",
            json={"name": model, "stream": False}
        )
        resp.raise_for_status()
    return {"status": "pulled", "model": model}

@router.get("/ollama/models")
async def list_ollama_models() -> dict:
    async with httpx.AsyncClient(timeout=10) as client:
        resp = await client.get(f"{OLLAMA_BASE_URL}/api/tags")
        resp.raise_for_status()
    return {"models": [m["name"] for m in resp.json().get("models", [])]}
```

---

## Error Handling

```python
class LLMError(Exception):
    pass

# In llm_chat():
try:
    return await _call_provider(...)
except httpx.TimeoutException:
    raise LLMError(f"Provider {provider} timed out after {timeout_s}s")
except httpx.HTTPStatusError as e:
    raise LLMError(f"Provider {provider} returned HTTP {e.response.status_code}: {e.response.text[:200]}")
except Exception as e:
    raise LLMError(f"Provider {provider} error: {str(e)}")
```

---

## Implementation Checklist

- [ ] Define `PROVIDER_CATALOG` with all providers, their models, and base URLs
- [ ] Implement `LLMConfig` with Pydantic Settings for env-based API keys
- [ ] Implement `get_api_key()` with runtime override support
- [ ] Implement `llm_chat()` dispatcher function
- [ ] Implement provider-specific `_call_*` functions (Ollama, OpenAI, Claude, Gemini)
- [ ] Handle OpenAI vision content arrays in `_extract_openai_text()`
- [ ] Implement `resolve_model_selection()` with priority order
- [ ] Expose `GET /providers` and `POST /providers` endpoints
- [ ] Expose Ollama `pull` and `list` endpoints (if Ollama is used)
- [ ] Add `LLMError` with timeout and HTTP error wrapping
- [ ] Test each provider integration independently before wiring to agents

---

## Environment Variables

```env
DEFAULT_PROVIDER=ollama
DEFAULT_MODEL=phi4
DEFAULT_TEMPERATURE=0.2

OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
GEMINI_API_KEY=AIza...

OLLAMA_BASE_URL=http://localhost:11434
LLM_TIMEOUT_S=180
```
