---
title: "Guard an Async Resource with Explicit Lifecycle States"
snippet_type: pattern
use_cases:
  - concurrency-control
  - lifecycle-management
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - gather-async-results-with-bounded-concurrency.md
  - stop-a-polling-worker-cooperatively-with-an-event.md
  - ../reliability-resilience/wait-for-a-predicate-until-a-monotonic-deadline.md
---

# Guard an Async Resource with Explicit Lifecycle States

## Idea and Problem

Serialize asynchronous start and stop transitions so callers can observe and enforce one explicit resource lifecycle.

A pair of booleans cannot distinguish an operation in progress from one that
failed. This guard moves through named states, rejects invalid or overlapping
transitions, applies a timeout request to each callback, and makes every unsuccessful
transition terminal instead of guessing whether a partly changed resource is
safe to reuse.

## When to Use

Use this guard when one event-loop-owned resource has separate asynchronous
start and stop callbacks, concurrent lifecycle calls are possible, and retrying
after a partial transition would be unsafe. The callbacks must cooperate with
cancellation and own any rollback needed after a partial failure. Prefer the
resource's native context manager when it already defines equivalent states,
timeouts, and cleanup behavior.

## Implementation

```python
import asyncio
import math
from collections.abc import Awaitable, Callable
from enum import Enum, auto
from types import TracebackType
from typing import Self


_MAX_TRANSITION_TIMEOUT = 86_400.0


class LifecycleState(Enum):
    NEW = auto()
    STARTING = auto()
    RUNNING = auto()
    STOPPING = auto()
    STOPPED = auto()
    FAILED = auto()


class LifecycleError(RuntimeError):
    pass


def _transition_timeout(value: float, *, name: str) -> float:
    if isinstance(value, bool) or not isinstance(value, (int, float)):
        raise TypeError(f"{name} must be a number")
    try:
        timeout = float(value)
    except OverflowError as error:
        raise ValueError(f"{name} must be finite") from error
    if not math.isfinite(timeout):
        raise ValueError(f"{name} must be finite")
    if not 0.0 < timeout <= _MAX_TRANSITION_TIMEOUT:
        raise ValueError(f"{name} is outside the supported range")
    return timeout


class AsyncLifecycleGuard:
    def __init__(
        self,
        start: Callable[[], Awaitable[None]],
        stop: Callable[[], Awaitable[None]],
        *,
        start_timeout: float,
        stop_timeout: float,
    ) -> None:
        if not callable(start) or not callable(stop):
            raise TypeError("start and stop must be callable")
        self._start = start
        self._stop = stop
        self._start_timeout = _transition_timeout(
            start_timeout,
            name="start_timeout",
        )
        self._stop_timeout = _transition_timeout(
            stop_timeout,
            name="stop_timeout",
        )
        self._state = LifecycleState.NEW
        self._transition_lock = asyncio.Lock()

    @property
    def state(self) -> LifecycleState:
        return self._state

    def require_running(self) -> None:
        if self._state is not LifecycleState.RUNNING:
            raise LifecycleError(
                f"resource is not running: {self._state.name}"
            )

    async def start(self) -> None:
        async with self._transition_lock:
            if self._state is not LifecycleState.NEW:
                raise LifecycleError(
                    f"cannot start from {self._state.name}"
                )
            self._state = LifecycleState.STARTING

        try:
            async with asyncio.timeout(self._start_timeout) as deadline:
                await self._start()
            if deadline.expired():
                raise TimeoutError("start callback exceeded its timeout")
            self._state = LifecycleState.RUNNING
        except BaseException:
            self._state = LifecycleState.FAILED
            raise

    async def stop(self) -> None:
        async with self._transition_lock:
            if self._state is not LifecycleState.RUNNING:
                raise LifecycleError(
                    f"cannot stop from {self._state.name}"
                )
            self._state = LifecycleState.STOPPING

        try:
            async with asyncio.timeout(self._stop_timeout) as deadline:
                await self._stop()
            if deadline.expired():
                raise TimeoutError("stop callback exceeded its timeout")
            self._state = LifecycleState.STOPPED
        except BaseException:
            self._state = LifecycleState.FAILED
            raise

    async def __aenter__(self) -> Self:
        await self.start()
        return self

    async def __aexit__(
        self,
        exc_type: type[BaseException] | None,
        exc: BaseException | None,
        traceback: TracebackType | None,
    ) -> None:
        await self.stop()
```

## Example

```python
events: list[str] = []


async def open_resource() -> None:
    await asyncio.sleep(0)
    events.append("started")


async def close_resource() -> None:
    await asyncio.sleep(0)
    events.append("stopped")


async def use_resource() -> LifecycleState:
    guard = AsyncLifecycleGuard(
        open_resource,
        close_resource,
        start_timeout=1.0,
        stop_timeout=1.0,
    )
    async with guard:
        guard.require_running()
        events.append("used")
    return guard.state


final_state = asyncio.run(use_resource())

assert (events, final_state) == (
    ["started", "used", "stopped"],
    LifecycleState.STOPPED,
)
```

## Trade-offs and Limitations

This guard serializes lifecycle transitions within one event loop; it does not
make other resource methods safe across tasks or OS threads. A transition that
fails or is cancelled enters `FAILED` permanently because the wrapper cannot
know which side effects occurred. The callback therefore remains responsible
for failure cleanup and external recovery.

The timeout context requests cancellation at the deadline and the explicit
expiration check prevents a callback that suppresses that cancellation from
being treated as successful. A callback that delays cancellation can still
make the call exceed the configured duration. A stop failure raised by
`__aexit__` can also replace an exception from the body, with the body
exception retained as context. This pattern provides explicit state, not
forced termination, retries, health checks, or dependency ordering.

## Related Snippets

<!-- catalog:related:start -->
- [Gather Async Results with Bounded Concurrency](gather-async-results-with-bounded-concurrency.md)
- [Stop a Polling Worker Cooperatively with an Event](stop-a-polling-worker-cooperatively-with-an-event.md)
- [Wait for a Predicate Until a Monotonic Deadline](../reliability-resilience/wait-for-a-predicate-until-a-monotonic-deadline.md)
<!-- catalog:related:end -->
