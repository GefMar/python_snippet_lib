---
title: "Encode Categories with Out-of-Fold Smoothed Target Means"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - fit-and-apply-an-exact-categorical-frequency-encoder.md
  - encode-cyclic-positions-with-sine-and-cosine.md
  - ../python-language/build-a-read-only-mapping-with-canonical-text-keys.md
---

# Encode Categories with Out-of-Fold Smoothed Target Means

## Idea and Problem

Encode bounded categorical training rows with smoothed target means calculated without any observation from the row's assigned fold.

For each row, both the category statistics and the smoothing prior come from
the complement of its whole fold. This prevents direct and fold-level target
leakage into the out-of-fold value. A separate mapping and default fitted on
all training rows are returned only for later inference.

## When to Use

Use this algorithm for finite binary targets when explicit validation folds
already express the required independence policy. Assign every member of a
group to the same fold for grouped data. A complement-of-fold calculation can
include future observations, so it is not suitable for causal temporal
validation; use an expanding-window encoder that only sees earlier rows
instead.

Keep the frozen inference mapping tied to the exact training split. Prefer a
maintained machine-learning or categorical-encoding library when a production
pipeline needs sample weights, multiclass targets, missing-value policy,
nested cross-validation, or splitters that preserve groups or time order.

## Implementation

```python
import math
import unicodedata
from collections.abc import Iterable, Mapping
from dataclasses import dataclass
from itertools import islice, zip_longest
from types import MappingProxyType


_MAX_ROWS = 100_000
_MAX_CATEGORIES = 20_000
_MAX_FOLDS = 64
_MAX_CATEGORY_BYTES = 256
_MAX_FOLD_ID = 2**31 - 1


@dataclass(frozen=True, slots=True)
class OutOfFoldTargetMeanEncoding:
    oof_values: tuple[float, ...]
    inference_by_category: Mapping[str, float]
    inference_default: float


def _canonical_category(value: object) -> str:
    if type(value) is not str:
        raise TypeError("categories must contain strings")
    if not value or value != value.strip() or not value.isprintable():
        raise ValueError("categories must be non-empty trimmed printable text")
    if value != unicodedata.normalize("NFC", value):
        raise ValueError("categories must already use NFC normalization")
    if len(value.encode("utf-8")) > _MAX_CATEGORY_BYTES:
        raise ValueError("a category exceeds the UTF-8 byte limit")
    return value


def _binary_target(value: object) -> int:
    if type(value) not in (int, float):
        raise TypeError("targets must contain integer or float binary values")
    try:
        converted = float(value)
    except OverflowError as error:
        raise ValueError("targets must be finite binary values") from error
    if not math.isfinite(converted) or converted not in (0.0, 1.0):
        raise ValueError("targets must be finite binary values")
    return int(converted)


def _fold_id(value: object) -> int:
    if type(value) is not int:
        raise TypeError("fold_ids must contain integers")
    if not 0 <= value <= _MAX_FOLD_ID:
        raise ValueError("fold IDs are outside the supported range")
    return value


def _bounded_rows(
    categories: Iterable[str],
    targets: Iterable[int | float],
    fold_ids: Iterable[int],
) -> tuple[tuple[str, int, int], ...]:
    inputs = (categories, targets, fold_ids)
    if any(isinstance(values, (str, bytes)) for values in inputs):
        raise TypeError("row inputs must be non-text iterables")

    missing = object()
    rows: list[tuple[str, int, int]] = []
    distinct_categories: set[str] = set()
    distinct_folds: set[int] = set()
    combined = zip_longest(*inputs, fillvalue=missing)
    for index, (category, target, fold) in enumerate(
        islice(combined, _MAX_ROWS + 1)
    ):
        if any(value is missing for value in (category, target, fold)):
            raise ValueError("categories, targets, and fold_ids must have equal length")
        if index == _MAX_ROWS:
            raise ValueError("row count exceeds the supported limit")

        checked_category = _canonical_category(category)
        checked_fold = _fold_id(fold)
        rows.append((checked_category, _binary_target(target), checked_fold))
        distinct_categories.add(checked_category)
        distinct_folds.add(checked_fold)
        if len(distinct_categories) > _MAX_CATEGORIES:
            raise ValueError("distinct category count exceeds the supported limit")
        if len(distinct_folds) > _MAX_FOLDS:
            raise ValueError("fold count exceeds the supported limit")

    if not rows:
        raise ValueError("at least one row is required")
    if len(distinct_folds) < 2:
        raise ValueError("at least two non-empty folds are required")
    return tuple(rows)


def _smoothed_mean(
    target_sum: int,
    count: int,
    *,
    prior: float,
    smoothing: float,
) -> float:
    if count == 0:
        return prior
    category_weight = count / (count + smoothing)
    return category_weight * (target_sum / count) + (1.0 - category_weight) * prior


def encode_out_of_fold_target_means(
    categories: Iterable[str],
    targets: Iterable[int | float],
    fold_ids: Iterable[int],
    *,
    smoothing: int | float,
) -> OutOfFoldTargetMeanEncoding:
    if type(smoothing) not in (int, float):
        raise TypeError("smoothing must be an integer or float")
    try:
        smoothing_value = float(smoothing)
    except OverflowError as error:
        raise ValueError("smoothing must be finite and positive") from error
    if not math.isfinite(smoothing_value) or smoothing_value <= 0.0:
        raise ValueError("smoothing must be finite and positive")

    rows = _bounded_rows(categories, targets, fold_ids)
    category_counts: dict[str, int] = {}
    category_sums: dict[str, int] = {}
    fold_counts: dict[int, int] = {}
    fold_sums: dict[int, int] = {}
    fold_category_counts: dict[tuple[int, str], int] = {}
    fold_category_sums: dict[tuple[int, str], int] = {}

    for category, target, fold in rows:
        category_counts[category] = category_counts.get(category, 0) + 1
        category_sums[category] = category_sums.get(category, 0) + target
        fold_counts[fold] = fold_counts.get(fold, 0) + 1
        fold_sums[fold] = fold_sums.get(fold, 0) + target
        fold_key = (fold, category)
        fold_category_counts[fold_key] = fold_category_counts.get(fold_key, 0) + 1
        fold_category_sums[fold_key] = fold_category_sums.get(fold_key, 0) + target

    total_count = len(rows)
    total_sum = sum(category_sums.values())
    for fold, fold_count in fold_counts.items():
        if fold_count == 0 or total_count - fold_count == 0:
            raise ValueError(f"fold {fold} must have a non-empty complement")

    oof_values: list[float] = []
    for category, _target, fold in rows:
        complement_count = total_count - fold_counts[fold]
        complement_sum = total_sum - fold_sums[fold]
        prior = complement_sum / complement_count
        fold_key = (fold, category)
        category_count = category_counts[category] - fold_category_counts.get(
            fold_key, 0
        )
        category_sum = category_sums[category] - fold_category_sums.get(fold_key, 0)
        oof_values.append(
            _smoothed_mean(
                category_sum,
                category_count,
                prior=prior,
                smoothing=smoothing_value,
            )
        )

    inference_default = total_sum / total_count
    inference_values = {
        category: _smoothed_mean(
            category_sums[category],
            category_counts[category],
            prior=inference_default,
            smoothing=smoothing_value,
        )
        for category in sorted(category_counts)
    }
    return OutOfFoldTargetMeanEncoding(
        oof_values=tuple(oof_values),
        inference_by_category=MappingProxyType(inference_values),
        inference_default=inference_default,
    )
```

## Example

```python
encoding = encode_out_of_fold_target_means(
    categories=("alpha", "alpha", "beta", "alpha", "beta", "gamma"),
    targets=(1, 0, 1, 1, 0, 0),
    fold_ids=(0, 0, 0, 1, 1, 1),
    smoothing=2,
)

rounded_oof = tuple(round(value, 6) for value in encoding.oof_values)
rounded_inference = {
    category: round(value, 6)
    for category, value in encoding.inference_by_category.items()
}

assert (rounded_oof, rounded_inference, encoding.inference_default) == (
    (0.555556, 0.555556, 0.222222, 0.583333, 0.777778, 0.666667),
    {"alpha": 0.6, "beta": 0.5, "gamma": 0.333333},
    0.5,
)
```

## Trade-offs and Limitations

Fitting costs linear time in the row count and stores bounded global,
per-fold, and per-fold-category statistics. High cardinality and many folds
increase memory use. Smoothing reduces variance for rare categories but adds
bias, and its strength must be selected without consulting the final test set.

The full-training mapping must never replace the out-of-fold values used to
train or select a model. Canonical categories are case-sensitive and must be
normalized before this boundary. Unseen inference categories receive the
full-training prior, which can hide distribution shift unless unseen rates are
monitored separately. This compact implementation intentionally omits sample
weights, missing values, multiclass targets, and incremental updates.

## Related Snippets

<!-- catalog:related:start -->
- [Fit and Apply an Exact Categorical Frequency Encoder](fit-and-apply-an-exact-categorical-frequency-encoder.md)
- [Encode Cyclic Positions with Sine and Cosine](encode-cyclic-positions-with-sine-and-cosine.md)
- [Build a Read-Only Mapping with Canonical Text Keys](../python-language/build-a-read-only-mapping-with-canonical-text-keys.md)
<!-- catalog:related:end -->
