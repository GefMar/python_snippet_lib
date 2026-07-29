---
title: "Assign Bounded Half-Open Integer Intervals to the Minimum Number of Reusable Rooms"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - select-a-maximum-cardinality-set-of-non-overlapping-half-open-integer-intervals.md
  - find-peak-coverage-spans-across-bounded-half-open-integer-intervals.md
  - partition-tagged-items-into-minimum-stable-conflict-free-groups.md
---

# Assign Bounded Half-Open Integer Intervals to the Minimum Number of Reusable Rooms

## Idea and Problem

Assign every bounded half-open interval to a reusable room while using the minimum possible number of rooms and making room identifiers deterministic.

Process intervals by start, stop, and declaration index. An active heap exposes
every room whose previous interval has ended; a second heap makes the smallest
available room identifier the next choice. A new room is necessary only when
all existing rooms overlap the current interval, which also proves that the
final room count is minimal.

## When to Use

Use this algorithm when every interval must be retained, overlapping intervals
need distinct reusable resources, and half-open integer endpoints already share
one timeline. The stable assignment is useful for reproducible schedules,
lanes, fixtures, or visual layouts where touching intervals may reuse a room.

Use ordinary interval scheduling when some intervals may be discarded, or a
capacity-aware optimizer when resources differ. Setup gaps, preferences,
preemption, online arrivals, and mutable schedules require additional policies
that this fixed snapshot does not model.

## Implementation

```python
from heapq import heappop, heappush

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_ROOM_INTERVALS = 100_000


def assign_intervals_to_minimum_rooms(
    intervals: tuple[tuple[int, int], ...],
) -> tuple[int, tuple[int, ...]]:
    """Return the minimum room count and one deterministic assignment."""
    if type(intervals) is not tuple:
        raise TypeError("intervals must be an exact tuple")
    if len(intervals) > _MAX_ROOM_INTERVALS:
        raise ValueError("interval count exceeds the supported limit")

    for index, interval in enumerate(intervals):
        if type(interval) is not tuple:
            raise TypeError(f"intervals[{index}] must be an exact tuple")
        if len(interval) != 2:
            raise ValueError(f"intervals[{index}] must contain two items")
        start, stop = interval
        if type(start) is not int:
            raise TypeError(f"intervals[{index}].start must be an exact non-boolean integer")
        if type(stop) is not int:
            raise TypeError(f"intervals[{index}].stop must be an exact non-boolean integer")
        if not _MIN_INT64 <= start <= _MAX_INT64:
            raise ValueError(f"intervals[{index}].start is outside the signed 64-bit range")
        if not _MIN_INT64 <= stop <= _MAX_INT64:
            raise ValueError(f"intervals[{index}].stop is outside the signed 64-bit range")
        if start >= stop:
            raise ValueError(f"intervals[{index}] must have start before stop")

    ordered_indexes = sorted(
        range(len(intervals)),
        key=lambda index: (*intervals[index], index),
    )
    assignments = [-1] * len(intervals)
    active: list[tuple[int, int]] = []
    available: list[int] = []
    room_count = 0

    for index in ordered_indexes:
        start, stop = intervals[index]
        while active and active[0][0] <= start:
            _, room_id = heappop(active)
            heappush(available, room_id)

        if available:
            room_id = heappop(available)
        else:
            room_id = room_count
            room_count += 1

        assignments[index] = room_id
        heappush(active, (stop, room_id))

    return room_count, tuple(assignments)
```

## Example

```python
def list_based_room_assignment(
    intervals: tuple[tuple[int, int], ...],
) -> tuple[int, tuple[int, ...]]:
    room_stops: list[int] = []
    assignments = [-1] * len(intervals)
    order = sorted(range(len(intervals)), key=lambda index: (*intervals[index], index))

    for index in order:
        start, stop = intervals[index]
        reusable = [
            room_id for room_id, previous_stop in enumerate(room_stops) if previous_stop <= start
        ]
        if reusable:
            room_id = min(reusable)
            room_stops[room_id] = stop
        else:
            room_id = len(room_stops)
            room_stops.append(stop)
        assignments[index] = room_id

    return len(room_stops), tuple(assignments)


def intervals_overlap(left: tuple[int, int], right: tuple[int, int]) -> bool:
    return left[0] < right[1] and right[0] < left[1]


def brute_minimum_room_count(intervals: tuple[tuple[int, int], ...]) -> int:
    order = sorted(range(len(intervals)), key=lambda index: (*intervals[index], index))
    rooms: list[list[int]] = []
    best = len(intervals) + 1

    def search(offset: int) -> None:
        nonlocal best
        if offset == len(order):
            best = min(best, len(rooms))
            return
        if len(rooms) >= best:
            return

        interval_index = order[offset]
        for members in rooms:
            if all(
                not intervals_overlap(intervals[interval_index], intervals[member])
                for member in members
            ):
                members.append(interval_index)
                search(offset + 1)
                members.pop()

        rooms.append([interval_index])
        search(offset + 1)
        rooms.pop()

    search(0)
    return 0 if not intervals else best


def peak_half_open_overlap(intervals: tuple[tuple[int, int], ...]) -> int:
    if not intervals:
        return 0
    return max(
        sum(start <= point < stop for start, stop in intervals)
        for point in {start for start, _ in intervals}
    )


def exercise_tiny_room_assignments() -> int:
    from itertools import combinations, product

    options = tuple(combinations(range(5), 2))
    checked = 0
    for count in range(5):
        for intervals in product(options, repeat=count):
            actual = assign_intervals_to_minimum_rooms(intervals)
            assert actual == list_based_room_assignment(intervals)
            assert actual[0] == brute_minimum_room_count(intervals)
            assert actual[0] == peak_half_open_overlap(intervals)
            checked += 1
    return checked


checked_cases = exercise_tiny_room_assignments()
touching = assign_intervals_to_minimum_rooms(((0, 2), (2, 4), (4, 6)))
duplicates = assign_intervals_to_minimum_rooms(((0, 5), (0, 5), (0, 5)))
extrema = assign_intervals_to_minimum_rooms(
    ((_MIN_INT64, -1), (-1, 0), (_MIN_INT64, _MAX_INT64), (0, _MAX_INT64))
)

try:
    assign_intervals_to_minimum_rooms(((0, 0),))
except ValueError:
    empty_span_rejected = True
else:
    empty_span_rejected = False

try:
    assign_intervals_to_minimum_rooms(((0, True),))
except TypeError:
    boolean_rejected = True
else:
    boolean_rejected = False

assert (
    checked_cases,
    touching,
    duplicates,
    extrema,
    empty_span_rejected,
    boolean_rejected,
) == (
    11_111,
    (1, (0, 0, 0)),
    (3, (0, 1, 2)),
    (2, (0, 0, 1, 0)),
    True,
    True,
)
```

## Trade-offs and Limitations

Validation and sorting cost `O(N log N)` time overall. Each interval enters and
leaves the active heap once, and each reusable room identifier enters and
leaves the available heap as needed. The sorted indexes, assignment list, and
two heaps use `O(N)` auxiliary memory.

Allocating a room means every existing room is active at the current start, so
those overlapping intervals provide a matching lower bound on any assignment.
Half-open semantics allow a room ending at `start` to be reused immediately.
Room identifiers are deterministic for one declared input: intervals are
processed by `(start, stop, declaration_index)` and the smallest available id
wins. Reordering tied declarations can therefore relabel their assignments.

The function assigns every interval but returns no per-room schedule. It does
not support capacities, weights, setup gaps, preemption, online arrivals,
resource affinities, mutable updates, or alternative globally optimized label
policies.

## Related Snippets

<!-- catalog:related:start -->
- [Select a Maximum-Cardinality Set of Non-Overlapping Half-Open Integer Intervals](select-a-maximum-cardinality-set-of-non-overlapping-half-open-integer-intervals.md)
- [Find Peak Coverage Spans Across Bounded Half-Open Integer Intervals](find-peak-coverage-spans-across-bounded-half-open-integer-intervals.md)
- [Partition Tagged Items into Minimum Stable Conflict-Free Groups](partition-tagged-items-into-minimum-stable-conflict-free-groups.md)
<!-- catalog:related:end -->
