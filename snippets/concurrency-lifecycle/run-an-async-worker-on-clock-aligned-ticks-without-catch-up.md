---
title: "Run an Async Worker on Clock-Aligned Ticks Without Catch-Up"
snippet_type: pattern
use_cases:
  - automation
  - concurrency-control
  - lifecycle-management
tested_python:
  - "3.14"
dependencies: []
related:
  - stop-a-polling-worker-cooperatively-with-an-event.md
  - gather-async-results-with-bounded-concurrency.md
  - ../reliability-resilience/wait-for-a-predicate-until-a-monotonic-deadline.md
---

# Run an Async Worker on Clock-Aligned Ticks Without Catch-Up

## Idea and Problem

Run one asynchronous worker on shared wall-clock boundaries while skipping missed slots instead of replaying a burst after an overrun.

Each call receives the timestamp of its scheduled boundary. The loop waits for
the first future boundary, invokes at most one worker at a time, then computes
another future boundary from the current wall time. An interruptible stop event
ends idle waiting, and the returned report makes skipped slots visible.

## When to Use

Use this pattern for cooperative in-process maintenance whose executions must
align across processes or hosts, such as a lightweight sample or refresh every
minute. The worker must tolerate occasional skipped slots, and the system wall
clock must be an acceptable source of alignment. Use a durable scheduler when
every occurrence must run, survives process restarts, or needs distributed
ownership.

## Implementation

```python
import asyncio
import math
import time
from collections.abc import Awaitable, Callable
from dataclasses import dataclass


@dataclass(frozen=True, slots=True)
class TickRunReport:
    completed: int
    skipped: int


async def _wait_or_stop(delay: float, stop: asyncio.Event) -> bool:
    if stop.is_set():
        return True
    try:
        await asyncio.wait_for(stop.wait(), timeout=delay)
    except TimeoutError:
        return False
    return True


async def run_aligned(
    worker: Callable[[float], Awaitable[None]],
    *,
    period: float,
    stop: asyncio.Event,
    phase: float = 0.0,
    max_ticks: int = 1_000_000,
    clock: Callable[[], float] = time.time,
    wait_or_stop: Callable[[float, asyncio.Event], Awaitable[bool]] = _wait_or_stop,
) -> TickRunReport:
    if not isinstance(period, (int, float)) or isinstance(period, bool):
        raise TypeError("period must be a real number")
    try:
        period = float(period)
    except OverflowError as error:
        raise ValueError("period must be finite") from error
    if not math.isfinite(period) or not 0.001 <= period <= 86_400.0:
        raise ValueError("period must be finite and between 1 ms and one day")
    if not isinstance(phase, (int, float)) or isinstance(phase, bool):
        raise TypeError("phase must be a real number")
    try:
        phase = float(phase)
    except OverflowError as error:
        raise ValueError("phase must be finite") from error
    if not math.isfinite(phase) or not 0.0 <= phase < period:
        raise ValueError("phase must be finite and in the half-open period")
    if isinstance(max_ticks, bool) or not isinstance(max_ticks, int):
        raise TypeError("max_ticks must be an integer")
    if not 1 <= max_ticks <= 1_000_000:
        raise ValueError("max_ticks must be between 1 and 1,000,000")

    def checked_now() -> float:
        now = clock()
        if not isinstance(now, (int, float)) or isinstance(now, bool):
            raise TypeError("clock must return a real number")
        try:
            now = float(now)
        except OverflowError as error:
            raise ValueError("clock returned a non-finite value") from error
        if not math.isfinite(now):
            raise ValueError("clock returned a non-finite value")
        return now

    now = checked_now()
    cycle_position = (now - phase) / period
    if not math.isfinite(cycle_position):
        raise OverflowError("the clock cannot be represented at this period")
    cycle = math.floor(cycle_position) + 1
    scheduled = phase + cycle * period
    if not math.isfinite(scheduled) or scheduled <= now:
        raise OverflowError("the next boundary is not representable")

    completed = 0
    skipped = 0
    while completed < max_ticks and not stop.is_set():
        while True:
            now = checked_now()
            delay = scheduled - now
            if delay > 0.0:
                stopped = await wait_or_stop(delay, stop)
                if not isinstance(stopped, bool):
                    raise TypeError("wait_or_stop must return bool")
                if stopped or stop.is_set():
                    return TickRunReport(completed=completed, skipped=skipped)
                continue

            full_periods_late = math.floor((now - scheduled) / period)
            if full_periods_late >= 1:
                missed = full_periods_late + 1
                skipped += missed
                next_scheduled = scheduled + missed * period
                if (
                    not math.isfinite(next_scheduled)
                    or next_scheduled <= scheduled
                    or next_scheduled <= now
                ):
                    raise OverflowError("the next boundary is not representable")
                scheduled = next_scheduled
                continue
            break

        await worker(scheduled)
        completed += 1
        if completed >= max_ticks or stop.is_set():
            break

        now = checked_now()
        periods_to_next = 1
        if now >= scheduled:
            periods_to_next = math.floor((now - scheduled) / period) + 1
        skipped += periods_to_next - 1
        next_scheduled = scheduled + periods_to_next * period
        if (
            not math.isfinite(next_scheduled)
            or next_scheduled <= scheduled
            or next_scheduled <= now
        ):
            raise OverflowError("the next boundary is not representable")
        scheduled = next_scheduled

    return TickRunReport(completed=completed, skipped=skipped)
```

## Example

```python
class FakeClock:
    def __init__(self, value: float) -> None:
        self.value = value

    def now(self) -> float:
        return self.value

    async def wait(self, delay: float, stop: asyncio.Event) -> bool:
        self.value += delay
        await asyncio.sleep(0)
        return stop.is_set()


async def run_example() -> tuple[list[float], TickRunReport]:
    fake = FakeClock(2.0)
    stop = asyncio.Event()
    calls: list[float] = []

    async def worker(scheduled: float) -> None:
        calls.append(scheduled)
        if len(calls) == 1:
            fake.value += 25.0
        else:
            stop.set()

    report = await run_aligned(
        worker,
        period=10.0,
        stop=stop,
        max_ticks=5,
        clock=fake.now,
        wait_or_stop=fake.wait,
    )
    return calls, report


assert asyncio.run(run_example()) == (
    [10.0, 40.0],
    TickRunReport(completed=2, skipped=2),
)
```

## Trade-offs and Limitations

This is a wall-clock scheduler, so manual time changes or synchronization jumps
can lengthen a wait or skip boundaries. The default timeout wait is monotonic,
but wall time is resampled after every wait and worker call. Stopping during a
worker is observed only after that call returns. Worker exceptions and task
cancellation propagate; there are no retries, overlap, persistence, or replay.
The loop uses `O(1)` state. Injected clocks and waits make finite tests
deterministic, but production substitutes must preserve the same stop and
timing semantics.

## Related Snippets

<!-- catalog:related:start -->
- [Stop a Polling Worker Cooperatively with an Event](stop-a-polling-worker-cooperatively-with-an-event.md)
- [Gather Async Results with Bounded Concurrency](gather-async-results-with-bounded-concurrency.md)
- [Wait for a Predicate Until a Monotonic Deadline](../reliability-resilience/wait-for-a-predicate-until-a-monotonic-deadline.md)
<!-- catalog:related:end -->
