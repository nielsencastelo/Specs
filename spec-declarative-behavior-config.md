# Spec: Declarative Behavior Configuration via YAML

## Overview

This spec describes a pattern for externalizing AI agent behavior into YAML configuration files, rather than hardcoding domain logic, guardrails, or pedagogical rules into agent code. Each configuration file defines a complete behavioral profile for a domain or mode of operation.

This approach enables:
- Adding or modifying agent behavior without code changes
- Non-developers can tune agent behavior through YAML edits
- Multiple domains or personas from a single codebase
- Runtime hot-reloading or domain-switching per session

---

## Configuration File Structure

Each YAML file defines a single domain or behavioral context.

### Full Schema

```yaml
# ─── Identity ─────────────────────────────────────────────────────────────────
domain: string                      # Machine-readable unique identifier (snake_case)
display_name: string                # Human-readable label

system_identity: |                  # System prompt preamble injected into all agents
  You are a [role description].
  [Tone, behavior, and identity instructions.]

# ─── Scope ────────────────────────────────────────────────────────────────────
disciplines:
  - id: string                      # Machine-readable category ID
    name: string                    # Display name
    keywords:                       # Keywords used by Scope Guard hybrid heuristic
      - word1
      - word2

router_rules:
  fallback_discipline: string       # Default category when no match found
  allow_clarification: bool         # Whether router can ask clarifying questions

# ─── Pedagogy / Content Rules ─────────────────────────────────────────────────
pedagogy:
  default_style: string             # e.g. "didactic", "socratic", "concise"
  require_examples: bool            # Generator must include concrete examples
  require_quiz: bool                # Generator must include a mini-quiz
  require_next_steps: bool          # Generator must include next steps / recommendations
  difficulty_levels:                # Allowed difficulty levels
    - beginner
    - intermediate
    - advanced

# ─── Safety & Guardrails ──────────────────────────────────────────────────────
guardrails:
  - "Do not fabricate references or URLs."
  - "If uncertain, say so instead of guessing."
  - "Stay within the active domain. Do not answer off-topic questions."

# ─── Optional Extensions ──────────────────────────────────────────────────────
meta:
  version: "1.0"
  tags: [education, stem, adults]
  target_audience: string
```

### Minimal Example

```yaml
domain: software_engineering
display_name: Software Engineering

system_identity: |
  You are a software engineering tutor. Explain concepts clearly,
  use code examples where relevant, and tailor depth to the student's level.

disciplines:
  - id: backend
    name: Backend Development
    keywords: [api, server, database, rest, http, fastapi, django, express]
  - id: frontend
    name: Frontend Development
    keywords: [react, html, css, javascript, component, dom, typescript]
  - id: devops
    name: DevOps & Infrastructure
    keywords: [docker, kubernetes, ci, cd, deployment, nginx, terraform]

router_rules:
  fallback_discipline: general
  allow_clarification: true

pedagogy:
  default_style: didactic
  require_examples: true
  require_quiz: false
  require_next_steps: true

guardrails:
  - "Do not recommend deprecated or insecure practices."
  - "Always show complete, runnable code snippets."
  - "Do not pretend to have access to the student's codebase."
```

---

## Domain Loader

### Loading & Caching

```python
from pathlib import Path
from functools import lru_cache
import yaml
from pydantic import BaseModel

DOMAINS_DIR = Path("domains/")

class DisciplineConfig(BaseModel):
    id: str
    name: str
    keywords: list[str] = []

class PedagogyConfig(BaseModel):
    default_style: str = "didactic"
    require_examples: bool = True
    require_quiz: bool = False
    require_next_steps: bool = True

class RouterRules(BaseModel):
    fallback_discipline: str = "general"
    allow_clarification: bool = True

class DomainConfig(BaseModel):
    domain: str
    display_name: str
    system_identity: str
    disciplines: list[DisciplineConfig] = []
    router_rules: RouterRules = RouterRules()
    pedagogy: PedagogyConfig = PedagogyConfig()
    guardrails: list[str] = []

@lru_cache(maxsize=32)
def load_domain(domain_name: str) -> DomainConfig:
    # Normalize: spaces and hyphens → underscores
    key = domain_name.strip().lower().replace(" ", "_").replace("-", "_")

    path = DOMAINS_DIR / f"{key}.yaml"
    if not path.exists():
        raise ValueError(f"Domain config not found: {key}")

    with open(path) as f:
        data = yaml.safe_load(f)

    config = DomainConfig(**data)

    # Ensure fallback discipline exists
    discipline_ids = {d.id for d in config.disciplines}
    fallback = config.router_rules.fallback_discipline
    if fallback not in discipline_ids:
        config.disciplines.append(DisciplineConfig(id=fallback, name="General", keywords=[]))

    return config

def list_domains() -> list[dict]:
    configs = []
    for path in DOMAINS_DIR.glob("*.yaml"):
        try:
            cfg = load_domain(path.stem)
            configs.append({"domain": cfg.domain, "display_name": cfg.display_name})
        except Exception:
            pass
    return configs
```

### Cache Invalidation (for hot-reload)

```python
def reload_domain(domain_name: str) -> DomainConfig:
    load_domain.cache_clear()
    return load_domain(domain_name)
```

---

## Injecting Config into Agent Prompts

### System Prompt Composition

```python
def build_system_prompt(domain: DomainConfig, mode: str, difficulty: str) -> str:
    guardrails_text = "\n".join(f"- {g}" for g in domain.guardrails)

    pedagogy_instructions = []
    if domain.pedagogy.require_examples:
        pedagogy_instructions.append("Include at least one concrete example.")
    if domain.pedagogy.require_quiz:
        pedagogy_instructions.append("Include a short 2–3 question quiz at the end.")
    if domain.pedagogy.require_next_steps:
        pedagogy_instructions.append("Suggest 2–3 next steps or resources.")
    pedagogy_text = "\n".join(f"- {p}" for p in pedagogy_instructions)

    return f"""{domain.system_identity}

Current mode: {mode}
Target difficulty: {difficulty}

Content requirements:
{pedagogy_text}

Guardrails (always follow):
{guardrails_text}
"""
```

### Scope Guard Keyword Extraction

```python
def get_domain_keywords(domain: DomainConfig) -> set[str]:
    keywords = set()
    for discipline in domain.disciplines:
        keywords.update(k.lower() for k in discipline.keywords)
    return keywords

def keyword_match_score(question: str, domain: DomainConfig) -> tuple[int, list[str]]:
    keywords = get_domain_keywords(domain)
    words = set(question.lower().split())
    matched = list(words & keywords)
    return len(matched), matched
```

---

## Router Prompt Using Domain Config

```python
def build_router_prompt(question: str, domain: DomainConfig, context: str) -> str:
    disciplines_list = "\n".join(
        f"  - {d.id}: {d.name} (keywords: {', '.join(d.keywords[:5])})"
        for d in domain.disciplines
    )
    allow_clarification = domain.router_rules.allow_clarification

    return f"""You are a routing agent for the domain: {domain.display_name}.

Available disciplines:
{disciplines_list}

Fallback discipline: {domain.router_rules.fallback_discipline}
Can ask for clarification: {allow_clarification}

Conversation context:
{context or "None"}

User question: {question}

Classify the question. Output ONLY valid JSON:
{{
  "domain": "{domain.domain}",
  "discipline": "<discipline id from the list above>",
  "mode": "<explain|exercise|review|study_plan>",
  "difficulty": "<beginner|intermediate|advanced>",
  "needs_clarification": false,
  "clarification_message": null
}}"""
```

---

## Directory Structure

```
domains/
├── software_engineering.yaml
├── data_science.yaml
├── legal.yaml
├── healthcare.yaml
├── finance.yaml
└── general_assistant.yaml

teaching_methods/           # Optional: separate method configs
├── socratic.yaml
└── project_based.yaml
```

Each file is independent. The application loads them on demand and caches the result.

---

## Teaching Method Config (Optional Extension)

For applications that separate *what to teach* (domain) from *how to teach* (method):

```yaml
# teaching_methods/socratic.yaml
method: socratic
display_name: Socratic Method

style_instructions: |
  Guide the student to discover answers through questions rather than
  explaining directly. Ask leading questions. Validate reasoning, not just answers.

prompt_modifiers:
  - "Instead of explaining, ask questions that lead the student to the answer."
  - "After each student response, validate their reasoning before continuing."
  - "End every response with a question that advances understanding."
```

Merge into the Generator's system prompt alongside the domain config:

```python
method_config = load_teaching_method(session.method)
system_prompt = build_system_prompt(domain_config, mode, difficulty)
if method_config:
    system_prompt += "\n\nTeaching style:\n" + "\n".join(
        f"- {m}" for m in method_config.prompt_modifiers
    )
```

---

## API Endpoints

```python
@router.get("/domains")
async def list_available_domains() -> dict:
    return {"domains": list_domains()}

@router.get("/domains/{domain_name}")
async def get_domain_config(domain_name: str) -> dict:
    cfg = load_domain(domain_name)
    return {
        "domain": cfg.domain,
        "display_name": cfg.display_name,
        "disciplines": [{"id": d.id, "name": d.name} for d in cfg.disciplines],
        "pedagogy": cfg.pedagogy.model_dump(),
    }
```

---

## Implementation Checklist

- [ ] Define YAML schema with domain, disciplines, pedagogy, guardrails fields
- [ ] Implement `DomainConfig` Pydantic model with sensible defaults
- [ ] Implement `load_domain()` with `@lru_cache` and name normalization
- [ ] Auto-inject fallback discipline if not present in config
- [ ] Implement `build_system_prompt()` that compiles guardrails + pedagogy instructions
- [ ] Implement `keyword_match_score()` for Scope Guard heuristic
- [ ] Implement `build_router_prompt()` that includes discipline list from config
- [ ] Add `GET /domains` and `GET /domains/{name}` API endpoints
- [ ] Store YAML files in a `domains/` directory outside application code
- [ ] Document YAML schema for domain authors (non-developers)

---

## Benefits of This Pattern

| Hardcoded logic | YAML config |
|-----------------|-------------|
| Domain change = code PR | Domain change = YAML edit |
| Single domain per deployment | Many domains, same codebase |
| Developer needed for guardrail updates | Domain expert can edit directly |
| Hard to A/B test pedagogy | Swap YAML, no restart needed |
| Guardrails drift into agent prompts | All guardrails centralized, auditable |
