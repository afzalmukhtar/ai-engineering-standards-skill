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
- [ ] No unused/dead models exist in the codebase
- [ ] Function signatures use models, not `Dict[str, Any]`
- [ ] If using `response_format`, verified current provider API via Context7
