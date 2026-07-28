---
title: "Merge Bounded Sorted Integer Runs with Observable Source-Order Ties"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - sort-newline-terminated-binary-records-with-bounded-merge-passes.md
  - ../data-processing/join-bounded-strictly-increasing-sequences-by-the-latest-prior-timestamp.md
  - ../data-processing/aggregate-consecutive-values-into-weighted-runs.md
---

# Merge Bounded Sorted Integer Runs with Observable Source-Order Ties

## Idea and Problem

Merge several already sorted integer runs while retaining exactly where every output value appeared.

Each value is decorated lazily with its run declaration index and original item
index. Standard-library `heapq.merge` compares those triples, so values remain
primary while equal values use the earlier run and then their position inside
that run as explicit, observable ties. Complete validation happens before the
merge begins.

## When to Use

Use this algorithm when a small set of bounded, in-memory runs is already
non-decreasing and downstream code needs a deterministic merged snapshot with
source provenance. The returned indexes are useful for audit-friendly
reconciliation, deterministic fixtures, or later access to parallel metadata
owned by the caller.

Use `heapq.merge` directly when only values are needed and the input contract
is already trusted. Use an external merge process for files or data larger than
memory, and keep a lazy iterator contract when materializing the complete
result would be inappropriate.

## Implementation

```python
from collections.abc import Iterator
from dataclasses import dataclass
from heapq import merge

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_RUNS = 64
_MAX_TOTAL_VALUES = 20_000


@dataclass(frozen=True, slots=True)
class MergedInteger:
    value: int
    run_index: int
    item_index: int


def _decorated_run(
    run: tuple[int, ...],
    *,
    run_index: int,
) -> Iterator[tuple[int, int, int]]:
    for item_index, value in enumerate(run):
        yield value, run_index, item_index


def merge_sorted_integer_runs(
    runs: tuple[tuple[int, ...], ...],
) -> tuple[MergedInteger, ...]:
    """Return a provenance-preserving merge under explicit source-order ties."""
    if type(runs) is not tuple:
        raise TypeError("runs must be an exact tuple")
    if not 1 <= len(runs) <= _MAX_RUNS:
        raise ValueError("run count is outside the supported range")

    total_values = 0
    for run_index, run in enumerate(runs):
        if type(run) is not tuple:
            raise TypeError(f"runs[{run_index}] must be an exact tuple")
        if len(run) > _MAX_TOTAL_VALUES - total_values:
            raise ValueError("aggregate value count exceeds the supported limit")

        previous_value: int | None = None
        for item_index, value in enumerate(run):
            if type(value) is not int:
                raise TypeError(f"runs[{run_index}][{item_index}] must be an exact integer")
            if not _MIN_INT64 <= value <= _MAX_INT64:
                raise ValueError(
                    f"runs[{run_index}][{item_index}] is outside the signed 64-bit range"
                )
            if previous_value is not None and value < previous_value:
                raise ValueError(f"runs[{run_index}] must be non-decreasing")
            previous_value = value

        total_values += len(run)

    decorated_runs = (
        _decorated_run(run, run_index=run_index) for run_index, run in enumerate(runs)
    )
    return tuple(
        MergedInteger(value, run_index, item_index)
        for value, run_index, item_index in merge(*decorated_runs)
    )
```

## Example

```python
runs = (
    (1, 4, 4, 9),
    (),
    (1, 4, 7),
    (1, 4, 4, 10),
)
merged = merge_sorted_integer_runs(runs)
all_empty = merge_sorted_integer_runs(((), ()))

try:
    merge_sorted_integer_runs(((2, 1),))
except ValueError:
    decreasing_rejected = True
else:
    decreasing_rejected = False

assert (merged, all_empty, decreasing_rejected) == (
    (
        MergedInteger(1, 0, 0),
        MergedInteger(1, 2, 0),
        MergedInteger(1, 3, 0),
        MergedInteger(4, 0, 1),
        MergedInteger(4, 0, 2),
        MergedInteger(4, 2, 1),
        MergedInteger(4, 3, 1),
        MergedInteger(4, 3, 2),
        MergedInteger(7, 2, 2),
        MergedInteger(9, 0, 3),
        MergedInteger(10, 3, 3),
    ),
    (),
    True,
)
```

## Trade-offs and Limitations

Validation costs `O(N)` time. Merging `N` values from `K` runs costs
`O(N log K)` time and retains at most one decorated head per non-empty run,
while the required frozen result uses `O(N)` memory. Decoration is lazy inside
the merge, but this API deliberately materializes its complete bounded output.

Run order is policy: changing it can change the provenance order of equal
values even though the merged value sequence remains the same. Repeated values
inside one run retain their original item order. Empty runs and an all-empty
aggregate are valid, but the outer run tuple itself must be non-empty.

The function accepts only exact signed 64-bit integers and non-decreasing exact
tuples. It provides no arbitrary key, reverse ordering, custom heap, lazy public
iterator, file handling, externally sorted stream validation, or mutable
incremental state.

## Related Snippets

<!-- catalog:related:start -->
- [Sort Newline-Terminated Binary Records with Bounded Merge Passes](sort-newline-terminated-binary-records-with-bounded-merge-passes.md)
- [Join Bounded Strictly Increasing Sequences by the Latest Prior Timestamp](../data-processing/join-bounded-strictly-increasing-sequences-by-the-latest-prior-timestamp.md)
- [Aggregate Consecutive Values into Weighted Runs](../data-processing/aggregate-consecutive-values-into-weighted-runs.md)
<!-- catalog:related:end -->
