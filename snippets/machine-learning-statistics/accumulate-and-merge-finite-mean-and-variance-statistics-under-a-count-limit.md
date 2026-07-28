---
title: "Accumulate and Merge Finite Mean and Variance Statistics Under a Count Limit"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - calculate-a-symmetrically-trimmed-mean.md
  - compute-a-wilson-score-interval-for-a-binomial-proportion.md
  - fit-and-apply-a-frozen-pandas-median-z-score-profile.md
---

# Accumulate and Merge Finite Mean and Variance Statistics Under a Count Limit

## Idea and Problem

Accumulate count, mean, and squared deviations without retaining observations, then merge independently accumulated states without replaying their inputs.

Welford's recurrence adds one finite real observation to a frozen state. Chan's
parallel formula combines two such states. Both operations return new values,
enforce one caller-supplied count limit, and reject non-finite intermediate
results instead of publishing unusable statistics.

## When to Use

Use this algorithm for bounded telemetry, partitions, or batches when all
observations have equal weight and only the mean and variance are required.
Independent workers can accumulate local states and a coordinator can merge
them in an explicitly chosen order.

Choose the count limit before accumulation and pass the same limit to every
update and merge. Use an exact or higher-precision representation when binary
floating-point error is unacceptable, and use a statistical package when
weights, missing-value policy, higher moments, or confidence intervals are
part of the required model.

## Implementation

```python
import math
from dataclasses import dataclass
from numbers import Real

_MAX_EXACT_FLOAT_COUNT = 2**53


def _finite_intermediate(value: float, *, field: str) -> float:
    if not math.isfinite(value):
        raise OverflowError(f"{field} is not representable as a finite float")
    return value


@dataclass(frozen=True, slots=True)
class MeanVarianceState:
    count: int = 0
    mean: float = 0.0
    sum_squared_deviations: float = 0.0

    def __post_init__(self) -> None:
        if type(self.count) is not int:
            raise TypeError("count must be an exact integer")
        if not 0 <= self.count <= _MAX_EXACT_FLOAT_COUNT:
            raise ValueError("count is outside the supported range")
        if type(self.mean) is not float:
            raise TypeError("mean must be an exact float")
        if type(self.sum_squared_deviations) is not float:
            raise TypeError("sum_squared_deviations must be an exact float")
        if not math.isfinite(self.mean):
            raise ValueError("mean must be finite")
        if not math.isfinite(self.sum_squared_deviations):
            raise ValueError("sum_squared_deviations must be finite")
        if self.sum_squared_deviations < 0.0:
            raise ValueError("sum_squared_deviations must be non-negative")
        if self.count == 0 and (
            self.mean != 0.0 or self.sum_squared_deviations != 0.0
        ):
            raise ValueError("an empty state must have zero mean and deviation sum")
        if self.count == 1 and self.sum_squared_deviations != 0.0:
            raise ValueError("a one-observation state must have zero deviation sum")

    def population_variance(self) -> float:
        if self.count == 0:
            raise ValueError("population variance requires at least one observation")
        return self.sum_squared_deviations / self.count

    def sample_variance(self) -> float:
        if self.count < 2:
            raise ValueError("sample variance requires at least two observations")
        return self.sum_squared_deviations / (self.count - 1)


def _validated_count_limit(value: object) -> int:
    if type(value) is not int:
        raise TypeError("max_count must be an exact integer")
    if not 1 <= value <= _MAX_EXACT_FLOAT_COUNT:
        raise ValueError(
            f"max_count must be between 1 and {_MAX_EXACT_FLOAT_COUNT}"
        )
    return value


def _validated_state(
    value: object,
    *,
    max_count: int,
    field: str,
) -> MeanVarianceState:
    if type(value) is not MeanVarianceState:
        raise TypeError(f"{field} must be an exact MeanVarianceState")
    if value.count > max_count:
        raise ValueError(f"{field}.count exceeds max_count")
    return value


def _finite_sample(value: object) -> float:
    if isinstance(value, bool) or not isinstance(value, Real):
        raise TypeError("value must be a non-boolean real number")
    try:
        sample = float(value)
    except OverflowError as error:
        raise ValueError("value must be representable as a finite float") from error
    if not math.isfinite(sample):
        raise ValueError("value must be finite")
    return sample


def accumulate_mean_variance(
    state: MeanVarianceState,
    value: Real,
    *,
    max_count: int,
) -> MeanVarianceState:
    limit = _validated_count_limit(max_count)
    current = _validated_state(state, max_count=limit, field="state")
    if current.count == limit:
        raise OverflowError("accumulation would exceed max_count")
    sample = _finite_sample(value)

    next_count = current.count + 1
    delta = _finite_intermediate(sample - current.mean, field="mean delta")
    mean_increment = _finite_intermediate(
        delta / next_count,
        field="mean increment",
    )
    next_mean = _finite_intermediate(
        current.mean + mean_increment,
        field="updated mean",
    )
    residual = _finite_intermediate(
        sample - next_mean,
        field="updated residual",
    )
    deviation_increment = _finite_intermediate(
        delta * residual,
        field="deviation increment",
    )
    next_deviation_sum = _finite_intermediate(
        current.sum_squared_deviations + deviation_increment,
        field="updated deviation sum",
    )
    if next_deviation_sum < 0.0:
        raise ArithmeticError("updated deviation sum lost its non-negative invariant")
    return MeanVarianceState(next_count, next_mean, next_deviation_sum)


def merge_mean_variance(
    left: MeanVarianceState,
    right: MeanVarianceState,
    *,
    max_count: int,
) -> MeanVarianceState:
    limit = _validated_count_limit(max_count)
    left_state = _validated_state(left, max_count=limit, field="left")
    right_state = _validated_state(right, max_count=limit, field="right")
    combined_count = left_state.count + right_state.count
    if combined_count > limit:
        raise OverflowError("merged state would exceed max_count")
    if left_state.count == 0:
        return right_state
    if right_state.count == 0:
        return left_state

    delta = _finite_intermediate(
        right_state.mean - left_state.mean,
        field="mean delta",
    )
    right_fraction = right_state.count / combined_count
    combined_mean = _finite_intermediate(
        left_state.mean + delta * right_fraction,
        field="merged mean",
    )
    delta_squared = _finite_intermediate(delta * delta, field="squared mean delta")
    cross_weight = left_state.count * right_state.count / combined_count
    cross_term = _finite_intermediate(
        delta_squared * cross_weight,
        field="between-group deviation",
    )
    within_group_sum = _finite_intermediate(
        left_state.sum_squared_deviations + right_state.sum_squared_deviations,
        field="within-group deviation sum",
    )
    combined_deviation_sum = _finite_intermediate(
        within_group_sum + cross_term,
        field="merged deviation sum",
    )
    if combined_deviation_sum < 0.0:
        raise ArithmeticError("merged deviation sum lost its non-negative invariant")
    return MeanVarianceState(
        combined_count,
        combined_mean,
        combined_deviation_sum,
    )
```

## Example

```python
limit = 8
empty = MeanVarianceState()
left = empty
right = empty
whole = empty

for sample in (1.0, 2.0, 3.0):
    left = accumulate_mean_variance(left, sample, max_count=limit)
for sample in (4.0, 5.0):
    right = accumulate_mean_variance(right, sample, max_count=limit)
for sample in (1.0, 2.0, 3.0, 4.0, 5.0):
    whole = accumulate_mean_variance(whole, sample, max_count=limit)

merged = merge_mean_variance(left, right, max_count=limit)
left_before_merge = left

single = accumulate_mean_variance(empty, 7, max_count=limit)
try:
    single.sample_variance()
except ValueError:
    one_sample_rejected = True
else:
    one_sample_rejected = False

try:
    accumulate_mean_variance(empty, True, max_count=limit)
except TypeError:
    boolean_rejected = True
else:
    boolean_rejected = False

try:
    accumulate_mean_variance(merged, 6.0, max_count=5)
except OverflowError:
    count_limit_enforced = True
else:
    count_limit_enforced = False

try:
    accumulate_mean_variance(
        MeanVarianceState(1, 1e308, 0.0),
        -1e308,
        max_count=2,
    )
except OverflowError:
    non_finite_intermediate_rejected = True
else:
    non_finite_intermediate_rejected = False

assert (
    merged == whole,
    merged.count,
    math.isclose(merged.mean, 3.0),
    math.isclose(merged.population_variance(), 2.0),
    math.isclose(merged.sample_variance(), 2.5),
    single.population_variance(),
    left == left_before_merge,
    one_sample_rejected,
    boolean_rejected,
    count_limit_enforced,
    non_finite_intermediate_rejected,
) == (True, 5, True, True, True, 0.0, True, True, True, True, True)
```

## Trade-offs and Limitations

Each accumulation takes constant time and memory; merging also takes constant
time and does not inspect original observations. The state stores the second
central-moment numerator, so population variance divides it by count and sample
variance divides it by count minus one. The empty and one-observation cases are
rejected where those denominators do not define the requested statistic.

Floating-point addition is not associative. Different observation orders and
different merge trees can therefore produce slightly different means and
variances even though Welford and Chan avoid the severe cancellation of a
one-pass sum-of-squares formula. The count cap of 2**53 keeps every supported
integer count exactly representable as a float, but it does not make the
statistics exact. Individually finite values can still produce an
unrepresentable delta or squared deviation; the operation then fails without
changing either input state.

The constructor validates structural invariants but cannot prove that a
manually created non-empty state came from real observations. The algorithm
does not retain samples, remove a prior sample, track weights, ignore missing
values, estimate higher moments, or provide a concurrency protocol.

## Related Snippets

<!-- catalog:related:start -->
- [Calculate a Symmetrically Trimmed Mean](calculate-a-symmetrically-trimmed-mean.md)
- [Compute a Wilson Score Interval for a Binomial Proportion](compute-a-wilson-score-interval-for-a-binomial-proportion.md)
- [Fit and Apply a Frozen pandas Median Z-Score Profile](fit-and-apply-a-frozen-pandas-median-z-score-profile.md)
<!-- catalog:related:end -->
