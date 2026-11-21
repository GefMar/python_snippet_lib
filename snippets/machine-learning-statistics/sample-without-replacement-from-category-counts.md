---
title: "Sample Without Replacement from Category Counts"
snippet_type: recipe
use_cases:
  - data-transformation
  - testing
tested_python:
  - "3.14"
dependencies: []
related:
  - ../data-processing/sample-a-stream-with-a-fixed-size-reservoir.md
  - ../data-processing/sample-a-weighted-stream-with-a-fixed-size-reservoir.md
---

# Sample Without Replacement from Category Counts

## Idea and Problem

Draw individual occurrences from a finite inventory while keeping the population in compact category counts.

Pass category names and their multiplicities to `Random.sample` using
`counts`. Each occurrence can be selected once. A category with several
occurrences can appear several times in the result, without building an
expanded list of the inventory.

## When to Use

Use this for simulations or test data when the starting count of each
category is known and a draw should remove one available occurrence from
that draw's pool. Pass an owned random generator and keep the category order
stable when reproducing a run.

## Implementation

```python
import sys
from collections.abc import Mapping
from random import Random


def sample_categories(inventory: Mapping[str, int], size: int, rng: Random) -> tuple[str, ...]:
    if type(size) is not int or size < 0:
        raise ValueError("sample size must be a non-negative integer")

    categories: list[str] = []
    counts: list[int] = []
    for category, count in inventory.items():
        if type(count) is not int or count < 0:
            raise ValueError("category counts must be non-negative integers")
        categories.append(category)
        counts.append(count)

    total = sum(counts)
    if total > sys.maxsize:
        raise ValueError("the implicit population exceeds the supported size")
    if size > total:
        raise ValueError("sample size exceeds the inventory")
    if size == 0:
        return ()

    return tuple(rng.sample(categories, k=size, counts=counts))
```

## Example

```python
from collections import Counter

inventory = {"basic": 5, "standard": 0, "premium": 3}
partial = sample_categories(inventory, 3, Random(17))
repeated = sample_categories(inventory, 3, Random(17))
full = sample_categories(inventory, 8, Random(23))
empty = sample_categories({"basic": 0}, 0, Random(29))

assert (
    len(partial),
    partial == repeated,
    Counter(partial) <= Counter(inventory),
    Counter(full),
    empty,
    inventory,
) == (
    3,
    True,
    True,
    Counter({"basic": 5, "premium": 3}),
    (),
    {"basic": 5, "standard": 0, "premium": 3},
)
```

## Trade-offs and Limitations

Each call samples from the supplied inventory and leaves its counts alone.
To make several draws share one decreasing inventory, update the counts
between calls. Returning a sample requires space for its selected items,
so a sample close to the population size can still be large.

The result contains labels rather than unique occurrence IDs. Supply an
explicit population when individual identities matter. Use a reservoir
algorithm when the total is unknown. Seeded sequences are reproducible
under the same runtime and inputs, including input category order.

## Related Snippets

<!-- catalog:related:start -->
- [Sample a Stream with a Fixed-Size Reservoir](../data-processing/sample-a-stream-with-a-fixed-size-reservoir.md)
- [Sample a Weighted Stream with a Fixed-Size Reservoir](../data-processing/sample-a-weighted-stream-with-a-fixed-size-reservoir.md)
<!-- catalog:related:end -->
