---
title: "Flag Groupwise Numeric Outliers with IQR Fences"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - fit-and-apply-fixed-quantile-clipping-bounds.md
  - calculate-a-symmetrically-trimmed-mean.md
  - ../data-processing/check-a-value-against-an-asymmetric-tolerance-band.md
---

# Flag Groupwise Numeric Outliers with IQR Fences

## Idea and Problem

Flag observations strictly outside per-group interquartile-range fences while preserving enough context to inspect every decision.

Each sufficiently large group is sorted independently. Its first and third
quartiles use the inclusive linear rule: for quantile `q`, interpolate at
position `q * (n - 1)` between adjacent sorted values. The lower and upper
fences are `Q1 - multiplier * IQR` and `Q3 + multiplier * IQR`. The result
contains immutable fences, complete candidate records with original input
indices, and group names that did not meet the requested sample-size threshold.

## When to Use

Use this algorithm as a bounded diagnostic when numeric observations belong to
meaningfully comparable groups and an IQR rule has been selected in advance.
Observation names must be unique, group labels must be stable, values must be
finite real numbers, and the multiplier must be positive and finite.

The minimum group size is explicit because quartiles from very small samples
are unstable. Candidate order follows input order, while fences and
insufficient-group names are sorted by group name, so repeated calls with the
same observations have deterministic results.

## Implementation

```python
import math
from collections.abc import Iterable
from dataclasses import dataclass
from itertools import islice
from numbers import Real


_MAX_OBSERVATIONS = 10_000
_MAX_GROUPS = 512
_MAX_NAME_CHARS = 120


@dataclass(frozen=True, slots=True)
class NumericObservation:
    name: str
    group: str
    value: Real


@dataclass(frozen=True, slots=True)
class IqrFence:
    group: str
    observation_count: int
    first_quartile: float
    third_quartile: float
    lower: float
    upper: float


@dataclass(frozen=True, slots=True)
class OutlierCandidate:
    index: int
    name: str
    group: str
    value: float


@dataclass(frozen=True, slots=True)
class GroupwiseOutlierReport:
    fences: tuple[IqrFence, ...]
    candidates: tuple[OutlierCandidate, ...]
    insufficient_groups: tuple[str, ...]


def _label(value: object, *, field: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be a string")
    if not 1 <= len(value) <= _MAX_NAME_CHARS:
        raise ValueError(f"{field} length is outside the supported range")
    if value != value.strip() or not value.isprintable():
        raise ValueError(f"{field} must be stripped printable text")
    return value


def _finite_float(value: object, *, field: str) -> float:
    if isinstance(value, bool) or not isinstance(value, Real):
        raise TypeError(f"{field} must be a real number")
    try:
        result = float(value)
    except OverflowError as error:
        raise ValueError(f"{field} must fit in a finite float") from error
    if not math.isfinite(result):
        raise ValueError(f"{field} must be finite")
    return result


def _inclusive_linear_quartile(sorted_values: list[float], q: float) -> float:
    position = q * (len(sorted_values) - 1)
    lower_index = math.floor(position)
    fraction = position - lower_index
    upper_index = min(lower_index + 1, len(sorted_values) - 1)
    return (
        (1.0 - fraction) * sorted_values[lower_index]
        + fraction * sorted_values[upper_index]
    )


def flag_groupwise_iqr_outliers(
    observations: Iterable[NumericObservation],
    *,
    multiplier: Real = 1.5,
    minimum_group_size: int = 4,
) -> GroupwiseOutlierReport:
    if isinstance(observations, (str, bytes)):
        raise TypeError("observations must be a non-text iterable")
    if isinstance(minimum_group_size, bool) or not isinstance(
        minimum_group_size, int
    ):
        raise TypeError("minimum_group_size must be an integer")
    if not 2 <= minimum_group_size <= _MAX_OBSERVATIONS:
        raise ValueError("minimum_group_size is outside the supported range")
    factor = _finite_float(multiplier, field="multiplier")
    if factor <= 0.0:
        raise ValueError("multiplier must be positive")

    snapshot = tuple(islice(observations, _MAX_OBSERVATIONS + 1))
    if len(snapshot) > _MAX_OBSERVATIONS:
        raise ValueError("too many observations")

    normalized: list[tuple[str, str, float]] = []
    values_by_group: dict[str, list[float]] = {}
    names: set[str] = set()
    for observation in snapshot:
        if type(observation) is not NumericObservation:
            raise TypeError("each item must be a NumericObservation")
        name = _label(observation.name, field="observation name")
        group = _label(observation.group, field="group")
        if name in names:
            raise ValueError("observation names must be unique")
        names.add(name)
        value = _finite_float(observation.value, field="observation value")
        normalized.append((name, group, value))
        values_by_group.setdefault(group, []).append(value)
        if len(values_by_group) > _MAX_GROUPS:
            raise ValueError("too many groups")

    fences_by_group: dict[str, IqrFence] = {}
    insufficient: list[str] = []
    for group in sorted(values_by_group):
        values = values_by_group[group]
        if len(values) < minimum_group_size:
            insufficient.append(group)
            continue
        ordered = sorted(values)
        first = _inclusive_linear_quartile(ordered, 0.25)
        third = _inclusive_linear_quartile(ordered, 0.75)
        spread = third - first
        lower = first - factor * spread
        upper = third + factor * spread
        if not all(map(math.isfinite, (spread, lower, upper))):
            raise ValueError("a group produces fences outside finite-float range")
        fences_by_group[group] = IqrFence(
            group=group,
            observation_count=len(values),
            first_quartile=first,
            third_quartile=third,
            lower=lower,
            upper=upper,
        )

    candidates = tuple(
        OutlierCandidate(index, name, group, value)
        for index, (name, group, value) in enumerate(normalized)
        if group in fences_by_group
        and (
            value < fences_by_group[group].lower
            or value > fences_by_group[group].upper
        )
    )
    return GroupwiseOutlierReport(
        fences=tuple(fences_by_group[group] for group in sorted(fences_by_group)),
        candidates=candidates,
        insufficient_groups=tuple(insufficient),
    )
```

## Example

```python
observations = [
    NumericObservation("north-1", "north", 1.0),
    NumericObservation("south-1", "south", 8.0),
    NumericObservation("north-2", "north", 1.0),
    NumericObservation("north-3", "north", 1.0),
    NumericObservation("south-2", "south", 9.0),
    NumericObservation("north-4", "north", 1.0),
    NumericObservation("north-5", "north", 5.0),
]
report = flag_groupwise_iqr_outliers(
    observations,
    multiplier=1.5,
    minimum_group_size=4,
)
north = report.fences[0]

# With zero IQR, a value exactly on the fence is retained and 5.0 is outside.
assert (
    north.group,
    north.first_quartile,
    north.third_quartile,
    north.lower,
    north.upper,
    report.candidates,
    report.insufficient_groups,
) == (
    "north",
    1.0,
    1.0,
    1.0,
    1.0,
    (OutlierCandidate(6, "north-5", "north", 5.0),),
    ("south",),
)
```

## Trade-offs and Limitations

Materializing observations and sorting each eligible group costs `O(n)` memory
and up to `O(n log n)` time overall. The implementation accepts empty input and
reports no fences; callers that require data should enforce that separately. It
also rejects a mathematically valid calculation when its fences cannot be
represented as finite floats.

An IQR of zero collapses both fences to one value, so every strictly different
value becomes a candidate. Small groups make interpolated quartiles unstable,
and skewed or heavy-tailed distributions can produce many legitimate tail
observations. This is a diagnostic, not an instruction to delete data. Review
candidate names and values with domain context, and use a model designed for
the distribution when IQR fences are not appropriate.

## Related Snippets

<!-- catalog:related:start -->
- [Fit and Apply Fixed Quantile Clipping Bounds](fit-and-apply-fixed-quantile-clipping-bounds.md)
- [Calculate a Symmetrically Trimmed Mean](calculate-a-symmetrically-trimmed-mean.md)
- [Check a Value Against an Asymmetric Tolerance Band](../data-processing/check-a-value-against-an-asymmetric-tolerance-band.md)
<!-- catalog:related:end -->
