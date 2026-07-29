---
title: "Select the Lower Weighted Median of Bounded Integer Observations"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - compute-an-exact-integer-median-and-unscaled-median-absolute-deviation.md
  - compute-exact-full-window-trailing-medians-for-bounded-integers.md
  - calculate-a-symmetrically-trimmed-mean.md
---

# Select the Lower Weighted Median of Bounded Integer Observations

## Idea and Problem

Select an exact lower median when each bounded integer observation carries a non-negative integer weight.

Integer weight can be understood as multiplicity without expanding the sample.
After equal values are grouped, the lower weighted median is the smallest value
whose cumulative weight reaches `(total_weight + 1) // 2`. This rule returns an
observed integer even when the total weight is even and the two central
expanded observations differ.

## When to Use

Use this algorithm for a finite in-memory set of exact integer observations
when weights represent counts or another non-negative integral importance
measure. It is useful when expanding large multiplicities would waste memory
and a reproducible lower-median tie policy is required.

Use a different definition when weights are fractional, an interpolated
quantile is required, or the upper central value should win an even-weight tie.
Choose a streaming quantile structure when observations cannot be retained and
sorted, accepting that its accuracy and update contract will be different.

## Implementation

```python
from itertools import groupby

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_WEIGHTED_OBSERVATIONS = 100_000
_MAX_TOTAL_WEIGHT = 10**18


def select_lower_weighted_median(
    observations: tuple[tuple[int, int], ...],
) -> int:
    """Return the lower weighted median as one original integer value."""
    if type(observations) is not tuple:
        raise TypeError("observations must be an exact tuple")
    if not 1 <= len(observations) <= _MAX_WEIGHTED_OBSERVATIONS:
        raise ValueError("observation count is outside the supported range")

    validated: list[tuple[int, int]] = []
    total_weight = 0
    for index, observation in enumerate(observations):
        if type(observation) is not tuple:
            raise TypeError(f"observations[{index}] must be an exact tuple")
        if len(observation) != 2:
            raise ValueError(f"observations[{index}] must contain two items")

        value, weight = observation
        if type(value) is not int:
            raise TypeError(f"observations[{index}][0] must be an exact integer")
        if not _MIN_INT64 <= value <= _MAX_INT64:
            raise ValueError(f"observations[{index}][0] is outside the signed 64-bit range")

        if type(weight) is not int:
            raise TypeError(f"observations[{index}][1] must be an exact integer")
        if not 0 <= weight <= _MAX_TOTAL_WEIGHT:
            raise ValueError(f"observations[{index}][1] is outside the supported range")
        validated.append((value, weight))
        total_weight += weight

    if not 1 <= total_weight <= _MAX_TOTAL_WEIGHT:
        raise ValueError("total weight is outside the supported range")

    threshold = (total_weight + 1) // 2
    cumulative_weight = 0
    ordered = sorted(validated, key=lambda item: item[0])
    for value, group in groupby(ordered, key=lambda item: item[0]):
        cumulative_weight += sum(weight for _, weight in group)
        if cumulative_weight >= threshold:
            return value

    raise AssertionError("positive total weight must reach its threshold")
```

## Example

```python
def expanded_lower_median(observations: tuple[tuple[int, int], ...]) -> int:
    from statistics import median_low

    expanded: list[int] = []
    for value, weight in observations:
        expanded.extend([value] * weight)
    return median_low(expanded)


def satisfies_lower_median_inequalities(
    observations: tuple[tuple[int, int], ...],
    candidate: int,
) -> bool:
    total = sum(weight for _, weight in observations)
    threshold = (total + 1) // 2
    weight_below = sum(weight for value, weight in observations if value < candidate)
    weight_through = sum(weight for value, weight in observations if value <= candidate)
    return weight_below < threshold <= weight_through


def exercise_weighted_median_examples() -> tuple[object, ...]:
    from itertools import permutations, product

    cases_checked = 0
    for values in product((-1, 0, 2), repeat=3):
        for weights in product(range(3), repeat=3):
            if sum(weights) == 0:
                continue
            observations = tuple(zip(values, weights, strict=True))
            selected = select_lower_weighted_median(observations)
            assert selected == expanded_lower_median(observations)
            assert satisfies_lower_median_inequalities(observations, selected)
            cases_checked += 1

    permutation_case = ((10, 0), (-5, 4), (10, 2), (99, 0), (8, 4))
    permuted_inputs = set(permutations(permutation_case))
    permuted_results = {
        select_lower_weighted_median(observations) for observations in permuted_inputs
    }
    large_invariant = all(
        satisfies_lower_median_inequalities(
            observations,
            select_lower_weighted_median(observations),
        )
        for observations in permuted_inputs
    )

    extreme_tie = select_lower_weighted_median(((_MIN_INT64, 1), (_MAX_INT64, 1)))
    maximum_weight = select_lower_weighted_median(((7, _MAX_TOTAL_WEIGHT),))

    invalid_inputs = (
        (((1, 0),), ValueError),
        (((True, 1),), TypeError),
        (((1, False),), TypeError),
        (((1, 2, 3),), ValueError),
        (((0, _MAX_TOTAL_WEIGHT), (1, 1)), ValueError),
    )
    rejected = 0
    for observations, expected_error in invalid_inputs:
        try:
            select_lower_weighted_median(observations)
        except expected_error:
            rejected += 1

    return (
        cases_checked,
        len(permuted_inputs),
        permuted_results,
        large_invariant,
        extreme_tie,
        maximum_weight,
        rejected,
    )


assert exercise_weighted_median_examples() == (
    702,
    120,
    {8},
    True,
    _MIN_INT64,
    7,
    5,
)
```

## Trade-offs and Limitations

For `N` declared observations, validation, sorting, and grouping take
`O(N log N)` time. The validated and sorted references use `O(N)` memory. The
total and cumulative weights remain exact Python integers while the declared
total cap bounds their values.

Zero-weight observations are validated but do not affect the selected value.
Grouping equal values makes duplicate layout irrelevant, and the lower rule
returns the first value that reaches the central threshold. For an even total
weight this intentionally differs from averaging the two central expanded
observations or choosing the upper one.

The function does not accept fractional or negative weights, interpolate
quantiles, calculate a weighted MAD, maintain a rolling window, mutate stored
state, or provide an approximate streaming summary.

## Related Snippets

<!-- catalog:related:start -->
- [Compute an Exact Integer Median and Unscaled Median Absolute Deviation](compute-an-exact-integer-median-and-unscaled-median-absolute-deviation.md)
- [Compute Exact Full-Window Trailing Medians for Bounded Integers](compute-exact-full-window-trailing-medians-for-bounded-integers.md)
- [Calculate a Symmetrically Trimmed Mean](calculate-a-symmetrically-trimmed-mean.md)
<!-- catalog:related:end -->
