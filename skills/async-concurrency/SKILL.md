---
name: async-concurrency
description: >-
  Async, parallel, and threading patterns for Python projects. Use when writing
  asyncio code, gather patterns, semaphores, thread pools, concurrent pipelines,
  or any code mixing sync and async. Covers asyncio.gather correctness, failure
  handling, to_thread bridging, timing instrumentation, and concurrency testing.
---

# Async & Concurrency

## Core Principles

1. **gather() is the concurrency primitive** — independent tasks use `asyncio.gather()`, not sequential await
2. **Failures must not cascade** — always `return_exceptions=True`, always check each result
3. **Sync in async = to_thread** — synchronous library calls inside async functions use `asyncio.to_thread()`
4. **Prove it with timing** — if you claim concurrency, log timestamps that prove overlap

## The Gather Pattern (Correct)

### Fan-Out with Failure Isolation

```python
import asyncio
import time
import logging

log = logging.getLogger(__name__)

async def fan_out(tasks):
    async def run_one(task):
        start = time.monotonic()
        log.info(f"[{task.id}] Started")
        try:
            result = await do_work(task)
        except Exception as e:
            log.error(f"[{task.id}] Failed after {time.monotonic() - start:.2f}s: {e}")
            raise
        log.info(f"[{task.id}] Completed in {time.monotonic() - start:.2f}s")
        return result

    results = await asyncio.gather(
        *(run_one(t) for t in tasks),
        return_exceptions=True,  # MANDATORY — never omit this
    )

    settled = []
    for task, result in zip(tasks, results):
        if isinstance(result, Exception):  # MANDATORY — always check each
            task.result = TaskResult(status="failed", error=str(result))
        else:
            task.result = result
        settled.append(task)

    return settled
```

### Sequential Anti-Pattern (NEVER DO THIS)

```python
# BAD: runs researchers one at a time — defeats the purpose
for task in tasks:
    result = await researcher.invoke(task)  # sequential!
    results.append(result)

# GOOD: runs all researchers concurrently
results = await asyncio.gather(
    *(researcher.invoke(t) for t in tasks),
    return_exceptions=True, # To be used when one failure shouldn't halt others
)
```

## Bridging Sync and Async

### Wrapping Synchronous Libraries

```python
# BAD: blocking call inside async function freezes the event loop
async def search(query: str) -> str:
    return requests.get(f"https://api.example.com?q={query}").text  # BLOCKS!

# GOOD: offload to thread pool
async def search(query: str) -> str:
    return await asyncio.to_thread(
        requests.get, f"https://api.example.com?q={query}"
    )
```

### LLM Client Pattern

```python
def chat_sync(messages, **kwargs):
    """Synchronous LLM call — safe for threads."""
    return completion(model=MODEL, messages=messages, **kwargs)

async def achat(messages, **kwargs):
    """Async wrapper — offloads sync call to thread pool."""
    return await asyncio.to_thread(chat_sync, messages, **kwargs)
```

### Rules

- `time.sleep()` in async function — **NEVER**. Use `await asyncio.sleep()`
- `requests.get()` in async function — **NEVER** directly. Use `asyncio.to_thread(requests.get, ...)`
- Creating `asyncio.Semaphore()` at module level — **NEVER**. Semaphores must be created inside an async context or passed as a parameter

## Semaphore Pattern

### Rate Limiting Concurrent Operations

```python
class RateLimitedClient:
    def __init__(self, max_concurrent: int = 5):
        self._semaphore: asyncio.Semaphore | None = None
        self._max = max_concurrent

    def _get_semaphore(self) -> asyncio.Semaphore:
        if self._semaphore is None:
            self._semaphore = asyncio.Semaphore(self._max)
        return self._semaphore

    async def call(self, query: str) -> str:
        async with self._get_semaphore():
            return await asyncio.to_thread(self._sync_call, query)
```

### Semaphore vs Concurrency Proof

If you use `Semaphore(1)` for rate limiting, be aware:
- Tasks still *launch* concurrently via gather
- But only one executes the semaphore-guarded section at a time
- This means timing tests won't show full overlap — document this tradeoff

## Timing Instrumentation

### Per-Task Timing

```python
import time

async def run_task(task):
    start = time.monotonic()
    result = await do_work(task)
    elapsed = time.monotonic() - start
    log.info(f"[{task.id}] Completed in {elapsed:.2f}s")
    return result
```

### Pipeline-Level Timing

```python
async def run_pipeline(query: str) -> str:
    pipeline_start = time.monotonic()

    state = AgentState(messages=[{"role": "user", "content": query}])
    # ... run pipeline ...

    total = time.monotonic() - pipeline_start
    log.info(f"Pipeline completed in {total:.2f}s")
    return state.messages[-1]["content"]
```

## Mock Tool for Concurrency Testing

When the real tool is an external API, provide a mock mode that simulates latency:

```python
import asyncio
import random

async def mock_search(query: str, max_results: int = 5) -> str:
    delay = random.uniform(0.5, 1.5)
    await asyncio.sleep(delay)

    keywords = {
        "price": "Price range: $99-$149. Available on major retailers.",
        "review": "4.2/5 stars based on 1,200+ reviews. Users praise battery life.",
        "battery": "Battery: 40 hours playback. Fast charge: 10 min = 4 hours.",
        "availability": "In stock at Amazon, Flipkart. Ships in 2-3 days.",
        "features": "Bluetooth 5.2, ANC, 40mm drivers, foldable design.",
    }

    for keyword, response in keywords.items():
        if keyword in query.lower():
            return response
    return f"General search results for: {query}"

async def mock_search_with_failures(query: str, **kwargs) -> str:
    if "fail" in query.lower():
        raise ConnectionError(f"Simulated network failure for: {query}")
    return await mock_search(query, **kwargs)
```

## Testing Concurrency

### Timing Assertion Test

```python
@pytest.mark.asyncio
async def test_researchers_run_concurrently():
    """3 tasks of ~1s each should complete in ~1s, not ~3s."""
    async def slow_task(task):
        await asyncio.sleep(1.0)
        return TaskResult(status="success", findings="data")

    tasks = [Task(id=f"T{i}", description=f"task {i}") for i in range(3)]

    start = time.monotonic()
    results = await asyncio.gather(
        *(slow_task(t) for t in tasks),
        return_exceptions=True,
    )
    elapsed = time.monotonic() - start

    assert elapsed < 2.0, f"Expected ~1s, got {elapsed:.2f}s — tasks ran sequentially"
    assert all(not isinstance(r, Exception) for r in results)
```

### Partial Failure Test

```python
@pytest.mark.asyncio
async def test_one_failure_does_not_kill_others():
    async def maybe_fail(task):
        if task.id == "FAIL":
            raise RuntimeError("simulated crash")
        await asyncio.sleep(0.5)
        return TaskResult(status="success", findings="ok")

    tasks = [
        Task(id="OK-1", description="good"),
        Task(id="FAIL", description="bad"),
        Task(id="OK-2", description="good"),
    ]

    results = await asyncio.gather(
        *(maybe_fail(t) for t in tasks),
        return_exceptions=True,
    )

    assert not isinstance(results[0], Exception)
    assert isinstance(results[1], Exception)
    assert not isinstance(results[2], Exception)
```

### Total Failure Test

```python
@pytest.mark.asyncio
async def test_all_fail_pipeline_still_completes():
    """Even if every researcher fails, the writer must still run."""
    # Mock all researchers to raise
    # Assert: state.next == "writer" after fan-out
    # Assert: writer produces a report (even if it says "no data available")
```

## Anti-Patterns to Flag

| Pattern | Problem | Fix |
|---|---|---|
| `for t in tasks: await work(t)` | Sequential, not concurrent | `asyncio.gather(*(work(t) for t in tasks))` |
| `gather()` without `return_exceptions=True` | One failure kills all siblings | Always pass `return_exceptions=True` |
| No `isinstance(result, Exception)` check | Treats exceptions as valid results | Always check each gather result |
| `time.sleep(n)` in async function | Blocks the event loop | `await asyncio.sleep(n)` |
| `requests.get()` in async function | Blocks the event loop | `await asyncio.to_thread(requests.get, ...)` |
| Module-level `asyncio.Semaphore()` | Fails if created before event loop | Create inside async context |
| No timing logs | Can't prove concurrency | Add `time.monotonic()` start/end |

## Checklist Before Moving On

- [ ] `asyncio.gather()` used for all independent concurrent tasks
- [ ] `return_exceptions=True` on every gather call
- [ ] `isinstance(result, Exception)` check after every gather
- [ ] No `time.sleep()` in async functions
- [ ] All sync library calls wrapped with `asyncio.to_thread()`
- [ ] Per-task timing logged with `time.monotonic()`
- [ ] Total pipeline time logged at orchestrator level
- [ ] At least one test asserts concurrent timing (N tasks in ~1x time, not Nx)
- [ ] At least one test asserts partial failure isolation
- [ ] Mock tool with `asyncio.sleep()` available for testing
