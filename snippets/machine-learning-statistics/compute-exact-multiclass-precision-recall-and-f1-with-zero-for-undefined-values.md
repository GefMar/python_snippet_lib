---
title: "Compute Exact Multiclass Precision, Recall, and F1 with Zero for Undefined Values"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - build-an-exact-bounded-multiclass-confusion-matrix.md
  - compute-exact-unweighted-cohens-kappa-from-a-confusion-matrix.md
  - build-an-exact-precision-recall-curve-from-tied-integer-scores.md
---

# Compute Exact Multiclass Precision, Recall, and F1 with Zero for Undefined Values

## Idea and Problem

Derive exact per-class, macro, support-weighted, and micro precision, recall, and F1 scores from one bounded multiclass confusion matrix under an explicit zero-for-undefined policy.

Rows represent actual classes and columns represent predicted classes. Each
class is evaluated one versus the rest: its row total is its support, its
column total is its predicted count, and its diagonal cell is its true-positive
count. `Fraction` keeps every reported score exact.

## When to Use

Use this function when a bounded batch has exactly one actual and one predicted
class per observation, the class order is already fixed, and the dense
confusion matrix is the authoritative input. It is useful for deterministic
reports and tests that must make their treatment of absent classes explicit.

This profile defines precision, recall, or F1 as exactly zero whenever that
score's denominator is zero. Macro averages include every declared class,
including classes absent from both actual and predicted observations. Compare
reports only when they use the same class vocabulary and undefined-value
policy.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction

_MAX_CLASSES = 64
_MAX_INT64 = (1 << 63) - 1


@dataclass(frozen=True, slots=True)
class ClassPrecisionRecallF1:
    index: int
    true_positive: int
    false_positive: int
    false_negative: int
    support: int
    predicted: int
    precision: Fraction
    recall: Fraction
    f1: Fraction


@dataclass(frozen=True, slots=True)
class MulticlassPrecisionRecallF1:
    class_count: int
    total: int
    per_class: tuple[ClassPrecisionRecallF1, ...]
    macro_precision: Fraction
    macro_recall: Fraction
    macro_f1: Fraction
    weighted_precision: Fraction
    weighted_recall: Fraction
    weighted_f1: Fraction
    micro_precision: Fraction
    micro_recall: Fraction
    micro_f1: Fraction


def _ratio_or_zero(numerator: int, denominator: int) -> Fraction:
    return Fraction(0) if denominator == 0 else Fraction(numerator, denominator)


def exact_multiclass_precision_recall_f1(
    matrix: tuple[tuple[int, ...], ...],
) -> MulticlassPrecisionRecallF1:
    """Return exact multiclass metrics with zero for undefined ratios."""
    if type(matrix) is not tuple:
        raise TypeError("matrix must be an exact tuple")
    class_count = len(matrix)
    if not 2 <= class_count <= _MAX_CLASSES:
        raise ValueError("matrix size is outside the supported range")

    row_totals: list[int] = []
    column_totals = [0] * class_count
    total = 0
    for row_index, row in enumerate(matrix):
        if type(row) is not tuple:
            raise TypeError(f"matrix[{row_index}] must be an exact tuple")
        if len(row) != class_count:
            raise ValueError("matrix must be square")

        row_total = 0
        for column_index, count in enumerate(row):
            if type(count) is not int:
                raise TypeError(f"matrix[{row_index}][{column_index}] must be an exact integer")
            if not 0 <= count <= _MAX_INT64:
                raise ValueError(
                    f"matrix[{row_index}][{column_index}] is outside the supported range"
                )
            row_total += count
            column_totals[column_index] += count
        row_totals.append(row_total)
        total += row_total

    if not 1 <= total <= _MAX_INT64:
        raise ValueError("matrix aggregate is outside the supported range")

    per_class = tuple(
        ClassPrecisionRecallF1(
            index=index,
            true_positive=matrix[index][index],
            false_positive=column_totals[index] - matrix[index][index],
            false_negative=row_totals[index] - matrix[index][index],
            support=row_totals[index],
            predicted=column_totals[index],
            precision=_ratio_or_zero(matrix[index][index], column_totals[index]),
            recall=_ratio_or_zero(matrix[index][index], row_totals[index]),
            f1=_ratio_or_zero(
                2 * matrix[index][index],
                row_totals[index] + column_totals[index],
            ),
        )
        for index in range(class_count)
    )

    macro_precision = sum((metrics.precision for metrics in per_class), Fraction()) / class_count
    macro_recall = sum((metrics.recall for metrics in per_class), Fraction()) / class_count
    macro_f1 = sum((metrics.f1 for metrics in per_class), Fraction()) / class_count
    weighted_precision = (
        sum(
            (metrics.support * metrics.precision for metrics in per_class),
            Fraction(),
        )
        / total
    )
    weighted_recall = (
        sum(
            (metrics.support * metrics.recall for metrics in per_class),
            Fraction(),
        )
        / total
    )
    weighted_f1 = (
        sum(
            (metrics.support * metrics.f1 for metrics in per_class),
            Fraction(),
        )
        / total
    )

    micro_true_positive = sum(metrics.true_positive for metrics in per_class)
    micro_false_positive = sum(metrics.false_positive for metrics in per_class)
    micro_false_negative = sum(metrics.false_negative for metrics in per_class)

    return MulticlassPrecisionRecallF1(
        class_count=class_count,
        total=total,
        per_class=per_class,
        macro_precision=macro_precision,
        macro_recall=macro_recall,
        macro_f1=macro_f1,
        weighted_precision=weighted_precision,
        weighted_recall=weighted_recall,
        weighted_f1=weighted_f1,
        micro_precision=_ratio_or_zero(
            micro_true_positive,
            micro_true_positive + micro_false_positive,
        ),
        micro_recall=_ratio_or_zero(
            micro_true_positive,
            micro_true_positive + micro_false_negative,
        ),
        micro_f1=_ratio_or_zero(
            2 * micro_true_positive,
            2 * micro_true_positive + micro_false_positive + micro_false_negative,
        ),
    )


```

## Example

```python
matrix = (
    (8, 2, 0),
    (0, 1, 0),
    (0, 0, 0),
)
report = exact_multiclass_precision_recall_f1(matrix)
harmonic_of_macro_precision_and_recall = (
    2
    * report.macro_precision
    * report.macro_recall
    / (report.macro_precision + report.macro_recall)
)

assert report.per_class == (
    ClassPrecisionRecallF1(0, 8, 0, 2, 10, 8, Fraction(1), Fraction(4, 5), Fraction(8, 9)),
    ClassPrecisionRecallF1(1, 1, 2, 0, 1, 3, Fraction(1, 3), Fraction(1), Fraction(1, 2)),
    ClassPrecisionRecallF1(2, 0, 0, 0, 0, 0, Fraction(0), Fraction(0), Fraction(0)),
)
assert (
    report.total,
    report.macro_precision,
    report.macro_recall,
    report.macro_f1,
    report.weighted_precision,
    report.weighted_recall,
    report.weighted_f1,
    report.micro_precision,
    report.micro_recall,
    report.micro_f1,
) == (
    11,
    Fraction(4, 9),
    Fraction(3, 5),
    Fraction(25, 54),
    Fraction(31, 33),
    Fraction(9, 11),
    Fraction(169, 198),
    Fraction(9, 11),
    Fraction(9, 11),
    Fraction(9, 11),
)
assert report.macro_f1 != harmonic_of_macro_precision_and_recall
```

## Trade-offs and Limitations

Validation and metric construction take `O(K^2)` time for `K` classes. Row
totals, column totals, and the immutable per-class result use `O(K)` additional
memory. Python integer arithmetic is exact, but `Fraction` construction,
reduction, addition, and multiplication are not constant-cost operations.

Macro precision, recall, and F1 are separate arithmetic means over all classes;
macro F1 is not the harmonic mean of macro precision and macro recall. The
weighted metrics average each per-class metric by actual support. With this
complete single-label matrix, weighted recall and all three micro metrics equal
the diagonal count divided by the total, while weighted precision and weighted
F1 need not do so.

This function consumes counts only. It does not associate indices with labels,
build a confusion matrix from observations, normalize counts, process class or
sample weights, evaluate probabilities, or support multilabel outcomes. The
zero-for-undefined convention can conceal why a class has no actual or
predicted observations, so retain the reported support and predicted counts
when presenting the scores.

## Related Snippets

<!-- catalog:related:start -->
- [Build an Exact Bounded Multiclass Confusion Matrix](build-an-exact-bounded-multiclass-confusion-matrix.md)
- [Compute Exact Unweighted Cohen's Kappa from a Confusion Matrix](compute-exact-unweighted-cohens-kappa-from-a-confusion-matrix.md)
- [Build an Exact Precision-Recall Curve from Tied Integer Scores](build-an-exact-precision-recall-curve-from-tied-integer-scores.md)
<!-- catalog:related:end -->
