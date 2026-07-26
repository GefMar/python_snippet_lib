---
title: "Measure and Freeze Elapsed Time in a Context"
snippet_type: idiom
use_cases:
  - observability
  - performance-optimization
tested_python:
  - "3.14"
dependencies: []
related:
  - count-values-in-fixed-upper-bound-bins.md
  - scope-structured-log-fields-with-context-variables.md
---

# Measure and Freeze Elapsed Time in a Context

## Idea and Problem

Measure intermediate and final elapsed durations from one monotonic clock while freezing the final value when a context exits.

The timer is both a single-use context manager and a callable. Calls inside the
context sample the clock; calls after exit return the stored final duration.
The exit path records that duration even when the protected operation raises,
without suppressing the original exception.

## When to Use

Use this idiom when one operation needs checkpoints as well as a stable total
duration for logging or metrics. The clock must have monotonic,
`time.perf_counter()`-compatible semantics and return seconds as a finite
number. Inject a deterministic clock in tests. Use a tracing or metrics system
when timing must span processes, survive restarts, carry context across async
tasks, or aggregate distributions.

## Implementation

```python
import math
from collections.abc import Callable
from time import perf_counter
from types import TracebackType


def _read_finite_time(clock: Callable[[], int | float]) -> float:
    value = clock()
    if isinstance(value, bool) or not isinstance(value, (int, float)):
        raise TypeError("clock must return an integer or float")
    try:
        converted = float(value)
    except OverflowError as error:
        raise ValueError("clock result must be representable as a float") from error
    if not math.isfinite(converted):
        raise ValueError("clock result must be finite")
    return converted


class ElapsedTimer:
    def __init__(
        self,
        *,
        clock: Callable[[], int | float] = perf_counter,
    ) -> None:
        if not callable(clock):
            raise TypeError("clock must be callable")
        self._clock = clock
        self._started_at: float | None = None
        self._finished_at: float | None = None
        self._entered = False

    def __enter__(self) -> "ElapsedTimer":
        if self._entered:
            raise RuntimeError("timer instances are single-use")
        self._started_at = _read_finite_time(self._clock)
        self._entered = True
        return self

    def __call__(self) -> float:
        if not self._entered or self._started_at is None:
            raise RuntimeError("timer has not been entered")
        endpoint = (
            self._finished_at
            if self._finished_at is not None
            else _read_finite_time(self._clock)
        )
        if endpoint < self._started_at:
            raise ValueError("clock must not move backwards")
        return endpoint - self._started_at

    def __exit__(
        self,
        exception_type: type[BaseException] | None,
        exception: BaseException | None,
        traceback: TracebackType | None,
    ) -> bool:
        if not self._entered or self._started_at is None:
            raise RuntimeError("timer has not been entered")
        if self._finished_at is not None:
            raise RuntimeError("timer context has already exited")
        finished_at = _read_finite_time(self._clock)
        if finished_at < self._started_at:
            raise ValueError("clock must not move backwards")
        self._finished_at = finished_at
        return False
```

## Example

```python
ticks = iter([10.0, 10.25, 10.75, 11.0])
timer = ElapsedTimer(clock=lambda: next(ticks))

try:
    timer()
except RuntimeError:
    pre_entry_rejected = True
else:
    pre_entry_rejected = False

with timer as elapsed:
    first_checkpoint = elapsed()
    second_checkpoint = elapsed()

frozen_total = elapsed()

try:
    with timer:
        pass
except RuntimeError:
    reuse_rejected = True
else:
    reuse_rejected = False


class OperationFailed(Exception):
    pass


failure_ticks = iter([20.0, 20.5])
failed_timer = ElapsedTimer(clock=lambda: next(failure_ticks))
try:
    with failed_timer:
        raise OperationFailed
except OperationFailed:
    exception_propagated = True
else:
    exception_propagated = False

assert (
    pre_entry_rejected,
    first_checkpoint,
    second_checkpoint,
    frozen_total,
    reuse_rejected,
    exception_propagated,
    failed_timer(),
) == (True, 0.25, 0.75, 1.0, True, True, 0.5)
```

## Trade-offs and Limitations

Each instance measures one context only and is not thread-safe. It reports
wall-clock-style elapsed time from a monotonic performance counter, not CPU
time, and does not subtract suspension or scheduling delay. A failing injected
clock during `__exit__` can replace an exception from the protected block;
production clocks should therefore be dependable and side-effect free. The
timer does not emit logs, metrics, traces, labels, or histograms by itself, and
its values are meaningful only within the clock's local process lifetime.

## Related Snippets

<!-- catalog:related:start -->
- [Count Values in Fixed Upper-Bound Bins](count-values-in-fixed-upper-bound-bins.md)
- [Scope Structured Log Fields with Context Variables](scope-structured-log-fields-with-context-variables.md)
<!-- catalog:related:end -->
