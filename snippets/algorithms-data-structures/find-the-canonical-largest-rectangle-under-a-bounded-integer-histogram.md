---
title: "Find the Canonical Largest Rectangle Under a Bounded Integer Histogram"
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
  - maintain-point-replacements-and-half-open-range-minima-with-a-segment-tree.md
  - build-a-bounded-integer-summed-area-table-for-half-open-rectangle-sums.md
---

# Find the Canonical Largest Rectangle Under a Bounded Integer Histogram

## Idea and Problem

Find one deterministic maximum-area positive rectangle beneath a bounded histogram of unit-width integer bars.

For each positive height, the useful rectangle extends across the largest
contiguous span whose bars are at least that high. An increasing monotonic
stack records the earliest start for unresolved heights. A lower bar closes
every taller span, and a final zero sentinel closes the spans that reach the
end. Equal heights keep their earliest start instead of adding a duplicate
stack entry.

The result is `(start, stop, height, area)` with a half-open bar interval. It
maximizes area and then minimizes `(start, stop, height)` lexicographically, so
ties always have one canonical answer. Empty and all-zero histograms return
`None` because no positive rectangle exists.

## When to Use

Use this algorithm for one bounded in-memory histogram with equal-width bars
when both the maximum area and a deterministic witness are required. Typical
inputs include capacity, frequency, or occupancy distributions that have
already been normalized to non-negative integer heights.

Use a direct span enumeration for tiny inputs when the simpler implementation
is preferable. Use a range-minimum structure for many unrelated interval
queries or mutable heights. Choose a different model for variable-width bars,
online updates, two-dimensional rectangles, or enumeration of every optimum.

## Implementation

```python
_MAX_INT64 = (1 << 63) - 1
_MAX_BAR_COUNT = 100_000


def find_canonical_largest_histogram_rectangle(
    heights: tuple[int, ...],
) -> tuple[int, int, int, int] | None:
    """Return start, stop, height, and area for the canonical best rectangle."""
    if type(heights) is not tuple:
        raise TypeError("heights must be an exact tuple")
    if len(heights) > _MAX_BAR_COUNT:
        raise ValueError("bar count exceeds the supported limit")

    for index, height in enumerate(heights):
        if type(height) is not int:
            raise TypeError(f"heights[{index}] must be an exact non-boolean integer")
        if not 0 <= height <= _MAX_INT64:
            raise ValueError(f"heights[{index}] is outside the unsigned height range")

    stack: list[tuple[int, int]] = []
    best_key: tuple[int, int, int, int] | None = None

    for stop in range(len(heights) + 1):
        current_height = heights[stop] if stop < len(heights) else 0
        start = stop

        while stack and stack[-1][1] > current_height:
            candidate_start, height = stack.pop()
            area = height * (stop - candidate_start)
            candidate_key = (-area, candidate_start, stop, height)
            if best_key is None or candidate_key < best_key:
                best_key = candidate_key
            start = candidate_start

        if current_height > 0 and (not stack or stack[-1][1] < current_height):
            stack.append((start, current_height))

    if best_key is None:
        return None
    negative_area, start, stop, height = best_key
    return start, stop, height, -negative_area
```

## Example

```python
def find_largest_histogram_rectangle_brute_force(
    heights: tuple[int, ...],
) -> tuple[int, int, int, int] | None:
    best_key: tuple[int, int, int, int] | None = None
    for start in range(len(heights)):
        minimum_height = _MAX_INT64
        for stop in range(start + 1, len(heights) + 1):
            minimum_height = min(minimum_height, heights[stop - 1])
            if minimum_height == 0:
                continue
            area = minimum_height * (stop - start)
            candidate_key = (-area, start, stop, minimum_height)
            if best_key is None or candidate_key < best_key:
                best_key = candidate_key

    if best_key is None:
        return None
    negative_area, start, stop, height = best_key
    return start, stop, height, -negative_area


def decode_bounded_histogram(
    encoded: int,
    length: int,
    alphabet: tuple[int, ...],
) -> tuple[int, ...]:
    heights: list[int] = []
    for _ in range(length):
        encoded, alphabet_index = divmod(encoded, len(alphabet))
        heights.append(alphabet[alphabet_index])
    return tuple(heights)


checked_cases = 0
alphabet = (0, 1, 2, 3)
for length in range(8):
    for encoded in range(len(alphabet) ** length):
        heights = decode_bounded_histogram(encoded, length, alphabet)
        assert find_canonical_largest_histogram_rectangle(
            heights
        ) == find_largest_histogram_rectangle_brute_force(heights)
        checked_cases += 1

cases = (
    (2, 1, 5, 6, 2, 3),
    (3, 3, 3),
    (2, 0, 2),
    (1, 2, 3, 4),
    (4, 3, 2, 1),
    (),
    (0, 0, 0),
    (_MAX_INT64, _MAX_INT64),
)
results = tuple(find_canonical_largest_histogram_rectangle(case) for case in cases)

try:
    find_canonical_largest_histogram_rectangle((1, True))
except TypeError:
    boolean_rejected = True
else:
    boolean_rejected = False

try:
    find_canonical_largest_histogram_rectangle((1, -1))
except ValueError:
    negative_rejected = True
else:
    negative_rejected = False

try:
    find_canonical_largest_histogram_rectangle((0,) * (_MAX_BAR_COUNT + 1))
except ValueError:
    oversized_rejected = True
else:
    oversized_rejected = False

assert (results, checked_cases, boolean_rejected, negative_rejected, oversized_rejected) == (
    (
        (2, 4, 5, 10),
        (0, 3, 3, 9),
        (0, 1, 2, 2),
        (1, 4, 2, 6),
        (0, 2, 3, 6),
        None,
        None,
        (0, 2, _MAX_INT64, 2 * _MAX_INT64),
    ),
    21_845,
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

Validation and the monotonic-stack scan take `O(n)` time. Each positive stack
entry is pushed and popped at most once, and the stack uses `O(n)` auxiliary
memory in the worst case. Python integer arithmetic keeps the area exact even
when `height * width` exceeds the signed 64-bit range.

The function accepts only an exact tuple containing at most 100,000 exact
integers from zero through `2**63 - 1`. Bars have unit width, the interval is
half-open, and only positive rectangles compete. The result contains one
canonical optimum; the function does not support negative or floating-point
heights, variable bar widths, mutable or streaming updates, two-dimensional
matrices, or all-optimum enumeration.

## Related Snippets

<!-- catalog:related:start -->
- [Compute Full-Window Trailing Maxima with a Monotonic Index Deque](compute-full-window-trailing-maxima-with-a-monotonic-index-deque.md)
- [Maintain Point Replacements and Half-Open Range Minima with a Segment Tree](maintain-point-replacements-and-half-open-range-minima-with-a-segment-tree.md)
- [Build a Bounded Integer Summed-Area Table for Half-Open Rectangle Sums](build-a-bounded-integer-summed-area-table-for-half-open-rectangle-sums.md)
<!-- catalog:related:end -->
