---
title: "Maintain a Bounded Integer Multiset with Fenwick Rank and Select"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - maintain-bounded-point-adds-and-half-open-range-sums-with-a-fenwick-tree.md
  - select-a-zero-based-order-statistic-from-bounded-integers-with-three-way-quickselect.md
  - answer-static-k-th-smallest-queries-on-bounded-integer-ranges-with-a-persistent-segment-tree.md
---

# Maintain a Bounded Integer Multiset with Fenwick Rank and Select

## Idea and Problem

Own counts over a fixed sorted integer universe so duplicate-aware insertion, removal, rank, and selection remain bounded and logarithmic.

A coordinate map resolves mutations and exact-count queries, while a Fenwick
tree stores prefix occurrence totals. `rank(value)` sums the coordinates below
the insertion point of any signed 64-bit integer, and `select(index)` uses
Fenwick binary lifting to find the coordinate containing one zero-based
occurrence without expanding duplicates into a list.

## When to Use

Use this structure when the complete coordinate universe is known in advance,
duplicates matter, and a bounded mutable population needs frequent rank or
order-statistic queries. Typical uses include compact simulation state,
bounded frequency indexes, and deterministic test or scheduling models.

Use a sorted tree when coordinates must be inserted dynamically, a hash-based
counter when only exact counts matter, or a flat sorted sequence when updates
are rare enough that simpler linear rebuilding is preferable.

## Implementation

```python
from bisect import bisect_left

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_UNIVERSE_SIZE = 4_096
_MAX_COUNT = 1_000_000


class FenwickIntegerMultiset:
    """Maintain bounded multiplicities over one fixed integer universe."""

    __slots__ = ("_counts", "_positions", "_total", "_tree", "_universe")

    def __init__(
        self,
        universe: tuple[int, ...],
        initial_counts: tuple[int, ...],
    ) -> None:
        if type(universe) is not tuple:
            raise TypeError("universe must be an exact tuple")
        if not 1 <= len(universe) <= _MAX_UNIVERSE_SIZE:
            raise ValueError("universe size is outside the supported range")
        if type(initial_counts) is not tuple:
            raise TypeError("initial_counts must be an exact tuple")
        if len(initial_counts) != len(universe):
            raise ValueError("initial_counts must match the universe length")

        previous: int | None = None
        for value in universe:
            if type(value) is not int:
                raise TypeError("universe must contain exact integers")
            if not _MIN_INT64 <= value <= _MAX_INT64:
                raise ValueError("universe values must be signed 64-bit integers")
            if previous is not None and value <= previous:
                raise ValueError("universe must be strictly increasing")
            previous = value

        owned_counts: list[int] = []
        total = 0
        for count in initial_counts:
            if type(count) is not int:
                raise TypeError("initial_counts must contain exact integers")
            if not 0 <= count <= _MAX_COUNT:
                raise ValueError("each initial count must be between zero and the cap")
            total += count
            if total > _MAX_COUNT:
                raise ValueError("the initial total exceeds the supported cap")
            owned_counts.append(count)

        tree = [0, *owned_counts]
        for position in range(1, len(tree)):
            parent = position + (position & -position)
            if parent < len(tree):
                tree[parent] += tree[position]

        self._universe = universe
        self._positions = {value: index for index, value in enumerate(universe)}
        self._counts = owned_counts
        self._tree = tree
        self._total = total

    def __len__(self) -> int:
        return self._total

    @staticmethod
    def _validated_value(value: object) -> int:
        if type(value) is not int:
            raise TypeError("value must be an exact integer")
        if not _MIN_INT64 <= value <= _MAX_INT64:
            raise ValueError("value must be a signed 64-bit integer")
        return value

    @staticmethod
    def _validated_positive_count(count: object) -> int:
        if type(count) is not int:
            raise TypeError("count must be an exact integer")
        if not 1 <= count <= _MAX_COUNT:
            raise ValueError("count must be between one and the supported cap")
        return count

    def _known_position(self, value: object) -> int:
        checked_value = self._validated_value(value)
        try:
            return self._positions[checked_value]
        except KeyError:
            raise ValueError("mutation value is outside the fixed universe") from None

    def _prefix_sum(self, stop: int) -> int:
        total = 0
        while stop > 0:
            total += self._tree[stop]
            stop -= stop & -stop
        return total

    def _commit_change(
        self,
        position: int,
        delta: int,
        *,
        new_coordinate_count: int,
        new_total: int,
    ) -> None:
        self._counts[position] = new_coordinate_count
        self._total = new_total
        tree_position = position + 1
        while tree_position < len(self._tree):
            self._tree[tree_position] += delta
            tree_position += tree_position & -tree_position

    def add(self, value: int, count: int = 1) -> None:
        """Add a positive multiplicity after validating every affected cap."""
        position = self._known_position(value)
        checked_count = self._validated_positive_count(count)
        new_coordinate_count = self._counts[position] + checked_count
        new_total = self._total + checked_count
        if new_coordinate_count > _MAX_COUNT:
            raise ValueError("the coordinate count would exceed the supported cap")
        if new_total > _MAX_COUNT:
            raise ValueError("the total count would exceed the supported cap")
        self._commit_change(
            position,
            checked_count,
            new_coordinate_count=new_coordinate_count,
            new_total=new_total,
        )

    def remove(self, value: int, count: int = 1) -> None:
        """Remove a positive multiplicity without allowing underflow."""
        position = self._known_position(value)
        checked_count = self._validated_positive_count(count)
        new_coordinate_count = self._counts[position] - checked_count
        new_total = self._total - checked_count
        if new_coordinate_count < 0 or new_total < 0:
            raise ValueError("removal would underflow the stored multiplicity")
        self._commit_change(
            position,
            -checked_count,
            new_coordinate_count=new_coordinate_count,
            new_total=new_total,
        )

    def count(self, value: int) -> int:
        """Return one coordinate count, or zero outside the fixed universe."""
        checked_value = self._validated_value(value)
        position = self._positions.get(checked_value)
        return 0 if position is None else self._counts[position]

    def rank(self, value: int) -> int:
        """Return the number of occurrences strictly below value."""
        checked_value = self._validated_value(value)
        position = bisect_left(self._universe, checked_value)
        return self._prefix_sum(position)

    def select(self, index: int) -> int:
        """Return the value at one zero-based occurrence index."""
        if type(index) is not int:
            raise TypeError("index must be an exact integer")
        if not 0 <= index < self._total:
            raise IndexError("occurrence index is outside the multiset")

        position = 0
        prefix = 0
        step = 1 << (len(self._universe).bit_length() - 1)
        while step > 0:
            candidate = position + step
            if candidate < len(self._tree) and prefix + self._tree[candidate] <= index:
                position = candidate
                prefix += self._tree[candidate]
            step >>= 1
        return self._universe[position]


```

## Example

```python
multiset = FenwickIntegerMultiset((-5, 0, 7, 10), (2, 0, 3, 1))

assert len(multiset) == 6
assert multiset.count(7) == 3
assert multiset.count(6) == 0
assert (multiset.rank(-5), multiset.rank(7), multiset.rank(8)) == (0, 2, 5)
assert tuple(multiset.select(index) for index in range(len(multiset))) == (
    -5,
    -5,
    7,
    7,
    7,
    10,
)

multiset.add(0, 2)
multiset.remove(7, 2)
assert tuple(multiset.select(index) for index in range(len(multiset))) == (
    -5,
    -5,
    0,
    0,
    7,
    10,
)

before_underflow = (len(multiset), multiset.count(7))
try:
    multiset.remove(7, 2)
except ValueError:
    pass
else:
    raise AssertionError("coordinate underflow was accepted")
assert (len(multiset), multiset.count(7)) == before_underflow

capped = FenwickIntegerMultiset((1, 2), (999_999, 1))
before_cap = (len(capped), capped.count(2))
try:
    capped.add(2)
except ValueError:
    pass
else:
    raise AssertionError("total overflow was accepted")
assert (len(capped), capped.count(2)) == before_cap

before_unknown = (len(multiset), multiset.count(0))
try:
    multiset.add(99)
except ValueError:
    pass
else:
    raise AssertionError("an unknown mutation coordinate was accepted")
assert (len(multiset), multiset.count(0)) == before_unknown
```

## Trade-offs and Limitations

Validation and Fenwick construction take `O(U)` time for `U` coordinates.
Mutations, rank, and selection take `O(log U)` time; exact counting uses the
coordinate map, and the structure uses `O(U)` memory. The universe has at most
4,096 strictly increasing signed 64-bit integers, while every coordinate count
and the total population are capped at 1,000,000.

The fixed universe cannot grow, and the mutable object offers no iterator,
snapshot, persistence, or synchronization. Callers must serialize concurrent
access. Rejected additions and removals leave both the counts and Fenwick tree
unchanged because all coordinate and total checks precede mutation.

## Related Snippets

<!-- catalog:related:start -->
- [Maintain Bounded Point Adds and Half-Open Range Sums with a Fenwick Tree](maintain-bounded-point-adds-and-half-open-range-sums-with-a-fenwick-tree.md)
- [Select a Zero-Based Order Statistic from Bounded Integers with Three-Way Quickselect](select-a-zero-based-order-statistic-from-bounded-integers-with-three-way-quickselect.md)
- [Answer Static K-th-Smallest Queries on Bounded Integer Ranges with a Persistent Segment Tree](answer-static-k-th-smallest-queries-on-bounded-integer-ranges-with-a-persistent-segment-tree.md)
<!-- catalog:related:end -->
