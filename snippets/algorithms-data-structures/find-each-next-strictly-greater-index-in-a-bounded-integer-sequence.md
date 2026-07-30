---
title: "Find Each Next Strictly Greater Index in a Bounded Integer Sequence"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - compute-full-window-trailing-maxima-with-a-monotonic-index-deque.md
  - find-the-canonical-largest-rectangle-under-a-bounded-integer-histogram.md
  - count-strict-inversions-in-a-bounded-integer-sequence.md
---

# Find Each Next Strictly Greater Index in a Bounded Integer Sequence

## Idea and Problem

Find the earliest later position with a strictly greater value for every index in one bounded integer sequence.

A stack holds indexes that have not yet seen a greater value. Its values are
non-increasing from bottom to top. When a new value is greater than the value
at the top index, the current index is the first possible answer for that
entry, so the entry is resolved and removed. Equal values remain together on
the stack because equality does not satisfy the strict comparison.

The result contains an index or `None` at every input position. Returning
indexes preserves both the matching value and its exact location without a
sentinel that could collide with an input value.

## When to Use

Use this scan when every element of one immutable sequence needs its nearest
strictly greater successor. It is useful for resolving the next threshold
crossing, comparing later capacities, or preparing index-based spans without
performing a separate suffix scan for each position.

Use a direct nested scan for tiny inputs when simplicity matters more than
linear scaling. Use a different stack condition for greater-or-equal queries,
and a different boundary model when the sequence is circular or arrives as an
unbounded stream.

## Implementation

```python
_MIN_NEXT_GREATER_INT = -(1 << 63)
_MAX_NEXT_GREATER_INT = (1 << 63) - 1
_MAX_NEXT_GREATER_VALUES = 262_144


def find_next_strictly_greater_indices(
    values: tuple[int, ...],
) -> tuple[int | None, ...]:
    """Return each earliest later index containing a strictly greater value."""
    if type(values) is not tuple:
        raise TypeError("values must be an exact tuple")
    if len(values) > _MAX_NEXT_GREATER_VALUES:
        raise ValueError("value count exceeds the supported limit")

    for index, value in enumerate(values):
        if type(value) is not int:
            raise TypeError(f"values[{index}] must be an exact non-boolean integer")
        if not _MIN_NEXT_GREATER_INT <= value <= _MAX_NEXT_GREATER_INT:
            raise ValueError(f"values[{index}] is outside the signed 64-bit range")

    next_indices: list[int | None] = [None] * len(values)
    unresolved: list[int] = []

    for index, value in enumerate(values):
        while unresolved and values[unresolved[-1]] < value:
            next_indices[unresolved.pop()] = index
        unresolved.append(index)

    return tuple(next_indices)
```

## Example

```python
def find_next_strictly_greater_indices_by_scan(
    values: tuple[int, ...],
) -> tuple[int | None, ...]:
    answers: list[int | None] = []
    for index, value in enumerate(values):
        answer = next(
            (
                later_index
                for later_index in range(index + 1, len(values))
                if values[later_index] > value
            ),
            None,
        )
        answers.append(answer)
    return tuple(answers)


def decode_short_sequence(
    encoding: int,
    length: int,
    alphabet: tuple[int, ...],
) -> tuple[int, ...]:
    values: list[int] = []
    for _ in range(length):
        encoding, alphabet_index = divmod(encoding, len(alphabet))
        values.append(alphabet[alphabet_index])
    return tuple(values)


checked_sequences = 0
alphabet = (-1, 0, 1)
for length in range(9):
    for encoding in range(len(alphabet) ** length):
        values = decode_short_sequence(encoding, length, alphabet)
        assert find_next_strictly_greater_indices(
            values
        ) == find_next_strictly_greater_indices_by_scan(values)
        checked_sequences += 1

cases = (
    (),
    (2, 1, 2, 4, 3),
    (3, 3, 4),
    (1, 2, 3),
    (3, 2, 1),
    (_MIN_NEXT_GREATER_INT, _MAX_NEXT_GREATER_INT),
)
results = tuple(find_next_strictly_greater_indices(values) for values in cases)

maximum_equal = (0,) * _MAX_NEXT_GREATER_VALUES
maximum_result = find_next_strictly_greater_indices(maximum_equal)

rejected = 0
for invalid_values in (
    (True,),
    (_MIN_NEXT_GREATER_INT - 1,),
    (_MAX_NEXT_GREATER_INT + 1,),
    (0,) * (_MAX_NEXT_GREATER_VALUES + 1),
):
    try:
        find_next_strictly_greater_indices(invalid_values)
    except (TypeError, ValueError):
        rejected += 1

assert (
    checked_sequences,
    results,
    len(maximum_result),
    maximum_result[0],
    maximum_result[-1],
    rejected,
) == (
    9_841,
    (
        (),
        (3, 2, 3, None, None),
        (2, 2, None),
        (1, 2, None),
        (None, None, None),
        (1, None),
    ),
    _MAX_NEXT_GREATER_VALUES,
    None,
    None,
    4,
)
```

## Trade-offs and Limitations

Validation and the monotonic-stack scan take `O(N)` time. Every index enters
the stack once and leaves it at most once. The unresolved stack, mutable result
list, and immutable returned tuple use `O(N)` memory; all three can briefly
coexist at return time.

The function accepts an exact tuple of at most 262,144 exact non-Boolean
signed-64-bit integers, including the empty tuple. Equal values do not resolve
one another, and every answer is the earliest qualifying later index rather
than merely an arbitrary greater value.

The input is one immutable snapshot. The implementation does not provide
greater-or-equal semantics, circular wraparound, value sentinels, streaming
finalization, mutable updates, custom keys, or a choice among later matches.

## Related Snippets

<!-- catalog:related:start -->
- [Compute Full-Window Trailing Maxima with a Monotonic Index Deque](compute-full-window-trailing-maxima-with-a-monotonic-index-deque.md)
- [Find the Canonical Largest Rectangle Under a Bounded Integer Histogram](find-the-canonical-largest-rectangle-under-a-bounded-integer-histogram.md)
- [Count Strict Inversions in a Bounded Integer Sequence](count-strict-inversions-in-a-bounded-integer-sequence.md)
<!-- catalog:related:end -->
