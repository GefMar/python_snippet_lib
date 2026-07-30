---
title: "Apportion a Bounded Integer Total by Exact Weights with Largest Remainders"
snippet_type: algorithm
use_cases:
  - data-transformation
  - resource-management
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - apportion-a-non-negative-integer-total-without-rounding-drift.md
  - partition-bounded-non-negative-weights-into-exactly-k-contiguous-groups-with-minimum-peak-load.md
  - ../configuration-serialization/convert-decimal-values-to-exact-minor-units.md
---

# Apportion a Bounded Integer Total by Exact Weights with Largest Remainders

## Idea and Problem

Turn proportional integer weights into whole-unit shares that add up to one exact total.

Independent rounding can lose or invent units. The largest-remainder method
first assigns the floor of every exact quota, then gives the remaining units
to the greatest fractional remainders. Keeping each remainder as an integer
numerator avoids floating-point comparisons.

Equal remainders are resolved by the smaller declaration index, so the same
inputs always produce the same allocation.

## When to Use

Use this algorithm when a fixed count of indivisible units must be distributed
in proportion to non-negative integer weights and exact total preservation is
more important than preserving a particular rounding convention. Examples
include bounded capacity shares, sample quotas, display buckets, or coarse
resource budgets whose weights are already agreed.

Use equal apportionment when recipients have no meaningful weights. Choose a
domain-specific optimizer when recipients also have minimums, maximums,
priorities, eligibility rules, or costs, because those constraints can make a
largest-remainder result infeasible or undesirable.

## Implementation

```python
from dataclasses import dataclass

_MAX_PARTS = 4_096
_MAX_SIGNED_64 = (1 << 63) - 1


@dataclass(frozen=True, slots=True)
class LargestRemainderApportionment:
    total: int
    total_weight: int
    shares: tuple[int, ...]
    floor_shares: tuple[int, ...]
    remainder_numerators: tuple[int, ...]
    awarded_indexes: tuple[int, ...]


def apportion_by_largest_remainders(
    total: int,
    weights: tuple[int, ...],
) -> LargestRemainderApportionment:
    """Return one deterministic exact-weight largest-remainder allocation."""
    if type(total) is not int:
        raise TypeError("total must be an exact integer")
    if not 0 <= total <= _MAX_SIGNED_64:
        raise ValueError("total is outside the supported range")
    if type(weights) is not tuple:
        raise TypeError("weights must be an exact tuple")
    if not 1 <= len(weights) <= _MAX_PARTS:
        raise ValueError("weight count is outside the supported range")

    for index, weight in enumerate(weights):
        if type(weight) is not int:
            raise TypeError(f"weights[{index}] must be an exact integer")
        if not 0 <= weight <= _MAX_SIGNED_64:
            raise ValueError(f"weights[{index}] is outside the supported range")

    total_weight = sum(weights)
    if not 1 <= total_weight <= _MAX_SIGNED_64:
        raise ValueError("aggregate weight is outside the supported range")

    floors: list[int] = []
    remainders: list[int] = []
    for weight in weights:
        floor, remainder = divmod(total * weight, total_weight)
        floors.append(floor)
        remainders.append(remainder)

    units_left = total - sum(floors)
    ranked_indexes = sorted(
        range(len(weights)),
        key=lambda index: (-remainders[index], index),
    )
    awarded_indexes = tuple(ranked_indexes[:units_left])

    shares = floors.copy()
    for index in awarded_indexes:
        shares[index] += 1

    return LargestRemainderApportionment(
        total=total,
        total_weight=total_weight,
        shares=tuple(shares),
        floor_shares=tuple(floors),
        remainder_numerators=tuple(remainders),
        awarded_indexes=awarded_indexes,
    )
```

## Example

```python


weighted = apportion_by_largest_remainders(7, (5, 3, 2))
tie = apportion_by_largest_remainders(2, (1, 1, 1))
scaled = apportion_by_largest_remainders(7, (50, 30, 20))
with_zero = apportion_by_largest_remainders(5, (0, 4, 1))

assert weighted.shares == (4, 2, 1)
assert weighted.floor_shares == (3, 2, 1)
assert weighted.remainder_numerators == (5, 1, 4)
assert weighted.awarded_indexes == (0,)
assert tie.shares == (1, 1, 0)
assert tie.awarded_indexes == (0, 1)
assert scaled.shares == weighted.shares
assert with_zero.shares == (0, 4, 1)
assert sum(weighted.shares) == weighted.total


def is_rejected(candidate_total: object, candidate_weights: object) -> bool:
    try:
        apportion_by_largest_remainders(  # type: ignore[arg-type]
            candidate_total,
            candidate_weights,
        )
    except TypeError:
        return True
    except ValueError:
        return True
    return False


assert is_rejected(True, (1,))
assert is_rejected(1, (0, 0))
assert is_rejected(1, [1])
```

## Trade-offs and Limitations

For `n` weights, validation and quota calculation take `O(n)` time, ranking
takes `O(n log n)` time, and the immutable evidence uses `O(n)` memory. Python
integers preserve every product and remainder exactly; the signed-64-bit input
and aggregate caps bound accepted values rather than intermediate arithmetic.

Each final share is either the floor or ceiling of its exact quota, zero-weight
recipients remain at zero, and multiplying all weights by the same admitted
positive factor preserves the result. Those properties do not make the method
universally fair. Increasing the total can make a recipient lose a unit, and
adding or removing a recipient can change unrelated shares. Stable index ties
also give earlier recipients a deliberate advantage when remainders match.

The function has no minimum-share, maximum-share, indivisible-bundle,
eligibility, stochastic tie, or multi-stage carry policy. The inputs must
already express the intended policy; the algorithm only performs one bounded
Hamilton-style apportionment.

## Related Snippets

<!-- catalog:related:start -->
- [Apportion a Non-Negative Integer Total Without Rounding Drift](apportion-a-non-negative-integer-total-without-rounding-drift.md)
- [Partition Bounded Non-Negative Weights into Exact-K Contiguous Groups by Minimum Peak Load](partition-bounded-non-negative-weights-into-exactly-k-contiguous-groups-with-minimum-peak-load.md)
- [Convert Decimal Values to Exact Minor Units](../configuration-serialization/convert-decimal-values-to-exact-minor-units.md)
<!-- catalog:related:end -->
