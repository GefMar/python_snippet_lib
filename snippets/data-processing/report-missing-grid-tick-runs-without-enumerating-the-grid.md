---
title: "Report Missing Grid-Tick Runs Without Enumerating the Grid"
snippet_type: algorithm
use_cases:
  - data-transformation
  - observability
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - overlay-aligned-time-series-at-the-finest-step.md
  - partition-strictly-increasing-integer-timestamps-into-idle-gap-sessions.md
---

# Report Missing Grid-Tick Runs Without Enumerating the Grid

## Idea and Problem

Report maximal runs of missing scheduled ticks from a sparse stream of observed ticks.

The expected grid contains `count` ticks starting at `start`, spaced `step`
units apart. Convert each observation to its grid index with `divmod()`.
Subtract consecutive index boundaries to find how many ticks are missing,
including runs before the first observation and after the last. Represent
each run by its first missing tick and the number of missing ticks.

## When to Use

Use this for completeness reports on periodic samples or scheduled batch
slots. Supply the expected start, step and count explicitly. Observations
must use the same integer clock, lie exactly on the grid and arrive in
strictly increasing order. Resolve timing jitter or duplicates before this
audit if the producer permits them.

## Implementation

```python
from collections.abc import Iterable, Iterator
from dataclasses import dataclass


@dataclass(frozen=True, slots=True)
class MissingRun:
    first_tick: int
    count: int


def missing_grid_runs(
    observed: Iterable[int], *, start: int, step: int, count: int
) -> Iterator[MissingRun]:
    """Yield maximal absent runs from the declared regular grid."""
    if type(start) is not int or type(step) is not int or type(count) is not int:
        raise TypeError("start, step and count must be integers")
    if step <= 0 or count < 0:
        raise ValueError("step must be positive and count must be non-negative")

    next_index = 0
    for tick in observed:
        if type(tick) is not int:
            raise TypeError("observed ticks must be integers")
        index, remainder = divmod(tick - start, step)
        if remainder or not next_index <= index < count:
            raise ValueError("observed ticks must belong to the grid and strictly increase")
        if next_index < index:
            yield MissingRun(start + next_index * step, index - next_index)
        next_index = index + 1

    if next_index < count:
        yield MissingRun(start + next_index * step, count - next_index)
```

## Example

```python
# Expected ticks are -3, 1, 5, 9, 13, 17, 21, 25.
gaps = tuple(missing_grid_runs(iter((1, 5, 17)), start=-3, step=4, count=8))
unobserved = tuple(missing_grid_runs((), start=10, step=2, count=10**12))

assert (gaps, unobserved) == (
    (MissingRun(-3, 1), MissingRun(9, 2), MissingRun(21, 2)),
    (MissingRun(10, 10**12),),
)
```

## Trade-offs and Limitations

The iterator keeps one next-expected index and visits each observation once.
An arbitrarily long gap yields one result, so the amount of work does not
grow with the number of missing ticks. An empty grid yields no runs when
there are also no observations.

Validation happens during iteration. A late invalid observation can raise
after earlier runs have been yielded, so stage results first if the consumer
requires all-or-nothing behavior. A trailing run becomes known after the
input is exhausted.

## Related Snippets

<!-- catalog:related:start -->
- [Overlay Aligned Time Series at the Finest Step](overlay-aligned-time-series-at-the-finest-step.md)
- [Partition Strictly Increasing Integer Timestamps into Idle-Gap Sessions](partition-strictly-increasing-integer-timestamps-into-idle-gap-sessions.md)
<!-- catalog:related:end -->
