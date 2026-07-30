---
title: "Build Bounded Per-Key Validity Histories from Versioned Change Records"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - join-non-overlapping-half-open-validity-intervals-by-exact-overlap.md
  - join-bounded-strictly-increasing-sequences-by-the-latest-prior-timestamp.md
  - compare-ordered-integer-time-series-snapshots-with-explicit-tolerance.md
---

# Build Bounded Per-Key Validity Histories from Versioned Change Records

## Idea and Problem

Turn an unordered bounded set of logical-version changes into canonical non-overlapping validity intervals for each key.

A value begins at its record's version and remains effective until the next
record for that key or an explicit common horizon. Sorting removes input-order
dependence. Consecutive records carrying the same exact scalar state are
coalesced, so every emitted boundary represents an observable change rather
than merely another record.

## When to Use

Use this transformation when the complete change set for a bounded logical
version range is already in memory and downstream code needs canonical
half-open `[start, end)` intervals. Versions are coordinates on a caller-owned
logical axis, not timestamps; gaps simply mean that the previous value keeps
applying.

Use a database or stream processor when records can arrive late, histories are
incomplete, writers race, or corrections need both transaction time and valid
time. This closed model has no tombstone: `None` is an ordinary value. Model deletion
as an explicit domain state before using the function if deletion semantics
are required.

## Implementation

```python
from dataclasses import dataclass

_MAX_INT64 = 2**63 - 1
_MAX_RECORDS = 4_096
_MAX_KEYS = 256
_MAX_KEY_BYTES = 128
_MAX_STRING_VALUE_BYTES = 1_024
_MAX_AGGREGATE_TEXT_BYTES = 1_048_576


@dataclass(frozen=True, slots=True)
class VersionedChange:
    key: str
    version: int
    value: object


@dataclass(frozen=True, slots=True)
class ValueValidity:
    key: str
    start: int
    end: int
    value: object


def _utf8_size(value: str, *, field: str, maximum: int) -> int:
    try:
        size = len(value.encode("utf-8"))
    except UnicodeEncodeError:
        raise ValueError(f"{field} must be UTF-8 encodable") from None
    if not 1 <= size <= maximum:
        raise ValueError(f"{field} is outside its UTF-8 byte limit")
    return size


def _validated_scalar(value: object, *, field: str) -> int:
    if value is None or type(value) is bool:
        return 0
    if type(value) is int:
        if not -(2**63) <= value <= _MAX_INT64:
            raise ValueError(f"{field} integer is outside the signed 64-bit range")
        return 0
    if type(value) is str:
        try:
            size = len(value.encode("utf-8"))
        except UnicodeEncodeError:
            raise ValueError(f"{field} must be UTF-8 encodable") from None
        if size > _MAX_STRING_VALUE_BYTES:
            raise ValueError(f"{field} string exceeds its UTF-8 byte limit")
        return size
    raise TypeError(f"{field} must be exactly None, bool, int, or str")


def _same_scalar(first: object, second: object) -> bool:
    return type(first) is type(second) and first == second


def build_validity_histories(
    records: tuple[VersionedChange, ...],
    *,
    horizon: int,
) -> tuple[ValueValidity, ...]:
    if type(records) is not tuple:
        raise TypeError("records must be an exact tuple")
    if len(records) > _MAX_RECORDS:
        raise ValueError("record count exceeds the supported limit")
    if type(horizon) is not int:
        raise TypeError("horizon must be an exact integer")
    if not 1 <= horizon <= _MAX_INT64:
        raise ValueError("horizon is outside 1..2**63-1")

    grouped: dict[str, list[tuple[int, object]]] = {}
    coordinates: set[tuple[str, int]] = set()
    aggregate_text_bytes = 0

    for index, record in enumerate(records):
        field = f"records[{index}]"
        if type(record) is not VersionedChange:
            raise TypeError(f"{field} must be an exact VersionedChange")
        if type(record.key) is not str:
            raise TypeError(f"{field}.key must be an exact string")
        aggregate_text_bytes += _utf8_size(
            record.key,
            field=f"{field}.key",
            maximum=_MAX_KEY_BYTES,
        )
        if type(record.version) is not int:
            raise TypeError(f"{field}.version must be an exact integer")
        if not 0 <= record.version < horizon:
            raise ValueError(f"{field}.version is outside 0..horizon-1")
        aggregate_text_bytes += _validated_scalar(
            record.value,
            field=f"{field}.value",
        )
        if aggregate_text_bytes > _MAX_AGGREGATE_TEXT_BYTES:
            raise ValueError("record text exceeds the aggregate UTF-8 byte limit")

        coordinate = (record.key, record.version)
        if coordinate in coordinates:
            raise ValueError("records contain a duplicate key/version coordinate")
        coordinates.add(coordinate)
        grouped.setdefault(record.key, []).append((record.version, record.value))
        if len(grouped) > _MAX_KEYS:
            raise ValueError("record set exceeds the distinct-key limit")

    intervals: list[ValueValidity] = []
    for key in sorted(grouped):
        changes = sorted(grouped[key], key=lambda item: item[0])
        current_start, current_value = changes[0]
        for version, value in changes[1:]:
            if _same_scalar(value, current_value):
                continue
            intervals.append(
                ValueValidity(key, current_start, version, current_value)
            )
            current_start = version
            current_value = value
        intervals.append(ValueValidity(key, current_start, horizon, current_value))

    return tuple(intervals)
```

## Example

```python
def state_at(
    intervals: tuple[ValueValidity, ...],
    key: str,
    version: int,
) -> tuple[bool, object]:
    for interval in intervals:
        if interval.key == key and interval.start <= version < interval.end:
            return True, interval.value
    return False, None


def exercise_small_histories() -> int:
    from itertools import permutations, product

    values = (None, False, 0, "x")
    checked = 0
    for states in product(values, repeat=4):
        canonical_records = tuple(
            VersionedChange("key", version, value)
            for version, value in enumerate(states)
        )
        expected = build_validity_histories(canonical_records, horizon=4)
        for declaration_order in permutations(canonical_records):
            observed = build_validity_histories(declaration_order, horizon=4)
            assert observed == expected
            for version, value in enumerate(states):
                assert state_at(observed, "key", version) == (True, value)
            checked += 1
    return checked


checked = exercise_small_histories()

sample = build_validity_histories(
    (
        VersionedChange("beta", 4, "warm"),
        VersionedChange("alpha", 3, 1),
        VersionedChange("alpha", 0, True),
        VersionedChange("beta", 1, "warm"),
        VersionedChange("alpha", 5, 1),
    ),
    horizon=8,
)

try:
    build_validity_histories(
        (
            VersionedChange("key", 1, "first"),
            VersionedChange("key", 1, "second"),
        ),
        horizon=3,
    )
except ValueError:
    duplicate_rejected = True
else:
    duplicate_rejected = False

assert (
    checked == 6_144
    and sample
    == (
        ValueValidity("alpha", 0, 3, True),
        ValueValidity("alpha", 3, 8, 1),
        ValueValidity("beta", 1, 8, "warm"),
    )
    and duplicate_rejected
)
```

## Trade-offs and Limitations

For `R` records, grouping and sorting take `O(R log R + B)` time for `B`
inspected UTF-8 bytes. The function uses `O(R)` memory and emits at most `R`
intervals. Exact type-and-value comparison deliberately keeps `False`, `0`,
and `None` as three different states despite ordinary Python equality between
the first two.

The common finite horizon keeps every result compatible with ordinary
half-open interval algorithms. Changing that horizon, interpreting versions as
time, representing deletion, or accepting corrections requires a different
domain contract rather than a small modification to this one.

## Related Snippets

<!-- catalog:related:start -->
- [Join Non-Overlapping Half-Open Validity Intervals by Exact Overlap](join-non-overlapping-half-open-validity-intervals-by-exact-overlap.md)
- [Join Bounded Strictly Increasing Sequences by the Latest Prior Timestamp](join-bounded-strictly-increasing-sequences-by-the-latest-prior-timestamp.md)
- [Compare Ordered Integer Time-Series Snapshots with Explicit Tolerance](compare-ordered-integer-time-series-snapshots-with-explicit-tolerance.md)
<!-- catalog:related:end -->
