---
title: "Count Strict Inversions in a Bounded Integer Sequence"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - merge-bounded-sorted-integer-runs-with-observable-source-order-ties.md
  - find-a-longest-strictly-increasing-integer-subsequence-with-earliest-index-ties.md
  - maintain-bounded-point-adds-and-half-open-range-sums-with-a-fenwick-tree.md
---

# Count Strict Inversions in a Bounded Integer Sequence

## Idea and Problem

Count index pairs that appear out of ascending order without materializing the pairs or changing the input sequence.

An inversion is a pair of indexes `i < j` for which `values[i] > values[j]`.
A bottom-up merge sort counts every such pair when a right-run value precedes
the unmerged suffix of the left run. Choosing the left value when both values
are equal keeps the merge stable and ensures equal values are never counted.

## When to Use

Use this algorithm to measure disorder in one bounded, in-memory snapshot of
integer values. It is useful when a quadratic pair scan is too expensive and
the exact number of strict inversions is required rather than the pairs
themselves.

Use a direct nested loop for very small inputs when simplicity matters more
than asymptotic performance. Use a Fenwick tree or another indexed structure
when values arrive incrementally, and use a domain-specific ranking method
when custom keys, tied ranks, or Kendall-style statistics define a different
notion of disagreement.

## Implementation

```python
_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_VALUE_COUNT = 100_000


def count_strict_inversions(values: tuple[int, ...]) -> int:
    """Return the number of pairs i < j for which values[i] > values[j]."""
    if type(values) is not tuple:
        raise TypeError("values must be an exact tuple")
    if len(values) > _MAX_VALUE_COUNT:
        raise ValueError("value count exceeds the supported limit")

    for index, value in enumerate(values):
        if type(value) is not int:
            raise TypeError(f"values[{index}] must be an exact non-boolean integer")
        if not _MIN_INT64 <= value <= _MAX_INT64:
            raise ValueError(f"values[{index}] is outside the signed 64-bit range")

    source = list(values)
    target = [0] * len(source)
    inversion_count = 0
    run_width = 1

    while run_width < len(source):
        for start in range(0, len(source), 2 * run_width):
            middle = min(start + run_width, len(source))
            stop = min(start + 2 * run_width, len(source))
            left = start
            right = middle
            destination = start

            while left < middle and right < stop:
                if source[left] <= source[right]:
                    target[destination] = source[left]
                    left += 1
                else:
                    target[destination] = source[right]
                    right += 1
                    inversion_count += middle - left
                destination += 1

            while left < middle:
                target[destination] = source[left]
                left += 1
                destination += 1
            while right < stop:
                target[destination] = source[right]
                right += 1
                destination += 1

        source, target = target, source
        run_width *= 2

    return inversion_count
```

## Example

```python
def count_strict_inversions_brute_force(values: tuple[int, ...]) -> int:
    total = 0
    for left_index in range(len(values)):
        for right_index in range(left_index + 1, len(values)):
            total += values[left_index] > values[right_index]
    return total


def decode_bounded_sequence(
    encoded: int,
    length: int,
    alphabet: tuple[int, ...],
) -> tuple[int, ...]:
    values: list[int] = []
    for _ in range(length):
        encoded, alphabet_index = divmod(encoded, len(alphabet))
        values.append(alphabet[alphabet_index])
    return tuple(values)


checked_cases = 0
alphabet = (-1, 0, 2)
for length in range(8):
    for encoded in range(len(alphabet) ** length):
        values = decode_bounded_sequence(encoded, length, alphabet)
        assert count_strict_inversions(values) == count_strict_inversions_brute_force(values)
        checked_cases += 1

duplicate_heavy = (3, 3, 2, 2, 1, 1)
duplicate_snapshot = duplicate_heavy
duplicate_count = count_strict_inversions(duplicate_heavy)
extrema_count = count_strict_inversions((_MAX_INT64, _MIN_INT64, _MIN_INT64))
descending_count = count_strict_inversions(tuple(range(64, 0, -1)))

try:
    count_strict_inversions((1, True))
except TypeError:
    boolean_rejected = True
else:
    boolean_rejected = False

try:
    count_strict_inversions((0,) * (_MAX_VALUE_COUNT + 1))
except ValueError:
    oversized_rejected = True
else:
    oversized_rejected = False

assert (
    duplicate_count,
    duplicate_heavy == duplicate_snapshot,
    extrema_count,
    descending_count,
    checked_cases,
    boolean_rejected,
    oversized_rejected,
) == (12, True, 2, 2_016, 3_280, True, True)
```

## Trade-offs and Limitations

Validation takes `O(n)` time and the iterative merge passes take `O(n log n)`
time. The two working lists use `O(n)` auxiliary memory. The returned count is
exact and can reach `n * (n - 1) // 2`; at the declared input limit that is
`4_999_950_000`.

The function accepts only an exact tuple containing at most 100,000 exact
signed 64-bit integers. It counts strict value inversions in a fixed snapshot;
it does not return the contributing pairs, accept custom comparison keys,
support mutable updates or streaming input, or implement a tied-rank
correlation statistic.

## Related Snippets

<!-- catalog:related:start -->
- [Merge Bounded Sorted Integer Runs with Observable Source-Order Ties](merge-bounded-sorted-integer-runs-with-observable-source-order-ties.md)
- [Find a Longest Strictly Increasing Integer Subsequence with Earliest-Index Ties](find-a-longest-strictly-increasing-integer-subsequence-with-earliest-index-ties.md)
- [Maintain Bounded Point Adds and Half-Open Range Sums with a Fenwick Tree](maintain-bounded-point-adds-and-half-open-range-sums-with-a-fenwick-tree.md)
<!-- catalog:related:end -->
