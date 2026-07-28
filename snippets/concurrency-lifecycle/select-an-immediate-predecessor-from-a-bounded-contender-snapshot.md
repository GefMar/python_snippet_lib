---
title: "Select an Immediate Predecessor from a Bounded Contender Snapshot"
snippet_type: algorithm
use_cases:
  - concurrency-control
  - lifecycle-management
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - report-equal-share-lease-ownership-excess-from-a-bounded-queue-snapshot.md
  - plan-bounded-worker-replacements-from-generation-and-restart-state.md
  - ../reliability-resilience/decide-whether-a-bounded-work-snapshot-permits-a-new-attempt.md
---

# Select an Immediate Predecessor from a Bounded Contender Snapshot

## Idea and Problem

Select at most one immediate active predecessor from a complete, revisioned, bounded contender snapshot.

Each contender has a unique bounded ID, a unique nonnegative sequence
position, and an explicit `ACTIVE` or `INACTIVE` session state. After validating
the immutable tuple's canonical order, the classifier emits `FIRST`,
`WAIT_ON_PREDECESSOR`, `LOST`, or `REPLAN`; it never performs the coordination
operation that might follow.

## When to Use

Use this algorithm after an external coordinator returns one finite contender
snapshot with an explicit completeness marker and revision. Supply the current
contender ID and the oldest revision the consumer is willing to interpret.

An incomplete snapshot or revision below that minimum produces `REPLAN`.
Within an acceptable complete snapshot, a missing or inactive self produces
`LOST`; the first active contender produces `FIRST`; every later active
contender receives only the closest preceding active contender.

## Implementation

```python
import re
from dataclasses import dataclass
from enum import StrEnum

_MAX_CONTENDERS = 256
_MAX_ID_BYTES = 64
_MAX_TOTAL_ID_BYTES = 16 * 1024
_MAX_INTEGER = 2**63 - 1
_ID = re.compile(r"[A-Za-z0-9][A-Za-z0-9._:-]{0,63}", re.ASCII)


class SnapshotCompleteness(StrEnum):
    COMPLETE = "COMPLETE"
    INCOMPLETE = "INCOMPLETE"


class ContenderState(StrEnum):
    ACTIVE = "ACTIVE"
    INACTIVE = "INACTIVE"


class PredecessorOutcome(StrEnum):
    FIRST = "FIRST"
    WAIT_ON_PREDECESSOR = "WAIT_ON_PREDECESSOR"
    LOST = "LOST"
    REPLAN = "REPLAN"


@dataclass(frozen=True, slots=True)
class Contender:
    contender_id: str
    sequence: int
    state: ContenderState


@dataclass(frozen=True, slots=True)
class ContenderSnapshot:
    contenders: tuple[Contender, ...]
    revision: int
    completeness: SnapshotCompleteness


@dataclass(frozen=True, slots=True)
class PredecessorDecision:
    outcome: PredecessorOutcome
    predecessor_id: str | None


def _bounded_id(value: object, *, field: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if len(value) > _MAX_ID_BYTES or _ID.fullmatch(value) is None:
        raise ValueError(f"{field} must be a bounded conservative ASCII token")
    return value


def _bounded_integer(value: object, *, field: str) -> int:
    if type(value) is not int:
        raise TypeError(f"{field} must be an exact non-boolean integer")
    if not 0 <= value <= _MAX_INTEGER:
        raise ValueError(f"{field} is outside the supported integer range")
    return value


def _snapshot(value: object) -> ContenderSnapshot:
    if type(value) is not ContenderSnapshot:
        raise TypeError("snapshot must be an exact ContenderSnapshot")
    if type(value.contenders) is not tuple:
        raise TypeError("snapshot.contenders must be an exact tuple")
    if len(value.contenders) > _MAX_CONTENDERS:
        raise ValueError("snapshot exceeds the supported contender count")
    revision = _bounded_integer(value.revision, field="snapshot.revision")
    if type(value.completeness) is not SnapshotCompleteness:
        raise TypeError("snapshot.completeness must be an exact SnapshotCompleteness")

    checked: list[Contender] = []
    seen_ids: set[str] = set()
    seen_sequences: set[int] = set()
    previous_sequence: int | None = None
    total_id_bytes = 0
    for index, raw_contender in enumerate(value.contenders):
        field = f"snapshot.contenders[{index}]"
        if type(raw_contender) is not Contender:
            raise TypeError(f"{field} must be an exact Contender")
        contender_id = _bounded_id(
            raw_contender.contender_id,
            field=f"{field}.contender_id",
        )
        sequence = _bounded_integer(
            raw_contender.sequence,
            field=f"{field}.sequence",
        )
        if type(raw_contender.state) is not ContenderState:
            raise TypeError(f"{field}.state must be an exact ContenderState")
        if contender_id in seen_ids:
            raise ValueError("contender IDs must be unique")
        if sequence in seen_sequences:
            raise ValueError("contender sequences must be unique")
        if previous_sequence is not None and sequence <= previous_sequence:
            raise ValueError("contenders must be in strictly increasing sequence order")

        seen_ids.add(contender_id)
        seen_sequences.add(sequence)
        previous_sequence = sequence
        total_id_bytes += len(contender_id)
        if total_id_bytes > _MAX_TOTAL_ID_BYTES:
            raise ValueError("contender IDs exceed the aggregate byte limit")
        checked.append(Contender(contender_id, sequence, raw_contender.state))

    return ContenderSnapshot(tuple(checked), revision, value.completeness)


def select_immediate_predecessor(
    snapshot: ContenderSnapshot,
    *,
    self_id: str,
    minimum_revision: int,
) -> PredecessorDecision:
    checked_snapshot = _snapshot(snapshot)
    checked_self_id = _bounded_id(self_id, field="self_id")
    checked_minimum = _bounded_integer(
        minimum_revision,
        field="minimum_revision",
    )

    if (
        checked_snapshot.completeness is SnapshotCompleteness.INCOMPLETE
        or checked_snapshot.revision < checked_minimum
    ):
        return PredecessorDecision(PredecessorOutcome.REPLAN, None)

    previous_active: str | None = None
    for contender in checked_snapshot.contenders:
        if contender.contender_id == checked_self_id:
            if contender.state is ContenderState.INACTIVE:
                return PredecessorDecision(PredecessorOutcome.LOST, None)
            if previous_active is None:
                return PredecessorDecision(PredecessorOutcome.FIRST, None)
            return PredecessorDecision(
                PredecessorOutcome.WAIT_ON_PREDECESSOR,
                previous_active,
            )
        if contender.state is ContenderState.ACTIVE:
            previous_active = contender.contender_id

    return PredecessorDecision(PredecessorOutcome.LOST, None)
```

## Example

```python
snapshot = ContenderSnapshot(
    contenders=(
        Contender("session-a", 4, ContenderState.ACTIVE),
        Contender("session-old", 7, ContenderState.INACTIVE),
        Contender("session-b", 9, ContenderState.ACTIVE),
        Contender("session-c", 12, ContenderState.ACTIVE),
    ),
    revision=18,
    completeness=SnapshotCompleteness.COMPLETE,
)

first = select_immediate_predecessor(
    snapshot,
    self_id="session-a",
    minimum_revision=18,
)
wait = select_immediate_predecessor(
    snapshot,
    self_id="session-c",
    minimum_revision=17,
)
inactive = select_immediate_predecessor(
    snapshot,
    self_id="session-old",
    minimum_revision=18,
)
missing = select_immediate_predecessor(
    snapshot,
    self_id="session-missing",
    minimum_revision=18,
)
stale = select_immediate_predecessor(
    snapshot,
    self_id="session-c",
    minimum_revision=19,
)
incomplete = select_immediate_predecessor(
    ContenderSnapshot(
        snapshot.contenders,
        revision=20,
        completeness=SnapshotCompleteness.INCOMPLETE,
    ),
    self_id="session-c",
    minimum_revision=19,
)

assert (first, wait, inactive, missing, stale, incomplete) == (
    PredecessorDecision(PredecessorOutcome.FIRST, None),
    PredecessorDecision(
        PredecessorOutcome.WAIT_ON_PREDECESSOR,
        "session-b",
    ),
    PredecessorDecision(PredecessorOutcome.LOST, None),
    PredecessorDecision(PredecessorOutcome.LOST, None),
    PredecessorDecision(PredecessorOutcome.REPLAN, None),
    PredecessorDecision(PredecessorOutcome.REPLAN, None),
)
```

## Trade-offs and Limitations

Validation and selection take `O(n)` time and `O(n)` additional memory for at
most 256 contenders. IDs are 1 to 64 ASCII bytes with a 16 KiB aggregate
limit. Revisions and sequence positions are exact non-boolean integers in the
nonnegative signed 64-bit range. Duplicate IDs or sequences, noncanonical
order, invalid markers or states, malformed types, and exceeded limits raise
`TypeError` or `ValueError`; `REPLAN` is reserved for a valid observation that
is explicitly incomplete or too old.

Strictly increasing sequence positions define the canonical order; gaps are
allowed and imply neither elapsed time nor fairness. Inactive records are
skipped when choosing the immediate active predecessor, so retaining explicit
activity state matters. The result does not establish fairness, lock
ownership, session validity, or fencing. The helper creates no keys, installs
no watches, waits for nothing, performs no refresh or deletion, and makes no
network request. Before acting, a consumer must reread authoritative state and
externally revalidate its own activity and the selected predecessor.

## Related Snippets

<!-- catalog:related:start -->
- [Report Equal-Share Lease Ownership Excess from a Bounded Queue Snapshot](report-equal-share-lease-ownership-excess-from-a-bounded-queue-snapshot.md)
- [Plan Bounded Worker Replacements from Generation and Restart State](plan-bounded-worker-replacements-from-generation-and-restart-state.md)
- [Decide Whether a Bounded Work Snapshot Permits a New Attempt](../reliability-resilience/decide-whether-a-bounded-work-snapshot-permits-a-new-attempt.md)
<!-- catalog:related:end -->
