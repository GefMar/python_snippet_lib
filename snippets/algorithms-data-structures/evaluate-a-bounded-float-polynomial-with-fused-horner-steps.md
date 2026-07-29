---
title: "Evaluate a Bounded Float Polynomial with Fused Horner Steps"
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
  - ../machine-learning-statistics/compute-a-bounded-numerically-stable-log-sum-exp-for-finite-floats.md
  - ../machine-learning-statistics/fit-an-exact-ordinary-least-squares-line-to-bounded-integer-points.md
---

# Evaluate a Bounded Float Polynomial with Fused Horner Steps

## Idea and Problem

Evaluate a finite float polynomial with Horner's method while rounding each multiply-add step only once.

Ordinary `accumulator * x + coefficient` evaluation rounds the multiplication
before the addition. `math.fma` computes that pair as though it had infinite
precision and then performs one rounding, which can retain information that
the two-operation form loses. Horner's arrangement also needs only one fused
step per coefficient after the leading term.

## When to Use

Use this algorithm for a small, already ordered tuple of binary64 coefficients
when the Python 3.14 fused-operation semantics are the desired numeric
contract. Coefficients are supplied in descending degree order, and every
input and intermediate result must remain finite.

Use `Decimal`, `Fraction`, interval arithmetic, or a numerical library when a
different numeric model, explicit error bounds, vectorized evaluation, or
several related polynomial operations are required. Fused steps improve local
rounding but do not make the complete polynomial exact.

## Implementation

```python
import math
from fractions import Fraction
from random import Random

_MAX_COEFFICIENTS = 256


def evaluate_polynomial_fused(
    coefficients: tuple[float, ...],
    x: float,
) -> float:
    if type(coefficients) is not tuple:
        raise TypeError("coefficients must be an exact tuple")
    if not 1 <= len(coefficients) <= _MAX_COEFFICIENTS:
        raise ValueError("coefficient count is outside the supported range")
    if type(x) is not float:
        raise TypeError("x must be an exact float")
    if not math.isfinite(x):
        raise ValueError("x must be finite")

    for coefficient in coefficients:
        if type(coefficient) is not float:
            raise TypeError("coefficients must contain exact floats")
        if not math.isfinite(coefficient):
            raise ValueError("coefficients must be finite")

    result = coefficients[0]
    for index in range(1, len(coefficients)):
        coefficient = coefficients[index]
        try:
            result = math.fma(result, x, coefficient)
        except OverflowError as error:
            raise ValueError("fused polynomial evaluation overflowed") from error
        if not math.isfinite(result):
            raise ValueError("fused polynomial evaluation became non-finite")
    return result
```

## Example

```python
def exact_step_reference(
    coefficients: tuple[float, ...],
    x: float,
) -> float:
    result = coefficients[0]
    for coefficient in coefficients[1:]:
        exact_step = (
            Fraction.from_float(result) * Fraction.from_float(x)
            + Fraction.from_float(coefficient)
        )
        result = float(exact_step)
    return result


coefficients = (1e16, -1e16)
point = 1.0000000000000002
fused = evaluate_polynomial_fused(coefficients, point)
separate = coefficients[0] * point + coefficients[1]

random = Random(2026)
for _ in range(100):
    sample_coefficients = tuple(
        random.uniform(-100.0, 100.0)
        for _ in range(random.randint(1, 8))
    )
    sample_x = random.uniform(-2.0, 2.0)
    assert evaluate_polynomial_fused(
        sample_coefficients,
        sample_x,
    ) == exact_step_reference(sample_coefficients, sample_x)

assert fused == 2.220446049250313
assert separate == 2.0
assert evaluate_polynomial_fused((3.5,), -100.0) == 3.5
```

## Trade-offs and Limitations

The algorithm takes linear time and constant additional state. A fused
multiply-add may be faster on hardware with direct support, but this recipe
does not promise a performance improvement on every Python build or platform.
Its primary contract is one correctly rounded fused operation per Horner step.

The result still accumulates rounding error across steps and may be
ill-conditioned for the chosen coefficient basis and input. This helper does
not reorder coefficients, construct or interpolate polynomials, compute
derivatives, accept complex or exact numeric types, vectorize work, use a
BLAS/GPU backend, compensate across multiple passes, or produce interval or
forward-error bounds.

## Related Snippets

<!-- catalog:related:start -->
- [Interpolate a Global Polynomial Exactly from Bounded Integer Points](interpolate-a-global-polynomial-exactly-from-bounded-integer-points.md)
- [Compute a Bounded Numerically Stable Log-Sum-Exp for Finite Floats](../machine-learning-statistics/compute-a-bounded-numerically-stable-log-sum-exp-for-finite-floats.md)
- [Fit an Exact Ordinary Least-Squares Line to Bounded Integer Points](../machine-learning-statistics/fit-an-exact-ordinary-least-squares-line-to-bounded-integer-points.md)
<!-- catalog:related:end -->
