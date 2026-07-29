---
title: "Compute an Exact Poisson-Binomial PMF from Bounded Rational Probabilities"
snippet_type: algorithm
use_cases:
  - data-transformation
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - compute-an-exact-paired-two-sided-sign-test.md
  - compute-fishers-exact-test-with-a-probability-ordered-two-sided-p-value.md
  - compute-an-exact-two-sample-permutation-test-for-a-mean-difference.md
---

# Compute an Exact Poisson-Binomial PMF from Bounded Rational Probabilities

## Idea and Problem

Compute the complete distribution of the number of successes across independent Bernoulli trials whose probabilities are exact rational numbers.

The Poisson-binomial distribution allows each trial to have a different
success probability. A descending in-place convolution updates the coefficient
for every possible success count without overwriting a coefficient that is
still needed for the current trial.

Using `Fraction` preserves exact probability mass, including for probabilities
that have no finite binary floating-point representation.

## When to Use

Use this dynamic program for a small collection of independent yes-or-no events
when their individual probabilities are known as exact fractions and every
probability for the total success count is needed.

It is useful as a reference calculation, a deterministic test oracle, or a
small exact risk model. Use a numerical or specialized method when the number
of trials or the rational numerators and denominators make exact arithmetic too
expensive.

## Implementation

```python
from fractions import Fraction

_MAX_TRIALS = 64
_MAX_DENOMINATOR = 1_000_000


def exact_poisson_binomial_pmf(
    probabilities: tuple[tuple[int, int], ...],
) -> tuple[Fraction, ...]:
    """Return exact mass for zero through all admitted Bernoulli successes."""
    if type(probabilities) is not tuple:
        raise TypeError("probabilities must be an exact tuple")
    if len(probabilities) > _MAX_TRIALS:
        raise ValueError("trial count exceeds the supported limit")

    validated_pairs: list[tuple[int, int]] = []
    for probability in probabilities:
        if type(probability) is not tuple or len(probability) != 2:
            raise TypeError(
                "each probability must be an exact numerator-denominator tuple"
            )
        numerator, denominator = probability
        if type(numerator) is not int or type(denominator) is not int:
            raise TypeError("probability components must be exact integers")
        if not 1 <= denominator <= _MAX_DENOMINATOR:
            raise ValueError("probability denominator is outside the supported range")
        if not 0 <= numerator <= denominator:
            raise ValueError("probability numerator must be between zero and denominator")
        validated_pairs.append((numerator, denominator))

    exact_probabilities = tuple(
        Fraction(numerator, denominator)
        for numerator, denominator in validated_pairs
    )
    mass = [Fraction(0) for _ in range(len(exact_probabilities) + 1)]
    mass[0] = Fraction(1)

    admitted_trials = 0
    for probability in exact_probabilities:
        failure_probability = 1 - probability
        for successes in range(admitted_trials + 1, 0, -1):
            mass[successes] = (
                mass[successes] * failure_probability
                + mass[successes - 1] * probability
            )
        mass[0] *= failure_probability
        admitted_trials += 1

    return tuple(mass)
```

## Example

```python
probabilities = ((1, 2), (1, 3), (1, 4))
mass = exact_poisson_binomial_pmf(probabilities)

assert mass == (
    Fraction(1, 4),
    Fraction(11, 24),
    Fraction(1, 4),
    Fraction(1, 24),
)
assert sum(mass) == 1
assert sum(successes * value for successes, value in enumerate(mass)) == Fraction(
    13,
    12,
)

reordered = exact_poisson_binomial_pmf(tuple(reversed(probabilities)))
assert reordered == mass

fixed_outcomes = exact_poisson_binomial_pmf(((0, 1), (1, 1), (1, 1)))
assert fixed_outcomes == (0, 0, 1, 0)

assert exact_poisson_binomial_pmf(()) == (Fraction(1),)
```

## Trade-offs and Limitations

For `n` trials, the dynamic program performs `O(n^2)` exact `Fraction`
operations and stores `O(n)` probability values. Numerators and denominators
can acquire many bits as unrelated rational probabilities are convolved, so
the bit-operation cost may grow substantially even within the 64-trial limit.

The result assumes mutually independent Bernoulli trials and reports only the
probability mass function. The function does not accept approximate floats,
model dependence between events, compute a cumulative or tail probability,
sample outcomes, estimate input probabilities, or perform a hypothesis test.

## Related Snippets

<!-- catalog:related:start -->
- [Compute an Exact Paired Two-Sided Sign Test](compute-an-exact-paired-two-sided-sign-test.md)
- [Compute Fisher's Exact Test with a Probability-Ordered Two-Sided P-Value](compute-fishers-exact-test-with-a-probability-ordered-two-sided-p-value.md)
- [Compute an Exact Two-Sample Permutation Test for a Mean Difference](compute-an-exact-two-sample-permutation-test-for-a-mean-difference.md)
<!-- catalog:related:end -->
