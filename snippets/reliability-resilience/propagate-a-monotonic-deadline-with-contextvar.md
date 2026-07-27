---
title: "Propagate a Monotonic Deadline with ContextVar"
snippet_type: pattern
use_cases:
  - concurrency-control
  - lifecycle-management
  - retry-recovery
tested_python:
  - "3.14"
dependencies: []
related:
  - wait-for-a-predicate-until-a-monotonic-deadline.md
  - ../concurrency-lifecycle/submit-a-callable-with-a-snapshot-of-the-current-context.md
  - ../configuration-serialization/parse-compact-durations-into-timedelta.md
---

# Propagate a Monotonic Deadline with ContextVar

## Idea and Problem

Carry one process-local monotonic deadline through nested calls so every outgoing operation consumes the same remaining time budget.

Each scope stores an absolute deadline in a `ContextVar`. A nested scope may
shorten its parent but cannot extend it, and an operation receives the smaller
of its configured timeout and the remaining shared budget. Exhaustion is
detected before the operation is invoked.

## When to Use

Use this pattern when several framework-neutral adapters participate in one
bounded operation and each already accepts a relative timeout. It works well
with asynchronous tasks created inside the scope because tasks capture their
creation context. Keep one `DeadlineContext` instance at the application
boundary and inject a monotonic clock for deterministic tests.

Pass explicit timeout arguments when call graphs are small or context would
hide an important API dependency. Use the framework's native cancellation or
deadline primitive when it already propagates and enforces equivalent
semantics.

## Implementation

```python
import math
import time
from collections.abc import Callable, Iterator
from contextlib import contextmanager
from contextvars import ContextVar
from typing import TypeVar


ResultT = TypeVar("ResultT")
_MAX_BUDGET_SECONDS = 86_400.0


class DeadlineError(RuntimeError):
    pass


class DeadlineNotSetError(DeadlineError):
    pass


class DeadlineExceededError(DeadlineError, TimeoutError):
    pass


def _budget(value: float, *, name: str) -> float:
    if isinstance(value, bool) or not isinstance(value, (int, float)):
        raise TypeError(f"{name} must be numeric")
    try:
        normalized = float(value)
    except OverflowError as error:
        raise ValueError(f"{name} must be finite") from error
    if (
        not math.isfinite(normalized)
        or not 0.0 < normalized <= _MAX_BUDGET_SECONDS
    ):
        raise ValueError(f"{name} must be positive and bounded")
    return normalized


class DeadlineContext:
    def __init__(self, clock: Callable[[], float] = time.monotonic) -> None:
        if not callable(clock):
            raise TypeError("clock must be callable")
        self._clock = clock
        self._deadline: ContextVar[float | None] = ContextVar(
            "monotonic_operation_deadline",
            default=None,
        )

    def _now(self) -> float:
        now = self._clock()
        if isinstance(now, bool) or not isinstance(now, (int, float)):
            raise TypeError("clock must return a number")
        normalized = float(now)
        if not math.isfinite(normalized):
            raise ValueError("clock must return a finite value")
        return normalized

    @contextmanager
    def after(self, timeout: float) -> Iterator[float]:
        requested = self._now() + _budget(timeout, name="timeout")
        if not math.isfinite(requested):
            raise ValueError("the resulting deadline must be finite")
        parent = self._deadline.get()
        effective = requested if parent is None else min(parent, requested)
        token = self._deadline.set(effective)
        try:
            yield effective
        finally:
            self._deadline.reset(token)

    def timeout_for(self, configured_timeout: float | None = None) -> float:
        deadline = self._deadline.get()
        if deadline is None:
            if configured_timeout is None:
                raise DeadlineNotSetError("no deadline scope is active")
            return _budget(
                configured_timeout,
                name="configured_timeout",
            )
        remaining = deadline - self._now()
        if remaining <= 0.0:
            raise DeadlineExceededError("the shared deadline is exhausted")
        if configured_timeout is None:
            return remaining
        configured = _budget(
            configured_timeout,
            name="configured_timeout",
        )
        return min(configured, remaining)

    def invoke(
        self,
        operation: Callable[[float], ResultT],
        *,
        configured_timeout: float,
    ) -> ResultT:
        if not callable(operation):
            raise TypeError("operation must be callable")
        timeout = self.timeout_for(configured_timeout)
        return operation(timeout)
```

## Example

```python
class FakeClock:
    def __init__(self) -> None:
        self.now = 100.0

    def monotonic(self) -> float:
        return self.now


clock = FakeClock()
deadlines = DeadlineContext(clock.monotonic)
observed_timeouts: list[float] = []


def outgoing_call(timeout: float) -> str:
    observed_timeouts.append(timeout)
    return "ok"


with deadlines.after(10.0) as outer_deadline:
    clock.now = 102.0
    first = deadlines.invoke(outgoing_call, configured_timeout=3.0)
    with deadlines.after(20.0) as non_extending_deadline:
        inherited = deadlines.timeout_for()
    with deadlines.after(1.0) as shorter_deadline:
        shortened = deadlines.timeout_for(5.0)

    clock.now = 110.0
    try:
        deadlines.invoke(outgoing_call, configured_timeout=1.0)
    except DeadlineExceededError:
        exhausted_before_call = True
    else:
        exhausted_before_call = False

try:
    deadlines.timeout_for()
except DeadlineNotSetError:
    scope_was_reset = True
else:
    scope_was_reset = False
outside_scope_timeout = deadlines.timeout_for(2.0)

assert (
    first,
    outer_deadline,
    non_extending_deadline,
    inherited,
    shorter_deadline,
    shortened,
    exhausted_before_call,
    observed_timeouts,
    scope_was_reset,
    outside_scope_timeout,
) == (
    "ok",
    110.0,
    110.0,
    8.0,
    103.0,
    1.0,
    True,
    [3.0],
    True,
    2.0,
)
```

## Trade-offs and Limitations

The context carries policy but does not interrupt an operation. Each adapter
must pass the returned relative timeout to an API that actually enforces it,
and an operation that ignores cancellation may still overrun. A timeout is a
snapshot of the remaining budget, so time spent between calculation and use
also consumes that budget.

Child asyncio tasks capture the context present when they are created; they
may therefore retain a deadline after the parent scope has syntactically
exited. Ordinary worker-thread inheritance depends on how the thread is
created and on the interpreter build; pass an explicit copied or empty context
instead of assuming a default. `asyncio.to_thread()` does propagate the current
context. The absolute monotonic value is meaningful only inside the same
process and must never be serialized as a cross-process or network deadline or
passed directly to an event-loop API that uses a different clock origin. The
clock must not move backward, and this helper provides no tracing, retry,
cleanup, or per-operation cancellation policy.

## Related Snippets

<!-- catalog:related:start -->
- [Wait for a Predicate Until a Monotonic Deadline](wait-for-a-predicate-until-a-monotonic-deadline.md)
- [Submit a Callable with a Snapshot of the Current Context](../concurrency-lifecycle/submit-a-callable-with-a-snapshot-of-the-current-context.md)
- [Parse Compact Durations into timedelta](../configuration-serialization/parse-compact-durations-into-timedelta.md)
<!-- catalog:related:end -->
