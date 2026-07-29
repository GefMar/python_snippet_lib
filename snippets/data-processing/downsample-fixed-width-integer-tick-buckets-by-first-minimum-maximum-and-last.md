---
title: "Downsample Fixed-Width Integer Tick Buckets by First, Minimum, Maximum, and Last"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - integrate-regular-rate-samples-across-explicit-time-buckets.md
  - partition-strictly-increasing-integer-timestamps-into-idle-gap-sessions.md
  - select-one-record-per-key-with-an-explicit-ranking-rule.md
---

# Downsample Fixed-Width Integer Tick Buckets by First, Minimum, Maximum, and Last

## Idea and Problem

Reduce ordered integer samples while retaining each fixed-width bucket's boundary observations and value extremes in their original order.

The first and last samples preserve the visible entry and exit of a bucket,
while its minimum and maximum preserve the local value range. Selecting source
indices first makes role collisions explicit: a singleton or one extreme can
fill several roles but appears only once in the result.

## When to Use

Use this algorithm when samples have strictly increasing non-negative integer
ticks, buckets are aligned to tick zero, and a compact overview should retain
more shape than an average or one representative per bucket. It works well for
bounded chart preparation, diagnostic summaries, and review data whose
original sample values must not be synthesized.

Choose the bucket width in the same units as the ticks. Use resampling or an
aggregation policy instead when evenly spaced output, interpolated values, or
statistical summaries are required. Calendar periods need calendar-aware
boundaries rather than integer division.

## Implementation

```python
_MAX_DOWNSAMPLE_SAMPLES = 100_000
_MAX_INT63 = (1 << 63) - 1
_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1


def downsample_integer_tick_buckets(
    samples: tuple[tuple[int, int], ...],
    width: int,
) -> tuple[tuple[int, int], ...]:
    """Return original first, minimum, maximum, and last bucket samples."""
    if type(samples) is not tuple:
        raise TypeError("samples must be an exact tuple")
    if len(samples) > _MAX_DOWNSAMPLE_SAMPLES:
        raise ValueError("sample count exceeds the supported limit")
    if type(width) is not int:
        raise TypeError("width must be an exact non-boolean integer")
    if not 1 <= width <= _MAX_INT63:
        raise ValueError("width is outside the supported range")

    previous_tick = -1
    for index, sample in enumerate(samples):
        if type(sample) is not tuple:
            raise TypeError(f"samples[{index}] must be an exact tuple")
        if len(sample) != 2:
            raise ValueError(f"samples[{index}] must contain exactly two items")

        tick, value = sample
        if type(tick) is not int:
            raise TypeError(f"samples[{index}].tick must be an exact non-boolean integer")
        if not 0 <= tick <= _MAX_INT63:
            raise ValueError(f"samples[{index}].tick is outside the supported range")
        if tick <= previous_tick:
            raise ValueError("sample ticks must be strictly increasing")
        if type(value) is not int:
            raise TypeError(f"samples[{index}].value must be an exact non-boolean integer")
        if not _MIN_INT64 <= value <= _MAX_INT64:
            raise ValueError(f"samples[{index}].value is outside the supported range")
        previous_tick = tick

    if not samples:
        return ()

    current_bucket = samples[0][0] // width
    first_index = 0
    minimum_index = 0
    maximum_index = 0
    last_index = 0
    result: list[tuple[int, int]] = []

    for index in range(1, len(samples)):
        tick, value = samples[index]
        bucket = tick // width
        if bucket != current_bucket:
            for selected_index in sorted({first_index, minimum_index, maximum_index, last_index}):
                result.append(samples[selected_index])
            current_bucket = bucket
            first_index = index
            minimum_index = index
            maximum_index = index
        else:
            if value < samples[minimum_index][1]:
                minimum_index = index
            if value > samples[maximum_index][1]:
                maximum_index = index
        last_index = index

    for selected_index in sorted({first_index, minimum_index, maximum_index, last_index}):
        result.append(samples[selected_index])
    return tuple(result)
```

## Example

```python
source = (
    (0, 5),
    (2, 1),
    (5, 9),
    (9, 1),
    (10, 7),
    (13, 7),
    (19, 7),
    (31, -2),
)
downsampled = downsample_integer_tick_buckets(source, width=10)

try:
    downsample_integer_tick_buckets((), width=0)
except ValueError:
    empty_width_rejected = True
else:
    empty_width_rejected = False

assert downsampled == (
    source[0],
    source[1],
    source[2],
    source[3],
    source[4],
    source[6],
    source[7],
)
assert downsampled[0] is source[0] and downsampled[-1] is source[-1]
assert empty_width_rejected
```

## Trade-offs and Limitations

Validation and selection take `O(n)` time. The function retains constant state
for the current bucket, while the returned tuple and its temporary result list
use output-proportional memory. At most four original samples are emitted per
non-empty bucket, so the output contains at most
`min(n, 4 * non_empty_bucket_count)` samples.

The reduction can be much smaller than the input when buckets are dense, but a
singleton bucket or a bucket with four independently selected roles produces
no size reduction. Earliest occurrences win equal minimum or maximum ties, and
deduplicated role indices are always emitted in original order.

Buckets are the zero-anchored integer ranges determined by `tick // width`;
missing bucket ids produce no output. The function does not interpolate,
average, weight, reorder, or generate samples, and it does not implement
calendar buckets, time-zone conversion, LTTB, streaming state, or duplicate
tick handling.

## Related Snippets

<!-- catalog:related:start -->
- [Integrate Regular Rate Samples Across Explicit Time Buckets](integrate-regular-rate-samples-across-explicit-time-buckets.md)
- [Partition Strictly Increasing Integer Timestamps into Idle-Gap Sessions](partition-strictly-increasing-integer-timestamps-into-idle-gap-sessions.md)
- [Select One Record per Key with an Explicit Ranking Rule](select-one-record-per-key-with-an-explicit-ranking-rule.md)
<!-- catalog:related:end -->
