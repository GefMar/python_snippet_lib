---
title: "Classify an Owner-Scoped Heartbeat Lease Snapshot"
snippet_type: algorithm
use_cases:
  - concurrency-control
  - lifecycle-management
  - retry-recovery
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - decide-whether-a-bounded-work-snapshot-permits-a-new-attempt.md
  - ../concurrency-lifecycle/select-a-reusable-lease-for-an-explicit-remaining-horizon.md
  - ../concurrency-lifecycle/report-equal-share-lease-ownership-excess-from-a-bounded-queue-snapshot.md
---

# Classify an Owner-Scoped Heartbeat Lease Snapshot

## Idea and Problem

Classify one owner-scoped request against a bounded heartbeat lease snapshot without authorizing any action.

A snapshot is exactly `RELEASED`, with neither owner nor heartbeat, or `HELD`,
with both a bounded owner and heartbeat. `CLAIM`, `RENEW`, `RELEASE`, and
`RECLAIM_CHECK` produce only a compatibility outcome, a reason, and the
caller's unchanged opaque correlation ID. No successor state is calculated.

## When to Use

Use this algorithm after reading one coherent lease observation when request
handling needs a small, deterministic first-pass classification. The caller
supplies the observation time and positive stale interval as bounded integer
time values from the same time domain.

`COMPATIBLE` means compatible with this snapshot only. Every real claim,
renewal, or release still requires one atomic external operation that
revalidates authoritative state. A stale heartbeat is evidence for further
coordination, never authority to reclaim: `EXTERNAL_FENCE_REQUIRED` says that
an external fencing protocol must settle the race before any action.

## Implementation

```python
import re
from dataclasses import dataclass
from enum import StrEnum

_MAX_TIME = 2**63 - 1
_MAX_OWNER_BYTES = 64
_MAX_CORRELATION_BYTES = 128
_OWNER = re.compile(r"[A-Za-z0-9][A-Za-z0-9._:-]{0,63}", re.ASCII)


class LeaseState(StrEnum):
    RELEASED = "RELEASED"
    HELD = "HELD"


class RequestKind(StrEnum):
    CLAIM = "CLAIM"
    RENEW = "RENEW"
    RELEASE = "RELEASE"
    RECLAIM_CHECK = "RECLAIM_CHECK"


class Compatibility(StrEnum):
    COMPATIBLE = "COMPATIBLE"
    INCOMPATIBLE = "INCOMPATIBLE"
    EXTERNAL_FENCE_REQUIRED = "EXTERNAL_FENCE_REQUIRED"


class Reason(StrEnum):
    RELEASED = "RELEASED"
    ALREADY_HELD = "ALREADY_HELD"
    OWNER_MATCH = "OWNER_MATCH"
    OWNER_MISMATCH = "OWNER_MISMATCH"
    NOT_HELD = "NOT_HELD"
    NOT_STALE = "NOT_STALE"
    STALE_HEARTBEAT = "STALE_HEARTBEAT"


@dataclass(frozen=True, slots=True)
class HeartbeatLeaseSnapshot:
    state: LeaseState
    owner: str | None
    heartbeat_at: int | None
    observed_at: int


@dataclass(frozen=True, slots=True)
class LeaseClassification:
    outcome: Compatibility
    reason: Reason
    correlation_id: str


def _bounded_time(value: object, *, field: str, positive: bool = False) -> int:
    if type(value) is not int:
        raise TypeError(f"{field} must be an exact non-boolean integer")
    lower = 1 if positive else 0
    if not lower <= value <= _MAX_TIME:
        raise ValueError(f"{field} is outside the supported time range")
    return value


def _owner(value: object, *, field: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if len(value) > _MAX_OWNER_BYTES or _OWNER.fullmatch(value) is None:
        raise ValueError(f"{field} must be a bounded conservative ASCII token")
    return value


def _correlation_id(value: object) -> str:
    if type(value) is not str:
        raise TypeError("correlation_id must be an exact string")
    if not 1 <= len(value) <= _MAX_CORRELATION_BYTES:
        raise ValueError("correlation_id is outside the supported byte range")
    try:
        size = len(value.encode("utf-8"))
    except UnicodeEncodeError as exc:
        raise ValueError("correlation_id must be valid UTF-8 text") from exc
    if not 1 <= size <= _MAX_CORRELATION_BYTES:
        raise ValueError("correlation_id is outside the supported byte range")
    return value


def _snapshot(value: object) -> HeartbeatLeaseSnapshot:
    if type(value) is not HeartbeatLeaseSnapshot:
        raise TypeError("snapshot must be an exact HeartbeatLeaseSnapshot")
    if type(value.state) is not LeaseState:
        raise TypeError("snapshot.state must be an exact LeaseState")

    observed_at = _bounded_time(value.observed_at, field="snapshot.observed_at")
    if value.state is LeaseState.RELEASED:
        if value.owner is not None or value.heartbeat_at is not None:
            raise ValueError("a RELEASED snapshot has no owner or heartbeat")
        return HeartbeatLeaseSnapshot(
            LeaseState.RELEASED,
            None,
            None,
            observed_at,
        )

    owner = _owner(value.owner, field="snapshot.owner")
    heartbeat_at = _bounded_time(
        value.heartbeat_at,
        field="snapshot.heartbeat_at",
    )
    if heartbeat_at > observed_at:
        raise ValueError("snapshot.heartbeat_at must not exceed observed_at")
    return HeartbeatLeaseSnapshot(
        LeaseState.HELD,
        owner,
        heartbeat_at,
        observed_at,
    )


def classify_heartbeat_lease_request(
    snapshot: HeartbeatLeaseSnapshot,
    *,
    request: RequestKind,
    requester: str,
    stale_after: int,
    correlation_id: str,
) -> LeaseClassification:
    checked_snapshot = _snapshot(snapshot)
    if type(request) is not RequestKind:
        raise TypeError("request must be an exact RequestKind")
    checked_requester = _owner(requester, field="requester")
    checked_stale_after = _bounded_time(
        stale_after,
        field="stale_after",
        positive=True,
    )
    checked_correlation_id = _correlation_id(correlation_id)

    def result(
        outcome: Compatibility,
        reason: Reason,
    ) -> LeaseClassification:
        return LeaseClassification(outcome, reason, checked_correlation_id)

    if request is RequestKind.CLAIM:
        if checked_snapshot.state is LeaseState.RELEASED:
            return result(Compatibility.COMPATIBLE, Reason.RELEASED)
        return result(Compatibility.INCOMPATIBLE, Reason.ALREADY_HELD)

    if request is RequestKind.RECLAIM_CHECK:
        if checked_snapshot.state is LeaseState.RELEASED:
            return result(Compatibility.INCOMPATIBLE, Reason.NOT_HELD)
        assert checked_snapshot.heartbeat_at is not None
        age = checked_snapshot.observed_at - checked_snapshot.heartbeat_at
        if age < checked_stale_after:
            return result(Compatibility.INCOMPATIBLE, Reason.NOT_STALE)
        return result(
            Compatibility.EXTERNAL_FENCE_REQUIRED,
            Reason.STALE_HEARTBEAT,
        )

    if checked_snapshot.state is LeaseState.RELEASED:
        return result(Compatibility.INCOMPATIBLE, Reason.NOT_HELD)
    if checked_snapshot.owner != checked_requester:
        return result(Compatibility.INCOMPATIBLE, Reason.OWNER_MISMATCH)
    return result(Compatibility.COMPATIBLE, Reason.OWNER_MATCH)
```

## Example

```python
released = HeartbeatLeaseSnapshot(LeaseState.RELEASED, None, None, 100)
fresh = HeartbeatLeaseSnapshot(LeaseState.HELD, "worker-a", 91, 100)
stale_at_boundary = HeartbeatLeaseSnapshot(
    LeaseState.HELD,
    "worker-a",
    90,
    100,
)
correlation = "trace/alpha-7"

claim_released = classify_heartbeat_lease_request(
    released,
    request=RequestKind.CLAIM,
    requester="worker-a",
    stale_after=10,
    correlation_id=correlation,
)
claim_same_owner = classify_heartbeat_lease_request(
    fresh,
    request=RequestKind.CLAIM,
    requester="worker-a",
    stale_after=10,
    correlation_id=correlation,
)
renew_same_owner = classify_heartbeat_lease_request(
    fresh,
    request=RequestKind.RENEW,
    requester="worker-a",
    stale_after=10,
    correlation_id=correlation,
)
release_other_owner = classify_heartbeat_lease_request(
    fresh,
    request=RequestKind.RELEASE,
    requester="worker-b",
    stale_after=10,
    correlation_id=correlation,
)
reclaim_released = classify_heartbeat_lease_request(
    released,
    request=RequestKind.RECLAIM_CHECK,
    requester="worker-b",
    stale_after=10,
    correlation_id=correlation,
)
reclaim_fresh = classify_heartbeat_lease_request(
    fresh,
    request=RequestKind.RECLAIM_CHECK,
    requester="worker-b",
    stale_after=10,
    correlation_id=correlation,
)
reclaim_stale = classify_heartbeat_lease_request(
    stale_at_boundary,
    request=RequestKind.RECLAIM_CHECK,
    requester="worker-b",
    stale_after=10,
    correlation_id=correlation,
)

assert (
    (claim_released.outcome, claim_released.reason),
    (claim_same_owner.outcome, claim_same_owner.reason),
    (renew_same_owner.outcome, renew_same_owner.reason),
    (release_other_owner.outcome, release_other_owner.reason),
    (reclaim_released.outcome, reclaim_released.reason),
    (reclaim_fresh.outcome, reclaim_fresh.reason),
    (reclaim_stale.outcome, reclaim_stale.reason),
) == (
    (Compatibility.COMPATIBLE, Reason.RELEASED),
    (Compatibility.INCOMPATIBLE, Reason.ALREADY_HELD),
    (Compatibility.COMPATIBLE, Reason.OWNER_MATCH),
    (Compatibility.INCOMPATIBLE, Reason.OWNER_MISMATCH),
    (Compatibility.INCOMPATIBLE, Reason.NOT_HELD),
    (Compatibility.INCOMPATIBLE, Reason.NOT_STALE),
    (Compatibility.EXTERNAL_FENCE_REQUIRED, Reason.STALE_HEARTBEAT),
)
assert all(
    decision.correlation_id is correlation
    for decision in (
        claim_released,
        claim_same_owner,
        renew_same_owner,
        release_other_owner,
        reclaim_released,
        reclaim_fresh,
        reclaim_stale,
    )
)
```

## Trade-offs and Limitations

Classification uses constant time and bounded immutable output. Times and the
positive stale interval are exact non-boolean integers in the nonnegative
signed 64-bit range. Owners are conservative ASCII tokens of at most 64 bytes;
the opaque correlation ID is 1 to 128 UTF-8 bytes and is neither normalized nor
interpreted. Malformed types raise `TypeError`; invalid states, pairings,
ranges, tokens, or future heartbeats raise `ValueError`.

The inclusive stale boundary is `observed_at - heartbeat_at >= stale_after`.
Clock-domain agreement and observation coherence remain caller obligations.
The correlation ID is echoed unchanged in every valid classification; it is
not a revision, compare-and-swap value, fence, or replay guarantee. The
classifier performs no mutation or I/O and returns no permission, ownership assertion,
successor state, revision, compare-and-swap instruction, or fence. In
particular, heartbeat age alone can never make reclamation safe.

## Related Snippets

<!-- catalog:related:start -->
- [Decide Whether a Bounded Work Snapshot Permits a New Attempt](decide-whether-a-bounded-work-snapshot-permits-a-new-attempt.md)
- [Select a Reusable Lease for an Explicit Remaining Horizon](../concurrency-lifecycle/select-a-reusable-lease-for-an-explicit-remaining-horizon.md)
- [Report Equal-Share Lease Ownership Excess from a Bounded Queue Snapshot](../concurrency-lifecycle/report-equal-share-lease-ownership-excess-from-a-bounded-queue-snapshot.md)
<!-- catalog:related:end -->
