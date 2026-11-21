---
title: "Tighten One asyncio Deadline After Reading a Work Budget"
snippet_type: pattern
use_cases:
  - concurrency-control
  - lifecycle-management
tested_python:
  - "3.14"
dependencies: []
related:
  - reuse-one-pending-future-across-non-cancelling-poll-timeouts.md
  - ../reliability-resilience/propagate-a-monotonic-deadline-with-contextvar.md
---

# Tighten One asyncio Deadline After Reading a Work Budget

## Idea and Problem

Start one asyncio timeout before reading a budget, then tighten its deadline before starting work.

Both phases share the original cutoff. A returned duration can move that cutoff earlier but never extend it. Check the deadline before calling the work factory, since a coroutine can complete synchronously without giving the timeout callback a turn.

## When to Use

Use this for a two-phase local asyncio operation whose budget is learned asynchronously. Pass the outer cutoff as an absolute value from the current event loop clock, and return a duration in seconds from the budget reader.

## Implementation

```python
import asyncio
from collections.abc import Awaitable, Callable
from math import isfinite


async def run_with_budget[T](
    initial_cutoff: float,
    read_budget: Callable[[], Awaitable[float]],
    work: Callable[[], Awaitable[T]],
) -> T:
    if not isfinite(initial_cutoff):
        raise ValueError("initial cutoff must be finite")

    loop = asyncio.get_running_loop()
    async with asyncio.timeout_at(initial_cutoff) as timeout:
        if initial_cutoff <= loop.time():
            raise TimeoutError("no time remains to read the budget")
        seconds = await read_budget()
        if not isfinite(seconds) or seconds < 0:
            raise ValueError("budget must be a finite nonnegative duration")

        now = loop.time()
        cutoff = min(initial_cutoff, now + seconds)
        if cutoff <= now:
            raise TimeoutError("no time remains to start work")
        timeout.reschedule(cutoff)
        return await work()
```

## Example

```python
async def check_budget() -> tuple[str, list[str]]:
    calls = []

    async def work() -> str:
        calls.append("started")
        return "done"

    async def budget(seconds: float) -> float:
        await asyncio.sleep(0)
        return seconds

    loop = asyncio.get_running_loop()
    result = await run_with_budget(loop.time() + 5, lambda: budget(2), work)

    try:
        await run_with_budget(loop.time() + 5, lambda: budget(0), work)
    except TimeoutError:
        pass
    else:
        raise AssertionError("zero budget must expire")

    assert calls == ["started"]

    async def blocked_work() -> str:
        try:
            await asyncio.Event().wait()
        finally:
            calls.append("stopped")

    try:
        await run_with_budget(loop.time() + 5, lambda: budget(0.02), blocked_work)
    except TimeoutError:
        pass
    else:
        raise AssertionError("blocking work must time out")

    return result, calls


assert asyncio.run(check_budget()) == ("done", ["started", "stopped"])
```

## Trade-offs and Limitations

Timeout delivery is cooperative: blocking work or code that suppresses cancellation can outlive this budget. The timeout does not roll back side effects. Do not interpret an arbitrary `TimeoutError` raised by either callable as proof that this timeout expired. Event-loop clock readings are local; do not transmit them as shared timestamps.

## Related Snippets

<!-- catalog:related:start -->
- [Reuse One Pending Future Across Non-Cancelling Poll Timeouts](reuse-one-pending-future-across-non-cancelling-poll-timeouts.md)
- [Propagate a Monotonic Deadline with ContextVar](../reliability-resilience/propagate-a-monotonic-deadline-with-contextvar.md)
<!-- catalog:related:end -->
