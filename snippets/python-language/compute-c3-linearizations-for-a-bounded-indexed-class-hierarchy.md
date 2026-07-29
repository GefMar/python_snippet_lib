---
title: "Compute C3 Linearizations for a Bounded Indexed Class Hierarchy"
snippet_type: algorithm
use_cases:
  - automation
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - dispatch-on-an-exact-tuple-of-argument-types.md
  - collect-decorated-methods-in-class-definition-order.md
  - type-a-narrow-structural-interface-with-protocol.md
---

# Compute C3 Linearizations for a Bounded Indexed Class Hierarchy

## Idea and Problem

Compute the Python-style C3 linearization of one class in a closed indexed hierarchy without creating runtime classes.

Each tuple position is a class index, and its value lists direct base indexes in
declared order. C3 recursively linearizes those bases, then repeatedly selects
the first sequence head that appears in none of the remaining sequence tails.
That merge preserves local precedence order and monotonicity.

Malformed references and cycles are graph errors. A closed acyclic graph can
still impose contradictory base orders; only that C3 inconsistency returns
`None`.

## When to Use

Use this algorithm to validate or explain a small declarative hierarchy that is
intended to follow ordinary Python multiple-inheritance ordering. Integer
indexes keep the model independent of imports and make the result easy to
compare in tests.

Use Python's actual class construction when runtime types are the desired
result. Use a purpose-built hierarchy model when resolution rules differ from
C3 or when diagnostics must identify a minimal contradictory constraint set.

## Implementation

```python
from collections import Counter

_MAX_C3_CLASSES = 100


def _merge_c3(
    sequences: tuple[tuple[int, ...], ...],
) -> tuple[int, ...] | None:
    active = [sequence for sequence in sequences if sequence]
    positions = [0] * len(active)
    tail_counts: Counter[int] = Counter()
    for sequence in active:
        tail_counts.update(sequence[1:])

    merged: list[int] = []
    while True:
        candidate: int | None = None
        has_head = False

        for sequence, position in zip(active, positions, strict=True):
            if position == len(sequence):
                continue
            has_head = True
            head = sequence[position]
            if tail_counts[head] == 0:
                candidate = head
                break

        if not has_head:
            return tuple(merged)
        if candidate is None:
            return None

        merged.append(candidate)
        for index, sequence in enumerate(active):
            position = positions[index]
            if position == len(sequence) or sequence[position] != candidate:
                continue
            new_position = position + 1
            positions[index] = new_position
            if new_position < len(sequence):
                new_head = sequence[new_position]
                tail_counts[new_head] -= 1


def compute_c3_linearization(
    base_orders: tuple[tuple[int, ...], ...],
    *,
    target: int,
) -> tuple[int, ...] | None:
    """Return the target's C3 order, or None for an inconsistent merge."""
    if type(base_orders) is not tuple:
        raise TypeError("base_orders must be an exact tuple")
    class_count = len(base_orders)
    if not 1 <= class_count <= _MAX_C3_CLASSES:
        raise ValueError("class count is outside the supported range")

    for class_index, bases in enumerate(base_orders):
        if type(bases) is not tuple:
            raise TypeError(f"base_orders[{class_index}] must be an exact tuple")
        seen: set[int] = set()
        for position, base in enumerate(bases):
            if type(base) is not int:
                raise TypeError(f"base_orders[{class_index}][{position}] must be an exact integer")
            if not 0 <= base < class_count:
                raise ValueError(
                    f"base_orders[{class_index}][{position}] is outside the closed class index"
                )
            if base in seen:
                raise ValueError(f"base_orders[{class_index}] contains a duplicate base")
            seen.add(base)

    if type(target) is not int:
        raise TypeError("target must be an exact integer")
    if not 0 <= target < class_count:
        raise ValueError("target is outside the closed class index")

    states = [0] * class_count

    def validate_acyclic(class_index: int) -> None:
        if states[class_index] == 1:
            raise ValueError("base graph contains a cycle")
        if states[class_index] == 2:
            return
        states[class_index] = 1
        for base in base_orders[class_index]:
            validate_acyclic(base)
        states[class_index] = 2

    for class_index in range(class_count):
        validate_acyclic(class_index)

    memo: dict[int, tuple[int, ...] | None] = {}

    def linearize(class_index: int) -> tuple[int, ...] | None:
        if class_index in memo:
            return memo[class_index]

        bases = base_orders[class_index]
        base_linearizations: list[tuple[int, ...]] = []
        for base in bases:
            base_result = linearize(base)
            if base_result is None:
                memo[class_index] = None
                return None
            base_linearizations.append(base_result)

        merged = _merge_c3((*base_linearizations, bases))
        if merged is None:
            memo[class_index] = None
            return None

        result = (class_index, *merged)
        memo[class_index] = result
        return result

    return linearize(target)
```

## Example

```python
# 0 is the common root. Classes 3 and 4 request opposite orders for 1 and 2.
# Class 5 tries to inherit from both and therefore has no valid C3 merge.
base_orders = (
    (),
    (0,),
    (0,),
    (1, 2),
    (2, 1),
    (3, 4),
)

assert compute_c3_linearization(base_orders, target=0) == (0,)
assert compute_c3_linearization(base_orders, target=3) == (3, 1, 2, 0)
assert compute_c3_linearization(base_orders, target=4) == (4, 2, 1, 0)
assert compute_c3_linearization(base_orders, target=5) is None

try:
    compute_c3_linearization(((1,), (0,)), target=0)
except ValueError:
    cycle_rejected = True
else:
    cycle_rejected = False

assert cycle_rejected
```

## Trade-offs and Limitations

For `n` indexed classes, cached ancestor orders occupy `O(n^2)` space. Tail
counts make one C3 merge quadratic in its bounded participating orders, giving
`O(n^3)` worst-case time across all recursively requested linearizations.
Cycle validation and recursion depth are bounded by 100 classes.

The input index is closed: every base is one of its declared positions.
Duplicate direct bases, missing indexes, an invalid target, and any cycle
anywhere in the full graph raise an exception. `None` is reserved for a valid
acyclic target dependency graph whose C3 merge has no eligible head. The result
does not include an explanation or a minimal conflicting subset.

The ordering models ordinary pure-Python C3 semantics, but it does not create
classes or reproduce every condition enforced by `type`. Custom metaclass
`mro()` methods, extension-type layout conflicts, slots, descriptors, class
initialization hooks, and import behavior are outside this representation.

## Related Snippets

<!-- catalog:related:start -->
- [Dispatch on an Exact Tuple of Argument Types](dispatch-on-an-exact-tuple-of-argument-types.md)
- [Collect Decorated Methods in Class Definition Order](collect-decorated-methods-in-class-definition-order.md)
- [Type a Narrow Structural Interface with Protocol](type-a-narrow-structural-interface-with-protocol.md)
<!-- catalog:related:end -->
