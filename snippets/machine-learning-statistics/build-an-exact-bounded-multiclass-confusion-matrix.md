---
title: "Build an Exact Bounded Multiclass Confusion Matrix"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - assemble-out-of-fold-scores-from-explicit-validation-splits.md
  - select-a-forecast-vector-only-when-it-beats-a-frozen-baseline.md
  - measure-drift-between-two-fixed-bin-count-distributions-with-psi.md
---

# Build an Exact Bounded Multiclass Confusion Matrix

## Idea and Problem

Count actual and predicted labels into a dense immutable matrix while preserving one explicit label order and exact one-versus-rest evidence.

Rows represent actual labels and columns represent predicted labels. Freezing
the label vocabulary before counting prevents misspellings from silently
creating new classes. Per-label true positives, false positives, false
negatives, true negatives, and support are derived from the same matrix, so
their totals remain auditable.

## When to Use

Use this algorithm to evaluate a bounded batch of hard multiclass predictions
when the complete label vocabulary is known and a dense matrix is small enough.
Keep the caller's label order stable across runs when matrices will be compared
or rendered together.

Choose a specialized metrics library when observations carry weights,
probabilities, multiple labels, abstentions, or a required averaging policy.
Use a sparse representation when the label vocabulary is too large for
quadratic result memory.

## Implementation

```python
from dataclasses import dataclass

_MAX_LABELS = 64
_MAX_LABEL_CHARACTERS = 128
_MAX_LABEL_BYTES = 512
_MAX_AGGREGATE_LABEL_BYTES = 8_192
_MAX_OBSERVATIONS = 65_536


@dataclass(frozen=True, slots=True)
class LabelConfusionCounts:
    label: str
    true_positive: int
    false_positive: int
    false_negative: int
    true_negative: int
    support: int


@dataclass(frozen=True, slots=True)
class MulticlassConfusionMatrix:
    labels: tuple[str, ...]
    matrix: tuple[tuple[int, ...], ...]
    total: int
    per_label: tuple[LabelConfusionCounts, ...]


def _validated_label_index(labels: object) -> dict[str, int]:
    if type(labels) is not tuple:
        raise TypeError("labels must be an exact tuple")
    if not 2 <= len(labels) <= _MAX_LABELS:
        raise ValueError("label count is outside the supported range")

    index_by_label: dict[str, int] = {}
    aggregate_bytes = 0
    for index, label in enumerate(labels):
        if type(label) is not str:
            raise TypeError(f"labels[{index}] must be an exact string")
        if not 1 <= len(label) <= _MAX_LABEL_CHARACTERS:
            raise ValueError(f"labels[{index}] has an unsupported character count")
        try:
            encoded = label.encode("utf-8")
        except UnicodeEncodeError as error:
            raise ValueError(f"labels[{index}] must be valid UTF-8 text") from error
        if len(encoded) > _MAX_LABEL_BYTES:
            raise ValueError(f"labels[{index}] exceeds the byte limit")
        aggregate_bytes += len(encoded)
        if aggregate_bytes > _MAX_AGGREGATE_LABEL_BYTES:
            raise ValueError("aggregate label bytes exceed the supported limit")
        if label in index_by_label:
            raise ValueError("labels must be unique")
        index_by_label[label] = index
    return index_by_label


def _validate_observations(
    values: object,
    *,
    name: str,
    index_by_label: dict[str, int],
) -> tuple[str, ...]:
    if type(values) is not tuple:
        raise TypeError(f"{name} must be an exact tuple")
    if not 1 <= len(values) <= _MAX_OBSERVATIONS:
        raise ValueError(f"{name} count is outside the supported range")
    for index, label in enumerate(values):
        if type(label) is not str:
            raise TypeError(f"{name}[{index}] must be an exact string")
        if label not in index_by_label:
            raise ValueError(f"{name}[{index}] is not a registered label")
    return values


def build_multiclass_confusion_matrix(
    labels: tuple[str, ...],
    actual: tuple[str, ...],
    predicted: tuple[str, ...],
) -> MulticlassConfusionMatrix:
    """Return exact counts in caller-defined label order."""
    index_by_label = _validated_label_index(labels)
    validated_actual = _validate_observations(
        actual,
        name="actual",
        index_by_label=index_by_label,
    )
    validated_predicted = _validate_observations(
        predicted,
        name="predicted",
        index_by_label=index_by_label,
    )
    if len(validated_actual) != len(validated_predicted):
        raise ValueError("actual and predicted must have equal lengths")

    label_count = len(labels)
    mutable_matrix = [[0] * label_count for _ in range(label_count)]
    for actual_label, predicted_label in zip(
        validated_actual,
        validated_predicted,
        strict=True,
    ):
        mutable_matrix[index_by_label[actual_label]][index_by_label[predicted_label]] += 1

    row_totals = [sum(row) for row in mutable_matrix]
    column_totals = [
        sum(mutable_matrix[row][column] for row in range(label_count))
        for column in range(label_count)
    ]
    total = len(validated_actual)
    per_label = tuple(
        LabelConfusionCounts(
            label=label,
            true_positive=mutable_matrix[index][index],
            false_positive=column_totals[index] - mutable_matrix[index][index],
            false_negative=row_totals[index] - mutable_matrix[index][index],
            true_negative=(
                total - row_totals[index] - column_totals[index] + mutable_matrix[index][index]
            ),
            support=row_totals[index],
        )
        for index, label in enumerate(labels)
    )
    return MulticlassConfusionMatrix(
        labels=labels,
        matrix=tuple(tuple(row) for row in mutable_matrix),
        total=total,
        per_label=per_label,
    )
```

## Example

```python
labels = ("cat", "dog", "fox")
actual = ("cat", "cat", "dog", "dog", "fox", "fox")
predicted = ("cat", "dog", "dog", "fox", "fox", "cat")

report = build_multiclass_confusion_matrix(labels, actual, predicted)
swapped = build_multiclass_confusion_matrix(labels, predicted, actual)

try:
    build_multiclass_confusion_matrix(labels, actual, (*predicted[:-1], "owl"))
except ValueError:
    unknown_label_rejected = True
else:
    unknown_label_rejected = False

assert (report, swapped.matrix, unknown_label_rejected) == (
    MulticlassConfusionMatrix(
        labels=labels,
        matrix=((1, 1, 0), (0, 1, 1), (1, 0, 1)),
        total=6,
        per_label=(
            LabelConfusionCounts("cat", 1, 1, 1, 3, 2),
            LabelConfusionCounts("dog", 1, 1, 1, 3, 2),
            LabelConfusionCounts("fox", 1, 1, 1, 3, 2),
        ),
    ),
    ((1, 0, 1), (1, 1, 0), (0, 1, 1)),
    True,
)
```

## Trade-offs and Limitations

Validation and counting take `O(N + K^2)` time for `N` observations and `K`
labels. The dense immutable matrix and per-label summaries require `O(K^2)`
result memory, with another dense mutable matrix retained during construction.

Labels are matched as exact Unicode strings without case folding or
normalization, and their caller-defined order determines every row and column.
The per-label values use a one-versus-rest interpretation. This implementation
does not infer labels or calculate probabilities, weights, multilabel outcomes,
abstentions, normalization, precision, recall, F1, or macro/micro averages.

## Related Snippets

<!-- catalog:related:start -->
- [Assemble Out-of-Fold Scores from Explicit Validation Splits](assemble-out-of-fold-scores-from-explicit-validation-splits.md)
- [Select a Forecast Vector Only When It Beats a Frozen Baseline](select-a-forecast-vector-only-when-it-beats-a-frozen-baseline.md)
- [Measure Drift Between Two Fixed-Bin Count Distributions with PSI](measure-drift-between-two-fixed-bin-count-distributions-with-psi.md)
<!-- catalog:related:end -->
