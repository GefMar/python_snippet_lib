---
title: "Maintain Bounded Disjoint Sets with Rollback Snapshots"
snippet_type: algorithm
use_cases:
  - data-transformation
  - resource-management
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - partition-a-bounded-undirected-graph-into-deterministic-components-with-union-find.md
  - build-a-deterministic-minimum-spanning-forest-with-kruskals-algorithm.md
---

# Maintain Bounded Disjoint Sets with Rollback Snapshots

## Idea and Problem

Maintain connectivity among indexed elements while allowing successful unions to be rolled back to an earlier owned snapshot.

Union by size keeps every parent chain logarithmically shallow without path
compression. Each successful merge changes exactly one parent and one root
size, so recording those old values is enough to undo it. Redundant unions
change no state and consume no history entry.

A frozen snapshot records its owning structure, history depth, and the unique
state identifier at that depth. This rejects a token from another instance and
a stale token from a future branch that was rolled back and replaced.

## When to Use

Use this structure in bounded backtracking or offline algorithms that add
temporary undirected connections, explore a branch, and then restore an exact
ancestor state. Connectivity, representative, and component-size queries stay
available between snapshots and rollbacks.

Use ordinary union-find with path compression when unions only move forward;
it has a better amortized bound and less bookkeeping. Choose a persistent data
structure when several historical branches must remain queryable at once. A
rollback snapshot is an ancestor mark, not an immutable version or redo log.

## Implementation

```python
from dataclasses import dataclass, field

_MAX_ROLLBACK_VERTEX_COUNT = 100_000


@dataclass(frozen=True, slots=True)
class RollbackSnapshot:
    depth: int
    _owner: object = field(repr=False)
    _state_id: int = field(repr=False)


class RollbackDisjointSets:
    """Own union-by-size disjoint sets with ancestor-state rollback."""

    __slots__ = (
        "_history",
        "_next_state_id",
        "_owner",
        "_parents",
        "_sizes",
        "_state_ids",
    )

    def __init__(self, vertex_count: int) -> None:
        if type(vertex_count) is not int:
            raise TypeError("vertex_count must be an exact integer")
        if not 0 <= vertex_count <= _MAX_ROLLBACK_VERTEX_COUNT:
            raise ValueError("vertex_count is outside 0..100000")

        self._parents = list(range(vertex_count))
        self._sizes = [1] * vertex_count
        self._history: list[tuple[int, int, int]] = []
        self._state_ids = [0]
        self._next_state_id = 1
        self._owner = object()

    def _validated_vertex(self, vertex: int) -> int:
        if type(vertex) is not int:
            raise TypeError("vertex must be an exact integer")
        if not 0 <= vertex < len(self._parents):
            raise ValueError("vertex is outside the structure")
        return vertex

    def find(self, vertex: int) -> int:
        """Return the current representative without path compression."""
        current = self._validated_vertex(vertex)
        while self._parents[current] != current:
            current = self._parents[current]
        return current

    def connected(self, first: int, second: int) -> bool:
        """Return whether two validated vertices share a representative."""
        return self.find(first) == self.find(second)

    def component_size(self, vertex: int) -> int:
        """Return the number of vertices in one current component."""
        return self._sizes[self.find(vertex)]

    def union(self, first: int, second: int) -> bool:
        """Merge two components and report whether state changed."""
        first_root = self.find(first)
        second_root = self.find(second)
        if first_root == second_root:
            return False

        if self._sizes[first_root] < self._sizes[second_root] or (
            self._sizes[first_root] == self._sizes[second_root]
            and first_root > second_root
        ):
            first_root, second_root = second_root, first_root

        self._history.append(
            (second_root, first_root, self._sizes[first_root])
        )
        self._parents[second_root] = first_root
        self._sizes[first_root] += self._sizes[second_root]
        self._state_ids.append(self._next_state_id)
        self._next_state_id += 1
        return True

    def snapshot(self) -> RollbackSnapshot:
        """Return an opaque token for the exact current history prefix."""
        return RollbackSnapshot(
            depth=len(self._history),
            _owner=self._owner,
            _state_id=self._state_ids[-1],
        )

    def rollback(self, snapshot: RollbackSnapshot) -> None:
        """Restore an ancestor snapshot on this instance's current branch."""
        if type(snapshot) is not RollbackSnapshot:
            raise TypeError("snapshot must be an exact RollbackSnapshot")
        if snapshot._owner is not self._owner:
            raise ValueError("snapshot belongs to another structure")
        if not 0 <= snapshot.depth <= len(self._history):
            raise ValueError("snapshot is not an ancestor of the current state")
        if self._state_ids[snapshot.depth] != snapshot._state_id:
            raise ValueError("snapshot belongs to a replaced history branch")

        while len(self._history) > snapshot.depth:
            child_root, parent_root, old_parent_size = self._history.pop()
            self._parents[child_root] = child_root
            self._sizes[parent_root] = old_parent_size
            self._state_ids.pop()
```

## Example

```python
def observed_partition(
    disjoint_sets: RollbackDisjointSets,
    vertex_count: int,
) -> tuple[tuple[int, ...], ...]:
    groups: dict[int, list[int]] = {}
    for vertex in range(vertex_count):
        groups.setdefault(disjoint_sets.find(vertex), []).append(vertex)
    return tuple(sorted(tuple(group) for group in groups.values()))


def merge_partition(
    partition: tuple[tuple[int, ...], ...],
    first: int,
    second: int,
) -> tuple[tuple[int, ...], ...]:
    first_group = next(group for group in partition if first in group)
    second_group = next(group for group in partition if second in group)
    if first_group == second_group:
        return partition
    merged = tuple(sorted((*first_group, *second_group)))
    remaining = [
        group for group in partition if group not in (first_group, second_group)
    ]
    return tuple(sorted((*remaining, merged)))


def exercise_branching_sequences(
    disjoint_sets: RollbackDisjointSets,
    expected: tuple[tuple[int, ...], ...],
    depth: int,
) -> int:
    assert observed_partition(disjoint_sets, 4) == expected
    if depth == 0:
        return 1

    checked = 1
    base = disjoint_sets.snapshot()
    for first in range(4):
        for second in range(first, 4):
            next_expected = merge_partition(expected, first, second)
            changed = disjoint_sets.union(first, second)
            assert changed is (next_expected != expected)
            checked += exercise_branching_sequences(
                disjoint_sets,
                next_expected,
                depth - 1,
            )
            disjoint_sets.rollback(base)
    return checked


small = RollbackDisjointSets(4)
checked_states = exercise_branching_sequences(
    small,
    ((0,), (1,), (2,), (3,)),
    3,
)

branched = RollbackDisjointSets(4)
root_snapshot = branched.snapshot()
assert branched.union(0, 1)
abandoned_snapshot = branched.snapshot()
branched.rollback(root_snapshot)
assert branched.union(2, 3)

stale_rejected = False
try:
    branched.rollback(abandoned_snapshot)
except ValueError:
    stale_rejected = True

foreign_rejected = False
try:
    RollbackDisjointSets(4).rollback(branched.snapshot())
except ValueError:
    foreign_rejected = True

boundary = RollbackDisjointSets(_MAX_ROLLBACK_VERTEX_COUNT)
boundary_snapshot = boundary.snapshot()
assert boundary.union(0, _MAX_ROLLBACK_VERTEX_COUNT - 1)
boundary.rollback(boundary_snapshot)

assert (
    checked_states == 1_111
    and stale_rejected
    and foreign_rejected
    and not branched.connected(0, 1)
    and branched.connected(2, 3)
    and branched.component_size(2) == 2
    and boundary.component_size(0) == 1
)
```

## Trade-offs and Limitations

Union by size bounds tree height by `O(log V)`, so `find`, `connected`,
`component_size`, and `union` take `O(log V)` worst-case time. Creating a
snapshot is `O(1)`. Rolling back takes `O(K)` time for `K` successful merges
undone. Parent, size, current history, and state identifiers use `O(V)` space;
the monotonically increasing Python state counter is not fixed-width.

Path compression is intentionally absent because its many parent rewrites
would need extra history. Equal-size roots use the smaller root index as the
parent, making representatives deterministic for a fixed operation sequence.
Redundant unions leave both history depth and state identity unchanged.

Snapshots are owner-bound marks into the current history lineage. A token from
another instance, a deeper state, or a branch replaced after rollback is
rejected. A rollback destroys later state; there is no redo, branching query,
serialization, deletion, persistence, transaction isolation, or thread safety.

## Related Snippets

<!-- catalog:related:start -->
- [Partition a Bounded Undirected Graph into Deterministic Components with Union-Find](partition-a-bounded-undirected-graph-into-deterministic-components-with-union-find.md)
- [Build a Deterministic Minimum Spanning Forest with Kruskal's Algorithm](build-a-deterministic-minimum-spanning-forest-with-kruskals-algorithm.md)
<!-- catalog:related:end -->
