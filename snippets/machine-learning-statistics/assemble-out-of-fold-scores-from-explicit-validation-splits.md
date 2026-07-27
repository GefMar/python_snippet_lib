---
title: "Assemble Out-of-Fold Scores from Explicit Validation Splits"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - encode-categories-with-out-of-fold-smoothed-target-means.md
  - score-feature-importances-against-bounded-null-runs.md
  - ../data-processing/select-one-record-per-key-with-an-explicit-ranking-rule.md
---

# Assemble Out-of-Fold Scores from Explicit Validation Splits

## Idea and Problem

Assemble independently computed validation scores into one immutable row-order result while proving that every source row is owned by exactly one fold.

Model fitting, split construction, and metric selection are separate policies.
This small boundary accepts only completed fold records, checks their coverage,
and places each finite score back at its original zero-based row index. The
parallel ownership tuple makes the responsible fold visible for auditing.

## When to Use

Use this algorithm when a cross-validation runner already emits row indices and
one scalar score per validation row. It is useful before evaluating an
out-of-fold vector, persisting it as a feature, or comparing results from
different split strategies.

Do not use it to create folds, train estimators, calculate metrics, or decide
whether random, grouped, stratified, or temporal validation is appropriate.
Those choices must be completed before the fold records reach this function.

## Implementation

```python
import math
from dataclasses import dataclass


_MAX_ROWS = 100_000
_MAX_FOLDS = 128
_MAX_FOLD_ID = (1 << 31) - 1


@dataclass(frozen=True, slots=True)
class ValidationFoldScores:
    fold_id: int
    row_indices: tuple[int, ...]
    scores: tuple[float, ...]


@dataclass(frozen=True, slots=True)
class OutOfFoldScores:
    scores_by_row: tuple[float, ...]
    fold_by_row: tuple[int, ...]


def assemble_out_of_fold_scores(
    row_count: int,
    folds: tuple[ValidationFoldScores, ...],
) -> OutOfFoldScores:
    if type(row_count) is not int:
        raise TypeError("row_count must be an exact integer")
    if not 2 <= row_count <= _MAX_ROWS:
        raise ValueError("row_count is outside the supported range")
    if type(folds) is not tuple:
        raise TypeError("folds must be an exact tuple")
    if not 2 <= len(folds) <= _MAX_FOLDS:
        raise ValueError("fold count is outside the supported range")

    scores_by_row: list[float | None] = [None] * row_count
    fold_by_row: list[int | None] = [None] * row_count
    known_fold_ids: set[int] = set()
    assigned_rows = 0

    for fold in folds:
        if type(fold) is not ValidationFoldScores:
            raise TypeError("folds must contain exact ValidationFoldScores values")
        if type(fold.fold_id) is not int:
            raise TypeError("fold IDs must be exact integers")
        if not 0 <= fold.fold_id <= _MAX_FOLD_ID:
            raise ValueError("a fold ID is outside the supported range")
        if fold.fold_id in known_fold_ids:
            raise ValueError("fold IDs must be unique")
        known_fold_ids.add(fold.fold_id)

        if type(fold.row_indices) is not tuple:
            raise TypeError("fold row indices must be an exact tuple")
        if type(fold.scores) is not tuple:
            raise TypeError("fold scores must be an exact tuple")
        if not fold.row_indices or len(fold.row_indices) != len(fold.scores):
            raise ValueError("each fold must contain aligned non-empty rows and scores")
        if len(fold.row_indices) > row_count:
            raise ValueError("a fold exceeds the supported row count")

        assigned_rows += len(fold.row_indices)
        if assigned_rows > row_count:
            raise ValueError("folds contain more assignments than source rows")

        for row_index, score in zip(
            fold.row_indices,
            fold.scores,
            strict=True,
        ):
            if type(row_index) is not int:
                raise TypeError("row indices must be exact integers")
            if not 0 <= row_index < row_count:
                raise ValueError("a row index is outside the source range")
            if type(score) is not float:
                raise TypeError("scores must be exact floats")
            if not math.isfinite(score):
                raise ValueError("scores must be finite")
            if fold_by_row[row_index] is not None:
                raise ValueError("a source row belongs to more than one fold")
            scores_by_row[row_index] = score
            fold_by_row[row_index] = fold.fold_id

    if assigned_rows != row_count or any(
        owner is None for owner in fold_by_row
    ):
        raise ValueError("folds must cover every source row exactly once")

    return OutOfFoldScores(
        scores_by_row=tuple(
            score for score in scores_by_row if score is not None
        ),
        fold_by_row=tuple(
            owner for owner in fold_by_row if owner is not None
        ),
    )
```

## Example

```python
folds = (
    ValidationFoldScores(
        fold_id=7,
        row_indices=(2, 0),
        scores=(0.8, 0.1),
    ),
    ValidationFoldScores(
        fold_id=9,
        row_indices=(3, 1),
        scores=(0.9, 0.2),
    ),
)
assembled = assemble_out_of_fold_scores(4, folds)

try:
    assemble_out_of_fold_scores(
        3,
        (
            ValidationFoldScores(1, (0, 1), (0.1, 0.2)),
            ValidationFoldScores(2, (1, 2), (0.3, 0.4)),
        ),
    )
except ValueError:
    overlapping_validation_rows_rejected = True
else:
    overlapping_validation_rows_rejected = False

assert (assembled, folds[0].row_indices, overlapping_validation_rows_rejected) == (
    OutOfFoldScores(
        scores_by_row=(0.1, 0.2, 0.8, 0.9),
        fold_by_row=(7, 9, 7, 9),
    ),
    (2, 0),
    True,
)
```

## Trade-offs and Limitations

Validation and assembly take linear time and allocate two lists plus two frozen
tuples with at most 100,000 entries. Exact built-in tuples, integers, and floats
exclude active containers and numeric subclasses. Fold record order and row
order inside a fold do not affect the row-order output.

The function deliberately knows nothing about feature matrices, labels,
estimators, probabilities, class ordering, split leakage, grouping, timestamps,
or score interpretation. Exact coverage proves only that each row received one
value; it does not prove that the upstream model was trained without that row.
Preserve the split evidence separately when that provenance must be audited.

## Related Snippets

<!-- catalog:related:start -->
- [Encode Categories with Out-of-Fold Smoothed Target Means](encode-categories-with-out-of-fold-smoothed-target-means.md)
- [Score Feature Importances Against Bounded Null Runs](score-feature-importances-against-bounded-null-runs.md)
- [Select One Record per Key with an Explicit Ranking Rule](../data-processing/select-one-record-per-key-with-an-explicit-ranking-rule.md)
<!-- catalog:related:end -->
