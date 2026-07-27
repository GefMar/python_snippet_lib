---
title: "Track Current and Peak Scoped Work with a Synchronized Counter"
snippet_type: pattern
use_cases:
  - concurrency-control
  - observability
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - run-bounded-thread-work-by-priority-and-submission-order.md
  - ../observability-operations/measure-and-freeze-elapsed-time-in-a-context.md
  - ../observability-operations/process-log-records-in-a-background-thread-with-queuelistener.md
---

# Track Current and Peak Scoped Work with a Synchronized Counter

## Idea and Problem

Count entered work scopes and retain their lifetime high-water mark without letting exceptions leave the current count inflated.

A plain increment followed by a distant decrement is easy to get wrong. An
exception, early return, or control-flow signal can skip the decrement, while
concurrent updates can lose changes. Put both operations behind one context
manager and serialize the small state transition with a lock. The resulting
snapshot is suitable for diagnostics because `current` describes active
scopes and `peak` never decreases during the counter's lifetime.

## When to Use

Use this pattern when threads in one process enter a clearly defined unit of
work and you need a cheap current/peak diagnostic. It fits bounded worker
pools, callback adapters, and other synchronous code where every operation can
be wrapped in a `with` statement.

Choose an explicit `maximum_active` that represents a defensive accounting
bound. This counter observes scopes; it is not a semaphore and does not wait
for capacity. An entry beyond the configured bound fails before changing the
state.

## Implementation

```python
from collections.abc import Iterator
from contextlib import contextmanager
from dataclasses import dataclass
from threading import Lock


_MAX_SUPPORTED_ACTIVE = 1_000_000


@dataclass(frozen=True, slots=True)
class WorkSnapshot:
    current: int
    peak: int


class ActiveWorkLimitError(RuntimeError):
    pass


class ScopedWorkCounter:
    def __init__(self, *, maximum_active: int) -> None:
        if type(maximum_active) is not int:
            raise TypeError("maximum_active must be an exact int")
        if not 1 <= maximum_active <= _MAX_SUPPORTED_ACTIVE:
            raise ValueError(
                "maximum_active must be between 1 and "
                f"{_MAX_SUPPORTED_ACTIVE}"
            )

        self._maximum_active = maximum_active
        self._current = 0
        self._peak = 0
        self._lock = Lock()

    @contextmanager
    def track(self) -> Iterator[None]:
        self._enter()
        try:
            yield
        finally:
            self._leave()

    def snapshot(self) -> WorkSnapshot:
        with self._lock:
            return WorkSnapshot(current=self._current, peak=self._peak)

    def _enter(self) -> None:
        with self._lock:
            if self._current >= self._maximum_active:
                raise ActiveWorkLimitError("active work bound reached")

            self._current += 1
            self._peak = max(self._peak, self._current)

    def _leave(self) -> None:
        with self._lock:
            if self._current == 0:
                raise RuntimeError("scoped work counter underflow")
            self._current -= 1
```

## Example

```python
counter = ScopedWorkCounter(maximum_active=2)
original_error = LookupError("example failure")

try:
    with counter.track():
        with counter.track():
            assert counter.snapshot() == WorkSnapshot(current=2, peak=2)
        raise original_error
except LookupError as caught_error:
    assert caught_error is original_error

assert counter.snapshot() == WorkSnapshot(current=0, peak=2)

try:
    with counter.track():
        with counter.track():
            with counter.track():
                raise AssertionError("unreachable")
except ActiveWorkLimitError:
    pass

assert counter.snapshot() == WorkSnapshot(current=0, peak=2)
```

## Trade-offs and Limitations

Every entry, exit, and snapshot takes the same lock. The critical section is
constant-size, but very hot paths can still contend. The configured bound is
a defensive invariant rather than backpressure: callers that need waiting or
fair admission should use a semaphore or a scheduler instead.

The values cover one counter object in one process. They do not measure queue
depth, CPU usage, request health, or work abandoned by a crashed process.
Combining snapshots from multiple processes requires a separate aggregation
contract. A leaked context that never exits also remains counted; the pattern
cannot recover ownership after arbitrary process termination.

## Related Snippets

<!-- catalog:related:start -->
- [Run Bounded Thread Work by Priority and Submission Order](run-bounded-thread-work-by-priority-and-submission-order.md)
- [Measure and Freeze Elapsed Time in a Context](../observability-operations/measure-and-freeze-elapsed-time-in-a-context.md)
- [Process Log Records in a Background Thread with QueueListener](../observability-operations/process-log-records-in-a-background-thread-with-queuelistener.md)
<!-- catalog:related:end -->
