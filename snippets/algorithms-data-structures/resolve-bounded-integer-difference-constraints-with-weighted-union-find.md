---
title: "Resolve Bounded Integer Difference Constraints with Weighted Union-Find"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - partition-a-bounded-undirected-graph-into-deterministic-components-with-union-find.md
  - answer-bounded-offline-dynamic-connectivity-queries-with-rollback-disjoint-sets.md
  - compute-bounded-bellman-ford-distances-and-detect-reachable-negative-cycles.md
---

# Resolve Bounded Integer Difference Constraints with Weighted Union-Find

## Idea and Problem

Maintain exact integer potential-difference equations and answer relative-potential queries without choosing absolute component origins.

A weighted union-find stores each item's potential relative to its parent. Finding a
root accumulates those offsets and compresses the path without changing any equation.
When an observation joins two components, the observed difference determines the new
root-to-root offset. An observation inside one component instead checks the already
implied difference and rejects contradictory input.

## When to Use

Use this for a bounded, append-only batch of exact integer difference observations
followed by relative-potential queries. Examples include reconciling coordinate
offsets, sequence positions, counter deltas, or unit conversions whose scale is
already fixed and whose relations are equalities.

Use graph traversal when you also need paths or witnesses. Use a general constraint
solver when relations include inequalities, ranges, choices, changing observations,
or non-additive transformations. Use a rollback or dynamic-connectivity structure
when observations must be removed or queries must be interleaved with historical
snapshots.

## Implementation

```python
_MAX_ITEM_COUNT = 65_536
_MAX_OBSERVATION_COUNT = 131_072
_MAX_QUERY_COUNT = 131_072
_MIN_DECLARED_DIFFERENCE = -(1 << 63)
_MAX_DECLARED_DIFFERENCE = (1 << 63) - 1


def _difference_endpoint(value: object, item_count: int, field: str) -> int:
    if type(value) is not int:
        raise TypeError(f"{field} must be an exact int")
    if not 0 <= value < item_count:
        raise ValueError(f"{field} is outside the item range")
    return value


def resolve_potential_differences(
    item_count: int,
    observations: tuple[tuple[int, int, int], ...],
    queries: tuple[tuple[int, int], ...],
) -> tuple[int | None, ...]:
    """Validate difference observations and answer the requested differences."""
    if type(item_count) is not int:
        raise TypeError("item_count must be an exact int")
    if not 0 <= item_count <= _MAX_ITEM_COUNT:
        raise ValueError("item_count is outside 0..65,536")
    if type(observations) is not tuple:
        raise TypeError("observations must be an exact tuple")
    if len(observations) > _MAX_OBSERVATION_COUNT:
        raise ValueError("observations exceed 131,072 items")
    if type(queries) is not tuple:
        raise TypeError("queries must be an exact tuple")
    if len(queries) > _MAX_QUERY_COUNT:
        raise ValueError("queries exceed 131,072 items")

    parent = list(range(item_count))
    size = [1] * item_count
    offset_to_parent = [0] * item_count

    def find(item: int) -> tuple[int, int]:
        current = item
        path: list[tuple[int, int]] = []
        total_offset = 0
        while parent[current] != current:
            edge_offset = offset_to_parent[current]
            path.append((current, edge_offset))
            total_offset += edge_offset
            current = parent[current]

        remaining_offset = total_offset
        for path_item, edge_offset in path:
            parent[path_item] = current
            offset_to_parent[path_item] = remaining_offset
            remaining_offset -= edge_offset
        return current, total_offset

    for index, observation in enumerate(observations):
        if type(observation) is not tuple:
            raise TypeError(f"observations[{index}] must be an exact tuple")
        if len(observation) != 3:
            raise ValueError(f"observations[{index}] must contain three items")

        left = _difference_endpoint(observation[0], item_count, f"observations[{index}][0]")
        right = _difference_endpoint(observation[1], item_count, f"observations[{index}][1]")
        difference = observation[2]
        if type(difference) is not int:
            raise TypeError(f"observations[{index}][2] must be an exact int")
        if not _MIN_DECLARED_DIFFERENCE <= difference <= _MAX_DECLARED_DIFFERENCE:
            raise ValueError(f"observations[{index}][2] is outside signed 64-bit range")

        left_root, left_offset = find(left)
        right_root, right_offset = find(right)
        if left_root == right_root:
            if right_offset - left_offset != difference:
                raise ValueError(f"observations[{index}] contradicts earlier observations")
            continue

        root_offset = difference + left_offset - right_offset
        if size[left_root] >= size[right_root]:
            parent[right_root] = left_root
            offset_to_parent[right_root] = root_offset
            size[left_root] += size[right_root]
        else:
            parent[left_root] = right_root
            offset_to_parent[left_root] = -root_offset
            size[right_root] += size[left_root]

    answers: list[int | None] = []
    for index, query in enumerate(queries):
        if type(query) is not tuple:
            raise TypeError(f"queries[{index}] must be an exact tuple")
        if len(query) != 2:
            raise ValueError(f"queries[{index}] must contain two items")
        left = _difference_endpoint(query[0], item_count, f"queries[{index}][0]")
        right = _difference_endpoint(query[1], item_count, f"queries[{index}][1]")
        left_root, left_offset = find(left)
        right_root, right_offset = find(right)
        answers.append(right_offset - left_offset if left_root == right_root else None)

    return tuple(answers)
```

## Example

```python
from collections import deque
from itertools import product
from random import Random


def graph_oracle(
    item_count: int,
    observations: tuple[tuple[int, int, int], ...],
    queries: tuple[tuple[int, int], ...],
) -> tuple[int | None, ...]:
    adjacency: list[list[tuple[int, int]]] = [[] for _ in range(item_count)]
    for left, right, difference in observations:
        adjacency[left].append((right, difference))
        adjacency[right].append((left, -difference))

    component = [-1] * item_count
    potential = [0] * item_count
    for start in range(item_count):
        if component[start] != -1:
            continue
        component[start] = start
        pending = deque((start,))
        while pending:
            item = pending.popleft()
            for neighbor, difference in adjacency[item]:
                candidate = potential[item] + difference
                if component[neighbor] == -1:
                    component[neighbor] = start
                    potential[neighbor] = candidate
                    pending.append(neighbor)
                elif potential[neighbor] != candidate:
                    raise ValueError("contradictory graph")

    return tuple(
        potential[right] - potential[left] if component[left] == component[right] else None
        for left, right in queries
    )


tiny_checked = 0
for item_count in range(4):
    edges = tuple(
        (left, right) for left in range(item_count) for right in range(left + 1, item_count)
    )
    queries = tuple((left, right) for left in range(item_count) for right in range(item_count))
    for potentials in product((-1, 0, 1), repeat=item_count):
        for edge_mask in range(1 << len(edges)):
            observations = tuple(
                (left, right, potentials[right] - potentials[left])
                for edge_index, (left, right) in enumerate(edges)
                if edge_mask & (1 << edge_index)
            )
            expected = graph_oracle(item_count, observations, queries)
            reordered = tuple(
                (right, left, -difference) for left, right, difference in reversed(observations)
            )
            assert (
                resolve_potential_differences(item_count, observations, queries)
                == resolve_potential_differences(item_count, reordered, queries)
                == expected
            )
            tiny_checked += 1

rng = Random(0)
random_checked = 0
for _ in range(300):
    item_count = rng.randrange(33)
    potentials = [rng.randrange(-1_000_000, 1_000_001) for _ in range(item_count)]
    observations_list: list[tuple[int, int, int]] = []
    for _ in range(rng.randrange(3 * item_count + 1)):
        left = rng.randrange(item_count)
        right = rng.randrange(item_count)
        observations_list.append((left, right, potentials[right] - potentials[left]))
    observations = tuple(observations_list)
    queries = tuple(
        (rng.randrange(item_count), rng.randrange(item_count)) for _ in range(2 * item_count)
    )
    shuffled = list(observations)
    rng.shuffle(shuffled)
    expected = graph_oracle(item_count, observations, queries)
    assert (
        resolve_potential_differences(item_count, observations, queries)
        == resolve_potential_differences(item_count, tuple(shuffled), queries)
        == expected
    )
    random_checked += 1

duplicates_and_reverse = ((0, 1, 7), (0, 1, 7), (1, 0, -7))
disconnected_queries = ((0, 1), (1, 0), (0, 2), (2, 2))
duplicate_answers = resolve_potential_differences(3, duplicates_and_reverse, disconnected_queries)

contradictions = (
    ((0, 1, 3), (1, 2, 4), (0, 2, 8)),
    ((0, 0, 1),),
    ((0, 1, 7), (1, 0, -8)),
)
contradictions_rejected = 0
for observations in contradictions:
    try:
        resolve_potential_differences(3, observations, ())
    except ValueError:
        contradictions_rejected += 1

chain_size = _MAX_ITEM_COUNT
long_chain = tuple((item, item + 1, 1) for item in range(chain_size - 1))
chain_answers = resolve_potential_differences(
    chain_size,
    long_chain,
    ((0, chain_size - 1), (chain_size - 1, 0), (42, 42)),
)

maximum_observations = ((0, 0, 0),) * _MAX_OBSERVATION_COUNT
maximum_queries = ((0, 0),) * _MAX_QUERY_COUNT
maximum_answers = resolve_potential_differences(1, maximum_observations, maximum_queries)

invalid_calls = (
    (True, (), ()),
    (-1, (), ()),
    (_MAX_ITEM_COUNT + 1, (), ()),
    (1, [], ()),
    (1, (), []),
    (1, ([0, 0, 0],), ()),
    (1, ((0, 0),), ()),
    (1, ((0, 0, True),), ()),
    (1, ((-1, 0, 0),), ()),
    (1, ((0, 1, 0),), ()),
    (1, ((0, 0, _MIN_DECLARED_DIFFERENCE - 1),), ()),
    (1, ((0, 0, _MAX_DECLARED_DIFFERENCE + 1),), ()),
    (1, (), ([0, 0],)),
    (1, (), ((0,),)),
    (1, (), ((True, 0),)),
    (1, ((0, 0, 0),) * (_MAX_OBSERVATION_COUNT + 1), ()),
    (1, (), ((0, 0),) * (_MAX_QUERY_COUNT + 1)),
)
invalid_rejected = 0
for item_count, observations, queries in invalid_calls:
    try:
        resolve_potential_differences(
            item_count,
            observations,
            queries,  # type: ignore[arg-type]
        )
    except (TypeError, ValueError):
        invalid_rejected += 1

assert (
    tiny_checked == 238
    and random_checked == 300
    and duplicate_answers == (7, -7, None, 0)
    and contradictions_rejected == len(contradictions)
    and chain_answers == (chain_size - 1, 1 - chain_size, 0)
    and len(maximum_answers) == _MAX_QUERY_COUNT
    and maximum_answers[0] == maximum_answers[-1] == 0
    and invalid_rejected == len(invalid_calls)
)
```

## Trade-offs and Limitations

With union by size and path compression, processing `N` items, `O`
observations, and `Q` queries takes `O((N + O + Q) * alpha(N))` amortized time,
where `alpha` is the inverse Ackermann function. The union-find arrays use `O(N)`
auxiliary space; the returned tuple necessarily uses `O(Q)` additional space.

Only each declared observation is restricted to signed 64-bit range. Python integers
accumulate exact root offsets, so a valid chain and its returned difference may exceed
that range. Absolute potentials remain intentionally undetermined within each connected
component, and queries across components return `None`.

The complete batch is validated and processed in declaration order. Repeated, reversed,
and self observations are accepted when consistent; the first contradiction raises
`ValueError` without an explanatory cycle witness. The function does not support
floating-point tolerances, modular differences, inequality constraints, observation
removal, rollback, or concurrent mutation.

## Related Snippets

<!-- catalog:related:start -->
- [Partition a Bounded Undirected Graph into Deterministic Components with Union-Find](partition-a-bounded-undirected-graph-into-deterministic-components-with-union-find.md)
- [Answer Bounded Offline Dynamic Connectivity Queries with Rollback Disjoint Sets](answer-bounded-offline-dynamic-connectivity-queries-with-rollback-disjoint-sets.md)
- [Compute Bounded Bellman-Ford Distances and Detect Reachable Negative Cycles](compute-bounded-bellman-ford-distances-and-detect-reachable-negative-cycles.md)
<!-- catalog:related:end -->
