---
title: "Compute an Exact Fixed-Count Binary Runs Test with a Probability-Ordered Two-Sided P-Value"
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
  - compute-fishers-exact-test-with-a-probability-ordered-two-sided-p-value.md
  - compute-an-exact-paired-two-sided-sign-test.md
  - find-an-exact-single-change-point-in-a-bounded-integer-sequence.md
---

# Compute an Exact Fixed-Count Binary Runs Test with a Probability-Ordered Two-Sided P-Value

## Idea and Problem

Test the ordering of a bounded binary sequence against the exact null distribution obtained by uniformly rearranging its observed zeros and ones.

A run is one maximal block of equal values, so the observed run count is one
plus the number of changes between adjacent positions. Conditioning on the two
symbol counts makes every distinct arrangement equally likely under the null.
For each feasible run count, products of binomial coefficients count the ways
to split the zeros and ones into non-empty runs without enumerating those
arrangements.

The two-sided p-value sums every run-count probability mass less than or equal
to the observed run-count mass. Probability ties are included. This explicit
ordering matters because discrete two-sided conventions, such as doubling a
smaller tail, need not agree.

## When to Use

Use this test when an ordered sequence contains exactly two encoded outcomes,
the observed numbers of each outcome should be held fixed, and uniform order
under that fixed-count null is a defensible model. Exact `Fraction` evidence is
useful for small reference analyses and deterministic statistical tests.

Choose the null model and two-sided convention before inspecting the sequence.
Use a designed analysis when positions are not exchangeable under the null,
observations are clustered or weighted, sampling stops adaptively, or the
binary encoding was chosen after seeing the order. Use change-point analysis
when the location and magnitude of a transition, rather than a global ordering
test, are the objective.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction
from math import comb

_MAX_BINARY_VALUES = 200


@dataclass(frozen=True, slots=True)
class BinaryRunMass:
    run_count: int
    arrangement_count: int
    probability: Fraction


@dataclass(frozen=True, slots=True)
class BinaryRunsTestResult:
    sample_size: int
    zero_count: int
    one_count: int
    observed_runs: int
    total_arrangements: int
    observed_arrangement_count: int
    observed_run_probability: Fraction
    distribution: tuple[BinaryRunMass, ...]
    probability_ordered_two_sided_p_value: Fraction


def _positive_composition_count(total: int, part_count: int) -> int:
    if part_count == 0:
        return int(total == 0)
    if total < part_count:
        return 0
    return comb(total - 1, part_count - 1)


def _arrangements_with_run_count(
    zero_count: int,
    one_count: int,
    run_count: int,
) -> int:
    half_runs = run_count // 2
    if run_count % 2 == 0:
        return (
            2
            * _positive_composition_count(zero_count, half_runs)
            * _positive_composition_count(one_count, half_runs)
        )

    return _positive_composition_count(zero_count, half_runs + 1) * _positive_composition_count(
        one_count, half_runs
    ) + _positive_composition_count(zero_count, half_runs) * _positive_composition_count(
        one_count, half_runs + 1
    )


def exact_fixed_count_binary_runs_test(
    values: tuple[int, ...],
) -> BinaryRunsTestResult:
    """Return exact fixed-count run probabilities and a two-sided p-value."""
    if type(values) is not tuple:
        raise TypeError("values must be an exact tuple")
    if not 1 <= len(values) <= _MAX_BINARY_VALUES:
        raise ValueError("value count is outside the supported range")

    zero_count = 0
    observed_runs = 1
    previous: int | None = None
    for index, value in enumerate(values):
        if type(value) is not int:
            raise TypeError(f"values[{index}] must be an exact integer")
        if value not in (0, 1):
            raise ValueError(f"values[{index}] must be zero or one")
        if value == 0:
            zero_count += 1
        if previous is not None and value != previous:
            observed_runs += 1
        previous = value

    sample_size = len(values)
    one_count = sample_size - zero_count
    total_arrangements = comb(sample_size, one_count)
    masses: list[BinaryRunMass] = []
    observed_arrangement_count: int | None = None

    for run_count in range(1, sample_size + 1):
        arrangement_count = _arrangements_with_run_count(
            zero_count,
            one_count,
            run_count,
        )
        if arrangement_count == 0:
            continue
        masses.append(
            BinaryRunMass(
                run_count=run_count,
                arrangement_count=arrangement_count,
                probability=Fraction(arrangement_count, total_arrangements),
            )
        )
        if run_count == observed_runs:
            observed_arrangement_count = arrangement_count

    if observed_arrangement_count is None:
        raise AssertionError("the observed run count must belong to the exact support")
    if sum(mass.arrangement_count for mass in masses) != total_arrangements:
        raise AssertionError("run-count weights must cover every fixed-count arrangement")

    two_sided_arrangement_count = sum(
        mass.arrangement_count
        for mass in masses
        if mass.arrangement_count <= observed_arrangement_count
    )
    return BinaryRunsTestResult(
        sample_size=sample_size,
        zero_count=zero_count,
        one_count=one_count,
        observed_runs=observed_runs,
        total_arrangements=total_arrangements,
        observed_arrangement_count=observed_arrangement_count,
        observed_run_probability=Fraction(observed_arrangement_count, total_arrangements),
        distribution=tuple(masses),
        probability_ordered_two_sided_p_value=Fraction(
            two_sided_arrangement_count,
            total_arrangements,
        ),
    )
```

## Example

```python
def verify_tiny_grouped_arrangements(max_length: int) -> int:
    from collections import Counter
    from itertools import pairwise, product

    grouped: dict[tuple[int, int], list[tuple[int, ...]]] = {}
    for length in range(1, max_length + 1):
        for arrangement in product((0, 1), repeat=length):
            key = (arrangement.count(0), arrangement.count(1))
            grouped.setdefault(key, []).append(arrangement)

    checked = 0
    for arrangements in grouped.values():
        run_weights = Counter(
            1 + sum(left != right for left, right in pairwise(value)) for value in arrangements
        )
        expected_distribution = tuple(sorted(run_weights.items()))
        for arrangement in arrangements:
            result = exact_fixed_count_binary_runs_test(arrangement)
            observed_weight = run_weights[result.observed_runs]
            expected_two_sided = Fraction(
                sum(weight for weight in run_weights.values() if weight <= observed_weight),
                len(arrangements),
            )
            assert (
                result.total_arrangements == len(arrangements)
                and tuple((mass.run_count, mass.arrangement_count) for mass in result.distribution)
                == expected_distribution
                and result.observed_run_probability == Fraction(observed_weight, len(arrangements))
                and result.probability_ordered_two_sided_p_value == expected_two_sided
            )
            checked += 1
    return checked


clustered = exact_fixed_count_binary_runs_test((0, 0, 0, 1, 1, 1))
alternating = exact_fixed_count_binary_runs_test((0, 1, 0, 1, 0, 1))
one_symbol = exact_fixed_count_binary_runs_test((1, 1, 1, 1))
bool_rejected = False
try:
    exact_fixed_count_binary_runs_test((0, True, 1))
except TypeError:
    bool_rejected = True

assert (
    clustered.distribution,
    clustered.observed_runs,
    clustered.probability_ordered_two_sided_p_value,
    alternating.observed_runs,
    alternating.probability_ordered_two_sided_p_value,
    one_symbol.distribution,
    one_symbol.probability_ordered_two_sided_p_value,
    verify_tiny_grouped_arrangements(7),
    bool_rejected,
) == (
    (
        BinaryRunMass(2, 2, Fraction(1, 10)),
        BinaryRunMass(3, 4, Fraction(1, 5)),
        BinaryRunMass(4, 8, Fraction(2, 5)),
        BinaryRunMass(5, 4, Fraction(1, 5)),
        BinaryRunMass(6, 2, Fraction(1, 10)),
    ),
    2,
    Fraction(1, 5),
    6,
    Fraction(1, 5),
    (BinaryRunMass(1, 1, Fraction(1, 1)),),
    Fraction(1, 1),
    254,
    True,
)
```

## Trade-offs and Limitations

For `n` values, validation, support construction, and probability ordering use
`O(n)` iterations and store at most `n` probability masses. Each
`math.comb`, integer multiplication, and `Fraction` reduction operates on
multi-word integers rather than in constant time. The calculation counts as
many as `C(200, 100)` arrangements symbolically; it never materializes them.

The distribution retains the exact arrangement count and probability of every
feasible run count. A one-symbol input has one arrangement, one run, observed
mass one, and p-value one. Reversing a sequence preserves the complete result.
Complementing every symbol swaps the reported zero and one counts while
preserving the run-count distribution and p-value.

The p-value is exact only for the conditional uniform-arrangement null. It
orders run-count point masses, includes all probability ties, and is neither a
doubled smaller tail nor a normal approximation. It does not estimate a
Bernoulli probability, prove independence, locate a change point, report an
effect size or confidence interval, choose a significance threshold, or adjust
multiple tests. Inputs cannot contain other labels, booleans, missing values,
weights, floats, or streaming observations.

## Related Snippets

<!-- catalog:related:start -->
- [Compute Fisher's Exact Test with a Probability-Ordered Two-Sided P-Value](compute-fishers-exact-test-with-a-probability-ordered-two-sided-p-value.md)
- [Compute an Exact Paired Two-Sided Sign Test](compute-an-exact-paired-two-sided-sign-test.md)
- [Find an Exact Single Change Point in a Bounded Integer Sequence](find-an-exact-single-change-point-in-a-bounded-integer-sequence.md)
<!-- catalog:related:end -->
