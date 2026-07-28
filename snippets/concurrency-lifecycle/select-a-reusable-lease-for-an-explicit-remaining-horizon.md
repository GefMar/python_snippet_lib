---
title: "Select a Reusable Lease for an Explicit Remaining Horizon"
snippet_type: algorithm
use_cases:
  - concurrency-control
  - lifecycle-management
  - resource-management
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - report-equal-share-lease-ownership-excess-from-a-bounded-queue-snapshot.md
  - ../algorithms-data-structures/choose-the-first-eligible-candidate-from-ordered-priority-groups.md
  - ../data-processing/select-one-record-per-key-with-an-explicit-ranking-rule.md
---

# Select a Reusable Lease for an Explicit Remaining Horizon

## Idea and Problem

Select one lease snapshot whose exact capabilities and inclusive remaining lifetime satisfy a caller-supplied horizon.

Every input is validated before eligibility is evaluated. A lease is eligible
only when it has started by `observed_at`, its capability set exactly equals
the required set, and its expiry is at or after
`observed_at + required_duration`. The newest start wins, with the
lexicographically smallest lease ID breaking equal-start ties.

## When to Use

Use this pure selection step when one component already holds a complete,
immutable snapshot of at most 256 neutral leases and must make a deterministic
advisory choice among leases that have already started. Supply the exact `UTC`
observation value used to build the decision and an exact positive `timedelta`
no greater than 366 days.

The snapshot can become stale immediately. Before use, another component must
atomically revalidate the authoritative lease identity, capability set, and
remaining horizon while claiming it. Selection alone never grants ownership.

## Implementation

```python
import re
from dataclasses import dataclass
from datetime import UTC, datetime, timedelta

_MAX_LEASES = 256
_MAX_CAPABILITIES_PER_SET = 32
_MAX_LEASE_ID_BYTES = 64
_MAX_CAPABILITY_BYTES = 48
_MAX_TOTAL_TEXT_BYTES = 128 * 1_024
_MAX_REQUIRED_DURATION = timedelta(days=366)
_TOKEN = re.compile(r"[a-z][a-z0-9._:-]*", re.ASCII)


class NoReusableLeaseError(LookupError):
    pass


@dataclass(frozen=True, slots=True)
class LeaseSnapshot:
    lease_id: str
    capabilities: frozenset[str]
    started_at: datetime
    expires_at: datetime


@dataclass(frozen=True, slots=True)
class ReusableLeaseChoice:
    lease_id: str
    capabilities: tuple[str, ...]
    started_at: datetime
    expires_at: datetime
    required_until: datetime


def _validated_token(
    value: object,
    *,
    field: str,
    byte_limit: int,
) -> tuple[str, int]:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if not 1 <= len(value) <= byte_limit:
        raise ValueError(f"{field} length is outside the supported range")
    try:
        encoded = value.encode("utf-8")
    except UnicodeEncodeError as error:
        raise ValueError(f"{field} must be valid UTF-8 text") from error
    if len(encoded) > byte_limit:
        raise ValueError(f"{field} exceeds its UTF-8 byte limit")
    if _TOKEN.fullmatch(value) is None:
        raise ValueError(f"{field} must be a conservative ASCII identifier")
    return value, len(encoded)


def _validated_utc(value: object, *, field: str) -> datetime:
    if type(value) is not datetime:
        raise TypeError(f"{field} must be an exact datetime")
    if value.tzinfo is not UTC:
        raise ValueError(f"{field} must use the exact UTC timezone")
    return value


def select_reusable_lease(
    snapshots: tuple[LeaseSnapshot, ...],
    *,
    required_capabilities: frozenset[str],
    observed_at: datetime,
    required_duration: timedelta,
) -> ReusableLeaseChoice:
    if type(snapshots) is not tuple:
        raise TypeError("snapshots must be an exact tuple")
    if len(snapshots) > _MAX_LEASES:
        raise ValueError("snapshot count exceeds the supported limit")
    if type(required_capabilities) is not frozenset:
        raise TypeError("required_capabilities must be an exact frozenset")
    if len(required_capabilities) > _MAX_CAPABILITIES_PER_SET:
        raise ValueError("required capability count exceeds the supported limit")

    observation = _validated_utc(observed_at, field="observed_at")
    if type(required_duration) is not timedelta:
        raise TypeError("required_duration must be an exact timedelta")
    if not timedelta(0) < required_duration <= _MAX_REQUIRED_DURATION:
        raise ValueError("required_duration is outside the supported range")
    try:
        required_until = observation + required_duration
    except OverflowError as error:
        raise ValueError("the required horizon is not representable") from error

    total_text_bytes = 0
    checked_required: set[str] = set()
    for raw_capability in required_capabilities:
        capability, size = _validated_token(
            raw_capability,
            field="required capability",
            byte_limit=_MAX_CAPABILITY_BYTES,
        )
        checked_required.add(capability)
        total_text_bytes += size

    known_lease_ids: set[str] = set()
    for position, snapshot in enumerate(snapshots):
        field = f"snapshots[{position}]"
        if type(snapshot) is not LeaseSnapshot:
            raise TypeError(f"{field} must be an exact LeaseSnapshot")
        lease_id, size = _validated_token(
            snapshot.lease_id,
            field=f"{field}.lease_id",
            byte_limit=_MAX_LEASE_ID_BYTES,
        )
        total_text_bytes += size
        if lease_id in known_lease_ids:
            raise ValueError("lease IDs must be unique")
        known_lease_ids.add(lease_id)

        if type(snapshot.capabilities) is not frozenset:
            raise TypeError(f"{field}.capabilities must be an exact frozenset")
        if len(snapshot.capabilities) > _MAX_CAPABILITIES_PER_SET:
            raise ValueError("a capability set exceeds the supported count")
        for raw_capability in snapshot.capabilities:
            _, size = _validated_token(
                raw_capability,
                field=f"{field} capability",
                byte_limit=_MAX_CAPABILITY_BYTES,
            )
            total_text_bytes += size

        started_at = _validated_utc(
            snapshot.started_at,
            field=f"{field}.started_at",
        )
        expires_at = _validated_utc(
            snapshot.expires_at,
            field=f"{field}.expires_at",
        )
        if started_at > expires_at:
            raise ValueError("a lease start must not be after its expiry")
        if total_text_bytes > _MAX_TOTAL_TEXT_BYTES:
            raise ValueError("snapshot text exceeds the aggregate UTF-8 byte limit")

    required_set = frozenset(checked_required)
    selected: LeaseSnapshot | None = None
    for snapshot in snapshots:
        if (
            snapshot.started_at > observation
            or snapshot.capabilities != required_set
            or snapshot.expires_at < required_until
        ):
            continue
        if (
            selected is None
            or snapshot.started_at > selected.started_at
            or (
                snapshot.started_at == selected.started_at and snapshot.lease_id < selected.lease_id
            )
        ):
            selected = snapshot

    if selected is None:
        raise NoReusableLeaseError("no lease satisfies the required horizon")
    return ReusableLeaseChoice(
        lease_id=selected.lease_id,
        capabilities=tuple(sorted(selected.capabilities)),
        started_at=selected.started_at,
        expires_at=selected.expires_at,
        required_until=required_until,
    )
```

## Example

```python
observed_at = datetime(2032, 6, 1, 10, 0, tzinfo=UTC)
required = frozenset({"read", "stable"})
snapshots = (
    LeaseSnapshot(
        "older",
        required,
        datetime(2032, 6, 1, 9, 30, tzinfo=UTC),
        datetime(2032, 6, 1, 12, 0, tzinfo=UTC),
    ),
    LeaseSnapshot(
        "zeta",
        required,
        datetime(2032, 6, 1, 9, 50, tzinfo=UTC),
        datetime(2032, 6, 1, 11, 0, tzinfo=UTC),
    ),
    LeaseSnapshot(
        "alpha",
        required,
        datetime(2032, 6, 1, 9, 50, tzinfo=UTC),
        datetime(2032, 6, 1, 10, 30, tzinfo=UTC),
    ),
    LeaseSnapshot(
        "newer-extra",
        frozenset({"read", "stable", "write"}),
        datetime(2032, 6, 1, 9, 55, tzinfo=UTC),
        datetime(2032, 6, 1, 12, 0, tzinfo=UTC),
    ),
    LeaseSnapshot(
        "future",
        required,
        datetime(2032, 6, 1, 10, 5, tzinfo=UTC),
        datetime(2032, 6, 1, 12, 0, tzinfo=UTC),
    ),
)

choice = select_reusable_lease(
    snapshots,
    required_capabilities=required,
    observed_at=observed_at,
    required_duration=timedelta(minutes=30),
)

try:
    select_reusable_lease(
        snapshots,
        required_capabilities=frozenset({"archive"}),
        observed_at=observed_at,
        required_duration=timedelta(minutes=5),
    )
except NoReusableLeaseError:
    no_match_is_explicit = True
else:
    no_match_is_explicit = False

try:
    choice.lease_id = "changed"
except AttributeError:
    output_is_frozen = True
else:
    output_is_frozen = False

assert (
    choice,
    no_match_is_explicit,
    output_is_frozen,
    choice.capabilities is not snapshots[2].capabilities,
) == (
    ReusableLeaseChoice(
        lease_id="alpha",
        capabilities=("read", "stable"),
        started_at=datetime(2032, 6, 1, 9, 50, tzinfo=UTC),
        expires_at=datetime(2032, 6, 1, 10, 30, tzinfo=UTC),
        required_until=datetime(2032, 6, 1, 10, 30, tzinfo=UTC),
    ),
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

Validation and selection take `O(nc)` time for `n` bounded snapshots and at
most `c = 32` capabilities per snapshot. Unique-ID tracking, the validated
required set, and the frozen output use `O(n + c)` auxiliary space. Per-token,
aggregate-text, duration, capability-count, and snapshot-count ceilings keep
work and retained validation state finite.

Equality at either the observation start or required horizon is deliberately
eligible. A lease starting after the observation instant is not yet reusable.
Capability matching is exact, so a lease with one additional capability is not
a match.
The output is a new frozen value with a newly materialized capability tuple;
it retains no snapshot object or mutable input alias. Identifiers use Python's
lexicographic string order after conservative ASCII validation.

The choice is advisory over stale snapshots. An authoritative component must
atomically revalidate and claim the same lease before use. This function owns
no clocks, caches, credentials, provider calls, acquisition, creation, renewal,
release, or cleanup behavior, and it performs no mutation or I/O.

## Related Snippets

<!-- catalog:related:start -->
- [Report Equal-Share Lease Ownership Excess from a Bounded Queue Snapshot](report-equal-share-lease-ownership-excess-from-a-bounded-queue-snapshot.md)
- [Choose the First Eligible Candidate from Ordered Priority Groups](../algorithms-data-structures/choose-the-first-eligible-candidate-from-ordered-priority-groups.md)
- [Select One Record per Key with an Explicit Ranking Rule](../data-processing/select-one-record-per-key-with-an-explicit-ranking-rule.md)
<!-- catalog:related:end -->
