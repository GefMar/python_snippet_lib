---
title: "Compute a Distant Linear-Recurrence Term Modulo an Integer by Polynomial Reduction"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - interpolate-a-global-polynomial-exactly-from-bounded-integer-points.md
  - compute-an-exact-integer-matrix-determinant-with-bareiss-elimination.md
  - combine-a-bounded-system-of-possibly-non-coprime-integer-congruences.md
---

# Compute a Distant Linear-Recurrence Term Modulo an Integer by Polynomial Reduction

## Idea and Problem

Compute one far-away term of a bounded constant-coefficient integer recurrence without generating every preceding term.

For an order-`k` recurrence, its characteristic relation replaces `x**k`
with a weighted sum of lower powers. Binary exponentiation can therefore build
`x**n` while reducing every intermediate polynomial to degree below `k`. The
remaining coefficient of each `x**i` is the multiplier of initial term
`a[i]`.

All arithmetic is reduced modulo one declared positive integer. This keeps the
answer canonical and prevents the usually rapid growth of exact recurrence
terms while still allowing indexes far beyond an iterative scan.

## When to Use

Use this algorithm when a fixed recurrence has at most 64 initial terms, only
one distant modular value is needed, and the relation is
`a[n] = sum(c[j] * a[n - 1 - j] for j in range(k))`. Coefficient `c[0]`
therefore multiplies the immediately preceding term. Signed initial values and
coefficients are accepted and normalized by the modulus.

Prefer a direct loop for nearby indexes or when every intermediate term is
needed. Matrix exponentiation can be easier to extend when several coupled
state variables evolve together. Use a specialized algebra or number-theory
library for much larger orders, several moduli, polynomial coefficients, or
non-constant recurrence rules.

## Implementation

```python
_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_RECURRENCE_ORDER = 64
_MAX_RECURRENCE_INDEX = 10**18


def _validate_recurrence_values(values: object, *, field: str) -> tuple[int, ...]:
    if type(values) is not tuple:
        raise TypeError(f"{field} must be an exact tuple")
    for position, value in enumerate(values):
        if type(value) is not int:
            raise TypeError(f"{field}[{position}] must be an exact integer")
        if not _MIN_INT64 <= value <= _MAX_INT64:
            raise ValueError(f"{field}[{position}] is outside the signed 64-bit range")
    return values


def linear_recurrence_term_modulo(
    initial: tuple[int, ...],
    coefficients: tuple[int, ...],
    index: int,
    modulus: int,
) -> int:
    """Return a[index] modulo modulus under newest-term-first coefficients."""
    if type(initial) is not tuple:
        raise TypeError("initial must be an exact tuple")
    if type(coefficients) is not tuple:
        raise TypeError("coefficients must be an exact tuple")
    order = len(initial)
    if not 1 <= order <= _MAX_RECURRENCE_ORDER:
        raise ValueError("recurrence order is outside the supported range")
    if len(coefficients) != order:
        raise ValueError("initial and coefficients must have equal lengths")
    checked_initial = _validate_recurrence_values(initial, field="initial")
    checked_coefficients = _validate_recurrence_values(coefficients, field="coefficients")
    if type(index) is not int:
        raise TypeError("index must be an exact non-boolean integer")
    if not 0 <= index <= _MAX_RECURRENCE_INDEX:
        raise ValueError("index is outside the supported range")
    if type(modulus) is not int:
        raise TypeError("modulus must be an exact non-boolean integer")
    if not 2 <= modulus <= _MAX_INT64:
        raise ValueError("modulus is outside the supported range")

    reduced_coefficients = tuple(value % modulus for value in checked_coefficients)

    def multiply_and_reduce(
        left: tuple[int, ...],
        right: tuple[int, ...],
    ) -> tuple[int, ...]:
        product = [0] * (2 * order - 1)
        for left_degree, left_value in enumerate(left):
            if left_value == 0:
                continue
            for right_degree, right_value in enumerate(right):
                product[left_degree + right_degree] = (
                    product[left_degree + right_degree] + left_value * right_value
                ) % modulus

        for degree in range(2 * order - 2, order - 1, -1):
            leading = product[degree]
            if leading == 0:
                continue
            for offset, coefficient in enumerate(reduced_coefficients):
                target_degree = degree - 1 - offset
                product[target_degree] = (product[target_degree] + leading * coefficient) % modulus
        return tuple(product[:order])

    accumulated = (1,) + (0,) * (order - 1)
    if order == 1:
        power = (reduced_coefficients[0],)
    else:
        power = (0, 1) + (0,) * (order - 2)

    remaining_index = index
    while remaining_index:
        if remaining_index & 1:
            accumulated = multiply_and_reduce(accumulated, power)
        remaining_index >>= 1
        if remaining_index:
            power = multiply_and_reduce(power, power)

    return (
        sum(
            multiplier * (value % modulus)
            for multiplier, value in zip(accumulated, checked_initial, strict=True)
        )
        % modulus
    )
```

## Example

```python
def recurrence_term_iteratively(
    initial: tuple[int, ...],
    coefficients: tuple[int, ...],
    index: int,
    modulus: int,
) -> int:
    history = [value % modulus for value in initial]
    for term_index in range(len(initial), index + 1):
        history.append(
            sum(
                coefficient * history[term_index - 1 - offset]
                for offset, coefficient in enumerate(coefficients)
            )
            % modulus
        )
    return history[index]


def multiply_square_matrices(
    left: tuple[tuple[int, ...], ...],
    right: tuple[tuple[int, ...], ...],
    modulus: int,
) -> tuple[tuple[int, ...], ...]:
    order = len(left)
    return tuple(
        tuple(
            sum(left[row][inner] * right[inner][column] for inner in range(order)) % modulus
            for column in range(order)
        )
        for row in range(order)
    )


def power_square_matrix(
    matrix: tuple[tuple[int, ...], ...],
    exponent: int,
    modulus: int,
) -> tuple[tuple[int, ...], ...]:
    order = len(matrix)
    result = tuple(tuple(int(row == column) for column in range(order)) for row in range(order))
    factor = matrix
    while exponent:
        if exponent & 1:
            result = multiply_square_matrices(result, factor, modulus)
        exponent >>= 1
        if exponent:
            factor = multiply_square_matrices(factor, factor, modulus)
    return result


def recurrence_term_by_companion_matrix(
    initial: tuple[int, ...],
    coefficients: tuple[int, ...],
    index: int,
    modulus: int,
) -> int:
    order = len(initial)
    if index < order:
        return initial[index] % modulus

    companion = (
        tuple(value % modulus for value in coefficients),
        *(tuple(int(column == row - 1) for column in range(order)) for row in range(1, order)),
    )
    powered = power_square_matrix(companion, index - order + 1, modulus)
    state = tuple(value % modulus for value in reversed(initial))
    return sum(powered[0][column] * state[column] for column in range(order)) % modulus


def exercise_small_recurrences() -> int:
    from itertools import product

    checked = 0
    for order in range(1, 4):
        candidates = tuple(product((-1, 0, 1), repeat=order))
        for initial in candidates:
            for coefficients in candidates:
                for modulus in (2, 5, 11):
                    for index in range(12):
                        expected = recurrence_term_iteratively(
                            initial,
                            coefficients,
                            index,
                            modulus,
                        )
                        assert (
                            linear_recurrence_term_modulo(
                                initial,
                                coefficients,
                                index,
                                modulus,
                            )
                            == expected
                        )
                        assert (
                            recurrence_term_by_companion_matrix(
                                initial,
                                coefficients,
                                index,
                                modulus,
                            )
                            == expected
                        )
                        checked += 1
    return checked


def raises(error_type: type[Exception], operation) -> bool:
    try:
        operation()
    except error_type:
        return True
    return False


huge_index = _MAX_RECURRENCE_INDEX
large_modulus = _MAX_INT64
huge_fibonacci = linear_recurrence_term_modulo((0, 1), (1, 1), huge_index, large_modulus)
maximum_order_value = linear_recurrence_term_modulo(
    tuple(range(_MAX_RECURRENCE_ORDER)),
    (1,) + (0,) * (_MAX_RECURRENCE_ORDER - 1),
    huge_index,
    large_modulus,
)

assert (
    exercise_small_recurrences(),
    huge_fibonacci,
    recurrence_term_by_companion_matrix((0, 1), (1, 1), huge_index, large_modulus),
    linear_recurrence_term_modulo((3,), (-2,), huge_index, large_modulus),
    3 * pow(-2, huge_index, large_modulus) % large_modulus,
    maximum_order_value,
    raises(ValueError, lambda: linear_recurrence_term_modulo((), (), 0, 2)),
    raises(ValueError, lambda: linear_recurrence_term_modulo((1,), (1, 2), 0, 2)),
    raises(ValueError, lambda: linear_recurrence_term_modulo((1,), (1,), -1, 2)),
    raises(ValueError, lambda: linear_recurrence_term_modulo((1,), (1,), 0, 1)),
    raises(TypeError, lambda: linear_recurrence_term_modulo((1,), (True,), 0, 2)),
    raises(TypeError, lambda: linear_recurrence_term_modulo((1,), (1,), True, 2)),
) == (
    29_484,
    huge_fibonacci,
    huge_fibonacci,
    3 * pow(-2, huge_index, large_modulus) % large_modulus,
    3 * pow(-2, huge_index, large_modulus) % large_modulus,
    63,
    True,
    True,
    True,
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

For order `k` and index `n`, polynomial multiplication and reduction use
`O(k^2 log n)` modular operations and `O(k)` auxiliary coefficients. Python
integer multiplication and remainder are not unit-cost operations, although
reducing after each update keeps operands tied to the declared modulus rather
than to the usually much larger unreduced sequence terms.

The result is always the representative from zero through `modulus - 1`.
Changing the modulus changes the sequence being observed but not the declared
integer recurrence before reduction. The implementation validates every
input before returning even when the requested index lies among the initial
terms.

The function handles one homogeneous scalar recurrence with constant
coefficients. It does not return intervening terms, recover missing
coefficients, handle index-dependent or nonlinear rules, accept matrices or
polynomials as terms, work with negative indexes, combine several moduli, or
provide cryptographic guarantees.

## Related Snippets

<!-- catalog:related:start -->
- [Interpolate a Global Polynomial Exactly from Bounded Integer Points](interpolate-a-global-polynomial-exactly-from-bounded-integer-points.md)
- [Compute an Exact Integer-Matrix Determinant with Bareiss Elimination](compute-an-exact-integer-matrix-determinant-with-bareiss-elimination.md)
- [Combine a Bounded System of Possibly Non-Coprime Integer Congruences](combine-a-bounded-system-of-possibly-non-coprime-integer-congruences.md)
<!-- catalog:related:end -->
