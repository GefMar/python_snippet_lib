---
title: "Poll an Owned Source Until a Monotonic Idle Deadline"
snippet_type: pattern
use_cases:
  - concurrency-control
  - lifecycle-management
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - stop-a-polling-worker-cooperatively-with-an-event.md
  - own-one-external-mount-lease-with-exception-safe-cleanup.md
  - ../reliability-resilience/wait-for-a-predicate-until-a-monotonic-deadline.md
---

# Poll an Owned Source Until a Monotonic Idle Deadline

## Idea and Problem

Eagerly poll one owned source through explicit item, idle, and end outcomes while one absolute monotonic deadline bounds each uninterrupted idle period.

The first `Idle` starts a deadline. Early readiness may wake the injected wait
callback, but repeated idle results keep the original deadline; an `Item`
clears it so the next idle period receives a fresh budget. Reaching the exact
deadline closes the source without one extra read. A single poll budget bounds
both callback calls and retained items because every item consumes one poll.

## When to Use

Use this pattern for a cooperative in-process adapter that transfers cleanup
ownership to one eager collection operation. The read callback must always
return `Item`, `Idle`, or `End`, including when an item payload itself is
`None`. Supply a monotonic clock and a readiness-aware wait callback that use
the same time basis; tests can inject a manual clock without sleeping.

The idle deadline cannot interrupt a blocking read. A source read that can
block must enforce its own timeout no longer than the lifecycle policy allows.
Use a lazy iterator when retaining all items is inappropriate, or an async
primitive when readiness is naturally asynchronous.

## Implementation

```python
import math
import re
from collections.abc import Callable
from dataclasses import dataclass
from enum import Enum
from typing import Generic, TypeVar

ItemT = TypeVar("ItemT")
_MAX_POLLS = 10_000
_MIN_IDLE_TIMEOUT = 0.01
_MAX_IDLE_TIMEOUT = 3_600.0
_EXCEPTION_KIND = re.compile(
    r"[A-Za-z_][A-Za-z0-9_]{0,63}",
    re.ASCII,
).fullmatch


@dataclass(frozen=True, slots=True)
class Item(Generic[ItemT]):
    value: ItemT


@dataclass(frozen=True, slots=True)
class Idle:
    pass


@dataclass(frozen=True, slots=True)
class End:
    pass


@dataclass(frozen=True, slots=True)
class OwnedPollSource(Generic[ItemT]):
    read: Callable[[], Item[ItemT] | Idle | End]
    close: Callable[[], None]


class PollStopKind(Enum):
    END = "end"
    IDLE_TIMEOUT = "idle_timeout"
    POLL_BUDGET_EXHAUSTED = "poll_budget_exhausted"
    FAILED = "failed"


class PollFailureKind(Enum):
    READ_RAISED = "read_raised"
    INVALID_OUTCOME = "invalid_outcome"
    CLOCK_RAISED = "clock_raised"
    CLOCK_VALUE_INVALID = "clock_value_invalid"
    CLOCK_MOVED_BACKWARD = "clock_moved_backward"
    WAIT_RAISED = "wait_raised"
    CLOSE_RAISED = "close_raised"


@dataclass(frozen=True, slots=True)
class PollFailure:
    kind: PollFailureKind
    exception_kind: str | None = None


@dataclass(frozen=True, slots=True)
class OwnedPollReport(Generic[ItemT]):
    items: tuple[ItemT, ...]
    polls: int
    stop: PollStopKind
    primary_failure: PollFailure | None
    close_failure: PollFailure | None


def _stable_exception_kind(error: Exception) -> str:
    name = type(error).__name__
    return name if _EXCEPTION_KIND(name) is not None else "CallbackError"


def _raised_failure(
    kind: PollFailureKind,
    error: Exception,
) -> PollFailure:
    return PollFailure(kind, _stable_exception_kind(error))


def _sample_clock(
    clock: Callable[[], float],
    previous: float | None,
) -> tuple[float | None, PollFailure | None]:
    try:
        raw = clock()
    except Exception as error:
        return None, _raised_failure(PollFailureKind.CLOCK_RAISED, error)
    if type(raw) not in (int, float):
        return None, PollFailure(PollFailureKind.CLOCK_VALUE_INVALID)
    try:
        current = float(raw)
    except OverflowError:
        return None, PollFailure(PollFailureKind.CLOCK_VALUE_INVALID)
    if not math.isfinite(current):
        return None, PollFailure(PollFailureKind.CLOCK_VALUE_INVALID)
    if previous is not None and current < previous:
        return None, PollFailure(PollFailureKind.CLOCK_MOVED_BACKWARD)
    return current, None


def _read_once(
    source: OwnedPollSource[ItemT],
) -> tuple[object, PollFailure | None]:
    try:
        return source.read(), None
    except Exception as error:
        return None, _raised_failure(PollFailureKind.READ_RAISED, error)


def _wait_once(
    wait: Callable[[float], None],
    remaining: float,
) -> PollFailure | None:
    try:
        wait(remaining)
    except Exception as error:
        return _raised_failure(PollFailureKind.WAIT_RAISED, error)
    return None


def _close_once(source: OwnedPollSource[ItemT]) -> PollFailure | None:
    try:
        source.close()
    except Exception as error:
        return _raised_failure(PollFailureKind.CLOSE_RAISED, error)
    return None


def poll_owned_source(
    source: OwnedPollSource[ItemT],
    *,
    idle_timeout: int | float,
    poll_budget: int,
    clock: Callable[[], float],
    wait: Callable[[float], None],
) -> OwnedPollReport[ItemT]:
    if type(source) is not OwnedPollSource:
        raise TypeError("source must be an exact OwnedPollSource")
    if not callable(source.read) or not callable(source.close):
        raise TypeError("source read and close callbacks must be callable")
    if type(idle_timeout) not in (int, float):
        raise TypeError("idle_timeout must be an exact int or float")
    try:
        timeout = float(idle_timeout)
    except OverflowError:
        raise ValueError("idle_timeout is not representable as a float") from None
    if not math.isfinite(timeout) or not _MIN_IDLE_TIMEOUT <= timeout <= _MAX_IDLE_TIMEOUT:
        raise ValueError("idle_timeout must be finite and from 0.01 through 3600")
    if type(poll_budget) is not int or not 1 <= poll_budget <= _MAX_POLLS:
        raise ValueError("poll_budget must be an integer from 1 through 10000")
    if not callable(clock) or not callable(wait):
        raise TypeError("clock and wait must be callable")

    items: list[ItemT] = []
    polls = 0
    stop = PollStopKind.FAILED
    primary_failure: PollFailure | None = None
    idle_deadline: float | None = None
    try:
        last_time, primary_failure = _sample_clock(clock, None)

        while primary_failure is None:
            if polls == poll_budget:
                stop = PollStopKind.POLL_BUDGET_EXHAUSTED
                break

            polls += 1
            outcome, primary_failure = _read_once(source)
            if primary_failure is not None:
                break
            if type(outcome) not in (Item, Idle, End):
                primary_failure = PollFailure(PollFailureKind.INVALID_OUTCOME)
                break

            now, primary_failure = _sample_clock(clock, last_time)
            if primary_failure is not None:
                break
            if now is None:
                raise AssertionError("a valid clock sample is missing")
            last_time = now

            if idle_deadline is not None and now >= idle_deadline:
                stop = PollStopKind.IDLE_TIMEOUT
                break
            if type(outcome) is Item:
                items.append(outcome.value)
                idle_deadline = None
                continue
            if type(outcome) is End:
                stop = PollStopKind.END
                break

            if idle_deadline is None:
                idle_deadline = now + timeout
                if not math.isfinite(idle_deadline):
                    primary_failure = PollFailure(
                        PollFailureKind.CLOCK_VALUE_INVALID
                    )
                    break
            if now >= idle_deadline:
                stop = PollStopKind.IDLE_TIMEOUT
                break

            primary_failure = _wait_once(wait, idle_deadline - now)
            if primary_failure is not None:
                break
            after_wait, primary_failure = _sample_clock(clock, last_time)
            if primary_failure is not None:
                break
            if after_wait is None:
                raise AssertionError("a valid clock sample is missing")
            last_time = after_wait
            if after_wait >= idle_deadline:
                stop = PollStopKind.IDLE_TIMEOUT
                break
    except BaseException as primary_error:
        try:
            source.close()
        except BaseException as close_error:
            raise BaseExceptionGroup(
                "polling and owned-source close both raised",
                (primary_error, close_error),
            ) from None
        raise

    close_failure = _close_once(source)
    return OwnedPollReport(
        items=tuple(items),
        polls=polls,
        stop=stop,
        primary_failure=primary_failure,
        close_failure=close_failure,
    )
```

## Example

```python
class ManualClock:
    def __init__(self, early_wakes: tuple[float, ...] = ()) -> None:
        self.now = 0.0
        self.early_wakes = list(early_wakes)
        self.waits: list[float] = []

    def monotonic(self) -> float:
        return self.now

    def wait(self, maximum: float) -> None:
        self.waits.append(maximum)
        advance = self.early_wakes.pop(0) if self.early_wakes else maximum
        self.now += min(advance, maximum)


class ScriptedSource:
    def __init__(self, outcomes: tuple[Item[str] | Idle | End, ...]) -> None:
        self.outcomes = outcomes
        self.index = 0
        self.close_calls = 0

    def read(self) -> Item[str] | Idle | End:
        if self.index == len(self.outcomes):
            return End()
        outcome = self.outcomes[self.index]
        self.index += 1
        return outcome

    def close(self) -> None:
        self.close_calls += 1


manual = ManualClock(early_wakes=(0.5,))
scripted = ScriptedSource(
    (Item("first"), Idle(), Item("second"), Idle())
)
report = poll_owned_source(
    OwnedPollSource(scripted.read, scripted.close),
    idle_timeout=2,
    poll_budget=20,
    clock=manual.monotonic,
    wait=manual.wait,
)


class FailingSource:
    def __init__(self) -> None:
        self.close_calls = 0

    def read(self) -> Item[str] | Idle | End:
        raise LookupError("private read detail")

    def close(self) -> None:
        self.close_calls += 1
        raise OSError("private close detail")


failing = FailingSource()
failure_report = poll_owned_source(
    OwnedPollSource(failing.read, failing.close),
    idle_timeout=1,
    poll_budget=5,
    clock=ManualClock().monotonic,
    wait=lambda _remaining: None,
)


class BackwardClock:
    def __init__(self) -> None:
        self.samples = iter((4.0, 3.0))

    def monotonic(self) -> float:
        return next(self.samples)


backward_source = ScriptedSource((Idle(),))
backward_report = poll_owned_source(
    OwnedPollSource(backward_source.read, backward_source.close),
    idle_timeout=1,
    poll_budget=5,
    clock=BackwardClock().monotonic,
    wait=lambda _remaining: None,
)


class ExactDeadlineSource:
    def __init__(self, manual_clock: ManualClock) -> None:
        self.manual_clock = manual_clock
        self.reads = 0
        self.close_calls = 0

    def read(self) -> Item[str] | Idle | End:
        self.reads += 1
        if self.reads == 1:
            return Idle()
        self.manual_clock.now = 1.0
        return Item("at-the-deadline")

    def close(self) -> None:
        self.close_calls += 1


boundary_clock = ManualClock(early_wakes=(0.5,))
boundary_source = ExactDeadlineSource(boundary_clock)
boundary_report = poll_owned_source(
    OwnedPollSource(boundary_source.read, boundary_source.close),
    idle_timeout=1,
    poll_budget=5,
    clock=boundary_clock.monotonic,
    wait=boundary_clock.wait,
)


class InterruptedSource:
    def __init__(self) -> None:
        self.close_calls = 0

    def read(self) -> Item[str] | Idle | End:
        raise KeyboardInterrupt

    def close(self) -> None:
        self.close_calls += 1


interrupted_source = InterruptedSource()
try:
    poll_owned_source(
        OwnedPollSource(interrupted_source.read, interrupted_source.close),
        idle_timeout=1,
        poll_budget=5,
        clock=ManualClock().monotonic,
        wait=lambda _remaining: None,
    )
except KeyboardInterrupt:
    interrupt_propagated = True
else:
    interrupt_propagated = False

assert (
    report.items,
    report.polls,
    report.stop,
    manual.waits,
    manual.now,
    scripted.close_calls,
    failure_report.primary_failure,
    failure_report.close_failure,
    "private" not in repr(failure_report),
    failing.close_calls,
    backward_report.primary_failure,
    backward_source.close_calls,
    boundary_report.items,
    boundary_report.polls,
    boundary_report.stop,
    boundary_source.close_calls,
    interrupt_propagated,
    interrupted_source.close_calls,
) == (
    ("first", "second"),
    4,
    PollStopKind.IDLE_TIMEOUT,
    [2.0, 2.0],
    2.5,
    1,
    PollFailure(PollFailureKind.READ_RAISED, "LookupError"),
    PollFailure(PollFailureKind.CLOSE_RAISED, "OSError"),
    True,
    1,
    PollFailure(PollFailureKind.CLOCK_MOVED_BACKWARD),
    1,
    (),
    2,
    PollStopKind.IDLE_TIMEOUT,
    1,
    True,
    1,
)
```

## Trade-offs and Limitations

The operation is eager and retains at most 10,000 item references. Frozen
outcome and report records plus tuple storage prevent structural mutation, but
an `Item` payload may itself be mutable; copy or freeze payloads at the source
boundary when the caller needs deep immutability. A poll that returns an item
on the last permitted call keeps that item, then the next loop check reports
poll-budget exhaustion.

Every operational exit shown by the report makes exactly one close callback:
normal end, exact idle timeout, exhausted poll budget, invalid outcome,
backward or invalid clock, and ordinary read, wait, or clock failure. Primary
and close failures occupy separate fields, and only stable enum values plus a
bounded exception type name are retained. Messages, exception objects,
tracebacks, and frames are discarded. Input validation happens before
ownership begins and therefore does not call `close`.

Only `Exception` subclasses become failure records. A process-control
`BaseException` from polling triggers exactly one close attempt and then
propagates. If that close attempt also raises, a `BaseExceptionGroup` propagates
both live failures; this exceptional path is deliberately outside the frozen,
no-retained-exception report contract. The wait callback may return early and
can be invoked many times; if it returns without time advancing or readiness
changing, the poll budget still terminates the loop. The idle timeout does not
bound one blocking `read`, `wait`, or `close` call, so each external adapter
needs its own finite blocking policy.

## Related Snippets

<!-- catalog:related:start -->
- [Stop a Polling Worker Cooperatively with an Event](stop-a-polling-worker-cooperatively-with-an-event.md)
- [Own One External Mount Lease with Exception-Safe Cleanup](own-one-external-mount-lease-with-exception-safe-cleanup.md)
- [Wait for a Predicate Until a Monotonic Deadline](../reliability-resilience/wait-for-a-predicate-until-a-monotonic-deadline.md)
<!-- catalog:related:end -->
