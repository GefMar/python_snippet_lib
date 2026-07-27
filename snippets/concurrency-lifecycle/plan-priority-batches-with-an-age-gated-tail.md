---
title: "Plan Priority Batches with an Age-Gated Tail"
snippet_type: algorithm
use_cases:
  - concurrency-control
  - lifecycle-management
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - run-bounded-thread-work-by-priority-and-submission-order.md
  - drain-bounded-deferred-writes-outside-the-queue-lock.md
  - ../data-processing/batch-items-by-estimated-byte-size.md
---

# Plan Priority Batches with an Age-Gated Tail

## Idea and Problem

Partition a bounded queue snapshot into priority-ordered ready batches and an exact immutable remainder without executing any work.

Items in the same canonical group share one priority and urgency policy. Full
batches are ready immediately; the final partial batch becomes ready only when
its oldest item has waited for at least `max_tail_age` ticks. Urgent groups are
considered first and do not consume the regular-batch budget, but every planned
batch still consumes the mandatory total budget.

## When to Use

Use this algorithm when one queue owner can take a finite immutable snapshot,
inject one trusted current tick, and atomically reconcile the returned plan
with the real queue. It is useful when partial batches should wait briefly for
more items without delaying already full batches.

Use a durable scheduler when multiple consumers race, work survives process
failure, or leases and in-flight suppression are required. Define a different
fairness policy when strict group priority can starve lower-ranked groups.

## Implementation

```python
import re
from collections.abc import Iterable
from dataclasses import dataclass
from itertools import islice


_MAX_ITEMS = 10_000
_MAX_BATCH_SIZE = 1_000
_MAX_BATCHES = 10_000
_MAX_TICK = (1 << 63) - 1
_MAX_TAIL_AGE = 1_000_000_000
_MAX_ABSOLUTE_PRIORITY = 1_000_000_000
_MAX_IDENTIFIER_LENGTH = 64
_IDENTIFIER = re.compile(r"[a-z][a-z0-9]*(?:-[a-z0-9]+)*\Z", re.ASCII)


def _canonical_identifier(value: object, *, name: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{name} must be text")
    if len(value) > _MAX_IDENTIFIER_LENGTH or _IDENTIFIER.fullmatch(value) is None:
        raise ValueError(f"{name} must be a canonical lowercase identifier")
    return value


def _exact_integer(
    value: object,
    *,
    name: str,
    minimum: int,
    maximum: int,
) -> int:
    if type(value) is not int:
        raise TypeError(f"{name} must be an integer")
    if not minimum <= value <= maximum:
        raise ValueError(f"{name} is outside the supported range")
    return value


@dataclass(frozen=True, slots=True)
class QueuedItem:
    item_id: str
    group: str
    priority: int
    urgent: bool
    enqueued_tick: int

    def __post_init__(self) -> None:
        _canonical_identifier(self.item_id, name="item_id")
        _canonical_identifier(self.group, name="group")
        _exact_integer(
            self.priority,
            name="priority",
            minimum=-_MAX_ABSOLUTE_PRIORITY,
            maximum=_MAX_ABSOLUTE_PRIORITY,
        )
        if type(self.urgent) is not bool:
            raise TypeError("urgent must be a boolean")
        _exact_integer(
            self.enqueued_tick,
            name="enqueued_tick",
            minimum=0,
            maximum=_MAX_TICK,
        )


@dataclass(frozen=True, slots=True)
class PlannedBatch:
    group: str
    priority: int
    urgent: bool
    full: bool
    items: tuple[QueuedItem, ...]


@dataclass(frozen=True, slots=True)
class PriorityBatchPlan:
    batches: tuple[PlannedBatch, ...]
    deferred: tuple[QueuedItem, ...]


@dataclass(slots=True)
class _GroupState:
    priority: int
    urgent: bool
    indexed_items: list[tuple[int, QueuedItem]]


def plan_priority_batches(
    items: Iterable[QueuedItem],
    *,
    now: int,
    batch_size: int,
    max_tail_age: int,
    max_regular_batches: int,
    max_total_batches: int,
) -> PriorityBatchPlan:
    current_tick = _exact_integer(
        now,
        name="now",
        minimum=0,
        maximum=_MAX_TICK,
    )
    size = _exact_integer(
        batch_size,
        name="batch_size",
        minimum=1,
        maximum=_MAX_BATCH_SIZE,
    )
    tail_age = _exact_integer(
        max_tail_age,
        name="max_tail_age",
        minimum=0,
        maximum=_MAX_TAIL_AGE,
    )
    regular_limit = _exact_integer(
        max_regular_batches,
        name="max_regular_batches",
        minimum=0,
        maximum=_MAX_BATCHES,
    )
    total_limit = _exact_integer(
        max_total_batches,
        name="max_total_batches",
        minimum=0,
        maximum=_MAX_BATCHES,
    )

    try:
        queued = tuple(islice(items, _MAX_ITEMS + 1))
    except TypeError as error:
        raise TypeError("items must be an iterable") from error
    if len(queued) > _MAX_ITEMS:
        raise ValueError("item count exceeds the supported limit")
    if any(type(item) is not QueuedItem for item in queued):
        raise TypeError("every item must be a QueuedItem")

    identifiers: set[str] = set()
    groups: dict[str, _GroupState] = {}
    for input_index, item in enumerate(queued):
        if item.item_id in identifiers:
            raise ValueError("item identifiers must be unique")
        identifiers.add(item.item_id)
        if item.enqueued_tick > current_tick:
            raise ValueError("an item was enqueued after now")

        group = groups.get(item.group)
        if group is None:
            groups[item.group] = _GroupState(
                priority=item.priority,
                urgent=item.urgent,
                indexed_items=[(input_index, item)],
            )
        else:
            if group.priority != item.priority or group.urgent is not item.urgent:
                raise ValueError("one group must have one priority and urgency policy")
            group.indexed_items.append((input_index, item))

    ordered_groups = sorted(
        groups.items(),
        key=lambda pair: (
            not pair[1].urgent,
            -pair[1].priority,
            pair[0],
        ),
    )

    batches: list[PlannedBatch] = []
    selected_ids: set[str] = set()
    regular_batches = 0
    for group_name, group in ordered_groups:
        ordered_items = tuple(
            item
            for _, item in sorted(
                group.indexed_items,
                key=lambda pair: (pair[1].enqueued_tick, pair[0]),
            )
        )
        for start in range(0, len(ordered_items), size):
            batch_items = ordered_items[start : start + size]
            full = len(batch_items) == size
            ready = full or current_tick - batch_items[0].enqueued_tick >= tail_age
            within_total_limit = len(batches) < total_limit
            within_regular_limit = group.urgent or regular_batches < regular_limit
            if not ready or not within_total_limit or not within_regular_limit:
                continue

            batches.append(
                PlannedBatch(
                    group=group_name,
                    priority=group.priority,
                    urgent=group.urgent,
                    full=full,
                    items=batch_items,
                )
            )
            selected_ids.update(item.item_id for item in batch_items)
            if not group.urgent:
                regular_batches += 1

    deferred = tuple(item for item in queued if item.item_id not in selected_ids)
    return PriorityBatchPlan(tuple(batches), deferred)
```

## Example

```python
items = (
    QueuedItem("alerts-c", "alerts", 3, True, 92),
    QueuedItem("alpha-b", "alpha", 8, False, 91),
    QueuedItem("alerts-a", "alerts", 3, True, 90),
    QueuedItem("young-a", "young", 10, False, 99),
    QueuedItem("alerts-b", "alerts", 3, True, 90),
    QueuedItem("beta-a", "beta", 5, False, 80),
    QueuedItem("alerts-d", "alerts", 3, True, 93),
    QueuedItem("alpha-a", "alpha", 8, False, 89),
    QueuedItem("alerts-tail", "alerts", 3, True, 99),
)

plan = plan_priority_batches(
    items,
    now=100,
    batch_size=2,
    max_tail_age=5,
    max_regular_batches=1,
    max_total_batches=3,
)
urgent_only = plan_priority_batches(
    items,
    now=100,
    batch_size=2,
    max_tail_age=5,
    max_regular_batches=0,
    max_total_batches=2,
)

try:
    plan_priority_batches(
        items + (QueuedItem("future-a", "future", 1, False, 101),),
        now=100,
        batch_size=2,
        max_tail_age=5,
        max_regular_batches=1,
        max_total_batches=3,
    )
except ValueError:
    future_item_rejected = True
else:
    future_item_rejected = False

assert (
    tuple(
        (batch.group, tuple(item.item_id for item in batch.items))
        for batch in plan.batches
    ),
    tuple(item.item_id for item in plan.deferred),
    tuple(batch.urgent for batch in urgent_only.batches),
    len(urgent_only.deferred),
    future_item_rejected,
) == (
    (
        ("alerts", ("alerts-a", "alerts-b")),
        ("alerts", ("alerts-c", "alerts-d")),
        ("alpha", ("alpha-a", "alpha-b")),
    ),
    ("young-a", "beta-a", "alerts-tail"),
    (True, True),
    5,
    True,
)
```

## Trade-offs and Limitations

Materialization is bounded by `_MAX_ITEMS`. Grouping is linear, and sorting
costs `O(n log n)` in the worst case. Returned batches follow urgent status,
descending priority, and canonical group name; items within one group follow
enqueue tick and then original snapshot position. Deferred items retain their
original snapshot order and appear exactly once outside the selected batches.

Priority ordering is not a fairness guarantee and can starve lower groups.
The plan is only a value: it does not read a clock, remove or enqueue items,
reserve capacity, persist state, suppress in-flight work, retry failures, or
coordinate competing consumers. The caller must atomically reconcile stable
item identifiers before acting on a plan.

## Related Snippets

<!-- catalog:related:start -->
- [Run Bounded Thread Work by Priority and Submission Order](run-bounded-thread-work-by-priority-and-submission-order.md)
- [Drain Bounded Deferred Writes Outside the Queue Lock](drain-bounded-deferred-writes-outside-the-queue-lock.md)
- [Batch Items by Estimated Byte Size](../data-processing/batch-items-by-estimated-byte-size.md)
<!-- catalog:related:end -->
