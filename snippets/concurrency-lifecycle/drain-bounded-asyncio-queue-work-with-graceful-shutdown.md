---
title: "Drain Bounded asyncio Queue Work with Graceful Shutdown"
snippet_type: pattern
use_cases:
  - concurrency-control
  - lifecycle-management
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - gather-async-results-with-bounded-concurrency.md
  - run-bounded-thread-work-by-priority-and-submission-order.md
  - collect-a-bounded-async-generator-prefix-with-deterministic-closure.md
---

# Drain Bounded asyncio Queue Work with Graceful Shutdown

## Idea and Problem

Process one bounded async batch through a capacity-limited queue, drain every accepted item, and stop workers without sentinel values.

Python's graceful `asyncio.Queue.shutdown()` rejects later producers while
allowing queued items to be received. Each worker pairs one successful `get()`
with exactly one `task_done()` in `finally`, so ordinary handler failures can
be captured without making `join()` hang. Frozen success and failure records
restore input order independently of worker scheduling.

## When to Use

Use this pattern for one in-memory batch whose async handler may run safely on
any of a fixed number of workers. Choose queue capacity as the maximum pending
work and inspect every returned failure. The handler contract is one awaitable
result or an ordinary `Exception` for each invocation.

Use a durable work system for crash recovery, cross-process consumers,
redelivery or transactional acknowledgement. Use streaming ownership instead
of this collector when the input or returned outcomes cannot be materialized
as one bounded tuple.

## Implementation

```python
import asyncio
from collections.abc import Awaitable, Callable
from dataclasses import dataclass

_MAX_ITEMS = 10_000
_MAX_WORKERS = 64
_MAX_QUEUE_CAPACITY = 10_000


@dataclass(frozen=True, slots=True)
class WorkSuccess[T]:
    index: int
    value: T


@dataclass(frozen=True, slots=True)
class WorkFailure:
    index: int
    exception: Exception


type WorkOutcome[T] = WorkSuccess[T] | WorkFailure


class HandlerCancellationError(RuntimeError):
    def __init__(self, index: int) -> None:
        self.index = index
        super().__init__("a handler raised CancelledError without task cancellation")


def _bounded_queue_integer(
    value: object,
    *,
    name: str,
    maximum: int,
) -> int:
    if type(value) is not int:
        raise TypeError(f"{name} must be an exact integer")
    if not 1 <= value <= maximum:
        raise ValueError(f"{name} is outside the supported range")
    return value


async def drain_async_queue_batch[T, R](
    items: tuple[T, ...],
    handler: Callable[[T], Awaitable[R]],
    *,
    worker_count: int,
    queue_capacity: int,
) -> tuple[WorkOutcome[R], ...]:
    if type(items) is not tuple:
        raise TypeError("items must be an exact tuple")
    if len(items) > _MAX_ITEMS:
        raise ValueError("item count exceeds the supported limit")
    if not callable(handler):
        raise TypeError("handler must be callable")
    checked_worker_count = _bounded_queue_integer(
        worker_count,
        name="worker_count",
        maximum=_MAX_WORKERS,
    )
    checked_capacity = _bounded_queue_integer(
        queue_capacity,
        name="queue_capacity",
        maximum=_MAX_QUEUE_CAPACITY,
    )

    queue: asyncio.Queue[tuple[int, T]] = asyncio.Queue(maxsize=checked_capacity)
    outcomes: list[WorkOutcome[R] | None] = [None] * len(items)

    async def worker() -> None:
        while True:
            try:
                index, item = await queue.get()
            except asyncio.QueueShutDown:
                return
            try:
                try:
                    value = await handler(item)
                except asyncio.CancelledError:
                    task = asyncio.current_task()
                    if task is None or task.cancelling():
                        raise
                    raise HandlerCancellationError(index) from None
                except Exception as error:
                    outcomes[index] = WorkFailure(
                        index,
                        error.with_traceback(None),
                    )
                else:
                    outcomes[index] = WorkSuccess(index, value)
            finally:
                queue.task_done()

    async with asyncio.TaskGroup() as task_group:
        try:
            for _ in range(checked_worker_count):
                task_group.create_task(worker())
            for indexed_item in enumerate(items):
                await queue.put(indexed_item)
            queue.shutdown(immediate=False)
            await queue.join()
        except BaseException:
            queue.shutdown(immediate=True)
            raise

    if any(outcome is None for outcome in outcomes):
        raise AssertionError("a completed queue batch is missing an outcome")
    return tuple(outcome for outcome in outcomes if outcome is not None)
```

## Example

```python
async def run_example() -> tuple[WorkOutcome[int], ...]:
    calls: dict[int, int] = {}

    async def handle(value: int) -> int:
        calls[value] = calls.get(value, 0) + 1
        await asyncio.sleep(0)
        if value == 2:
            raise ValueError("item rejected")
        return value * 10

    outcomes = await drain_async_queue_batch(
        (1, 2, 3),
        handle,
        worker_count=2,
        queue_capacity=1,
    )
    assert calls == {1: 1, 2: 1, 3: 1}
    return outcomes


outcomes = asyncio.run(run_example())
assert outcomes[0] == WorkSuccess(0, 10)
assert isinstance(outcomes[1], WorkFailure)
assert outcomes[1].index == 1
assert type(outcomes[1].exception) is ValueError
assert outcomes[1].exception.__traceback__ is None
assert outcomes[2] == WorkSuccess(2, 30)
```

## Trade-offs and Limitations

The queue bounds pending items, but the exact input tuple and one outcome per
item remain materialized, so total memory is `O(n)`. Every item is invoked
exactly once on the successful-drain path; successful results and ordinary
failures occupy original input positions. The frozen failure wrapper contains
the original mutable exception with its traceback removed, and that exception
can still contain application data in its type, arguments, cause or context.

Normal completion calls graceful shutdown before `join()`, after which empty
workers exit on `QueueShutDown`. Outer cancellation, startup failure and fatal
worker exceptions trigger immediate shutdown and structured cleanup. Native
`TaskGroup` behavior preserves cancellation, directly re-raises
`KeyboardInterrupt` and `SystemExit`, and groups other simultaneous fatal
failures. A directly raised `CancelledError` without a cancellation request is
translated into a fatal handler-contract error. No completion order is
promised, and cancellation cannot roll back or stop external effects that a
handler already started.

## Related Snippets

<!-- catalog:related:start -->
- [Gather Async Results with Bounded Concurrency](gather-async-results-with-bounded-concurrency.md)
- [Run Bounded Thread Work by Priority and Submission Order](run-bounded-thread-work-by-priority-and-submission-order.md)
- [Collect a Bounded Async-Generator Prefix with Deterministic Closure](collect-a-bounded-async-generator-prefix-with-deterministic-closure.md)
<!-- catalog:related:end -->
