---
title: "Audit Bounded Keyed Half-Open Intervals with Gap and Overlap Evidence"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - join-non-overlapping-half-open-validity-intervals-by-exact-overlap.md
  - measure-time-in-a-state-within-a-half-open-window.md
  - ../algorithms-data-structures/find-peak-coverage-spans-across-bounded-half-open-integer-intervals.md
---

# Audit Bounded Keyed Half-Open Intervals with Gap and Overlap Evidence

## Idea and Problem

Audit complete keyed interval coverage without hiding where data is missing or which original records overlap.

One declared key order and one half-open integer domain define the expected
space. After validating every record, an endpoint sweep emits all uncovered
segments and every overlap segment together with its active input positions.
Positions make duplicate and nested records auditable without choosing a
winner or mutating the input.

## When to Use

Use this audit for a bounded in-memory snapshot whose records should cover one
domain exactly once per declared key. It fits validity histories, ownership
windows, schedules, and partition manifests when both gaps and conflicting
record identities must be reported before downstream use.

Use a peak-coverage calculation when only the maximum count matters, or a
repair-specific policy when gaps should be filled or overlapping values should
be reconciled. This function neither infers keys nor merges, clips, reorders,
or selects records on the caller's behalf.

## Implementation

```python
from dataclasses import dataclass

_MIN_INT64 = -(2**63)
_MAX_INT64 = 2**63 - 1
_MAX_KEYS = 64
_MAX_KEY_BYTES = 128
_MAX_KEY_TEXT_BYTES = 8_192
_MAX_INTERVALS = 4_096
_MAX_EVIDENCE_POSITIONS = 65_536


@dataclass(frozen=True, slots=True)
class KeyedInterval:
    key: str
    start: int
    stop: int


@dataclass(frozen=True, slots=True)
class CoverageGap:
    key: str
    start: int
    stop: int


@dataclass(frozen=True, slots=True)
class CoverageOverlap:
    key: str
    start: int
    stop: int
    positions: tuple[int, ...]


@dataclass(frozen=True, slots=True)
class CoverageAudit:
    gaps: tuple[CoverageGap, ...]
    overlaps: tuple[CoverageOverlap, ...]


def _validated_keys(value: object) -> tuple[str, ...]:
    if type(value) is not tuple:
        raise TypeError("keys must be an exact tuple")
    if not 1 <= len(value) <= _MAX_KEYS:
        raise ValueError("key count is outside 1..64")

    seen: set[str] = set()
    aggregate_bytes = 0
    for index, key in enumerate(value):
        if type(key) is not str:
            raise TypeError(f"keys[{index}] must be an exact string")
        try:
            size = len(key.encode("utf-8"))
        except UnicodeEncodeError:
            raise ValueError(f"keys[{index}] must be UTF-8 encodable") from None
        if not 1 <= size <= _MAX_KEY_BYTES:
            raise ValueError(f"keys[{index}] is outside its UTF-8 byte limit")
        if key in seen:
            raise ValueError("keys must be unique")
        seen.add(key)
        aggregate_bytes += size
    if aggregate_bytes > _MAX_KEY_TEXT_BYTES:
        raise ValueError("keys exceed the aggregate UTF-8 byte limit")
    return value


def _validated_endpoint(value: object, *, field: str) -> int:
    if type(value) is not int:
        raise TypeError(f"{field} must be an exact integer")
    if not _MIN_INT64 <= value <= _MAX_INT64:
        raise ValueError(f"{field} is outside the signed 64-bit range")
    return value


def audit_interval_coverage(
    keys: tuple[str, ...],
    domain_start: int,
    domain_stop: int,
    intervals: tuple[KeyedInterval, ...],
) -> CoverageAudit:
    checked_keys = _validated_keys(keys)
    checked_start = _validated_endpoint(domain_start, field="domain_start")
    checked_stop = _validated_endpoint(domain_stop, field="domain_stop")
    if checked_start >= checked_stop:
        raise ValueError("the domain must have start < stop")

    if type(intervals) is not tuple:
        raise TypeError("intervals must be an exact tuple")
    if len(intervals) > _MAX_INTERVALS:
        raise ValueError("interval count exceeds 4096")

    key_indexes = {key: index for index, key in enumerate(checked_keys)}
    by_key: list[list[tuple[int, int, int]]] = [
        [] for _ in checked_keys
    ]
    for position, interval in enumerate(intervals):
        field = f"intervals[{position}]"
        if type(interval) is not KeyedInterval:
            raise TypeError(f"{field} must be an exact KeyedInterval")
        if type(interval.key) is not str:
            raise TypeError(f"{field}.key must be an exact string")
        if interval.key not in key_indexes:
            raise ValueError(f"{field}.key is outside the declared key domain")
        start = _validated_endpoint(interval.start, field=f"{field}.start")
        stop = _validated_endpoint(interval.stop, field=f"{field}.stop")
        if not checked_start <= start < stop <= checked_stop:
            raise ValueError(f"{field} is outside the declared half-open domain")
        by_key[key_indexes[interval.key]].append((start, stop, position))

    gaps: list[CoverageGap] = []
    overlaps: list[CoverageOverlap] = []
    evidence_positions = 0
    for key, key_intervals in zip(checked_keys, by_key, strict=True):
        starts: dict[int, list[int]] = {}
        stops: dict[int, list[int]] = {}
        boundaries = {checked_start, checked_stop}
        for start, stop, position in key_intervals:
            starts.setdefault(start, []).append(position)
            stops.setdefault(stop, []).append(position)
            boundaries.add(start)
            boundaries.add(stop)

        ordered_boundaries = sorted(boundaries)
        active: set[int] = set()
        for boundary_index, left in enumerate(ordered_boundaries[:-1]):
            for position in stops.get(left, ()):
                active.remove(position)
            for position in starts.get(left, ()):
                active.add(position)
            right = ordered_boundaries[boundary_index + 1]

            if not active:
                if gaps and gaps[-1].key == key and gaps[-1].stop == left:
                    gaps[-1] = CoverageGap(key, gaps[-1].start, right)
                else:
                    gaps.append(CoverageGap(key, left, right))
            elif len(active) > 1:
                positions = tuple(sorted(active))
                evidence_positions += len(positions)
                if evidence_positions > _MAX_EVIDENCE_POSITIONS:
                    raise ValueError("overlap evidence exceeds the position limit")
                if (
                    overlaps
                    and overlaps[-1].key == key
                    and overlaps[-1].stop == left
                    and overlaps[-1].positions == positions
                ):
                    overlaps[-1] = CoverageOverlap(
                        key,
                        overlaps[-1].start,
                        right,
                        positions,
                    )
                else:
                    overlaps.append(CoverageOverlap(key, left, right, positions))

    return CoverageAudit(tuple(gaps), tuple(overlaps))
```

## Example

```python
def unit_cell_reference(
    keys: tuple[str, ...],
    domain_start: int,
    domain_stop: int,
    intervals: tuple[KeyedInterval, ...],
) -> CoverageAudit:
    gaps: list[CoverageGap] = []
    overlaps: list[CoverageOverlap] = []
    for key in keys:
        for left in range(domain_start, domain_stop):
            positions = tuple(
                position
                for position, interval in enumerate(intervals)
                if interval.key == key and interval.start <= left < interval.stop
            )
            if not positions:
                if gaps and gaps[-1].key == key and gaps[-1].stop == left:
                    gaps[-1] = CoverageGap(key, gaps[-1].start, left + 1)
                else:
                    gaps.append(CoverageGap(key, left, left + 1))
            elif len(positions) > 1:
                if (
                    overlaps
                    and overlaps[-1].key == key
                    and overlaps[-1].stop == left
                    and overlaps[-1].positions == positions
                ):
                    overlaps[-1] = CoverageOverlap(
                        key,
                        overlaps[-1].start,
                        left + 1,
                        positions,
                    )
                else:
                    overlaps.append(
                        CoverageOverlap(key, left, left + 1, positions)
                    )
    return CoverageAudit(tuple(gaps), tuple(overlaps))


def check_tiny_interval_sets() -> int:
    possible = tuple(
        (start, stop)
        for start in range(4)
        for stop in range(start + 1, 5)
    )
    checked = 0
    for subset_mask in range(1 << len(possible)):
        selected = tuple(
            KeyedInterval("k", start, stop)
            for bit, (start, stop) in enumerate(possible)
            if subset_mask & (1 << bit)
        )
        for declaration in (selected, tuple(reversed(selected))):
            assert audit_interval_coverage(("k",), 0, 4, declaration) == (
                unit_cell_reference(("k",), 0, 4, declaration)
            )
            checked += 1
    return checked


sample_intervals = (
    KeyedInterval("alpha", 0, 2),
    KeyedInterval("alpha", 1, 3),
    KeyedInterval("alpha", 4, 6),
)
sample = audit_interval_coverage(("alpha", "beta"), 0, 6, sample_intervals)
touching = audit_interval_coverage(
    ("k",),
    0,
    4,
    (KeyedInterval("k", 0, 2), KeyedInterval("k", 2, 4)),
)
evidence_heavy = tuple(
    KeyedInterval("k", index, 1_024 - index)
    for index in range(512)
)
try:
    audit_interval_coverage(("k",), 0, 1_024, evidence_heavy)
except ValueError:
    evidence_cap_enforced = True
else:
    evidence_cap_enforced = False

assert (
    check_tiny_interval_sets() == 2_048
    and sample
    == CoverageAudit(
        gaps=(CoverageGap("alpha", 3, 4), CoverageGap("beta", 0, 6)),
        overlaps=(CoverageOverlap("alpha", 1, 2, (0, 1)),),
    )
    and touching == CoverageAudit((), ())
    and evidence_cap_enforced
)
```

## Trade-offs and Limitations

For `R` records, endpoint sorting takes `O(R log R)` time. Materializing each
sorted active-position tuple adds up to `O(E log R)` work for `E` stored
position references. Input, event state, and output use `O(R + E)` memory, and
the explicit evidence cap prevents deeply nested spans from producing an
unbounded quadratic report.

Evidence positions intentionally depend on input order even though gap and
overlap geometry does not. Half-open touching boundaries are not overlaps.
The audit reports structural coverage only; it does not compare record values,
infer whether a gap is acceptable, or make the result safe against changes
that occur after the supplied snapshot.

## Related Snippets

<!-- catalog:related:start -->
- [Join Non-Overlapping Half-Open Validity Intervals by Exact Overlap](join-non-overlapping-half-open-validity-intervals-by-exact-overlap.md)
- [Measure Time in a State Within a Half-Open Window](measure-time-in-a-state-within-a-half-open-window.md)
- [Find Peak Coverage Spans Across Bounded Half-Open Integer Intervals](../algorithms-data-structures/find-peak-coverage-spans-across-bounded-half-open-integer-intervals.md)
<!-- catalog:related:end -->
