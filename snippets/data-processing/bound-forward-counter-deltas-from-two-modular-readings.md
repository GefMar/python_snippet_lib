---
title: "Bound Forward Counter Deltas from Two Modular Readings"
snippet_type: recipe
use_cases:
  - data-transformation
  - observability
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../networking-protocols/unwrap-one-uint32-serial-around-an-explicit-absolute-reference.md
  - ../observability-operations/measure-cache-hit-ratios-from-monotonic-counter-snapshots.md
---

# Bound Forward Counter Deltas from Two Modular Readings

## Idea and Problem

Summarize every possible forward increment consistent with two readings of a wrapping counter and a known upper bound.

The smallest possible delta is the forward modular difference. Every other
candidate adds a positive multiple of the modulus. The upper bound determines
the last candidate and the number of candidates without visiting each one.
A single candidate establishes an exact delta under the supplied assumptions.

## When to Use

Use this when both readings belong to one uninterrupted nondecreasing
counter, with a known wrap modulus and an inclusive upper bound on the
increment. A maximum event rate and a reliable elapsed-time bound can provide
that limit. The caller must establish continuity across counter restarts
before using these readings together.

## Implementation

```python
from dataclasses import dataclass


@dataclass(frozen=True, slots=True)
class ForwardDeltas:
    first: int
    last: int
    count: int


def bound_forward_deltas(
    previous: int, current: int, *, modulus: int, max_delta: int
) -> ForwardDeltas | None:
    """Summarize the feasible progression, whose step equals modulus."""
    if any(type(value) is not int for value in (previous, current, modulus, max_delta)):
        raise TypeError("readings, modulus and max_delta must be integers")
    if modulus < 2 or max_delta < 0:
        raise ValueError("modulus must exceed one and max_delta must be non-negative")
    if not (0 <= previous < modulus and 0 <= current < modulus):
        raise ValueError("readings must lie in [0, modulus)")

    first = (current - previous) % modulus
    if first > max_delta:
        return None
    extra_cycles = (max_delta - first) // modulus
    return ForwardDeltas(first, first + extra_cycles * modulus, extra_cycles + 1)
```

## Example

```python
assert (
    bound_forward_deltas(250, 3, modulus=256, max_delta=10),
    bound_forward_deltas(250, 3, modulus=256, max_delta=530),
    bound_forward_deltas(250, 3, modulus=256, max_delta=8),
    bound_forward_deltas(12, 12, modulus=256, max_delta=256),
) == (ForwardDeltas(9, 9, 1), ForwardDeltas(9, 521, 3), None, ForwardDeltas(0, 256, 2))
```

## Trade-offs and Limitations

The result uses a fixed number of integers regardless of the number of
possible wraps. The candidates are exactly `count` terms from `first` to
`last` in steps of `modulus`. Values between consecutive terms are outside
the solution set.

Repeated readings can indicate zero progress or one or more complete wraps.
If several candidates remain, use additional sampling or a stronger bound
before choosing one. An impossible result points to incompatible readings,
an incorrect bound or a failed assumption about the counter epoch.

## Related Snippets

<!-- catalog:related:start -->
- [Unwrap One uint32 Serial Around an Explicit Absolute Reference](../networking-protocols/unwrap-one-uint32-serial-around-an-explicit-absolute-reference.md)
- [Measure Cache Hit Ratios from Monotonic Counter Snapshots](../observability-operations/measure-cache-hit-ratios-from-monotonic-counter-snapshots.md)
<!-- catalog:related:end -->
