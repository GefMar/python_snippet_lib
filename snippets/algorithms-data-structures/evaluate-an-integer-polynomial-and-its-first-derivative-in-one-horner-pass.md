---
title: "Evaluate an Integer Polynomial and Its First Derivative in One Horner Pass"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - evaluate-a-bounded-float-polynomial-with-fused-horner-steps.md
  - interpolate-a-global-polynomial-exactly-from-bounded-integer-points.md
---

# Evaluate an Integer Polynomial and Its First Derivative in One Horner Pass

## Idea and Problem

Evaluate an integer polynomial and its first derivative together without building powers or a second coefficient sequence.

Horner's rule builds the next partial polynomial as `x * p(x) + c`.
Its derivative is `p(x) + x * p'(x)`. Update the derivative first so that
it still sees the previous partial value. Then advance the value with the
usual Horner step. Both accumulators stay exact integers.

## When to Use

Use this when an exact evaluation and slope are needed at the same integer
point. Provide a finite iterable of integer coefficients in descending power
order, from the leading coefficient through the constant. An empty iterable
represents the zero polynomial, and leading zero coefficients are allowed.

## Implementation

```python
from collections.abc import Iterable


def value_and_derivative(coefficients: Iterable[int], x: int) -> tuple[int, int]:
    """Return (value, first derivative) at x for descending-power coefficients."""
    if type(x) is not int:
        raise TypeError("x must be an integer")

    value = derivative = 0
    for coefficient in coefficients:
        if type(coefficient) is not int:
            raise TypeError("coefficients must be integers")
        derivative = derivative * x + value
        value = value * x + coefficient
    return value, derivative
```

## Example

```python
# 3*x**3 - 2*x**2 + 5, whose derivative is 9*x**2 - 4*x.
coefficients = (3, -2, 0, 5)

assert (
    value_and_derivative(iter(coefficients), -2),
    value_and_derivative(coefficients, 0),
    value_and_derivative(coefficients, 2),
    value_and_derivative((7,), -2),
    value_and_derivative((), 9),
) == ((-27, 44), (5, 0), (21, 28), (7, 0), (0, 0))
```

## Trade-offs and Limitations

The loop performs a fixed number of arithmetic operations per coefficient
and uses two running accumulators. Python integers can grow with the degree,
coefficient sizes and evaluation point, increasing the cost of each step.

The result is the algebraic derivative evaluated at an integer point. Use an
appropriate numeric representation when evaluation points or coefficients
are fractional. Each call consumes its coefficient iterable.

## Related Snippets

<!-- catalog:related:start -->
- [Evaluate a Bounded Float Polynomial with Fused Horner Steps](evaluate-a-bounded-float-polynomial-with-fused-horner-steps.md)
- [Interpolate a Global Polynomial Exactly from Bounded Integer Points](interpolate-a-global-polynomial-exactly-from-bounded-integer-points.md)
<!-- catalog:related:end -->
