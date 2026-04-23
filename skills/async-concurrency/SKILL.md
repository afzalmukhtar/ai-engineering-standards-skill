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
5. **Streaming pipelines use queues + sentinels** — producer-consumer stages communicate through `asyncio.Queue`, and shutdown is signalled by a sentinel value, not by exception propagation

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

### Correlated Timing for Concurrent Work Units

When multiple units of work run concurrently, a timing log is only useful if it's **correlated** — each line must carry a stable identifier for the work unit. Otherwise you cannot distinguish overlapping executions from sequential ones in the log stream.

```python
async def process_unit(unit_id: str | int, payload):
    async with semaphore:
        log.info("unit=%s — started (n=%d)", unit_id, len(payload))
        t0 = time.perf_counter()
        result = await do_work(payload)
        elapsed = time.perf_counter() - t0
        log.info("unit=%s — completed in %.2fs", unit_id, elapsed)
        return result
```

Rules:
- Every start/end log for concurrent work must carry the same correlation token (`unit=...`, `job=...`, `request=...`).
- Use `time.perf_counter()` for elapsed measurement, `time.monotonic()` for duration across awaits, never `time.time()` (wall-clock can jump).
- Log both the start and the end — an end-only log cannot prove overlap.

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

## Producer-Consumer Pipelines

Use this pattern whenever data flows through multiple async stages with independent lifecycles: ingestion → processing → output, events → enrichment → storage, requests → workers → responses. The same shape applies to logs, events, messages, jobs, frames, or any other streaming workload.

### The Canonical Shape

```
Producer ──► Queue A ──► Processor ──► Queue B ──► Terminal Consumer
              (buffer)    (transform)    (buffer)
```

- Each stage is its own `asyncio.Task` running concurrently.
- Stages communicate **only** through `asyncio.Queue`, never shared mutables.
- Stages know nothing about each other's internal state — only the queue contract.
- An end-of-stream signal (sentinel) is enqueued when the producer is done.

### Why Queues and Not Shared Collections

- **Backpressure**: a bounded `asyncio.Queue(maxsize=N)` blocks the producer when the consumer falls behind, preventing unbounded memory growth.
- **Thread/task safety**: `put`/`get` are atomic — no locks needed.
- **Decoupling**: producer and consumer can be tested, replaced, or rate-limited independently.
- **Natural shutdown**: one sentinel through the queue cleanly terminates the consumer.

### Sentinel Pattern

An end-of-stream marker travels through the same queue as the data, so the consumer doesn't need a separate "done" channel.

```python
# Option A: None sentinel with Optional queue type (simplest)
queue: asyncio.Queue[Item | None] = asyncio.Queue()
await queue.put(None)  # producer signals end-of-stream

# Option B: typed sentinel (when None has a different meaning in your domain)
END_OF_STREAM = Item(kind="__eos__")
queue: asyncio.Queue[Item] = asyncio.Queue()
await queue.put(END_OF_STREAM)

# Consumer
while True:
    item = await queue.get()
    if item is None:              # or: item is END_OF_STREAM / item.is_sentinel()
        await downstream.put(None)  # CRITICAL: propagate before breaking
        break
    process(item)
```

**Every intermediate stage must propagate the sentinel downstream before breaking its own loop.** A stage that swallows the sentinel leaves every downstream consumer blocked on `queue.get()` forever — this is the single most common hang bug in queue pipelines.

### Bounded-Wait Draining (batching with a timeout)

When a stage wants to collect up to N items but shouldn't wait indefinitely for a slow producer:

```python
async def drain_batch(queue, max_size: int, timeout_s: float) -> tuple[list, bool]:
    """Collect up to max_size items. Returns (batch, stream_ended)."""
    batch: list = []
    stream_ended = False

    first = await queue.get()  # block for the first item (no point draining an empty batch)
    if first is None:
        return batch, True
    batch.append(first)

    while len(batch) < max_size:
        try:
            item = queue.get_nowait()  # fast path: drain whatever is already buffered
        except asyncio.QueueEmpty:
            try:
                item = await asyncio.wait_for(queue.get(), timeout=timeout_s)
            except asyncio.TimeoutError:
                break  # producer is slow — ship the partial batch
        if item is None:
            stream_ended = True
            break
        batch.append(item)

    return batch, stream_ended
```

The `get_nowait` + `wait_for` hybrid fills batches at producer speed without stalling when the producer is slow.

### Fire-and-Forget with Tracked Completion

When work unit N+1 should start being gathered while work unit N is still being processed (long I/O per unit):

```python
async def run(self):
    in_flight: set[asyncio.Task] = set()
    unit_num = 0

    while True:
        work, stream_ended = await self._next_unit()
        if work:
            unit_num += 1
            task = asyncio.create_task(self._process(work, unit_num))
            in_flight.add(task)
            task.add_done_callback(in_flight.discard)  # auto-cleanup

        if stream_ended:
            if in_flight:
                # return_exceptions=True is MANDATORY here — see below
                results = await asyncio.gather(*in_flight, return_exceptions=True)
                for r in results:
                    if isinstance(r, Exception):
                        log.error("in-flight task failed: %s", r)
            await self._downstream.put(None)  # propagate sentinel
            break
```

The `in_flight` set with `add_done_callback(set.discard)` gives you automatic garbage collection of completed tasks while preserving strong references (which `asyncio` requires — see [PEP 3156](https://peps.python.org/pep-3156/)).

### Shutdown-Gather Hazard

Any `asyncio.gather()` that runs **before** propagating a shutdown signal must use `return_exceptions=True`. If any in-flight task raises, gather re-raises, the sentinel `put(...)` never runs, and every downstream consumer blocks forever on `queue.get()`.

```python
# BAD: one task raises → sentinel never propagates → downstream hangs
await asyncio.gather(*in_flight)
await downstream.put(None)

# GOOD: collect exceptions, log, always propagate
results = await asyncio.gather(*in_flight, return_exceptions=True)
for r in results:
    if isinstance(r, Exception):
        log.error("task failed during shutdown: %s", r)
await downstream.put(None)
```

### Throttling Fire-and-Forget

Unbounded `create_task` can overwhelm rate-limited downstream systems (APIs, DBs, external services). Pair `create_task` with a semaphore whose limit is injected via the constructor:

```python
class Processor:
    def __init__(self, in_q, out_q, semaphore: asyncio.Semaphore):
        self._in = in_q
        self._out = out_q
        self._sem = semaphore

    async def _process(self, work, unit_num):
        async with self._sem:
            result = await self._do_work(work)
            for r in result:
                await self._out.put(r)
```

Never construct the semaphore at module level — construct it inside an async context (usually the bootstrap function) and inject it.

### Wiring Stages in the Bootstrap

```python
async def main():
    in_q: asyncio.Queue = asyncio.Queue()
    mid_q: asyncio.Queue = asyncio.Queue()
    sem = asyncio.Semaphore(max_concurrent)

    producer = Producer(in_q)
    processor = Processor(in_q, mid_q, sem)
    consumer = Consumer(mid_q)

    results = await asyncio.gather(
        producer.run(),
        processor.run(),
        consumer.run(),
        return_exceptions=True,
    )
    for name, result in zip(("producer", "processor", "consumer"), results):
        if isinstance(result, Exception):
            log.error("%s stage failed: %s", name, result)
```

### Pipeline Anti-Patterns

| Pattern | Problem | Fix |
|---|---|---|
| Consumer breaks on sentinel without propagating | Downstream stages hang forever | `put(None)` on downstream queue before `break` |
| `gather()` at shutdown without `return_exceptions=True` | One task error prevents sentinel propagation → hang | Always `return_exceptions=True`, log exceptions, then propagate sentinel |
| Producer + consumer with one queue and no sentinel | Consumer never knows when to stop | Use sentinel value or `queue.join()` + `task_done()` |
| Shared list/dict between stages instead of a queue | Races and no backpressure | Use `asyncio.Queue` |
| Bounded batch collector using only `await queue.get()` | Blocks forever when producer is slower than batch size | Use `get_nowait` + `wait_for(timeout)` hybrid |
| Unbounded `create_task` for external-service calls | Rate-limit storms | Wrap in semaphore; inject via constructor |
| `create_task` without retaining a reference | Task may be garbage-collected mid-execution | Store in a `set`, use `add_done_callback(set.discard)` |

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
| `gather(*in_flight)` at shutdown without `return_exceptions=True` | Sentinel never propagates → downstream hangs | Always `return_exceptions=True`, then propagate sentinel |
| No `isinstance(result, Exception)` check | Treats exceptions as valid results | Always check each gather result |
| `time.sleep(n)` in async function | Blocks the event loop | `await asyncio.sleep(n)` |
| `requests.get()` in async function | Blocks the event loop | `await asyncio.to_thread(requests.get, ...)` |
| Module-level `asyncio.Semaphore()` | Fails if created before event loop | Create inside async context |
| No timing logs | Can't prove concurrency | Add `time.monotonic()` start/end |
| Timing logs without a correlation token | Can't distinguish concurrent from sequential work in log output | Include `unit=/job=/request=` in every start and end log |
| Consumer breaks on sentinel without re-emitting it | Downstream consumers hang on `queue.get()` | `put(None)` / `put(SENTINEL)` before `break` |
| Pipeline stages communicating via shared dict/list | Races and no backpressure | Use `asyncio.Queue` between every pair of stages |

## Checklist Before Moving On

- [ ] `asyncio.gather()` used for all independent concurrent tasks
- [ ] `return_exceptions=True` on **every** gather call, including shutdown gathers
- [ ] `isinstance(result, Exception)` check after every gather
- [ ] No `time.sleep()` in async functions
- [ ] All sync library calls wrapped with `asyncio.to_thread()`
- [ ] Per-task timing logged with `time.monotonic()`
- [ ] Concurrent work units log start + end with a correlation token (`unit=/job=/request=`)
- [ ] Total pipeline time logged at orchestrator level
- [ ] At least one test asserts concurrent timing (N tasks in ~1x time, not Nx)
- [ ] At least one test asserts partial failure isolation
- [ ] Mock tool with `asyncio.sleep()` available for testing

### Additional for Producer-Consumer Pipelines

- [ ] Every stage communicates via `asyncio.Queue`, not shared mutables
- [ ] Sentinel value is defined once in `models/` (typed sentinel or `None`)
- [ ] Every intermediate stage propagates the sentinel downstream before breaking
- [ ] Batch drain uses `get_nowait` + `wait_for(timeout)` hybrid for fast producers
- [ ] Fire-and-forget tasks are tracked in a `set` with `add_done_callback(set.discard)`
- [ ] Shutdown gather uses `return_exceptions=True` and logs each exception
- [ ] Semaphore is injected via constructor, not created at module level
