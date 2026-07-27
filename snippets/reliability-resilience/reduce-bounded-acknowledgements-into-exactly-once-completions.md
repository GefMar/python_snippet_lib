---
title: "Reduce Bounded Acknowledgements into Exactly-Once Completions"
snippet_type: algorithm
use_cases:
  - retry-recovery
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - retry-only-eligible-items-in-a-bounded-batch.md
  - model-independent-blocking-reasons-as-an-immutable-set.md
  - ../concurrency-lifecycle/elect-one-final-releaser-from-bounded-named-leases.md
---

# Reduce Bounded Acknowledgements into Exactly-Once Completions

## Idea and Problem

Reconcile a complete bounded acknowledgement log into one completion per item and an exact account of what remains missing.

Acknowledgements can be repeated when a producer retries or a log contains
duplicates. Treating each item-source pair as a fact makes replay idempotent,
while recording only the event that first satisfies every required source
preserves a deterministic completion order.

## When to Use

Use this algorithm after a finite acknowledgement history has been collected
and the complete sets of expected items and required sources are known. IDs
must be stable conservative ASCII tokens, and input order must be meaningful:
event order determines first completion, while declaration order determines
pending and missing-source order.

This reducer classifies the supplied log only. Use a separate coordination
mechanism when callers must wait for acknowledgements that have not arrived.

## Implementation

```python
import re
from dataclasses import dataclass


_MAX_REQUIRED_SOURCES = 16
_MAX_EXPECTED_ITEMS = 512
_MAX_ACKNOWLEDGEMENTS = 8_192
_ID = re.compile(r"[a-z][a-z0-9._:-]{0,63}", re.ASCII)


@dataclass(frozen=True, slots=True)
class Acknowledgement:
    item_id: str
    source_id: str


@dataclass(frozen=True, slots=True)
class PendingAcknowledgements:
    item_id: str
    missing_sources: tuple[str, ...]


@dataclass(frozen=True, slots=True)
class AcknowledgementReconciliation:
    completed_items: tuple[str, ...]
    pending_items: tuple[PendingAcknowledgements, ...]


def _validated_id(value: object, *, field: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if _ID.fullmatch(value) is None:
        raise ValueError(f"{field} must be a 1-64 byte conservative ASCII ID")
    return value


def _validated_unique_ids(
    value: object,
    *,
    field: str,
    maximum: int,
) -> tuple[str, ...]:
    if type(value) is not tuple:
        raise TypeError(f"{field} must be an exact tuple")
    if not 1 <= len(value) <= maximum:
        raise ValueError(f"{field} count must be between 1 and {maximum}")

    validated: list[str] = []
    seen: set[str] = set()
    for index, raw_id in enumerate(value):
        identifier = _validated_id(raw_id, field=f"{field}[{index}]")
        if identifier in seen:
            raise ValueError(f"{field} must contain unique IDs")
        seen.add(identifier)
        validated.append(identifier)
    return tuple(validated)


def _validated_acknowledgements(value: object) -> tuple[Acknowledgement, ...]:
    if type(value) is not tuple:
        raise TypeError("acknowledgements must be an exact tuple")
    if len(value) > _MAX_ACKNOWLEDGEMENTS:
        raise ValueError(
            f"acknowledgements must contain at most {_MAX_ACKNOWLEDGEMENTS} events"
        )

    validated: list[Acknowledgement] = []
    for index, event in enumerate(value):
        if type(event) is not Acknowledgement:
            raise TypeError(
                f"acknowledgements[{index}] must be an exact Acknowledgement"
            )
        item_id = _validated_id(
            event.item_id,
            field=f"acknowledgements[{index}].item_id",
        )
        source_id = _validated_id(
            event.source_id,
            field=f"acknowledgements[{index}].source_id",
        )
        validated.append(Acknowledgement(item_id, source_id))
    return tuple(validated)


def reduce_acknowledgements(
    required_sources: tuple[str, ...],
    expected_items: tuple[str, ...],
    acknowledgements: tuple[Acknowledgement, ...],
) -> AcknowledgementReconciliation:
    """Validate and reconcile one complete immutable acknowledgement log."""
    sources = _validated_unique_ids(
        required_sources,
        field="required_sources",
        maximum=_MAX_REQUIRED_SOURCES,
    )
    items = _validated_unique_ids(
        expected_items,
        field="expected_items",
        maximum=_MAX_EXPECTED_ITEMS,
    )
    events = _validated_acknowledgements(acknowledgements)

    known_sources = frozenset(sources)
    known_items = frozenset(items)
    for index, event in enumerate(events):
        if event.item_id not in known_items:
            raise ValueError(
                f"acknowledgements[{index}] refers to an unknown item"
            )
        if event.source_id not in known_sources:
            raise ValueError(
                f"acknowledgements[{index}] refers to an unknown source"
            )

    received = {item_id: set() for item_id in items}
    completed: list[str] = []
    completed_set: set[str] = set()
    for event in events:
        received[event.item_id].add(event.source_id)
        if (
            event.item_id not in completed_set
            and len(received[event.item_id]) == len(sources)
        ):
            completed.append(event.item_id)
            completed_set.add(event.item_id)

    pending = tuple(
        PendingAcknowledgements(
            item_id,
            tuple(
                source_id
                for source_id in sources
                if source_id not in received[item_id]
            ),
        )
        for item_id in items
        if item_id not in completed_set
    )
    return AcknowledgementReconciliation(tuple(completed), pending)
```

## Example

```python
result = reduce_acknowledgements(
    required_sources=("reader", "checker", "archiver"),
    expected_items=("unit-1", "unit-2", "unit-3"),
    acknowledgements=(
        Acknowledgement("unit-2", "reader"),
        Acknowledgement("unit-1", "reader"),
        Acknowledgement("unit-2", "checker"),
        Acknowledgement("unit-2", "archiver"),
        Acknowledgement("unit-2", "reader"),  # repeated after completion
        Acknowledgement("unit-1", "checker"),
        Acknowledgement("unit-1", "archiver"),
        Acknowledgement("unit-1", "checker"),  # repeated after completion
        Acknowledgement("unit-3", "archiver"),
    ),
)

assert result == AcknowledgementReconciliation(
    completed_items=("unit-2", "unit-1"),
    pending_items=(
        PendingAcknowledgements(
            item_id="unit-3",
            missing_sources=("reader", "checker"),
        ),
    ),
)
```

## Trade-offs and Limitations

Validation and reduction are linear in the log size, with bounded state for
512 items, 16 sources, and 8,192 events. Every event is validated before
reduction, so an invalid late event rejects the whole log even if all items
completed earlier. Duplicate events remain accepted facts and do not create a
second completion.

"Exactly once" describes each item's appearance in this frozen report, not
exactly-once delivery or execution of an external effect. The function does
not mutate a live barrier, write a checkpoint, persist state, manage a queue,
poll, use clocks or expiry, perform I/O, or provide thread safety. A later log
can produce a different reconciliation, so callers that need durable progress
must define that boundary separately.

## Related Snippets

<!-- catalog:related:start -->
- [Retry Only Eligible Items in a Bounded Batch](retry-only-eligible-items-in-a-bounded-batch.md)
- [Model Independent Blocking Reasons as an Immutable Set](model-independent-blocking-reasons-as-an-immutable-set.md)
- [Elect One Final Releaser from Bounded Named Leases](../concurrency-lifecycle/elect-one-final-releaser-from-bounded-named-leases.md)
<!-- catalog:related:end -->
