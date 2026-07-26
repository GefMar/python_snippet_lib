---
title: "Sample a Weighted Stream with a Fixed-Size Reservoir"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - sample-a-stream-with-a-fixed-size-reservoir.md
  - sample-stream-items-independently-with-a-fixed-probability.md
---

# Sample a Weighted Stream with a Fixed-Size Reservoir

## Idea and Problem

Draw a fixed-size weighted sample without replacement from a finite one-pass stream while retaining only the current reservoir.

Each item receives a random log-priority divided by its positive weight, and a
min-heap retains the `k` greatest priorities. Higher weights make larger
priorities more likely without materializing the input. The injected random
generator makes a reviewed run reproducible, and selected items are returned
in their original stream order.

## When to Use

Use weighted reservoir sampling when the population length is unknown, items
cannot all be retained, and a documented positive weight should influence a
sample without replacement. Keep the weight scale and reference data fixed
for reproducible replays. For recency weighting, compute every weight against
one captured reference time; do not read a moving clock inside the callback.

## Implementation

```python
import heapq
import math
from collections.abc import Callable, Iterable
from random import Random
from typing import TypeVar


ItemT = TypeVar("ItemT")


def weighted_reservoir_sample(
    items: Iterable[ItemT],
    sample_size: int,
    weight: Callable[[ItemT], float],
    *,
    rng: Random,
) -> list[ItemT]:
    if isinstance(sample_size, bool) or not isinstance(sample_size, int):
        raise TypeError("sample_size must be an integer")
    if sample_size < 0:
        raise ValueError("sample_size must be non-negative")
    if not callable(weight):
        raise TypeError("weight must be callable")
    if not isinstance(rng, Random):
        raise TypeError("rng must be an instance of random.Random")
    if sample_size == 0:
        return []

    reservoir: list[tuple[float, int, int, ItemT]] = []
    for source_index, item in enumerate(items):
        raw_weight = weight(item)
        if isinstance(raw_weight, bool) or not isinstance(raw_weight, (int, float)):
            raise TypeError("weights must be real numbers")
        item_weight = float(raw_weight)
        if not math.isfinite(item_weight) or item_weight <= 0.0:
            raise ValueError("weights must be finite and positive")

        priority = math.log1p(-rng.random()) / item_weight
        entry = (priority, -source_index, source_index, item)
        if len(reservoir) < sample_size:
            heapq.heappush(reservoir, entry)
        elif entry > reservoir[0]:
            heapq.heapreplace(reservoir, entry)

    return [entry[3] for entry in sorted(reservoir, key=lambda entry: entry[2])]
```

## Example

```python
weighted_items = [
    ("tiny", 0.5),
    ("small", 1.0),
    ("medium", 2.0),
    ("large", 4.0),
    ("largest", 8.0),
]
weight_calls: list[str] = []


def item_weight(item: tuple[str, float]) -> float:
    weight_calls.append(item[0])
    return item[1]


sample = weighted_reservoir_sample(
    weighted_items,
    3,
    item_weight,
    rng=Random(11),
)

source_consumed = False


def source() -> Iterable[int]:
    global source_consumed
    source_consumed = True
    yield 1


empty = weighted_reservoir_sample(
    source(),
    0,
    lambda item: float(item),
    rng=Random(1),
)

try:
    weighted_reservoir_sample(["invalid"], 1, lambda _item: 0.0, rng=Random(1))
except ValueError:
    zero_weight_rejected = True
else:
    zero_weight_rejected = False

assert (
    [name for name, _weight in sample],
    weight_calls,
    empty,
    source_consumed,
    zero_weight_rejected,
) == (
    ["small", "large", "largest"],
    ["tiny", "small", "medium", "large", "largest"],
    [],
    False,
    True,
)
```

## Trade-offs and Limitations

Processing `n` items costs `O(n log k)` time, `O(k)` memory, one weight call,
and one pseudorandom draw per item. The function must exhaust a finite stream
before returning, and its output is source-ordered rather than randomly
ordered. With `k > 1`, an item's inclusion probability is not simply its
weight divided by the total weight. Extreme weight ratios and finite floating-
point random values can collapse priorities into ties; earlier input wins an
exact tie. Filter intentionally excluded items before calling because zero and
negative weights are rejected. A seeded `Random` is reproducible, not
cryptographically secure, and changing Python's generator or the priority
formula changes exact seeded samples.

## Related Snippets

<!-- catalog:related:start -->
- [Sample a Stream with a Fixed-Size Reservoir](sample-a-stream-with-a-fixed-size-reservoir.md)
- [Sample Stream Items Independently with a Fixed Probability](sample-stream-items-independently-with-a-fixed-probability.md)
<!-- catalog:related:end -->
