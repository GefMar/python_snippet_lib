---
title: "Build an Exact Binary ROC Curve from Tied Integer Scores"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - build-an-exact-precision-recall-curve-from-tied-integer-scores.md
  - compute-exact-binary-roc-auc-with-tied-integer-scores.md
  - build-an-exact-bounded-multiclass-confusion-matrix.md
---

# Build an Exact Binary ROC Curve from Tied Integer Scores

## Idea and Problem

Build every observed binary ROC operating point without splitting equal scores or rounding rates to floating point.

Scores are consumed from highest to lowest. An entire equal-score group enters
the predicted-positive set before its point is emitted, so each point has the
explicit rule `score >= threshold`. Exact confusion counts and `Fraction`
rates make the evidence at each real threshold directly inspectable.

## When to Use

Use this algorithm for a bounded binary evaluation sample when scores are exact
integers, higher means more positive, both classes occur, and every observed
threshold needs deterministic confusion evidence. It is useful in tests, small
reports, and exact comparisons with independently computed ROC AUC.

Use a statistical or machine-learning library for sample weights,
floating-point score conventions, interpolation, plotting endpoints,
confidence intervals, or out-of-core evaluation. Precision-recall curves often
communicate rare-positive behavior more directly; threshold choice still needs
application-specific costs and constraints.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_OBSERVATIONS = 65_536


@dataclass(frozen=True, slots=True)
class BinaryRocPoint:
    threshold: int
    true_positives: int
    false_positives: int
    true_negatives: int
    false_negatives: int
    true_positive_rate: Fraction
    false_positive_rate: Fraction


@dataclass(frozen=True, slots=True)
class BinaryRocCurve:
    positives: int
    negatives: int
    points: tuple[BinaryRocPoint, ...]


def build_binary_roc_curve(
    scores: tuple[int, ...],
    labels: tuple[int, ...],
) -> BinaryRocCurve:
    """Return exact points for each distinct score under score >= threshold."""
    if type(scores) is not tuple:
        raise TypeError("scores must be an exact tuple")
    if type(labels) is not tuple:
        raise TypeError("labels must be an exact tuple")
    if not 2 <= len(scores) <= _MAX_OBSERVATIONS:
        raise ValueError("observation count is outside the supported range")
    if len(labels) != len(scores):
        raise ValueError("scores and labels must have equal lengths")

    for index, score in enumerate(scores):
        if type(score) is not int:
            raise TypeError(f"scores[{index}] must be an exact integer")
        if not _MIN_INT64 <= score <= _MAX_INT64:
            raise ValueError(f"scores[{index}] is outside the signed 64-bit range")

    positives = 0
    for index, label in enumerate(labels):
        if type(label) is not int:
            raise TypeError(f"labels[{index}] must be an exact integer")
        if label not in (0, 1):
            raise ValueError(f"labels[{index}] must be zero or one")
        positives += label

    negatives = len(labels) - positives
    if positives == 0 or negatives == 0:
        raise ValueError("labels must contain both binary classes")

    ordered = sorted(
        zip(scores, labels, strict=True),
        key=lambda observation: observation[0],
        reverse=True,
    )
    points: list[BinaryRocPoint] = []
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
            BinaryRocPoint(
                threshold=threshold,
                true_positives=true_positives,
                false_positives=false_positives,
                true_negatives=negatives - false_positives,
                false_negatives=positives - true_positives,
                true_positive_rate=Fraction(true_positives, positives),
                false_positive_rate=Fraction(false_positives, negatives),
            )
        )
        start = stop

    return BinaryRocCurve(positives, negatives, tuple(points))


```

## Example

```python
scores = (9, 9, 7, 7, 2)
labels = (1, 0, 1, 1, 0)

curve = build_binary_roc_curve(scores, labels)
expected_points = (
    BinaryRocPoint(9, 1, 1, 1, 2, Fraction(1, 3), Fraction(1, 2)),
    BinaryRocPoint(7, 3, 1, 1, 0, Fraction(1, 1), Fraction(1, 2)),
    BinaryRocPoint(2, 3, 2, 0, 0, Fraction(1, 1), Fraction(1, 1)),
)


def trapezoid_area_from_origin(value: BinaryRocCurve) -> Fraction:
    area = Fraction(0)
    previous_fpr = Fraction(0)
    previous_tpr = Fraction(0)
    for point in value.points:
        area += (
            (point.false_positive_rate - previous_fpr)
            * (point.true_positive_rate + previous_tpr)
            / 2
        )
        previous_fpr = point.false_positive_rate
        previous_tpr = point.true_positive_rate
    return area


reordered = build_binary_roc_curve(
    tuple(reversed(scores)),
    tuple(reversed(labels)),
)

try:
    build_binary_roc_curve((3, 2), (1, True))
except TypeError:
    boolean_label_rejected = True
else:
    boolean_label_rejected = False

assert (
    curve,
    reordered,
    trapezoid_area_from_origin(curve),
    boolean_label_rejected,
) == (
    BinaryRocCurve(3, 2, expected_points),
    curve,
    Fraction(7, 12),
    True,
)
```

## Trade-offs and Limitations

Validation is linear. Sorting takes `O(n log n)` time and `O(n)` memory; the
grouped scan is `O(n)`, and the immutable result holds one point for each of
the `d` distinct scores. Counts and rates remain exact throughout.

The result deliberately omits a synthetic threshold above the maximum score.
Every emitted observed threshold selects at least one observation, so the
returned sequence never contains the `(FPR=0, TPR=0)` origin. The final
observed point is always `(1, 1)`. Callers that integrate the curve must add
the origin according to their chosen ROC convention, as the example does.

ROC rates and precision-recall rates answer different questions even though
both curves group the same tied thresholds. This function does not calculate
AUC, interpolate, select an operating point, accept weights or missing labels,
handle multiple classes, measure calibration, or establish generalization,
fairness, or operational usefulness.

## Related Snippets

<!-- catalog:related:start -->
- [Build an Exact Precision-Recall Curve from Tied Integer Scores](build-an-exact-precision-recall-curve-from-tied-integer-scores.md)
- [Compute Exact Binary ROC AUC with Tied Integer Scores](compute-exact-binary-roc-auc-with-tied-integer-scores.md)
- [Build an Exact Bounded Multiclass Confusion Matrix](build-an-exact-bounded-multiclass-confusion-matrix.md)
<!-- catalog:related:end -->
