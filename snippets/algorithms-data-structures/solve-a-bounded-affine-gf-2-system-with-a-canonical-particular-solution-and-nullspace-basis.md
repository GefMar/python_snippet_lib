---
title: "Solve a Bounded Affine GF(2) System with a Canonical Particular Solution and Nullspace Basis"
snippet_type: algorithm
use_cases:
  - data-transformation
  - testing
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - build-a-canonical-reduced-xor-basis-for-bounded-unsigned-integers.md
  - compute-exact-rational-reduced-row-echelon-form-and-rank-for-a-bounded-integer-matrix.md
  - find-one-deterministic-satisfying-assignment-for-bounded-2-cnf.md
---

# Solve a Bounded Affine GF(2) System with a Canonical Particular Solution and Nullspace Basis

## Idea and Problem

Describe every solution of a bounded binary linear system using one particular bit mask and an ordered nullspace basis.

Each equation stores its coefficients as bits in one Python integer and its
right-hand side as `0` or `1`. Row addition over GF(2) is therefore integer
XOR. Eliminating pivot columns from every other row produces reduced row
echelon form without division because the only nonzero field element is one.

Setting every free variable to zero gives a reproducible particular solution.
Setting one free variable at a time gives basis vectors for the homogeneous
nullspace, so every solution is `particular XOR` a subset of those vectors.

## When to Use

Use this representation for bounded parity constraints, binary incidence
models, small coding experiments, or test oracles that need both an
inconsistency result and a compact description of all satisfying assignments.
Variable `i` is bit `i`, which makes evaluation and solution enumeration
direct.

Use a sparse finite-field library for much larger systems. Use a Boolean SAT
solver when constraints include disjunction, negation structure, or nonlinear
terms, and use rational or floating linear algebra when coefficients are not
binary field elements.

## Implementation

```python
from dataclasses import dataclass
from itertools import combinations
from random import Random

_MAX_GF2_VARIABLES = 256
_MAX_GF2_EQUATIONS = 1_024


@dataclass(frozen=True, slots=True)
class Gf2AffineSolution:
    rank: int
    pivot_columns: tuple[int, ...]
    free_columns: tuple[int, ...]
    particular: int
    nullspace_basis: tuple[int, ...]


def solve_affine_gf2_system(
    variable_count: int,
    equations: tuple[tuple[int, int], ...],
) -> Gf2AffineSolution | None:
    """Return a canonical affine solution description, or None if inconsistent."""
    if type(variable_count) is not int:
        raise TypeError("variable_count must be an exact integer")
    if not 1 <= variable_count <= _MAX_GF2_VARIABLES:
        raise ValueError("variable_count is outside 1..256")
    if type(equations) is not tuple:
        raise TypeError("equations must be an exact tuple")
    if len(equations) > _MAX_GF2_EQUATIONS:
        raise ValueError("equation count exceeds 1,024")

    mask_limit = 1 << variable_count
    rows: list[list[int]] = []
    for index, equation in enumerate(equations):
        if type(equation) is not tuple or len(equation) != 2:
            raise TypeError(f"equations[{index}] must be an exact pair")
        coefficient_mask, rhs_bit = equation
        if type(coefficient_mask) is not int:
            raise TypeError(f"equations[{index}][0] must be an exact integer")
        if not 0 <= coefficient_mask < mask_limit:
            raise ValueError(f"equations[{index}][0] exceeds the variable width")
        if type(rhs_bit) is not int:
            raise TypeError(f"equations[{index}][1] must be an exact integer")
        if rhs_bit not in (0, 1):
            raise ValueError(f"equations[{index}][1] must be zero or one")
        rows.append([coefficient_mask, rhs_bit])

    pivot_columns: list[int] = []
    pivot_row = 0
    for column in range(variable_count):
        source_row = next(
            (
                row_index
                for row_index in range(pivot_row, len(rows))
                if (rows[row_index][0] >> column) & 1
            ),
            None,
        )
        if source_row is None:
            continue

        rows[pivot_row], rows[source_row] = rows[source_row], rows[pivot_row]
        pivot_mask, pivot_rhs = rows[pivot_row]
        for row_index, row in enumerate(rows):
            if row_index != pivot_row and ((row[0] >> column) & 1):
                row[0] ^= pivot_mask
                row[1] ^= pivot_rhs

        pivot_columns.append(column)
        pivot_row += 1
        if pivot_row == len(rows):
            break

    if any(coefficient_mask == 0 and rhs_bit == 1 for coefficient_mask, rhs_bit in rows):
        return None

    pivot_set = set(pivot_columns)
    free_columns = tuple(
        column for column in range(variable_count) if column not in pivot_set
    )

    particular = 0
    for row_index, column in enumerate(pivot_columns):
        if rows[row_index][1]:
            particular |= 1 << column

    basis: list[int] = []
    for free_column in free_columns:
        vector = 1 << free_column
        for row_index, pivot_column in enumerate(pivot_columns):
            if (rows[row_index][0] >> free_column) & 1:
                vector |= 1 << pivot_column
        basis.append(vector)

    return Gf2AffineSolution(
        rank=len(pivot_columns),
        pivot_columns=tuple(pivot_columns),
        free_columns=free_columns,
        particular=particular,
        nullspace_basis=tuple(basis),
    )
```

## Example

```python
def satisfies_equations(
    assignment: int,
    equations: tuple[tuple[int, int], ...],
) -> bool:
    return all((assignment & mask).bit_count() % 2 == rhs for mask, rhs in equations)


def described_assignments(solution: Gf2AffineSolution) -> set[int]:
    assignments: set[int] = set()
    basis = solution.nullspace_basis
    for selector in range(1 << len(basis)):
        assignment = solution.particular
        for index, vector in enumerate(basis):
            if (selector >> index) & 1:
                assignment ^= vector
        assignments.add(assignment)
    return assignments


checked_systems = 0
for variable_count in range(1, 4):
    possible_equations = tuple(
        (mask, rhs)
        for mask in range(1 << variable_count)
        for rhs in (0, 1)
    )
    for equation_count in range(4):
        for equations in combinations(possible_equations, equation_count):
            expected = {
                assignment
                for assignment in range(1 << variable_count)
                if satisfies_equations(assignment, equations)
            }
            actual = solve_affine_gf2_system(variable_count, equations)
            if expected:
                assert actual is not None
                assert described_assignments(actual) == expected
                assert solve_affine_gf2_system(
                    variable_count,
                    tuple(reversed(equations)) + equations,
                ) == actual
            else:
                assert actual is None
            checked_systems += 1

rng = Random(23)
for _ in range(500):
    variable_count = rng.randrange(1, 9)
    equations = tuple(
        (rng.randrange(1 << variable_count), rng.randrange(2))
        for _ in range(rng.randrange(13))
    )
    expected = {
        assignment
        for assignment in range(1 << variable_count)
        if satisfies_equations(assignment, equations)
    }
    actual = solve_affine_gf2_system(variable_count, equations)
    assert (actual is None) == (not expected)
    if actual is not None:
        assert described_assignments(actual) == expected

identity = tuple((1 << column, column & 1) for column in range(_MAX_GF2_VARIABLES))
maximum_system = identity + ((0, 0),) * (_MAX_GF2_EQUATIONS - len(identity))
maximum_solution = solve_affine_gf2_system(_MAX_GF2_VARIABLES, maximum_system)
contradiction = solve_affine_gf2_system(3, ((0, 1),))

for invalid_equations in (((8, 0),), ((0, 2),)):
    try:
        solve_affine_gf2_system(3, invalid_equations)
    except ValueError:
        pass
    else:
        raise AssertionError("accepted an out-of-profile equation")

assert (
    checked_systems,
    contradiction,
    maximum_solution is not None and maximum_solution.rank,
    maximum_solution is not None and len(maximum_solution.nullspace_basis),
    maximum_solution is not None
    and satisfies_equations(maximum_solution.particular, maximum_system),
) == (805, None, 256, 0, True)
```

## Trade-offs and Limitations

Elimination performs `O(VE)` bounded Python-integer row tests or XOR updates
for `V` variables and `E` equations, and keeps `O(E + V)` Python integers.
Those are not constant-cost machine-word operations: their bit work grows with
the declared row width, which is capped at 256 bits here.

The reduced form, free-zero particular solution, and ascending-free-column
basis are independent of equation declaration order and duplicate rows. They
are canonical for this bit-numbering convention, but a different variable
ordering produces a different representation of the same abstract system.

The solver materializes all equations. It does not provide sparse or
incremental elimination, weighted optimization over satisfying assignments,
solutions over larger finite fields, approximate numerical solving,
cryptographic constant-time behavior, or explanations for an inconsistent
subset.

## Related Snippets

<!-- catalog:related:start -->
- [Build a Canonical Reduced XOR Basis for Bounded Unsigned Integers](build-a-canonical-reduced-xor-basis-for-bounded-unsigned-integers.md)
- [Compute Exact Rational Reduced Row-Echelon Form and Rank for a Bounded Integer Matrix](compute-exact-rational-reduced-row-echelon-form-and-rank-for-a-bounded-integer-matrix.md)
- [Find One Deterministic Satisfying Assignment for Bounded 2-CNF](find-one-deterministic-satisfying-assignment-for-bounded-2-cnf.md)
<!-- catalog:related:end -->
