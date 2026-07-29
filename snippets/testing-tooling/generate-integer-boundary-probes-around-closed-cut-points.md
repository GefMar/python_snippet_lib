---
title: "Generate Integer Boundary Probes Around Closed Cut Points"
snippet_type: testing-technique
use_cases:
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - audit-a-bounded-test-matrix-for-complete-pairwise-coverage.md
  - generate-a-seeded-metric-with-bounded-flapping-runs.md
  - shrink-a-bounded-failing-sequence-to-a-one-deletion-minimal-subsequence.md
---

# Generate Integer Boundary Probes Around Closed Cut Points

## Idea and Problem

Generate a small deterministic set of signed-integer test inputs at, immediately below, and immediately above declared cut points.

Boundary behavior often changes at an inclusive limit or threshold. Including
the closed domain endpoints and the in-domain neighbors of every cut point
exercises those transitions without enumerating the complete integer domain.
Sorting and deduplicating the result makes duplicate or reordered cut points
irrelevant.

## When to Use

Use these probes when a bounded integer function has reviewed cut points where
validation, classification, or calculation behavior can change. The returned
tuple is useful as one input axis for focused unit tests, especially when the
full signed range is too large to enumerate.

Keep expected results in an independent test oracle. Add representative
interior values when behavior can vary away from the declared boundaries, and
use a separate combinatorial strategy when several input axes interact.

## Implementation

```python
_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_CUT_POINTS = 256


def generate_integer_boundary_probes(
    lower: int,
    upper: int,
    cut_points: tuple[int, ...],
) -> tuple[int, ...]:
    """Return sorted unique endpoints and in-domain cut-point neighbors."""
    if type(lower) is not int:
        raise TypeError("lower must be an exact non-boolean integer")
    if not _MIN_INT64 <= lower <= _MAX_INT64:
        raise ValueError("lower is outside the signed 64-bit range")

    if type(upper) is not int:
        raise TypeError("upper must be an exact non-boolean integer")
    if not _MIN_INT64 <= upper <= _MAX_INT64:
        raise ValueError("upper is outside the signed 64-bit range")
    if lower > upper:
        raise ValueError("lower must not exceed upper")

    if type(cut_points) is not tuple:
        raise TypeError("cut_points must be an exact tuple")
    if len(cut_points) > _MAX_CUT_POINTS:
        raise ValueError("cut_points exceeds the supported count")

    probes = {lower, upper}
    for index, cut in enumerate(cut_points):
        if type(cut) is not int:
            raise TypeError(f"cut_points[{index}] must be an exact non-boolean integer")
        if not _MIN_INT64 <= cut <= _MAX_INT64:
            raise ValueError(f"cut_points[{index}] is outside the signed 64-bit range")
        if not lower <= cut <= upper:
            raise ValueError(f"cut_points[{index}] is outside the closed domain")

        if cut > lower:
            probes.add(cut - 1)
        probes.add(cut)
        if cut < upper:
            probes.add(cut + 1)

    return tuple(sorted(probes))
```

## Example

```python
def direct_boundary_oracle(
    lower: int,
    upper: int,
    cut_points: tuple[int, ...],
) -> tuple[int, ...]:
    candidates = [lower]
    if upper != lower:
        candidates.append(upper)
    for cut in cut_points:
        for candidate in (cut - 1, cut, cut + 1):
            if lower <= candidate <= upper and candidate not in candidates:
                candidates.append(candidate)
    return tuple(sorted(candidates))


def exercise_small_domains() -> None:
    from itertools import product

    for small_lower in range(-2, 3):
        for small_upper in range(small_lower, 3):
            domain = tuple(range(small_lower, small_upper + 1))
            for cut_count in range(4):
                for small_cuts in product(domain, repeat=cut_count):
                    assert generate_integer_boundary_probes(
                        small_lower,
                        small_upper,
                        small_cuts,
                    ) == direct_boundary_oracle(
                        small_lower,
                        small_upper,
                        small_cuts,
                    )


exercise_small_domains()

assert generate_integer_boundary_probes(-5, 5, (2, 0, 2)) == (
    -5,
    -1,
    0,
    1,
    2,
    3,
    5,
)
assert generate_integer_boundary_probes(
    _MIN_INT64,
    _MAX_INT64,
    (_MIN_INT64, _MAX_INT64),
) == (_MIN_INT64, _MIN_INT64 + 1, _MAX_INT64 - 1, _MAX_INT64)
assert generate_integer_boundary_probes(
    _MIN_INT64,
    _MIN_INT64,
    (_MIN_INT64,),
) == (_MIN_INT64,)
assert generate_integer_boundary_probes(
    _MAX_INT64,
    _MAX_INT64,
    (_MAX_INT64,),
) == (_MAX_INT64,)
```

## Trade-offs and Limitations

For `K` cut-point entries, validation, set construction, and sorting take
`O(K log K)` time and use `O(K)` memory. The tuple contains at most `3K + 2`
values. Guarding each neighbor by the closed domain prevents either calculation
from crossing the admitted signed 64-bit range, including at `MIN_INT64` and
`MAX_INT64`.

The domain endpoints are always included, even when no cut points are supplied.
Duplicate cut points do not add probes, and cut-point order does not affect the
result. These values exercise only declared integer boundaries; they do not
generate expected results, random cases, floating-point neighbors, date or text
boundaries, property-based strategies, or combinatorial cross-products.

## Related Snippets

<!-- catalog:related:start -->
- [Audit a Bounded Test Matrix for Complete Pairwise Coverage](audit-a-bounded-test-matrix-for-complete-pairwise-coverage.md)
- [Generate a Seeded Metric with Bounded Flapping Runs](generate-a-seeded-metric-with-bounded-flapping-runs.md)
- [Shrink a Bounded Failing Sequence to a One-Deletion-Minimal Subsequence](shrink-a-bounded-failing-sequence-to-a-one-deletion-minimal-subsequence.md)
<!-- catalog:related:end -->
