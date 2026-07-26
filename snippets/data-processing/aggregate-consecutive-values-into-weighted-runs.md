---
title: "Aggregate Consecutive Values into Weighted Runs"
snippet_type: algorithm
use_cases:
  - data-transformation
tested_python:
  - "3.14"
dependencies: []
related:
  - yield-stream-items-with-bounded-neighbor-context.md
---

# Aggregate Consecutive Values into Weighted Runs

## Idea and Problem

Combine adjacent equal keys in an ordered stream while preserving run order, item count, and a finite non-negative total weight.

Global grouping loses positional information when the same key appears again
later. A streaming run aggregator retains that distinction and uses only the
current run as working state. Initializing from the first item keeps `None`,
zero, and other falsey values valid as keys.

## When to Use

Use this algorithm when input order is already meaningful and only consecutive
equal values belong together, such as compacting observations or summarizing
encoded runs. The caller must define equality and supply a non-negative weight
for each item. Use global aggregation instead when positions and run boundaries
do not matter.

## Implementation

```python
from collections.abc import Iterable, Iterator
from dataclasses import dataclass
from math import isfinite
from typing import Generic, TypeVar


KeyT = TypeVar("KeyT")


@dataclass(frozen=True, slots=True)
class WeightedValue(Generic[KeyT]):
    key: KeyT
    weight: int | float


@dataclass(frozen=True, slots=True)
class WeightedRun(Generic[KeyT]):
    key: KeyT
    count: int
    total_weight: float


def _validated_weight(value: int | float) -> float:
    if isinstance(value, bool) or not isinstance(value, (int, float)):
        raise TypeError("weight must be an integer or float")
    try:
        normalized = float(value)
    except OverflowError as error:
        raise ValueError("weight must be finite") from error
    if not isfinite(normalized) or normalized < 0:
        raise ValueError("weight must be finite and non-negative")
    return normalized


def aggregate_weighted_runs(
    values: Iterable[WeightedValue[KeyT]],
) -> Iterator[WeightedRun[KeyT]]:
    iterator = iter(values)
    try:
        first = next(iterator)
    except StopIteration:
        return

    current_key = first.key
    current_count = 1
    current_weight = _validated_weight(first.weight)

    for item in iterator:
        item_weight = _validated_weight(item.weight)
        if item.key == current_key:
            current_count += 1
            combined_weight = current_weight + item_weight
            if not isfinite(combined_weight):
                raise ValueError("run total weight must remain finite")
            current_weight = combined_weight
            continue

        yield WeightedRun(current_key, current_count, current_weight)
        current_key = item.key
        current_count = 1
        current_weight = item_weight

    yield WeightedRun(current_key, current_count, current_weight)
```

## Example

```python
values = (
    WeightedValue(None, 1),
    WeightedValue(None, 0),
    WeightedValue("middle", 2.5),
    WeightedValue(None, 3),
)
runs = list(aggregate_weighted_runs(values))
empty = list(aggregate_weighted_runs(()))

try:
    list(aggregate_weighted_runs((WeightedValue("invalid", -1),)))
except ValueError:
    negative_rejected = True
else:
    negative_rejected = False

try:
    list(aggregate_weighted_runs((WeightedValue("invalid", float("nan")),)))
except ValueError:
    non_finite_rejected = True
else:
    non_finite_rejected = False

try:
    list(
        aggregate_weighted_runs(
            (WeightedValue("large", 1e308), WeightedValue("large", 1e308))
        )
    )
except ValueError:
    overflowing_total_rejected = True
else:
    overflowing_total_rejected = False

assert (
    runs,
    empty,
    negative_rejected,
    non_finite_rejected,
    overflowing_total_rejected,
) == (
    [
        WeightedRun(None, 2, 1.0),
        WeightedRun("middle", 1, 2.5),
        WeightedRun(None, 1, 3.0),
    ],
    [],
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

The iterator is lazy, so an invalid late item can raise after earlier runs have
already been consumed. Floating-point totals inherit normal rounding behavior;
use a domain-specific exact numeric type and validator when exact sums matter.
Key equality may execute user code or be expensive. The function does not sort
input, merge equal non-adjacent runs, infer weights, or compute ratios.

## Related Snippets

<!-- catalog:related:start -->
- [Yield Stream Items with Bounded Neighbor Context](yield-stream-items-with-bounded-neighbor-context.md)
<!-- catalog:related:end -->
