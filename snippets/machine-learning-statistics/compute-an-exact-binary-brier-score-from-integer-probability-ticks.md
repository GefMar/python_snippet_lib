---
title: "Compute an Exact Binary Brier Score from Integer Probability Ticks"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - compute-exact-binary-roc-auc-with-tied-integer-scores.md
  - build-an-exact-bounded-multiclass-confusion-matrix.md
  - select-a-forecast-vector-only-when-it-beats-a-frozen-baseline.md
---

# Compute an Exact Binary Brier Score from Integer Probability Ticks

## Idea and Problem

Measure the mean squared error of bounded binary probability forecasts exactly when every probability uses one declared integer scale.

For each observation, zero ticks represents probability zero and `scale`
ticks represents probability one. Squaring the difference between the forecast
and the Boolean outcome in the same units gives an integer contribution.
Summing those contributions before constructing one `Fraction` produces the
reduced Brier score without floating-point rounding.

## When to Use

Use this algorithm when a non-empty evaluation batch contains binary outcomes
and probability forecasts already quantized to one common scale. It is useful
for reproducible checks, exact regression assertions, and comparisons where
small floating-point representation differences would be distracting. Lower
scores are better, with zero for perfect forecasts and one for forecasts that
are completely wrong with full confidence.

Keep the observation population and probability scale fixed when comparing
scores. Use a statistical metrics library when observations need weights,
multiclass scoring, uncertainty estimates, calibration decomposition, or
integration with a larger evaluation workflow.

## Implementation

```python
from fractions import Fraction

_MAX_OBSERVATIONS = 100_000
_MAX_SCALE = 10**9


def compute_exact_binary_brier_score(
    observations: tuple[tuple[int, bool], ...],
    scale: int,
) -> Fraction:
    """Return the exact mean squared error of scaled binary forecasts."""
    if type(observations) is not tuple:
        raise TypeError("observations must be an exact tuple")
    if not 1 <= len(observations) <= _MAX_OBSERVATIONS:
        raise ValueError(
            f"observations must contain between 1 and {_MAX_OBSERVATIONS} items"
        )
    if type(scale) is not int:
        raise TypeError("scale must be an exact non-boolean integer")
    if not 1 <= scale <= _MAX_SCALE:
        raise ValueError(f"scale must be between 1 and {_MAX_SCALE}")

    total_squared_error = 0
    for index, observation in enumerate(observations):
        if type(observation) is not tuple:
            raise TypeError(f"observations[{index}] must be an exact tuple")
        if len(observation) != 2:
            raise ValueError(f"observations[{index}] must contain exactly two items")

        tick, outcome = observation
        if type(tick) is not int:
            raise TypeError(
                f"observations[{index}].tick must be an exact non-boolean integer"
            )
        if not 0 <= tick <= scale:
            raise ValueError(
                f"observations[{index}].tick must be between 0 and scale"
            )
        if type(outcome) is not bool:
            raise TypeError(f"observations[{index}].outcome must be an exact boolean")

        target_tick = scale if outcome else 0
        error = tick - target_tick
        total_squared_error += error * error

    return Fraction(
        total_squared_error,
        len(observations) * scale * scale,
    )
```

## Example

```python
def reference_brier_score(
    observations: tuple[tuple[int, bool], ...],
    scale: int,
) -> Fraction:
    squared_errors = tuple(
        (Fraction(tick, scale) - int(outcome)) ** 2
        for tick, outcome in observations
    )
    return sum(squared_errors, start=Fraction()) / len(observations)


cases = (
    (((0, False), (100, True)), 100),
    (((100, False), (0, True)), 100),
    (((50, False), (50, True)), 100),
    (((25, False), (75, True), (40, True)), 100),
)

scores = tuple(
    compute_exact_binary_brier_score(observations, scale)
    for observations, scale in cases
)
for observations, scale in cases:
    assert compute_exact_binary_brier_score(
        observations,
        scale,
    ) == reference_brier_score(observations, scale)

original = ((10, False), (65, True), (90, True))
complement = tuple((100 - tick, not outcome) for tick, outcome in original)
complement_invariant = compute_exact_binary_brier_score(
    original,
    100,
) == compute_exact_binary_brier_score(complement, 100)

assert (scores, complement_invariant) == (
    (Fraction(0), Fraction(1), Fraction(1, 4), Fraction(97, 600)),
    True,
)
```

## Trade-offs and Limitations

The function performs `O(n)` exact-integer arithmetic operations and uses
`O(1)` auxiliary state beyond the supplied observations and returned value.
Those operations are not constant-cost for unbounded integers: multiplication,
addition, and the final fraction reduction depend on operand bit lengths. Under
the declared limits, both the unreduced numerator and denominator are at most
`100_000 * (10**9)**2`, which fits in 77 bits.

The score combines probability calibration and confidence into one mean
squared error; it does not explain which component caused a difference. The
function accepts only exact tuples, one shared integer scale, unweighted binary
outcomes, and no missing values. It does not rank forecasts, select a decision
threshold, decompose calibration, estimate uncertainty, perform inference, or
choose between models.

## Related Snippets

<!-- catalog:related:start -->
- [Compute Exact Binary ROC AUC with Tied Integer Scores](compute-exact-binary-roc-auc-with-tied-integer-scores.md)
- [Build an Exact Bounded Multiclass Confusion Matrix](build-an-exact-bounded-multiclass-confusion-matrix.md)
- [Select a Forecast Vector Only When It Beats a Frozen Baseline](select-a-forecast-vector-only-when-it-beats-a-frozen-baseline.md)
<!-- catalog:related:end -->
