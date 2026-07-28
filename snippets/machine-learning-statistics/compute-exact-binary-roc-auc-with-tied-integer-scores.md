---
title: "Compute Exact Binary ROC AUC with Tied Integer Scores"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - build-an-exact-bounded-multiclass-confusion-matrix.md
  - assemble-out-of-fold-scores-from-explicit-validation-splits.md
  - select-a-forecast-vector-only-when-it-beats-a-frozen-baseline.md
---

# Compute Exact Binary ROC AUC with Tied Integer Scores

## Idea and Problem

Measure how consistently higher integer scores rank positive observations above negative observations while preserving exact evidence for every cross-class pair.

Each positive-negative pair is concordant when the positive score is higher,
discordant when it is lower, and tied when the scores are equal. ROC AUC is the
share of concordant pairs plus half the share of tied pairs. Processing equal
scores as groups counts those outcomes without comparing every pair directly.

## When to Use

Use this algorithm for a bounded binary evaluation set with exact integer
scores when score order measures positive-class preference and an exact,
threshold-independent ranking summary is useful. Both classes must be present,
and the scoring direction must already be fixed so higher means more positive.

Choose a statistical library when scores are floating point, observations have
weights, uncertainty estimates are required, or the same workflow must produce
ROC curve points and threshold diagnostics. Use calibration and threshold
metrics alongside AUC when score magnitudes or concrete decisions matter.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_OBSERVATIONS = 65_536


@dataclass(frozen=True, slots=True)
class BinaryRocAuc:
    positives: int
    negatives: int
    concordant: int
    tied: int
    discordant: int
    auc: Fraction


def compute_binary_roc_auc(
    scores: tuple[int, ...],
    labels: tuple[int, ...],
) -> BinaryRocAuc:
    """Return exact binary ROC AUC evidence with higher scores as positive."""
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
    )
    concordant = 0
    tied = 0
    negatives_below = 0
    start = 0

    while start < len(ordered):
        stop = start + 1
        score = ordered[start][0]
        group_positives = ordered[start][1]
        group_negatives = 1 - ordered[start][1]

        while stop < len(ordered) and ordered[stop][0] == score:
            group_positives += ordered[stop][1]
            group_negatives += 1 - ordered[stop][1]
            stop += 1

        concordant += group_positives * negatives_below
        tied += group_positives * group_negatives
        negatives_below += group_negatives
        start = stop

    pair_count = positives * negatives
    discordant = pair_count - concordant - tied
    auc = Fraction(2 * concordant + tied, 2 * pair_count)
    return BinaryRocAuc(
        positives=positives,
        negatives=negatives,
        concordant=concordant,
        tied=tied,
        discordant=discordant,
        auc=auc,
    )
```

## Example

```python
scores = (9, 8, 8, 3, 3, 1)
labels = (1, 0, 1, 0, 1, 0)

result = compute_binary_roc_auc(scores, labels)
reordered = compute_binary_roc_auc(tuple(reversed(scores)), tuple(reversed(labels)))
all_tied = compute_binary_roc_auc((5, 5), (1, 0))

try:
    compute_binary_roc_auc((1, 2), (0, True))
except TypeError:
    boolean_rejected = True
else:
    boolean_rejected = False

assert (result, reordered, all_tied, boolean_rejected) == (
    BinaryRocAuc(3, 3, 6, 2, 1, Fraction(7, 9)),
    result,
    BinaryRocAuc(1, 1, 0, 1, 0, Fraction(1, 2)),
    True,
)
```

## Trade-offs and Limitations

Validation takes `O(n)` time. Sorting takes `O(n log n)` time and `O(n)`
memory, after which the equal-score scan is linear. The returned counts and
`Fraction` are exact; no binary floating-point rounding is introduced. The
input cap also bounds the quadratic number represented by the pair counts even
though the implementation never enumerates those pairs.

Only relative score order and ties affect the result, so score distances carry
no meaning here. The function fixes higher scores as more positive and accepts
only exact signed 64-bit integer scores and exact `0` or `1` labels. It does not
infer direction, choose thresholds, produce an ROC curve, accept weights or
multiple classes, measure calibration, estimate confidence intervals, or
interpret whether the observed discrimination is useful in a specific domain.

## Related Snippets

<!-- catalog:related:start -->
- [Build an Exact Bounded Multiclass Confusion Matrix](build-an-exact-bounded-multiclass-confusion-matrix.md)
- [Assemble Out-of-Fold Scores from Explicit Validation Splits](assemble-out-of-fold-scores-from-explicit-validation-splits.md)
- [Select a Forecast Vector Only When It Beats a Frozen Baseline](select-a-forecast-vector-only-when-it-beats-a-frozen-baseline.md)
<!-- catalog:related:end -->
