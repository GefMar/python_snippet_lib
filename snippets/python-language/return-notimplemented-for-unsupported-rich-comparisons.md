---
title: "Return NotImplemented for Unsupported Rich Comparisons"
snippet_type: idiom
use_cases:
  - interoperability
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - model-a-quantity-with-one-canonical-unit.md
  - dispatch-on-an-exact-tuple-of-argument-types.md
  - ../data-processing/check-a-value-against-an-asymmetric-tolerance-band.md
---

# Return NotImplemented for Unsupported Rich Comparisons

## Idea and Problem

Implement equality and ordering for one immutable value type without claiming that unrelated operands or subclasses are comparable.

A rich-comparison method should return the `NotImplemented` singleton when it
does not support the other operand. Python can then try the reflected method on
the other operand. If neither side implements the operation, equality falls
back to `False`, while ordering raises `TypeError` instead of inventing an
arbitrary result.

## When to Use

Use this idiom for a closed value type whose comparison domain is deliberately
limited to instances of that exact runtime type. The example admits only exact
integers from `-1_000_000` through `1_000_000` and defines only equality and
less-than comparison.

Return `False` directly when a supported same-type value is genuinely unequal.
Return `NotImplemented` when the operand type is unsupported so Python retains
its normal bilateral dispatch. Raise a domain error during construction rather
than from a later comparison.

## Implementation

```python
from collections.abc import Callable
from dataclasses import dataclass
from types import NotImplementedType

_MIN_BOUNDED_RANK = -1_000_000
_MAX_BOUNDED_RANK = 1_000_000


@dataclass(frozen=True, slots=True)
class BoundedRank:
    value: int

    def __post_init__(self) -> None:
        if type(self.value) is not int:
            raise TypeError("value must be an exact integer")
        if not _MIN_BOUNDED_RANK <= self.value <= _MAX_BOUNDED_RANK:
            raise ValueError("value is outside -1000000..1000000")

    def __eq__(self, other: object) -> bool | NotImplementedType:
        if type(self) is not BoundedRank or type(other) is not BoundedRank:
            return NotImplemented
        return self.value == other.value

    def __lt__(self, other: object) -> bool | NotImplementedType:
        if type(self) is not BoundedRank or type(other) is not BoundedRank:
            return NotImplemented
        return self.value < other.value
```

## Example

```python
class RankSubclass(BoundedRank):
    pass


class ReflectedUpperBound:
    def __init__(self, upper_bound: int) -> None:
        self.upper_bound = upper_bound
        self.calls = 0

    def __gt__(self, other: object) -> bool:
        self.calls += 1
        return type(other) is BoundedRank and other.value < self.upper_bound


def raises_type_error(operation: Callable[[], object]) -> bool:
    try:
        operation()
    except TypeError:
        return True
    return False


rank = BoundedRank(7)
same_rank = BoundedRank(7)
higher_rank = BoundedRank(8)
subclass_rank = RankSubclass(7)
reflected_bound = ReflectedUpperBound(10)

assert rank.__eq__("7") is NotImplemented
assert rank.__lt__("7") is NotImplemented
assert rank.__eq__(subclass_rank) is NotImplemented
assert subclass_rank.__eq__(rank) is NotImplemented

assert rank == same_rank
assert rank != higher_rank
assert rank < higher_rank
assert (rank == object()) is False
assert (rank == subclass_rank) is False
assert raises_type_error(lambda: rank < object())

assert (rank < reflected_bound) is True
assert reflected_bound.calls == 1

assert BoundedRank(_MIN_BOUNDED_RANK).value == _MIN_BOUNDED_RANK
assert BoundedRank(_MAX_BOUNDED_RANK).value == _MAX_BOUNDED_RANK
assert raises_type_error(lambda: BoundedRank(True))
```

## Trade-offs and Limitations

Construction and each supported comparison take `O(1)` time and space. Exact
type checks intentionally exclude subclasses, even when they inherit the same
fields and methods. Unsupported operands are not coerced to integers.

Only `__eq__` and `__lt__` are implemented. Other ordering expressions work
only when Python can negotiate them through one of those methods or a supported
method on the other operand; otherwise they raise `TypeError`. Never
truth-test a direct rich-comparison result because `NotImplemented` is a
dispatch signal, not a Boolean comparison answer.

## Related Snippets

<!-- catalog:related:start -->
- [Model a Quantity with One Canonical Unit](model-a-quantity-with-one-canonical-unit.md)
- [Dispatch on an Exact Tuple of Argument Types](dispatch-on-an-exact-tuple-of-argument-types.md)
- [Check a Value Against an Asymmetric Tolerance Band](../data-processing/check-a-value-against-an-asymmetric-tolerance-band.md)
<!-- catalog:related:end -->
