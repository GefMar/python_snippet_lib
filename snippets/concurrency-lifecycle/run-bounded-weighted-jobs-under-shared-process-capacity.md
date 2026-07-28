---
title: "Run Bounded Weighted Jobs Under Shared Process Capacity"
snippet_type: pattern
use_cases:
  - concurrency-control
  - lifecycle-management
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - run-bounded-thread-work-by-priority-and-submission-order.md
  - ../algorithms-data-structures/report-exact-capacity-deficits-for-bounded-resource-profiles.md
  - ../observability-operations/share-bounded-counters-and-duration-histograms-across-spawned-processes.md
---

# Run Bounded Weighted Jobs Under Shared Process Capacity

## Idea and Problem

Run a finite process batch while keeping the sum of declared costs for live children within one fixed capacity.

A worker count alone cannot express that one job is expected to consume more of
a constrained resource than another. Give every job a positive integer cost,
scan the frozen queue in stable order for work that fits, and stop the complete
batch on the first start failure, child failure, or shared deadline. Because no
new work enters the queue, bounded backfilling cannot starve a skipped job.

## When to Use

Use this pattern for a small batch of independent, pickle-compatible jobs when
an approximate CPU, memory, or external-service cost is known before launch.
Create the jobs and shared state from one explicit `spawn` context, and keep
each target at module scope so a fresh interpreter can import it.

Use a plain process pool when jobs have roughly equal cost. Use an external
scheduler when work must survive a parent crash, arrive continuously, return
large values, retry selectively, or coordinate capacity across hosts. A cost
is an admission hint; it does not measure or enforce real resource use.

## Implementation

```python
import math
import re
import time
from collections.abc import Callable
from dataclasses import dataclass
from enum import StrEnum
from multiprocessing.connection import wait
from multiprocessing.context import BaseContext
from multiprocessing.process import BaseProcess

_JOB_NAME = re.compile(r"[a-z][a-z0-9_-]{0,31}", re.ASCII).fullmatch
_MAX_JOBS = 32
_MAX_CAPACITY = 64
_MAX_RUNTIME_SECONDS = 3_600.0
_TERMINATE_GRACE_SECONDS = 1.0
_KILL_GRACE_SECONDS = 1.0


@dataclass(frozen=True, slots=True)
class WeightedProcessJob:
    name: str
    cost: int
    target: Callable[[], None]


class WeightedJobState(StrEnum):
    SUCCEEDED = "succeeded"
    FAILED = "failed"
    START_FAILED = "start-failed"
    CANCELLED = "cancelled"
    TIMED_OUT = "timed-out"
    NOT_STARTED = "not-started"


class WeightedRunState(StrEnum):
    COMPLETED = "completed"
    CHILD_FAILED = "child-failed"
    START_FAILED = "start-failed"
    TIMED_OUT = "timed-out"


@dataclass(frozen=True, slots=True)
class WeightedJobOutcome:
    name: str
    cost: int
    state: WeightedJobState
    exit_code: int | None


@dataclass(frozen=True, slots=True)
class WeightedProcessReport:
    state: WeightedRunState
    outcomes: tuple[WeightedJobOutcome, ...]
    peak_cost: int
    elapsed: float


def _runtime_seconds(value: object) -> float:
    if isinstance(value, bool) or not isinstance(value, (int, float)):
        raise TypeError("timeout must be numeric")
    try:
        timeout = float(value)
    except (OverflowError, TypeError, ValueError) as error:
        raise ValueError("timeout must fit in a finite float") from error
    if not math.isfinite(timeout) or not 0.01 <= timeout <= _MAX_RUNTIME_SECONDS:
        raise ValueError("timeout is outside the supported range")
    return timeout


def _validated_jobs(
    value: object,
    *,
    capacity: int,
) -> tuple[WeightedProcessJob, ...]:
    if type(value) is not tuple:
        raise TypeError("jobs must be an exact tuple")
    if not 1 <= len(value) <= _MAX_JOBS:
        raise ValueError("job count is outside the supported range")

    names: set[str] = set()
    validated: list[WeightedProcessJob] = []
    for job in value:
        if type(job) is not WeightedProcessJob:
            raise TypeError("jobs must contain exact WeightedProcessJob values")
        if type(job.name) is not str or _JOB_NAME(job.name) is None:
            raise ValueError("job names must be conservative ASCII identifiers")
        if job.name in names:
            raise ValueError("job names must be unique")
        if type(job.cost) is not int:
            raise TypeError("job costs must be exact integers")
        if not 1 <= job.cost <= capacity:
            raise ValueError("job cost is outside the shared capacity")
        if not callable(job.target):
            raise TypeError("job targets must be callable")
        names.add(job.name)
        validated.append(WeightedProcessJob(job.name, job.cost, job.target))
    return tuple(validated)


def _terminate_and_reap(
    processes: dict[int, BaseProcess],
) -> dict[int, int]:
    started: dict[int, BaseProcess] = {}
    for job_index, process in processes.items():
        if process.pid is None:
            process.close()
        else:
            started[job_index] = process

    for process in started.values():
        if process.is_alive():
            process.terminate()

    terminate_deadline = time.monotonic() + _TERMINATE_GRACE_SECONDS
    for process in started.values():
        process.join(max(0.0, terminate_deadline - time.monotonic()))

    survivors = tuple(process for process in started.values() if process.is_alive())
    for process in survivors:
        process.kill()

    kill_deadline = time.monotonic() + _KILL_GRACE_SECONDS
    for process in survivors:
        process.join(max(0.0, kill_deadline - time.monotonic()))
    if any(process.is_alive() for process in survivors):
        raise RuntimeError("a killed child could not be reaped")

    exit_codes: dict[int, int] = {}
    for job_index, process in started.items():
        process.join()
        exit_code = process.exitcode
        if exit_code is None:
            raise RuntimeError("a reaped child has no exit code")
        exit_codes[job_index] = exit_code
        process.close()
    return exit_codes


def _build_report(
    run_state: WeightedRunState,
    jobs: tuple[WeightedProcessJob, ...],
    states: list[WeightedJobState],
    exit_codes: list[int | None],
    *,
    peak_cost: int,
    started_at: float,
) -> WeightedProcessReport:
    finished_at = time.monotonic()
    if finished_at < started_at:
        raise RuntimeError("monotonic clock moved backward")
    return WeightedProcessReport(
        state=run_state,
        outcomes=tuple(
            WeightedJobOutcome(job.name, job.cost, states[index], exit_codes[index])
            for index, job in enumerate(jobs)
        ),
        peak_cost=peak_cost,
        elapsed=finished_at - started_at,
    )


def run_bounded_weighted_jobs(
    context: BaseContext,
    jobs: tuple[WeightedProcessJob, ...],
    *,
    capacity: int,
    timeout: float,
) -> WeightedProcessReport:
    if not isinstance(context, BaseContext):
        raise TypeError("context must be a multiprocessing context")
    if type(capacity) is not int:
        raise TypeError("capacity must be an exact integer")
    if not 1 <= capacity <= _MAX_CAPACITY:
        raise ValueError("capacity is outside the supported range")
    jobs = _validated_jobs(jobs, capacity=capacity)
    timeout = _runtime_seconds(timeout)

    started_at = time.monotonic()
    deadline = started_at + timeout
    if not math.isfinite(deadline) or deadline <= started_at:
        raise ValueError("timeout must advance to a finite deadline")

    pending = list(range(len(jobs)))
    running: dict[int, BaseProcess] = {}
    owned: dict[int, BaseProcess] = {}
    states = [WeightedJobState.NOT_STARTED for _job in jobs]
    exit_codes: list[int | None] = [None for _job in jobs]
    running_cost = 0
    peak_cost = 0

    def stop_running(state: WeightedJobState) -> None:
        nonlocal running_cost
        stopped = _terminate_and_reap(owned)
        for job_index, exit_code in stopped.items():
            if states[job_index] is WeightedJobState.NOT_STARTED:
                states[job_index] = state
            exit_codes[job_index] = exit_code
        running.clear()
        owned.clear()
        running_cost = 0

    try:
        while pending or running:
            if time.monotonic() >= deadline:
                stop_running(WeightedJobState.TIMED_OUT)
                return _build_report(
                    WeightedRunState.TIMED_OUT,
                    jobs,
                    states,
                    exit_codes,
                    peak_cost=peak_cost,
                    started_at=started_at,
                )

            remaining_pending: list[int] = []
            start_failed = False
            for job_index in pending:
                job = jobs[job_index]
                if start_failed or job.cost + running_cost > capacity:
                    remaining_pending.append(job_index)
                    continue
                if time.monotonic() >= deadline:
                    remaining_pending.append(job_index)
                    continue

                process = context.Process(
                    name=f"bounded-weighted-{job.name}",
                    target=job.target,
                )
                owned[job_index] = process
                try:
                    process.start()
                except BaseException as start_error:
                    if not isinstance(start_error, Exception):
                        raise
                    states[job_index] = WeightedJobState.START_FAILED
                    start_failed = True
                    continue

                running[job_index] = process
                running_cost += job.cost
                peak_cost = max(peak_cost, running_cost)
            pending = remaining_pending

            if start_failed:
                stop_running(WeightedJobState.CANCELLED)
                return _build_report(
                    WeightedRunState.START_FAILED,
                    jobs,
                    states,
                    exit_codes,
                    peak_cost=peak_cost,
                    started_at=started_at,
                )
            if not running:
                continue

            remaining_time = deadline - time.monotonic()
            if remaining_time <= 0.0:
                continue
            ready_sentinels = set(
                wait(
                    [process.sentinel for process in running.values()],
                    timeout=remaining_time,
                )
            )
            if not ready_sentinels:
                continue

            child_failed = False
            for job_index in tuple(running):
                process = running[job_index]
                if process.sentinel not in ready_sentinels:
                    continue
                process.join()
                exit_code = process.exitcode
                if exit_code is None:
                    raise RuntimeError("a completed child has no exit code")
                process.close()
                del running[job_index]
                del owned[job_index]
                running_cost -= jobs[job_index].cost
                exit_codes[job_index] = exit_code
                if exit_code == 0:
                    states[job_index] = WeightedJobState.SUCCEEDED
                else:
                    states[job_index] = WeightedJobState.FAILED
                    child_failed = True

            if child_failed:
                stop_running(WeightedJobState.CANCELLED)
                return _build_report(
                    WeightedRunState.CHILD_FAILED,
                    jobs,
                    states,
                    exit_codes,
                    peak_cost=peak_cost,
                    started_at=started_at,
                )
    except BaseException:
        if owned:
            stop_running(WeightedJobState.CANCELLED)
        raise

    return _build_report(
        WeightedRunState.COMPLETED,
        jobs,
        states,
        exit_codes,
        peak_cost=peak_cost,
        started_at=started_at,
    )
```

## Example

```python
import multiprocessing
import time
from functools import partial


def record_weighted_scope(
    current,
    peak,
    lock,
    first_wave_barrier,
    cost: int,
    duration: float,
) -> None:
    with lock:
        current.value += cost
        peak.value = max(peak.value, current.value)
    try:
        if first_wave_barrier is not None:
            first_wave_barrier.wait(1.0)
        time.sleep(duration)
    finally:
        with lock:
            current.value -= cost


example_passed = True
if __name__ == "__main__":
    spawn_context = multiprocessing.get_context("spawn")
    current_cost = spawn_context.RawValue("i", 0)
    observed_peak = spawn_context.RawValue("i", 0)
    shared_lock = spawn_context.Lock()
    first_wave_barrier = spawn_context.Barrier(2)
    jobs = (
        WeightedProcessJob(
            "assemble-bundle",
            7,
            partial(
                record_weighted_scope,
                current_cost,
                observed_peak,
                shared_lock,
                first_wave_barrier,
                7,
                0.15,
            ),
        ),
        WeightedProcessJob(
            "verify-manifest",
            4,
            partial(
                record_weighted_scope,
                current_cost,
                observed_peak,
                shared_lock,
                None,
                4,
                0.05,
            ),
        ),
        WeightedProcessJob(
            "write-summary",
            3,
            partial(
                record_weighted_scope,
                current_cost,
                observed_peak,
                shared_lock,
                first_wave_barrier,
                3,
                0.05,
            ),
        ),
    )
    report = run_bounded_weighted_jobs(
        spawn_context,
        jobs,
        capacity=10,
        timeout=5.0,
    )
    example_passed = (
        report.state is WeightedRunState.COMPLETED
        and tuple(outcome.state for outcome in report.outcomes)
        == (WeightedJobState.SUCCEEDED,) * 3
        and report.peak_cost == 10
        and observed_peak.value == 10
        and current_cost.value == 0
    )

assert example_passed
```

## Trade-offs and Limitations

The scheduler scans at most 32 pending jobs per admission pass. It may bypass a
temporarily blocked expensive job, but the input is closed and finite, so no
later arrival can starve it. Fail-fast behavior deliberately cancels unrelated
live jobs after one child fails; use independent batches when partial progress
should continue.

Process startup counts against the shared work deadline. Cleanup has its own
short escalation grace and can make reported elapsed time exceed the requested
timeout. `terminate()` and `kill()` do not roll back external effects, and the
runner carries exit codes rather than return values or remote tracebacks. Every
target must therefore be independently safe to interrupt and diagnose.

## Related Snippets

<!-- catalog:related:start -->
- [Run Bounded Thread Work by Priority and Submission Order](run-bounded-thread-work-by-priority-and-submission-order.md)
- [Report Exact Capacity Deficits for Bounded Resource Profiles](../algorithms-data-structures/report-exact-capacity-deficits-for-bounded-resource-profiles.md)
- [Share Bounded Counters and Duration Histograms Across Spawned Processes](../observability-operations/share-bounded-counters-and-duration-histograms-across-spawned-processes.md)
<!-- catalog:related:end -->
