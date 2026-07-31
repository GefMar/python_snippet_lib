---
title: "Maintain a Bounded Indexed Min-Priority Queue with Exact Updates"
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
  - find-one-deterministic-shortest-path-in-a-bounded-non-negatively-weighted-directed-graph.md
  - maintain-a-bounded-integer-multiset-with-fenwick-rank-and-select.md
  - select-the-stable-k-smallest-bounded-records-with-a-max-heap.md
---

# Maintain a Bounded Indexed Min-Priority Queue with Exact Updates

## Idea and Problem

Maintain one bounded min-priority queue whose integer items can be inserted, reprioritized, removed, and popped without leaving stale heap entries.

Python's `heapq` deliberately has no handle-based priority-update operation.
Pushing a replacement entry and ignoring stale copies is often sufficient, but
the heap can then grow with the number of updates rather than the number of
live items. An indexed heap instead keeps a position table from every supported
item ID to its one current heap slot.

Changing a priority repairs the existing slot upward or downward. Removing an
arbitrary item moves the last heap item into the gap, updates its recorded
position, and repairs only that path. Equal priorities are resolved by the
smallest item ID, so every observable minimum is deterministic.

## When to Use

Use this structure for a bounded, dense item-ID domain when priorities change
repeatedly and retaining stale entries is undesirable. It fits graph
frontiers, schedulers, simulations, and deterministic tests that need exact
increase, decrease, deletion, and minimum-pop behavior in logarithmic time.

Use ordinary `heapq` with lazy deletion when updates are infrequent, the item
domain is sparse or unbounded, or the simpler approach has an acceptable memory
bound. Use a sorted collection or specialized library when arbitrary keys,
custom comparison rules, persistence, concurrency, or meld operations are part
of the contract.

## Implementation

```python
from dataclasses import dataclass


_MAX_INDEXED_QUEUE_CAPACITY = 4_096
_MIN_SIGNED_64 = -(1 << 63)
_MAX_SIGNED_64 = (1 << 63) - 1


@dataclass(frozen=True, slots=True)
class PriorityItem:
    item: int
    priority: int


class IndexedMinPriorityQueue:
    """Maintain one live heap entry per bounded integer item ID."""

    __slots__ = ("_capacity", "_heap", "_positions", "_priorities")

    def __init__(self, capacity: int) -> None:
        if type(capacity) is not int:
            raise TypeError("capacity must be an exact integer")
        if not 1 <= capacity <= _MAX_INDEXED_QUEUE_CAPACITY:
            raise ValueError("capacity is outside 1..4,096")

        self._capacity = capacity
        self._heap: list[int] = []
        self._positions = [-1] * capacity
        self._priorities: list[int | None] = [None] * capacity

    def __len__(self) -> int:
        return len(self._heap)

    def _validate_item(self, item: int) -> None:
        if type(item) is not int:
            raise TypeError("item must be an exact integer")
        if not 0 <= item < self._capacity:
            raise ValueError("item is outside the queue capacity")

    def _entry(self, item: int) -> tuple[int, int]:
        priority = self._priorities[item]
        if priority is None:
            raise AssertionError("every heap item must have a priority")
        return priority, item

    def _swap(self, first: int, second: int) -> None:
        first_item = self._heap[first]
        second_item = self._heap[second]
        self._heap[first], self._heap[second] = second_item, first_item
        self._positions[first_item] = second
        self._positions[second_item] = first

    def _sift_up(self, position: int) -> None:
        while position:
            parent = (position - 1) // 2
            if self._entry(self._heap[parent]) <= self._entry(self._heap[position]):
                return
            self._swap(parent, position)
            position = parent

    def _sift_down(self, position: int) -> None:
        size = len(self._heap)
        while True:
            left = 2 * position + 1
            if left >= size:
                return
            right = left + 1
            smallest = left
            if right < size and self._entry(self._heap[right]) < self._entry(self._heap[left]):
                smallest = right
            if self._entry(self._heap[position]) <= self._entry(self._heap[smallest]):
                return
            self._swap(position, smallest)
            position = smallest

    def set_priority(self, item: int, priority: int) -> None:
        """Insert item or replace its current priority in place."""
        self._validate_item(item)
        if type(priority) is not int:
            raise TypeError("priority must be an exact integer")
        if not _MIN_SIGNED_64 <= priority <= _MAX_SIGNED_64:
            raise ValueError("priority is outside the signed 64-bit range")

        position = self._positions[item]
        if position == -1:
            self._priorities[item] = priority
            self._positions[item] = len(self._heap)
            self._heap.append(item)
            self._sift_up(len(self._heap) - 1)
            return

        previous = self._priorities[item]
        if previous is None:
            raise AssertionError("a positioned item must have a priority")
        self._priorities[item] = priority
        if priority < previous:
            self._sift_up(position)
        elif priority > previous:
            self._sift_down(position)

    def _remove_at(self, position: int) -> None:
        removed_item = self._heap[position]
        last_item = self._heap.pop()
        self._positions[removed_item] = -1
        self._priorities[removed_item] = None
        if position == len(self._heap):
            return

        self._heap[position] = last_item
        self._positions[last_item] = position
        parent = (position - 1) // 2
        if position and self._entry(last_item) < self._entry(self._heap[parent]):
            self._sift_up(position)
        else:
            self._sift_down(position)

    def discard(self, item: int) -> bool:
        """Remove item if present and report whether it was live."""
        self._validate_item(item)
        position = self._positions[item]
        if position == -1:
            return False
        self._remove_at(position)
        return True

    def peek_min(self) -> PriorityItem:
        """Return the minimum live item without removing it."""
        if not self._heap:
            raise IndexError("peek from an empty indexed priority queue")
        item = self._heap[0]
        priority = self._priorities[item]
        if priority is None:
            raise AssertionError("the heap root must have a priority")
        return PriorityItem(item=item, priority=priority)

    def pop_min(self) -> PriorityItem:
        """Remove and return the minimum live item."""
        minimum = self.peek_min()
        self._remove_at(0)
        return minimum
```

## Example

```python
from random import Random


def assert_raises(error_type, operation) -> None:
    try:
        operation()
    except error_type:
        return
    raise AssertionError(f"{error_type.__name__} was not raised")


def assert_matches(
    queue: IndexedMinPriorityQueue,
    reference: dict[int, int],
) -> None:
    assert len(queue) == len(reference)
    if reference:
        expected_priority, expected_item = min(
            (priority, item) for item, priority in reference.items()
        )
        assert queue.peek_min() == PriorityItem(expected_item, expected_priority)
    else:
        assert_raises(IndexError, queue.peek_min)

    # These representation checks prove that lazy stale copies never remain.
    assert len(queue._heap) == len(reference)
    assert set(queue._heap) == set(reference)
    for position, item in enumerate(queue._heap):
        assert queue._positions[item] == position
        assert queue._priorities[item] == reference[item]
        if position:
            parent = (position - 1) // 2
            parent_item = queue._heap[parent]
            assert (reference[parent_item], parent_item) <= (reference[item], item)
    for item in range(queue._capacity):
        if item not in reference:
            assert queue._positions[item] == -1
            assert queue._priorities[item] is None


generator = Random(0x1D3E_185)
queue = IndexedMinPriorityQueue(8)
reference: dict[int, int] = {}
random_steps = 20_000
for _ in range(random_steps):
    operation = generator.randrange(5)
    item = generator.randrange(8)
    priority = generator.randrange(-20, 21)
    if operation <= 1:
        queue.set_priority(item, priority)
        reference[item] = priority
    elif operation == 2:
        assert queue.discard(item) == (reference.pop(item, None) is not None)
    elif operation == 3:
        if reference:
            expected_priority, expected_item = min((value, key) for key, value in reference.items())
            assert queue.pop_min() == PriorityItem(expected_item, expected_priority)
            del reference[expected_item]
        else:
            assert_raises(IndexError, queue.pop_min)
    elif reference:
        expected_priority, expected_item = min((value, key) for key, value in reference.items())
        assert queue.peek_min() == PriorityItem(expected_item, expected_priority)
    else:
        assert_raises(IndexError, queue.peek_min)
    assert_matches(queue, reference)

maximum_queue = IndexedMinPriorityQueue(_MAX_INDEXED_QUEUE_CAPACITY)
maximum_reference = {item: item for item in range(_MAX_INDEXED_QUEUE_CAPACITY)}
for item, priority in maximum_reference.items():
    maximum_queue.set_priority(item, priority)

# Exercise an increase at the old root and a decrease at the old last item.
maximum_reference[0] = _MAX_SIGNED_64
maximum_queue.set_priority(0, _MAX_SIGNED_64)
last_item = _MAX_INDEXED_QUEUE_CAPACITY - 1
maximum_reference[last_item] = _MIN_SIGNED_64
maximum_queue.set_priority(last_item, _MIN_SIGNED_64)
expected_drain = tuple(
    PriorityItem(item, priority)
    for priority, item in sorted((priority, item) for item, priority in maximum_reference.items())
)
actual_drain = tuple(maximum_queue.pop_min() for _ in range(len(maximum_queue)))

validation_queue = IndexedMinPriorityQueue(2)
rejected = 0
for operation, expected_error in (
    (lambda: IndexedMinPriorityQueue(True), TypeError),
    (lambda: IndexedMinPriorityQueue(0), ValueError),
    (lambda: IndexedMinPriorityQueue(4_097), ValueError),
    (lambda: validation_queue.set_priority(True, 0), TypeError),
    (lambda: validation_queue.set_priority(2, 0), ValueError),
    (lambda: validation_queue.set_priority(0, True), TypeError),
    (lambda: validation_queue.set_priority(0, _MIN_SIGNED_64 - 1), ValueError),
    (lambda: validation_queue.set_priority(0, _MAX_SIGNED_64 + 1), ValueError),
    (lambda: validation_queue.discard(-1), ValueError),
):
    try:
        operation()
    except expected_error:
        rejected += 1
    else:
        raise AssertionError("invalid input must be rejected")

assert random_steps == 20_000 and actual_drain == expected_drain and rejected == 9
```

## Trade-offs and Limitations

The queue preallocates two capacity-sized tables, so it uses `O(capacity)`
memory even when nearly empty. Its heap stores exactly one integer per live
item. Insert, priority change, discard, and pop take `O(log n)` time for `n`
live items; peek and length take `O(1)` time.

Capacity, item IDs, and priorities are deliberately closed contracts. Item IDs
must be exact integers in `0..capacity-1`, priorities must be exact signed-64
integers, and `bool` is rejected. `set_priority` is an upsert operation, while
discarding an absent valid item is a reported no-op. Empty minimum operations
raise `IndexError`.

The implementation is mutable and not thread-safe. It does not accept arbitrary
hashable objects, caller-defined comparison keys, duplicate live entries,
unbounded growth, iteration in priority order, heap melding, persistence, or
decrease-only handles.

## Related Snippets

<!-- catalog:related:start -->
- [Find One Deterministic Shortest Path in a Bounded Non-Negatively Weighted Directed Graph](find-one-deterministic-shortest-path-in-a-bounded-non-negatively-weighted-directed-graph.md)
- [Maintain a Bounded Integer Multiset with Fenwick Rank and Select](maintain-a-bounded-integer-multiset-with-fenwick-rank-and-select.md)
- [Select the Stable K Smallest Bounded Records with a Max-Heap](select-the-stable-k-smallest-bounded-records-with-a-max-heap.md)
<!-- catalog:related:end -->
