---
title: "Fit and Apply an Exact Categorical Frequency Encoder"
snippet_type: recipe
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - encode-categories-with-out-of-fold-smoothed-target-means.md
  - measure-drift-between-two-fixed-bin-count-distributions-with-psi.md
  - ../python-language/build-a-read-only-mapping-with-canonical-text-keys.md
---

# Fit and Apply an Exact Categorical Frequency Encoder

## Idea and Problem

Fit an immutable exact count-and-total mapping for bounded categories, then encode known frequencies and map unseen categories to zero.

The fitted state stores integer pairs instead of rounded training floats, so a
known category's frequency always derives from its exact count and the exact
training total. Each category remains distinct; the recipe never combines rare
values into an implicit fallback bucket.

## When to Use

Use frequency encoding when a model accepts numeric features and category
prevalence is a useful signal, while one-hot expansion would be inconvenient.
Fit only on the training partition, freeze that state, and apply it unchanged
to validation, test, and inference data. Refit on the chosen full training set
only after model selection.

Use a maintained preprocessing library when a pipeline needs learned missing
value handling, feature-name propagation, sparse matrices, cross-validation
integration, or stable serialization across library versions. Track unseen
category rates separately because a rising zero-encoding rate can indicate
distribution shift.

## Implementation

```python
import unicodedata
from collections.abc import Iterable, Mapping
from dataclasses import dataclass
from itertools import islice
from types import MappingProxyType


_MAX_TRAINING_ROWS = 100_000
_MAX_TRANSFORM_ROWS = 100_000
_MAX_DISTINCT_CATEGORIES = 20_000
_MAX_CATEGORY_BYTES = 256


def _frequency_category(value: object) -> str:
    if type(value) is not str:
        raise TypeError("categories must contain strings")
    if not value or value != value.strip() or not value.isprintable():
        raise ValueError("categories must be non-empty trimmed printable text")
    if value != unicodedata.normalize("NFC", value):
        raise ValueError("categories must already use NFC normalization")
    if len(value.encode("utf-8")) > _MAX_CATEGORY_BYTES:
        raise ValueError("a category exceeds the UTF-8 byte limit")
    return value


def _bounded_frequency_categories(
    values: Iterable[str],
    *,
    limit: int,
    allow_empty: bool,
) -> tuple[str, ...]:
    if isinstance(values, (str, bytes)):
        raise TypeError("categories must be a non-text iterable")
    snapshot = tuple(islice(values, limit + 1))
    if len(snapshot) > limit:
        raise ValueError("category input exceeds the supported row limit")
    if not snapshot and not allow_empty:
        raise ValueError("at least one training category is required")
    return tuple(_frequency_category(value) for value in snapshot)


@dataclass(frozen=True, slots=True)
class ExactFrequencyEncoder:
    count_and_total_by_category: Mapping[str, tuple[int, int]]

    def transform(self, categories: Iterable[str]) -> tuple[float, ...]:
        snapshot = _bounded_frequency_categories(
            categories,
            limit=_MAX_TRANSFORM_ROWS,
            allow_empty=True,
        )
        encoded: list[float] = []
        for category in snapshot:
            count_and_total = self.count_and_total_by_category.get(category)
            if count_and_total is None:
                encoded.append(0.0)
            else:
                count, total = count_and_total
                encoded.append(count / total)
        return tuple(encoded)


def fit_exact_frequency_encoder(
    categories: Iterable[str],
) -> ExactFrequencyEncoder:
    snapshot = _bounded_frequency_categories(
        categories,
        limit=_MAX_TRAINING_ROWS,
        allow_empty=False,
    )
    counts: dict[str, int] = {}
    for category in snapshot:
        counts[category] = counts.get(category, 0) + 1
        if len(counts) > _MAX_DISTINCT_CATEGORIES:
            raise ValueError("distinct category count exceeds the supported limit")

    total = len(snapshot)
    exact_frequencies = {
        category: (counts[category], total) for category in sorted(counts)
    }
    return ExactFrequencyEncoder(
        count_and_total_by_category=MappingProxyType(exact_frequencies)
    )
```

## Example

```python
encoder = fit_exact_frequency_encoder(("red", "blue", "red", "green"))
encoded = encoder.transform(("green", "red", "amber"))
exact_state = dict(encoder.count_and_total_by_category)

try:
    encoder.count_and_total_by_category["red"] = (4, 4)
except TypeError:
    state_is_immutable = True
else:
    state_is_immutable = False

assert (encoded, exact_state, state_is_immutable) == (
    (0.25, 0.5, 0.0),
    {"blue": (1, 4), "green": (1, 4), "red": (2, 4)},
    True,
)
```

## Trade-offs and Limitations

Fitting is linear in the training row count, transformation is linear in the
input row count, and the frozen mapping uses space proportional to category
cardinality. The distinct-category limit fails explicitly instead of silently
aggregating rare values. Very large vocabularies may need hashing, a reviewed
rare-category policy, or a maintained encoder with controlled memory use.

Equal-frequency categories collide on the same scalar, so this representation
can discard category identity and interactions. Frequency can also shift over
time; unseen values map to zero rather than a learned rare mass, and that policy
must match model expectations. Categories are case-sensitive canonical text,
and missing values need a separate explicit representation before fitting.

## Related Snippets

<!-- catalog:related:start -->
- [Encode Categories with Out-of-Fold Smoothed Target Means](encode-categories-with-out-of-fold-smoothed-target-means.md)
- [Measure Drift Between Two Fixed-Bin Count Distributions with PSI](measure-drift-between-two-fixed-bin-count-distributions-with-psi.md)
- [Build a Read-Only Mapping with Canonical Text Keys](../python-language/build-a-read-only-mapping-with-canonical-text-keys.md)
<!-- catalog:related:end -->
