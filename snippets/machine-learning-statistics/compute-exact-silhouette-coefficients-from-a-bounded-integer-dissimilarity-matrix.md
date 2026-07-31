---
title: "Compute Exact Silhouette Coefficients from a Bounded Integer Dissimilarity Matrix"
snippet_type: algorithm
use_cases:
  - data-transformation
  - testing
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - compute-an-exact-adjusted-rand-index-from-two-bounded-integer-partitions.md
  - fit-exact-one-dimensional-k-means-with-contiguous-cluster-dp.md
  - compute-an-exact-normalized-one-dimensional-wasserstein-distance.md
---

# Compute Exact Silhouette Coefficients from a Bounded Integer Dissimilarity Matrix

## Idea and Problem

Evaluate one supplied clustering from exact pairwise dissimilarities without losing its per-sample evidence to floating-point rounding.

For each sample, `a` is its mean dissimilarity to the other members of its own
cluster. The value `b` is the smallest mean dissimilarity from that sample to
any other cluster. Their normalized difference `(b - a) / max(a, b)` is the
silhouette coefficient: values near one indicate separation, while negative
values indicate that another cluster is closer on average.

Integer dissimilarities make both means rational. Returning `Fraction`
evidence preserves exact comparisons between candidate clusters and exposes
the singleton and all-zero conventions explicitly.

## When to Use

Use this calculation for a small, fully materialized, symmetric dissimilarity
matrix when a partition already exists and exact per-sample diagnostics are
useful for tests or review. Off-diagonal zero is allowed, so duplicate or
indistinguishable samples remain representable.

Use a vectorized statistics package for large dense matrices, sample-based
approximations, sparse neighborhoods, or direct feature-vector metrics. Decide
whether silhouette analysis is meaningful before using it with arbitrary
non-metric dissimilarities.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction
from random import Random

_MAX_SILHOUETTE_SAMPLES = 512
_MAX_DISSIMILARITY = 2**31 - 1
_MIN_CLUSTER_LABEL = -(2**31)
_MAX_CLUSTER_LABEL = 2**31 - 1
_MAX_SILHOUETTE_CLUSTERS = 64


@dataclass(frozen=True, slots=True)
class SampleSilhouette:
    own_cluster_size: int
    within_mean: Fraction | None
    nearest_other_mean: Fraction
    coefficient: Fraction


@dataclass(frozen=True, slots=True)
class SilhouetteEvidence:
    samples: tuple[SampleSilhouette, ...]
    mean_coefficient: Fraction


def exact_silhouette_coefficients(
    dissimilarities: tuple[tuple[int, ...], ...],
    labels: tuple[int, ...],
) -> SilhouetteEvidence:
    """Return exact silhouette evidence for one bounded supplied partition."""
    if type(dissimilarities) is not tuple:
        raise TypeError("dissimilarities must be an exact tuple")
    sample_count = len(dissimilarities)
    if not 3 <= sample_count <= _MAX_SILHOUETTE_SAMPLES:
        raise ValueError("sample count is outside 3..512")

    for row_index, row in enumerate(dissimilarities):
        if type(row) is not tuple:
            raise TypeError(f"dissimilarities[{row_index}] must be an exact tuple")
        if len(row) != sample_count:
            raise ValueError("dissimilarity matrix must be square")
        for column_index, distance in enumerate(row):
            if type(distance) is not int:
                raise TypeError("dissimilarities must be exact integers")
            if not 0 <= distance <= _MAX_DISSIMILARITY:
                raise ValueError("dissimilarity is outside the supported range")
            if row_index == column_index and distance != 0:
                raise ValueError("dissimilarity diagonal must be zero")
            if column_index < row_index and distance != dissimilarities[column_index][row_index]:
                raise ValueError("dissimilarity matrix must be symmetric")

    if type(labels) is not tuple:
        raise TypeError("labels must be an exact tuple")
    if len(labels) != sample_count:
        raise ValueError("labels must contain one value per sample")
    members_by_label: dict[int, list[int]] = {}
    for sample_index, label in enumerate(labels):
        if type(label) is not int:
            raise TypeError(f"labels[{sample_index}] must be an exact integer")
        if not _MIN_CLUSTER_LABEL <= label <= _MAX_CLUSTER_LABEL:
            raise ValueError(f"labels[{sample_index}] is outside the signed 32-bit range")
        members_by_label.setdefault(label, []).append(sample_index)

    cluster_count = len(members_by_label)
    if not 2 <= cluster_count <= _MAX_SILHOUETTE_CLUSTERS:
        raise ValueError("represented cluster count is outside 2..64")
    if cluster_count >= sample_count:
        raise ValueError("at least one represented cluster must contain two samples")

    evidence: list[SampleSilhouette] = []
    for sample_index, own_label in enumerate(labels):
        totals = {label: 0 for label in members_by_label}
        for other_index, other_label in enumerate(labels):
            totals[other_label] += dissimilarities[sample_index][other_index]

        own_size = len(members_by_label[own_label])
        within_mean = None
        if own_size > 1:
            within_mean = Fraction(totals[own_label], own_size - 1)
        nearest_other_mean = min(
            Fraction(totals[label], len(members))
            for label, members in members_by_label.items()
            if label != own_label
        )

        coefficient = Fraction()
        if within_mean is not None:
            scale = max(within_mean, nearest_other_mean)
            if scale:
                coefficient = (nearest_other_mean - within_mean) / scale
        evidence.append(
            SampleSilhouette(
                own_cluster_size=own_size,
                within_mean=within_mean,
                nearest_other_mean=nearest_other_mean,
                coefficient=coefficient,
            )
        )

    return SilhouetteEvidence(
        samples=tuple(evidence),
        mean_coefficient=sum((item.coefficient for item in evidence), Fraction()) / sample_count,
    )
```

## Example

```python

def group_list_oracle(
    matrix: tuple[tuple[int, ...], ...],
    labels: tuple[int, ...],
) -> tuple[Fraction, ...]:
    groups = {
        label: tuple(index for index, candidate in enumerate(labels) if candidate == label)
        for label in set(labels)
    }
    scores: list[Fraction] = []
    for sample, label in enumerate(labels):
        own_peers = tuple(index for index in groups[label] if index != sample)
        if not own_peers:
            scores.append(Fraction())
            continue
        within = sum((Fraction(matrix[sample][index]) for index in own_peers), Fraction()) / len(
            own_peers
        )
        outside_means = tuple(
            sum((Fraction(matrix[sample][index]) for index in members), Fraction()) / len(members)
            for other_label, members in groups.items()
            if other_label != label
        )
        nearest = min(outside_means)
        scale = max(within, nearest)
        scores.append(Fraction() if scale == 0 else (nearest - within) / scale)
    return tuple(scores)


matrix = (
    (0, 1, 4),
    (1, 0, 3),
    (4, 3, 0),
)
result = exact_silhouette_coefficients(matrix, (0, 0, 1))
assert tuple(item.coefficient for item in result.samples) == (
    Fraction(3, 4),
    Fraction(2, 3),
    Fraction(),
)
assert result.mean_coefficient == Fraction(17, 36)

rng = Random(0x51_1A_0E)
checked = 0
for _ in range(5_000):
    size = rng.randrange(3, 9)
    cluster_count = rng.randrange(2, min(size, 5))
    labels = tuple(range(cluster_count)) + tuple(
        rng.randrange(cluster_count) for _ in range(size - cluster_count)
    )
    labels = tuple(labels[index] for index in rng.sample(range(size), size))
    rows = [[0] * size for _ in range(size)]
    for left in range(size):
        for right in range(left + 1, size):
            rows[left][right] = rows[right][left] = rng.randrange(6)
    candidate = tuple(tuple(row) for row in rows)
    actual = exact_silhouette_coefficients(candidate, labels)
    expected = group_list_oracle(candidate, labels)
    assert tuple(item.coefficient for item in actual.samples) == expected
    assert actual.mean_coefficient == sum(expected, Fraction()) / size
    checked += 1

scaled = tuple(tuple(value * 7 for value in row) for row in matrix)
assert tuple(
    item.coefficient for item in exact_silhouette_coefficients(scaled, (0, 0, 1)).samples
) == tuple(item.coefficient for item in result.samples)
assert checked == 5_000
```

## Trade-offs and Limitations

Validation and aggregation take `O(N²)` time for `N` samples. Beyond the
supplied dense matrix and `O(N)` result, each sample uses `O(K)` totals for `K`
clusters. The 512-sample cap bounds both matrix traversal and exact rational
arithmetic.

Singleton clusters receive coefficient zero and expose `within_mean=None`.
When both means are zero, the coefficient is also zero. Equal nearest-cluster
means need no label tie because the result intentionally omits a nearest
cluster identifier. Labels need not be consecutive, and relabeling does not
change scores.

Symmetry, nonnegativity, and a zero diagonal are enforced, but the triangle
inequality is not. This function evaluates one supplied partition; it does not
build clusters, choose their count, approximate large data, or provide a
sampling interpretation.

## Related Snippets

<!-- catalog:related:start -->
- [Compute an Exact Adjusted Rand Index from Two Bounded Integer Partitions](compute-an-exact-adjusted-rand-index-from-two-bounded-integer-partitions.md)
- [Fit Exact One-Dimensional k-Means with Contiguous-Cluster DP](fit-exact-one-dimensional-k-means-with-contiguous-cluster-dp.md)
- [Compute an Exact Normalized One-Dimensional Wasserstein Distance](compute-an-exact-normalized-one-dimensional-wasserstein-distance.md)
<!-- catalog:related:end -->
