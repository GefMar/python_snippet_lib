---
title: "Compute an Exact Apdex Score from Already Classified Counts"
snippet_type: algorithm
use_cases:
  - data-transformation
  - observability
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - count-values-in-fixed-upper-bound-bins.md
  - compute-a-validated-delta-between-cumulative-histogram-snapshots.md
  - classify-progress-from-complete-bounded-counter-snapshots.md
---

# Compute an Exact Apdex Score from Already Classified Counts

## Idea and Problem

Compute the Apdex ratio exactly after every observation has already been classified as satisfied, tolerating, or frustrated.

The score gives full credit to satisfied observations, half credit to
tolerating observations, and no credit to frustrated observations. Keeping the
result as a `Fraction` preserves the exact ratio instead of introducing binary
floating-point rounding.

## When to Use

Use this calculation for one coherent, bounded observation window whose counts
are mutually exclusive and were produced under one stable classification
policy. It is useful when a caller already owns the timing threshold, error
rules, and observation window and needs a reproducible aggregate score.

Classify raw timings and failures before this boundary. Compare separate scores
only when their classification and window policies are compatible. A score by
itself does not establish what users experienced or whether a service met an
agreement or objective.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction

_MAX_INT64 = (1 << 63) - 1


@dataclass(frozen=True, slots=True)
class ExactApdexScore:
    total: int
    score: Fraction


def _validated_apdex_count(value: object, *, name: str) -> int:
    if type(value) is not int:
        raise TypeError(f"{name} must be an exact non-boolean integer")
    if not 0 <= value <= _MAX_INT64:
        raise ValueError(f"{name} is outside the supported range")
    return value


def exact_apdex_from_classified_counts(
    *,
    satisfied: int,
    tolerating: int,
    frustrated: int,
) -> ExactApdexScore:
    """Return the exact Apdex score for one already classified count set."""
    satisfied_count = _validated_apdex_count(satisfied, name="satisfied")
    tolerating_count = _validated_apdex_count(tolerating, name="tolerating")
    frustrated_count = _validated_apdex_count(frustrated, name="frustrated")

    total = satisfied_count + tolerating_count + frustrated_count
    if not 1 <= total <= _MAX_INT64:
        raise ValueError("aggregate count is outside the supported range")

    return ExactApdexScore(
        total=total,
        score=Fraction(2 * satisfied_count + tolerating_count, 2 * total),
    )
```

## Example

```python
all_frustrated = exact_apdex_from_classified_counts(
    satisfied=0,
    tolerating=0,
    frustrated=4,
)
all_tolerating = exact_apdex_from_classified_counts(
    satisfied=0,
    tolerating=4,
    frustrated=0,
)
mixed = exact_apdex_from_classified_counts(
    satisfied=3,
    tolerating=2,
    frustrated=1,
)
all_satisfied = exact_apdex_from_classified_counts(
    satisfied=4,
    tolerating=0,
    frustrated=0,
)

assert (all_frustrated, all_tolerating, mixed, all_satisfied) == (
    ExactApdexScore(4, Fraction(0, 1)),
    ExactApdexScore(4, Fraction(1, 2)),
    ExactApdexScore(6, Fraction(2, 3)),
    ExactApdexScore(4, Fraction(1, 1)),
)
```

## Trade-offs and Limitations

Validation and scoring use a constant number of bounded integer operations and
`O(1)` memory. Every input and their positive aggregate must fit a signed
64-bit non-negative range. Python integers keep the one-bit-wider numerator
and denominator exact, and `Fraction` reduces the final ratio automatically.

This function deliberately starts after classification. It does not select the
timing threshold, decide how errors are classified, acquire observations,
define a window, or combine differently weighted samples. Changes to any of
those policies can make two scores incomparable.

Apdex is a compact aggregate, not evidence about an individual observation or
a complete view of a latency distribution. The result does not prove user
satisfaction, diagnose a cause, or establish SLA or SLO compliance.

## Related Snippets

<!-- catalog:related:start -->
- [Count Values in Fixed Upper-Bound Bins](count-values-in-fixed-upper-bound-bins.md)
- [Compute a Validated Delta Between Cumulative Histogram Snapshots](compute-a-validated-delta-between-cumulative-histogram-snapshots.md)
- [Classify Progress from Complete Bounded Counter Snapshots](classify-progress-from-complete-bounded-counter-snapshots.md)
<!-- catalog:related:end -->
