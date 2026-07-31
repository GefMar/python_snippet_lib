---
title: "Find an Exact Single Change Point in a Bounded Integer Sequence"
snippet_type: algorithm
use_cases:
  - data-transformation
  - testing
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - fit-exact-one-dimensional-k-means-with-contiguous-cluster-dp.md
  - detect-a-recent-drop-against-a-disjoint-pandas-baseline-window.md
  - fit-an-exact-ordinary-least-squares-line-to-bounded-integer-points.md
---

# Find an Exact Single Change Point in a Bounded Integer Sequence

## Idea and Problem

Find the earliest best place to split one ordered integer sequence into two constant-mean segments, using exact within-segment squared error.

For a segment with count `c`, total `s`, and squared total `q`, its mean is
`s / c` and its sum of squared deviations is `q - s**2 / c`. Running left
totals and whole-sequence totals therefore evaluate every permitted split in
constant auxiliary space. The input order remains meaningful and is never
sorted.

The selected split minimizes the sum of the two segment errors. Updating only
for a strictly smaller error preserves the earliest split on a tie. The result
reports a change point only when the two-segment fit strictly improves upon one
mean fitted to the complete sequence.

## When to Use

Use this algorithm for a bounded, already ordered series when one descriptive
level shift is a useful deterministic summary. Exact `Fraction` means and
squared errors make boundary selection reproducible, including when several
splits have the same cost.

`NoChangePoint` means only that no permitted split reduces the observed
least-squares error. It is not statistical evidence that the data-generating
process stayed constant. Without a penalty, minimum improvement, or sampling
model, ordinary noise whose best left and right means differ will usually
produce a positive improvement.

Use a time-series or change-point package when the work needs several changes,
online detection, penalties, significance or confidence statements,
seasonality, weights, missing observations, or a model of variance and
dependence.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MIN_SEQUENCE_LENGTH = 4
_MAX_SEQUENCE_LENGTH = 4_096


@dataclass(frozen=True, slots=True)
class NoChangePoint:
    mean: Fraction
    squared_error: Fraction


@dataclass(frozen=True, slots=True)
class ExactChangePoint:
    split: int
    left_mean: Fraction
    right_mean: Fraction
    segmented_squared_error: Fraction
    unsplit_squared_error: Fraction
    improvement: Fraction


def _exact_squared_error(
    count: int,
    total: int,
    squared_total: int,
) -> Fraction:
    numerator = count * squared_total - total * total
    if numerator < 0:
        raise AssertionError("an exact segment squared error cannot be negative")
    return Fraction(numerator, count)


def find_exact_single_change_point(
    values: tuple[int, ...],
    *,
    minimum_segment_size: int,
) -> NoChangePoint | ExactChangePoint:
    """Return the earliest exact best split, or evidence of no improvement."""
    if type(values) is not tuple:
        raise TypeError("values must be an exact tuple")
    if not _MIN_SEQUENCE_LENGTH <= len(values) <= _MAX_SEQUENCE_LENGTH:
        raise ValueError("value count is outside the supported range")
    if type(minimum_segment_size) is not int:
        raise TypeError("minimum_segment_size must be an exact integer")
    if not 1 <= minimum_segment_size <= len(values) // 2:
        raise ValueError("minimum_segment_size is outside the supported range")

    total = 0
    squared_total = 0
    for index, value in enumerate(values):
        if type(value) is not int:
            raise TypeError(f"values[{index}] must be an exact integer")
        if not _MIN_INT64 <= value <= _MAX_INT64:
            raise ValueError(f"values[{index}] is outside the signed 64-bit range")
        total += value
        squared_total += value * value

    value_count = len(values)
    unsplit_mean = Fraction(total, value_count)
    unsplit_squared_error = _exact_squared_error(
        value_count,
        total,
        squared_total,
    )

    last_split = value_count - minimum_segment_size
    left_total = 0
    left_squared_total = 0
    best_split: int | None = None
    best_left_total = 0
    best_segmented_error: Fraction | None = None

    for split in range(1, last_split + 1):
        value = values[split - 1]
        left_total += value
        left_squared_total += value * value
        if split < minimum_segment_size:
            continue

        right_count = value_count - split
        right_total = total - left_total
        right_squared_total = squared_total - left_squared_total
        segmented_error = _exact_squared_error(
            split,
            left_total,
            left_squared_total,
        ) + _exact_squared_error(
            right_count,
            right_total,
            right_squared_total,
        )
        if best_segmented_error is None or segmented_error < best_segmented_error:
            best_split = split
            best_left_total = left_total
            best_segmented_error = segmented_error

    if best_split is None or best_segmented_error is None:
        raise AssertionError("validated segment limits must permit a split")

    improvement = unsplit_squared_error - best_segmented_error
    if improvement < 0:
        raise AssertionError("a two-segment optimum cannot exceed the unsplit error")
    if improvement == 0:
        return NoChangePoint(
            mean=unsplit_mean,
            squared_error=unsplit_squared_error,
        )

    return ExactChangePoint(
        split=best_split,
        left_mean=Fraction(best_left_total, best_split),
        right_mean=Fraction(total - best_left_total, value_count - best_split),
        segmented_squared_error=best_segmented_error,
        unsplit_squared_error=unsplit_squared_error,
        improvement=improvement,
    )
```

## Example

```python
def direct_statistics(values: tuple[int, ...]) -> tuple[Fraction, Fraction]:
    mean = sum((Fraction(value) for value in values), Fraction()) / len(values)
    squared_error = sum(
        ((Fraction(value) - mean) ** 2 for value in values),
        Fraction(),
    )
    return mean, squared_error


def direct_oracle(
    values: tuple[int, ...],
    minimum_segment_size: int,
) -> NoChangePoint | ExactChangePoint:
    unsplit_mean, unsplit_error = direct_statistics(values)
    candidates = []
    for split in range(minimum_segment_size, len(values) - minimum_segment_size + 1):
        left_mean, left_error = direct_statistics(values[:split])
        right_mean, right_error = direct_statistics(values[split:])
        candidates.append((left_error + right_error, split, left_mean, right_mean))

    segmented_error, split, left_mean, right_mean = min(
        candidates,
        key=lambda candidate: (candidate[0], candidate[1]),
    )
    improvement = unsplit_error - segmented_error
    if improvement == 0:
        return NoChangePoint(unsplit_mean, unsplit_error)
    return ExactChangePoint(
        split=split,
        left_mean=left_mean,
        right_mean=right_mean,
        segmented_squared_error=segmented_error,
        unsplit_squared_error=unsplit_error,
        improvement=improvement,
    )


def exhaustive_inputs():
    from itertools import product

    for length in (4, 5):
        for candidate_values in product((-1, 0, 1), repeat=length):
            for minimum_size in range(1, length // 2 + 1):
                yield candidate_values, minimum_size


step = find_exact_single_change_point(
    (0, 0, 0, 10, 10, 10),
    minimum_segment_size=2,
)
constant = find_exact_single_change_point(
    (7, 7, 7, 7),
    minimum_segment_size=1,
)
earliest_tie = find_exact_single_change_point(
    (0, 1, 0, 1),
    minimum_segment_size=1,
)
extremes = find_exact_single_change_point(
    (_MIN_INT64, _MIN_INT64, _MAX_INT64, _MAX_INT64),
    minimum_segment_size=2,
)
maximum_count = find_exact_single_change_point(
    (0,) * _MAX_SEQUENCE_LENGTH,
    minimum_segment_size=_MAX_SEQUENCE_LENGTH // 2,
)
translated_step = find_exact_single_change_point(
    (17, 17, 17, 27, 27, 27),
    minimum_segment_size=2,
)
negated_step = find_exact_single_change_point(
    (0, 0, 0, -10, -10, -10),
    minimum_segment_size=2,
)

exhaustive_count = 0
for candidate_values, minimum_size in exhaustive_inputs():
    assert find_exact_single_change_point(
        candidate_values,
        minimum_segment_size=minimum_size,
    ) == direct_oracle(candidate_values, minimum_size)
    exhaustive_count += 1

invalid_calls = (
    (([0, 0, 1, 1],), 1, TypeError),
    (((0, 1, 2),), 1, ValueError),
    (((0,) * (_MAX_SEQUENCE_LENGTH + 1),), 1, ValueError),
    (((0, 0, 1, True),), 1, TypeError),
    (((0, 0, 1, _MAX_INT64 + 1),), 1, ValueError),
    (((0, 0, 1, 1),), True, TypeError),
    (((0, 0, 1, 1),), 0, ValueError),
    (((0, 0, 1, 1),), 3, ValueError),
)
rejected = 0
for positional, minimum_size, expected_error in invalid_calls:
    try:
        find_exact_single_change_point(
            *positional,
            minimum_segment_size=minimum_size,
        )
    except expected_error:
        rejected += 1

range_width = (1 << 64) - 1
assert (
    step,
    constant,
    earliest_tie,
    extremes,
    maximum_count,
    exhaustive_count,
    rejected,
) == (
    ExactChangePoint(
        3,
        Fraction(0),
        Fraction(10),
        Fraction(0),
        Fraction(150),
        Fraction(150),
    ),
    NoChangePoint(Fraction(7), Fraction(0)),
    ExactChangePoint(
        1,
        Fraction(0),
        Fraction(2, 3),
        Fraction(2, 3),
        Fraction(1),
        Fraction(1, 3),
    ),
    ExactChangePoint(
        2,
        Fraction(_MIN_INT64),
        Fraction(_MAX_INT64),
        Fraction(0),
        Fraction(range_width * range_width),
        Fraction(range_width * range_width),
    ),
    NoChangePoint(Fraction(0), Fraction(0)),
    648,
    len(invalid_calls),
)
assert isinstance(step, ExactChangePoint)
assert isinstance(translated_step, ExactChangePoint)
assert isinstance(negated_step, ExactChangePoint)
assert (
    translated_step.split,
    translated_step.left_mean,
    translated_step.right_mean,
    translated_step.segmented_squared_error,
    translated_step.improvement,
) == (
    step.split,
    step.left_mean + 17,
    step.right_mean + 17,
    step.segmented_squared_error,
    step.improvement,
)
assert (
    negated_step.split,
    negated_step.left_mean,
    negated_step.right_mean,
    negated_step.segmented_squared_error,
    negated_step.improvement,
) == (
    step.split,
    -step.left_mean,
    -step.right_mean,
    step.segmented_squared_error,
    step.improvement,
)
```

## Trade-offs and Limitations

Validation, whole-sequence accumulation, and the split scan each take `O(n)`
exact-arithmetic operations. Running totals and the retained best result use
`O(1)` auxiliary state beyond the input and returned dataclass. Python integer
and `Fraction` costs still grow with operand bit length; signed-64-bit inputs
can produce squared totals wider than 64 bits.

Every accepted split leaves at least `minimum_segment_size` observations on
both sides. Candidate splits are visited from left to right, and replacing the
incumbent only for a strictly smaller segmented error makes the earliest
minimum deterministic. Because one common mean is a valid special case of two
segment means, exact improvement cannot be negative; a negative value signals
an implementation invariant failure rather than a no-change result.

The algorithm models one abrupt change in mean under unweighted squared loss.
It does not detect a change in variance, fit slopes, sort the sequence, handle
floating-point or missing observations, select multiple changes, operate
online, attach a minimum practical effect, or return a p-value, confidence
interval, causal explanation, or forecast.

## Related Snippets

<!-- catalog:related:start -->
- [Fit Exact One-Dimensional k-Means with Contiguous-Cluster DP](fit-exact-one-dimensional-k-means-with-contiguous-cluster-dp.md)
- [Detect a Recent Drop Against a Disjoint pandas Baseline Window](detect-a-recent-drop-against-a-disjoint-pandas-baseline-window.md)
- [Fit an Exact Ordinary Least-Squares Line to Bounded Integer Points](fit-an-exact-ordinary-least-squares-line-to-bounded-integer-points.md)
<!-- catalog:related:end -->
