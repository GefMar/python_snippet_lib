---
title: "Sample Stream Items Independently with a Fixed Probability"
snippet_type: algorithm
use_cases:
  - data-transformation
tested_python:
  - "3.14"
dependencies: []
related:
  - sample-a-stream-with-a-fixed-size-reservoir.md
---

# Sample Stream Items Independently with a Fixed Probability

## Idea and Problem

Select each stream item independently with a known probability without storing the unsampled input.

Bernoulli sampling makes one random decision per item and yields matches
lazily. Unlike a fixed-size reservoir, it keeps the inclusion probability
constant while allowing the final sample size to vary.

## When to Use

Use this algorithm when independent inclusion is more important than an exact
sample size, the stream can be consumed only once, and downstream work can
handle natural size variation. Inject a seeded random generator for repeatable
tests or simulations. Use reservoir sampling when exactly `k` items are
required from a stream of unknown length.

## Implementation

```python
import math
import random
from collections.abc import Iterable, Iterator
from typing import TypeVar


ItemT = TypeVar("ItemT")


def _validated_probability(value: float, *, allow_zero: bool) -> float:
    if isinstance(value, bool) or not isinstance(value, (int, float)):
        raise TypeError("probability must be numeric")
    probability = float(value)
    if not math.isfinite(probability):
        raise ValueError("probability must be finite")
    if allow_zero:
        if not 0.0 <= probability <= 1.0:
            raise ValueError("probability must be between 0 and 1")
    elif not 0.0 < probability <= 1.0:
        raise ValueError("probability must be greater than 0 and at most 1")
    return probability


def sample_independently(
    items: Iterable[ItemT],
    probability: float,
    *,
    rng: random.Random | None = None,
) -> Iterator[ItemT]:
    probability = _validated_probability(probability, allow_zero=True)
    source = random.Random() if rng is None else rng

    def generate() -> Iterator[ItemT]:
        for item in items:
            if source.random() < probability:
                yield item

    return generate()


def estimate_population(sample_count: int, probability: float) -> float:
    if isinstance(sample_count, bool) or not isinstance(sample_count, int):
        raise TypeError("sample_count must be an integer")
    if sample_count < 0:
        raise ValueError("sample_count must be non-negative")
    probability = _validated_probability(probability, allow_zero=False)
    try:
        estimate = sample_count / probability
    except OverflowError as error:
        raise OverflowError("population estimate is too large") from error
    if not math.isfinite(estimate):
        raise OverflowError("population estimate is too large")
    return estimate
```

## Example

```python
observed = []


def source():
    for value in range(10):
        observed.append(value)
        yield value


iterator = sample_independently(source(), 0.3, rng=random.Random(7))
was_lazy = observed == []
selected = list(iterator)

assert (
    was_lazy,
    observed,
    selected,
    list(sample_independently(range(4), 0.0, rng=random.Random(1))),
    list(sample_independently(range(4), 1.0, rng=random.Random(1))),
    estimate_population(4, 0.25),
) == (
    True,
    list(range(10)),
    [1, 3, 6, 8],
    [],
    [0, 1, 2, 3],
    16.0,
)
```

## Trade-offs and Limitations

The sample size is random and may be zero; `sample_count / probability` is only
a point estimate, not a confidence interval or a guarantee for one run. The
default generator is pseudorandom and this recipe provides no cryptographic,
weighted, stratified, or distributed-sampling guarantees. Every input still
costs one random draw. The iterator does not terminate on an infinite source;
with probability zero it consumes items without yielding, so callers must
bound consumption. Very small probabilities and huge counts can make the
floating-point estimate unrepresentable, in which case the helper raises
`OverflowError`.

## Related Snippets

<!-- catalog:related:start -->
- [Sample a Stream with a Fixed-Size Reservoir](sample-a-stream-with-a-fixed-size-reservoir.md)
<!-- catalog:related:end -->
