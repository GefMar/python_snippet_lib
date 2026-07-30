---
title: "Correlate Bounded Async Results by Original Task as They Complete"
snippet_type: pattern
use_cases:
  - concurrency-control
  - lifecycle-management
  - resource-management
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - gather-async-results-with-bounded-concurrency.md
  - race-a-preferred-async-read-against-bounded-alternatives.md
  - collect-thread-pool-results-and-errors-as-futures-complete.md
---

# Correlate Bounded Async Results by Original Task as They Complete

## Idea and Problem

Collect a bounded set of already-started tasks in completion order while retaining each original task's stable label and cleaning up every owned task on all exits.

Asynchronous iteration over `asyncio.as_completed()` yields the originally
supplied task objects on Python 3.13 or later. A task-to-label map can therefore
correlate each result without wrapping the awaitable or relying on completion
order. A timeout stops observation but does not cancel pending tasks, so the
collector must cancel and drain its complete owned set before returning or
raising.

## When to Use

Use this pattern inside a running event loop for an exact tuple of 1 through 32
unique, already-created, still-pending `asyncio.Task` objects from that same
loop. Give each task a unique conservative ASCII label and use completion-order
outcomes when early processing or task identity matters more than input order.

The caller owns every task until the complete validation pass succeeds. After
that point the collector owns all of them through completion or cleanup. Use a
`TaskGroup` when lexical fail-fast structure matters, bounded workers when all
jobs should not start at once, and an async iterator or queue for an unbounded
stream.

## Implementation

```python
import asyncio
import math
import re
from dataclasses import dataclass

_MAX_ASYNC_TASKS = 32
_MAX_TIMEOUT_SECONDS = 300.0
_TASK_LABEL = re.compile(r"[a-z][a-z0-9._-]{0,63}", re.ASCII)


@dataclass(frozen=True, slots=True)
class AsyncTaskOutcome[ResultT]:
    label: str
    succeeded: bool
    value: ResultT | None
    error: Exception | None


async def _cancel_and_drain_tasks(
    tasks: tuple[asyncio.Task[object], ...],
) -> None:
    for task in tasks:
        if not task.done():
            task.cancel()

    drain = asyncio.gather(*tasks, return_exceptions=True)
    interrupted = False
    while not drain.done():
        try:
            await asyncio.shield(drain)
        except asyncio.CancelledError:
            interrupted = True
    if interrupted:
        raise asyncio.CancelledError


async def collect_tasks_as_completed[ResultT](
    entries: tuple[tuple[str, asyncio.Task[ResultT]], ...],
    *,
    timeout: int | float,
) -> tuple[AsyncTaskOutcome[ResultT], ...]:
    if type(entries) is not tuple:
        raise TypeError("entries must be an exact tuple")
    if not 1 <= len(entries) <= _MAX_ASYNC_TASKS:
        raise ValueError("task count is outside the supported range")
    if type(timeout) not in (int, float):
        raise TypeError("timeout must be an exact integer or float")
    try:
        timeout_seconds = float(timeout)
    except OverflowError as error:
        raise ValueError("timeout must be representable as a float") from error
    if (
        not math.isfinite(timeout_seconds)
        or not 0.0 <= timeout_seconds <= _MAX_TIMEOUT_SECONDS
    ):
        raise ValueError("timeout is outside the supported range")

    loop = asyncio.get_running_loop()
    current_task = asyncio.current_task()
    labels: set[str] = set()
    tasks: set[asyncio.Task[ResultT]] = set()
    validated: list[tuple[str, asyncio.Task[ResultT]]] = []
    for index, entry in enumerate(entries):
        if type(entry) is not tuple:
            raise TypeError(f"entries[{index}] must be an exact tuple")
        if len(entry) != 2:
            raise ValueError(f"entries[{index}] must contain a label and task")
        label, task = entry
        if type(label) is not str:
            raise TypeError(f"entries[{index}].label must be an exact string")
        if _TASK_LABEL.fullmatch(label) is None:
            raise ValueError(f"entries[{index}].label has an invalid format")
        if label in labels:
            raise ValueError("task labels must be unique")
        if not isinstance(task, asyncio.Task):
            raise TypeError(f"entries[{index}].task must be an asyncio.Task")
        if task.get_loop() is not loop:
            raise ValueError(f"entries[{index}].task belongs to another event loop")
        if task is current_task:
            raise ValueError("the collector cannot own its current task")
        if task.done():
            raise ValueError(f"entries[{index}].task must still be pending")
        if task in tasks:
            raise ValueError("task objects must be unique")
        labels.add(label)
        tasks.add(task)
        validated.append((label, task))

    # Ownership transfers only after every entry and option has been validated.
    owned_tasks = tuple(task for _, task in validated)
    label_by_task = {task: label for label, task in validated}
    outcomes: list[AsyncTaskOutcome[ResultT]] = []
    try:
        async for completed in asyncio.as_completed(
            owned_tasks,
            timeout=timeout_seconds,
        ):
            if completed not in label_by_task:
                raise RuntimeError("as_completed returned an unknown task")
            label = label_by_task[completed]
            try:
                value = completed.result()
            except Exception as error:
                outcomes.append(
                    AsyncTaskOutcome(
                        label=label,
                        succeeded=False,
                        value=None,
                        error=error,
                    )
                )
            else:
                outcomes.append(
                    AsyncTaskOutcome(
                        label=label,
                        succeeded=True,
                        value=value,
                        error=None,
                    )
                )
    finally:
        await _cancel_and_drain_tasks(owned_tasks)

    return tuple(outcomes)
```

## Example

```python
async def run_example() -> tuple[tuple[str, ...], bool, bool, bool]:
    release_failure = asyncio.Event()
    release_success = asyncio.Event()

    async def fail_first() -> str:
        await release_failure.wait()
        raise ValueError("expected failure")

    async def succeed_second() -> str:
        await release_success.wait()
        return "ready"

    failed_task = asyncio.create_task(fail_first(), name="example:failed")
    successful_task = asyncio.create_task(
        succeed_second(),
        name="example:successful",
    )

    async def release_in_order() -> None:
        release_failure.set()
        try:
            await failed_task
        except ValueError:
            pass
        release_success.set()

    releaser = asyncio.create_task(release_in_order(), name="example:releaser")
    outcomes = await collect_tasks_as_completed(
        (("failed", failed_task), ("successful", successful_task)),
        timeout=1,
    )
    await releaser

    cleanup_started = asyncio.Event()
    cleanup_finished = asyncio.Event()
    never_release = asyncio.Event()

    async def remain_pending() -> str:
        cleanup_started.set()
        try:
            await never_release.wait()
            return "unreachable"
        finally:
            cleanup_finished.set()

    pending_task = asyncio.create_task(remain_pending(), name="example:pending")
    await cleanup_started.wait()
    try:
        await collect_tasks_as_completed(
            (("pending", pending_task),),
            timeout=0,
        )
    except TimeoutError:
        timeout_reported = True
    else:
        timeout_reported = False

    return (
        tuple(outcome.label for outcome in outcomes),
        isinstance(outcomes[0].error, ValueError),
        timeout_reported,
        pending_task.cancelled() and cleanup_finished.is_set(),
    )


assert asyncio.run(run_example()) == (
    ("failed", "successful"),
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

Validation and collection use `O(T)` time and memory for at most 32 tasks.
Every task is already scheduled, so this pattern bounds bookkeeping but not
concurrency, work admission, or downstream load. Completion order is
intentionally nondeterministic unless the tasks' own protocol imposes an order.
Returned failure outcomes retain their exception objects and tracebacks until
the caller releases them.

Only ordinary `Exception` instances become outcomes. A task cancellation,
process-control exception, caller cancellation, timeout, identity violation,
or other internal failure propagates after every owned unfinished task is
cancelled and drained. `asyncio.as_completed()` does not cancel pending tasks
when its timeout expires. Cleanup can still wait indefinitely if an owned task
suppresses cancellation or blocks the event-loop thread, so all tasks must be
trusted, finite, and cancellation-cooperative.

The identity guarantee depends on asynchronous iteration introduced in Python
3.13. Plain iteration over `as_completed()` yields wrapper coroutines instead
and does not provide this correlation contract. The collector accepts only
tasks from its current loop, rejects tasks that are already done, and performs
no retries, per-task deadlines, result sorting, serialization, or cross-loop
coordination. A validation failure leaves every supplied task under the
caller's ownership.

## Related Snippets

<!-- catalog:related:start -->
- [Gather Async Results with Bounded Concurrency](gather-async-results-with-bounded-concurrency.md)
- [Race a Preferred Async Read Against Bounded Alternatives](race-a-preferred-async-read-against-bounded-alternatives.md)
- [Collect Thread-Pool Results and Errors as Futures Complete](collect-thread-pool-results-and-errors-as-futures-complete.md)
<!-- catalog:related:end -->
