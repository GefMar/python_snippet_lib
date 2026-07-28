---
title: "Plan a Keyed Worker-Set Reload from a Coalesced SIGHUP Request"
snippet_type: pattern
use_cases:
  - automation
  - concurrency-control
  - lifecycle-management
tested_python:
  - "3.14"
dependencies: []
related:
  - stop-a-polling-worker-cooperatively-with-an-event.md
  - run-bounded-thread-work-by-priority-and-submission-order.md
---

# Plan a Keyed Worker-Set Reload from a Coalesced SIGHUP Request

## Idea and Problem

Turn one level-triggered reload request and two bounded worker-name snapshots into a frozen plan without reading configuration or touching any worker.

A module boolean lets a minimal SIGHUP handler record intent without acquiring
a Python lock. Several requests before the main-thread coordinator reaches a
safe point deliberately coalesce into one observation. The pure planner then
preserves desired order for retained and new names, and current order for
names that must stop.

## When to Use

Use this pattern in a Unix daemon whose worker membership is keyed by stable
names and reconciled by one lifecycle owner. The owner must already have exact
current and desired tuples and must define how workers start, become visible,
stop, and report failures.

Use a queue or counter when every request must be processed separately. Use a
supervisor or service manager when the safer reload policy is to replace the
complete process.

## Implementation

```python
import re
from dataclasses import dataclass


_MAX_WORKERS = 64
_WORKER_NAME = re.compile(r"[a-z][a-z0-9_-]{0,63}", re.ASCII)


_reload_requested = False


def sighup_handler(_signal_number: int, _frame: object) -> None:
    global _reload_requested
    _reload_requested = True


def take_coalesced_reload_request() -> bool:
    global _reload_requested
    if not _reload_requested:
        return False
    _reload_requested = False
    return True


def _validated_worker_names(
    names: object,
    *,
    field: str,
) -> tuple[str, ...]:
    if type(names) is not tuple:
        raise TypeError(f"{field} must be an exact tuple")
    if len(names) > _MAX_WORKERS:
        raise ValueError(f"{field} exceeds the worker limit")

    validated: list[str] = []
    seen: set[str] = set()
    for name in names:
        if type(name) is not str:
            raise TypeError(f"{field} must contain exact strings")
        if _WORKER_NAME.fullmatch(name) is None:
            raise ValueError(f"{field} contains an invalid worker name")
        if name in seen:
            raise ValueError(f"{field} contains a duplicate worker name")
        seen.add(name)
        validated.append(name)
    return tuple(validated)


@dataclass(frozen=True, slots=True)
class WorkerSetReloadPlan:
    current: tuple[str, ...]
    desired: tuple[str, ...]
    retained: tuple[str, ...]
    start: tuple[str, ...]
    stop: tuple[str, ...]


def plan_worker_set_reload(
    current: tuple[str, ...],
    desired: tuple[str, ...],
) -> WorkerSetReloadPlan:
    current_names = _validated_worker_names(current, field="current")
    desired_names = _validated_worker_names(desired, field="desired")
    current_set = frozenset(current_names)
    desired_set = frozenset(desired_names)

    return WorkerSetReloadPlan(
        current=current_names,
        desired=desired_names,
        retained=tuple(name for name in desired_names if name in current_set),
        start=tuple(name for name in desired_names if name not in current_set),
        stop=tuple(name for name in current_names if name not in desired_set),
    )
```

## Example

```python
from dataclasses import FrozenInstanceError


sighup_handler(1, None)
sighup_handler(1, None)
request_taken = take_coalesced_reload_request()
request_coalesced = not take_coalesced_reload_request()

plan = plan_worker_set_reload(
    ("edge-a", "legacy", "edge-b"),
    ("edge-b", "edge-c", "edge-a"),
)
empty_plan = plan_worker_set_reload((), ())

try:
    plan.start = ()
except FrozenInstanceError:
    frozen = True
else:
    frozen = False

try:
    plan_worker_set_reload(("edge-a", "edge-a"), ())
except ValueError:
    duplicate_rejected = True
else:
    duplicate_rejected = False

assert (
    request_taken,
    request_coalesced,
    plan,
    empty_plan,
    frozen,
    duplicate_rejected,
) == (
    True,
    True,
    WorkerSetReloadPlan(
        current=("edge-a", "legacy", "edge-b"),
        desired=("edge-b", "edge-c", "edge-a"),
        retained=("edge-b", "edge-a"),
        start=("edge-c",),
        stop=("legacy",),
    ),
    WorkerSetReloadPlan((), (), (), (), ()),
    True,
    True,
)
```

## Trade-offs and Limitations

The boolean is level-triggered rather than counted. A `True` value consumed at
the safe point represents every request recorded so far; a handler assignment
after it is reset remains pending for the next safe point. The handler,
`take_coalesced_reload_request()`, and any `signal.signal` installation are
main-thread-only. The boolean is not a synchronization primitive and must not
be read or written by worker threads. SIGHUP is Unix-specific, and this
pattern intentionally neither installs a handler nor delivers a real signal.

The planner performs `O(n)` work over at most 64 current and 64 desired names.
It compares membership only; configuration changes for a retained name need a
separate version or replacement policy. It performs no configuration I/O,
worker calls, logging, retry, loop, or shutdown.

The lifecycle owner should validate the desired snapshot first, start every
name in `start`, and publish the desired set only after all starts succeed. A
start failure should clean up only workers created by that attempt and leave
the current set published. After publication, names in `stop` should retire
under an explicit deadline, with cleanup failures retained for observation or
retry. Those policies cannot be made safe by this pure plan alone.

## Related Snippets

<!-- catalog:related:start -->
- [Stop a Polling Worker Cooperatively with an Event](stop-a-polling-worker-cooperatively-with-an-event.md)
- [Run Bounded Thread Work by Priority and Submission Order](run-bounded-thread-work-by-priority-and-submission-order.md)
<!-- catalog:related:end -->
