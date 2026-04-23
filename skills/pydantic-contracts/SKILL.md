---
name: pydantic-contracts
description: >-
  Enforce Pydantic data contract patterns in Python projects. Use when defining
  models, schemas, state objects, API payloads, agent state, or any structured
  data that crosses module boundaries. Covers model design, validation, Literal
  types, serialization, and wiring models into the actual data path.
---

# Pydantic Data Contracts

## Core Principles

1. **Models before logic** — define every Pydantic model before writing any function that consumes it
2. **No raw dicts across boundaries** — every object that crosses a module boundary must be a `BaseModel`
3. **If it's defined, it must be used** — dead models are worse than no models; delete unused ones
4. **Structured over stringly-typed** — use `Literal`, `Enum`, typed fields; never `str` for known shapes
5. **Correlation fields must actually correlate** — any field whose purpose is to link related entities must share a value across the group; never auto-generate per entity
6. **Declare all instance state in `__init__`** — every `self.*` attribute used by any method must be set (with a type annotation) in the constructor; first-appearances inside methods silently reset and hide from tooling

## Patterns

### Status Fields — Always Use Literal

```python
# BAD: stringly-typed, no validation
class Result(BaseModel):
    status: str
    data: Optional[str] = None

# GOOD: constrained, self-documenting
class Result(BaseModel):
    status: Literal["success", "failed", "pending"]
    data: Optional[str] = None
    error: Optional[str] = None
```

### Nested Models for Complex Results

```python
# BAD: flat string result loses structure
class Task(BaseModel):
    id: str
    description: str
    result: Optional[str] = None  # "Error: failed" — is this an error or data?

# GOOD: structured result preserves metadata
class TaskResult(BaseModel):
    status: Literal["success", "failed"]
    findings: str = ""
    error: Optional[str] = None
    source_urls: list[str] = Field(default_factory=list)

class Task(BaseModel):
    id: str
    description: str
    result: Optional[TaskResult] = None
```

### Loading External Data — Always Validate

```python
# BAD: trusting raw JSON
data = json.loads(response_text)
task = Task(**data)  # no validation, KeyError on bad shape

# GOOD: validate and get clear errors
try:
    task = Task.model_validate_json(response_text)
except ValidationError as e:
    log.error(f"Invalid task data: {e}")
    task = Task(id="fallback", description=original_query)
```

### LLM Output Parsing

```python
# BAD: hope the LLM returns valid JSON
result = json.loads(llm_response)

# GOOD: parse with fallback
from pydantic import ValidationError

def parse_llm_output(raw: str, model: type[BaseModel], fallback):
    text = raw.strip()
    if text.startswith("```"):
        text = text.split("\n", 1)[1].rsplit("```", 1)[0].strip()
    try:
        return model.model_validate_json(text)
    except (ValidationError, json.JSONDecodeError) as e:
        log.warning(f"LLM output failed validation: {e}")
        return fallback
```

### Serialization for LLM Context

```python
# BAD: passing Pydantic object directly to prompt string
prompt = f"Here is the data: {my_model}"  # calls __repr__, ugly

# GOOD: explicit JSON serialization
prompt = f"Here is the data:\n{my_model.model_dump_json(indent=2)}"
```

### Correlation Fields — Shared, Not Random

A **correlation field** is any field whose purpose is to link related entities together (trace_id, request_id, session_id, conversation_id, transaction_id, saga_id, workflow_id, parent_id, thread_id). Its value is only meaningful when the same value appears on every entity in the group.

Auto-generating a correlation field per entity breaks the contract the field promises. `default_factory=uuid.uuid4` on a correlation field is almost always wrong — the field becomes decorative and any downstream consumer that groups by it will produce a group size of 1.

```python
# BAD: every entity gets a unique id — the "correlation" is a lie
class Event(BaseModel):
    source: str
    payload: dict
    correlation_id: str = Field(default_factory=lambda: uuid.uuid4().hex)

# Fanning out one logical action across three events produces three different
# correlation_ids. Nothing groups, nothing traces, the field is dead.

# GOOD: the caller that knows the entities are related sets the shared value
class Event(BaseModel):
    source: str
    payload: dict
    correlation_id: str  # required — caller decides grouping

def emit_related(sources: list[str], payloads: list[dict]) -> list[Event]:
    shared_id = uuid.uuid4().hex  # one id for the whole logical group
    return [Event(source=s, payload=p, correlation_id=shared_id)
            for s, p in zip(sources, payloads)]
```

The rule generalizes: if removing an entity's correlation value and regenerating it would change system behavior, it's a correlation field and the caller owns it.

Special case — LLM-supplied correlation fields: when a model returns a correlation id as part of its output, treat it as hallucination-prone and overwrite from the source record:

```python
parsed = ResponseModel.model_validate_json(raw)
# model may have invented or mis-copied the id
grounded = parsed.model_copy(update={"correlation_id": source.correlation_id})
```

### Instance State Hygiene

Every `self.*` attribute a class uses must be declared in `__init__`. Attributes that first appear inside a method are invisible to type checkers and static analysis, silently reset every time that method is re-entered, and make the class's state surface impossible to audit from the constructor alone.

```python
# BAD: state first appears inside a method
class Worker:
    def __init__(self, queue):
        self._queue = queue

    async def run(self):
        self._call_count = 0        # first appearance — resets on every run()
        self._last_error = None     # same
        while True:
            ...

# GOOD: full state surface visible at construction
class Worker:
    def __init__(self, queue):
        self._queue = queue
        self._call_count: int = 0
        self._last_error: Exception | None = None

    async def run(self):
        while True:
            ...
```

Rules:
- Every `self.x` referenced anywhere in the class has a corresponding `self.x = …` in `__init__`.
- Use explicit type annotations on non-trivial state (`self._x: int = 0`) — this is the only signal a reader gets about the attribute's type.
- If a method needs to reset state at the top of its body, that's a signal the same instance is being reused for what should be independent runs — consider whether a fresh instance per run is more appropriate, or whether a `reset()` method should make the resetting explicit.
- Avoid mutable class-level defaults (`items: list = []` on the class body) — they are shared across instances. Initialize in `__init__`.

### Return Types — Never Dict[str, Any]

```python
# BAD: opaque return type
def process(data: dict) -> dict[str, Any]:
    return {"status": "ok", "items": [...]}

# GOOD: typed contract
class ProcessResult(BaseModel):
    status: Literal["ok", "error"]
    items: list[Item]
    error: Optional[str] = None

def process(data: InputModel) -> ProcessResult:
    ...
```

## Anti-Patterns to Flag

| Pattern | Problem | Fix |
|---|---|---|
| `result: Optional[str]` for structured data | Loses status/error metadata | Create a typed result model |
| `Dict[str, Any]` as return type | Caller has no contract | Define a Pydantic model |
| Model defined but never instantiated | Dead code, confusing | Delete it or wire it in |
| `**data` unpacking without validation | Silent data corruption | Use `model_validate()` |
| `status: str` | No constraint on values | Use `Literal["success", "failed"]` |
| Correlation field uses `default_factory=uuid.uuid4` per entity | Field doesn't actually correlate — group size is always 1 | Require caller to supply; caller generates one id per logical group |
| `self.x = …` first appears inside a method | Silent reset on re-entry, invisible to type checker / linters | Declare every `self.x` (with type annotation) in `__init__` |
| LLM-supplied correlation ids trusted verbatim | Model can fabricate or mis-copy ids | `model_copy(update={"correlation_id": source.correlation_id})` after parse |
| Mutable default on class body (`items: list = []`) | Shared across instances | Initialize in `__init__` |

## Provider-Specific Structured Output

When using Pydantic models as `response_format` for LLM calls, the wiring differs
by provider and SDK version. Use Context7 (`resolve-library-id` then `query-docs`)
to look up the current structured output API for your provider before writing the
integration code. Do not assume the API from memory — it changes frequently.

## Checklist Before Moving On

- [ ] Every data structure crossing a module boundary is a `BaseModel`
- [ ] Status fields use `Literal` types, not `str`
- [ ] Error fields are `Optional[str] = None`, not embedded in data strings
- [ ] LLM JSON output is parsed via `model_validate_json()` with fallback
- [ ] Correlation fields (anything intended to link related entities) are supplied by the caller, not auto-generated per entity
- [ ] LLM-supplied correlation ids are overwritten from the source record after parsing
- [ ] Every `self.*` attribute is declared in `__init__` with a type annotation; no attribute first-appears inside a method
- [ ] No mutable default (`list`, `dict`, `set`) on a class body — initialize in `__init__`
- [ ] No unused/dead models exist in the codebase
- [ ] Function signatures use models, not `Dict[str, Any]`
- [ ] If using `response_format`, verified current provider API via Context7
