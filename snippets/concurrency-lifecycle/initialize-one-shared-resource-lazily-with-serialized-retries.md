---
title: "Initialize One Shared Resource Lazily with Serialized Retries"
snippet_type: pattern
use_cases:
  - concurrency-control
  - lifecycle-management
  - retry-recovery
tested_python:
  - "3.14"
dependencies: []
related:
  - guard-an-async-resource-with-explicit-lifecycle-states.md
  - ../python-language/cache-one-zero-argument-method-result-per-weakly-referenced-instance.md
  - ../reliability-resilience/wait-for-a-predicate-until-a-monotonic-deadline.md
---

# Initialize One Shared Resource Lazily with Serialized Retries

## Idea and Problem

Serialize first access to a shared synchronous resource, retry only approved creation failures, and publish the value only after construction succeeds.

The factory and any configured backoff run under one lock. Concurrent callers
therefore observe the same successfully created object, while an exhausted or
unapproved failure remains uncached so a later call can try again.

## When to Use

Use this pattern when construction must be delayed until first access, only one
thread may construct at a time, and a small fixed retry schedule is acceptable.
Supply a narrow `retry_if` predicate for known transient creation exceptions
and inject the sleeper when tests must avoid wall-clock delays. Ensure the
factory does not call `get()` on the same wrapper and cleans up every partially
allocated resource before an attempt raises.

Initialize eagerly when startup should fail fast, construction is cheap, or
holding a lock during connection setup would create unacceptable contention.
Use a fuller lifecycle abstraction when the value needs reset, asynchronous
creation, health checks, ownership transfer, or coordinated shutdown.

## Implementation

```python
import math
import time
from collections.abc import Callable
from threading import Lock
from typing import Generic, TypeVar, cast


ResourceT = TypeVar("ResourceT")
_MAX_RETRY_DELAYS = 7
_MAX_DELAY_SECONDS = 60.0


def _never_retry(_error: Exception) -> bool:
    return False


class LazySharedResource(Generic[ResourceT]):
    def __init__(
        self,
        factory: Callable[[], ResourceT],
        *,
        retry_delays: tuple[float, ...] = (),
        retry_if: Callable[[Exception], bool] = _never_retry,
        sleep: Callable[[float], None] = time.sleep,
    ) -> None:
        for name, callback in (
            ("factory", factory),
            ("retry_if", retry_if),
            ("sleep", sleep),
        ):
            if not callable(callback):
                raise TypeError(f"{name} must be callable")
        if not isinstance(retry_delays, tuple):
            raise TypeError("retry_delays must be a tuple")
        if len(retry_delays) > _MAX_RETRY_DELAYS:
            raise ValueError("too many retry delays")
        normalized_delays: list[float] = []
        for delay in retry_delays:
            if isinstance(delay, bool) or not isinstance(delay, (int, float)):
                raise TypeError("retry delays must be numbers")
            normalized = float(delay)
            if not math.isfinite(normalized) or not 0 <= normalized <= _MAX_DELAY_SECONDS:
                raise ValueError("a retry delay is outside the supported range")
            normalized_delays.append(normalized)

        self._factory = factory
        self._retry_delays = tuple(normalized_delays)
        self._retry_if = retry_if
        self._sleep = sleep
        self._lock = Lock()
        self._initialized = False
        self._value: ResourceT | None = None

    @property
    def initialized(self) -> bool:
        with self._lock:
            return self._initialized

    def get(self) -> ResourceT:
        with self._lock:
            if self._initialized:
                return cast(ResourceT, self._value)

            for attempt in range(len(self._retry_delays) + 1):
                try:
                    created = self._factory()
                except Exception as error:
                    if (
                        attempt == len(self._retry_delays)
                        or not self._retry_if(error)
                    ):
                        raise
                    self._sleep(self._retry_delays[attempt])
                else:
                    self._value = created
                    self._initialized = True
                    return created

        raise AssertionError("the bounded attempt loop must return or raise")
```

## Example

```python
from threading import Barrier, Thread


class TemporaryCreationError(RuntimeError):
    pass


created_resource = object()
factory_calls = 0
sleeps: list[float] = []


def create_resource() -> object:
    global factory_calls
    factory_calls += 1
    if factory_calls == 1:
        raise TemporaryCreationError("not ready")
    return created_resource


lazy = LazySharedResource(
    create_resource,
    retry_delays=(0.25,),
    retry_if=lambda error: isinstance(error, TemporaryCreationError),
    sleep=sleeps.append,
)
barrier = Barrier(3)
results: list[object] = []
results_lock = Lock()


def access() -> None:
    barrier.wait()
    result = lazy.get()
    with results_lock:
        results.append(result)


threads = [Thread(target=access) for _ in range(2)]
for thread in threads:
    thread.start()
barrier.wait()
for thread in threads:
    thread.join(timeout=2)

failure_calls = 0


def always_fail() -> object:
    global failure_calls
    failure_calls += 1
    raise ValueError("still unavailable")


uncached_failure = LazySharedResource(always_fail)
for _attempt in range(2):
    try:
        uncached_failure.get()
    except ValueError:
        pass

assert (
    len(results),
    all(result is created_resource for result in results),
    factory_calls,
    sleeps,
    lazy.initialized,
    failure_calls,
    uncached_failure.initialized,
) == (2, True, 2, [0.25], True, 2, False)
```

## Trade-offs and Limitations

All concurrent callers wait while the factory, retry predicate, and sleeper run
inside the lock. This is intentional serialization, but long connection
attempts can create high latency. Retries occur only within one `get()` call;
there is no global deadline, jitter, circuit breaker, or cancellation. A
factory that re-enters the same wrapper deadlocks because the lock is not
reentrant.

Only `Exception` failures are considered for retry. `BaseException` subclasses
propagate immediately, as do errors from the predicate or sleeper, and none are
cached. Because a failed attempt publishes no handle, its factory alone can
release partially allocated state; retrying a leaky factory compounds the
leak. The wrapper neither closes nor resets the resource and makes no claim
that operations on the returned object are thread-safe. If shutdown owns the
value, establish that lifecycle separately and do not let callers outlive it.

## Related Snippets

<!-- catalog:related:start -->
- [Guard an Async Resource with Explicit Lifecycle States](guard-an-async-resource-with-explicit-lifecycle-states.md)
- [Cache One Zero-Argument Method Result per Weakly Referenced Instance](../python-language/cache-one-zero-argument-method-result-per-weakly-referenced-instance.md)
- [Wait for a Predicate Until a Monotonic Deadline](../reliability-resilience/wait-for-a-predicate-until-a-monotonic-deadline.md)
<!-- catalog:related:end -->
