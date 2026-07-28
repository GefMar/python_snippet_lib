---
title: "Refresh Bounded Named Targets at Their Nearest Monotonic Deadline"
snippet_type: pattern
use_cases:
  - caching
  - concurrency-control
  - lifecycle-management
  - retry-recovery
tested_python:
  - "3.14"
dependencies: []
related:
  - stop-a-polling-worker-cooperatively-with-an-event.md
  - refresh-an-async-value-within-a-bounded-stale-window.md
  - ../testing-tooling/wait-for-named-queue-conditions-under-one-monotonic-deadline.md
---

# Refresh Bounded Named Targets at Their Nearest Monotonic Deadline

## Idea and Problem

Own one non-daemon worker that refreshes several named targets at their nearest monotonic deadlines without polling or holding its state lock during callbacks.

A fixed set of refreshable resources may have different success intervals and
failure cooldowns. Waiting for the earliest current deadline avoids periodic
polling, while a condition notification lets an owner request earlier work.
Processing simultaneously due targets in declaration order and rescheduling
from callback completion makes the behavior deterministic and prevents runtime
duration from shortening the intended interval.

## When to Use

Use this owner when 1-32 trusted, cooperative, zero-argument callbacks can run
sequentially in one thread. Deadlines must come from `time.monotonic()`, and
each callback must have its own I/O timeout so shutdown latency is bounded by
the callback contract. Ordinary callback failures are isolated and retried
after a positive cooldown.

Use a bounded executor when refreshes must overlap. Use an async task owner in
an async application. If a callback cannot return cooperatively, isolate it in
a process or another system that supports the required termination policy.

## Implementation

```python
import math
import re
import time
from collections.abc import Callable
from dataclasses import dataclass
from threading import TIMEOUT_MAX, Condition, Thread, current_thread


_MAX_TARGETS = 32
_MAX_WAIT_SECONDS = min(30 * 24 * 60 * 60, TIMEOUT_MAX / 2)
_NAME = re.compile(r"[a-z][a-z0-9_-]{0,63}", re.ASCII)


@dataclass(frozen=True, slots=True)
class RefreshTarget:
    name: str
    callback: Callable[[], None]
    first_deadline: float
    success_interval: float
    failure_cooldown: float


@dataclass(frozen=True, slots=True)
class RefreshFailure:
    target_name: str
    exception_type: str


class RefreshWorkerTerminated(RuntimeError):
    def __init__(self, failure: RefreshFailure) -> None:
        self.failure = failure
        super().__init__(
            f"refresh worker terminated at {failure.target_name} "
            f"with {failure.exception_type}"
        )


def _time_value(
    value: object,
    field: str,
    *,
    positive: bool,
    bounded_duration: bool = False,
) -> float:
    if type(value) not in (int, float):
        raise TypeError(f"{field} must be an exact int or float")
    converted = float(value)
    if not math.isfinite(converted):
        raise ValueError(f"{field} must be finite")
    if positive:
        if converted <= 0:
            raise ValueError(f"{field} must be positive")
    elif converted < 0:
        raise ValueError(f"{field} must be non-negative")
    if bounded_duration and converted > _MAX_WAIT_SECONDS:
        raise ValueError(f"{field} exceeds the platform-safe wait limit")
    return converted


class DeadlineRefresher:
    def __init__(self, targets: tuple[RefreshTarget, ...]) -> None:
        if type(targets) is not tuple:
            raise TypeError("targets must be an exact tuple")
        if not 1 <= len(targets) <= _MAX_TARGETS:
            raise ValueError(f"targets must contain 1..{_MAX_TARGETS} records")

        checked: list[RefreshTarget] = []
        deadlines: list[float] = []
        names: set[str] = set()
        for target in targets:
            if type(target) is not RefreshTarget:
                raise TypeError("targets must contain exact RefreshTarget records")
            if type(target.name) is not str:
                raise TypeError("target name must be an exact string")
            if _NAME.fullmatch(target.name) is None:
                raise ValueError("target name is invalid")
            if target.name in names:
                raise ValueError("target names must be unique")
            if not callable(target.callback):
                raise TypeError("target callback must be callable")
            names.add(target.name)
            checked.append(target)
            first_deadline = _time_value(
                target.first_deadline,
                "first_deadline",
                positive=False,
            )
            if first_deadline - time.monotonic() > _MAX_WAIT_SECONDS:
                raise ValueError("first_deadline exceeds the platform-safe wait limit")
            deadlines.append(first_deadline)
            _time_value(
                target.success_interval,
                "success_interval",
                positive=True,
                bounded_duration=True,
            )
            _time_value(
                target.failure_cooldown,
                "failure_cooldown",
                positive=True,
                bounded_duration=True,
            )

        self._targets = tuple(checked)
        self._index = {target.name: index for index, target in enumerate(checked)}
        self._deadlines = deadlines
        self._in_flight = [False] * len(checked)
        self._pending_deadlines: list[float | None] = [None] * len(checked)
        self._failures: dict[str, RefreshFailure] = {}
        self._fatal_failure: RefreshFailure | None = None
        self._condition = Condition()
        self._thread: Thread | None = None
        self._started = False
        self._stopping = False

    def start(self) -> None:
        with self._condition:
            if self._started:
                raise RuntimeError("refresher cannot be started twice")
            self._started = True
            thread = Thread(
                target=self._run,
                name="deadline-refresher",
                daemon=False,
            )
            self._thread = thread
            try:
                thread.start()
            except BaseException:
                self._thread = None
                self._started = False
                raise

    def request_earlier(self, target_name: str, deadline: float) -> None:
        if type(target_name) is not str:
            raise TypeError("target_name must be an exact string")
        requested = _time_value(deadline, "deadline", positive=False)
        if requested - time.monotonic() > _MAX_WAIT_SECONDS:
            raise ValueError("deadline exceeds the platform-safe wait limit")
        with self._condition:
            if not self._started or self._stopping:
                raise RuntimeError("refresher is not running")
            try:
                index = self._index[target_name]
            except KeyError:
                raise ValueError("unknown target name") from None
            if self._in_flight[index]:
                pending = self._pending_deadlines[index]
                if pending is None or requested < pending:
                    self._pending_deadlines[index] = requested
                self._condition.notify()
            elif requested < self._deadlines[index]:
                self._deadlines[index] = requested
                self._condition.notify()

    def failure_snapshot(self) -> tuple[RefreshFailure, ...]:
        with self._condition:
            return tuple(
                self._failures[target.name]
                for target in self._targets
                if target.name in self._failures
            )

    def stop(self, timeout: float) -> None:
        wait_limit = _time_value(
            timeout,
            "timeout",
            positive=True,
            bounded_duration=True,
        )
        with self._condition:
            if not self._started or self._thread is None:
                raise RuntimeError("refresher has not been started")
            thread = self._thread
            if current_thread() is thread:
                raise RuntimeError("worker thread cannot join itself")
            self._stopping = True
            self._condition.notify()

        thread.join(wait_limit)
        if thread.is_alive():
            raise TimeoutError("refresh callback did not stop before the deadline")
        with self._condition:
            fatal_failure = self._fatal_failure
        if fatal_failure is not None:
            raise RefreshWorkerTerminated(fatal_failure)

    def _run(self) -> None:
        while True:
            with self._condition:
                while True:
                    if self._stopping:
                        return
                    now = time.monotonic()
                    nearest = min(self._deadlines)
                    if nearest <= now:
                        due = tuple(
                            index
                            for index, deadline in enumerate(self._deadlines)
                            if deadline <= now
                        )
                        break
                    self._condition.wait(timeout=nearest - now)

            for index in due:
                with self._condition:
                    if self._stopping:
                        return
                    self._in_flight[index] = True
                target = self._targets[index]
                try:
                    target.callback()
                except Exception as error:
                    interval = target.failure_cooldown
                    failure = RefreshFailure(
                        target_name=target.name,
                        exception_type=type(error).__name__[:64],
                    )
                except BaseException as error:
                    fatal_failure = RefreshFailure(
                        target_name=target.name,
                        exception_type=type(error).__name__[:64],
                    )
                    with self._condition:
                        self._in_flight[index] = False
                        self._pending_deadlines[index] = None
                        self._failures[target.name] = fatal_failure
                        self._fatal_failure = fatal_failure
                        self._stopping = True
                        self._condition.notify_all()
                    return
                else:
                    interval = target.success_interval
                    failure = None

                completed_at = time.monotonic()
                with self._condition:
                    scheduled = completed_at + interval
                    pending = self._pending_deadlines[index]
                    self._deadlines[index] = (
                        scheduled if pending is None else min(scheduled, pending)
                    )
                    self._in_flight[index] = False
                    self._pending_deadlines[index] = None
                    if failure is None:
                        self._failures.pop(target.name, None)
                    else:
                        self._failures[target.name] = failure
```

## Example

```python
from threading import Event


refreshed = Event()
calls: list[str] = []


def refresh_index() -> None:
    calls.append("index")
    refreshed.set()


worker = DeadlineRefresher(
    (
        RefreshTarget(
            name="index",
            callback=refresh_index,
            first_deadline=time.monotonic() + 60.0,
            success_interval=60.0,
            failure_cooldown=5.0,
        ),
    )
)
worker.start()
worker.request_earlier("index", time.monotonic())
completed = refreshed.wait(timeout=2.0)
worker.stop(timeout=2.0)

assert completed and calls == ["index"] and worker.failure_snapshot() == ()
```

## Trade-offs and Limitations

Each scheduling pass scans at most 32 deadlines, so its work is `O(t)` with
`O(t)` state. First deadlines, intervals, cooldowns, and join timeouts are
capped at the smaller of 30 days and half of `threading.TIMEOUT_MAX`, keeping
every timed wait below the platform limit. Callbacks run sequentially and
outside the condition lock; a slow callback delays later targets that are
already due. A request arriving during its callback is retained as a pending
deadline and can cause an immediate second pass. An ordinary `Exception` is
recorded by bounded type name and isolated from other targets.

`stop()` is deliberately observable rather than pretending to kill Python
code: after a timeout, the non-daemon thread remains in stopping state and the
owner may call `stop()` again to join it after the callback returns. A callback
`BaseException` terminates the worker, makes later requests fail, and causes
`stop()` to raise `RefreshWorkerTerminated` with bounded target/type metadata;
the original exception object and message are deliberately not retained. The
class cannot be restarted and does not own cache values, freshness checks,
callback deadlines, parallelism, persistence, jitter, logging, or wall-clock
schedules.

## Related Snippets

<!-- catalog:related:start -->
- [Stop a Polling Worker Cooperatively with an Event](stop-a-polling-worker-cooperatively-with-an-event.md)
- [Refresh an Async Value Within a Bounded Stale Window](refresh-an-async-value-within-a-bounded-stale-window.md)
- [Wait for Named Queue Conditions Under One Monotonic Deadline](../testing-tooling/wait-for-named-queue-conditions-under-one-monotonic-deadline.md)
<!-- catalog:related:end -->
