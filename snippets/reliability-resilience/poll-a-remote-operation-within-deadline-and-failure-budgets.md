---
title: "Poll a Remote Operation Within Deadline and Failure Budgets"
snippet_type: integration
use_cases:
  - automation
  - networking
  - resource-management
  - retry-recovery
tested_python:
  - "3.14"
dependencies: []
related:
  - wait-for-a-predicate-until-a-monotonic-deadline.md
  - ../networking-protocols/collect-matching-cursor-pages-with-an-explicit-page-budget.md
  - compensate-completed-workflow-steps-in-reverse-order.md
---

# Poll a Remote Operation Within Deadline and Failure Budgets

## Idea and Problem

Observe a remote operation until it succeeds or fails while bounding elapsed time, status reads, and consecutive transient read failures independently.

The adapter exposes only three states: pending, succeeded, and failed. Polling
uses one monotonic deadline, clamps every sleep to the remaining time, retries
only explicitly approved fetch exceptions, and keeps progress callbacks outside
the retry boundary.

## When to Use

Use this integration when a remote API starts work separately and exposes a
cheap, idempotent status read. Adapt the transport response into the immutable
`OperationStatus` before returning it. Select narrow exception classes that
mean the status read may safely be repeated, and make the progress callback
quick and side-effect-aware.

Prefer a completion event, callback, or streaming API when the remote system
offers one. Use a service-specific client when it defines retry headers,
backoff, authentication refresh, or richer terminal states. Keep cancellation,
result retrieval, and remote cleanup in the caller because their safety policy
cannot be inferred from polling state.

## Implementation

```python
import math
import re
import time
from collections.abc import Callable
from dataclasses import dataclass
from enum import StrEnum


_OPERATION_ID = re.compile(r"[A-Za-z0-9][A-Za-z0-9._:-]{0,199}")
_MAX_TIMEOUT_SECONDS = 86_400.0
_MAX_INTERVAL_SECONDS = 300.0
_MAX_POLLS = 10_000
_MAX_CONSECUTIVE_FAILURES = 20


class OperationPhase(StrEnum):
    PENDING = "pending"
    SUCCEEDED = "succeeded"
    FAILED = "failed"


@dataclass(frozen=True, slots=True)
class OperationStatus:
    phase: OperationPhase
    detail: str = ""

class OperationPollingError(RuntimeError):
    pass


class OperationFailedError(OperationPollingError):
    def __init__(self, status: OperationStatus) -> None:
        self.status = status
        super().__init__(status.detail or "the remote operation failed")


class OperationPollingTimeoutError(OperationPollingError, TimeoutError):
    pass


class OperationPollingLimitError(OperationPollingError):
    pass


class OperationStatusReadError(OperationPollingError):
    pass


def _duration(value: float, *, name: str, maximum: float) -> float:
    if isinstance(value, bool) or not isinstance(value, (int, float)):
        raise TypeError(f"{name} must be numeric")
    try:
        normalized = float(value)
    except OverflowError as error:
        raise ValueError(f"{name} must be finite") from error
    if not math.isfinite(normalized) or not 0.0 < normalized <= maximum:
        raise ValueError(f"{name} must be positive and bounded")
    return normalized


def _bounded_integer(value: int, *, name: str, lower: int, upper: int) -> int:
    if isinstance(value, bool) or not isinstance(value, int):
        raise TypeError(f"{name} must be an integer")
    if not lower <= value <= upper:
        raise ValueError(f"{name} is outside the supported range")
    return value


def _retry_types(retry_on: tuple[type[Exception], ...]) -> None:
    if (
        not isinstance(retry_on, tuple)
        or not retry_on
        or len(retry_on) > 8
        or any(
            not isinstance(error_type, type)
            or not issubclass(error_type, Exception)
            for error_type in retry_on
        )
    ):
        raise TypeError("retry_on must contain 1 to 8 Exception classes")


def _clock_value(clock: Callable[[], float]) -> float:
    value = clock()
    if isinstance(value, bool) or not isinstance(value, (int, float)):
        raise TypeError("clock must return a number")
    try:
        normalized = float(value)
    except OverflowError as error:
        raise ValueError("clock must return a finite value") from error
    if not math.isfinite(normalized):
        raise ValueError("clock must return a finite value")
    return normalized


def _validate_status(status: object) -> OperationStatus:
    if type(status) is not OperationStatus:
        raise TypeError("fetch_status must return an exact OperationStatus")
    if not isinstance(status.phase, OperationPhase):
        raise TypeError("status phase must be an OperationPhase")
    if not isinstance(status.detail, str):
        raise TypeError("status detail must be a string")
    if len(status.detail) > 500 or (
        status.detail and not status.detail.isprintable()
    ):
        raise ValueError("status detail must be bounded printable text")
    return status


def poll_remote_operation(
    fetch_status: Callable[[str, float], OperationStatus],
    operation_id: str,
    *,
    timeout: float,
    interval: float,
    max_polls: int,
    max_consecutive_failures: int,
    retry_on: tuple[type[Exception], ...] = (OSError,),
    on_progress: Callable[[OperationStatus, int], None] | None = None,
    clock: Callable[[], float] = time.monotonic,
    sleeper: Callable[[float], None] = time.sleep,
) -> OperationStatus:
    for name, callback in (
        ("fetch_status", fetch_status),
        ("clock", clock),
        ("sleeper", sleeper),
    ):
        if not callable(callback):
            raise TypeError(f"{name} must be callable")
    if on_progress is not None and not callable(on_progress):
        raise TypeError("on_progress must be callable or None")
    if not isinstance(operation_id, str) or _OPERATION_ID.fullmatch(operation_id) is None:
        raise ValueError("operation_id has an unsupported format")
    timeout = _duration(timeout, name="timeout", maximum=_MAX_TIMEOUT_SECONDS)
    interval = _duration(interval, name="interval", maximum=_MAX_INTERVAL_SECONDS)
    max_polls = _bounded_integer(
        max_polls,
        name="max_polls",
        lower=1,
        upper=_MAX_POLLS,
    )
    max_consecutive_failures = _bounded_integer(
        max_consecutive_failures,
        name="max_consecutive_failures",
        lower=0,
        upper=_MAX_CONSECUTIVE_FAILURES,
    )
    _retry_types(retry_on)

    started_at = _clock_value(clock)
    deadline = started_at + timeout
    if not math.isfinite(deadline):
        raise ValueError("the resulting polling deadline must be finite")
    consecutive_failures = 0

    for poll_number in range(1, max_polls + 1):
        now = _clock_value(clock)
        if now >= deadline:
            raise OperationPollingTimeoutError("the polling deadline was reached")

        try:
            status = fetch_status(operation_id, deadline - now)
        except retry_on as error:
            if _clock_value(clock) >= deadline:
                raise OperationPollingTimeoutError(
                    "the polling deadline was reached during a status read"
                ) from error
            consecutive_failures += 1
            if consecutive_failures > max_consecutive_failures:
                raise OperationStatusReadError(
                    "the consecutive status-read failure budget was exhausted"
                ) from error
        else:
            consecutive_failures = 0
            after_read = _clock_value(clock)
            if after_read >= deadline:
                raise OperationPollingTimeoutError(
                    "the polling deadline was reached during a status read"
                )
            status = _validate_status(status)
            if on_progress is not None:
                on_progress(status, poll_number)
            if status.phase is OperationPhase.SUCCEEDED:
                return status
            if status.phase is OperationPhase.FAILED:
                raise OperationFailedError(status)

        if poll_number == max_polls:
            raise OperationPollingLimitError("max_polls was exhausted")

        now = _clock_value(clock)
        remaining = deadline - now
        if remaining <= 0.0:
            raise OperationPollingTimeoutError("the polling deadline was reached")
        sleeper(min(interval, remaining))

    raise AssertionError("the bounded polling loop must return or raise")
```

## Example

```python
class TemporaryStatusError(OSError):
    pass


class FakeTime:
    def __init__(self) -> None:
        self.now = 0.0
        self.sleeps: list[float] = []

    def monotonic(self) -> float:
        return self.now

    def sleep(self, duration: float) -> None:
        self.sleeps.append(duration)
        self.now += duration


fake = FakeTime()
reads = 0
progress: list[tuple[OperationPhase, int]] = []
transport_timeouts: list[float] = []


def fetch_status(operation_id: str, transport_timeout: float) -> OperationStatus:
    global reads
    assert operation_id == "operation-17"
    transport_timeouts.append(transport_timeout)
    reads += 1
    if reads == 1:
        raise TemporaryStatusError("temporary transport failure")
    if reads == 2:
        return OperationStatus(OperationPhase.PENDING, "working")
    return OperationStatus(OperationPhase.SUCCEEDED)


result = poll_remote_operation(
    fetch_status,
    "operation-17",
    timeout=5.0,
    interval=2.0,
    max_polls=4,
    max_consecutive_failures=1,
    retry_on=(TemporaryStatusError,),
    on_progress=lambda status, poll: progress.append((status.phase, poll)),
    clock=fake.monotonic,
    sleeper=fake.sleep,
)

terminal = FakeTime()
try:
    poll_remote_operation(
        lambda _operation_id, _timeout: OperationStatus(
            OperationPhase.FAILED,
            "rejected",
        ),
        "operation-18",
        timeout=1.0,
        interval=0.25,
        max_polls=2,
        max_consecutive_failures=0,
        clock=terminal.monotonic,
        sleeper=terminal.sleep,
    )
except OperationFailedError as error:
    failure_detail = error.status.detail
else:
    failure_detail = ""

assert (
    result.phase,
    reads,
    fake.sleeps,
    progress,
    transport_timeouts,
    failure_detail,
) == (
    OperationPhase.SUCCEEDED,
    3,
    [2.0, 2.0],
    [
        (OperationPhase.PENDING, 2),
        (OperationPhase.SUCCEEDED, 3),
    ],
    [5.0, 3.0, 1.0],
    "rejected",
)
```

## Trade-offs and Limitations

Every poll consumes remote and local resources, and fixed intervals can align
many clients into bursts. This helper adds neither exponential backoff nor
jitter. A fetch call, callback, sleeper, or clock that blocks can overrun the
deadline because Python cannot forcibly interrupt it. The poll limit counts
both successful status reads and transiently failed attempts; the consecutive
failure count resets only after a valid status is returned.

Only exceptions raised by `fetch_status` are eligible for `retry_on`. The
adapter receives the current positive remaining time and must apply it to its
transport; the deadline is checked again when it returns. Status validation,
progress callbacks, clock access, and sleeping remain outside the catch so
internal defects are not mistaken for transport failures. The helper does not
start, cancel, forget, delete, or fetch the result of the remote operation. The
caller must define those actions, especially after local timeout when remote
work may still be running.

## Related Snippets

<!-- catalog:related:start -->
- [Wait for a Predicate Until a Monotonic Deadline](wait-for-a-predicate-until-a-monotonic-deadline.md)
- [Collect Matching Cursor Pages with an Explicit Page Budget](../networking-protocols/collect-matching-cursor-pages-with-an-explicit-page-budget.md)
- [Compensate Completed Workflow Steps in Reverse Order](compensate-completed-workflow-steps-in-reverse-order.md)
<!-- catalog:related:end -->
