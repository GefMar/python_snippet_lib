---
title: "Compute the Chance of At Least One Independent Success with Stable Logs"
snippet_type: recipe
use_cases:
  - data-transformation
tested_python:
  - "3.14"
dependencies: []
related:
  - compute-a-bounded-numerically-stable-log-sum-exp-for-finite-floats.md
---

# Compute the Chance of At Least One Independent Success with Stable Logs

## Idea and Problem

Compute the chance of at least one independent success without losing very small probabilities to subtraction.

The complement is the probability that all trials fail. Sum the log failure probabilities, then convert from the log with a function that computes the difference from one accurately near zero.

## When to Use

Use this for a finite iterable of independent Bernoulli probabilities, which need not be equal. It handles zero trials, impossible trials, and a certain success.

## Implementation

```python
from collections.abc import Iterable
from math import expm1, fsum, inf, log1p


def chance_of_any_success(probabilities: Iterable[float]) -> float:
    def log_failure(probability: float) -> float:
        if not 0.0 <= probability <= 1.0:
            raise ValueError("probabilities must lie between zero and one")
        return -inf if probability == 1.0 else log1p(-probability)

    total = fsum(log_failure(p) for p in probabilities)
    return 0.0 if total == 0.0 else -expm1(total)
```

## Example

```python
from math import isclose

tiny = chance_of_any_success((1e-18, 2e-18))
rejected_late_invalid_value = False
try:
    chance_of_any_success((1.0, float("nan")))
except ValueError:
    rejected_late_invalid_value = True

assert (
    isclose(tiny, 3e-18, rel_tol=1e-15),
    chance_of_any_success((0.5, 0.25)),
    chance_of_any_success(()),
    chance_of_any_success((0.4, 1.0, 0.2)),
    rejected_late_invalid_value,
) == (True, 0.625, 0.0, 1.0, True)
```

## Trade-offs and Limitations

Independence is essential; marginals do not determine this probability for correlated trials. This is a floating-point approximation. The answer can round to one when failure is extremely rare; retain the sum of log failures if the remaining failure probability is the value you need.

## Related Snippets

<!-- catalog:related:start -->
- [Compute a Bounded Numerically Stable Log-Sum-Exp for Finite Floats](compute-a-bounded-numerically-stable-log-sum-exp-for-finite-floats.md)
<!-- catalog:related:end -->
