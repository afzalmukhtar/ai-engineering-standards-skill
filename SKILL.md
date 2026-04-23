---
name: ai-engineering-standards
description: >-
  Enforce production-quality patterns for Python AI/ML projects: Pydantic data
  contracts, LLM prompt engineering, multi-agent architecture, async concurrency,
  and project structure. Use when building agents, LLM pipelines, RAG systems,
  tool-calling flows, or any Python AI project. Routes to domain-specific
  sub-skills based on the current task.
---

# AI Engineering Standards

Orchestrator skill that routes to domain-specific sub-skills. Invoke the
relevant sub-skill based on what you are currently doing.

## Routing Table

| You are doing... | Invoke sub-skill |
|---|---|
| Drawing an architecture diagram or SVG spec before coding | `build-agent-architecture-svg` (personal skill) |
| About to implement anything (pre-flight before writing code) | `coding-discipline` |
| Defining data models, schemas, state objects, or function signatures | `pydantic-contracts` |
| Writing system/user prompts, few-shot examples, or structured output specs | `llm-prompts` |
| Designing agent classes, orchestrators, tool routing, or multi-agent flows | `agent-architecture` |
| Writing async code, gather patterns, thread pools, or concurrent pipelines | `async-concurrency` |
| Setting up project layout, config, tests, CI, or cleaning up code | `project-structure` |

## When Multiple Apply

Use this priority order:
0. **Diagram first** — design what to build (architecture SVG, whiteboard, spec)
1. **Discipline second** — govern how you build it (think, simplify, be surgical, verify)
2. **Data contracts** — define models before writing logic that consumes them
3. **Architecture** — decide agent boundaries before implementing agents
4. **Prompts** — write prompts after you know what data flows in/out
5. **Async** — add concurrency after the sequential path works
6. **Structure last** — clean up and organize after the feature is working

## Quick Reference: Universal Rules

These apply regardless of which sub-skill is active:

- Never hardcode secrets, API keys, or credentials in source code
- Every function that crosses a module boundary must have typed parameters and return types
- Every module must have at least one test before shipping
- Dead code must be deleted, not commented out
- Config values live in one place (env, settings file, or constants module)
- When accepting AI-Agent generated code, read every line — if you cannot explain it, rewrite it
