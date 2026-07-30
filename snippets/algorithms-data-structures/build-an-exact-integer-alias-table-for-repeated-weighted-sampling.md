---
title: "Build an Exact Integer Alias Table for Repeated Weighted Sampling"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - testing
tested_python:
  - "3.14"
dependencies: []
related:
  - ../data-processing/sample-a-weighted-stream-with-a-fixed-size-reservoir.md
  - ../machine-learning-statistics/select-the-lower-weighted-median-of-bounded-integer-observations.md
  - apportion-a-non-negative-integer-total-without-rounding-drift.md
---

# Build an Exact Integer Alias Table for Repeated Weighted Sampling

## Idea and Problem

Preprocess bounded positive integer weights so each later weighted choice needs one uniform bounded integer draw and constant-time table lookup.

Vose's alias method gives every original index one equally likely column. Each
column keeps an integer threshold for its own index and an alias that receives
the rest of the column. Scaling each weight by the number of columns preserves
the distribution exactly: across all column-ticket cells, index `i` occupies
`len(weights) * weights[i]` cells. Integer thresholds avoid floating-point
rounding at the boundary.

## When to Use

Use an alias table when one immutable weighted distribution will serve many
independent draws and exact integer proportions matter. Obtain a uniform
integer in `range(table.draw_count)` from a separately governed random-number
source, then pass that draw to `resolve()`. This keeps random-source policy and
table mechanics independent and makes the lookup easy to test exhaustively.

Prefer a cumulative-weight scan for a small number of draws, because it avoids
preprocessing. Use reservoir sampling for an unknown-length stream, and use a
dedicated algorithm when sampling must be without replacement or weights can
change between choices.

## Implementation

```python
from collections import deque
from dataclasses import dataclass

_MAX_ALIAS_COLUMNS = 256
_MAX_ALIAS_WEIGHT = (1 << 31) - 1
_MAX_TOTAL_WEIGHT = (1 << 63) - 1


@dataclass(frozen=True, slots=True)
class IntegerAliasTable:
    thresholds: tuple[int, ...]
    aliases: tuple[int, ...]
    total_weight: int

    @property
    def draw_count(self) -> int:
        return len(self.thresholds) * self.total_weight

    def resolve(self, draw: int) -> int:
        """Resolve one uniform integer in range(draw_count) to a weighted index."""
        if type(draw) is not int:
            raise TypeError("draw must be an exact non-boolean integer")
        if not 0 <= draw < self.draw_count:
            raise ValueError("draw is outside the table's uniform range")

        column, ticket = divmod(draw, self.total_weight)
        if ticket < self.thresholds[column]:
            return column
        return self.aliases[column]


def build_integer_alias_table(weights: tuple[int, ...]) -> IntegerAliasTable:
    """Build a deterministic exact alias table for positive integer weights."""
    if type(weights) is not tuple:
        raise TypeError("weights must be an exact tuple")
    if not 1 <= len(weights) <= _MAX_ALIAS_COLUMNS:
        raise ValueError("weight count is outside the supported range")

    total_weight = 0
    for index, weight in enumerate(weights):
        if type(weight) is not int:
            raise TypeError(f"weights[{index}] must be an exact non-boolean integer")
        if not 1 <= weight <= _MAX_ALIAS_WEIGHT:
            raise ValueError(f"weights[{index}] is outside the supported range")
        total_weight += weight
        if total_weight > _MAX_TOTAL_WEIGHT:
            raise ValueError("total weight exceeds the signed 64-bit range")

    column_count = len(weights)
    scaled_weights = [weight * column_count for weight in weights]
    thresholds = [total_weight] * column_count
    aliases = list(range(column_count))
    underfull = deque(
        index for index, scaled in enumerate(scaled_weights) if scaled < total_weight
    )
    overfull = deque(
        index for index, scaled in enumerate(scaled_weights) if scaled > total_weight
    )

    while underfull and overfull:
        under_index = underfull.popleft()
        over_index = overfull.popleft()
        thresholds[under_index] = scaled_weights[under_index]
        aliases[under_index] = over_index

        scaled_weights[over_index] -= total_weight - scaled_weights[under_index]
        if scaled_weights[over_index] < total_weight:
            underfull.append(over_index)
        elif scaled_weights[over_index] > total_weight:
            overfull.append(over_index)

    if underfull or overfull:
        raise RuntimeError("exact alias construction invariant failed")
    return IntegerAliasTable(tuple(thresholds), tuple(aliases), total_weight)
```

## Example

```python
def exhaustive_alias_counts(weights: tuple[int, ...]) -> tuple[int, ...]:
    table = build_integer_alias_table(weights)
    observed = [0] * len(weights)
    for draw in range(table.draw_count):
        observed[table.resolve(draw)] += 1
    expected = tuple(len(weights) * weight for weight in weights)
    assert tuple(observed) == expected
    return tuple(observed)


small_cases = ((1,), (1, 2, 3), (4, 1, 1, 2), (2, 2, 2))
checked_cells = sum(
    sum(exhaustive_alias_counts(weights))
    for weights in small_cases
)

deterministic = build_integer_alias_table((1, 2, 3, 4))
maximum = build_integer_alias_table((_MAX_ALIAS_WEIGHT,) * _MAX_ALIAS_COLUMNS)

rejected = 0
for invalid_weights in ((), (True,), (0,), (_MAX_ALIAS_WEIGHT + 1,)):
    try:
        build_integer_alias_table(invalid_weights)
    except (TypeError, ValueError):
        rejected += 1

for invalid_draw in (True, -1, deterministic.draw_count):
    try:
        deterministic.resolve(invalid_draw)
    except (TypeError, ValueError):
        rejected += 1

assert (
    checked_cells,
    deterministic.thresholds,
    deterministic.aliases,
    deterministic.resolve(0),
    deterministic.resolve(deterministic.draw_count - 1),
    maximum.resolve(maximum.draw_count - 1),
    rejected,
) == (
    69,
    (4, 8, 6, 10),
    (2, 3, 3, 3),
    0,
    3,
    _MAX_ALIAS_COLUMNS - 1,
    7,
)
```

## Trade-offs and Limitations

Construction takes `O(n)` time and memory. FIFO underfull and overfull queues
make the table deterministic for a fixed input order; exact-full columns keep
the total threshold and a self alias. Each `resolve()` call takes `O(1)` time
and makes no random-number-generator call. Python integer arithmetic is exact,
but its cost still grows with operand bit width.

The 256-column, per-weight, and total-weight limits are explicit operational
bounds. A uniform flattened draw is essential: biased draws produce biased
choices. The table does not own, seed, validate, or make security claims about
the caller's random source.

The weights and table are immutable snapshots. This implementation does not
accept floating-point weights, update a distribution in place, coordinate
concurrent mutation, sample without replacement, or preserve an external
identity beyond each weight's tuple index.

## Related Snippets

<!-- catalog:related:start -->
- [Sample a Weighted Stream with a Fixed-Size Reservoir](../data-processing/sample-a-weighted-stream-with-a-fixed-size-reservoir.md)
- [Select the Lower Weighted Median of Bounded Integer Observations](../machine-learning-statistics/select-the-lower-weighted-median-of-bounded-integer-observations.md)
- [Apportion a Non-Negative Integer Total Without Rounding Drift](apportion-a-non-negative-integer-total-without-rounding-drift.md)
<!-- catalog:related:end -->
