---
title: "Sample a Stream with a Fixed-Size Reservoir"
snippet_type: algorithm
use_cases:
  - data-transformation
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - yield-stream-items-with-bounded-neighbor-context.md
---

# Sample a Stream with a Fixed-Size Reservoir

## Idea and Problem

Choose up to a fixed number of positions uniformly from a single-pass stream without knowing its length in advance.

The first `k` items fill a reservoir. For the item at one-based position `n`,
the algorithm chooses an integer in `[0, n)` and replaces a reservoir slot only
when that integer is below `k`. Consequently, every observed position has the
same probability of appearing in the final reservoir while memory remains
bounded by `k` items.

## When to Use

Use reservoir sampling for exploratory samples from large files, generators,
or other finite one-pass iterables when materializing the full input is
undesirable. Inject a dedicated `random.Random` so tests and reproducible jobs
control the pseudorandom sequence. A size of zero returns immediately without
consuming the source.

## Implementation

```python
from collections.abc import Iterable
from random import Random
from typing import TypeVar


ItemT = TypeVar("ItemT")


def reservoir_sample(
    items: Iterable[ItemT],
    sample_size: int,
    *,
    rng: Random,
) -> list[ItemT]:
    if isinstance(sample_size, bool) or not isinstance(sample_size, int):
        raise TypeError("sample_size must be an integer")
    if sample_size < 0:
        raise ValueError("sample_size must be non-negative")
    if not isinstance(rng, Random):
        raise TypeError("rng must be an instance of random.Random")
    if sample_size == 0:
        return []

    reservoir: list[ItemT] = []
    for seen_count, item in enumerate(items, start=1):
        if seen_count <= sample_size:
            reservoir.append(item)
            continue

        replacement_index = rng.randrange(seen_count)
        if replacement_index < sample_size:
            reservoir[replacement_index] = item

    return reservoir
```

## Example

```python
from collections.abc import Iterator
from random import Random


consumed = False


def source() -> Iterator[int]:
    global consumed
    consumed = True
    yield from range(3)


empty = reservoir_sample(source(), 0, rng=Random(1))
short = reservoir_sample(iter(["a", "b"]), 5, rng=Random(1))
sample = reservoir_sample(range(20), 5, rng=Random(7))

try:
    reservoir_sample(range(3), -1, rng=Random(1))
except ValueError:
    negative_size_rejected = True
else:
    negative_size_rejected = False

assert (
    empty,
    consumed,
    short,
    sample,
    negative_size_rejected,
) == ([], False, ["a", "b"], [14, 16, 17, 3, 4], True)
```

## Trade-offs and Limitations

The algorithm still reads all `n` items and performs `O(n)` random draws after
the reservoir fills, while storing `O(k)` references. Reservoir slot order is
not source order and is not an independent random permutation. A seeded
`random.Random` is reproducible but not cryptographically secure. Statistical
uniformity follows from the algorithm; avoid flaky tests that try to infer it
from a small number of random runs.

## Related Snippets

<!-- catalog:related:start -->
- [Yield Stream Items with Bounded Neighbor Context](yield-stream-items-with-bounded-neighbor-context.md)
<!-- catalog:related:end -->
