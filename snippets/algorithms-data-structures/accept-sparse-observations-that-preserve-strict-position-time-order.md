---
title: "Accept Sparse Observations That Preserve Strict Position-Time Order"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - find-a-point-in-disjoint-half-open-intervals.md
  - ../data-processing/select-one-record-per-key-with-an-explicit-ranking-rule.md
  - ../data-processing/yield-stream-items-with-bounded-neighbor-context.md
---

# Accept Sparse Observations That Preserve Strict Position-Time Order

## Idea and Problem

Accept sparse position-time observations online only when the accepted state remains strictly increasing by position.

Arrival order is authoritative. For each new position, only the nearest
accepted neighbors can constrain its time: the left time must be smaller and
the right time must be larger. The first accepted observation for a position
wins, while conflicts are retained separately for inspection.

## When to Use

Use this greedy filter when observations arrive in a trusted priority order,
positions are unique logical slots, and preserving a strict local ordering is
more important than finding the largest possible subset. It fits bounded data
cleanup or validation before a later ordered calculation. Use an offline
optimization algorithm when input order is not authoritative, or a versioned
state model when corrections, retractions, and late-event watermarks matter.

## Implementation

```python
from bisect import bisect_left
from collections.abc import Iterable
from dataclasses import dataclass


_MAX_OBSERVATIONS = 10_000


@dataclass(frozen=True, slots=True)
class PositionObservation:
    position: int
    observed_at: int


@dataclass(frozen=True, slots=True)
class MonotonicObservationResult:
    accepted: tuple[PositionObservation, ...]
    rejected: tuple[PositionObservation, ...]


def _validate_observation(observation: PositionObservation) -> None:
    if not isinstance(observation, PositionObservation):
        raise TypeError("observations must contain PositionObservation values")
    if (
        isinstance(observation.position, bool)
        or not isinstance(observation.position, int)
    ):
        raise TypeError("position must be an integer")
    if observation.position < 0:
        raise ValueError("position must be non-negative")
    if (
        isinstance(observation.observed_at, bool)
        or not isinstance(observation.observed_at, int)
    ):
        raise TypeError("observed_at must be an integer")


def accept_strictly_ordered_observations(
    observations: Iterable[PositionObservation],
) -> MonotonicObservationResult:
    positions: list[int] = []
    accepted_by_position: dict[int, PositionObservation] = {}
    accepted: list[PositionObservation] = []
    rejected: list[PositionObservation] = []

    for observation_count, observation in enumerate(observations, start=1):
        if observation_count > _MAX_OBSERVATIONS:
            raise ValueError("observations exceed the supported count")
        _validate_observation(observation)

        insertion_index = bisect_left(positions, observation.position)
        if observation.position in accepted_by_position:
            rejected.append(observation)
            continue

        left = (
            accepted_by_position[positions[insertion_index - 1]]
            if insertion_index > 0
            else None
        )
        right = (
            accepted_by_position[positions[insertion_index]]
            if insertion_index < len(positions)
            else None
        )
        violates_left = (
            left is not None
            and observation.observed_at <= left.observed_at
        )
        violates_right = (
            right is not None
            and observation.observed_at >= right.observed_at
        )
        if violates_left or violates_right:
            rejected.append(observation)
            continue

        positions.insert(insertion_index, observation.position)
        accepted_by_position[observation.position] = observation
        accepted.append(observation)

    return MonotonicObservationResult(tuple(accepted), tuple(rejected))
```

## Example

```python
observations = (
    PositionObservation(0, 10),
    PositionObservation(3, 40),
    PositionObservation(1, 20),
    PositionObservation(2, 35),
    PositionObservation(2, 30),
    PositionObservation(4, 39),
    PositionObservation(4, 50),
)
result = accept_strictly_ordered_observations(observations)
ordered = tuple(sorted(result.accepted, key=lambda item: item.position))
strictly_increasing = all(
    left.observed_at < right.observed_at
    for left, right in zip(ordered, ordered[1:])
)

assert (result, strictly_increasing) == (
    MonotonicObservationResult(
        accepted=(
            PositionObservation(0, 10),
            PositionObservation(3, 40),
            PositionObservation(1, 20),
            PositionObservation(2, 35),
            PositionObservation(4, 50),
        ),
        rejected=(
            PositionObservation(2, 30),
            PositionObservation(4, 39),
        ),
    ),
    True,
)
```

## Trade-offs and Limitations

The result is greedy and irreversible: changing arrival order can change which
observations survive, and the result is not a longest or maximum-cardinality
monotonic subset. Equal times are rejected because the invariant is strict,
and an accepted position cannot be corrected later. The returned tuples retain
every input object, including rejected values. List insertion is linear, so
the worst case is `O(n^2)` time with `O(n)` memory; the 10,000-item ceiling
keeps that trade-off explicit. The function has no event-time watermark,
expiry, persistence, concurrency control, or domain-specific conflict policy.

## Related Snippets

<!-- catalog:related:start -->
- [Find a Point in Disjoint Half-Open Intervals](find-a-point-in-disjoint-half-open-intervals.md)
- [Select One Record per Key with an Explicit Ranking Rule](../data-processing/select-one-record-per-key-with-an-explicit-ranking-rule.md)
- [Yield Stream Items with Bounded Neighbor Context](../data-processing/yield-stream-items-with-bounded-neighbor-context.md)
<!-- catalog:related:end -->
