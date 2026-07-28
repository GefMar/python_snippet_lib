---
title: "Build and Evaluate a Bounded Binary Assignment Constraint System"
snippet_type: algorithm
use_cases:
  - data-transformation
  - resource-management
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - report-exact-capacity-deficits-for-bounded-resource-profiles.md
  - partition-tagged-items-into-minimum-stable-conflict-free-groups.md
---

# Build and Evaluate a Bounded Binary Assignment Constraint System

## Idea and Problem

Build a bounded, solver-neutral system of sparse integer constraints for assigning one category and option to every slot, then evaluate complete binary assignments against it.

For zero-based coordinates, variable `x(slot, category, option)` has flat index
`((slot * category_count) + category) * option_count + option`. The builder emits
rows in three exact groups: slot equalities in slot order, category capacities in
category order, then adjacency rows in left-slot order followed by category
order. Stable IDs follow the same coordinates, so an evaluator can report all
violations without depending on solver-specific names or result objects.

## When to Use

Use this algorithm when every slot must select exactly one category/option,
each category has one total weighted capacity, and adjacent slots may not select
the same category. It fits small deterministic test fixtures, interchange
boundaries, and preprocessing for a separately owned sparse solver adapter.

`weights` is an exact tuple with one value per `(category, option)`, at index
`category * option_count + option`; `capacities` has one value per category.
Dimensions must be exact positive integers, and all values must be exact
non-negative integers within the documented constants. Proposed assignments
must be exact tuples containing one exact integer zero or one per variable.

## Implementation

```python
from dataclasses import dataclass
from enum import StrEnum

_MAX_SLOTS = 128
_MAX_CATEGORIES = 64
_MAX_OPTIONS = 64
_MAX_WEIGHT_VALUES = 4_096
_MAX_VARIABLES = 32_768
_MAX_ROWS = 8_192
_MAX_TERMS = 100_000
_MAX_VALUE = (1 << 31) - 1
_MAX_WEIGHTED_ROW_TOTAL = (1 << 40) - 1


class Relation(StrEnum):
    EQUAL = "=="
    AT_MOST = "<="


@dataclass(frozen=True, slots=True)
class SparseTerm:
    variable_index: int
    coefficient: int


@dataclass(frozen=True, slots=True)
class ConstraintRow:
    constraint_id: str
    relation: Relation
    terms: tuple[SparseTerm, ...]
    rhs: int


@dataclass(frozen=True, slots=True)
class BinaryAssignmentSystem:
    slot_count: int
    category_count: int
    option_count: int
    variable_count: int
    rows: tuple[ConstraintRow, ...]


def _require_dimension(value: object, *, name: str, maximum: int) -> int:
    if type(value) is not int:
        raise TypeError(f"{name} must be an exact int")
    if not 1 <= value <= maximum:
        raise ValueError(f"{name} is outside the supported positive range")
    return value


def _require_value(value: object, *, name: str) -> int:
    if type(value) is not int:
        raise TypeError(f"{name} must be an exact int")
    if not 0 <= value <= _MAX_VALUE:
        raise ValueError(f"{name} is outside the supported non-negative range")
    return value


def _variable_index(
    slot: int,
    category: int,
    option: int,
    *,
    category_count: int,
    option_count: int,
) -> int:
    return ((slot * category_count) + category) * option_count + option


def build_binary_assignment_system(
    *,
    slot_count: int,
    category_count: int,
    option_count: int,
    weights: tuple[int, ...],
    capacities: tuple[int, ...],
) -> BinaryAssignmentSystem:
    slot_count = _require_dimension(
        slot_count,
        name="slot_count",
        maximum=_MAX_SLOTS,
    )
    category_count = _require_dimension(
        category_count,
        name="category_count",
        maximum=_MAX_CATEGORIES,
    )
    option_count = _require_dimension(
        option_count,
        name="option_count",
        maximum=_MAX_OPTIONS,
    )

    category_option_count = category_count * option_count
    if category_option_count > _MAX_WEIGHT_VALUES:
        raise ValueError("category-by-option size exceeds the weight-value limit")
    variable_count = slot_count * category_option_count
    if variable_count > _MAX_VARIABLES:
        raise ValueError("slot-by-category-by-option size exceeds the variable limit")

    adjacency_row_count = (slot_count - 1) * category_count
    row_count = slot_count + category_count + adjacency_row_count
    if row_count > _MAX_ROWS:
        raise ValueError("generated row count exceeds the limit")

    if type(weights) is not tuple:
        raise TypeError("weights must be an exact tuple")
    if len(weights) != category_option_count:
        raise ValueError("weights must contain one value per category and option")
    if type(capacities) is not tuple:
        raise TypeError("capacities must be an exact tuple")
    if len(capacities) != category_count:
        raise ValueError("capacities must contain one value per category")

    positive_weight_count = 0
    category_weight_totals = [0] * category_count
    for weight_index, value in enumerate(weights):
        weight = _require_value(value, name=f"weights[{weight_index}]")
        category = weight_index // option_count
        category_weight_totals[category] += weight
        if weight > 0:
            positive_weight_count += 1
    for category, total in enumerate(category_weight_totals):
        if slot_count * total > _MAX_WEIGHTED_ROW_TOTAL:
            raise ValueError(f"weighted row {category} exceeds the aggregate limit")

    for category, value in enumerate(capacities):
        _require_value(value, name=f"capacities[{category}]")

    equality_term_count = variable_count
    capacity_term_count = slot_count * positive_weight_count
    adjacency_term_count = adjacency_row_count * (2 * option_count)
    term_count = equality_term_count + capacity_term_count + adjacency_term_count
    if term_count > _MAX_TERMS:
        raise ValueError("generated sparse term count exceeds the limit")

    rows: list[ConstraintRow] = []
    for slot in range(slot_count):
        terms = tuple(
            SparseTerm(
                _variable_index(
                    slot,
                    category,
                    option,
                    category_count=category_count,
                    option_count=option_count,
                ),
                1,
            )
            for category in range(category_count)
            for option in range(option_count)
        )
        rows.append(
            ConstraintRow(
                constraint_id=f"slot:{slot}:exactly-one",
                relation=Relation.EQUAL,
                terms=terms,
                rhs=1,
            )
        )

    for category in range(category_count):
        terms = tuple(
            SparseTerm(
                _variable_index(
                    slot,
                    category,
                    option,
                    category_count=category_count,
                    option_count=option_count,
                ),
                weights[category * option_count + option],
            )
            for slot in range(slot_count)
            for option in range(option_count)
            if weights[category * option_count + option] > 0
        )
        rows.append(
            ConstraintRow(
                constraint_id=f"category:{category}:capacity",
                relation=Relation.AT_MOST,
                terms=terms,
                rhs=capacities[category],
            )
        )

    for left_slot in range(slot_count - 1):
        for category in range(category_count):
            terms = tuple(
                SparseTerm(
                    _variable_index(
                        slot,
                        category,
                        option,
                        category_count=category_count,
                        option_count=option_count,
                    ),
                    1,
                )
                for slot in (left_slot, left_slot + 1)
                for option in range(option_count)
            )
            rows.append(
                ConstraintRow(
                    constraint_id=(f"adjacent:{left_slot}:{left_slot + 1}:category:{category}"),
                    relation=Relation.AT_MOST,
                    terms=terms,
                    rhs=1,
                )
            )

    return BinaryAssignmentSystem(
        slot_count=slot_count,
        category_count=category_count,
        option_count=option_count,
        variable_count=variable_count,
        rows=tuple(rows),
    )


def evaluate_binary_assignment(
    system: BinaryAssignmentSystem,
    assignment: tuple[int, ...],
) -> tuple[str, ...]:
    if type(system) is not BinaryAssignmentSystem:
        raise TypeError("system must be an exact BinaryAssignmentSystem")
    if type(assignment) is not tuple:
        raise TypeError("assignment must be an exact tuple")
    if len(assignment) != system.variable_count:
        raise ValueError("assignment length must equal the system variable count")
    for variable_index, value in enumerate(assignment):
        if type(value) is not int:
            raise TypeError(f"assignment[{variable_index}] must be an exact int")
        if value not in (0, 1):
            raise ValueError(f"assignment[{variable_index}] must be zero or one")

    violated: list[str] = []
    for row in system.rows:
        left_hand_side = sum(
            assignment[term.variable_index] * term.coefficient for term in row.terms
        )
        if row.relation is Relation.EQUAL:
            is_satisfied = left_hand_side == row.rhs
        elif row.relation is Relation.AT_MOST:
            is_satisfied = left_hand_side <= row.rhs
        else:
            raise ValueError("system contains an unsupported relation")
        if not is_satisfied:
            violated.append(row.constraint_id)
    return tuple(violated)
```

## Example

```python
system = build_binary_assignment_system(
    slot_count=3,
    category_count=2,
    option_count=2,
    weights=(2, 3, 1, 4),
    capacities=(4, 4),
)


def assignment_for(*coordinates: tuple[int, int, int]) -> tuple[int, ...]:
    selected = tuple(
        ((slot * system.category_count) + category) * system.option_count + option
        for slot, category, option in coordinates
    )
    return tuple(int(variable in selected) for variable in range(system.variable_count))


valid = assignment_for((0, 0, 0), (1, 1, 0), (2, 0, 0))
missing_slot = assignment_for((0, 0, 0), (1, 1, 0))
over_capacity = assignment_for((0, 0, 0), (1, 1, 0), (2, 0, 1))
adjacent_category = assignment_for((0, 0, 0), (1, 0, 0), (2, 1, 0))
row_ids = tuple(row.constraint_id for row in system.rows)

assert (
    row_ids,
    evaluate_binary_assignment(system, valid),
    evaluate_binary_assignment(system, missing_slot),
    evaluate_binary_assignment(system, over_capacity),
    evaluate_binary_assignment(system, adjacent_category),
) == (
    (
        "slot:0:exactly-one",
        "slot:1:exactly-one",
        "slot:2:exactly-one",
        "category:0:capacity",
        "category:1:capacity",
        "adjacent:0:1:category:0",
        "adjacent:0:1:category:1",
        "adjacent:1:2:category:0",
        "adjacent:1:2:category:1",
    ),
    (),
    ("slot:2:exactly-one",),
    ("category:0:capacity",),
    ("adjacent:0:1:category:0",),
)
```

## Trade-offs and Limitations

The builder validates all dimensions, input lengths, integer values, bounded
products, weighted row totals, and exact variable, row, and sparse-term counts
before constructing any output records. It runs in `O(T)` time and space for
the generated term count `T`. Evaluation takes `O(T)` time and can return
`O(R)` IDs for `R` rows, in addition to its `O(V)` assignment input. Rows,
terms, and the enclosing system are frozen tuples of frozen records, and no
caller container is retained.

The fixed limits allow at most 128 slots, 64 categories, 64 options, 4,096
weight values, 32,768 variables, 8,192 rows, and 100,000 emitted terms; the
aggregate checks mean those maxima are not all usable together. Individual
weights and capacities are at most `2**31 - 1`, and a weighted row's maximum
left-hand side may not exceed `2**40 - 1`.

The evaluator is scoped to a `BinaryAssignmentSystem` returned by the builder.
It validates the complete assignment boundary and rejects relation values
outside the closed enum, but it does not reinterpret arbitrary caller-built
row graphs as another constraint language.

This is a fixed model, not a generic constraint DSL. It has no objectives,
solver invocation or adapters, callbacks, feasibility claims, NumPy use, or
file, network, and subprocess behavior. Generated systems may be infeasible;
the builder only represents and bounds them. Large cases need a separately
owned sparse representation and solver adapter rather than these Python object
rows.

## Related Snippets

<!-- catalog:related:start -->
- [Report Exact Capacity Deficits for Bounded Resource Profiles](report-exact-capacity-deficits-for-bounded-resource-profiles.md)
- [Partition Tagged Items into Minimum Stable Conflict-Free Groups](partition-tagged-items-into-minimum-stable-conflict-free-groups.md)
<!-- catalog:related:end -->
