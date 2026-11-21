---
title: "Measure Time Above a Threshold Between Linear Samples"
snippet_type: algorithm
use_cases:
  - data-transformation
  - observability
tested_python:
  - "3.14"
dependencies: []
related:
  - interpolate-increasing-integer-time-series-points-exactly.md
  - measure-time-in-a-state-within-a-half-open-window.md
  - summarize-an-irregular-integer-gauge-with-expiry.md
---

# Measure Time Above a Threshold Between Linear Samples

## Idea and Problem

Compute the exact time an interpolated signal spends strictly above a threshold between its first and last samples.

With linear interpolation, a segment contributes its entire duration when both
endpoints exceed the threshold and zero when neither exceeds it. Otherwise,
the positive endpoint's distance above the threshold, divided by the total
change in value, gives the fraction of the segment that qualifies. An isolated
point on the threshold has zero duration. A plateau on the threshold also
contributes zero.

## When to Use

Use this for irregularly spaced measurements on an integer clock when a straight
line between samples is a defensible model. Supply a finite iterable of
`(tick, value)` pairs in strictly increasing tick order, with values and the
threshold in the same integer units. The result is a `Fraction` in tick units.

## Implementation

```python
from collections.abc import Iterable
from fractions import Fraction


def time_above_threshold(samples: Iterable[tuple[int, int]], threshold: int) -> Fraction:
    """Measure the portion of the sampled span strictly above threshold."""
    if type(threshold) is not int:
        raise TypeError("threshold must be an integer")

    duration = Fraction(0)
    previous_tick: int | None = None
    previous_offset = 0
    for tick, value in samples:
        if type(tick) is not int or type(value) is not int:
            raise TypeError("sample ticks and values must be integers")
        offset = value - threshold

        if previous_tick is not None:
            width = tick - previous_tick
            if width <= 0:
                raise ValueError("sample ticks must strictly increase")
            if previous_offset > 0 and offset > 0:
                duration += width
            elif previous_offset > 0 or offset > 0:
                duration += Fraction(
                    width * max(previous_offset, offset), abs(offset - previous_offset)
                )

        previous_tick, previous_offset = tick, offset

    return duration
```

## Example

```python
# At threshold 10, the rising segment contributes 10/3 seconds and the
# falling segment contributes 28/5 seconds. The remaining segments add zero.
readings = ((0, 8), (5, 14), (12, 9), (14, 10), (17, 10))

assert (
    time_above_threshold(iter(readings), threshold=10),
    time_above_threshold(((0, 10), (3, 13), (7, 10)), threshold=10),
    time_above_threshold(((0, 11), (4, 11)), threshold=10),
    time_above_threshold(((5, 14),), threshold=10),
) == (Fraction(134, 15), Fraction(7), Fraction(4), Fraction(0))
```

## Trade-offs and Limitations

The calculation takes one pass. It keeps the last endpoint and a running
rational total, whose numerator and denominator can grow across many crossings.
Empty input and a single sample have zero duration. The domain ends at the
last observed tick, so reports requiring more time need an explicit model for
that coverage.

Sparse observations can hide real excursions. Split the input at data gaps
when interpolation across a gap is unjustified. Use stepwise state durations
when each recorded value stays in effect until the next change.

## Related Snippets

<!-- catalog:related:start -->
- [Interpolate Increasing Integer Time-Series Points Exactly](interpolate-increasing-integer-time-series-points-exactly.md)
- [Measure Time in a State Within a Half-Open Window](measure-time-in-a-state-within-a-half-open-window.md)
- [Summarize an Irregular Integer Gauge with Expiry](summarize-an-irregular-integer-gauge-with-expiry.md)
<!-- catalog:related:end -->
