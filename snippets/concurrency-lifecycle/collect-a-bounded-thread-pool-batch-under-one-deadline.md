---
title: "Collect a Bounded Thread-Pool Batch Under One Deadline"
snippet_type: pattern
use_cases:
  - concurrency-control
  - lifecycle-management
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - collect-thread-pool-results-and-errors-as-futures-complete.md
  - stop-a-polling-worker-cooperatively-with-an-event.md
  - ../reliability-resilience/wait-for-a-predicate-until-a-monotonic-deadline.md
---

# Collect a Bounded Thread-Pool Batch Under One Deadline

## Idea and Problem

Observe a bounded batch of thread-pool calls under one cumulative deadline while preserving input-ordered successes, failures, and unfinished work.

The caller supplies and retains ownership of the executor. The helper starts
one monotonic deadline before submission, waits only for its remaining budget,
and freezes the resulting state into three tuples. Calls still unfinished at
the deadline are never inspected for a result; cancellation is merely
attempted before they are reported.

## When to Use

Use this pattern when 1-64 independent, in-process calls may complete or fail
at different times and a partial report is more useful than discarding the
whole batch at a shared deadline. Each call needs a unique stable name, must be
safe to run concurrently, and should arrange its own cooperative stopping
mechanism if it may continue beyond the collection budget.

The executor must have a lifecycle longer than this helper call. Use a process
boundary when work must be terminated independently, or another concurrency
model when its cancellation semantics are the actual requirement.

## Implementation

```python
import math
import re
import time
from collections.abc import Callable
from concurrent.futures import ThreadPoolExecutor, wait
from dataclasses import dataclass
from typing import Generic, TypeVar


ResultT = TypeVar("ResultT")
_CALL_NAME = re.compile(r"[a-z][a-z0-9_-]{0,63}", re.ASCII)
_MAX_CALLS = 64
_MAX_TIMEOUT_SECONDS = 86_400.0


@dataclass(frozen=True, slots=True)
class NamedThreadCall(Generic[ResultT]):
    name: str
    call: Callable[[], ResultT]


@dataclass(frozen=True, slots=True)
class ThreadCallSuccess(Generic[ResultT]):
    name: str
    value: ResultT


@dataclass(frozen=True, slots=True)
class ThreadCallFailure:
    name: str
    error: Exception


@dataclass(frozen=True, slots=True)
class ThreadCallUnfinished:
    name: str
    cancelled_before_start: bool


@dataclass(frozen=True, slots=True)
class ThreadBatchReport(Generic[ResultT]):
    successful: tuple[ThreadCallSuccess[ResultT], ...]
    failed: tuple[ThreadCallFailure, ...]
    unfinished: tuple[ThreadCallUnfinished, ...]


def collect_thread_batch(
    executor: ThreadPoolExecutor,
    calls: tuple[NamedThreadCall[ResultT], ...],
    *,
    timeout: int | float,
) -> ThreadBatchReport[ResultT]:
    if type(executor) is not ThreadPoolExecutor:
        raise TypeError("executor must be an exact ThreadPoolExecutor")
    if type(calls) is not tuple:
        raise TypeError("calls must be an exact tuple")
    if not 1 <= len(calls) <= _MAX_CALLS:
        raise ValueError("call count is outside the supported range")

    names: set[str] = set()
    for item in calls:
        if type(item) is not NamedThreadCall:
            raise TypeError("calls must contain exact NamedThreadCall values")
        if type(item.name) is not str:
            raise TypeError("call names must be text")
        if _CALL_NAME.fullmatch(item.name) is None:
            raise ValueError("a call name is invalid")
        if item.name in names:
            raise ValueError("call names must be unique")
        if not callable(item.call):
            raise TypeError("every call must be callable")
        names.add(item.name)

    if isinstance(timeout, bool) or not isinstance(timeout, (int, float)):
        raise TypeError("timeout must be numeric")
    try:
        timeout_seconds = float(timeout)
    except OverflowError:
        raise ValueError("timeout must be representable as a float") from None
    if not math.isfinite(timeout_seconds) or timeout_seconds < 0:
        raise ValueError("timeout must be finite and non-negative")
    if timeout_seconds > _MAX_TIMEOUT_SECONDS:
        raise ValueError("timeout exceeds the supported duration")

    deadline = time.monotonic() + timeout_seconds
    futures = tuple(executor.submit(item.call) for item in calls)
    remaining = max(0.0, deadline - time.monotonic())
    _, unfinished_futures = wait(futures, timeout=remaining)
    cancellation_results = {
        future: future.cancel()
        for future in futures
        if future in unfinished_futures
    }

    successful: list[ThreadCallSuccess[ResultT]] = []
    failed: list[ThreadCallFailure] = []
    unfinished: list[ThreadCallUnfinished] = []
    for item, future in zip(calls, futures):
        if future in unfinished_futures:
            unfinished.append(
                ThreadCallUnfinished(
                    name=item.name,
                    cancelled_before_start=cancellation_results[future],
                )
            )
            continue
        try:
            value = future.result()
        except Exception as error:
            failed.append(ThreadCallFailure(item.name, error))
        else:
            successful.append(ThreadCallSuccess(item.name, value))

    return ThreadBatchReport(
        successful=tuple(successful),
        failed=tuple(failed),
        unfinished=tuple(unfinished),
    )
```

## Example

```python
from threading import Event


started = Event()
release = Event()


def wait_for_release() -> str:
    started.set()
    release.wait()
    return "late"


def reject_value() -> int:
    raise LookupError("rejected")


executor = ThreadPoolExecutor(max_workers=3, thread_name_prefix="batch-example")
try:
    report = collect_thread_batch(
        executor,
        (
            NamedThreadCall("held", wait_for_release),
            NamedThreadCall("ready", lambda: 21),
            NamedThreadCall("rejected", reject_value),
        ),
        timeout=0.1,
    )
    returned_while_running = started.is_set() and not release.is_set()
finally:
    release.set()
    executor.shutdown(wait=True, cancel_futures=True)

assert (
    tuple((item.name, item.value) for item in report.successful),
    tuple((item.name, type(item.error)) for item in report.failed),
    tuple(
        (item.name, item.cancelled_before_start)
        for item in report.unfinished
    ),
    returned_while_running,
) == (
    (("ready", 21),),
    (("rejected", LookupError),),
    (("held", False),),
    True,
)
```

## Trade-offs and Limitations

The deadline includes submission time, but it bounds only this helper's wait.
It does not bound callable runtime, later executor shutdown, or interpreter
exit. `Future.cancel()` can prevent queued work from starting; it returns
false for a running call, and Python provides no safe way to force-stop that
thread. A call that finishes just after the wait snapshot can therefore remain
reported as unfinished.

The report's dataclasses and category tuples are immutable, while successful
values and exception objects retain the mutability chosen by each callable.
Only `Exception` subclasses become failures; process-control exceptions still
propagate. Submission errors also propagate without a partial report, and any
already-submitted calls remain under the caller's executor ownership.

The helper eagerly submits the complete bounded batch. It provides no retry,
rate control, per-call deadline, priority policy, or mechanism for shutting
down the executor. Putting a caller-owned executor in a context manager
immediately around this call can still introduce a later shutdown wait; choose
that lifecycle deliberately.

## Related Snippets

<!-- catalog:related:start -->
- [Collect Thread-Pool Results and Errors as Futures Complete](collect-thread-pool-results-and-errors-as-futures-complete.md)
- [Stop a Polling Worker Cooperatively with an Event](stop-a-polling-worker-cooperatively-with-an-event.md)
- [Wait for a Predicate Until a Monotonic Deadline](../reliability-resilience/wait-for-a-predicate-until-a-monotonic-deadline.md)
<!-- catalog:related:end -->
