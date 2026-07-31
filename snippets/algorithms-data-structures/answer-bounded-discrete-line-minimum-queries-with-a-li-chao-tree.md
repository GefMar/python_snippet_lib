---
title: "Answer Bounded Discrete Line-Minimum Queries with a Li Chao Tree"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - maintain-point-replacements-and-half-open-range-minima-with-a-segment-tree.md
  - answer-static-half-open-range-minimum-queries-with-a-sparse-table.md
  - ../machine-learning-statistics/fit-an-exact-ordinary-least-squares-line-to-bounded-integer-points.md
---

# Answer Bounded Discrete Line-Minimum Queries with a Li Chao Tree

## Idea and Problem

Find the minimum among many integer lines at each point in one known discrete coordinate domain without scanning every line per query.

A Li Chao tree stores one line at each interval node. During insertion, the
line that is better at the interval midpoint stays at that node; the other can
still win on at most one side and descends there. A query follows one
root-to-leaf path and compares every stored candidate on that path.

Treating `(line value, declaration index)` as the comparison key makes exact
intersections, equal slopes, and identical lines deterministic. The coordinate
domain is the supplied sorted query tuple, so no empty numeric range or
floating-point midpoint is involved.

## When to Use

Use this static form when every line applies to the whole domain, all query
coordinates are known before construction, and exact minima must be answered
for many sparse integer coordinates. It is especially useful when a direct
`lines × queries` scan is too expensive but coordinate compression is natural.

Use a convex-hull trick when slopes or queries arrive in an order that gives a
stronger monotonicity guarantee. Use a segment tree of Li Chao trees for line
segments or interval-restricted lifetimes. A direct scan is clearer for tiny
inputs.

## Implementation

```python
from dataclasses import dataclass
from random import Random

_MAX_LI_CHAO_ITEMS = 4_096
_MIN_SIGNED_64 = -(2**63)
_MAX_SIGNED_64 = 2**63 - 1


@dataclass(frozen=True, slots=True)
class LineMinimum:
    x: int
    value: int
    line_index: int


def bounded_discrete_line_minima(
    lines: tuple[tuple[int, int], ...],
    query_xs: tuple[int, ...],
) -> tuple[LineMinimum, ...]:
    """Return exact line minima on one frozen, strictly increasing domain."""
    if type(lines) is not tuple:
        raise TypeError("lines must be an exact tuple")
    if not 1 <= len(lines) <= _MAX_LI_CHAO_ITEMS:
        raise ValueError("line count is outside 1..4096")

    for line_index, line in enumerate(lines):
        if type(line) is not tuple or len(line) != 2:
            raise TypeError(f"lines[{line_index}] must be an exact pair")
        slope, intercept = line
        if type(slope) is not int or type(intercept) is not int:
            raise TypeError("line coefficients must be exact integers")
        if not _MIN_SIGNED_64 <= slope <= _MAX_SIGNED_64:
            raise ValueError("line slope is outside the signed 64-bit range")
        if not _MIN_SIGNED_64 <= intercept <= _MAX_SIGNED_64:
            raise ValueError("line intercept is outside the signed 64-bit range")

    if type(query_xs) is not tuple:
        raise TypeError("query_xs must be an exact tuple")
    if not 1 <= len(query_xs) <= _MAX_LI_CHAO_ITEMS:
        raise ValueError("query count is outside 1..4096")
    previous_x: int | None = None
    for x in query_xs:
        if type(x) is not int:
            raise TypeError("query coordinates must be exact integers")
        if not _MIN_SIGNED_64 <= x <= _MAX_SIGNED_64:
            raise ValueError("query coordinate is outside the signed 64-bit range")
        if previous_x is not None and x <= previous_x:
            raise ValueError("query coordinates must be strictly increasing")
        previous_x = x

    tree: list[int | None] = [None] * (4 * len(query_xs))

    def key(line_index: int, x: int) -> tuple[int, int]:
        slope, intercept = lines[line_index]
        return slope * x + intercept, line_index

    def insert(node: int, left: int, right: int, incoming: int) -> None:
        incumbent = tree[node]
        if incumbent is None:
            tree[node] = incoming
            return

        middle = (left + right) // 2
        if key(incoming, query_xs[middle]) < key(incumbent, query_xs[middle]):
            tree[node], incoming = incoming, incumbent
            incumbent = tree[node]
        if right - left == 1:
            return
        if key(incoming, query_xs[left]) < key(incumbent, query_xs[left]):
            insert(node * 2, left, middle, incoming)
        elif key(incoming, query_xs[right - 1]) < key(
            incumbent,
            query_xs[right - 1],
        ):
            insert(node * 2 + 1, middle, right, incoming)

    for line_index in range(len(lines)):
        insert(1, 0, len(query_xs), line_index)

    answers: list[LineMinimum] = []
    for coordinate_index, x in enumerate(query_xs):
        node = 1
        left = 0
        right = len(query_xs)
        best: tuple[int, int] | None = None
        while True:
            line_index = tree[node]
            if line_index is not None:
                candidate = key(line_index, x)
                if best is None or candidate < best:
                    best = candidate
            if right - left == 1:
                break
            middle = (left + right) // 2
            if coordinate_index < middle:
                node = node * 2
                right = middle
            else:
                node = node * 2 + 1
                left = middle
        if best is None:
            raise AssertionError("every query must encounter an inserted line")
        answers.append(LineMinimum(x=x, value=best[0], line_index=best[1]))
    return tuple(answers)
```

## Example

```python
def direct_line_minima(
    lines: tuple[tuple[int, int], ...],
    query_xs: tuple[int, ...],
) -> tuple[LineMinimum, ...]:
    return tuple(
        LineMinimum(
            x,
            *min((slope * x + intercept, index) for index, (slope, intercept) in enumerate(lines)),
        )
        for x in query_xs
    )


sample_lines = ((2, 3), (-1, 8), (0, 4), (2, 3))
sample_xs = (-2, 0, 3, 8)
assert bounded_discrete_line_minima(sample_lines, sample_xs) == (
    LineMinimum(-2, -1, 0),
    LineMinimum(0, 3, 0),
    LineMinimum(3, 4, 2),
    LineMinimum(8, 0, 1),
)

rng = Random(0x1C_A0)
checked = 0
for _ in range(5_000):
    line_count = rng.randrange(1, 13)
    query_count = rng.randrange(1, 13)
    lines = tuple((rng.randrange(-8, 9), rng.randrange(-12, 13)) for _ in range(line_count))
    query_xs = tuple(sorted(rng.sample(range(-24, 25), query_count)))
    assert bounded_discrete_line_minima(lines, query_xs) == direct_line_minima(lines, query_xs)
    checked += 1

assert checked == 5_000
```

## Trade-offs and Limitations

Construction takes `O(L log Q)` time for `L` lines and `Q` frozen coordinates;
all answers then take `O(Q log Q)` time. The tree retains `O(Q)` indexes. Exact
Python integers avoid overflow in `slope * x + intercept`, although very large
products cost more than fixed-width arithmetic.

This implementation is static: coordinates must be supplied once, and every
line covers that entire discrete domain. It does not support deletions,
continuous-domain queries, or interval-restricted lines. The tie rule favors
the earliest declaration; it does not attempt to remove redundant lines.

## Related Snippets

<!-- catalog:related:start -->
- [Maintain Point Replacements and Half-Open Range Minima with a Segment Tree](maintain-point-replacements-and-half-open-range-minima-with-a-segment-tree.md)
- [Answer Static Half-Open Range-Minimum Queries with a Sparse Table](answer-static-half-open-range-minimum-queries-with-a-sparse-table.md)
- [Fit an Exact Ordinary Least-Squares Line to Bounded Integer Points](../machine-learning-statistics/fit-an-exact-ordinary-least-squares-line-to-bounded-integer-points.md)
<!-- catalog:related:end -->
