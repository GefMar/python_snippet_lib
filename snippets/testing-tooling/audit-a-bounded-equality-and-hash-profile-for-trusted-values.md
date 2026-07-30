---
title: "Audit a Bounded Equality and Hash Profile for Trusted Values"
snippet_type: testing-technique
use_cases:
  - testing
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - audit-a-bounded-less-than-relation-for-strict-weak-ordering.md
  - compare-bounded-python-expressions-structurally-without-execution.md
  - ../python-language/return-notimplemented-for-unsupported-rich-comparisons.md
---

# Audit a Bounded Equality and Hash Profile for Trusted Values

## Idea and Problem

Exercise equality and hashing on a small fixture before relying on its values as dictionary keys or set members.

The audit evaluates every ordered equality pair twice, then checks reflexivity,
symmetry, and transitivity. Only a complete stable Boolean relation reaches the
hash phase, where each value is hashed twice and equal pairs must have equal
hashes. The first violation identifies only its kind and fixture indexes, so
the report never retains values, representations, exceptions, or type names.

## When to Use

Use this helper when testing a bounded collection of trusted domain values that
are intended to behave like conventional immutable value keys. It can expose a
forgotten `__hash__`, state-dependent methods, an asymmetric equality rule, or
equal objects whose hashes disagree.

This is deliberately stricter than Python's general rich-comparison protocol.
Comparison methods may return non-Boolean objects, and values such as NaNs are
valid Python objects even though they are not reflexively equal. Such behavior
is reported because it does not fit this reliable-key profile.

## Implementation

```python
from dataclasses import dataclass
from enum import StrEnum

_MAX_EQUALITY_HASH_ITEMS = 24


class EqualityHashViolationKind(StrEnum):
    EQUALITY_RAISED = "equality-raised"
    EQUALITY_NON_BOOLEAN = "equality-non-boolean"
    EQUALITY_UNSTABLE = "equality-unstable"
    NON_REFLEXIVE = "non-reflexive"
    NON_SYMMETRIC = "non-symmetric"
    NON_TRANSITIVE = "non-transitive"
    HASH_RAISED = "hash-raised"
    HASH_UNSTABLE = "hash-unstable"
    EQUAL_VALUES_DIFFERENT_HASHES = "equal-values-different-hashes"


@dataclass(frozen=True, slots=True)
class EqualityHashViolation:
    kind: EqualityHashViolationKind
    indexes: tuple[int, ...]


@dataclass(frozen=True, slots=True)
class EqualityHashAudit:
    violation: EqualityHashViolation | None

    @property
    def valid(self) -> bool:
        return self.violation is None


def audit_equality_and_hash_profile(
    values: tuple[object, ...],
) -> EqualityHashAudit:
    if type(values) is not tuple:
        raise TypeError("values must be an exact tuple")
    if not 1 <= len(values) <= _MAX_EQUALITY_HASH_ITEMS:
        raise ValueError("value count is outside 1..24")

    size = len(values)
    equality = [[False] * size for _ in values]
    for left_index, left in enumerate(values):
        for right_index, right in enumerate(values):
            try:
                first = left == right
            except Exception:
                return EqualityHashAudit(
                    EqualityHashViolation(
                        EqualityHashViolationKind.EQUALITY_RAISED,
                        (left_index, right_index),
                    )
                )
            if type(first) is not bool:
                return EqualityHashAudit(
                    EqualityHashViolation(
                        EqualityHashViolationKind.EQUALITY_NON_BOOLEAN,
                        (left_index, right_index),
                    )
                )
            try:
                second = left == right
            except Exception:
                return EqualityHashAudit(
                    EqualityHashViolation(
                        EqualityHashViolationKind.EQUALITY_RAISED,
                        (left_index, right_index),
                    )
                )
            if type(second) is not bool:
                return EqualityHashAudit(
                    EqualityHashViolation(
                        EqualityHashViolationKind.EQUALITY_NON_BOOLEAN,
                        (left_index, right_index),
                    )
                )
            if first != second:
                return EqualityHashAudit(
                    EqualityHashViolation(
                        EqualityHashViolationKind.EQUALITY_UNSTABLE,
                        (left_index, right_index),
                    )
                )
            equality[left_index][right_index] = first

    for index in range(size):
        if not equality[index][index]:
            return EqualityHashAudit(
                EqualityHashViolation(
                    EqualityHashViolationKind.NON_REFLEXIVE,
                    (index,),
                )
            )

    for left in range(size):
        for right in range(left + 1, size):
            if equality[left][right] != equality[right][left]:
                return EqualityHashAudit(
                    EqualityHashViolation(
                        EqualityHashViolationKind.NON_SYMMETRIC,
                        (left, right),
                    )
                )

    for left in range(size):
        for middle in range(size):
            for right in range(size):
                if len({left, middle, right}) < 3:
                    continue
                if equality[left][middle] and equality[middle][right] and not equality[left][right]:
                    return EqualityHashAudit(
                        EqualityHashViolation(
                            EqualityHashViolationKind.NON_TRANSITIVE,
                            (left, middle, right),
                        )
                    )

    hashes: list[int] = []
    for index, value in enumerate(values):
        try:
            first_hash = hash(value)
            second_hash = hash(value)
        except Exception:
            return EqualityHashAudit(
                EqualityHashViolation(
                    EqualityHashViolationKind.HASH_RAISED,
                    (index,),
                )
            )
        if first_hash != second_hash:
            return EqualityHashAudit(
                EqualityHashViolation(
                    EqualityHashViolationKind.HASH_UNSTABLE,
                    (index,),
                )
            )
        hashes.append(first_hash)

    for left in range(size):
        for right in range(left + 1, size):
            if equality[left][right] and hashes[left] != hashes[right]:
                return EqualityHashAudit(
                    EqualityHashViolation(
                        EqualityHashViolationKind.EQUAL_VALUES_DIFFERENT_HASHES,
                        (left, right),
                    )
                )

    return EqualityHashAudit(None)


```

## Example

```python
@dataclass(frozen=True, slots=True)
class GoodKey:
    namespace: str
    identifier: int


class DirectionalValue:
    def __init__(self, rank: int) -> None:
        self.rank = rank

    def __eq__(self, other: object) -> bool:
        if type(other) is not DirectionalValue:
            return False
        return self is other or self.rank < other.rank

    def __hash__(self) -> int:
        return self.rank


class NearbyValue:
    def __init__(self, rank: int) -> None:
        self.rank = rank

    def __eq__(self, other: object) -> bool:
        return type(other) is NearbyValue and abs(self.rank - other.rank) <= 1

    def __hash__(self) -> int:
        return 0


class MismatchedHash:
    def __init__(self, key: str, hash_value: int) -> None:
        self.key = key
        self.hash_value = hash_value

    def __eq__(self, other: object) -> bool:
        return type(other) is MismatchedHash and self.key == other.key

    def __hash__(self) -> int:
        return self.hash_value


good = audit_equality_and_hash_profile((GoodKey("jobs", 7), GoodKey("jobs", 7), GoodKey("jobs", 8)))
nan_result = audit_equality_and_hash_profile((float("nan"),))
directional = audit_equality_and_hash_profile((DirectionalValue(1), DirectionalValue(2)))
nearby = audit_equality_and_hash_profile((NearbyValue(1), NearbyValue(2), NearbyValue(3)))
hash_mismatch = audit_equality_and_hash_profile(
    (MismatchedHash("same", 10), MismatchedHash("same", 20))
)
unhashable = audit_equality_and_hash_profile(([],))

assert (
    good.valid
    and nan_result.violation == EqualityHashViolation(EqualityHashViolationKind.NON_REFLEXIVE, (0,))
    and directional.violation
    == EqualityHashViolation(EqualityHashViolationKind.NON_SYMMETRIC, (0, 1))
    and nearby.violation
    == EqualityHashViolation(
        EqualityHashViolationKind.NON_TRANSITIVE,
        (0, 1, 2),
    )
    and hash_mismatch.violation
    == EqualityHashViolation(
        EqualityHashViolationKind.EQUAL_VALUES_DIFFERENT_HASHES,
        (0, 1),
    )
    and unhashable.violation == EqualityHashViolation(EqualityHashViolationKind.HASH_RAISED, (0,))
)
```

## Trade-offs and Limitations

For `n` values, a complete equality pass performs exactly `2n²` evaluations
before law checks and, if they pass, two hash calls per value. An invalid
observation returns earlier. The audit's own work takes
`O(n³)` time and `O(n²)` memory; user-defined operations can cost more.

Equality and hashing execute trusted application code. A method can hang,
mutate state, perform external work, or raise a `BaseException`, none of which
this helper makes safe. Two immediate observations detect some instability but
cannot prove future behavior. The audit checks only the supplied fixture, and
hash collisions between unequal values remain valid.

## Related Snippets

<!-- catalog:related:start -->
- [Audit a Bounded Less-Than Relation for Strict Weak Ordering](audit-a-bounded-less-than-relation-for-strict-weak-ordering.md)
- [Compare Bounded Python Expressions Structurally without Execution](compare-bounded-python-expressions-structurally-without-execution.md)
- [Return NotImplemented for Unsupported Rich Comparisons](../python-language/return-notimplemented-for-unsupported-rich-comparisons.md)
<!-- catalog:related:end -->
