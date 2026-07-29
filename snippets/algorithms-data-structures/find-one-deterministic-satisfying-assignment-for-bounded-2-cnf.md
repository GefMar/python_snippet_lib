---
title: "Find One Deterministic Satisfying Assignment for Bounded 2-CNF"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - partition-a-bounded-directed-graph-into-deterministic-strongly-connected-components.md
  - build-and-evaluate-a-bounded-binary-assignment-constraint-system.md
  - resolve-stable-ordering-constraints-with-topological-sort.md
---

# Find One Deterministic Satisfying Assignment for Bounded 2-CNF

## Idea and Problem

Decide whether a bounded two-literal Boolean formula is satisfiable and return one reproducible assignment when it is.

Each clause `(a or b)` is equivalent to two implications: `not a` implies
`b`, and `not b` implies `a`. A formula is unsatisfiable exactly when one
variable and its negation belong to the same strongly connected component of
that implication graph.

When they are separate, the components' deterministic source-to-sink order
chooses a value for every variable. Sorting and deduplicating implication edges
makes the result independent of clause order and of literal order inside each
clause.

## When to Use

Use this algorithm for one fully materialized 2-CNF formula with stable
zero-based variable positions. Literals use signed one-based notation:
`index + 1` means that variable is true and `-(index + 1)` means it is false.
The returned Boolean tuple is aligned with the zero-based variable positions.

This fits implication-like configuration checks, compatibility constraints,
and binary policy validation. Use a general SAT solver for clauses wider than
two literals, optimization objectives, incremental solving, assumptions,
proofs, or unsatisfiable cores.

## Implementation

```python
_MAX_2_CNF_VARIABLES = 20_000
_MAX_2_CNF_CLAUSES = 50_000


def _literal_node(literal: int) -> int:
    variable = abs(literal) - 1
    polarity = 1 if literal > 0 else 0
    return 2 * variable + polarity


def _finish_order(
    adjacency: tuple[tuple[int, ...], ...],
) -> tuple[int, ...]:
    visited = [False] * len(adjacency)
    finished: list[int] = []

    for start in range(len(adjacency)):
        if visited[start]:
            continue
        visited[start] = True
        stack: list[tuple[int, int]] = [(start, 0)]
        while stack:
            node, neighbor_index = stack[-1]
            if neighbor_index == len(adjacency[node]):
                stack.pop()
                finished.append(node)
                continue

            stack[-1] = (node, neighbor_index + 1)
            neighbor = adjacency[node][neighbor_index]
            if not visited[neighbor]:
                visited[neighbor] = True
                stack.append((neighbor, 0))
    return tuple(finished)


def _source_first_component_indexes(
    adjacency: tuple[tuple[int, ...], ...],
    reverse_adjacency: tuple[tuple[int, ...], ...],
) -> tuple[int, ...]:
    component_indexes = [-1] * len(adjacency)
    next_component = 0

    for start in reversed(_finish_order(adjacency)):
        if component_indexes[start] != -1:
            continue
        component_indexes[start] = next_component
        pending = [start]
        while pending:
            node = pending.pop()
            for predecessor in reversed(reverse_adjacency[node]):
                if component_indexes[predecessor] == -1:
                    component_indexes[predecessor] = next_component
                    pending.append(predecessor)
        next_component += 1
    return tuple(component_indexes)


def find_2_cnf_satisfying_assignment(
    variable_count: int,
    clauses: tuple[tuple[int, int], ...],
) -> tuple[bool, ...] | None:
    """Return one deterministic satisfying assignment, or None."""
    if type(variable_count) is not int:
        raise TypeError("variable_count must be an exact integer")
    if not 1 <= variable_count <= _MAX_2_CNF_VARIABLES:
        raise ValueError("variable_count is outside the supported range")
    if type(clauses) is not tuple:
        raise TypeError("clauses must be an exact tuple")
    if len(clauses) > _MAX_2_CNF_CLAUSES:
        raise ValueError("clause count exceeds the supported limit")

    validated_clauses: list[tuple[int, int]] = []
    for clause_index, clause in enumerate(clauses):
        if type(clause) is not tuple:
            raise TypeError(f"clauses[{clause_index}] must be an exact tuple")
        if len(clause) != 2:
            raise ValueError(f"clauses[{clause_index}] must contain two literals")

        first, second = clause
        for literal_index, literal in enumerate((first, second)):
            if type(literal) is not int:
                raise TypeError(
                    f"clauses[{clause_index}][{literal_index}] must be an exact integer"
                )
            if literal == 0:
                raise ValueError(f"clauses[{clause_index}] contains the zero literal")
            if abs(literal) > variable_count:
                raise ValueError(f"clauses[{clause_index}] contains an unknown variable")
        validated_clauses.append((first, second))

    implication_edges: list[tuple[int, int]] = []
    for first, second in validated_clauses:
        first_node = _literal_node(first)
        second_node = _literal_node(second)
        implication_edges.append((first_node ^ 1, second_node))
        implication_edges.append((second_node ^ 1, first_node))
    implication_edges.sort()

    node_count = 2 * variable_count
    adjacency_lists: list[list[int]] = [[] for _ in range(node_count)]
    reverse_lists: list[list[int]] = [[] for _ in range(node_count)]
    previous_edge: tuple[int, int] | None = None
    for edge in implication_edges:
        if edge == previous_edge:
            continue
        source, target = edge
        adjacency_lists[source].append(target)
        reverse_lists[target].append(source)
        previous_edge = edge

    adjacency = tuple(tuple(targets) for targets in adjacency_lists)
    reverse_adjacency = tuple(tuple(sources) for sources in reverse_lists)
    components = _source_first_component_indexes(adjacency, reverse_adjacency)

    assignment: list[bool] = []
    for variable in range(variable_count):
        false_node = 2 * variable
        true_node = false_node + 1
        if components[false_node] == components[true_node]:
            return None
        assignment.append(components[true_node] > components[false_node])
    return tuple(assignment)
```

## Example

```python
def literal_is_true(literal: int, assignment: tuple[bool, ...]) -> bool:
    value = assignment[abs(literal) - 1]
    return value if literal > 0 else not value


def formula_is_satisfied(
    clauses: tuple[tuple[int, int], ...],
    assignment: tuple[bool, ...],
) -> bool:
    return all(
        literal_is_true(first, assignment) or literal_is_true(second, assignment)
        for first, second in clauses
    )


def exhaustive_assignment(
    variable_count: int,
    clauses: tuple[tuple[int, int], ...],
) -> tuple[bool, ...] | None:
    from itertools import product

    for assignment in product((False, True), repeat=variable_count):
        if formula_is_satisfied(clauses, assignment):
            return assignment
    return None


def exercise_small_formulas() -> int:
    from itertools import combinations, combinations_with_replacement

    checked = 0
    for variable_count in range(1, 4):
        literals = tuple(range(-variable_count, 0)) + tuple(range(1, variable_count + 1))
        possible_clauses = tuple(combinations_with_replacement(literals, 2))
        for clause_count in range(4):
            for selected in combinations(possible_clauses, clause_count):
                clauses = tuple(selected)
                reference = exhaustive_assignment(variable_count, clauses)
                actual = find_2_cnf_satisfying_assignment(variable_count, clauses)
                assert (actual is None) == (reference is None)
                if actual is not None:
                    assert formula_is_satisfied(clauses, actual)

                reordered = tuple((second, first) for first, second in reversed(clauses))
                assert find_2_cnf_satisfying_assignment(variable_count, reordered) == actual
                checked += 1
    return checked


ordinary_clauses = ((1, 2), (-1, 3), (-2, -3))
ordinary_assignment = find_2_cnf_satisfying_assignment(3, ordinary_clauses)
empty_assignment = find_2_cnf_satisfying_assignment(4, ())
unsatisfiable = find_2_cnf_satisfying_assignment(1, ((1, 1), (-1, -1)))

long_chain_clauses = (
    (1, 1),
    *((-variable, variable + 1) for variable in range(1, _MAX_2_CNF_VARIABLES)),
)
long_chain_assignment = find_2_cnf_satisfying_assignment(
    _MAX_2_CNF_VARIABLES,
    long_chain_clauses,
)

maximum_tautologies = ((1, -1),) * _MAX_2_CNF_CLAUSES
maximum_assignment = find_2_cnf_satisfying_assignment(
    1,
    maximum_tautologies,
)

value_errors = 0
for variable_count, invalid_clauses in (
    (0, ()),
    (_MAX_2_CNF_VARIABLES + 1, ()),
    (1, ((1, -1),) * (_MAX_2_CNF_CLAUSES + 1)),
    (1, ((1,),)),
    (1, ((0, 1),)),
    (1, ((1, 2),)),
):
    try:
        find_2_cnf_satisfying_assignment(variable_count, invalid_clauses)
    except ValueError:
        value_errors += 1

type_errors = 0
for variable_count, invalid_clauses in (
    (True, ()),
    (1, []),
    (1, ([1, 1],)),
    (1, ((True, 1),)),
):
    try:
        find_2_cnf_satisfying_assignment(variable_count, invalid_clauses)
    except TypeError:
        type_errors += 1

assert (
    exercise_small_formulas(),
    empty_assignment,
    ordinary_assignment is not None and formula_is_satisfied(ordinary_clauses, ordinary_assignment),
    unsatisfiable,
    len(long_chain_assignment or ()),
    set(long_chain_assignment or ()),
    maximum_assignment,
    value_errors,
    type_errors,
) == (
    1_746,
    (False, False, False, False),
    True,
    None,
    20_000,
    {True},
    (False,),
    6,
    4,
)
```

## Trade-offs and Limitations

Sorting at most twice the clause count dominates the normalization cost. For
`n` variables and `m` clauses, the complete algorithm uses
`O(m log m + n + m)` time and `O(n + m)` memory. Both graph traversals are
iterative, so a long implication chain does not consume Python call-stack
depth.

The returned assignment is reproducible under the documented false-before-true
vertex order and sorted implication traversal. It is invariant to clause order,
literal order inside a clause, duplicate clauses, and duplicate implications,
but it is not promised to be the lexicographically smallest satisfying
assignment. Tautological clauses are accepted because they impose no
restriction.

The function returns no proof, unsatisfiable core, implication path, or count
of satisfying assignments. It handles exactly two literals per clause and one
complete static formula; it does not support wider clauses, incremental
updates, weighted preferences, optimization, assumptions, or partial
assignments.

## Related Snippets

<!-- catalog:related:start -->
- [Partition a Bounded Directed Graph into Deterministic Strongly Connected Components](partition-a-bounded-directed-graph-into-deterministic-strongly-connected-components.md)
- [Build and Evaluate a Bounded Binary Assignment Constraint System](build-and-evaluate-a-bounded-binary-assignment-constraint-system.md)
- [Resolve Stable Ordering Constraints with Topological Sort](resolve-stable-ordering-constraints-with-topological-sort.md)
<!-- catalog:related:end -->
