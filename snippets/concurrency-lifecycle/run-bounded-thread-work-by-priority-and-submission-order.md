---
title: "Run Bounded Thread Work by Priority and Submission Order"
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
  - gather-async-results-with-bounded-concurrency.md
  - stop-a-polling-worker-cooperatively-with-an-event.md
---

# Run Bounded Thread Work by Priority and Submission Order

## Idea and Problem

Run blocking callables on fixed worker threads while bounding queued work and ordering pending jobs by priority, then FIFO submission order.

Each accepted job receives a `Future`. A monotonic sequence makes equal
priorities stable without comparing functions or arguments, while Python
3.14's graceful queue shutdown lets workers drain accepted jobs and then exit.
Lower numeric values run before higher numeric values among pending jobs, and
submission never waits for capacity.

## When to Use

Use this small executor when synchronous jobs share one process, have a clear
integer priority, and a full pending queue should reject producers immediately.
Call `shutdown()` or use the context manager at the owning lifecycle boundary.
Keep each job independently safe to run on any worker and make the resources it
uses thread-safe.

Use `ThreadPoolExecutor` when priority and a bounded pending queue are not
requirements. Use an asynchronous queue for coroutine-native work, or an
external durable system for crash recovery, distribution, admission policy,
or process isolation.

## Implementation

```python
import queue
from collections.abc import Callable
from concurrent.futures import Future
from dataclasses import dataclass, field
from threading import Lock, Thread
from typing import Any


_MAX_WORKERS = 64
_MAX_PENDING = 100_000
_MAX_ABSOLUTE_PRIORITY = 1_000_000_000


@dataclass(order=True, slots=True)
class _PriorityJob:
    priority: int
    sequence: int
    future: Future[Any] = field(compare=False)
    function: Callable[..., Any] = field(compare=False)
    args: tuple[Any, ...] = field(compare=False)
    kwargs: dict[str, Any] = field(compare=False)


def _bounded_executor_integer(
    value: int,
    *,
    name: str,
    minimum: int,
    maximum: int,
) -> int:
    if isinstance(value, bool) or not isinstance(value, int):
        raise TypeError(f"{name} must be an integer")
    if not minimum <= value <= maximum:
        raise ValueError(f"{name} is outside the supported range")
    return value


class BoundedPriorityExecutor:
    def __init__(self, *, max_workers: int, max_pending: int) -> None:
        max_workers = _bounded_executor_integer(
            max_workers,
            name="max_workers",
            minimum=1,
            maximum=_MAX_WORKERS,
        )
        max_pending = _bounded_executor_integer(
            max_pending,
            name="max_pending",
            minimum=1,
            maximum=_MAX_PENDING,
        )
        self._queue: queue.PriorityQueue[_PriorityJob] = queue.PriorityQueue(
            maxsize=max_pending
        )
        self._state_lock = Lock()
        self._closed = False
        self._next_sequence = 0
        self._threads = tuple(
            Thread(
                target=self._worker,
                name=f"priority-worker-{index}",
                daemon=False,
            )
            for index in range(max_workers)
        )
        started_threads: list[Thread] = []
        try:
            for thread in self._threads:
                thread.start()
                started_threads.append(thread)
        except BaseException:
            self._closed = True
            self._queue.shutdown(immediate=True)
            for thread in started_threads:
                thread.join()
            raise

    def submit(
        self,
        priority: int,
        function: Callable[..., Any],
        /,
        *args: Any,
        **kwargs: Any,
    ) -> Future[Any]:
        priority = _bounded_executor_integer(
            priority,
            name="priority",
            minimum=-_MAX_ABSOLUTE_PRIORITY,
            maximum=_MAX_ABSOLUTE_PRIORITY,
        )
        if not callable(function):
            raise TypeError("function must be callable")

        future: Future[Any] = Future()
        with self._state_lock:
            if self._closed:
                raise RuntimeError("executor is shut down")
            job = _PriorityJob(
                priority=priority,
                sequence=self._next_sequence,
                future=future,
                function=function,
                args=args,
                kwargs=dict(kwargs),
            )
            self._queue.put_nowait(job)
            self._next_sequence += 1
        return future

    def _worker(self) -> None:
        while True:
            try:
                job = self._queue.get()
            except queue.ShutDown:
                return
            try:
                if job.future.set_running_or_notify_cancel():
                    try:
                        result = job.function(*job.args, **job.kwargs)
                    except BaseException as error:
                        job.future.set_exception(error)
                    else:
                        job.future.set_result(result)
            finally:
                self._queue.task_done()

    def shutdown(self, *, wait: bool = True) -> None:
        if not isinstance(wait, bool):
            raise TypeError("wait must be a boolean")
        with self._state_lock:
            if not self._closed:
                self._closed = True
                self._queue.shutdown(immediate=False)
        if wait:
            self._queue.join()
            for thread in self._threads:
                thread.join()

    def __enter__(self) -> "BoundedPriorityExecutor":
        return self

    def __exit__(self, *_error: object) -> None:
        self.shutdown()
```

## Example

```python
from threading import Event


started = Event()
release = Event()
run_order: list[str] = []


def hold_worker() -> str:
    started.set()
    if not release.wait(timeout=2):
        raise TimeoutError("example gate was not released")
    run_order.append("initial")
    return "initial"


def record(name: str) -> str:
    run_order.append(name)
    return name


def fail() -> None:
    raise ValueError("job failed")


with BoundedPriorityExecutor(max_workers=1, max_pending=4) as executor:
    initial = executor.submit(100, hold_worker)
    if not started.wait(timeout=2):
        raise TimeoutError("worker did not start")

    low = executor.submit(10, record, "low")
    equal_first = executor.submit(5, record, "equal-first")
    high = executor.submit(0, record, "high")
    equal_second = executor.submit(5, record, "equal-second")
    try:
        executor.submit(20, record, "overflow")
    except queue.Full:
        full_rejected = True
    else:
        full_rejected = False

    release.set()
    values = tuple(
        future.result(timeout=2)
        for future in (initial, high, equal_first, equal_second, low)
    )
    failed = executor.submit(0, fail)
    try:
        failed.result(timeout=2)
    except ValueError:
        error_propagated = True
    else:
        error_propagated = False

try:
    executor.submit(0, record, "late")
except RuntimeError:
    late_rejected = True
else:
    late_rejected = False

assert (
    values,
    run_order,
    full_rejected,
    error_propagated,
    late_rejected,
) == (
    ("initial", "high", "equal-first", "equal-second", "low"),
    ["initial", "high", "equal-first", "equal-second", "low"],
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

Priority is considered only when a worker calls `get`; it cannot interrupt a
running job, guarantee fairness, or prevent starvation if higher-priority work
keeps arriving. The queue cap covers jobs still waiting, not jobs already
running. `submit()` can race with workers consuming entries, so `queue.Full`
is an admission result for that instant rather than a promise about later
capacity.

`shutdown(wait=True)` waits for all accepted jobs. A callable that hangs can
therefore hang shutdown unless the callable enforces its own timeout and
cooperative cancellation. The owning thread must perform shutdown; a worker
cannot join itself. `Future.cancel()` prevents a queued job from
running, but this implementation does not remove its placeholder before a
worker dequeues it. Exceptions are retained by futures and do not stop worker
threads. The executor is process-local, non-durable, and offers no immediate
shutdown mode or automatic resource cleanup inside jobs.

## Related Snippets

<!-- catalog:related:start -->
- [Collect Thread-Pool Results and Errors as Futures Complete](collect-thread-pool-results-and-errors-as-futures-complete.md)
- [Gather Async Results with Bounded Concurrency](gather-async-results-with-bounded-concurrency.md)
- [Stop a Polling Worker Cooperatively with an Event](stop-a-polling-worker-cooperatively-with-an-event.md)
<!-- catalog:related:end -->
