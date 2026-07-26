---
title: "Count Values in Fixed Upper-Bound Bins"
snippet_type: algorithm
use_cases:
  - data-transformation
  - observability
tested_python:
  - "3.14"
dependencies: []
related: []
---

# Count Values in Fixed Upper-Bound Bins

## Idea and Problem

Count finite observations in stable right-closed bins defined by a strictly increasing sequence of upper bounds.

For bounds `b0 < b1 < ...`, the bins are `(-infinity, b0]`, `(b0, b1]`,
and so on, followed by `(bn, +infinity)`. `bisect_left` places a value equal to
an upper bound in the bin ending at that bound, keeping the implementation and
the documented interval labels consistent.

## When to Use

Use fixed bins when monitoring or reports require stable boundaries across
runs and raw observations do not need to be retained. Choose bounds from an
external measurement policy before collecting data. The collector is suitable
for one owner thread or task; coordinate access explicitly when observations
arrive concurrently.

## Implementation

```python
import math
from bisect import bisect_left
from collections.abc import Iterable
from dataclasses import dataclass
from numbers import Real


@dataclass(frozen=True, slots=True)
class BinCount:
    lower_exclusive: float | None
    upper_inclusive: float | None
    count: int


def _finite_number(value: Real, *, name: str) -> float:
    if isinstance(value, bool) or not isinstance(value, Real):
        raise TypeError(f"{name} must be a real number")
    try:
        converted = float(value)
    except OverflowError as error:
        raise ValueError(f"{name} must fit in a finite float") from error
    if not math.isfinite(converted):
        raise ValueError(f"{name} must be finite")
    return converted


class FixedUpperBoundBins:
    def __init__(self, bounds: Iterable[Real]) -> None:
        normalized = tuple(
            _finite_number(bound, name="bound")
            for bound in bounds
        )
        if not normalized:
            raise ValueError("at least one upper bound is required")
        if any(right <= left for left, right in zip(normalized, normalized[1:])):
            raise ValueError("upper bounds must be strictly increasing")
        self.bounds = normalized
        self._counts = [0] * (len(normalized) + 1)

    @property
    def counts(self) -> tuple[int, ...]:
        return tuple(self._counts)

    def observe(self, value: Real) -> None:
        observation = _finite_number(value, name="observation")
        self._counts[bisect_left(self.bounds, observation)] += 1

    def snapshot(self) -> tuple[BinCount, ...]:
        return tuple(
            BinCount(
                lower_exclusive=None if index == 0 else self.bounds[index - 1],
                upper_inclusive=(
                    self.bounds[index] if index < len(self.bounds) else None
                ),
                count=count,
            )
            for index, count in enumerate(self._counts)
        )
```

## Example

```python
histogram = FixedUpperBoundBins((0, 10))
for observation in (-5, 0, 0.5, 10, 11):
    histogram.observe(observation)

try:
    FixedUpperBoundBins((1, 1))
except ValueError:
    duplicate_bound_rejected = True
else:
    duplicate_bound_rejected = False

try:
    histogram.observe(float("nan"))
except ValueError:
    non_finite_rejected = True
else:
    non_finite_rejected = False

assert (
    histogram.counts,
    histogram.snapshot(),
    sum(histogram.counts),
    duplicate_bound_rejected,
    non_finite_rejected,
) == (
    (2, 2, 1),
    (
        BinCount(None, 0.0, 2),
        BinCount(0.0, 10.0, 2),
        BinCount(10.0, None, 1),
    ),
    5,
    True,
    True,
)
```

## Trade-offs and Limitations

The collector stores counts only, so original values, quantiles, and rebucketing
are unavailable. Boundaries are float-based and distinct exact numbers can
collapse during conversion, in which case construction rejects them. Counts
grow without a rolling-window or reset policy and updates are not synchronized.
Export names, labels, cumulative conversion, and monitoring-backend semantics
remain the caller's responsibility.

## Related Snippets

<!-- catalog:related:start -->
No related snippets.
<!-- catalog:related:end -->
