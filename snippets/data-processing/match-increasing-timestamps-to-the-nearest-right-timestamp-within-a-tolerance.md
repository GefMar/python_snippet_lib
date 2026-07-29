---
title: "Match Increasing Timestamps to the Nearest Right Timestamp Within a Tolerance"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - join-bounded-strictly-increasing-sequences-by-the-latest-prior-timestamp.md
  - join-two-strictly-increasing-streams-by-exact-timestamp.md
  - ../algorithms-data-structures/match-strict-mutual-nearest-neighbors-with-a-comparison-budget.md
---

# Match Increasing Timestamps to the Nearest Right Timestamp Within a Tolerance

## Idea and Problem

Match every increasing left timestamp to its nearest increasing right timestamp when the exact distance stays inside one inclusive tolerance.

For each left timestamp, only the right predecessor and the first right value
that is not smaller can be nearest. A single forward-moving cursor identifies
those two candidates. Comparing their distances chooses the nearer one, and an
equal distance deliberately selects the earlier right timestamp.

## When to Use

Use this algorithm for two bounded, already sorted timestamp snapshots when a
left item may use either a slightly earlier or a slightly later right item.
Many-to-one matching is intentional, so one right index can serve consecutive
left timestamps.

Use a latest-prior join when future right timestamps must never qualify. Choose
a one-to-one assignment algorithm when right items must be consumed, or a
stateful streaming join when late records can arrive after a match is emitted.

## Implementation

```python
from dataclasses import dataclass

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_TIMESTAMPS_PER_SIDE = 65_536


@dataclass(frozen=True, slots=True)
class NearestTimestampMatch:
    right_index: int
    right_timestamp: int
    distance: int


def _validate_increasing_timestamps(
    timestamps: object,
    *,
    name: str,
    allow_empty: bool,
) -> tuple[int, ...]:
    if type(timestamps) is not tuple:
        raise TypeError(f"{name} must be an exact tuple")
    minimum_count = 0 if allow_empty else 1
    if not minimum_count <= len(timestamps) <= _MAX_TIMESTAMPS_PER_SIDE:
        raise ValueError(f"{name} count is outside the supported range")

    previous: int | None = None
    for index, timestamp in enumerate(timestamps):
        if type(timestamp) is not int:
            raise TypeError(f"{name}[{index}] must be an exact integer")
        if not _MIN_INT64 <= timestamp <= _MAX_INT64:
            raise ValueError(f"{name}[{index}] is outside the signed 64-bit range")
        if previous is not None and timestamp <= previous:
            raise ValueError(f"{name} timestamps must strictly increase")
        previous = timestamp
    return timestamps


def match_nearest_right_timestamps(
    left_timestamps: tuple[int, ...],
    right_timestamps: tuple[int, ...],
    *,
    maximum_distance: int,
) -> tuple[NearestTimestampMatch | None, ...]:
    """Return one nearest-right result aligned with every left timestamp."""
    left = _validate_increasing_timestamps(
        left_timestamps,
        name="left_timestamps",
        allow_empty=False,
    )
    right = _validate_increasing_timestamps(
        right_timestamps,
        name="right_timestamps",
        allow_empty=True,
    )
    if type(maximum_distance) is not int:
        raise TypeError("maximum_distance must be an exact integer")
    if not 0 <= maximum_distance <= _MAX_INT64:
        raise ValueError("maximum_distance is outside the supported range")

    matches: list[NearestTimestampMatch | None] = []
    right_cursor = 0

    for left_timestamp in left:
        while right_cursor < len(right) and right[right_cursor] < left_timestamp:
            right_cursor += 1

        if not right:
            matches.append(None)
            continue

        if right_cursor == 0:
            candidate_index = 0
        elif right_cursor == len(right):
            candidate_index = len(right) - 1
        else:
            earlier_distance = left_timestamp - right[right_cursor - 1]
            later_distance = right[right_cursor] - left_timestamp
            candidate_index = (
                right_cursor - 1 if earlier_distance <= later_distance else right_cursor
            )

        candidate_timestamp = right[candidate_index]
        distance = abs(candidate_timestamp - left_timestamp)
        if distance <= maximum_distance:
            matches.append(
                NearestTimestampMatch(
                    right_index=candidate_index,
                    right_timestamp=candidate_timestamp,
                    distance=distance,
                )
            )
        else:
            matches.append(None)

    return tuple(matches)
```

## Example

```python
matches = match_nearest_right_timestamps(
    (0, 5, 6, 20),
    (2, 8, 30),
    maximum_distance=4,
)
empty_right = match_nearest_right_timestamps(
    (1, 3),
    (),
    maximum_distance=10,
)

assert (matches, empty_right) == (
    (
        NearestTimestampMatch(0, 2, 2),
        NearestTimestampMatch(0, 2, 3),
        NearestTimestampMatch(1, 8, 2),
        None,
    ),
    (None, None),
)
```

## Trade-offs and Limitations

Complete validation and matching take `O(L + R)` time because the right cursor
never moves backward. The frozen aligned result uses `O(L)` memory, while the
scan itself uses `O(1)` auxiliary state. Python computes distances exactly even
when subtracting two signed-64-bit endpoints produces a larger magnitude.

The inclusive tolerance applies independently to each left timestamp, and
right indexes may repeat. Inputs must already be strictly increasing; the
function does not sort, deduplicate, copy payloads, interpolate values, consume
right items once, attach timezone semantics, accept floating-point timestamps,
or handle late streaming records.

## Related Snippets

<!-- catalog:related:start -->
- [Join Bounded Strictly Increasing Sequences by the Latest Prior Timestamp](join-bounded-strictly-increasing-sequences-by-the-latest-prior-timestamp.md)
- [Join Two Strictly Increasing Streams by Exact Timestamp](join-two-strictly-increasing-streams-by-exact-timestamp.md)
- [Match Strict Mutual Nearest Neighbors with a Comparison Budget](../algorithms-data-structures/match-strict-mutual-nearest-neighbors-with-a-comparison-budget.md)
<!-- catalog:related:end -->
