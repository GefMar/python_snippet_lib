---
title: "Find a Strict Majority with Boyer-Moore Voting"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related: []
---

# Find a Strict Majority with Boyer-Moore Voting

## Idea and Problem

Select a possible majority with constant auxiliary state, then verify that it occurs strictly more than half the time.

Boyer-Moore voting cancels different values to leave the only possible strict
majority candidate. A second pass is essential because a candidate can remain
even when no majority exists. Returning a result object distinguishes no
majority from a valid majority whose value is `None`.

## When to Use

Use this algorithm for a reiterable sequence when equality is available but
building a frequency dictionary would be undesirable or values may be
unhashable. The requirement must be a strict majority, not merely the most
common value. For a one-shot iterator, either materialize it deliberately or
use a different contract that can obtain a second pass.

## Implementation

```python
from collections.abc import Sequence
from dataclasses import dataclass
from typing import Generic, TypeVar


ValueT = TypeVar("ValueT")


@dataclass(frozen=True, slots=True)
class MajorityResult(Generic[ValueT]):
    value: ValueT
    occurrences: int
    total: int


def find_strict_majority(
    values: Sequence[ValueT],
) -> MajorityResult[ValueT] | None:
    if not values:
        return None

    candidate = values[0]
    balance = 0
    for value in values:
        if balance == 0:
            candidate = value
            balance = 1
        elif value == candidate:
            balance += 1
        else:
            balance -= 1

    occurrences = sum(value == candidate for value in values)
    if occurrences <= len(values) // 2:
        return None
    return MajorityResult(candidate, occurrences, len(values))
```

## Example

```python
none_majority = find_strict_majority((None, "other", None))
zero_majority = find_strict_majority((0, 1, 0, 0))
unhashable_majority = find_strict_majority(([1], [2], [1]))
tie = find_strict_majority(("left", "right"))
candidate_without_majority = find_strict_majority(("a", "b", "c", "a"))
empty = find_strict_majority(())

assert (
    none_majority,
    zero_majority,
    unhashable_majority,
    tie,
    candidate_without_majority,
    empty,
) == (
    MajorityResult(None, 2, 3),
    MajorityResult(0, 3, 4),
    MajorityResult([1], 2, 3),
    None,
    None,
    None,
)
```

## Trade-offs and Limitations

The algorithm makes two passes and therefore requires a stable, reiterable
sequence. It uses constant auxiliary state but still performs linear equality
work, which can be costly for complex values. It does not find a plurality,
handle weighted votes, describe ties, or protect against values that mutate
between passes. The verification pass must not be removed.

## Related Snippets

<!-- catalog:related:start -->
No related snippets.
<!-- catalog:related:end -->
