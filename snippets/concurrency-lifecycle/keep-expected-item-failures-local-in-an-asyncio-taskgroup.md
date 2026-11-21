---
title: "Keep Expected Item Failures Local in an asyncio TaskGroup"
snippet_type: pattern
use_cases:
  - concurrency-control
  - lifecycle-management
tested_python:
  - "3.14"
dependencies: []
related:
  - gather-async-results-with-bounded-concurrency.md
  - correlate-bounded-async-results-by-original-task-as-they-complete.md
  - ../python-language/split-a-bounded-exceptiongroup-by-closed-exception-types.md
---

# Keep Expected Item Failures Local in an asyncio TaskGroup

## Idea and Problem

Let an expected missing item become a result while keeping fail-fast behavior for unexpected errors in an asynchronous batch.

A child exception that reaches a `TaskGroup` can cancel unfinished siblings.
If an absent item is an ordinary outcome, handle its specific exception inside
the child. This lets the other reads finish and leaves group-level cleanup in
place for unexpected errors and caller cancellation.

## When to Use

Use this for a small group of independent reads where a missing entry is
acceptable and successful reads return strings. Return `None` for the named
missing-item case. The reader must raise `ItemUnavailable` only when absence
has the meaning the caller expects, such as a removed lookup entry.

## Implementation

```python
import asyncio
from collections.abc import Awaitable, Callable


class ItemUnavailable(Exception):
    pass


async def read_items(
    keys: tuple[str, ...],
    read: Callable[[str], Awaitable[str]],
) -> tuple[str | None, ...]:
    async def read_one(key: str) -> str | None:
        try:
            return await read(key)
        except ItemUnavailable:
            return None

    async with asyncio.TaskGroup() as group:
        tasks = [group.create_task(read_one(key)) for key in keys]
    return tuple(task.result() for task in tasks)
```

## Example

```python
async def demonstrate() -> tuple[tuple[str | None, ...], tuple[str, ...], bool]:
    missing_reported = asyncio.Event()

    async def lookup(key: str) -> str:
        if key == "removed":
            missing_reported.set()
            raise ItemUnavailable(key)
        await missing_reported.wait()
        return "available"

    partial = await read_items(("removed", "current"), lookup)

    waiting_started = asyncio.Event()
    released = asyncio.Event()
    pending_read = asyncio.Event()

    async def broken_lookup(key: str) -> str:
        if key == "broken":
            await waiting_started.wait()
            raise RuntimeError("invalid response")
        try:
            waiting_started.set()
            await pending_read.wait()
            return "available"
        finally:
            released.set()

    unexpected_errors: tuple[str, ...] = ()
    try:
        await read_items(("waiting", "broken"), broken_lookup)
    except* RuntimeError as failures:
        unexpected_errors = tuple(str(error) for error in failures.exceptions)

    return partial, unexpected_errors, released.is_set()


assert asyncio.run(demonstrate()) == ((None, "available"), ("invalid response",), True)
```

## Trade-offs and Limitations

The group starts one task per key and returns results in input order after
all children finish. Use a bounded worker pool for many items. Choose an
explicit result type when `None` is already a valid value or when several
expected outcomes need different handling.

Unexpected ordinary exceptions still cancel pending siblings and reach the
caller through an exception group. Child code must cooperate with cancellation
and release owned resources when it exits. A cancelled child has no result,
so reading that child's result raises `CancelledError`. An `except*` around
the group can inspect failures after cleanup, by which point the cancellation
of siblings has already happened.

## Related Snippets

<!-- catalog:related:start -->
- [Gather Async Results with Bounded Concurrency](gather-async-results-with-bounded-concurrency.md)
- [Correlate Bounded Async Results by Original Task as They Complete](correlate-bounded-async-results-by-original-task-as-they-complete.md)
- [Split a Bounded ExceptionGroup by Closed Exception Types](../python-language/split-a-bounded-exceptiongroup-by-closed-exception-types.md)
<!-- catalog:related:end -->
