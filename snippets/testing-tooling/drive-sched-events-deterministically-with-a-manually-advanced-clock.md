---
title: "Drive sched Events Deterministically with a Manually Advanced Clock"
snippet_type: testing-technique
use_cases:
  - testing
  - lifecycle-management
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - wait-for-named-queue-conditions-under-one-monotonic-deadline.md
  - ../concurrency-lifecycle/refresh-bounded-named-targets-at-their-nearest-monotonic-deadline.md
  - ../concurrency-lifecycle/run-an-async-worker-on-clock-aligned-ticks-without-catch-up.md
---

# Drive sched Events Deterministically with a Manually Advanced Clock

## Idea and Problem

Exercise a finite Python scheduler scenario without sleeping by injecting an integer clock whose delay function advances virtual time immediately.

Each event uses a fixed recording callback, so the result exposes only
deadline, priority, cancellation, and ordering behavior. Equal deadlines run by
ascending priority. For equal deadline and priority, the example also records
CPython 3.14's sequence-counter tie behavior in original scheduling order.

## When to Use

Use this technique for deterministic unit tests of a small schedule assembled
from already-validated event specifications. It lets a test cover distant
deadlines and pre-run cancellation without depending on machine speed or a
wall-clock sleep.

Use a production clock abstraction, event loop, or concurrency test harness
when callbacks block, create more work, cross thread boundaries, or observe
real time. Exact FIFO ties are tested CPython 3.14 behavior, not a portability
promise of the documented deadline-and-priority scheduler contract.

## Implementation

```python
import re
import sched
from dataclasses import dataclass

_MAX_EVENTS = 64
_MAX_DEADLINE = 10**9
_MAX_PRIORITY = 10**6
_LABEL = re.compile(r"[A-Za-z_][A-Za-z0-9_]{0,63}", re.ASCII)


def _validated_label(value: object, *, name: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{name} must be exact text")
    if _LABEL.fullmatch(value) is None:
        raise ValueError(f"{name} must be an ASCII identifier")
    return value


@dataclass(frozen=True, slots=True)
class ScheduledEvent:
    label: str
    deadline: int
    priority: int

    def __post_init__(self) -> None:
        _validated_label(self.label, name="label")
        if type(self.deadline) is not int:
            raise TypeError("deadline must be an exact integer")
        if not 0 <= self.deadline <= _MAX_DEADLINE:
            raise ValueError("deadline is outside the supported range")
        if type(self.priority) is not int:
            raise TypeError("priority must be an exact integer")
        if not 0 <= self.priority <= _MAX_PRIORITY:
            raise ValueError("priority is outside the supported range")


@dataclass(frozen=True, slots=True)
class EventExecution:
    label: str
    virtual_time: int


@dataclass(frozen=True, slots=True)
class ScheduleTrace:
    executions: tuple[EventExecution, ...]
    final_time: int


class _ManualClock:
    def __init__(self) -> None:
        self.value = 0

    def now(self) -> int:
        return self.value

    def advance(self, delay: int) -> None:
        if type(delay) is not int or delay < 0:
            raise RuntimeError("sched requested an invalid virtual delay")
        self.value += delay


def drive_sched_events(
    events: tuple[ScheduledEvent, ...],
    *,
    cancel: tuple[str, ...] = (),
) -> ScheduleTrace:
    if type(events) is not tuple:
        raise TypeError("events must be an exact tuple")
    if len(events) > _MAX_EVENTS:
        raise ValueError("events exceed the supported count")
    if type(cancel) is not tuple:
        raise TypeError("cancel must be an exact tuple")
    if len(cancel) > _MAX_EVENTS:
        raise ValueError("cancellations exceed the supported count")

    labels: set[str] = set()
    for event in events:
        if type(event) is not ScheduledEvent:
            raise TypeError("events must contain exact ScheduledEvent values")
        if event.label in labels:
            raise ValueError("event labels must be unique")
        labels.add(event.label)

    cancellation_labels: set[str] = set()
    for value in cancel:
        label = _validated_label(value, name="cancellation label")
        if label in cancellation_labels:
            raise ValueError("cancellation labels must be unique")
        if label not in labels:
            raise ValueError("cancellation labels must name events")
        cancellation_labels.add(label)

    clock = _ManualClock()
    scheduler = sched.scheduler(clock.now, clock.advance)
    executions: list[EventExecution] = []

    def record(label: str) -> None:
        executions.append(EventExecution(label, clock.now()))

    handles = {
        event.label: scheduler.enterabs(
            event.deadline,
            event.priority,
            record,
            argument=(event.label,),
        )
        for event in events
    }
    for label in cancel:
        scheduler.cancel(handles[label])

    scheduler.run()
    return ScheduleTrace(tuple(executions), clock.now())
```

## Example

```python
events = (
    ScheduledEvent("late", deadline=8, priority=0),
    ScheduledEvent("same_first", deadline=4, priority=2),
    ScheduledEvent("higher_priority", deadline=4, priority=1),
    ScheduledEvent("same_second", deadline=4, priority=2),
    ScheduledEvent("cancel_me", deadline=12, priority=0),
)

trace = drive_sched_events(events, cancel=("cancel_me",))
empty_trace = drive_sched_events(())

assert trace == ScheduleTrace(
    executions=(
        EventExecution("higher_priority", 4),
        EventExecution("same_first", 4),
        EventExecution("same_second", 4),
        EventExecution("late", 8),
    ),
    final_time=8,
)
assert empty_trace == ScheduleTrace((), 0)
```

## Trade-offs and Limitations

The function schedules at most 64 events and uses linear result storage plus
the scheduler's priority queue. Virtual time jumps directly to each next
deadline; `sched` also calls the delay function with zero after an event, which
does not change this clock. Cancellation occurs before execution and must name
a unique admitted event.

Only fixed, synchronous recording callbacks are installed. The technique does
not model callback duration, blocking, threads, async tasks, races, clock
adjustment, or scheduler lag. The public scheduler behavior orders events by
deadline and priority; relying on insertion order for complete ties requires
the explicitly tested CPython 3.14 implementation or caller-assigned distinct
priorities.

## Related Snippets

<!-- catalog:related:start -->
- [Wait for Named Queue Conditions Under One Monotonic Deadline](wait-for-named-queue-conditions-under-one-monotonic-deadline.md)
- [Refresh Bounded Named Targets at Their Nearest Monotonic Deadline](../concurrency-lifecycle/refresh-bounded-named-targets-at-their-nearest-monotonic-deadline.md)
- [Run an Async Worker on Clock-Aligned Ticks Without Catch-Up](../concurrency-lifecycle/run-an-async-worker-on-clock-aligned-ticks-without-catch-up.md)
<!-- catalog:related:end -->
