---
title: "Overlay Aligned Time Series at the Finest Step"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - measure-time-in-a-state-within-a-half-open-window.md
  - ../algorithms-data-structures/find-a-point-in-disjoint-half-open-intervals.md
  - ../storage-databases/split-a-half-open-utc-range-across-ordered-storage-tiers.md
---

# Overlay Aligned Time Series at the Finest Step

## Idea and Problem

Overlay two regular in-memory series on their finest shared grid while using the coarser series only to fill missing or uncovered fine buckets.

Each value covers the half-open bucket beginning at its sample timestamp. The
series with the smaller step supplies the primary values; the left series has
precedence when the steps are equal. A numeric zero is a present value, while
only `None` requests a fallback from the other series.

## When to Use

Use this algorithm when both finite series are regular, their starts lie on
the same fine grid, and the coarser step is an exact multiple of the finer
step. It is useful for combining a detailed but sparse signal with a less
granular default over the union of their covered time ranges.

Resample with an explicit interpolation or aggregation policy when steps are
not aligned. Use a streaming or columnar implementation when the union would
not fit the fixed output limit.

## Implementation

```python
import math
from dataclasses import dataclass
from numbers import Real


_MAX_SERIES_VALUES = 100_000


@dataclass(frozen=True, slots=True)
class RegularSeries:
    start: int
    step: int
    values: tuple[float | None, ...]

    def __post_init__(self) -> None:
        if type(self.start) is not int:
            raise TypeError("start must be an integer")
        if type(self.step) is not int:
            raise TypeError("step must be an integer")
        if self.step <= 0:
            raise ValueError("step must be positive")
        if type(self.values) is not tuple:
            raise TypeError("values must be a tuple")
        if not 1 <= len(self.values) <= _MAX_SERIES_VALUES:
            raise ValueError("series length is outside the supported range")

        normalized: list[float | None] = []
        for value in self.values:
            if value is None:
                normalized.append(None)
                continue
            if isinstance(value, bool) or not isinstance(value, Real):
                raise TypeError("values must be real numbers or None")
            try:
                converted = float(value)
            except (OverflowError, TypeError, ValueError) as error:
                raise ValueError("values must convert to finite floats") from error
            if not math.isfinite(converted):
                raise ValueError("values must be finite")
            normalized.append(converted)

        object.__setattr__(self, "values", tuple(normalized))

    @property
    def stop(self) -> int:
        return self.start + self.step * len(self.values)


def _value_covering(series: RegularSeries, timestamp: int) -> float | None:
    offset = timestamp - series.start
    if offset < 0 or timestamp >= series.stop:
        return None
    return series.values[offset // series.step]


def overlay_at_finest_step(
    left: RegularSeries,
    right: RegularSeries,
) -> RegularSeries:
    if not isinstance(left, RegularSeries) or not isinstance(right, RegularSeries):
        raise TypeError("left and right must be RegularSeries instances")

    fine, coarse = (left, right) if left.step <= right.step else (right, left)
    if coarse.step % fine.step != 0:
        raise ValueError("the coarse step must be a multiple of the fine step")
    if (coarse.start - fine.start) % fine.step != 0:
        raise ValueError("series starts must lie on the same fine grid")

    output_start = min(left.start, right.start)
    output_stop = max(left.stop, right.stop)
    output_count = (output_stop - output_start) // fine.step
    if not 1 <= output_count <= _MAX_SERIES_VALUES:
        raise ValueError("output length is outside the supported range")

    def combined_value(index: int) -> float | None:
        timestamp = output_start + index * fine.step
        primary = _value_covering(fine, timestamp)
        if primary is not None:
            return primary
        return _value_covering(coarse, timestamp)

    values = tuple(combined_value(index) for index in range(output_count))
    return RegularSeries(start=output_start, step=fine.step, values=values)
```

## Example

```python
fine = RegularSeries(start=2, step=2, values=(None, 0.0, 4, None))
coarse = RegularSeries(start=0, step=4, values=(10, 20, 30))

combined = overlay_at_finest_step(fine, coarse)
equal_step = overlay_at_finest_step(
    RegularSeries(start=0, step=2, values=(None, 0.0)),
    RegularSeries(start=0, step=2, values=(8, 9)),
)

rejected = []
for invalid in (
    RegularSeries(start=0, step=3, values=(1,)),
    RegularSeries(start=1, step=4, values=(1,)),
):
    try:
        overlay_at_finest_step(fine, invalid)
    except ValueError:
        rejected.append(True)

try:
    overlay_at_finest_step(
        RegularSeries(start=0, step=1, values=(1,)),
        RegularSeries(start=_MAX_SERIES_VALUES, step=1, values=(2,)),
    )
except ValueError:
    oversized_union_rejected = True
else:
    oversized_union_rejected = False

assert (
    combined,
    equal_step.values,
    len(rejected),
    oversized_union_rejected,
) == (
    RegularSeries(0, 2, (10, 10, 0, 4, 30, 30)),
    (8.0, 0.0),
    2,
    True,
)
```

## Trade-offs and Limitations

Validation and overlay both run in `O(n)` time, and the result stores one new
tuple of at most `_MAX_SERIES_VALUES` floats or missing markers. The output
size is checked before that tuple is allocated. Input real numbers are
normalized to binary floats, so exact decimal or rational precision is not
preserved.

The algorithm performs no interpolation, downsampling, unit conversion, time
zone handling, or timestamp parsing. A coarse value covers its complete
half-open bucket, including portions outside the fine series. Consequently,
the output covers the union of both series rather than only their overlap.

## Related Snippets

<!-- catalog:related:start -->
- [Measure Time in a State Within a Half-Open Window](measure-time-in-a-state-within-a-half-open-window.md)
- [Find a Point in Disjoint Half-Open Intervals](../algorithms-data-structures/find-a-point-in-disjoint-half-open-intervals.md)
- [Split a Half-Open UTC Range Across Ordered Storage Tiers](../storage-databases/split-a-half-open-utc-range-across-ordered-storage-tiers.md)
<!-- catalog:related:end -->
