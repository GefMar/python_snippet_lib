---
title: "Build an Exact Precision-Recall Curve from Tied Integer Scores"
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
  - build-exact-equal-width-binary-calibration-bins-and-expected-calibration-error.md
---

# Build an Exact Precision-Recall Curve from Tied Integer Scores

## Idea and Problem

Build every real threshold operating point for a bounded binary score sample without splitting ties or introducing floating-point rounding.

Scores are processed from highest to lowest. All observations with one score
enter the predicted-positive set together, after which cumulative true and
false positives determine exact precision and recall. One point per distinct
score makes the rule explicit: an observation is predicted positive when its
score is greater than or equal to that point's threshold.

## When to Use

Use this for an exact, inspectable precision-recall table when scores are
signed integers, higher means more positive, and tied scores must share one
decision. It is useful for tests, small evaluation reports, and threshold
analysis where rational evidence is preferable to rounded decimals.

Use a statistical or machine-learning library for weighted samples,
floating-point scores, interpolation, plotting conventions, confidence
intervals, or large out-of-core datasets. Decide separately which operating
point, if any, meets the costs and constraints of the application.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_OBSERVATIONS = 65_536


@dataclass(frozen=True, slots=True)
class PrecisionRecallPoint:
    threshold: int
    true_positives: int
    false_positives: int
    precision: Fraction
    recall: Fraction


def build_precision_recall_curve(
    scores: tuple[int, ...],
    labels: tuple[int, ...],
) -> tuple[PrecisionRecallPoint, ...]:
    """Return exact points for every distinct score threshold."""
    if type(scores) is not tuple:
        raise TypeError("scores must be an exact tuple")
    if type(labels) is not tuple:
        raise TypeError("labels must be an exact tuple")
    if not 1 <= len(scores) <= _MAX_OBSERVATIONS:
        raise ValueError("observation count is outside the supported range")
    if len(labels) != len(scores):
        raise ValueError("scores and labels must have equal lengths")

    for index, score in enumerate(scores):
        if type(score) is not int:
            raise TypeError(f"scores[{index}] must be an exact integer")
        if not _MIN_INT64 <= score <= _MAX_INT64:
            raise ValueError(
                f"scores[{index}] is outside the signed 64-bit range"
            )

    positive_count = 0
    for index, label in enumerate(labels):
        if type(label) is not int:
            raise TypeError(f"labels[{index}] must be an exact integer")
        if label not in (0, 1):
            raise ValueError(f"labels[{index}] must be zero or one")
        positive_count += label
    if positive_count == 0:
        raise ValueError("labels must contain at least one positive observation")

    ordered = sorted(
        zip(scores, labels, strict=True),
        key=lambda observation: observation[0],
        reverse=True,
    )
    points: list[PrecisionRecallPoint] = []
    true_positives = 0
    false_positives = 0
    start = 0

    while start < len(ordered):
        threshold = ordered[start][0]
        stop = start
        while stop < len(ordered) and ordered[stop][0] == threshold:
            label = ordered[stop][1]
            true_positives += label
            false_positives += 1 - label
            stop += 1

        points.append(
            PrecisionRecallPoint(
                threshold=threshold,
                true_positives=true_positives,
                false_positives=false_positives,
                precision=Fraction(
                    true_positives,
                    true_positives + false_positives,
                ),
                recall=Fraction(true_positives, positive_count),
            )
        )
        start = stop

    return tuple(points)
```

## Example

```python
scores = (9, 9, 7, 7, 2)
labels = (1, 0, 1, 1, 0)

curve = build_precision_recall_curve(scores, labels)
expected = (
    PrecisionRecallPoint(9, 1, 1, Fraction(1, 2), Fraction(1, 3)),
    PrecisionRecallPoint(7, 3, 1, Fraction(3, 4), Fraction(1, 1)),
    PrecisionRecallPoint(2, 3, 2, Fraction(3, 5), Fraction(1, 1)),
)
reordered = build_precision_recall_curve(
    tuple(reversed(scores)),
    tuple(reversed(labels)),
)
all_positive = build_precision_recall_curve((5, 1), (1, 1))

try:
    build_precision_recall_curve((3, 2), (0, 0))
except ValueError:
    zero_positive_domain_rejected = True
else:
    zero_positive_domain_rejected = False

try:
    build_precision_recall_curve((3, 2), (1, True))
except TypeError:
    boolean_label_rejected = True
else:
    boolean_label_rejected = False

assert (
    curve == expected
    and reordered == expected
    and tuple(point.precision for point in all_positive)
    == (Fraction(1, 1), Fraction(1, 1))
    and zero_positive_domain_rejected
    and boolean_label_rejected
)
```

## Trade-offs and Limitations

Validation is linear. Sorting costs `O(n log n)` time and `O(n)` memory, and
the grouped scan is `O(n)`. The result contains one point per distinct score
and uses exact integer counts and `Fraction` values throughout.

The function returns only thresholds that occur in the sample. It deliberately
does not prepend a synthetic `(precision=1, recall=0)` endpoint, interpolate
between points, calculate area under the curve, or choose a threshold. An
all-negative sample is rejected because recall has no positive denominator;
an all-positive sample is defined and has precision one at every threshold.

The curve describes this bounded sample under one fixed score direction. It
does not establish calibration, uncertainty, fairness, causal validity,
generalization to new data, or the operational value of any threshold.

## Related Snippets

<!-- catalog:related:start -->
- [Compute Exact Binary ROC AUC with Tied Integer Scores](compute-exact-binary-roc-auc-with-tied-integer-scores.md)
- [Build an Exact Bounded Multiclass Confusion Matrix](build-an-exact-bounded-multiclass-confusion-matrix.md)
- [Build Exact Equal-Width Binary Calibration Bins and Expected Calibration Error](build-exact-equal-width-binary-calibration-bins-and-expected-calibration-error.md)
<!-- catalog:related:end -->
