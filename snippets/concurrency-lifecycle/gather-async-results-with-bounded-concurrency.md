---
title: "Gather Async Results with Bounded Concurrency"
snippet_type: recipe
use_cases:
  - concurrency-control
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related: []
---

# Gather Async Results with Bounded Concurrency

## Idea and Problem

Apply an asynchronous worker to a finite iterable while limiting active calls and preserving input order in the returned results.

A small `TaskGroup` shares one indexed iterator. Each task processes an item,
stores its result by index, and then claims the next item without an intervening
`await`, so iterator access cannot overlap on the event-loop thread. The task
count is at most the requested concurrency limit rather than one task per item.

## When to Use

Use this recipe for independent asynchronous operations when a local resource
or downstream service has a known concurrency ceiling and all results must be
returned in input order. The input must be a finite, non-blocking synchronous
iterable. Use a queue or async-iterator pipeline when input is unbounded,
produced asynchronously, or requires backpressure before iteration.

## Implementation

```python
from asyncio import CancelledError, TaskGroup
from collections.abc import Awaitable, Callable, Iterable
from typing import TypeVar


ItemT = TypeVar("ItemT")
ResultT = TypeVar("ResultT")


async def gather_bounded(
    items: Iterable[ItemT],
    worker: Callable[[ItemT], Awaitable[ResultT]],
    *,
    limit: int,
) -> list[ResultT]:
    if isinstance(limit, bool) or not isinstance(limit, int):
        raise TypeError("limit must be an integer")
    if limit <= 0:
        raise ValueError("limit must be positive")

    indexed_items = enumerate(items)
    initial_items: list[tuple[int, ItemT]] = []
    for _ in range(limit):
        try:
            initial_items.append(next(indexed_items))
        except StopIteration:
            break

    if not initial_items:
        return []

    results: dict[int, ResultT] = {}
    claimed_count = len(initial_items)

    async def run(initial: tuple[int, ItemT]) -> None:
        nonlocal claimed_count
        current = initial
        while True:
            index, item = current
            results[index] = await worker(item)
            try:
                current = next(indexed_items)
            except StopIteration:
                return
            claimed_count += 1

    async with TaskGroup() as tasks:
        for initial in initial_items:
            tasks.create_task(run(initial))

    if len(results) != claimed_count:
        raise CancelledError("a worker was cancelled before producing its result")
    return [results[index] for index in range(len(results))]
```

## Example

```python
import asyncio


async def run_example() -> tuple[list[int], int, list[int], bool, bool]:
    state = {"active": 0, "peak": 0}

    async def multiply(value: int) -> int:
        state["active"] += 1
        state["peak"] = max(state["peak"], state["active"])
        try:
            await asyncio.sleep(0)
            await asyncio.sleep(0)
            return value * 10
        finally:
            state["active"] -= 1

    ordered = await gather_bounded(range(8), multiply, limit=3)
    empty = await gather_bounded((), multiply, limit=3)

    try:
        await gather_bounded([1], multiply, limit=0)
    except ValueError:
        invalid_limit_rejected = True
    else:
        invalid_limit_rejected = False

    async def cancel_second(value: int) -> int:
        if value == 2:
            raise asyncio.CancelledError
        return value

    try:
        await gather_bounded(range(4), cancel_second, limit=1)
    except asyncio.CancelledError:
        worker_cancellation_propagated = True
    else:
        worker_cancellation_propagated = False

    return (
        ordered,
        state["peak"],
        empty,
        invalid_limit_rejected,
        worker_cancellation_propagated,
    )


assert asyncio.run(run_example()) == (
    [0, 10, 20, 30, 40, 50, 60, 70],
    3,
    [],
    True,
    True,
)
```

## Trade-offs and Limitations

The function stores every result and therefore still uses `O(n)` memory. It
bounds active worker calls and task allocation, but it does not implement a
request rate, deadline, retry, or downstream capacity policy. A worker failure
is raised through a `TaskGroup` exception group after sibling tasks are
cancelled and awaited. A worker that raises `CancelledError` is detected after
the group exits and cancellation is propagated instead of returning partial
results. Workers must propagate cancellation and clean up in `finally` blocks.
Slow synchronous input iteration still blocks the event loop.

## Related Snippets

<!-- catalog:related:start -->
No related snippets.
<!-- catalog:related:end -->
