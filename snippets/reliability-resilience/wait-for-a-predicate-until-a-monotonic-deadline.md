---
title: "Wait for a Predicate Until a Monotonic Deadline"
snippet_type: recipe
use_cases:
  - retry-recovery
tested_python:
  - "3.14"
dependencies: []
related:
  - ../configuration-serialization/parse-compact-durations-into-timedelta.md
  - ../concurrency-lifecycle/gather-async-results-with-bounded-concurrency.md
---

# Wait for a Predicate Until a Monotonic Deadline

## Idea and Problem

Poll synchronous state within one monotonic time budget instead of scattering unbounded sleep loops through calling code.

The predicate is evaluated immediately, and every later sleep is capped by the
remaining budget. Returning the predicate's first truthy value avoids a second
lookup after readiness has already been observed.

## When to Use

Use this recipe for short, synchronous waits on eventually consistent state,
such as a local file, an in-process status adapter, or a bounded test fake. The
predicate should be quick, side-effect-aware, and safe to call repeatedly. Use
an event, condition variable, callback, or async primitive when the producer
can signal readiness directly instead of being polled.

## Implementation

```python
import math
import time
from collections.abc import Callable
from typing import TypeVar


ResultT = TypeVar("ResultT")


class PredicateTimeoutError(TimeoutError):
    pass


def _duration(value: float, *, name: str, allow_zero: bool) -> float:
    if isinstance(value, bool) or not isinstance(value, (int, float)):
        raise TypeError(f"{name} must be numeric")
    duration = float(value)
    if not math.isfinite(duration):
        raise ValueError(f"{name} must be finite")
    if duration < 0 or (duration == 0 and not allow_zero):
        qualifier = "non-negative" if allow_zero else "positive"
        raise ValueError(f"{name} must be {qualifier}")
    return duration


def wait_for_predicate(
    predicate: Callable[[], ResultT],
    *,
    timeout: float,
    interval: float,
    message: str = "predicate did not become true before the deadline",
    clock: Callable[[], float] = time.monotonic,
    sleeper: Callable[[float], None] = time.sleep,
) -> ResultT:
    timeout = _duration(timeout, name="timeout", allow_zero=True)
    interval = _duration(interval, name="interval", allow_zero=False)
    deadline = clock() + timeout
    first_attempt = True

    while True:
        if not first_attempt and clock() > deadline:
            raise PredicateTimeoutError(message)
        first_attempt = False

        result = predicate()
        if result:
            return result

        remaining = deadline - clock()
        if remaining <= 0:
            raise PredicateTimeoutError(message)
        sleeper(min(interval, remaining))
```

## Example

```python
class FakeTime:
    def __init__(self) -> None:
        self.now = 0.0
        self.sleeps = []

    def monotonic(self) -> float:
        return self.now

    def sleep(self, duration: float) -> None:
        self.sleeps.append(duration)
        self.now += duration


fake = FakeTime()
attempts = 0


def readiness() -> str:
    global attempts
    attempts += 1
    return "ready" if attempts == 3 else ""


result = wait_for_predicate(
    readiness,
    timeout=5,
    interval=2,
    clock=fake.monotonic,
    sleeper=fake.sleep,
)

expired = FakeTime()
try:
    wait_for_predicate(
        lambda: False,
        timeout=1,
        interval=3,
        clock=expired.monotonic,
        sleeper=expired.sleep,
    )
except PredicateTimeoutError:
    timeout_raised = True
else:
    timeout_raised = False

overslept = FakeTime()


def oversleep(duration: float) -> None:
    overslept.sleeps.append(duration)
    overslept.now += duration + 1.0


try:
    wait_for_predicate(
        lambda: overslept.now >= 1.0,
        timeout=1,
        interval=1,
        clock=overslept.monotonic,
        sleeper=oversleep,
    )
except PredicateTimeoutError:
    late_success_rejected = True
else:
    late_success_rejected = False

assert (
    result,
    attempts,
    fake.sleeps,
    timeout_raised,
    expired.sleeps,
    late_success_rejected,
    overslept.sleeps,
    wait_for_predicate(lambda: 42, timeout=0, interval=1),
) == ("ready", 3, [2.0, 2.0], True, [1.0], True, [1.0], 42)
```

## Trade-offs and Limitations

Polling adds latency and repeated work, and this helper does not add backoff,
jitter, logging, or retries for predicate exceptions. Such exceptions
propagate immediately. A slow or hung predicate cannot be interrupted, so a
poll started no later than the deadline may finish after it; predicate runtime
still consumes the shared budget. A poll occurs at the exact deadline, but an
oversleep detected beyond it raises before another predicate call. Injected
clocks and sleepers must advance on the same monotonic time basis.

## Related Snippets

<!-- catalog:related:start -->
- [Parse Compact Durations into timedelta](../configuration-serialization/parse-compact-durations-into-timedelta.md)
- [Gather Async Results with Bounded Concurrency](../concurrency-lifecycle/gather-async-results-with-bounded-concurrency.md)
<!-- catalog:related:end -->
