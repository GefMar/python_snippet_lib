---
title: "Measure Drift Between Two Fixed-Bin Count Distributions with PSI"
snippet_type: algorithm
use_cases:
  - observability
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../observability-operations/count-values-in-fixed-upper-bound-bins.md
  - compute-a-wilson-score-interval-for-a-binomial-proportion.md
  - ../data-processing/check-a-value-against-an-asymmetric-tolerance-band.md
---

# Measure Drift Between Two Fixed-Bin Count Distributions with PSI

## Idea and Problem

Calculate Population Stability Index from two bounded count vectors that share one fixed bin definition while making zero-support behavior explicit.

The function normalizes each vector and sums the symmetric per-bin divergence
terms. Bins empty in both samples contribute zero; a bin present on only one
side returns infinity instead of hiding a support change behind an arbitrary
epsilon. Histogram construction, smoothing, and alert policy remain separate
decisions.

## When to Use

Use PSI as one descriptive drift signal when reference and current observations
were counted with exactly the same pre-established bin edges. Both samples need
positive total counts, and their bounded integer totals must be meaningful for
the same population and measurement process.

Use a statistical test or uncertainty model when significance is the question,
and inspect operational and model metrics before attributing impact. If sparse
bins need smoothing, define and validate that policy outside this function so
the reported value cannot silently change with an undocumented constant.

## Implementation

```python
import math
from collections.abc import Iterable
from itertools import islice


_MAX_BINS = 1_024
_MAX_TOTAL_COUNT = 2**53


def _bounded_counts(values: Iterable[int], *, name: str) -> tuple[int, ...]:
    if isinstance(values, (str, bytes)):
        raise TypeError(f"{name} must be a non-text iterable")
    snapshot = tuple(islice(values, _MAX_BINS + 1))
    if not 2 <= len(snapshot) <= _MAX_BINS:
        raise ValueError(f"{name} bin count is outside the supported range")

    total = 0
    for value in snapshot:
        if isinstance(value, bool) or not isinstance(value, int):
            raise TypeError(f"{name} must contain integer counts")
        if value < 0:
            raise ValueError(f"{name} must not contain negative counts")
        total += value
        if total > _MAX_TOTAL_COUNT:
            raise ValueError(f"{name} total exceeds the supported range")
    if total == 0:
        raise ValueError(f"{name} must contain at least one observation")
    return snapshot


def population_stability_index(
    reference_counts: Iterable[int],
    current_counts: Iterable[int],
) -> float:
    reference = _bounded_counts(reference_counts, name="reference_counts")
    current = _bounded_counts(current_counts, name="current_counts")
    if len(reference) != len(current):
        raise ValueError("count vectors must use the same number of bins")

    reference_total = sum(reference)
    current_total = sum(current)
    terms: list[float] = []
    for reference_count, current_count in zip(reference, current, strict=True):
        if reference_count == current_count == 0:
            continue
        if reference_count == 0 or current_count == 0:
            return math.inf
        reference_share = reference_count / reference_total
        current_share = current_count / current_total
        terms.append(
            (current_share - reference_share)
            * math.log(current_share / reference_share)
        )
    return math.fsum(terms)
```

## Example

```python
unchanged = population_stability_index((40, 30, 20, 10), (400, 300, 200, 100))
shifted = population_stability_index((40, 30, 20, 10), (20, 30, 30, 20))
new_support = population_stability_index((50, 50, 0), (40, 40, 20))

assert (
    math.isclose(unchanged, 0.0, abs_tol=1e-15),
    shifted > 0.0,
    new_support,
) == (True, True, math.inf)
```

## Trade-offs and Limitations

PSI depends strongly on bin edges, sample size, missing-value treatment, and
the chosen reference period. Recomputing bins independently destroys the
comparison. Sparse bins can make the unsmoothed value infinite; smoothing can
be useful, but it changes the metric and must be explicit and stable.

The result is symmetric and non-negative for valid finite inputs, yet it is not
a probability, causal explanation, significance test, or model-quality score.
There is no universal warning threshold. Monitor the value with sample counts,
data-quality checks, and domain-specific consequences rather than turning one
borrowed cutoff into an automatic incident.

## Related Snippets

<!-- catalog:related:start -->
- [Count Values in Fixed Upper-Bound Bins](../observability-operations/count-values-in-fixed-upper-bound-bins.md)
- [Compute a Wilson Score Interval for a Binomial Proportion](compute-a-wilson-score-interval-for-a-binomial-proportion.md)
- [Check a Value Against an Asymmetric Tolerance Band](../data-processing/check-a-value-against-an-asymmetric-tolerance-band.md)
<!-- catalog:related:end -->
