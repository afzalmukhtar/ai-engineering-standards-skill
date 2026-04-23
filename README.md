# AI Engineering Standards — Cursor Skill

Production-quality coding standards for Python AI/ML projects, packaged as a
[Cursor Agent Skill](https://docs.cursor.com/context/skills).

Covers five domains with a parent orchestrator that routes to the right
sub-skill based on what you're building.

## Sub-Skills

| Sub-skill | What it enforces |
|---|---|
| `pydantic-contracts` | Typed data models, field validation, serialization patterns |
| `llm-prompts` | System/user prompts, structured output specs, grounding rules |
| `agent-architecture` | Agent classes, orchestrators, tool dispatch, fan-out patterns |
| `async-concurrency` | asyncio.gather, thread pools, semaphores, failure isolation |
| `project-structure` | Project layout, config management, testing standards, code hygiene |

## Installation

### As a Personal Skill (all projects)

```bash
# Clone into your Cursor skills directory
git clone https://github.com/afzal-mukhtar/ai-engineering-standards-skill.git \
  ~/.cursor/skills/ai-engineering-standards
```

### As a Project Skill (single repo)

```bash
# Clone into your project's .cursor/skills directory
git clone https://github.com/afzal-mukhtar/ai-engineering-standards-skill.git \
  .cursor/skills/ai-engineering-standards
```

## Usage

The skill activates automatically when you're working on Python AI projects.
The parent `SKILL.md` routes to the correct sub-skill based on your current task:

- Defining data models? → `pydantic-contracts`
- Writing prompts? → `llm-prompts`
- Designing agent classes? → `agent-architecture`
- Writing async code? → `async-concurrency`
- Setting up project layout? → `project-structure`

## Priority Order

When multiple sub-skills could apply:

0. **Diagram first** — draw the architecture SVG before writing any code
1. **Data contracts** — define models before writing logic
2. **Architecture** — decide agent boundaries before implementing
3. **Prompts** — write prompts after you know what data flows in/out
4. **Async** — add concurrency after the sequential path works
5. **Structure** — clean up and organize last

## License

MIT
