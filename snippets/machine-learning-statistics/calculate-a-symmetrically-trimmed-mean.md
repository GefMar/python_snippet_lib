---
title: "Calculate a Symmetrically Trimmed Mean"
snippet_type: recipe
use_cases:
  - data-transformation
  - performance-optimization
tested_python:
  - "3.14"
dependencies: []
related: []
---

# Calculate a Symmetrically Trimmed Mean

## Idea and Problem

Reduce the influence of extreme observations by removing an explicit count from both sorted ends before taking the mean.

An integer trim count makes the sample-size policy unambiguous: exactly that
many low and high observations are discarded. Finite real values are converted
to floats and sorted once. Scaling retained values by their largest magnitude
lets `math.fsum` calculate representable large means without overflowing on an
intermediate total.

## When to Use

Use this recipe for a small, finite sample when the same symmetric trimming
rule has been chosen before looking at individual extremes. It can provide a
stable summary for noisy measurements, but the caller must retain enough data
after trimming and understand that the result intentionally ignores tail
observations.

## Implementation

```python
import math
from collections.abc import Iterable
from numbers import Real


def symmetric_trimmed_mean(
    values: Iterable[Real],
    *,
    trim_each_side: int,
) -> float:
    if isinstance(trim_each_side, bool) or not isinstance(trim_each_side, int):
        raise TypeError("trim_each_side must be an integer")
    if trim_each_side < 0:
        raise ValueError("trim_each_side must be non-negative")

    samples: list[float] = []
    for value in values:
        if isinstance(value, bool) or not isinstance(value, Real):
            raise TypeError("values must contain real numbers")
        try:
            sample = float(value)
        except OverflowError as error:
            raise ValueError("values must be representable as finite floats") from error
        if not math.isfinite(sample):
            raise ValueError("values must be finite")
        samples.append(sample)

    if len(samples) <= 2 * trim_each_side:
        raise ValueError("trimming must leave at least one observation")

    samples.sort()
    stop = len(samples) - trim_each_side if trim_each_side else None
    retained = samples[trim_each_side:stop]
    scale = max(abs(sample) for sample in retained)
    if scale == 0.0:
        return 0.0
    normalized_mean = math.fsum(sample / scale for sample in retained) / len(
        retained
    )
    return normalized_mean * scale
```

## Example

```python
import math


measurements = (value for value in [10.0, 10.2, 9.8, 80.0, -20.0])
trimmed = symmetric_trimmed_mean(measurements, trim_each_side=1)
ordinary = symmetric_trimmed_mean([1, 2, 3], trim_each_side=0)
duplicates = symmetric_trimmed_mean([0, 0, 5, 10, 10], trim_each_side=2)
large = symmetric_trimmed_mean([1e308, 1e308], trim_each_side=0)

try:
    symmetric_trimmed_mean([1.0, 2.0], trim_each_side=1)
except ValueError:
    empty_remainder_rejected = True
else:
    empty_remainder_rejected = False

try:
    symmetric_trimmed_mean([1.0, math.inf, 2.0], trim_each_side=0)
except ValueError:
    infinite_value_rejected = True
else:
    infinite_value_rejected = False

assert (
    trimmed,
    ordinary,
    duplicates,
    large,
    empty_remainder_rejected,
    infinite_value_rejected,
) == (10.0, 2.0, 5.0, 1e308, True, True)
```

## Trade-offs and Limitations

Sorting and materializing the sample costs `O(n log n)` time and `O(n)` memory.
The trim count can discard meaningful tail behavior and produces unstable
comparisons when different samples use different effective proportions. A
trimmed mean is not a percentile, confidence interval, anomaly detector, or
service-level latency metric; use a statistical method matched to those goals.
Conversion to float also loses precision and rejects finite values outside the
finite-float range.

## Related Snippets

<!-- catalog:related:start -->
No related snippets.
<!-- catalog:related:end -->
