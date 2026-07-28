---
title: "Poll with Deterministic Capped Backoff Under One Monotonic Deadline"
snippet_type: recipe
use_cases:
  - retry-recovery
  - testing
tested_python:
  - "3.14"
dependencies: []
related:
  - wait-for-a-predicate-until-a-monotonic-deadline.md
  - poll-a-remote-operation-within-deadline-and-failure-budgets.md
  - ../configuration-serialization/parse-compact-durations-into-timedelta.md
---

# Poll with Deterministic Capped Backoff Under One Monotonic Deadline

## Idea and Problem

Probe immediately and then reduce polling pressure with a deterministic capped delay while preserving one absolute monotonic deadline.

A growing interval helps when early readiness is common but sustained fixed-rate
polling is wasteful. Validate the complete schedule first, clamp every sleep to
remaining time, and report the last observed value with actual attempts and
elapsed time. The result distinguishes readiness from exhaustion without a
second probe after the budget has ended.

## When to Use

Use this recipe for a cheap idempotent local check whose producer cannot signal
an event or condition. It suits bounded tests, filesystem observation, and
small adapters where deterministic progression is preferable to randomized
jitter. Inject a clock and sleeper when exact schedule behavior must be tested.

Use event-driven notification when available, and add jitter outside this
primitive when many clients could synchronize. A slow or blocking predicate
needs its own timeout because a synchronous caller cannot interrupt it here.

## Implementation

```python
import math
import time
from collections.abc import Callable
from dataclasses import dataclass
from typing import Generic, TypeVar

ValueT = TypeVar("ValueT")
_MAX_TIMEOUT = 3_600.0
_MAX_DELAY = 300.0
_MAX_ATTEMPTS = 10_000
_MIN_DELAY = 0.001


def _duration(
    value: object,
    *,
    name: str,
    minimum: float,
    maximum: float,
) -> float:
    if isinstance(value, bool) or not isinstance(value, (int, float)):
        raise TypeError(f"{name} must be numeric")
    try:
        normalized = float(value)
    except (OverflowError, TypeError, ValueError) as error:
        raise ValueError(f"{name} must fit in a finite float") from error
    if not math.isfinite(normalized):
        raise ValueError(f"{name} must be finite")
    if not minimum <= normalized <= maximum:
        raise ValueError(f"{name} is outside the supported range")
    return normalized


def _clock_value(clock: Callable[[], float]) -> float:
    return _duration(
        clock(),
        name="clock value",
        minimum=-float.fromhex("0x1.fffffffffffffp+1023"),
        maximum=float.fromhex("0x1.fffffffffffffp+1023"),
    )


@dataclass(frozen=True, slots=True)
class BackoffOutcome(Generic[ValueT]):
    ready: bool
    last_value: ValueT
    attempts: int
    elapsed: float


def poll_with_capped_backoff(
    predicate: Callable[[], ValueT],
    is_ready: Callable[[ValueT], bool],
    *,
    timeout: float,
    initial_delay: float,
    maximum_delay: float,
    multiplier: float,
    maximum_attempts: int,
    clock: Callable[[], float] = time.monotonic,
    sleeper: Callable[[float], None] = time.sleep,
) -> BackoffOutcome[ValueT]:
    if not callable(predicate) or not callable(is_ready):
        raise TypeError("predicate and is_ready must be callable")
    if not callable(clock) or not callable(sleeper):
        raise TypeError("clock and sleeper must be callable")
    timeout = _duration(
        timeout,
        name="timeout",
        minimum=0.0,
        maximum=_MAX_TIMEOUT,
    )
    initial_delay = _duration(
        initial_delay,
        name="initial_delay",
        minimum=_MIN_DELAY,
        maximum=_MAX_DELAY,
    )
    maximum_delay = _duration(
        maximum_delay,
        name="maximum_delay",
        minimum=initial_delay,
        maximum=_MAX_DELAY,
    )
    multiplier = _duration(
        multiplier,
        name="multiplier",
        minimum=1.01,
        maximum=10.0,
    )
    if type(maximum_attempts) is not int:
        raise TypeError("maximum_attempts must be an exact integer")
    if not 1 <= maximum_attempts <= _MAX_ATTEMPTS:
        raise ValueError("maximum_attempts is outside the supported range")

    started = _clock_value(clock)
    deadline = started + timeout
    if not math.isfinite(deadline):
        raise ValueError("the resulting monotonic deadline must be finite")
    last_clock = started
    delay = initial_delay
    attempts = 0

    while True:
        value = predicate()
        attempts += 1
        ready = bool(is_ready(value))
        observed_at = _clock_value(clock)
        if observed_at < last_clock:
            raise RuntimeError("monotonic clock moved backward")
        last_clock = observed_at
        elapsed = observed_at - started

        if observed_at > deadline:
            return BackoffOutcome(False, value, attempts, elapsed)
        if ready:
            return BackoffOutcome(True, value, attempts, elapsed)
        if attempts >= maximum_attempts or observed_at >= deadline:
            return BackoffOutcome(False, value, attempts, elapsed)

        sleep_for = min(delay, deadline - observed_at)
        sleeper(sleep_for)
        after_sleep = _clock_value(clock)
        if after_sleep <= observed_at:
            raise RuntimeError("sleeper did not advance the monotonic clock")
        last_clock = after_sleep
        if after_sleep > deadline:
            return BackoffOutcome(
                False,
                value,
                attempts,
                after_sleep - started,
            )
        delay = min(maximum_delay, delay * multiplier)
```

## Example

```python
class FakeTime:
    def __init__(self) -> None:
        self.now = 0.0
        self.sleeps: list[float] = []

    def monotonic(self) -> float:
        return self.now

    def sleep(self, duration: float) -> None:
        self.sleeps.append(duration)
        self.now += duration


clock = FakeTime()
values = iter(("starting", "warming", "ready"))
success = poll_with_capped_backoff(
    lambda: next(values),
    lambda value: value == "ready",
    timeout=10,
    initial_delay=1,
    maximum_delay=4,
    multiplier=2,
    maximum_attempts=6,
    clock=clock.monotonic,
    sleeper=clock.sleep,
)

overslept = FakeTime()


def oversleep(duration: float) -> None:
    overslept.sleeps.append(duration)
    overslept.now += duration + 0.25


expired = poll_with_capped_backoff(
    lambda: "waiting",
    lambda value: value == "ready",
    timeout=1,
    initial_delay=2,
    maximum_delay=4,
    multiplier=2,
    maximum_attempts=4,
    clock=overslept.monotonic,
    sleeper=oversleep,
)

stalled = FakeTime()
try:
    poll_with_capped_backoff(
        lambda: False,
        bool,
        timeout=1,
        initial_delay=0.5,
        maximum_delay=1,
        multiplier=2,
        maximum_attempts=3,
        clock=stalled.monotonic,
        sleeper=lambda _duration: None,
    )
except RuntimeError:
    stalled_sleep_rejected = True
else:
    stalled_sleep_rejected = False

assert (
    success,
    clock.sleeps,
    expired,
    overslept.sleeps,
    stalled_sleep_rejected,
) == (
    BackoffOutcome(True, "ready", 3, 3.0),
    [1.0, 2.0],
    BackoffOutcome(False, "waiting", 1, 1.25),
    [1.0],
    True,
)
```

## Trade-offs and Limitations

Increasing delays reduce probe volume but also increase detection latency. The
schedule is deterministic, so a fleet of identical callers can still align;
add caller-owned jitter when that matters. Attempt exhaustion may stop before
the time budget, while an oversleep or slow predicate can report actual elapsed
time greater than the requested timeout.

Predicate and readiness exceptions propagate immediately and are not retries.
The helper is synchronous, does not cancel a predicate, and treats a clock that
moves backward or a sleeper that makes no progress as a contract failure. Use
an event, condition, async primitive, or remote-operation state machine when
those richer lifecycle semantics are available.

## Related Snippets

<!-- catalog:related:start -->
- [Wait for a Predicate Until a Monotonic Deadline](wait-for-a-predicate-until-a-monotonic-deadline.md)
- [Poll a Remote Operation Within Deadline and Failure Budgets](poll-a-remote-operation-within-deadline-and-failure-budgets.md)
- [Parse Compact Durations into timedelta](../configuration-serialization/parse-compact-durations-into-timedelta.md)
<!-- catalog:related:end -->
