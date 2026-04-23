---
name: agent-architecture
description: >-
  Multi-agent architecture patterns for Python AI projects. Use when designing
  agent classes, orchestrators, tool routing, state machines, fan-out patterns,
  or any system with multiple LLM-powered agents. Covers agent separation,
  orchestration, tool dispatch, retry logic, and loop safety.
---

# Agent Architecture

## Core Principles

1. **One agent, one job** — each agent class owns exactly one responsibility
2. **Agents never import agents** — inter-agent communication goes through shared state
3. **Orchestration is separate** — sequencing, routing, fan-out live in the orchestrator, not inside agents
4. **Tools are data, not control flow** — dispatch via registry dict, never if/else chains

## Agent Class Pattern

### Base Agent

```python
from abc import ABC, abstractmethod

class BaseAgent(ABC):
    def __init__(self, name: str, description: str):
        self.name = name
        self.description = description

    @abstractmethod
    async def invoke(self, state: AgentState) -> AgentState:
        """Single public method. Takes state, returns state."""
        ...
```

### Rules for Agent Classes

- **One public method**: either `run()` or `invoke()` — never both, never extras
- **Stateless execution**: all data flows through the state object, not instance variables
- **No side-channel communication**: agents don't call other agents, emit events, or write to shared mutables
- **Internal error handling**: catch exceptions inside the agent, set error fields on the result — never let exceptions propagate to the orchestrator unhandled

```python
class ResearcherAgent(BaseAgent):
    async def invoke(self, state: AgentState) -> AgentState:
        task = state.tasks[0]
        try:
            raw = await search_tool(task.description)
        except Exception as e:
            task.result = TaskResult(status="failed", error=str(e))
            return state

        try:
            synthesis = await achat([...])
        except Exception as e:
            task.result = TaskResult(status="failed", error=f"LLM failed: {e}")
            return state

        task.result = TaskResult(status="success", findings=synthesis)
        return state
```

## Orchestrator Pattern

### State Machine Router

```python
class Orchestrator:
    def __init__(self, agent_registry: dict):
        self.registry = agent_registry

    async def run(self, state: AgentState) -> AgentState:
        for _ in range(MAX_ITERATIONS):
            target = self.route(state)
            if target is None:
                break

            if target == "researcher":
                state = await self.fan_out(state)
            else:
                agent = self.registry[target]
                state = await agent.invoke(state)

            state.sender = target
        return state

    def route(self, state: AgentState) -> Optional[str]:
        transitions = {
            "user": "planner",
            "planner": "researcher",
            "researcher": "writer",
            "writer": None,
        }
        return transitions.get(state.sender)
```

### Key Orchestrator Rules

- **MAX_ITERATIONS is mandatory** — never `while True` without a cap
- **Transition table is explicit** — define all valid state transitions in one place
- **Fan-out uses `asyncio.gather`** — never sequential await in a loop for independent tasks
- **Unknown states halt cleanly** — log a warning and stop, don't crash

## Fan-Out Pattern

```python
async def fan_out(self, state: AgentState) -> AgentState:
    async def run_one(task):
        agent = self.researcher_factory(task_id=task.id)
        local_state = AgentState(tasks=[task], sender="orchestrator")
        return await agent.invoke(local_state)

    results = await asyncio.gather(
        *(run_one(t) for t in state.tasks),
        return_exceptions=True,
    )

    for task, result in zip(state.tasks, results):
        if isinstance(result, Exception):
            task.result = TaskResult(status="failed", error=str(result))
        else:
            task.result = result.tasks[0].result

    state.sender = "researcher"
    return state
```

## Tool Dispatch

### Registry Pattern

```python
TOOL_REGISTRY: dict[str, Callable] = {
    "web_search": web_search,
    "calculate": calculate,
    "lookup_db": lookup_db,
}

def dispatch_tool(name: str, args: dict) -> str:
    fn = TOOL_REGISTRY.get(name)
    if fn is None:
        return f"Error: unknown tool '{name}'"
    try:
        return str(fn(**args))
    except Exception as e:
        return f"Error executing {name}: {e}"
```

### Rules

- Tools are registered as `{"name": callable}` — never if/elif chains
- Tool results are always strings — serialize complex returns
- Failed tools return error strings, never raise into the LLM loop
- After a tool result message, set `tool_choice=None` or `"auto"` — never force consecutive tool calls

## LLM Call Wrapper

Every project needs a centralized LLM call function with retry logic:

```python
import asyncio
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=1, max=10),
    reraise=True,
)
def chat(messages, temperature=0.1, timeout=60):
    response = completion(model=MODEL, messages=messages, temperature=temperature, timeout=timeout)
    content = response.choices[0].message.content or ""
    if not content.strip():
        raise ValueError("LLM returned empty response")
    return content

async def achat(messages, **kwargs):
    return await asyncio.to_thread(chat, messages, **kwargs)
```

### LLM Call Rules

- **Retry with exponential backoff** — default 3 attempts
- **Empty response handling** — treat empty string as an error, retry
- **Centralized in one module** — `llm.py` or `config/llm.py`, never scattered
- **Sync wrapper + async bridge** — `chat()` is sync, `achat()` uses `to_thread()`

## Wiring / Bootstrap

All agent assembly happens in one file (`driver.py`, `main.py`, or `bootstrap.py`):

```python
def build_pipeline():
    planner = PlannerAgent(name="Planner", description="Decomposes tasks")
    writer = WriterAgent(name="Writer", description="Compiles reports")

    def researcher_factory(task_id):
        return ResearcherAgent(name=f"Researcher-{task_id}", description="Researches")

    return Orchestrator(agent_registry={
        "planner": planner,
        "researcher": researcher_factory,
        "writer": writer,
    })
```

## Anti-Patterns to Flag

| Pattern | Problem | Fix |
|---|---|---|
| `while True` without MAX_ITERATIONS | Infinite loop risk | Add counter and cap |
| Agent A imports Agent B | Tight coupling | Communicate via shared state only |
| Orchestration inside an agent | Violates single responsibility | Move to orchestrator |
| `if tool == "search": ...` chain | Brittle dispatch | Use registry dict |
| No retry on LLM calls | Transient failures crash pipeline | Add tenacity or manual backoff |
| `chat()` returns error string silently | Caller can't distinguish error from data | Raise on failure, catch in agent |

## Checklist Before Moving On

- [ ] Each agent has exactly one public method (`invoke` or `run`)
- [ ] No agent imports another agent class
- [ ] Orchestrator owns all sequencing, routing, and fan-out logic
- [ ] MAX_ITERATIONS defined and enforced in every loop
- [ ] LLM calls have retry with backoff and empty-response handling
- [ ] Tool dispatch uses a registry dict
- [ ] All agent wiring happens in one bootstrap file
