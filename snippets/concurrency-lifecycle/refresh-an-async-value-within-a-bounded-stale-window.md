---
title: "Refresh an Async Value Within a Bounded Stale Window"
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
  - initialize-one-shared-resource-lazily-with-serialized-retries.md
  - guard-an-async-resource-with-explicit-lifecycle-states.md
  - ../reliability-resilience/cache-values-with-a-monotonic-ttl-and-early-jitter.md
---

# Refresh an Async Value Within a Bounded Stale Window

## Idea and Problem

Keep serving one immutable async-produced value while a single background task refreshes it, but never serve that stale value beyond an explicit monotonic age limit.

A cold read waits for initialization. A read inside the stale window starts at
most one refresh and returns immediately, while a read beyond the hard boundary
waits for a fresh value or receives an observable refresh error. The wrapper
owns its task and provides explicit asynchronous shutdown.

## When to Use

Use this pattern for event-loop-local snapshots such as feature data, discovery
results, or immutable configuration whose brief staleness is acceptable. The
factory must return a self-contained value that does not require disposal, and
all callers must agree on one freshness policy. Inject a monotonic clock in
tests and choose a refresh timeout shorter than the operational deadline.

Do not use this wrapper for connections, clients, file handles, or other
resources that need leases and coordinated retirement. Refresh synchronously
when stale data is never acceptable, and use a cache with explicit keys and
eviction when values vary by request.

## Implementation

```python
import asyncio
import math
import time
from collections.abc import Awaitable, Callable
from typing import Generic, TypeVar, cast


ValueT = TypeVar("ValueT")
_MAX_WINDOW_SECONDS = 86_400.0


class AsyncRefreshError(RuntimeError):
    pass


class AsyncValueClosedError(AsyncRefreshError):
    pass


def _window(value: float, *, name: str, allow_zero: bool) -> float:
    if isinstance(value, bool) or not isinstance(value, (int, float)):
        raise TypeError(f"{name} must be numeric")
    try:
        normalized = float(value)
    except OverflowError as error:
        raise ValueError(f"{name} must be finite") from error
    valid_lower_bound = normalized >= 0.0 if allow_zero else normalized > 0.0
    if (
        not math.isfinite(normalized)
        or not valid_lower_bound
        or normalized > _MAX_WINDOW_SECONDS
    ):
        qualifier = "non-negative" if allow_zero else "positive"
        raise ValueError(f"{name} must be {qualifier} and bounded")
    return normalized


class AsyncStaleValue(Generic[ValueT]):
    def __init__(
        self,
        factory: Callable[[], Awaitable[ValueT]],
        *,
        fresh_for: float,
        stale_for: float,
        refresh_timeout: float,
        clock: Callable[[], float] = time.monotonic,
    ) -> None:
        if not callable(factory) or not callable(clock):
            raise TypeError("factory and clock must be callable")
        self._factory = factory
        self._fresh_for = _window(fresh_for, name="fresh_for", allow_zero=False)
        self._stale_for = _window(stale_for, name="stale_for", allow_zero=True)
        self._refresh_timeout = _window(
            refresh_timeout,
            name="refresh_timeout",
            allow_zero=False,
        )
        self._clock = clock
        self._lock = asyncio.Lock()
        self._loaded = False
        self._value: ValueT | None = None
        self._loaded_at = 0.0
        self._last_error: Exception | None = None
        self._refresh_task: asyncio.Task[None] | None = None
        self._closed = False

    @property
    def last_refresh_error(self) -> Exception | None:
        return self._last_error

    def _now(self) -> float:
        value = self._clock()
        if isinstance(value, bool) or not isinstance(value, (int, float)):
            raise TypeError("clock must return a number")
        try:
            normalized = float(value)
        except OverflowError as error:
            raise ValueError("clock must return a finite value") from error
        if not math.isfinite(normalized):
            raise ValueError("clock must return a finite value")
        return normalized

    async def _refresh(self) -> None:
        try:
            async with asyncio.timeout(self._refresh_timeout) as timeout:
                created = await self._factory()
            if timeout.expired():
                raise TimeoutError("the refresh exceeded its timeout")
            created_at = self._now()
        except Exception as error:
            async with self._lock:
                if not self._closed:
                    self._last_error = error
            return

        async with self._lock:
            if not self._closed:
                self._value = created
                self._loaded_at = created_at
                self._loaded = True
                self._last_error = None

    def _start_refresh_locked(self) -> asyncio.Task[None]:
        task = self._refresh_task
        if task is None or task.done():
            task = asyncio.create_task(self._refresh())
            self._refresh_task = task
        return task

    async def get(self) -> ValueT:
        async with self._lock:
            if self._closed:
                raise AsyncValueClosedError("the async value is closed")

            if self._loaded:
                age = max(0.0, self._now() - self._loaded_at)
                if age < self._fresh_for:
                    return cast(ValueT, self._value)
                if age < self._fresh_for + self._stale_for:
                    self._start_refresh_locked()
                    return cast(ValueT, self._value)

            refresh_task = self._start_refresh_locked()

        try:
            await asyncio.shield(refresh_task)
        except asyncio.CancelledError:
            current_task = asyncio.current_task()
            if current_task is not None and current_task.cancelling():
                raise
            async with self._lock:
                if self._closed and refresh_task.cancelled():
                    raise AsyncValueClosedError(
                        "the async value is closed"
                    ) from None
            raise

        async with self._lock:
            if self._closed:
                raise AsyncValueClosedError("the async value is closed")
            if self._loaded:
                age = max(0.0, self._now() - self._loaded_at)
                if age < self._fresh_for + self._stale_for:
                    return cast(ValueT, self._value)
            error = self._last_error

        if error is not None:
            raise AsyncRefreshError("no value is available within the stale limit") from error
        raise AsyncRefreshError("the refresh ended without publishing a value")

    async def aclose(self) -> None:
        async with self._lock:
            if self._closed:
                return
            self._closed = True
            refresh_task = self._refresh_task

        if refresh_task is not None and not refresh_task.done():
            refresh_task.cancel()
            try:
                await refresh_task
            except asyncio.CancelledError:
                pass
```

## Example

```python
class FakeClock:
    def __init__(self) -> None:
        self.now = 0.0

    def monotonic(self) -> float:
        return self.now


class TemporaryRefreshError(RuntimeError):
    pass


async def exercise_refresh() -> tuple[object, ...]:
    clock = FakeClock()
    calls = 0

    async def create_snapshot() -> str:
        nonlocal calls
        calls += 1
        await asyncio.sleep(0)
        if calls == 2:
            raise TemporaryRefreshError("source is temporarily unavailable")
        return f"snapshot-{calls}"

    value = AsyncStaleValue(
        create_snapshot,
        fresh_for=1.0,
        stale_for=2.0,
        refresh_timeout=1.0,
        clock=clock.monotonic,
    )
    first = await value.get()
    clock.now = 1.5
    stale = await value.get()
    await asyncio.sleep(0)
    await asyncio.sleep(0)
    failed_in_background = isinstance(
        value.last_refresh_error,
        TemporaryRefreshError,
    )

    clock.now = 2.0
    stale_during_retry = await value.get()
    await asyncio.sleep(0)
    await asyncio.sleep(0)
    refreshed = await value.get()
    await value.aclose()
    await value.aclose()
    return (
        first,
        stale,
        failed_in_background,
        stale_during_retry,
        refreshed,
        calls,
    )


assert asyncio.run(exercise_refresh()) == (
    "snapshot-1",
    "snapshot-1",
    True,
    "snapshot-1",
    "snapshot-3",
    3,
)
```

## Trade-offs and Limitations

The first caller after expiry triggers refresh; there is no proactive timer,
jitter, distributed coordination, keyed storage, or persistence. Calls inside
the stale window deliberately hide the current refresh latency, so consumers
must observe `last_refresh_error` separately. Every hard-boundary call may
start another attempt after a failed task, and this wrapper provides no retry
backoff or circuit breaker.

The task is strongly referenced and shielded so cancellation of one waiting
caller does not cancel work shared with other callers. `aclose()` does cancel
the owned task and waits for it, but a factory that suppresses cancellation can
delay shutdown. Asyncio timeouts are cooperative for the same reason. The
wrapper is confined to one event loop, and the injected clock must be monotonic.
Replacing disposable resources would leak or invalidate live users, which is
why this contract is restricted to immutable values.

## Related Snippets

<!-- catalog:related:start -->
- [Initialize One Shared Resource Lazily with Serialized Retries](initialize-one-shared-resource-lazily-with-serialized-retries.md)
- [Guard an Async Resource with Explicit Lifecycle States](guard-an-async-resource-with-explicit-lifecycle-states.md)
- [Cache Values with a Monotonic TTL and Early Jitter](../reliability-resilience/cache-values-with-a-monotonic-ttl-and-early-jitter.md)
<!-- catalog:related:end -->
