---
title: "Build Exact Equal-Width Binary Calibration Bins and Expected Calibration Error"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - compute-an-exact-binary-brier-score-from-integer-probability-ticks.md
  - ../observability-operations/count-values-in-fixed-upper-bound-bins.md
  - measure-drift-between-two-fixed-bin-count-distributions-with-psi.md
---

# Build Exact Equal-Width Binary Calibration Bins and Expected Calibration Error

## Idea and Problem

Group quantized binary probability forecasts into equal-width bins and compare each bin's exact mean forecast with its observed positive rate.

Bin `i` covers `[i / bin_count, (i + 1) / bin_count)`, except that the
final bin also contains probability one. Integer ticks can therefore be placed
without converting them to floats. Weighting each non-empty bin's absolute
forecast-outcome gap by its observation share produces an exact expected
calibration error (ECE).

## When to Use

Use this algorithm for a bounded binary evaluation batch whose probability
forecasts already use one declared integer scale. The immutable bin summaries
are useful for deterministic reports, regression checks, and calibration
diagnostics when equal-width boundaries have been chosen in advance.

Keep the scale, bin count, and evaluation population fixed when comparing
results. Use a proper scoring rule such as Brier score alongside ECE because
binning discards detail. Choose a statistics or model-evaluation library when
observations need weights, confidence intervals, adaptive bins, multiclass
probabilities, or a fitted recalibration method.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction

_MAX_OBSERVATIONS = 100_000
_MAX_SCALE = 10**9
_MAX_BINS = 256


@dataclass(frozen=True, slots=True)
class BinaryCalibrationBin:
    index: int
    lower_bound: Fraction
    upper_bound: Fraction
    upper_inclusive: bool
    count: int
    positives: int
    mean_forecast: Fraction | None
    observed_rate: Fraction | None
    absolute_gap: Fraction | None


@dataclass(frozen=True, slots=True)
class BinaryCalibrationSummary:
    scale: int
    observation_count: int
    bin_count: int
    bins: tuple[BinaryCalibrationBin, ...]
    expected_calibration_error: Fraction


def build_binary_calibration_bins(
    observations: tuple[tuple[int, bool], ...],
    *,
    scale: int,
    bin_count: int,
) -> BinaryCalibrationSummary:
    """Return exact equal-width binary calibration bins and ECE."""
    if type(observations) is not tuple:
        raise TypeError("observations must be an exact tuple")
    if not 1 <= len(observations) <= _MAX_OBSERVATIONS:
        raise ValueError("observation count is outside the supported range")
    if type(scale) is not int:
        raise TypeError("scale must be an exact integer")
    if not 1 <= scale <= _MAX_SCALE:
        raise ValueError("scale is outside the supported range")
    if type(bin_count) is not int:
        raise TypeError("bin_count must be an exact integer")
    if not 1 <= bin_count <= _MAX_BINS:
        raise ValueError("bin_count is outside the supported range")

    for index, observation in enumerate(observations):
        if type(observation) is not tuple:
            raise TypeError(f"observations[{index}] must be an exact tuple")
        if len(observation) != 2:
            raise ValueError(f"observations[{index}] must contain exactly two items")
        tick, outcome = observation
        if type(tick) is not int:
            raise TypeError(f"observations[{index}].tick must be an exact integer")
        if not 0 <= tick <= scale:
            raise ValueError(f"observations[{index}].tick must be between zero and scale")
        if type(outcome) is not bool:
            raise TypeError(f"observations[{index}].outcome must be an exact boolean")

    counts = [0] * bin_count
    positive_counts = [0] * bin_count
    tick_sums = [0] * bin_count
    for tick, outcome in observations:
        index = min(tick * bin_count // scale, bin_count - 1)
        counts[index] += 1
        positive_counts[index] += int(outcome)
        tick_sums[index] += tick

    bins: list[BinaryCalibrationBin] = []
    total_gap_numerator = 0
    for index, (count, positives, tick_sum) in enumerate(
        zip(counts, positive_counts, tick_sums, strict=True)
    ):
        mean_forecast: Fraction | None = None
        observed_rate: Fraction | None = None
        absolute_gap: Fraction | None = None
        if count:
            mean_forecast = Fraction(tick_sum, count * scale)
            observed_rate = Fraction(positives, count)
            absolute_gap = abs(mean_forecast - observed_rate)
            total_gap_numerator += abs(tick_sum - positives * scale)

        bins.append(
            BinaryCalibrationBin(
                index=index,
                lower_bound=Fraction(index, bin_count),
                upper_bound=Fraction(index + 1, bin_count),
                upper_inclusive=index == bin_count - 1,
                count=count,
                positives=positives,
                mean_forecast=mean_forecast,
                observed_rate=observed_rate,
                absolute_gap=absolute_gap,
            )
        )

    return BinaryCalibrationSummary(
        scale=scale,
        observation_count=len(observations),
        bin_count=bin_count,
        bins=tuple(bins),
        expected_calibration_error=Fraction(
            total_gap_numerator,
            len(observations) * scale,
        ),
    )
```

## Example

```python
def reference_calibration_bins(
    observations: tuple[tuple[int, bool], ...],
    *,
    scale: int,
    bin_count: int,
) -> BinaryCalibrationSummary:
    grouped: list[list[tuple[int, bool]]] = [[] for _ in range(bin_count)]
    for tick, outcome in observations:
        probability = Fraction(tick, scale)
        for index in range(bin_count):
            lower = Fraction(index, bin_count)
            upper = Fraction(index + 1, bin_count)
            if (lower <= probability < upper) or (index == bin_count - 1 and probability == upper):
                grouped[index].append((tick, outcome))
                break
        else:
            raise AssertionError("a valid probability must belong to one bin")

    bins: list[BinaryCalibrationBin] = []
    weighted_gap = Fraction()
    for index, group in enumerate(grouped):
        count = len(group)
        positives = sum(outcome for _, outcome in group)
        if count:
            mean_forecast = Fraction(sum(tick for tick, _ in group), count * scale)
            observed_rate = Fraction(positives, count)
            absolute_gap = abs(mean_forecast - observed_rate)
            weighted_gap += Fraction(count, len(observations)) * absolute_gap
        else:
            mean_forecast = None
            observed_rate = None
            absolute_gap = None
        bins.append(
            BinaryCalibrationBin(
                index=index,
                lower_bound=Fraction(index, bin_count),
                upper_bound=Fraction(index + 1, bin_count),
                upper_inclusive=index == bin_count - 1,
                count=count,
                positives=positives,
                mean_forecast=mean_forecast,
                observed_rate=observed_rate,
                absolute_gap=absolute_gap,
            )
        )

    return BinaryCalibrationSummary(
        scale=scale,
        observation_count=len(observations),
        bin_count=bin_count,
        bins=tuple(bins),
        expected_calibration_error=weighted_gap,
    )


def cases_of_length(
    atoms: tuple[tuple[int, bool], ...],
    length: int,
):
    if length == 0:
        yield ()
        return
    for prefix in cases_of_length(atoms, length - 1):
        for atom in atoms:
            yield (*prefix, atom)


for small_scale in range(1, 5):
    atoms = tuple((tick, outcome) for tick in range(small_scale + 1) for outcome in (False, True))
    for length in range(1, 4):
        for case in cases_of_length(atoms, length):
            for small_bin_count in range(1, 6):
                assert build_binary_calibration_bins(
                    case,
                    scale=small_scale,
                    bin_count=small_bin_count,
                ) == reference_calibration_bins(
                    case,
                    scale=small_scale,
                    bin_count=small_bin_count,
                )

summary = build_binary_calibration_bins(
    ((0, False), (1, False), (2, True), (3, True), (5, True)),
    scale=5,
    bin_count=3,
)
maximum = build_binary_calibration_bins(
    tuple((0, False) if index % 2 == 0 else (10**9, True) for index in range(100_000)),
    scale=10**9,
    bin_count=256,
)

rendered = tuple(
    (
        item.lower_bound,
        item.upper_bound,
        item.upper_inclusive,
        item.count,
        item.positives,
        item.mean_forecast,
        item.observed_rate,
        item.absolute_gap,
    )
    for item in summary.bins
)
assert (
    summary.expected_calibration_error,
    rendered,
    maximum.observation_count,
    maximum.bins[0].count,
    maximum.bins[-1].count,
    all(item.count == 0 and item.mean_forecast is None for item in maximum.bins[1:-1]),
    maximum.expected_calibration_error,
) == (
    Fraction(6, 25),
    (
        (Fraction(0), Fraction(1, 3), False, 2, 0, Fraction(1, 10), Fraction(0), Fraction(1, 10)),
        (Fraction(1, 3), Fraction(2, 3), False, 2, 2, Fraction(1, 2), Fraction(1), Fraction(1, 2)),
        (Fraction(2, 3), Fraction(1), True, 1, 1, Fraction(1), Fraction(1), Fraction(0)),
    ),
    100_000,
    50_000,
    50_000,
    True,
    Fraction(0),
)
```

## Trade-offs and Limitations

Validation and aggregation take `O(n + b)` time for `n` observations and `b`
bins. The three mutable integer arrays and immutable result use `O(b)` memory.
All reported rates, gaps, boundaries, and ECE values are exact `Fraction`
objects; no floating-point conversions occur.

Equal-width bins can be empty when the tick scale is coarse and can contain
very different sample counts. ECE changes with the bin count and population,
hides the direction of each gap through absolute values, and does not quantify
sampling uncertainty. Exact arithmetic does not make a sparse calibration
estimate statistically reliable. This implementation accepts only unweighted
binary outcomes on one shared scale and does not fit a calibration model,
select bins from the observations, render a reliability diagram, or choose a
decision threshold.

## Related Snippets

<!-- catalog:related:start -->
- [Compute an Exact Binary Brier Score from Integer Probability Ticks](compute-an-exact-binary-brier-score-from-integer-probability-ticks.md)
- [Count Values in Fixed Upper-Bound Bins](../observability-operations/count-values-in-fixed-upper-bound-bins.md)
- [Measure Drift Between Two Fixed-Bin Count Distributions with PSI](measure-drift-between-two-fixed-bin-count-distributions-with-psi.md)
<!-- catalog:related:end -->
