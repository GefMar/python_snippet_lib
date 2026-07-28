---
title: "Compute a Bounded Numerically Stable Log-Sum-Exp for Finite Floats"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - accumulate-and-merge-finite-mean-and-variance-statistics-under-a-count-limit.md
  - calculate-a-symmetrically-trimmed-mean.md
  - adjust-bounded-exact-p-values-with-holms-step-down-method.md
---

# Compute a Bounded Numerically Stable Log-Sum-Exp for Finite Floats

## Idea and Problem

Approximate the logarithm of a positive exponential sum without exponentiating its largest finite input directly.

For values `x`, the identity `log(sum(exp(x))) = peak +
log(sum(exp(x - peak)))` uses their maximum as `peak`. Every shifted value is
non-positive, so its exponential is at most one. `math.fsum` then combines the
bounded shifted contributions before the peak is restored.

## When to Use

Use this algorithm when a small in-memory tuple already contains finite
binary-floating-point log values, such as log probabilities or unnormalized
log weights, and only their log-domain sum is required. The complete tuple must
be available for validation before calculation.

Choose a numerical array library when values have axes, batches, gradients, or
device placement. Choose an exact or higher-precision representation when
binary-float rounding is unacceptable, and use a signed-log method when terms
can subtract rather than add.

## Implementation

```python
import math

_MAX_LOG_VALUE_COUNT = 10_000


def bounded_log_sum_exp(values: tuple[float, ...]) -> float:
    """Return an overflow-resistant floating log of a positive exponential sum."""
    if type(values) is not tuple:
        raise TypeError("values must be an exact tuple")
    if not 1 <= len(values) <= _MAX_LOG_VALUE_COUNT:
        raise ValueError("value count is outside the supported range")

    for index, value in enumerate(values):
        if type(value) is not float:
            raise TypeError(f"values[{index}] must be an exact float")
        if not math.isfinite(value):
            raise ValueError(f"values[{index}] must be finite")

    peak = max(values)
    shifted_sum = math.fsum(math.exp(value - peak) for value in values)
    result = peak + math.log(shifted_sum)
    if not math.isfinite(result):
        raise OverflowError("log-sum-exp result is not representable as a finite float")
    return result
```

## Example

```python
largest_finite = float.fromhex("0x1.fffffffffffffp+1023")
large = bounded_log_sum_exp((1_000.0, 1_000.0))
wide = bounded_log_sum_exp((-largest_finite, largest_finite))
single = bounded_log_sum_exp((-12.5,))

try:
    bounded_log_sum_exp((0.0, 1))
except TypeError:
    integer_rejected = True
else:
    integer_rejected = False

try:
    bounded_log_sum_exp((0.0, math.inf))
except ValueError:
    infinity_rejected = True
else:
    infinity_rejected = False

assert (
    math.isclose(large, 1_000.0 + math.log(2.0), rel_tol=0.0, abs_tol=1e-12),
    wide,
    single,
    integer_rejected,
    infinity_rejected,
) == (True, largest_finite, -12.5, True, True)
```

## Trade-offs and Limitations

Validation and calculation each take `O(n)` time. The input tuple is already
materialized; the generator avoids another Python list of contributions, while
`math.fsum` maintains implementation-owned partial sums. The 10,000-value cap
bounds both passes and that internal work.

The peak shift prevents exponent overflow for admitted inputs, but it does not
make the answer exact. Subtracting two finite extremes can produce negative
infinity, and exponentiating that shifted value correctly contributes zero.
Less extreme contributions can also underflow or round away. Input permutation,
platform `libm`, Python build, and intermediate rounding can change final bits.
The function rejects every non-finite input and provides no weights, axes,
softmax values, gradients, `Decimal` support, streaming state, or
cross-platform bitwise-reproducibility promise.

## Related Snippets

<!-- catalog:related:start -->
- [Accumulate and Merge Finite Mean and Variance Statistics Under a Count Limit](accumulate-and-merge-finite-mean-and-variance-statistics-under-a-count-limit.md)
- [Calculate a Symmetrically Trimmed Mean](calculate-a-symmetrically-trimmed-mean.md)
- [Adjust Bounded Exact P-Values with Holm's Step-Down Method](adjust-bounded-exact-p-values-with-holms-step-down-method.md)
<!-- catalog:related:end -->
