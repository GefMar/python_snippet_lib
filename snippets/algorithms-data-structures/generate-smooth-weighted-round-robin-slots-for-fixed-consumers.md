---
title: "Generate Smooth Weighted Round-Robin Slots for Fixed Consumers"
snippet_type: algorithm
use_cases:
  - data-transformation
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - apportion-a-bounded-integer-total-by-exact-weights-with-largest-remainders.md
  - ../data-processing/interleave-bounded-finite-tuple-sources-round-robin-until-exhausted.md
---

# Generate Smooth Weighted Round-Robin Slots for Fixed Consumers

## Idea and Problem

Schedule a finite sequence of equal-cost work slots across consumers with different weights, interleaving their turns.

Add each consumer's weight to its running score before selecting the next
slot. The highest score wins. Subtract the total of all weights from its
score to account for the slot it just received. When scores tie, the first
consumer in the input order wins.

## When to Use

Use this for a small, fixed set of consumers that stay eligible throughout
the schedule. Supply their positive integer weights in a stable order. Each
yielded index identifies the consumer for that slot. Choose the weights from
the intended long-run proportions for equal-cost work.

## Implementation

```python
from collections.abc import Iterator, Sequence


def weighted_slots(weights: Sequence[int], count: int) -> Iterator[int]:
    """Yield count smooth weighted round-robin choices as consumer indices."""
    if type(count) is not int or count < 0:
        raise ValueError("count must be a non-negative integer")
    fixed_weights = tuple(weights)
    if not fixed_weights or any(type(weight) is not int or weight <= 0 for weight in fixed_weights):
        raise ValueError("weights must be a nonempty sequence of positive integers")

    total = sum(fixed_weights)
    scores = [0] * len(fixed_weights)
    for _ in range(count):
        for index, weight in enumerate(fixed_weights):
            scores[index] += weight
        winner = max(range(len(scores)), key=scores.__getitem__)
        scores[winner] -= total
        yield winner
```

## Example

```python
queues = ("main", "review", "archive")
schedule = tuple(queues[index] for index in weighted_slots((5, 1, 1), count=7))

assert (schedule, tuple(weighted_slots((1, 1), count=4))) == (
    ("main", "main", "review", "main", "archive", "main", "main"),
    (0, 1, 0, 1),
)
```

## Trade-offs and Limitations

Each slot visits every consumer and the iterator retains one score per
consumer. Each new iterator starts its scores at zero, so use one iterator
for an uninterrupted sequence. Input validation happens when it is first
advanced.

Small prefixes may differ from a fixed-quota allocation. Use quota
apportionment when the final counts for one specific batch drive correctness.
Idle workers, changing weights, health checks and variable work costs require
a scheduler that accounts for those conditions.

## Related Snippets

<!-- catalog:related:start -->
- [Apportion a Bounded Integer Total by Exact Weights with Largest Remainders](apportion-a-bounded-integer-total-by-exact-weights-with-largest-remainders.md)
- [Interleave Bounded Finite Tuple Sources Round-Robin Until Exhausted](../data-processing/interleave-bounded-finite-tuple-sources-round-robin-until-exhausted.md)
<!-- catalog:related:end -->
