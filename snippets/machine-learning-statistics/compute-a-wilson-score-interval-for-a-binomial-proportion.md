---
title: "Compute a Wilson Score Interval for a Binomial Proportion"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - calculate-a-symmetrically-trimmed-mean.md
  - ../data-processing/check-a-value-against-an-asymmetric-tolerance-band.md
---

# Compute a Wilson Score Interval for a Binomial Proportion

## Idea and Problem

Estimate a two-sided confidence interval for a binomial proportion with Wilson score bounds instead of a symmetric Wald interval.

The observed rate alone hides sample-size uncertainty, and the usual
`p ± z * standard_error` approximation behaves poorly near zero or one.
Inverting the score test produces asymmetric bounds that remain inside the
probability range after a small floating-point clamp.

## When to Use

Use this algorithm for a fixed set of independent Bernoulli trials when you
need an approximate interval for one unknown success probability. Supply the
confidence level chosen before inspecting the result. Use a statistics package
or a domain-specific model for weighted observations, clustered data,
overdispersion, repeated peeking, or simultaneous comparisons across groups.

## Implementation

```python
import math
from dataclasses import dataclass
from statistics import NormalDist


@dataclass(frozen=True, slots=True)
class ProportionInterval:
    lower: float
    upper: float


def wilson_score_interval(
    successes: int,
    total: int,
    *,
    confidence: int | float = 0.95,
) -> ProportionInterval:
    if type(successes) is not int:
        raise TypeError("successes must be an integer")
    if type(total) is not int:
        raise TypeError("total must be an integer")
    if total <= 0:
        raise ValueError("total must be positive")
    if not 0 <= successes <= total:
        raise ValueError("successes must be between zero and total")
    if isinstance(confidence, bool) or not isinstance(confidence, (int, float)):
        raise TypeError("confidence must be numeric")

    try:
        confidence_value = float(confidence)
    except OverflowError as exc:
        raise ValueError("confidence must be finite") from exc
    if not math.isfinite(confidence_value) or not 0.0 < confidence_value < 1.0:
        raise ValueError("confidence must be finite and between zero and one")

    tail_probability = (1.0 - confidence_value) / 2.0
    z = -NormalDist().inv_cdf(tail_probability)
    estimate = successes / total
    inverse_total = 1 / total
    z_squared = z * z
    denominator = 1.0 + z_squared * inverse_total
    center = (estimate + 0.5 * z_squared * inverse_total) / denominator
    radius = (
        z
        * math.sqrt(
            (estimate * (1.0 - estimate) + 0.25 * z_squared * inverse_total)
            * inverse_total
        )
        / denominator
    )
    return ProportionInterval(
        lower=0.0 if successes == 0 else max(0.0, center - radius),
        upper=1.0 if successes == total else min(1.0, center + radius),
    )
```

## Example

```python
middle = wilson_score_interval(50, 100)
none_observed = wilson_score_interval(0, 20)
all_observed = wilson_score_interval(20, 20)
narrower = wilson_score_interval(50, 100, confidence=0.80)

try:
    wilson_score_interval(True, 10)
except TypeError:
    boolean_count_rejected = True
else:
    boolean_count_rejected = False

assert (
    math.isclose(middle.lower, 0.4038315303659956),
    math.isclose(middle.upper, 0.5961684696340044),
    none_observed.lower,
    all_observed.upper,
    narrower.upper - narrower.lower < middle.upper - middle.lower,
    boolean_count_rejected,
) == (True, True, 0.0, 1.0, True, True)
```

## Trade-offs and Limitations

Wilson bounds are an approximate frequentist interval under a binomial model;
they are not a posterior probability, prediction interval, effect-size rule,
or proof that one group is better than another. The implementation rejects
zero trials instead of inventing an interval without observations. Extremely
high confidence produces wide bounds and can magnify floating-point limits.
Calling the function repeatedly during data collection or across many groups
requires a sequential or multiple-comparison design that this snippet does not
provide. Use an exact, Bayesian, clustered, or overdispersed model when its
assumptions better match the data-generating process.

## Related Snippets

<!-- catalog:related:start -->
- [Calculate a Symmetrically Trimmed Mean](calculate-a-symmetrically-trimmed-mean.md)
- [Check a Value Against an Asymmetric Tolerance Band](../data-processing/check-a-value-against-an-asymmetric-tolerance-band.md)
<!-- catalog:related:end -->
