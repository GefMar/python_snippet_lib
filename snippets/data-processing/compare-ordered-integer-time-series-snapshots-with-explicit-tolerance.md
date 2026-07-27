---
title: "Compare Ordered Integer Time-Series Snapshots with Explicit Tolerance"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - join-two-strictly-increasing-streams-by-exact-timestamp.md
  - check-a-value-against-an-asymmetric-tolerance-band.md
  - overlay-aligned-time-series-at-the-finest-step.md
---

# Compare Ordered Integer Time-Series Snapshots with Explicit Tolerance

## Idea and Problem

Compare two bounded integer time-series snapshots in one linear pass while preserving missing, new, and materially changed points as distinct outcomes.

Strict timestamp ordering lets two indexes advance like a merge step. A point
present only in the earlier snapshot is `missing-current`, one present only in
the later snapshot is `new-current`, and a shared point is `changed` only when
its absolute value difference exceeds the inclusive tolerance boundary. Zero
is an ordinary value, not a missing marker.

## When to Use

Use this algorithm when both snapshots were captured under compatible
semantics, timestamps are exact integer identities, and one absolute integer
tolerance applies to every shared point. The frozen, timestamp-ordered result
fits deterministic validation, audit, or downstream rendering without
mutating either input.

Reconcile units, timestamp grids, and snapshot identity before comparison.
Choose a statistical method instead when the permitted difference must be
learned from variability, and resample explicitly when nearby timestamps are
intended to represent the same point.

## Implementation

```python
from dataclasses import dataclass
from typing import Literal

_MIN_INTEGER = -(1 << 63)
_MAX_INTEGER = (1 << 63) - 1
_MAX_TOLERANCE = (1 << 64) - 1
_MAX_POINTS = 4_096


ViolationKind = Literal["missing-current", "new-current", "changed"]


def _bounded_integer(
    value: object,
    *,
    name: str,
    minimum: int,
    maximum: int,
) -> int:
    if type(value) is not int:
        raise TypeError(f"{name} must be an exact integer")
    if not minimum <= value <= maximum:
        raise ValueError(f"{name} is outside the supported range")
    return value


@dataclass(frozen=True, slots=True)
class IntegerTimePoint:
    timestamp: int
    value: int

    def __post_init__(self) -> None:
        _bounded_integer(
            self.timestamp,
            name="timestamp",
            minimum=_MIN_INTEGER,
            maximum=_MAX_INTEGER,
        )
        _bounded_integer(
            self.value,
            name="value",
            minimum=_MIN_INTEGER,
            maximum=_MAX_INTEGER,
        )


@dataclass(frozen=True, slots=True)
class SnapshotViolation:
    timestamp: int
    kind: ViolationKind
    previous_value: int | None
    current_value: int | None


def _validated_snapshot(
    points: object,
    *,
    name: str,
) -> tuple[IntegerTimePoint, ...]:
    if type(points) is not tuple:
        raise TypeError(f"{name} must be an exact tuple")
    if len(points) > _MAX_POINTS:
        raise ValueError(f"{name} exceeds the supported point count")

    validated = []
    previous_timestamp: int | None = None
    for index, point in enumerate(points):
        if type(point) is not IntegerTimePoint:
            raise TypeError(f"{name}[{index}] must be an exact IntegerTimePoint")
        timestamp = _bounded_integer(
            point.timestamp,
            name=f"{name}[{index}].timestamp",
            minimum=_MIN_INTEGER,
            maximum=_MAX_INTEGER,
        )
        value = _bounded_integer(
            point.value,
            name=f"{name}[{index}].value",
            minimum=_MIN_INTEGER,
            maximum=_MAX_INTEGER,
        )
        if previous_timestamp is not None and timestamp <= previous_timestamp:
            raise ValueError(f"timestamps in {name} must be strictly increasing")
        validated.append(IntegerTimePoint(timestamp, value))
        previous_timestamp = timestamp
    return tuple(validated)


def compare_integer_time_series_snapshots(
    previous: tuple[IntegerTimePoint, ...],
    current: tuple[IntegerTimePoint, ...],
    *,
    tolerance: int,
) -> tuple[SnapshotViolation, ...]:
    earlier = _validated_snapshot(previous, name="previous")
    later = _validated_snapshot(current, name="current")
    allowed_difference = _bounded_integer(
        tolerance,
        name="tolerance",
        minimum=0,
        maximum=_MAX_TOLERANCE,
    )

    violations = []
    previous_index = 0
    current_index = 0
    while previous_index < len(earlier) or current_index < len(later):
        old = earlier[previous_index] if previous_index < len(earlier) else None
        new = later[current_index] if current_index < len(later) else None

        if new is None or (old is not None and old.timestamp < new.timestamp):
            violations.append(
                SnapshotViolation(
                    timestamp=old.timestamp,
                    kind="missing-current",
                    previous_value=old.value,
                    current_value=None,
                )
            )
            previous_index += 1
        elif old is None or new.timestamp < old.timestamp:
            violations.append(
                SnapshotViolation(
                    timestamp=new.timestamp,
                    kind="new-current",
                    previous_value=None,
                    current_value=new.value,
                )
            )
            current_index += 1
        else:
            if abs(new.value - old.value) > allowed_difference:
                violations.append(
                    SnapshotViolation(
                        timestamp=old.timestamp,
                        kind="changed",
                        previous_value=old.value,
                        current_value=new.value,
                    )
                )
            previous_index += 1
            current_index += 1

    return tuple(violations)
```

## Example

```python
previous = (
    IntegerTimePoint(10, 0),
    IntegerTimePoint(20, 100),
    IntegerTimePoint(30, 200),
    IntegerTimePoint(50, 500),
)
current = (
    IntegerTimePoint(10, 2),
    IntegerTimePoint(20, 103),
    IntegerTimePoint(40, 400),
    IntegerTimePoint(50, 498),
)

violations = compare_integer_time_series_snapshots(
    previous,
    current,
    tolerance=2,
)

try:
    compare_integer_time_series_snapshots(
        (IntegerTimePoint(1, 10), IntegerTimePoint(1, 11)),
        (),
        tolerance=0,
    )
except ValueError:
    duplicate_timestamp_rejected = True
else:
    duplicate_timestamp_rejected = False

assert (violations, duplicate_timestamp_rejected) == (
    (
        SnapshotViolation(20, "changed", 100, 103),
        SnapshotViolation(30, "missing-current", 200, None),
        SnapshotViolation(40, "new-current", None, 400),
    ),
    True,
)
```

## Trade-offs and Limitations

Validation and comparison take `O(n + m)` time. Normalized snapshots use
`O(n + m)` temporary space, and the immutable result uses `O(v)` space for at
most 8,192 violations. Exact tuples and frozen point objects deliberately
exclude lazy, mutable, or partially consumed inputs.

Tolerance is one inclusive absolute difference in the same integer unit as
every value; the algorithm provides no percentage rule, special policy for a
zero-valued previous point, interpolation, or outage semantics. It also does
not detect anomalies, forecast values, issue alerts, parse timestamps, or
perform I/O. Snapshot capture and identity, unit conversion, presentation, and
response policy remain outside this comparison primitive.

## Related Snippets

<!-- catalog:related:start -->
- [Join Two Strictly Increasing Streams by Exact Timestamp](join-two-strictly-increasing-streams-by-exact-timestamp.md)
- [Check a Value Against an Asymmetric Tolerance Band](check-a-value-against-an-asymmetric-tolerance-band.md)
- [Overlay Aligned Time Series at the Finest Step](overlay-aligned-time-series-at-the-finest-step.md)
<!-- catalog:related:end -->
