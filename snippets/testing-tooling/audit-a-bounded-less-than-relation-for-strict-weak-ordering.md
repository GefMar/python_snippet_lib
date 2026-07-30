---
title: "Audit a Bounded Less-Than Relation for Strict Weak Ordering"
snippet_type: testing-technique
use_cases:
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../algorithms-data-structures/resolve-stable-ordering-constraints-with-topological-sort.md
  - ../algorithms-data-structures/rank-and-unrank-index-permutations-in-itertools-permutations-order.md
  - audit-a-bounded-test-matrix-for-complete-pairwise-coverage.md
---

# Audit a Bounded Less-Than Relation for Strict Weak Ordering

## Idea and Problem

Audit a custom less-than callback on a bounded sample before using it with an algorithm that expects a strict weak ordering.

A valid relation is irreflexive and transitive. It also makes incomparability
transitive: if neither of two pairs is ordered, the outer pair must likewise be
unordered. That last rule permits meaningful equivalence classes while
preventing unstable comparisons such as “close enough” gaps. The audit records
indices, so equal or repeated sample values remain distinguishable.

## When to Use

Use this test helper for a small representative fixture when a domain-specific
less-than relation controls sorting, ranking, ordered containers, or selection.
It gives a concrete first witness when the sampled relation violates one of the
laws and also detects ordinary exceptions, non-Boolean answers, and answers
that change between two immediate calls.

This is evidence about one finite sample, not a proof for the callback's whole
domain. Prefer a normal key function when values can be mapped to an existing
ordered type; keys are simpler to reason about and let Python provide the
comparison laws.

## Implementation

```python
from collections.abc import Callable
from dataclasses import dataclass
from enum import StrEnum

_MAX_AUDIT_ITEMS = 32


class OrderingViolationKind(StrEnum):
    RAISED = "raised"
    NON_BOOLEAN = "non-boolean"
    INCONSISTENT = "inconsistent"
    REFLEXIVE = "reflexive"
    ASYMMETRIC = "asymmetric"
    NON_TRANSITIVE = "non-transitive"
    INCOMPARABILITY_NOT_TRANSITIVE = "incomparability-not-transitive"


@dataclass(frozen=True, slots=True)
class OrderingViolation:
    kind: OrderingViolationKind
    indexes: tuple[int, ...]
    detail: str | None = None


@dataclass(frozen=True, slots=True)
class OrderingAudit:
    violation: OrderingViolation | None

    @property
    def valid(self) -> bool:
        return self.violation is None


def audit_strict_weak_order[T](
    items: tuple[T, ...],
    less: Callable[[T, T], bool],
) -> OrderingAudit:
    if type(items) is not tuple:
        raise TypeError("items must be an exact tuple")
    if not 1 <= len(items) <= _MAX_AUDIT_ITEMS:
        raise ValueError("item count is outside 1..32")
    if not callable(less):
        raise TypeError("less must be callable")

    size = len(items)
    matrix = [[False] * size for _ in items]
    for left_index, left in enumerate(items):
        for right_index, right in enumerate(items):
            observations: list[bool] = []
            for _ in range(2):
                try:
                    result = less(left, right)
                except Exception as error:
                    return OrderingAudit(
                        OrderingViolation(
                            OrderingViolationKind.RAISED,
                            (left_index, right_index),
                            type(error).__name__,
                        )
                    )
                if type(result) is not bool:
                    return OrderingAudit(
                        OrderingViolation(
                            OrderingViolationKind.NON_BOOLEAN,
                            (left_index, right_index),
                            type(result).__name__,
                        )
                    )
                observations.append(result)
            if observations[0] != observations[1]:
                return OrderingAudit(
                    OrderingViolation(
                        OrderingViolationKind.INCONSISTENT,
                        (left_index, right_index),
                    )
                )
            matrix[left_index][right_index] = observations[0]

    for index in range(size):
        if matrix[index][index]:
            return OrderingAudit(
                OrderingViolation(OrderingViolationKind.REFLEXIVE, (index,))
            )

    for left in range(size):
        for right in range(size):
            if matrix[left][right] and matrix[right][left]:
                return OrderingAudit(
                    OrderingViolation(OrderingViolationKind.ASYMMETRIC, (left, right))
                )

    for left in range(size):
        for middle in range(size):
            for right in range(size):
                if (
                    matrix[left][middle]
                    and matrix[middle][right]
                    and not matrix[left][right]
                ):
                    return OrderingAudit(
                        OrderingViolation(
                            OrderingViolationKind.NON_TRANSITIVE,
                            (left, middle, right),
                        )
                    )

    def incomparable(left: int, right: int) -> bool:
        return not matrix[left][right] and not matrix[right][left]

    for left in range(size):
        for middle in range(size):
            for right in range(size):
                if (
                    incomparable(left, middle)
                    and incomparable(middle, right)
                    and not incomparable(left, right)
                ):
                    return OrderingAudit(
                        OrderingViolation(
                            OrderingViolationKind.INCOMPARABILITY_NOT_TRANSITIVE,
                            (left, middle, right),
                        )
                    )

    return OrderingAudit(None)
```

## Example

```python
def matrix_is_strict_weak(matrix: tuple[tuple[bool, ...], ...]) -> bool:
    size = len(matrix)

    def incomparable(left: int, right: int) -> bool:
        return not matrix[left][right] and not matrix[right][left]

    return (
        all(not matrix[index][index] for index in range(size))
        and all(
            not (matrix[left][right] and matrix[right][left])
            for left in range(size)
            for right in range(size)
        )
        and all(
            not (matrix[left][middle] and matrix[middle][right])
            or matrix[left][right]
            for left in range(size)
            for middle in range(size)
            for right in range(size)
        )
        and all(
            not (incomparable(left, middle) and incomparable(middle, right))
            or incomparable(left, right)
            for left in range(size)
            for middle in range(size)
            for right in range(size)
        )
    )


checked_matrices = 0
for mask in range(1 << 9):
    relation = tuple(
        tuple(bool(mask & (1 << (left * 3 + right))) for right in range(3))
        for left in range(3)
    )
    observed = audit_strict_weak_order(
        (0, 1, 2),
        lambda left, right, relation=relation: relation[left][right],
    )
    assert observed.valid is matrix_is_strict_weak(relation)
    checked_matrices += 1

ascending = audit_strict_weak_order((3, 1, 2), lambda left, right: left < right)
descending = audit_strict_weak_order((3, 1, 2), lambda left, right: left > right)
parity_classes = audit_strict_weak_order(
    (4, 2, 3, 1),
    lambda left, right: left % 2 < right % 2,
)
duplicates = audit_strict_weak_order(("same", "same"), lambda left, right: left < right)

reflexive = audit_strict_weak_order((0, 1), lambda left, right: left <= right)
cyclic = audit_strict_weak_order(
    (0, 1, 2),
    lambda left, right: (right - left) % 3 == 1,
)
gapped = audit_strict_weak_order((0, 1, 2), lambda left, right: left + 1 < right)

toggle_state = False


def toggling_less(_left: object, _right: object) -> bool:
    global toggle_state
    toggle_state = not toggle_state
    return toggle_state


inconsistent = audit_strict_weak_order((object(),), toggling_less)
non_boolean = audit_strict_weak_order((0,), lambda _left, _right: 1)


def raising_less(_left: object, _right: object) -> bool:
    raise LookupError("synthetic failure")


raised = audit_strict_weak_order((object(),), raising_less)

assert (
    checked_matrices == 512
    and ascending.valid
    and descending.valid
    and parity_classes.valid
    and duplicates.valid
    and reflexive.violation
    == OrderingViolation(OrderingViolationKind.REFLEXIVE, (0,))
    and cyclic.violation
    == OrderingViolation(OrderingViolationKind.NON_TRANSITIVE, (0, 1, 2))
    and gapped.violation
    == OrderingViolation(
        OrderingViolationKind.INCOMPARABILITY_NOT_TRANSITIVE,
        (0, 1, 2),
    )
    and inconsistent.violation
    == OrderingViolation(OrderingViolationKind.INCONSISTENT, (0, 0))
    and non_boolean.violation
    == OrderingViolation(OrderingViolationKind.NON_BOOLEAN, (0, 0), "int")
    and raised.violation
    == OrderingViolation(OrderingViolationKind.RAISED, (0, 0), "LookupError")
)
```

## Trade-offs and Limitations

The callback is invoked exactly `2 * N**2` times when observation succeeds.
The frozen matrix uses `O(N**2)` space, and the two transitivity scans take
`O(N**3)` time. The 32-item limit keeps that exhaustive audit intentional; it
is not suitable for production hot paths or large property-test samples.

The fixed check order makes a report reproducible: observation failures come
first in row-major order, followed by irreflexivity, asymmetry, transitivity,
and transitive incomparability. A strict weak order may contain equivalent
values, so incomparability is not itself an error. The helper does not require
a total order, sort the values, repair a relation, catch `BaseException`, make
untrusted callbacks safe, or prove behavior outside the sample. Two equal
observations also cannot guarantee that a stateful callback will stay stable.

## Related Snippets

<!-- catalog:related:start -->
- [Resolve Stable Ordering Constraints with Topological Sort](../algorithms-data-structures/resolve-stable-ordering-constraints-with-topological-sort.md)
- [Rank and Unrank Index Permutations in itertools.permutations Order](../algorithms-data-structures/rank-and-unrank-index-permutations-in-itertools-permutations-order.md)
- [Audit a Bounded Test Matrix for Complete Pairwise Coverage](audit-a-bounded-test-matrix-for-complete-pairwise-coverage.md)
<!-- catalog:related:end -->
