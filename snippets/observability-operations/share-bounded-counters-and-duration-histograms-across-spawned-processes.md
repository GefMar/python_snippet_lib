---
title: "Share Bounded Counters and Duration Histograms Across Spawned Processes"
snippet_type: pattern
use_cases:
  - concurrency-control
  - observability
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - ../concurrency-lifecycle/track-current-and-peak-scoped-work-with-a-synchronized-counter.md
  - measure-and-freeze-elapsed-time-in-a-context.md
  - measure-cache-hit-ratios-from-monotonic-counter-snapshots.md
---

# Share Bounded Counters and Duration Histograms Across Spawned Processes

## Idea and Problem

Keep small host-local counters and fixed duration buckets coherent when several related Python processes update them.

Ordinary integers and lists are copied when a process starts, so later updates
do not reach the parent. Create the scalar and array storage from one explicit
`multiprocessing` context before workers start, and protect a complete update
or snapshot with the same process-shared lock. The histogram uses inclusive
upper bounds plus one overflow bucket, making every finite non-negative
duration belong to exactly one bucket.

## When to Use

Use this pattern for a bounded process family on one host when a parent needs a
lightweight diagnostic snapshot without a separate metrics service. It fits a
small worker pool whose process objects and shared state have the same owner.
Choose the start method explicitly and pass the state only while constructing
children from that context.

Keep the bucket count small and choose bounds before work begins. Use an
external collector for unrelated processes, containers, multiple hosts,
long-lived durable totals, labels, quantiles, or high-cardinality metrics.

## Implementation

```python
import bisect
import math
import time
from collections.abc import Callable, Iterator
from contextlib import contextmanager
from dataclasses import dataclass
from itertools import pairwise
from multiprocessing.context import BaseContext

_MAX_BUCKETS = 16
_MAX_DURATION_SECONDS = 86_400.0
_MAX_DURATION_SUM = 1_000_000_000_000.0
_MAX_INCREMENT = 1_000_000
_SIGNED_64_MAX = (1 << 63) - 1


def _finite_float(value: object, *, name: str) -> float:
    if isinstance(value, bool) or not isinstance(value, (int, float)):
        raise TypeError(f"{name} must be numeric")
    try:
        normalized = float(value)
    except (OverflowError, TypeError, ValueError) as error:
        raise ValueError(f"{name} must fit in a finite float") from error
    if not math.isfinite(normalized):
        raise ValueError(f"{name} must be finite")
    return normalized


def _validated_bounds(value: object) -> tuple[float, ...]:
    if type(value) is not tuple:
        raise TypeError("upper_bounds must be an exact tuple")
    if not 1 <= len(value) <= _MAX_BUCKETS:
        raise ValueError("upper_bounds count is outside the supported range")
    bounds = tuple(_finite_float(bound, name="duration upper bound") for bound in value)
    if bounds[0] <= 0.0 or bounds[-1] > _MAX_DURATION_SECONDS:
        raise ValueError("duration upper bounds are outside the supported range")
    if any(left >= right for left, right in pairwise(bounds)):
        raise ValueError("duration upper bounds must be strictly increasing")
    return bounds


@dataclass(frozen=True, slots=True)
class ProcessMetricSnapshot:
    event_count: int
    observation_count: int
    duration_sum: float
    upper_bounds: tuple[float, ...]
    bucket_counts: tuple[int, ...]


class ProcessMetricState:
    def __init__(
        self,
        context: BaseContext,
        *,
        upper_bounds: tuple[float, ...],
    ) -> None:
        if not isinstance(context, BaseContext):
            raise TypeError("context must be a multiprocessing context")
        self._upper_bounds = _validated_bounds(upper_bounds)
        self._lock = context.RLock()
        self._event_count = context.RawValue("q", 0)
        self._observation_count = context.RawValue("q", 0)
        self._duration_sum = context.RawValue("d", 0.0)
        self._bucket_counts = context.RawArray(
            "q",
            len(self._upper_bounds) + 1,
        )

    def increment_events(self, amount: int = 1) -> int:
        if type(amount) is not int:
            raise TypeError("amount must be an exact integer")
        if not 1 <= amount <= _MAX_INCREMENT:
            raise ValueError("amount is outside the supported range")
        with self._lock:
            updated = self._event_count.value + amount
            if updated > _SIGNED_64_MAX:
                raise OverflowError("event counter would overflow signed 64-bit storage")
            self._event_count.value = updated
            return updated

    def observe_duration(self, seconds: float) -> None:
        duration = _finite_float(seconds, name="duration")
        if not 0.0 <= duration <= _MAX_DURATION_SECONDS:
            raise ValueError("duration is outside the supported range")
        bucket_index = bisect.bisect_left(self._upper_bounds, duration)

        with self._lock:
            observations = self._observation_count.value
            bucket_count = self._bucket_counts[bucket_index]
            if observations == _SIGNED_64_MAX or bucket_count == _SIGNED_64_MAX:
                raise OverflowError("duration count would overflow signed 64-bit storage")
            duration_sum = self._duration_sum.value + duration
            if not math.isfinite(duration_sum) or duration_sum > _MAX_DURATION_SUM:
                raise OverflowError("duration sum exceeds the supported range")

            self._observation_count.value = observations + 1
            self._bucket_counts[bucket_index] = bucket_count + 1
            self._duration_sum.value = duration_sum

    @contextmanager
    def measure(
        self,
        *,
        clock: Callable[[], float] = time.perf_counter,
    ) -> Iterator[None]:
        started = _finite_float(clock(), name="clock value")
        try:
            yield
        except BaseException as body_error:
            try:
                self._record_elapsed(started, clock)
            except BaseException as metric_error:
                body_error.add_note(f"duration recording failed with {type(metric_error).__name__}")
            raise
        else:
            self._record_elapsed(started, clock)

    def _record_elapsed(
        self,
        started: float,
        clock: Callable[[], float],
    ) -> None:
        finished = _finite_float(clock(), name="clock value")
        if finished < started:
            raise RuntimeError("performance clock moved backward")
        self.observe_duration(finished - started)

    def snapshot(self) -> ProcessMetricSnapshot:
        with self._lock:
            return ProcessMetricSnapshot(
                event_count=self._event_count.value,
                observation_count=self._observation_count.value,
                duration_sum=self._duration_sum.value,
                upper_bounds=self._upper_bounds,
                bucket_counts=tuple(self._bucket_counts),
            )
```

## Example

```python
import math
import multiprocessing
import time


def record_batch(
    state: ProcessMetricState,
    durations: tuple[float, ...],
) -> None:
    for duration in durations:
        state.increment_events()
        state.observe_duration(duration)


def run_process_example() -> bool:
    context = multiprocessing.get_context("spawn")
    state = ProcessMetricState(
        context,
        upper_bounds=(0.1, 0.5, 1.0),
    )
    workers = (
        context.Process(target=record_batch, args=(state, (0.05, 0.5))),
        context.Process(target=record_batch, args=(state, (0.7, 1.2))),
    )
    for worker in workers:
        worker.start()

    deadline = time.monotonic() + 10.0
    for worker in workers:
        worker.join(max(0.0, deadline - time.monotonic()))
    for worker in workers:
        if worker.is_alive():
            worker.terminate()
            worker.join(1.0)
        if worker.is_alive():
            worker.kill()
            worker.join(1.0)
    if any(worker.exitcode != 0 for worker in workers):
        return False

    fake_values = iter((20.0, 20.2))
    with state.measure(clock=lambda: next(fake_values)):
        pass
    snapshot = state.snapshot()
    return (
        snapshot.event_count == 4
        and snapshot.observation_count == 5
        and snapshot.bucket_counts == (1, 2, 1, 1)
        and math.isclose(snapshot.duration_sum, 2.65)
    )


if multiprocessing.current_process().name == "MainProcess":
    example_passed = run_process_example()
else:
    example_passed = True

assert example_passed
```

## Trade-offs and Limitations

Every mutation and snapshot takes one interprocess lock. That gives a coherent
multi-field view but can become a bottleneck on hot paths. Floating-point sums
also depend slightly on update order; bucket counts are the stable aggregation
when exact decimal totals matter. The timer preserves a body exception and
adds a note if metric recording itself fails, so diagnostics never replace the
original failure.

The state is meaningful only for related processes that inherited or received
the same synchronization primitives during process construction. It does not
discover workers, start or reap them, survive a parent restart, aggregate
across hosts, or recover increments lost when a process dies before taking the
lock. A caller must use one context consistently and own bounded child cleanup.

## Related Snippets

<!-- catalog:related:start -->
- [Track Current and Peak Scoped Work with a Synchronized Counter](../concurrency-lifecycle/track-current-and-peak-scoped-work-with-a-synchronized-counter.md)
- [Measure and Freeze Elapsed Time in a Context](measure-and-freeze-elapsed-time-in-a-context.md)
- [Measure Cache Hit Ratios from Monotonic Counter Snapshots](measure-cache-hit-ratios-from-monotonic-counter-snapshots.md)
<!-- catalog:related:end -->
