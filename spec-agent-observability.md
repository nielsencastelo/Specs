# Spec: Agent Observability & Execution Tracing

## Overview

This spec describes a lightweight observability pattern for multi-agent pipelines that produces structured execution traces, human-readable timelines, and quality metadata on every request — without requiring external observability infrastructure.

The goal is to make every agent execution fully auditable in the API response, enabling debugging, UX feedback loops, and quality monitoring without separate tooling.

---

## Core Data Structures

### Agent Trace Item

```python
from enum import Enum
from pydantic import BaseModel

class AgentStatus(str, Enum):
    pending  = "pending"
    running  = "running"
    done     = "done"
    skipped  = "skipped"
    error    = "error"

class AgentTraceItem(BaseModel):
    agent:       str          # "router", "scope_guard", "generator", "assessor", "refiner"
    title:       str          # Human-readable agent name
    status:      AgentStatus
    summary:     str          # What this agent did or decided
    duration_ms: int = 0      # Execution time in milliseconds
    round:       int = 0      # For refiner: which iteration (0 = first)
```

### Pipeline Response with Observability

```python
class PipelineResponse(BaseModel):
    # Core output
    answer: str

    # Observability
    agent_trace: list[AgentTraceItem]
    timeline: list[str]
    meta: dict

    # Agent outputs (optional, behind debug flag)
    route_debug:  dict | None = None
    scope_debug:  dict | None = None
    assess_debug: dict | None = None
```

---

## Trace Builder

Use a `TraceBuilder` helper to collect trace items during execution:

```python
import time

class TraceBuilder:
    def __init__(self, agent_definitions: list[dict]):
        """
        agent_definitions: [{"agent": "router", "title": "Router"}, ...]
        """
        self._trace: list[AgentTraceItem] = [
            AgentTraceItem(agent=a["agent"], title=a["title"], status=AgentStatus.pending, summary="")
            for a in agent_definitions
        ]
        self._timeline: list[str] = []
        self._start: dict[str, float] = {}

    def start(self, agent: str) -> None:
        self._start[agent] = time.time()
        self._set_status(agent, AgentStatus.running)

    def done(self, agent: str, summary: str, round: int = 0) -> None:
        elapsed_ms = int((time.time() - self._start.get(agent, time.time())) * 1000)
        self._set(agent, AgentStatus.done, summary, elapsed_ms, round)

    def skipped(self, agent: str, reason: str) -> None:
        self._set(agent, AgentStatus.skipped, reason)

    def error(self, agent: str, message: str) -> None:
        self._set(agent, AgentStatus.error, message)

    def event(self, message: str) -> None:
        self._timeline.append(message)

    def build(self) -> tuple[list[AgentTraceItem], list[str]]:
        return self._trace, self._timeline

    def _set_status(self, agent: str, status: AgentStatus) -> None:
        for item in self._trace:
            if item.agent == agent:
                item.status = status

    def _set(self, agent: str, status: AgentStatus, summary: str,
             duration_ms: int = 0, round: int = 0) -> None:
        for item in self._trace:
            if item.agent == agent:
                item.status = status
                item.summary = summary
                item.duration_ms = duration_ms
                item.round = round
```

---

## Usage in Pipeline Orchestrator

```python
AGENTS = [
    {"agent": "router",      "title": "Router"},
    {"agent": "scope_guard", "title": "Scope Guard"},
    {"agent": "generator",   "title": "Generator"},
    {"agent": "assessor",    "title": "Assessor"},
    {"agent": "refiner",     "title": "Refiner"},
]

async def run_pipeline(question, domain, session_id, debug=False):
    trace = TraceBuilder(AGENTS)
    t0 = time.time()

    trace.event("Request received.")

    # ── Router ────────────────────────────────────────────────────────────
    trace.start("router")
    route = await router_agent(question, domain)
    trace.done("router", f"Classified: mode={route.mode}, discipline={route.discipline}")
    trace.event(f"Router: mode={route.mode}, discipline={route.discipline}, difficulty={route.difficulty}.")

    if route.needs_clarification:
        trace.skipped("scope_guard", "Skipped — clarification needed before routing.")
        trace.skipped("generator", "Skipped — pending clarification.")
        trace.skipped("assessor", "Skipped — no response generated.")
        trace.skipped("refiner", "Skipped — no response to refine.")
        agent_trace, timeline = trace.build()
        return PipelineResponse(answer=route.clarification_message, agent_trace=agent_trace, timeline=timeline)

    # ── Scope Guard ───────────────────────────────────────────────────────
    trace.start("scope_guard")
    scope = await scope_guard_agent(question, domain, route)
    trace.done("scope_guard", f"in_scope={scope.in_scope}, confidence={scope.confidence:.2f}, matched={scope.matched_keywords[:3]}")
    trace.event(f"Scope Guard: in_scope={scope.in_scope}, confidence={scope.confidence:.2f}.")

    if not scope.in_scope:
        trace.skipped("generator", "Skipped — question is out of scope.")
        trace.skipped("assessor", "Skipped — no response generated.")
        trace.skipped("refiner", "Skipped — no response to refine.")
        agent_trace, timeline = trace.build()
        return PipelineResponse(answer=scope.redirect_message, agent_trace=agent_trace, timeline=timeline)

    # ── Generator ─────────────────────────────────────────────────────────
    trace.start("generator")
    draft = await generator_agent(question, route, domain)
    trace.done("generator", f"Draft produced ({len(draft)} chars).")
    trace.event(f"Generator produced draft ({len(draft)} chars).")

    # ── Assessor + Refiner loop ────────────────────────────────────────────
    final_answer = draft
    for round_num in range(max_refine_rounds + 1):

        trace.start("assessor")
        assessment = await assessor_agent(question, final_answer, route)
        trace.done("assessor", f"Score {assessment.score}/10 — {'approved' if assessment.ready else 'needs refinement'}.")
        trace.event(f"Assessor: score={assessment.score}/10, hallucination_risk={assessment.hallucination_risk}.")

        if assessment.ready:
            trace.skipped("refiner", f"Skipped — score {assessment.score} met threshold.")
            break

        if round_num < max_refine_rounds:
            trace.start("refiner")
            final_answer = await refiner_agent(question, final_answer, assessment)
            trace.done("refiner", f"Refinement round {round_num + 1} complete.", round=round_num + 1)
            trace.event(f"Refiner applied round {round_num + 1}.")
        else:
            trace.skipped("refiner", f"Max rounds ({max_refine_rounds}) reached, emitting best response.")

    duration_s = round(time.time() - t0, 2)
    trace.event(f"Response finalized in {duration_s}s.")

    agent_trace, timeline = trace.build()

    meta = {
        "duration_s": duration_s,
        "session_id": session_id,
        "domain": domain.domain,
        "provider": resolved_provider,
        "model": resolved_model,
        "score": assessment.score,
        "refine_rounds": round_num,
    }

    return PipelineResponse(
        answer=final_answer,
        agent_trace=agent_trace,
        timeline=timeline,
        meta=meta,
        scope_debug=scope.model_dump() if debug else None,
        assess_debug=assessment.model_dump() if debug else None,
    )
```

---

## Response Structure

Every API response includes the full trace:

```json
{
  "answer": "## Binary Trees\n\nA binary tree is...",

  "agent_trace": [
    {
      "agent": "router",
      "title": "Router",
      "status": "done",
      "summary": "Classified: mode=explain, discipline=data_structures, difficulty=beginner",
      "duration_ms": 312
    },
    {
      "agent": "scope_guard",
      "title": "Scope Guard",
      "status": "done",
      "summary": "in_scope=true, confidence=0.91, matched=['tree', 'binary', 'node']",
      "duration_ms": 298
    },
    {
      "agent": "generator",
      "title": "Generator",
      "status": "done",
      "summary": "Draft produced (1842 chars).",
      "duration_ms": 4211
    },
    {
      "agent": "assessor",
      "title": "Assessor",
      "status": "done",
      "summary": "Score 8.5/10 — approved.",
      "duration_ms": 891
    },
    {
      "agent": "refiner",
      "title": "Refiner",
      "status": "skipped",
      "summary": "Skipped — score 8.5 met threshold.",
      "duration_ms": 0
    }
  ],

  "timeline": [
    "Request received.",
    "Router: mode=explain, discipline=data_structures, difficulty=beginner.",
    "Scope Guard: in_scope=true, confidence=0.91.",
    "Generator produced draft (1842 chars).",
    "Assessor: score=8.5/10, hallucination_risk=low.",
    "Response finalized in 5.73s."
  ],

  "meta": {
    "duration_s": 5.73,
    "session_id": "uuid-...",
    "domain": "software_engineering",
    "provider": "openai",
    "model": "gpt-4.1-mini",
    "score": 8.5,
    "refine_rounds": 0
  }
}
```

---

## Frontend UX Integration

The trace enables a real-time "pipeline progress" UI. Render each `agent_trace` item as a step in an execution log:

```
● Router          [done]   312ms   Classified: mode=explain, discipline=data_structures
● Scope Guard     [done]   298ms   in_scope=true, confidence=0.91
● Generator       [done]  4211ms   Draft produced (1842 chars)
● Assessor        [done]   891ms   Score 8.5/10 — approved
○ Refiner         [skip]          Skipped — score met threshold
```

Status color conventions:
- `pending` → gray
- `running` → yellow/animated
- `done` → green
- `skipped` → muted/gray
- `error` → red

For streaming UX without WebSockets: poll or use SSE to receive status updates as each agent finishes.

---

## Logging

Write structured logs on every request for backend monitoring:

```python
import logging
import json

logger = logging.getLogger("pipeline")

def log_pipeline_result(session_id, question, meta, assessment):
    logger.info(json.dumps({
        "event":         "pipeline_complete",
        "session_id":    session_id,
        "domain":        meta["domain"],
        "provider":      meta["provider"],
        "model":         meta["model"],
        "score":         meta["score"],
        "refine_rounds": meta["refine_rounds"],
        "duration_s":    meta["duration_s"],
        "hallucination": assessment.hallucination_risk,
    }))
```

This produces machine-parseable logs that can be ingested by any log aggregator (ELK, Loki, CloudWatch).

---

## Implementation Checklist

- [ ] Define `AgentStatus` enum (pending, running, done, skipped, error)
- [ ] Define `AgentTraceItem` Pydantic model
- [ ] Implement `TraceBuilder` class with `start()`, `done()`, `skipped()`, `error()`, `event()`
- [ ] Initialize `TraceBuilder` at pipeline entry with all agent names
- [ ] Call `trace.start(agent)` before each agent call
- [ ] Call `trace.done(agent, summary)` after each successful agent call
- [ ] Call `trace.skipped(agent, reason)` for all early-exit agents
- [ ] Include `agent_trace`, `timeline`, and `meta` in every API response
- [ ] Add `debug` flag to response to optionally include full agent outputs
- [ ] Log structured JSON on pipeline completion for monitoring
- [ ] Build frontend step-progress UI using `agent_trace` status + summary
