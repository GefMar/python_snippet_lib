---
title: "Suppress Stale Keyed Events with Strictly Increasing Sequence Numbers"
snippet_type: algorithm
use_cases:
  - data-transformation
  - retry-recovery
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../observability-operations/resolve-the-latest-status-with-an-explicit-mapping.md
  - ../data-processing/select-one-record-per-key-with-an-explicit-ranking-rule.md
  - ../algorithms-data-structures/accept-sparse-observations-that-preserve-strict-position-time-order.md
---

# Suppress Stale Keyed Events with Strictly Increasing Sequence Numbers

## Idea and Problem

Accept a bounded batch of key and sequence markers only when each marker advances the latest known sequence for its key.

Validate the complete current snapshot and event batch before making any
decisions. Then process events in supplied order: an unknown key or a strictly
greater sequence advances the state, while an equal or lower sequence is
stale. Return original event indexes so the decision remains inspectable
without attaching payload semantics to the reducer.

## When to Use

Use this reducer when each key has one trustworthy non-negative sequence that
must increase whenever its state advances. It fits a bounded, already
materialized replay or reconciliation step where arrival order is authoritative
and the caller needs only the latest sequence plus accepted and stale indexes.

Use a durable compare-and-swap or transaction when several writers can update
the same key. Choose a domain-specific reducer when payloads affect state, and
use event identities, gap tracking, or event-time watermarks when sequence
comparison alone cannot classify an event safely.

## Implementation

```python
import re
from dataclasses import dataclass

_KEY_MATCH = re.compile(r"[a-z][a-z0-9._:-]{0,63}", re.ASCII).fullmatch
_MAX_SEQUENCE = (1 << 63) - 1
_MAX_KEYS = 4_096
_MAX_EVENTS = 20_000


@dataclass(frozen=True, slots=True)
class KeySequence:
    key: str
    sequence: int


@dataclass(frozen=True, slots=True)
class KeyedSequenceEvent:
    key: str
    sequence: int


@dataclass(frozen=True, slots=True)
class KeyedSequenceReduction:
    latest: tuple[KeySequence, ...]
    accepted_event_indexes: tuple[int, ...]
    stale_event_indexes: tuple[int, ...]


def _validate_key_sequence(key: object, sequence: object, *, label: str) -> None:
    if type(key) is not str:
        raise TypeError(f"{label}.key must be an exact string")
    if _KEY_MATCH(key) is None:
        raise ValueError(f"{label}.key must be a conservative ASCII identifier")
    if type(sequence) is not int:
        raise TypeError(f"{label}.sequence must be an exact integer")
    if not 0 <= sequence <= _MAX_SEQUENCE:
        raise ValueError(f"{label}.sequence is outside the supported range")


def suppress_stale_keyed_events(
    current: tuple[KeySequence, ...],
    events: tuple[KeyedSequenceEvent, ...],
) -> KeyedSequenceReduction:
    """Return a canonical latest snapshot and arrival-order decisions."""
    if type(current) is not tuple:
        raise TypeError("current must be an exact tuple")
    if type(events) is not tuple:
        raise TypeError("events must be an exact tuple")
    if len(current) > _MAX_KEYS:
        raise ValueError("current exceeds the supported key count")
    if len(events) > _MAX_EVENTS:
        raise ValueError("events exceed the supported event count")

    current_keys: set[str] = set()
    for index, entry in enumerate(current):
        if type(entry) is not KeySequence:
            raise TypeError(f"current[{index}] must be an exact KeySequence")
        _validate_key_sequence(
            entry.key,
            entry.sequence,
            label=f"current[{index}]",
        )
        if entry.key in current_keys:
            raise ValueError("current keys must be unique")
        current_keys.add(entry.key)

    distinct_keys = set(current_keys)
    for index, event in enumerate(events):
        if type(event) is not KeyedSequenceEvent:
            raise TypeError(f"events[{index}] must be an exact KeyedSequenceEvent")
        _validate_key_sequence(
            event.key,
            event.sequence,
            label=f"events[{index}]",
        )
        distinct_keys.add(event.key)
        if len(distinct_keys) > _MAX_KEYS:
            raise ValueError("current and events exceed the combined key limit")

    latest_by_key = {entry.key: entry.sequence for entry in current}
    accepted_indexes: list[int] = []
    stale_indexes: list[int] = []

    for index, event in enumerate(events):
        previous = latest_by_key.get(event.key)
        if previous is None or event.sequence > previous:
            latest_by_key[event.key] = event.sequence
            accepted_indexes.append(index)
        else:
            stale_indexes.append(index)

    return KeyedSequenceReduction(
        latest=tuple(KeySequence(key, latest_by_key[key]) for key in sorted(latest_by_key)),
        accepted_event_indexes=tuple(accepted_indexes),
        stale_event_indexes=tuple(stale_indexes),
    )
```

## Example

```python
current = (
    KeySequence("beta", 4),
    KeySequence("alpha", 2),
)
events = (
    KeyedSequenceEvent("alpha", 3),
    KeyedSequenceEvent("alpha", 3),
    KeyedSequenceEvent("beta", 2),
    KeyedSequenceEvent("gamma", 0),
    KeyedSequenceEvent("beta", 5),
    KeyedSequenceEvent("alpha", 4),
)

result = suppress_stale_keyed_events(current, events)
reordered = suppress_stale_keyed_events(
    (KeySequence("alpha", 2),),
    (
        KeyedSequenceEvent("alpha", 4),
        KeyedSequenceEvent("alpha", 3),
    ),
)

assert (result, reordered) == (
    KeyedSequenceReduction(
        latest=(
            KeySequence("alpha", 4),
            KeySequence("beta", 5),
            KeySequence("gamma", 0),
        ),
        accepted_event_indexes=(0, 3, 4, 5),
        stale_event_indexes=(1, 2),
    ),
    KeyedSequenceReduction(
        latest=(KeySequence("alpha", 4),),
        accepted_event_indexes=(0,),
        stale_event_indexes=(1,),
    ),
)
```

## Trade-offs and Limitations

Validation and reduction take `O(C + E)` time for `C` current entries and `E`
events. Sorting at most 4,096 final keys costs `O(K log K)` time. The state,
validation set, and returned decision indexes use `O(K + E)` memory within the
fixed limits.

Arrival order is part of the result. Reversing two increasing events can turn
the lower one from accepted into stale, so accepted and stale indexes are not
permutation invariant. Equal sequences are always stale; without payload or
event identity the reducer cannot distinguish an exact replay from a
conflicting reuse of a sequence. Sequence gaps are allowed.

Keys are exact, case-sensitive lowercase ASCII identifiers with a closed
64-byte grammar. The reducer has no clocks, payload transitions, persistence,
atomic side effects, compare-and-swap, concurrency control, or exactly-once
guarantee. It classifies one fully available batch and returns immutable data;
the caller owns every action based on that classification.

## Related Snippets

<!-- catalog:related:start -->
- [Resolve the Latest Status with an Explicit Mapping](../observability-operations/resolve-the-latest-status-with-an-explicit-mapping.md)
- [Select One Record per Key with an Explicit Ranking Rule](../data-processing/select-one-record-per-key-with-an-explicit-ranking-rule.md)
- [Accept Sparse Observations That Preserve Strict Position-Time Order](../algorithms-data-structures/accept-sparse-observations-that-preserve-strict-position-time-order.md)
<!-- catalog:related:end -->
