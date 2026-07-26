---
title: "Resolve the Latest Status with an Explicit Mapping"
snippet_type: pattern
use_cases:
  - interoperability
  - observability
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - measure-and-freeze-elapsed-time-in-a-context.md
  - scope-structured-log-fields-with-context-variables.md
  - ../data-processing/measure-time-in-a-state-within-a-half-open-window.md
---

# Resolve the Latest Status with an Explicit Mapping

## Idea and Problem

Resolve a current normalized status from a finite unordered event log without leaking external status names into the caller's state model.

Every encountered event is validated through an explicit mapping during one
scan. The greatest integer sequence wins, and later input wins an equal-
sequence tie. The result also records the winning source position; an empty
log returns the caller's explicit initial state rather than inventing an
external status.

## When to Use

Use this pattern when an integration exposes a bounded status history but no
separate current-state field, and one totally ordered sequence determines
recency. Keep the mapping near the integration boundary and update it when the
external vocabulary changes. Use a real transition reducer instead when
earlier events affect state, sequence values are only partially ordered, or
history may be unbounded.

## Implementation

```python
from collections.abc import Iterable, Mapping
from dataclasses import dataclass


@dataclass(frozen=True, slots=True)
class StatusEvent:
    sequence: int
    status: str
    message: str | None = None


@dataclass(frozen=True, slots=True)
class StatusSnapshot:
    state: str
    message: str | None
    sequence: int | None
    source_index: int | None


def resolve_latest_status(
    events: Iterable[StatusEvent],
    *,
    state_by_status: Mapping[str, str],
    initial_state: str,
) -> StatusSnapshot:
    if not isinstance(state_by_status, Mapping):
        raise TypeError("state_by_status must be a mapping")
    if not isinstance(initial_state, str) or not initial_state.strip():
        raise ValueError("initial_state must be non-empty text")

    latest: StatusSnapshot | None = None
    for source_index, event in enumerate(events):
        if not isinstance(event, StatusEvent):
            raise TypeError("events must contain StatusEvent values")
        if isinstance(event.sequence, bool) or not isinstance(event.sequence, int):
            raise TypeError("event sequences must be integers")
        if not isinstance(event.status, str) or not event.status:
            raise ValueError("event statuses must be non-empty text")
        if event.message is not None and not isinstance(event.message, str):
            raise TypeError("event messages must be text or None")

        try:
            state = state_by_status[event.status]
        except KeyError as error:
            raise ValueError(f"unknown external status: {event.status!r}") from error
        if not isinstance(state, str) or not state.strip():
            raise ValueError("mapped states must be non-empty text")

        candidate = StatusSnapshot(
            state=state,
            message=event.message,
            sequence=event.sequence,
            source_index=source_index,
        )
        if latest is None or (
            candidate.sequence,
            candidate.source_index,
        ) > (
            latest.sequence,
            latest.source_index,
        ):
            latest = candidate

    return latest or StatusSnapshot(initial_state, None, None, None)
```

## Example

```python
states = {
    "waiting": "pending",
    "working": "active",
    "finished": "complete",
}
snapshot = resolve_latest_status(
    (
        StatusEvent(7, "working", "first event at sequence seven"),
        StatusEvent(3, "waiting"),
        StatusEvent(7, "finished", "later tie"),
    ),
    state_by_status=states,
    initial_state="pending",
)
empty = resolve_latest_status(
    (),
    state_by_status=states,
    initial_state="pending",
)

try:
    resolve_latest_status(
        (StatusEvent(1, "unexpected"), StatusEvent(2, "finished")),
        state_by_status=states,
        initial_state="pending",
    )
except ValueError:
    unknown_rejected = True
else:
    unknown_rejected = False

assert (
    snapshot,
    empty,
    unknown_rejected,
) == (
    StatusSnapshot("complete", "later tie", 7, 2),
    StatusSnapshot("pending", None, None, None),
    True,
)
```

## Trade-offs and Limitations

The scan uses `O(1)` additional state, but it must finish before returning and
therefore cannot resolve an infinite stream. Strictly validating every event
means an unknown old status fails even when a newer known event would win;
this exposes contract drift instead of hiding it. Equal sequences deliberately
make input order significant. The helper assumes sequence values are globally
comparable and trustworthy. It does not validate legal transitions, replay
side effects, detect stale histories, persist checkpoints, or implement a
general event-sourcing fold.

## Related Snippets

<!-- catalog:related:start -->
- [Measure and Freeze Elapsed Time in a Context](measure-and-freeze-elapsed-time-in-a-context.md)
- [Scope Structured Log Fields with Context Variables](scope-structured-log-fields-with-context-variables.md)
- [Measure Time in a State Within a Half-Open Window](../data-processing/measure-time-in-a-state-within-a-half-open-window.md)
<!-- catalog:related:end -->
