---
title: "Compute Exact Jaccard Similarity for Bounded Unique Text-Feature Sets"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - fit-and-apply-an-exact-categorical-frequency-encoder.md
  - measure-drift-between-two-fixed-bin-count-distributions-with-psi.md
  - build-an-exact-bounded-multiclass-confusion-matrix.md
---

# Compute Exact Jaccard Similarity for Bounded Unique Text-Feature Sets

## Idea and Problem

Measure the overlap of two bounded text-feature sets as an exact intersection-to-union fraction.

Jaccard similarity divides the number of shared features by the number of
features present on either side. Validating unique input tuples makes the set
semantics explicit instead of silently discarding duplicates, while an exact
`Fraction` avoids introducing rounding into the reported score.

## When to Use

Use this measure when each input is already a finite set of exact categorical
features and presence matters but frequency does not. It can describe overlap
between tags, extracted characteristics, or other bounded binary feature
collections independently of their declaration order.

Define feature extraction before calling the function. Use a multiset,
weighted, fuzzy, or domain-specific measure when repeated values, importance,
spelling distance, or semantic similarity should influence the result.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction

_MAX_FEATURES_PER_SIDE = 4_096
_MAX_FEATURE_UTF8_BYTES = 256
_MAX_SIDE_UTF8_BYTES = 1 << 20


@dataclass(frozen=True, slots=True)
class ExactJaccardSimilarity:
    left_count: int
    right_count: int
    intersection_count: int
    union_count: int
    similarity: Fraction | None


def _validated_text_feature_set(
    features: object,
    *,
    name: str,
) -> frozenset[str]:
    if type(features) is not tuple:
        raise TypeError(f"{name} must be an exact tuple")
    if len(features) > _MAX_FEATURES_PER_SIDE:
        raise ValueError(f"{name} exceeds the feature-count limit")

    seen: set[str] = set()
    encoded_bytes = 0
    for index, feature in enumerate(features):
        if type(feature) is not str:
            raise TypeError(f"{name}[{index}] must be an exact string")
        if not feature:
            raise ValueError(f"{name}[{index}] must not be empty")
        try:
            feature_bytes = feature.encode("utf-8")
        except UnicodeEncodeError:
            raise ValueError(f"{name}[{index}] must be valid UTF-8 text") from None
        if len(feature_bytes) > _MAX_FEATURE_UTF8_BYTES:
            raise ValueError(f"{name}[{index}] exceeds the UTF-8 byte limit")
        encoded_bytes += len(feature_bytes)
        if encoded_bytes > _MAX_SIDE_UTF8_BYTES:
            raise ValueError(f"{name} exceeds the aggregate UTF-8 byte limit")
        if feature in seen:
            raise ValueError(f"{name}[{index}] is duplicated")
        seen.add(feature)

    return frozenset(seen)


def exact_jaccard_similarity(
    left_features: tuple[str, ...],
    right_features: tuple[str, ...],
) -> ExactJaccardSimilarity:
    """Return exact set-overlap counts and their Jaccard similarity."""
    left = _validated_text_feature_set(left_features, name="left_features")
    right = _validated_text_feature_set(right_features, name="right_features")
    intersection_count = len(left & right)
    union_count = len(left) + len(right) - intersection_count
    similarity = (
        None if union_count == 0 else Fraction(intersection_count, union_count)
    )
    return ExactJaccardSimilarity(
        left_count=len(left),
        right_count=len(right),
        intersection_count=intersection_count,
        union_count=union_count,
        similarity=similarity,
    )
```

## Example

```python
def brute_tuple_jaccard(
    left: tuple[str, ...],
    right: tuple[str, ...],
) -> tuple[int, int, Fraction | None]:
    intersection = sum(
        any(left_value == right_value for right_value in right)
        for left_value in left
    )
    union = list(left)
    for right_value in right:
        if not any(right_value == existing for existing in union):
            union.append(right_value)
    union_count = len(union)
    similarity = None if union_count == 0 else Fraction(intersection, union_count)
    return intersection, union_count, similarity


universe = ("cache", "queue", "retry", "trace")
for left_mask in range(1 << len(universe)):
    left = tuple(
        feature
        for index, feature in enumerate(universe)
        if left_mask & (1 << index)
    )
    for right_mask in range(1 << len(universe)):
        right = tuple(
            feature
            for index, feature in enumerate(universe)
            if right_mask & (1 << index)
        )
        result = exact_jaccard_similarity(left, right)
        assert (
            result.intersection_count,
            result.union_count,
            result.similarity,
        ) == brute_tuple_jaccard(left, right)

overlap = exact_jaccard_similarity(
    ("cache", "retry", "trace"),
    ("queue", "retry", "trace"),
)
empty = exact_jaccard_similarity((), ())

assert (overlap, empty.similarity) == (
    ExactJaccardSimilarity(3, 3, 2, 4, Fraction(1, 2)),
    None,
)
```

## Trade-offs and Limitations

For `B` admitted UTF-8 bytes and `L` and `R` features, validation and set
operations take expected `O(B + L + R)` hashing work and use `O(L + R)` memory.
Python's randomized string hashes do not affect the exact counts or result.
The returned immutable record uses constant additional space.

Two empty sets have no nonzero union, so this function returns `None` instead
of choosing one of the competing empty-set conventions. One empty side and one
non-empty side produce exact zero. Inputs are case-sensitive and
normalization-sensitive; no Unicode normalization or case folding is applied.

Duplicate features are rejected rather than treated as frequencies. The
function does not tokenize input, compare multisets, apply weights, perform
fuzzy or semantic matching, approximate with MinHash, or make inferential,
causal, or model-quality claims.

## Related Snippets

<!-- catalog:related:start -->
- [Fit and Apply an Exact Categorical Frequency Encoder](fit-and-apply-an-exact-categorical-frequency-encoder.md)
- [Measure Drift Between Two Fixed-Bin Count Distributions with PSI](measure-drift-between-two-fixed-bin-count-distributions-with-psi.md)
- [Build an Exact Bounded Multiclass Confusion Matrix](build-an-exact-bounded-multiclass-confusion-matrix.md)
<!-- catalog:related:end -->
