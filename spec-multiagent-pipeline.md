# Spec: Multi-Agent Pipeline Pattern

## Overview

This spec describes a sequential multi-agent pipeline for AI-powered applications that require structured, quality-controlled responses. The pattern separates concerns across specialized agents that each own a single responsibility in the response generation lifecycle.

The pipeline is appropriate when you need:
- Domain-scoped responses (chatbots, tutors, assistants with clear knowledge boundaries)
- Quality gates before delivering output to users
- Iterative refinement of generated content
- Auditability of each decision step

---

## Pipeline Architecture

```
User Input
    │
    ▼
┌─────────────────┐
│  Router Agent   │  → classifies intent, domain, mode, difficulty
└────────┬────────┘
         │
    ┌────▼────┐
    │  Input  │  → clarification request (early exit) or proceed
    └────┬────┘
         │
    ┌────▼──────────────┐
    │  Scope Guard Agent │  → validates question is within active domain
    └────────┬───────────┘
             │
        ┌────▼────┐
        │  Out of │  → redirect/fallback message (early exit) or proceed
        │  Scope? │
        └────┬────┘
             │
    ┌────────▼────────┐
    │ Generator Agent │  → produces the main response (Markdown or structured)
    └────────┬────────┘
             │
    ┌────────▼────────┐
    │ Assessor Agent  │  → scores quality, flags hallucination risk, lists issues
    └────────┬────────┘
             │
         ┌───▼───┐
         │ Score │  ≥ threshold → Final Response
         │  OK?  │  < threshold → Refiner Agent (loop up to N rounds)
         └───▼───┘
    ┌────────▼────────┐
    │ Refiner Agent   │  → rewrites based on assessor feedback
    └────────┬────────┘
             │ (back to Assessor or emit final)
             ▼
        Final Response
```

---

## Agent Definitions

### 1. Router Agent

**Responsibility**: Classify the incoming input to determine how the system should handle it.

**Inputs**:
- Raw user message
- Active domain/context
- Conversation history (optional)

**Outputs** (JSON):
```json
{
  "domain": "string",
  "category": "string",
  "mode": "explain | exercise | review | study_plan",
  "difficulty": "beginner | intermediate | advanced",
  "needs_clarification": false,
  "clarification_message": "string or null"
}
```

**Behavior**:
- If input is ambiguous, set `needs_clarification: true` and produce a clarification question
- Use temperature 0.0 for deterministic routing
- Keep prompt minimal and focused on classification only

**Prompt Template**:
```
You are a routing agent. Given the user's question and the active domain, 
classify the request.

Active domain: {domain}
User question: {question}
History (last N turns): {context}

Output ONLY valid JSON conforming to the schema. Do not explain.
```

---

### 2. Scope Guard Agent

**Responsibility**: Validate that the user's question is within the allowed scope of the active domain. Prevent the pipeline from generating off-topic responses.

**Inputs**:
- User message
- Active domain + domain keywords
- Router output (domain, category)

**Outputs** (JSON):
```json
{
  "in_scope": true,
  "confidence": 0.87,
  "matched_keywords": ["keyword1", "keyword2"],
  "action": "continue | ask_clarification | redirect_domain | general_fallback",
  "redirect_message": "string or null"
}
```

**Behavior**:
- Hybrid approach: combine keyword matching heuristics + LLM judgment
- If LLM says `in_scope` but keyword evidence is weak AND confidence < 0.75 → block anyway
- For out-of-scope: generate a polite redirect message in the user's language
- Use temperature 0.0

**Hybrid Guard Logic** (pseudocode):
```python
llm_result = call_agent_json(scope_guard_prompt)
keyword_hits = count_domain_keywords(question, domain.keywords)
evidence_weak = keyword_hits < MIN_KEYWORD_THRESHOLD

if llm_result.in_scope and evidence_weak and llm_result.confidence < 0.75:
    llm_result.in_scope = False
    llm_result.action = "redirect_domain"
```

---

### 3. Generator Agent

**Responsibility**: Generate the primary response. This is the main content-producing agent.

**Inputs**:
- User message
- Router classification (mode, difficulty, category)
- Domain configuration (guardrails, pedagogy, style)
- Session context (last N conversation turns)
- Language instruction

**Outputs**: Markdown-formatted text (or structured JSON for non-chat applications)

**Behavior**:
- Inject domain guardrails and behavioral rules into system prompt
- Inject conversation context to maintain continuity
- Use temperature 0.2–0.4 for a balance of coherence and variety
- Respect pedagogical or domain-specific configuration (e.g., require examples, require quiz)

**System Prompt Composition**:
```
{domain.system_identity}

Active guardrails:
{domain.guardrails joined as bullet list}

Respond in: {language}
Mode: {mode}
Difficulty: {difficulty}
Previous conversation context:
{session_context}
```

---

### 4. Assessor Agent

**Responsibility**: Score the generator's output for quality, correctness, and safety before delivering it to the user.

**Inputs**:
- Original question
- Generator's draft response
- Domain + mode context

**Outputs** (JSON):
```json
{
  "score": 8.5,
  "ready": true,
  "issues": ["list of identified issues"],
  "improvements": ["list of specific improvements"],
  "hallucination_risk": "low | medium | high"
}
```

**Scoring Criteria** (customize per domain):
- Technical correctness (0–2 pts)
- Domain consistency (0–2 pts)
- Clarity and didactic quality (0–2 pts)
- No hallucinations or fabricated facts (0–2 pts)
- Coverage completeness (0–2 pts)

**Behavior**:
- Set `ready: true` if `score >= min_score` (default 8)
- List concrete, actionable improvements if score is below threshold
- Use temperature 0.0 for deterministic scoring

---

### 5. Refiner Agent

**Responsibility**: Rewrite the generator's response based on assessor feedback.

**Inputs**:
- Original question
- Generator's draft response
- Assessor output (score, issues, improvements)

**Outputs**: Rewritten Markdown response (same format as Generator)

**Behavior**:
- Focus only on the listed issues and improvements
- Do not change content that was already scored well
- Use temperature 0.1 for minimal creative deviation
- Loop back to Assessor; stop when `score >= min_score` OR `rounds >= max_refine_rounds`

**Termination conditions**:
- `assessor.score >= min_score` → emit final response
- `rounds == max_refine_rounds` → emit best response seen so far

---

## Orchestration

### Entry Point

```python
async def process_request(
    question: str,
    domain: str,
    session_id: str,
    provider: str,
    model: str,
    language: str = "en",
    max_refine_rounds: int = 1,
    min_score: float = 8.0,
    per_agent_models: dict = {}
):
    # 1. Route
    route = await router_agent(question, domain, session_context)
    if route.needs_clarification:
        return clarification_response(route.clarification_message)

    # 2. Scope check
    scope = await scope_guard_agent(question, domain, route)
    if not scope.in_scope:
        return out_of_scope_response(scope.redirect_message)

    # 3. Generate
    session_ctx = await memory.get_context(session_id)
    draft = await generator_agent(question, route, domain_config, session_ctx, language)

    # 4. Assess + Refine loop
    for round in range(max_refine_rounds + 1):
        assessment = await assessor_agent(question, draft, route)
        if assessment.score >= min_score:
            break
        if round < max_refine_rounds:
            draft = await refiner_agent(question, draft, assessment)

    # 5. Persist + return
    await memory.save_turn(session_id, question, draft)
    return build_response(draft, route, assessment, trace)
```

### Per-Agent Model Overrides

Allow each agent to use a different provider/model:

```python
agent_models = {
    "router":      {"provider": "ollama", "model": "phi4"},
    "scope_guard": {"provider": "ollama", "model": "phi4"},
    "generator":   {"provider": "openai", "model": "gpt-4.1"},
    "assessor":    {"provider": "claude", "model": "claude-sonnet"},
    "refiner":     {"provider": "openai", "model": "gpt-4.1"},
}
```

This enables cost-quality tradeoffs: cheap fast models for routing/guard, powerful models for generation.

---

## Agent Trace & Observability

Every pipeline execution should produce an `agent_trace` list and a `timeline` for debugging and UX feedback.

```python
@dataclass
class AgentTraceItem:
    agent: str           # "router", "scope_guard", "generator", "assessor", "refiner"
    status: str          # "pending" | "running" | "done" | "skipped" | "error"
    summary: str         # Human-readable execution summary
    duration_ms: int
```

Timeline example:
```json
[
  "Request received.",
  "Provider resolved: openai.",
  "Router classified: domain=engineering, mode=explain, difficulty=intermediate.",
  "Scope Guard: in_scope=true, confidence=0.93.",
  "Generator produced draft (1240 chars).",
  "Assessor scored 7.2/10 — refine triggered.",
  "Refiner produced revision (round 1).",
  "Assessor scored 8.6/10 — approved.",
  "Response finalized."
]
```

Include `agent_trace` and `timeline` in every API response, behind a `debug` flag if desired.

---

## JSON Response Contract

```python
class PipelineResponse:
    answer: str                    # Final response (Markdown or plain text)
    route: RouterOut               # Classification output
    assessment: AssessorOut        # Final quality score
    agent_trace: list[TraceItem]   # Step-by-step execution log
    timeline: list[str]            # Human-readable milestones
    meta: dict                     # provider, model, session_id, language, domain, etc.
```

---

## Calling Agents with JSON Validation

Use a wrapper that retries on malformed JSON:

```python
async def call_agent_json(prompt, schema_class, provider, model, retries=2):
    for attempt in range(retries + 1):
        raw = await llm_chat(prompt, provider, model)
        try:
            data = extract_json(raw)
            return schema_class(**data)
        except (json.JSONDecodeError, ValidationError) as e:
            if attempt == retries:
                raise AgentError(f"Agent failed after {retries} retries: {e}")
```

For Ollama, pass the JSON schema natively via the `format` parameter to force valid JSON output.

---

## Implementation Checklist

- [ ] Define Pydantic schemas for each agent's output
- [ ] Implement `call_agent_json()` with retry logic
- [ ] Build prompt templates for each agent (externalize to files or config)
- [ ] Wire domain configuration into Generator's system prompt
- [ ] Implement Scope Guard hybrid (keywords + LLM)
- [ ] Build assess-refine loop with configurable threshold and max rounds
- [ ] Emit `agent_trace` and `timeline` in every response
- [ ] Support per-agent model overrides via request body or config endpoint
- [ ] Add session context injection into Generator
- [ ] Expose `/ask` endpoint that runs full pipeline

---

## Configuration Knobs

| Parameter | Default | Description |
|-----------|---------|-------------|
| `min_score` | 8.0 | Assessor score threshold to approve response |
| `max_refine_rounds` | 1 | Max Refiner iterations per request |
| `router_temperature` | 0.0 | Temperature for Router and Scope Guard |
| `generator_temperature` | 0.2 | Temperature for Generator and Refiner |
| `llm_timeout_s` | 180 | Timeout per LLM call |
| `agent_retry_count` | 2 | JSON parse retries per agent |
| `context_turns` | 5 | Number of past turns injected into Generator |
