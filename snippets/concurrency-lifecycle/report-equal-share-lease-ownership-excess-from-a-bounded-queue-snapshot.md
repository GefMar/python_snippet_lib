---
title: "Report Equal-Share Lease Ownership Excess from a Bounded Queue Snapshot"
snippet_type: algorithm
use_cases:
  - concurrency-control
  - resource-management
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - elect-one-final-releaser-from-bounded-named-leases.md
  - guard-readers-with-a-writer-priority-read-write-lock.md
---

# Report Equal-Share Lease Ownership Excess from a Bounded Queue Snapshot

## Idea and Problem

Report when one participant owns more queues than the ceiling of an equal-share allocation over a validated immutable snapshot.

Every active queue contains the same eligible participant set, although its
ordering may differ. The first participant is the observed owner. The result
records the arithmetic excess and sorted owned queue names without mutating
the snapshot or deciding which lease should be released.

## When to Use

Use this algorithm when up to 64 independent lease queues are observed at one
instant and every participant is eligible for every queue. It is useful as an
advisory input to a separately reviewed rebalance policy when equal ownership
counts are an appropriate target.

Do not apply the calculation when queues have different eligibility sets,
capacities, weights, priorities, or costs. Those constraints require an
allocation model rather than one global ceiling.

## Implementation

```python
import re
from dataclasses import dataclass


_MAX_QUEUES = 64
_MAX_PARTICIPANTS_PER_QUEUE = 32
_MAX_PARTICIPANT_ENTRIES = 1_024
_IDENTIFIER = re.compile(r"[a-z][a-z0-9._:-]{0,63}", re.ASCII)


def _identifier(value: object, *, field: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be exact text")
    if _IDENTIFIER.fullmatch(value) is None:
        raise ValueError(f"{field} must be a canonical bounded ASCII identifier")
    return value


@dataclass(frozen=True, slots=True)
class LeaseQueueSnapshot:
    queue_name: str
    participants: tuple[str, ...]


@dataclass(frozen=True, slots=True)
class LeaseOwnershipExcessReport:
    participant: str
    active_queues: int
    eligible_participants: tuple[str, ...]
    owned_count: int
    equal_share_ceiling: int
    excess: int
    owned_queues: tuple[str, ...]

    @property
    def exceeds_equal_share(self) -> bool:
        return self.excess > 0


def report_equal_share_lease_excess(
    queues: tuple[LeaseQueueSnapshot, ...],
    *,
    participant: str,
) -> LeaseOwnershipExcessReport:
    checked_participant = _identifier(participant, field="participant")
    if type(queues) is not tuple:
        raise TypeError("queues must be an exact tuple")
    if len(queues) > _MAX_QUEUES:
        raise ValueError("queue count exceeds the supported limit")
    if not queues:
        return LeaseOwnershipExcessReport(
            participant=checked_participant,
            active_queues=0,
            eligible_participants=(),
            owned_count=0,
            equal_share_ceiling=0,
            excess=0,
            owned_queues=(),
        )

    queue_names: set[str] = set()
    eligibility: frozenset[str] | None = None
    owned_queue_names: list[str] = []
    participant_entries = 0

    for queue in queues:
        if type(queue) is not LeaseQueueSnapshot:
            raise TypeError("queues must contain exact LeaseQueueSnapshot values")
        queue_name = _identifier(queue.queue_name, field="queue_name")
        if queue_name in queue_names:
            raise ValueError("queue names must be unique")
        queue_names.add(queue_name)

        if type(queue.participants) is not tuple:
            raise TypeError("participants must be an exact tuple")
        if not 1 <= len(queue.participants) <= _MAX_PARTICIPANTS_PER_QUEUE:
            raise ValueError("participant count is outside the per-queue limit")
        participant_entries += len(queue.participants)
        if participant_entries > _MAX_PARTICIPANT_ENTRIES:
            raise ValueError("total participant entries exceed the supported limit")

        queue_participants: set[str] = set()
        for raw_queue_participant in queue.participants:
            queue_participant = _identifier(
                raw_queue_participant,
                field="queue participant",
            )
            if queue_participant in queue_participants:
                raise ValueError("participants must be unique within each queue")
            queue_participants.add(queue_participant)

        queue_eligibility = frozenset(queue_participants)
        if eligibility is None:
            eligibility = queue_eligibility
        elif queue_eligibility != eligibility:
            raise ValueError("all active queues must have the same eligibility set")

        if queue.participants[0] == checked_participant:
            owned_queue_names.append(queue_name)

    assert eligibility is not None
    if checked_participant not in eligibility:
        raise ValueError("participant must be eligible for every active queue")

    active_queues = len(queues)
    eligible_participants = tuple(sorted(eligibility))
    equal_share_ceiling = (
        active_queues + len(eligible_participants) - 1
    ) // len(eligible_participants)
    owned_queues = tuple(sorted(owned_queue_names))
    owned_count = len(owned_queues)
    excess = max(0, owned_count - equal_share_ceiling)
    return LeaseOwnershipExcessReport(
        participant=checked_participant,
        active_queues=active_queues,
        eligible_participants=eligible_participants,
        owned_count=owned_count,
        equal_share_ceiling=equal_share_ceiling,
        excess=excess,
        owned_queues=owned_queues,
    )
```

## Example

```python
queues = (
    LeaseQueueSnapshot("lease-z", ("node-a", "node-b", "node-c")),
    LeaseQueueSnapshot("lease-b", ("node-b", "node-c", "node-a")),
    LeaseQueueSnapshot("lease-a", ("node-a", "node-c", "node-b")),
    LeaseQueueSnapshot("lease-m", ("node-a", "node-b", "node-c")),
)

report = report_equal_share_lease_excess(queues, participant="node-a")
empty = report_equal_share_lease_excess((), participant="node-a")

try:
    report_equal_share_lease_excess(queues, participant="node-z")
except ValueError:
    ineligible_rejected = True
else:
    ineligible_rejected = False

try:
    report_equal_share_lease_excess(
        (
            LeaseQueueSnapshot("first", ("node-a", "node-b")),
            LeaseQueueSnapshot("second", ("node-a", "node-c")),
        ),
        participant="node-a",
    )
except ValueError:
    heterogeneous_rejected = True
else:
    heterogeneous_rejected = False

assert (
    report,
    report.exceeds_equal_share,
    empty,
    empty.exceeds_equal_share,
    ineligible_rejected,
    heterogeneous_rejected,
) == (
    LeaseOwnershipExcessReport(
        participant="node-a",
        active_queues=4,
        eligible_participants=("node-a", "node-b", "node-c"),
        owned_count=3,
        equal_share_ceiling=2,
        excess=1,
        owned_queues=("lease-a", "lease-m", "lease-z"),
    ),
    True,
    LeaseOwnershipExcessReport(
        participant="node-a",
        active_queues=0,
        eligible_participants=(),
        owned_count=0,
        equal_share_ceiling=0,
        excess=0,
        owned_queues=(),
    ),
    False,
    True,
    True,
)
```

## Trade-offs and Limitations

Validation costs `O(e)` time and `O(p + q)` additional memory for bounded
participant entries `e`, eligible participants `p`, and queues `q`. Sorting
the eligible and owned identifiers gives deterministic reports. Requiring the
same eligibility set keeps the ceiling meaningful but excludes heterogeneous
placement constraints.

The report is advisory arithmetic over a possibly stale snapshot. It performs
no acquisition, release, mutation, sleeping, or I/O and provides no guarantee
of fairness, liveness, convergence, or safe stepdown. A caller must revalidate
the authoritative queue state before taking any ownership action.

## Related Snippets

<!-- catalog:related:start -->
- [Elect One Final Releaser from Bounded Named Leases](elect-one-final-releaser-from-bounded-named-leases.md)
- [Guard Readers with a Writer-Priority Read-Write Lock](guard-readers-with-a-writer-priority-read-write-lock.md)
<!-- catalog:related:end -->
