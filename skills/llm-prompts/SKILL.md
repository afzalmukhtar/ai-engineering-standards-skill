---
name: llm-prompts
description: >-
  LLM prompt engineering patterns for Python AI projects. Use when writing
  system prompts, user prompts, few-shot examples, structured output
  specifications, or configuring LLM calls. Covers prompt organization,
  output schema specification, grounding rules, and prompt testing.
---

# LLM Prompt Engineering

## Core Principles

1. **Prompts are config, not code** — store in `config/prompts.py` or `prompts/` directory, never inline in agent logic
2. **Explicit over vague** — "Return ONLY a JSON array with keys id and description" beats "be helpful"
3. **Constrain the output** — specify exact schema, field names, and value constraints in the prompt
4. **Ground the model** — "Use ONLY the provided data" prevents hallucination in synthesis tasks

## Prompt Organization

### File Structure

```
config/
  prompts.py          # all prompt templates as module-level constants
  # OR
prompts/
  planner.py          # one file per agent/role
  researcher.py
  writer.py
```

### Template Pattern

```python
PLANNER_SYSTEM = """You are a task planner. Given a user request,
decompose it into {min_tasks}-{max_tasks} independent, parallel-executable subtasks.

Current date: {now}

Output rules:
- Return ONLY a JSON array. No markdown fences, no explanation.
- Each element must have: "id" (string, e.g. "T1") and "description" (string, search-friendly query).
- Include the current year in queries where recency matters.

Example output:
[
  {{"id": "T1", "description": "iPhone 16 price India 2026"}},
  {{"id": "T2", "description": "Samsung Galaxy S26 review 2026"}}
]"""
```

### Key Elements Every Prompt Needs

| Element | Purpose | Example |
|---|---|---|
| **Role** | Sets behavioral frame | "You are a market research analyst" |
| **Input description** | What the model receives | "You will be given search results and a task" |
| **Output format** | Exact schema | "Return a JSON object with keys: summary, findings, gaps" |
| **Constraints** | Boundaries | "Keep under 200 words", "Do not invent data" |
| **Grounding rule** | Prevents hallucination | "Use ONLY the provided research results" |
| **Current date** | Recency awareness | "Current date: {now}" |

## Structured Output Specification

### In the Prompt

```python
WRITER_SYSTEM = """You are a report writer. Compile research findings into a structured report.

Output format (JSON):
{{
  "summary": "2-3 sentence executive summary",
  "sections": [
    {{
      "subtask_id": "T1",
      "heading": "section title",
      "content": "findings for this subtask"
    }}
  ],
  "gaps": ["T3", "T5"],
  "recommendation": "final recommendation based on findings"
}}

Rules:
- The "gaps" array must list IDs of any subtask where research failed or returned errors.
- Do NOT add information beyond what the research provides.
- If a research result contains "Error:", include that subtask ID in "gaps"."""
```

### Paired with Pydantic Validation

```python
class ReportSection(BaseModel):
    subtask_id: str
    heading: str
    content: str

class Report(BaseModel):
    summary: str
    sections: list[ReportSection]
    gaps: list[str] = Field(default_factory=list)
    recommendation: str

raw = await achat(messages)
report = parse_llm_output(raw, Report, fallback=Report(
    summary="Report generation failed",
    sections=[],
    gaps=[t.id for t in tasks],
    recommendation="Insufficient data to make recommendation"
))
```

## Grounding Patterns

### For Synthesis/RAG Tasks

```
- Use ONLY the data provided in the research results below.
- Do NOT add facts, prices, dates, or claims from your training data.
- If the provided data is insufficient, explicitly state what is missing.
- Cite the subtask ID (e.g. [T1]) when referencing a specific finding.
```

### For Planning/Decomposition Tasks

```
- Each subtask must be independently executable with no dependency on other subtasks.
- Write descriptions as search-engine-friendly queries, not full sentences.
- Generate between {min} and {max} subtasks.
```

### For Tool-Use Tasks

```
- You have access to these tools: {tool_names}
- Call exactly ONE tool per turn. Wait for the result before deciding next action.
- If no tool is appropriate, respond with your best answer using available context.
- Never fabricate tool results.
```

## Gap Reporting Pattern

When a pipeline has partial failures, the writer/synthesizer prompt must handle them:

```
Research results may contain errors (prefixed with "Error:").
For each failed subtask:
1. Include the subtask ID in the "gaps" array
2. Note in the report body: "Research for [subtask description] was unavailable"
3. Do NOT attempt to fill the gap with general knowledge
```

## Anti-Patterns to Flag

| Pattern | Problem | Fix |
|---|---|---|
| Prompt as f-string inside `invoke()` | Untestable, not reusable | Move to `config/prompts.py` |
| "Be helpful and accurate" | Too vague to constrain behavior | Specify exact output schema and rules |
| No output format spec | LLM returns unparseable text | Add JSON schema with example in prompt |
| No grounding rule | Model hallucinates beyond input | Add "Use ONLY the provided data" |
| No gap/error handling instruction | Silently drops failed inputs | Add explicit gap reporting rules |
| Hardcoded date | Stale temporal context | Use `{now}` placeholder, format at call time |

## Checklist Before Moving On

- [ ] All prompts live in `config/prompts.py` or `prompts/` directory
- [ ] Every prompt specifies exact output format (JSON schema or template)
- [ ] Synthesis prompts include grounding rule ("use ONLY provided data")
- [ ] Writer/report prompts include gap reporting instructions
- [ ] Planner prompts specify subtask count range
- [ ] Date-sensitive prompts include `{now}` placeholder
- [ ] Prompts are tested: feed known input, validate output parses correctly
