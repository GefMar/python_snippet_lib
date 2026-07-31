---
title: "Send a Required Flow at Minimum Cost with a Residual Certificate"
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
  - compute-a-bounded-integer-maximum-flow-and-canonical-minimum-cut-with-edmonds-karp.md
  - find-a-deterministic-minimum-cost-perfect-assignment-and-dual-certificate-with-the-hungarian-algorithm.md
  - compute-bounded-bellman-ford-distances-and-detect-reachable-negative-cycles.md
---

# Send a Required Flow at Minimum Cost with a Residual Certificate

## Idea and Problem

Send one exact amount through a bounded directed capacity network at minimum total cost, or return a cut proving that the requested amount is impossible.

Successive shortest augmenting paths grow an integral flow. Every original arc
owns a distinct forward and reverse residual arc, so cancellation remains
correct even when the input also contains an antiparallel arc. Internal
traversal follows canonical endpoint order, while returned flow values retain
the caller's arc order.

A feasible result includes vertex potentials. Non-negative reduced cost on
every positive-capacity residual arc proves that the residual graph has no
cost-improving cycle, which certifies minimum cost for the fixed flow value. If
augmentation becomes impossible, residual reachability supplies a source-side
cut whose capacity equals the achieved flow and proves infeasibility.

## When to Use

Use this algorithm for a small, fully materialized network with integral
capacities, non-negative per-unit costs, one source, one sink, and an exact
required amount. It is useful when the chosen edge flows must be retained and
the result needs a compact certificate that can be checked without rerunning
the optimizer.

Use ordinary maximum flow when costs do not matter. Use assignment-specific
algorithms for complete one-to-one matrices. Choose a specialized optimization
library for large networks, negative original costs, lower bounds, supplies,
several commodities, floating-point objectives, or repeated updates.

## Implementation

```python
from dataclasses import dataclass

_MAX_FLOW_VERTICES = 64
_MAX_FLOW_ARCS = 512
_MAX_ARC_VALUE = (1 << 31) - 1
_MAX_REQUIRED_FLOW = 4_096


class RequiredFlowInputError(ValueError):
    """Raised when a network violates the bounded flow contract."""


@dataclass(frozen=True, slots=True, order=True)
class CostedCapacityArc:
    source: int
    target: int
    capacity: int
    unit_cost: int


@dataclass(frozen=True, slots=True)
class FeasibleRequiredFlow:
    flow_value: int
    total_cost: int
    arc_flows: tuple[int, ...]
    vertex_potentials: tuple[int, ...]


@dataclass(frozen=True, slots=True)
class InfeasibleRequiredFlow:
    achieved_flow: int
    arc_flows: tuple[int, ...]
    source_side: tuple[int, ...]


@dataclass(slots=True)
class _ResidualArc:
    source: int
    target: int
    cost: int
    capacity: int
    twin: int
    original_index: int
    direction: int


def _validate_required_flow_input(
    vertex_count: int,
    arcs: tuple[CostedCapacityArc, ...],
    source: int,
    sink: int,
    required_flow: int,
) -> tuple[int, ...]:
    if type(vertex_count) is not int:
        raise TypeError("vertex_count must be an exact integer")
    if not 2 <= vertex_count <= _MAX_FLOW_VERTICES:
        raise RequiredFlowInputError("vertex_count is outside the supported range")
    if type(arcs) is not tuple:
        raise TypeError("arcs must be an exact tuple")
    if len(arcs) > _MAX_FLOW_ARCS:
        raise RequiredFlowInputError("arc count exceeds the supported limit")
    if type(source) is not int or type(sink) is not int:
        raise TypeError("source and sink must be exact integers")
    if not 0 <= source < vertex_count or not 0 <= sink < vertex_count:
        raise RequiredFlowInputError("source or sink is outside the graph")
    if source == sink:
        raise RequiredFlowInputError("source and sink must be distinct")
    if type(required_flow) is not int:
        raise TypeError("required_flow must be an exact integer")
    if not 0 <= required_flow <= _MAX_REQUIRED_FLOW:
        raise RequiredFlowInputError("required_flow is outside the supported range")

    seen_pairs: set[tuple[int, int]] = set()
    for arc_index, arc in enumerate(arcs):
        if type(arc) is not CostedCapacityArc:
            raise TypeError(f"arcs[{arc_index}] must be an exact CostedCapacityArc")
        if type(arc.source) is not int or type(arc.target) is not int:
            raise TypeError(f"arcs[{arc_index}] endpoints must be exact integers")
        if not 0 <= arc.source < vertex_count or not 0 <= arc.target < vertex_count:
            raise RequiredFlowInputError(f"arcs[{arc_index}] endpoint is outside the graph")
        if arc.source == arc.target:
            raise RequiredFlowInputError(f"arcs[{arc_index}] is a self-arc")
        if type(arc.capacity) is not int:
            raise TypeError(f"arcs[{arc_index}].capacity must be an exact integer")
        if not 1 <= arc.capacity <= _MAX_ARC_VALUE:
            raise RequiredFlowInputError(f"arcs[{arc_index}].capacity is outside range")
        if type(arc.unit_cost) is not int:
            raise TypeError(f"arcs[{arc_index}].unit_cost must be an exact integer")
        if not 0 <= arc.unit_cost <= _MAX_ARC_VALUE:
            raise RequiredFlowInputError(f"arcs[{arc_index}].unit_cost is outside range")
        pair = (arc.source, arc.target)
        if pair in seen_pairs:
            raise RequiredFlowInputError(f"arcs[{arc_index}] duplicates a directed pair")
        seen_pairs.add(pair)

    return tuple(
        sorted(
            range(len(arcs)),
            key=lambda index: (arcs[index].source, arcs[index].target),
        )
    )


def _build_residual_graph(
    vertex_count: int,
    arcs: tuple[CostedCapacityArc, ...],
    canonical_indices: tuple[int, ...],
) -> tuple[list[_ResidualArc], tuple[tuple[int, ...], ...]]:
    residual: list[_ResidualArc] = []
    outgoing: list[list[int]] = [[] for _ in range(vertex_count)]

    for original_index in canonical_indices:
        original = arcs[original_index]
        forward_index = len(residual)
        reverse_index = forward_index + 1
        residual.append(
            _ResidualArc(
                source=original.source,
                target=original.target,
                cost=original.unit_cost,
                capacity=original.capacity,
                twin=reverse_index,
                original_index=original_index,
                direction=1,
            )
        )
        residual.append(
            _ResidualArc(
                source=original.target,
                target=original.source,
                cost=-original.unit_cost,
                capacity=0,
                twin=forward_index,
                original_index=original_index,
                direction=-1,
            )
        )
        outgoing[original.source].append(forward_index)
        outgoing[original.target].append(reverse_index)

    for edge_indices in outgoing:
        edge_indices.sort(
            key=lambda index: (
                residual[index].target,
                residual[index].cost,
                0 if residual[index].direction == 1 else 1,
            )
        )
    return residual, tuple(tuple(edge_indices) for edge_indices in outgoing)


def _shortest_residual_path(
    vertex_count: int,
    residual: list[_ResidualArc],
    outgoing: tuple[tuple[int, ...], ...],
    source: int,
    sink: int,
) -> tuple[int, ...] | None:
    distances: list[int | None] = [None] * vertex_count
    predecessors = [-1] * vertex_count
    distances[source] = 0

    for _ in range(vertex_count - 1):
        changed = False
        for vertex in range(vertex_count):
            distance = distances[vertex]
            if distance is None:
                continue
            for residual_index in outgoing[vertex]:
                edge = residual[residual_index]
                if edge.capacity <= 0:
                    continue
                candidate = distance + edge.cost
                current = distances[edge.target]
                if current is None or candidate < current:
                    distances[edge.target] = candidate
                    predecessors[edge.target] = residual_index
                    changed = True
        if not changed:
            break

    for vertex in range(vertex_count):
        distance = distances[vertex]
        if distance is None:
            continue
        for residual_index in outgoing[vertex]:
            edge = residual[residual_index]
            target_distance = distances[edge.target]
            if (
                edge.capacity > 0
                and target_distance is not None
                and distance + edge.cost < target_distance
            ):
                raise RuntimeError("unexpected negative residual cycle")

    if distances[sink] is None:
        return None

    reverse_path: list[int] = []
    seen_vertices: set[int] = set()
    cursor = sink
    while cursor != source:
        if cursor in seen_vertices:
            raise RuntimeError("residual predecessor chain contains a cycle")
        seen_vertices.add(cursor)
        residual_index = predecessors[cursor]
        if residual_index < 0:
            raise RuntimeError("reachable sink has no residual predecessor")
        reverse_path.append(residual_index)
        cursor = residual[residual_index].source
    reverse_path.reverse()
    return tuple(reverse_path)


def _global_residual_potentials(
    vertex_count: int,
    residual: list[_ResidualArc],
    outgoing: tuple[tuple[int, ...], ...],
    source: int,
) -> tuple[int, ...]:
    distances = [0] * vertex_count
    for pass_index in range(vertex_count):
        changed = False
        for vertex in range(vertex_count):
            distance = distances[vertex]
            for residual_index in outgoing[vertex]:
                edge = residual[residual_index]
                candidate = distance + edge.cost
                if edge.capacity > 0 and candidate < distances[edge.target]:
                    distances[edge.target] = candidate
                    changed = True
        if not changed:
            break
        if pass_index == vertex_count - 1:
            raise RuntimeError("unexpected negative residual cycle")

    shift = distances[source]
    return tuple(distance - shift for distance in distances)


def _residual_source_side(
    residual: list[_ResidualArc],
    outgoing: tuple[tuple[int, ...], ...],
    source: int,
) -> tuple[int, ...]:
    reachable = [False] * len(outgoing)
    reachable[source] = True
    stack = [source]
    while stack:
        vertex = stack.pop()
        for residual_index in outgoing[vertex]:
            edge = residual[residual_index]
            if edge.capacity > 0 and not reachable[edge.target]:
                reachable[edge.target] = True
                stack.append(edge.target)
    return tuple(vertex for vertex, included in enumerate(reachable) if included)


def minimum_cost_required_flow(
    vertex_count: int,
    arcs: tuple[CostedCapacityArc, ...],
    source: int,
    sink: int,
    required_flow: int,
) -> FeasibleRequiredFlow | InfeasibleRequiredFlow:
    """Return a certified minimum-cost required flow or a maximum-flow cut."""
    canonical_indices = _validate_required_flow_input(
        vertex_count,
        arcs,
        source,
        sink,
        required_flow,
    )
    residual, outgoing = _build_residual_graph(vertex_count, arcs, canonical_indices)
    arc_flows = [0] * len(arcs)
    achieved_flow = 0

    while achieved_flow < required_flow:
        path = _shortest_residual_path(
            vertex_count,
            residual,
            outgoing,
            source,
            sink,
        )
        if path is None:
            break
        augmentation = required_flow - achieved_flow
        for residual_index in path:
            augmentation = min(augmentation, residual[residual_index].capacity)
        if augmentation <= 0:
            raise RuntimeError("augmenting path has no positive residual capacity")

        for residual_index in path:
            edge = residual[residual_index]
            edge.capacity -= augmentation
            residual[edge.twin].capacity += augmentation
            arc_flows[edge.original_index] += edge.direction * augmentation
        achieved_flow += augmentation

    potentials = _global_residual_potentials(
        vertex_count,
        residual,
        outgoing,
        source,
    )
    frozen_flows = tuple(arc_flows)
    if achieved_flow == required_flow:
        total_cost = sum(flow * arc.unit_cost for arc, flow in zip(arcs, frozen_flows, strict=True))
        return FeasibleRequiredFlow(
            flow_value=achieved_flow,
            total_cost=total_cost,
            arc_flows=frozen_flows,
            vertex_potentials=potentials,
        )

    source_side = _residual_source_side(residual, outgoing, source)
    if sink in source_side:
        raise RuntimeError("failed augmentation left the sink reachable")
    return InfeasibleRequiredFlow(
        achieved_flow=achieved_flow,
        arc_flows=frozen_flows,
        source_side=source_side,
    )
```

## Example

```python
def brute_flow_profile(
    vertex_count: int,
    arcs: tuple[CostedCapacityArc, ...],
    source: int,
    sink: int,
) -> dict[int, int]:
    from itertools import product

    best_cost_by_value: dict[int, int] = {}
    choices = (range(arc.capacity + 1) for arc in arcs)
    for flows in product(*choices):
        divergence = [0] * vertex_count
        for arc, flow in zip(arcs, flows, strict=True):
            divergence[arc.source] += flow
            divergence[arc.target] -= flow
        value = divergence[source]
        if value < 0 or divergence[sink] != -value:
            continue
        if any(
            divergence[vertex] != 0
            for vertex in range(vertex_count)
            if vertex not in (source, sink)
        ):
            continue
        cost = sum(arc.unit_cost * flow for arc, flow in zip(arcs, flows, strict=True))
        previous = best_cost_by_value.get(value)
        if previous is None or cost < previous:
            best_cost_by_value[value] = cost
    return best_cost_by_value


def verify_result(
    vertex_count: int,
    arcs: tuple[CostedCapacityArc, ...],
    source: int,
    sink: int,
    required_flow: int,
    result: FeasibleRequiredFlow | InfeasibleRequiredFlow,
) -> None:
    assert len(result.arc_flows) == len(arcs)
    divergence = [0] * vertex_count
    for arc, flow in zip(arcs, result.arc_flows, strict=True):
        assert 0 <= flow <= arc.capacity
        divergence[arc.source] += flow
        divergence[arc.target] -= flow

    if type(result) is FeasibleRequiredFlow:
        assert result.flow_value == required_flow
        assert divergence[source] == required_flow
        assert divergence[sink] == -required_flow
        assert all(
            divergence[vertex] == 0
            for vertex in range(vertex_count)
            if vertex not in (source, sink)
        )
        assert result.total_cost == sum(
            arc.unit_cost * flow for arc, flow in zip(arcs, result.arc_flows, strict=True)
        )
        assert len(result.vertex_potentials) == vertex_count
        assert result.vertex_potentials[source] == 0
        for arc, flow in zip(arcs, result.arc_flows, strict=True):
            if flow < arc.capacity:
                assert (
                    arc.unit_cost
                    + result.vertex_potentials[arc.source]
                    - result.vertex_potentials[arc.target]
                    >= 0
                )
            if flow > 0:
                assert (
                    -arc.unit_cost
                    + result.vertex_potentials[arc.target]
                    - result.vertex_potentials[arc.source]
                    >= 0
                )
        return

    assert type(result) is InfeasibleRequiredFlow
    assert result.achieved_flow < required_flow
    assert divergence[source] == result.achieved_flow
    assert divergence[sink] == -result.achieved_flow
    assert all(
        divergence[vertex] == 0 for vertex in range(vertex_count) if vertex not in (source, sink)
    )
    assert result.source_side == tuple(sorted(set(result.source_side)))
    source_side = set(result.source_side)
    assert source in source_side and sink not in source_side
    cut_capacity = sum(
        arc.capacity for arc in arcs if arc.source in source_side and arc.target not in source_side
    )
    assert cut_capacity == result.achieved_flow
    for arc, flow in zip(arcs, result.arc_flows, strict=True):
        if arc.source in source_side and arc.target not in source_side:
            assert flow == arc.capacity
        if arc.source not in source_side and arc.target in source_side:
            assert flow == 0


cancellation_arcs = (
    CostedCapacityArc(0, 1, 1, 0),
    CostedCapacityArc(0, 2, 1, 0),
    CostedCapacityArc(1, 3, 1, 0),
    CostedCapacityArc(1, 4, 1, 1),
    CostedCapacityArc(2, 3, 1, 0),
    CostedCapacityArc(3, 5, 1, 0),
    CostedCapacityArc(4, 5, 1, 0),
)
cancellation = minimum_cost_required_flow(6, cancellation_arcs, 0, 5, 2)
assert type(cancellation) is FeasibleRequiredFlow
verify_result(6, cancellation_arcs, 0, 5, 2, cancellation)
cancellation_profile = brute_flow_profile(6, cancellation_arcs, 0, 5)
reversed_arcs = tuple(reversed(cancellation_arcs))
reordered = minimum_cost_required_flow(6, reversed_arcs, 0, 5, 2)
assert type(reordered) is FeasibleRequiredFlow
verify_result(6, reversed_arcs, 0, 5, 2, reordered)

cut_arcs = (
    CostedCapacityArc(0, 1, 2, 0),
    CostedCapacityArc(1, 3, 1, 0),
    CostedCapacityArc(0, 2, 1, 1),
    CostedCapacityArc(2, 3, 1, 0),
)
infeasible = minimum_cost_required_flow(4, cut_arcs, 0, 3, 3)
verify_result(4, cut_arcs, 0, 3, 3, infeasible)
cut_profile = brute_flow_profile(4, cut_arcs, 0, 3)

antiparallel_arcs = (
    CostedCapacityArc(0, 1, 1, 1),
    CostedCapacityArc(1, 0, 1, 0),
    CostedCapacityArc(1, 2, 1, 1),
    CostedCapacityArc(0, 2, 1, 5),
)
antiparallel = minimum_cost_required_flow(3, antiparallel_arcs, 0, 2, 1)
assert type(antiparallel) is FeasibleRequiredFlow
verify_result(3, antiparallel_arcs, 0, 2, 1, antiparallel)
antiparallel_profile = brute_flow_profile(3, antiparallel_arcs, 0, 2)

zero = minimum_cost_required_flow(2, (), 0, 1, 0)
assert type(zero) is FeasibleRequiredFlow
verify_result(2, (), 0, 1, 0, zero)

boundary_arc = (CostedCapacityArc(0, 63, _MAX_ARC_VALUE, _MAX_ARC_VALUE),)
boundary = minimum_cost_required_flow(64, boundary_arc, 0, 63, _MAX_REQUIRED_FLOW)
assert type(boundary) is FeasibleRequiredFlow
verify_result(64, boundary_arc, 0, 63, _MAX_REQUIRED_FLOW, boundary)

assert (
    cancellation.arc_flows == (1, 1, 0, 1, 1, 1, 1)
    and cancellation.total_cost == 1
    and cancellation.total_cost == cancellation_profile[2]
    and reordered.total_cost == cancellation.total_cost
    and cancellation.arc_flows == tuple(reversed(reordered.arc_flows))
    and type(infeasible) is InfeasibleRequiredFlow
    and infeasible.achieved_flow == 2
    and infeasible.source_side == (0, 1)
    and infeasible.achieved_flow == max(cut_profile)
    and antiparallel.total_cost == antiparallel_profile[1] == 2
    and zero.arc_flows == ()
    and boundary.total_cost == _MAX_REQUIRED_FLOW * _MAX_ARC_VALUE
)
```

## Trade-offs and Limitations

Validation and residual construction take `O(V + E log E)` time and
`O(V + E)` space because canonicalization sorts the arcs. Each
augmentation uses Bellman-Ford over at most `2E` residual arcs, so the bounded
worst case is `O(FVE)` time for required flow `F`; every augmentation moves at
least one integral unit. The final global potential pass costs `O(VE)`.
Python integers keep costs, path distances, and totals exact beyond signed
64-bit range.

The feasible certificate is local to the returned flow and original network.
Flow conservation, capacity bounds, exact value and total cost must be checked
alongside the reduced-cost inequalities. Potentials admit an additive gauge;
this profile normalizes the source potential to zero. The infeasible result
does not return potentials because flow feasibility and equality between the
achieved value and directed cut capacity already prove that no larger flow can
exist.

Canonical traversal makes the selected flow independent of input arc order,
but it does not promise the lexicographically first optimum. Returned flow
positions always follow the caller's arc order. The implementation retains
separate residual identities for antiparallel originals and their cancellation
arcs instead of combining them into one net capacity.

Capacities must be positive integers, and original unit costs must be
non-negative. The function has no negative-cost inputs, lower bounds, node
supplies, multiple sources or sinks, floating-point costs, persistent state,
incremental updates, or concurrency control. Bellman-Ford is intentionally
transparent at this scale; a production optimizer can use reduced-cost
Dijkstra searches and stronger presolve for larger workloads.

## Related Snippets

<!-- catalog:related:start -->
- [Compute a Bounded Integer Maximum Flow and Canonical Minimum Cut with Edmonds-Karp](compute-a-bounded-integer-maximum-flow-and-canonical-minimum-cut-with-edmonds-karp.md)
- [Find a Deterministic Minimum-Cost Assignment and Dual Certificate with the Hungarian Algorithm](find-a-deterministic-minimum-cost-perfect-assignment-and-dual-certificate-with-the-hungarian-algorithm.md)
- [Compute Bounded Bellman-Ford Distances and Detect Reachable Negative Cycles](compute-bounded-bellman-ford-distances-and-detect-reachable-negative-cycles.md)
<!-- catalog:related:end -->
