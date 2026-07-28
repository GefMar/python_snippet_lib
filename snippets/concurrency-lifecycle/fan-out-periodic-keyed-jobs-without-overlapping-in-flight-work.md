---
title: "Fan Out Periodic Keyed Jobs Without Overlapping In-Flight Work"
snippet_type: pattern
use_cases:
  - automation
  - concurrency-control
  - lifecycle-management
tested_python:
  - "3.14"
dependencies: []
related:
  - refresh-bounded-named-targets-at-their-nearest-monotonic-deadline.md
  - reuse-one-pending-future-across-non-cancelling-poll-timeouts.md
  - collect-thread-pool-results-and-errors-as-futures-complete.md
---

# Fan Out Periodic Keyed Jobs Without Overlapping In-Flight Work

## Idea and Problem

Own one persistent thread pool that submits each periodic key at most once until its current future reaches a terminal state.

A slow periodic callback should not build a queue of duplicate invocations.
Keep one future slot per fixed key, harvest all completed slots before each
admission pass, and schedule the next occurrence from the observed completion
time. Late ticks then coalesce naturally instead of replaying missed periods.

## When to Use

Use this owner for a small fixed set of trusted blocking callbacks that may run
concurrently across keys but must never overlap with themselves. Call `tick()`
from one lifecycle thread, inject a monotonic clock for deterministic tests,
and give every callback its own bounded I/O timeout so `close()` can finish.

Use an asynchronous task owner for coroutine-native work, a process boundary
for work that must be terminated, or a durable scheduler when occurrences must
survive restarts. Threads share process state and cannot isolate native hangs,
memory corruption, or process-wide side effects.

## Implementation

```python
import math
import re
import threading
import time
from collections.abc import Callable
from concurrent.futures import Future, ThreadPoolExecutor
from dataclasses import dataclass
from enum import StrEnum

_KEY = re.compile(r"[a-z][a-z0-9_-]{0,31}", re.ASCII).fullmatch
_ERROR_KIND = re.compile(r"[A-Za-z_][A-Za-z0-9_]{0,63}", re.ASCII).fullmatch
_MAX_JOBS = 32
_MAX_WORKERS = 16
_MIN_INTERVAL = 0.01
_MAX_INTERVAL = 86_400.0
_MAX_CLOCK_ABSOLUTE = 1_000_000_000_000.0


@dataclass(frozen=True, slots=True)
class PeriodicKeyedJob:
    key: str
    success_interval: float
    failure_interval: float
    callback: Callable[[], None]


class PeriodicOutcomeState(StrEnum):
    SUCCEEDED = "succeeded"
    FAILED = "failed"
    CANCELLED = "cancelled"


@dataclass(frozen=True, slots=True)
class PeriodicJobOutcome:
    key: str
    state: PeriodicOutcomeState
    error_kind: str | None


@dataclass(frozen=True, slots=True)
class FanoutTickReport:
    observed_at: float
    completed: tuple[PeriodicJobOutcome, ...]
    submitted: tuple[str, ...]
    in_flight: tuple[str, ...]


@dataclass(frozen=True, slots=True)
class FanoutCloseReport:
    finalized: tuple[PeriodicJobOutcome, ...]


def _interval(value: object, *, name: str) -> float:
    if isinstance(value, bool) or not isinstance(value, (int, float)):
        raise TypeError(f"{name} must be numeric")
    try:
        normalized = float(value)
    except (OverflowError, TypeError, ValueError) as error:
        raise ValueError(f"{name} must fit in a finite float") from error
    if not math.isfinite(normalized) or not _MIN_INTERVAL <= normalized <= _MAX_INTERVAL:
        raise ValueError(f"{name} is outside the supported range")
    return normalized


def _validated_jobs(value: object) -> tuple[PeriodicKeyedJob, ...]:
    if type(value) is not tuple:
        raise TypeError("jobs must be an exact tuple")
    if not 1 <= len(value) <= _MAX_JOBS:
        raise ValueError("job count is outside the supported range")

    keys: set[str] = set()
    validated: list[PeriodicKeyedJob] = []
    for job in value:
        if type(job) is not PeriodicKeyedJob:
            raise TypeError("jobs must contain exact PeriodicKeyedJob values")
        if type(job.key) is not str or _KEY(job.key) is None:
            raise ValueError("job keys must be conservative ASCII identifiers")
        if job.key in keys:
            raise ValueError("job keys must be unique")
        if not callable(job.callback):
            raise TypeError("job callbacks must be callable")
        keys.add(job.key)
        validated.append(
            PeriodicKeyedJob(
                key=job.key,
                success_interval=_interval(
                    job.success_interval,
                    name="success_interval",
                ),
                failure_interval=_interval(
                    job.failure_interval,
                    name="failure_interval",
                ),
                callback=job.callback,
            )
        )
    return tuple(validated)


def _run_without_result(callback: Callable[[], None]) -> None:
    callback()


def _bounded_error_kind(error: BaseException) -> str:
    name = type(error).__name__
    return name if _ERROR_KIND(name) is not None else "Exception"


class KeyedPeriodicExecutor:
    def __init__(
        self,
        jobs: tuple[PeriodicKeyedJob, ...],
        *,
        max_workers: int,
        clock: Callable[[], float] = time.monotonic,
    ) -> None:
        self._jobs = _validated_jobs(jobs)
        if type(max_workers) is not int:
            raise TypeError("max_workers must be an exact integer")
        if not 1 <= max_workers <= min(_MAX_WORKERS, len(self._jobs)):
            raise ValueError("max_workers is outside the supported range")
        if not callable(clock):
            raise TypeError("clock must be callable")

        self._clock = clock
        started_at = self._read_clock()
        self._last_clock = started_at
        self._next_due = {job.key: started_at for job in self._jobs}
        self._futures: dict[str, Future[None]] = {}
        self._latest: dict[str, PeriodicJobOutcome] = {}
        self._executor = ThreadPoolExecutor(
            max_workers=max_workers,
            thread_name_prefix="periodic-keyed",
        )
        self._owner_lock = threading.Lock()
        self._stopped = False
        self._close_report: FanoutCloseReport | None = None

    def _read_clock(self) -> float:
        value = self._clock()
        if isinstance(value, bool) or not isinstance(value, (int, float)):
            raise TypeError("clock must return a numeric value")
        try:
            observed = float(value)
        except (OverflowError, TypeError, ValueError) as error:
            raise ValueError("clock value must fit in a finite float") from error
        if not math.isfinite(observed) or abs(observed) > _MAX_CLOCK_ABSOLUTE:
            raise ValueError("clock value is outside the supported range")
        return observed

    def _finish_future(self, key: str, future: Future[None]) -> PeriodicJobOutcome:
        if future.cancelled():
            return PeriodicJobOutcome(key, PeriodicOutcomeState.CANCELLED, None)
        try:
            future.result()
        except BaseException as error:
            return PeriodicJobOutcome(
                key,
                PeriodicOutcomeState.FAILED,
                _bounded_error_kind(error),
            )
        return PeriodicJobOutcome(key, PeriodicOutcomeState.SUCCEEDED, None)

    def tick(self) -> FanoutTickReport:
        if not self._owner_lock.acquire(blocking=False):
            raise RuntimeError("another lifecycle operation is active")
        try:
            if self._stopped:
                raise RuntimeError("fanout owner is stopped")
            observed_at = self._read_clock()
            if observed_at < self._last_clock:
                raise RuntimeError("monotonic clock moved backward")
            self._last_clock = observed_at

            completed: list[PeriodicJobOutcome] = []
            for job in self._jobs:
                future = self._futures.get(job.key)
                if future is None or not future.done():
                    continue
                outcome = self._finish_future(job.key, future)
                del self._futures[job.key]
                self._latest[job.key] = outcome
                interval = (
                    job.success_interval
                    if outcome.state is PeriodicOutcomeState.SUCCEEDED
                    else job.failure_interval
                )
                next_due = observed_at + interval
                if not math.isfinite(next_due) or next_due <= observed_at:
                    raise RuntimeError("job interval does not advance its next deadline")
                self._next_due[job.key] = next_due
                completed.append(outcome)

            submitted: list[str] = []
            for job in self._jobs:
                if job.key in self._futures or observed_at < self._next_due[job.key]:
                    continue
                self._futures[job.key] = self._executor.submit(
                    _run_without_result,
                    job.callback,
                )
                submitted.append(job.key)

            return FanoutTickReport(
                observed_at=observed_at,
                completed=tuple(completed),
                submitted=tuple(submitted),
                in_flight=tuple(job.key for job in self._jobs if job.key in self._futures),
            )
        finally:
            self._owner_lock.release()

    def latest_outcomes(self) -> tuple[PeriodicJobOutcome, ...]:
        if not self._owner_lock.acquire(blocking=False):
            raise RuntimeError("another lifecycle operation is active")
        try:
            return tuple(
                outcome for job in self._jobs if (outcome := self._latest.get(job.key)) is not None
            )
        finally:
            self._owner_lock.release()

    def close(self) -> FanoutCloseReport:
        if not self._owner_lock.acquire(blocking=False):
            raise RuntimeError("another lifecycle operation is active")
        try:
            if self._close_report is not None:
                return self._close_report
            self._stopped = True
            self._executor.shutdown(wait=True, cancel_futures=True)

            finalized: list[PeriodicJobOutcome] = []
            for job in self._jobs:
                future = self._futures.pop(job.key, None)
                if future is None:
                    continue
                outcome = self._finish_future(job.key, future)
                self._latest[job.key] = outcome
                finalized.append(outcome)
            self._close_report = FanoutCloseReport(tuple(finalized))
            return self._close_report
        finally:
            self._owner_lock.release()

    def __enter__(self) -> "KeyedPeriodicExecutor":
        if self._stopped:
            raise RuntimeError("fanout owner is stopped")
        return self

    def __exit__(self, _error_type, _error, _traceback) -> None:
        self.close()
```

## Example

```python
import time


class ManualClock:
    def __init__(self) -> None:
        self.now = 0.0

    def monotonic(self) -> float:
        return self.now

    def advance(self, seconds: float) -> None:
        self.now += seconds


clock = ManualClock()
slow_started = threading.Event()
second_slow_started = threading.Event()
release_slow = threading.Event()
invocation_lock = threading.Lock()
slow_invocations = 0


def slow_probe() -> None:
    global slow_invocations
    with invocation_lock:
        slow_invocations += 1
        invocation = slow_invocations
    slow_started.set()
    if invocation == 2:
        second_slow_started.set()
    if not release_slow.wait(3.0):
        raise TimeoutError("example release was not signaled")


def failing_probe() -> None:
    raise LookupError("synthetic failure")


owner = KeyedPeriodicExecutor(
    (
        PeriodicKeyedJob("slow-probe", 5.0, 3.0, slow_probe),
        PeriodicKeyedJob("failing-probe", 5.0, 20.0, failing_probe),
    ),
    max_workers=2,
    clock=clock.monotonic,
)
initial = owner.tick()
slow_was_started = slow_started.wait(1.0)

failure_seen = False
real_deadline = time.monotonic() + 1.0
while not failure_seen and time.monotonic() < real_deadline:
    observed = owner.tick()
    failure_seen = any(
        outcome.key == "failing-probe"
        and outcome.state is PeriodicOutcomeState.FAILED
        and outcome.error_kind == "LookupError"
        for outcome in observed.completed
    )
    if not failure_seen:
        time.sleep(0.001)

clock.advance(10.0)
while_slow = owner.tick()
release_slow.set()

slow_success_seen = False
real_deadline = time.monotonic() + 1.0
while not slow_success_seen and time.monotonic() < real_deadline:
    observed = owner.tick()
    slow_success_seen = any(
        outcome.key == "slow-probe"
        and outcome.state is PeriodicOutcomeState.SUCCEEDED
        for outcome in observed.completed
    )
    if not slow_success_seen:
        time.sleep(0.001)

clock.advance(5.0)
resubmitted = owner.tick()
second_run_started = second_slow_started.wait(1.0)
closed = owner.close()
try:
    owner.tick()
except RuntimeError:
    post_close_rejected = True
else:
    post_close_rejected = False

assert (
    initial.submitted,
    slow_was_started,
    failure_seen,
    while_slow.submitted,
    slow_success_seen,
    resubmitted.submitted,
    second_run_started,
    tuple(outcome.state for outcome in closed.finalized),
    slow_invocations,
    post_close_rejected,
) == (
    ("slow-probe", "failing-probe"),
    True,
    True,
    (),
    True,
    ("slow-probe",),
    True,
    (PeriodicOutcomeState.SUCCEEDED,),
    2,
    True,
)
```

## Trade-offs and Limitations

At most 32 futures exist because the key set is fixed and each key owns one
slot. A completion is rescheduled from the next tick that observes it, which
may delay later work but never creates catch-up submissions. Callback return
values and exception messages are discarded; only one bounded terminal outcome
per key remains available for diagnostics.

`close()` first rejects later ticks, cancels futures that have not started, and
then waits for running callbacks before harvesting every terminal future.
Python cannot forcibly stop a running thread, so this drain is bounded only by
the callbacks' own cooperative timeout contracts. The owner deliberately
rejects concurrent lifecycle calls rather than pretending that multiple
schedulers can mutate the same future slots safely.

## Related Snippets

<!-- catalog:related:start -->
- [Refresh Bounded Named Targets at Their Nearest Monotonic Deadline](refresh-bounded-named-targets-at-their-nearest-monotonic-deadline.md)
- [Reuse One Pending Future Across Non-Cancelling Poll Timeouts](reuse-one-pending-future-across-non-cancelling-poll-timeouts.md)
- [Collect Thread-Pool Results and Errors as Futures Complete](collect-thread-pool-results-and-errors-as-futures-complete.md)
<!-- catalog:related:end -->
