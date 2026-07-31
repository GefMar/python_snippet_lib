---
title: "Aggregate Bounded Sampled Stacks into Deterministic Collapsed Counts"
snippet_type: algorithm
use_cases:
  - data-transformation
  - observability
  - serialization
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - project-a-bounded-asyncio-call-graph-without-retaining-live-frames.md
  - compute-inclusive-and-exclusive-durations-for-a-bounded-nested-span-forest.md
  - count-values-in-fixed-upper-bound-bins.md
---

# Aggregate Bounded Sampled Stacks into Deterministic Collapsed Counts

## Idea and Problem

Aggregate bounded root-to-leaf stack samples into deterministic collapsed text without conflating a complete path with one of its prefixes.

Each input weight is a positive integer sample multiplicity. Identical frame
tuples have their weights added, while `("worker",)` and
`("worker", "parse")` remain separate paths. Sorting the complete tuples makes
record and text order independent of sample arrival order.

The closed frame-name alphabet needs no escaping: semicolons separate frames,
one ASCII space separates a path from its decimal count, and one line feed
terminates every record, including the last.

## When to Use

Use this algorithm after a profiler or diagnostic collector has already
symbolized a bounded snapshot into complete root-to-leaf frame tuples. It fits
reproducible fixtures, small profile diffs, and adapters that need one stable
collapsed-count artifact rather than individual observations.

Treat a weight only as the number of represented samples. Choose and document
the sampling clock, interval, target threads, and frame naming upstream. Frame
names must already use the strict visible-ASCII profile; normalize richer
symbols before calling this function or choose a format with explicit escaping.

## Implementation

```python
from dataclasses import dataclass

_MAX_SAMPLE_COUNT = 65_536
_MAX_FRAMES_PER_SAMPLE = 128
_MAX_TOTAL_FRAMES = 262_144
_MAX_FRAME_NAME_BYTES = 128
_MAX_TOTAL_FRAME_NAME_BYTES = 1_048_576
_MAX_TOTAL_WEIGHT = 2**63 - 1
_MAX_OUTPUT_BYTES = 3_145_728


@dataclass(frozen=True, slots=True)
class StackSample:
    frames: tuple[str, ...]
    weight: int


@dataclass(frozen=True, slots=True)
class CollapsedStackCount:
    frames: tuple[str, ...]
    count: int


@dataclass(frozen=True, slots=True)
class CollapsedStackProfile:
    records: tuple[CollapsedStackCount, ...]
    text: str


def aggregate_sampled_stacks(
    samples: tuple[StackSample, ...],
) -> CollapsedStackProfile:
    """Aggregate one bounded sampled-stack snapshot under a closed text profile."""
    if type(samples) is not tuple:
        raise TypeError("samples must be an exact tuple")
    if not 1 <= len(samples) <= _MAX_SAMPLE_COUNT:
        raise ValueError("sample count is outside the supported range")

    counts: dict[tuple[str, ...], int] = {}
    total_frames = 0
    total_name_bytes = 0
    total_weight = 0

    for sample_index, sample in enumerate(samples):
        if type(sample) is not StackSample:
            raise TypeError(f"samples[{sample_index}] must be an exact StackSample")

        frames = sample.frames
        if type(frames) is not tuple:
            raise TypeError(f"samples[{sample_index}].frames must be an exact tuple")
        if not 1 <= len(frames) <= _MAX_FRAMES_PER_SAMPLE:
            raise ValueError(f"samples[{sample_index}].frames has an unsupported frame count")

        weight = sample.weight
        if type(weight) is not int:
            raise TypeError(f"samples[{sample_index}].weight must be an exact integer")
        if weight <= 0:
            raise ValueError(f"samples[{sample_index}].weight must be positive")

        total_frames += len(frames)
        if total_frames > _MAX_TOTAL_FRAMES:
            raise ValueError("total frame count exceeds the supported limit")

        for frame_index, frame in enumerate(frames):
            if type(frame) is not str:
                raise TypeError(f"samples[{sample_index}].frames[{frame_index}] must be exact text")
            if not 1 <= len(frame) <= _MAX_FRAME_NAME_BYTES:
                raise ValueError(
                    f"samples[{sample_index}].frames[{frame_index}] has an unsupported length"
                )
            if any(not 0x21 <= ord(character) <= 0x7E or character == ";" for character in frame):
                raise ValueError(
                    f"samples[{sample_index}].frames[{frame_index}] is outside the "
                    "closed ASCII profile"
                )

            total_name_bytes += len(frame)
            if total_name_bytes > _MAX_TOTAL_FRAME_NAME_BYTES:
                raise ValueError("total frame-name bytes exceed the supported limit")

        total_weight += weight
        if total_weight > _MAX_TOTAL_WEIGHT:
            raise ValueError("total sample weight exceeds the supported limit")
        counts[frames] = counts.get(frames, 0) + weight

    records = tuple(CollapsedStackCount(frames, counts[frames]) for frames in sorted(counts))

    lines: list[str] = []
    output_bytes = 0
    for record in records:
        line = f"{';'.join(record.frames)} {record.count}\n"
        output_bytes += len(line)
        if output_bytes > _MAX_OUTPUT_BYTES:
            raise ValueError("collapsed text exceeds the supported byte limit")
        lines.append(line)

    return CollapsedStackProfile(records, "".join(lines))
```

## Example

```python

samples = (
    StackSample(("service", "parse"), 2),
    StackSample(("worker",), 4),
    StackSample(("service", "parse", "decode"), 1),
    StackSample(("service", "parse"), 3),
)
expected = CollapsedStackProfile(
    records=(
        CollapsedStackCount(("service", "parse"), 5),
        CollapsedStackCount(("service", "parse", "decode"), 1),
        CollapsedStackCount(("worker",), 4),
    ),
    text="service;parse 5\nservice;parse;decode 1\nworker 4\n",
)


def counter_oracle(
    candidate_samples: tuple[StackSample, ...],
) -> CollapsedStackProfile:
    from collections import Counter

    totals: Counter[tuple[str, ...]] = Counter()
    for sample in candidate_samples:
        totals[sample.frames] += sample.weight
    records = tuple(CollapsedStackCount(frames, totals[frames]) for frames in sorted(totals))
    text = "".join(f"{';'.join(record.frames)} {record.count}\n" for record in records)
    return CollapsedStackProfile(records, text)


def verify_permutations() -> int:
    from itertools import permutations

    checked = 0
    for permutation in permutations(samples):
        assert aggregate_sampled_stacks(permutation) == expected
        checked += 1
    return checked


def rejected(candidate: object) -> bool:
    try:
        aggregate_sampled_stacks(candidate)  # type: ignore[arg-type]
    except (TypeError, ValueError):
        return True
    return False


profile = aggregate_sampled_stacks(samples)
checked_permutations = verify_permutations()
unsplit = aggregate_sampled_stacks(
    (
        StackSample(("service", "parse"), 5),
        StackSample(("service", "parse", "decode"), 1),
        StackSample(("worker",), 4),
    )
)

one_frame = StackSample(("x",), 1)
many_frames = StackSample(("x",) * _MAX_FRAMES_PER_SAMPLE, 1)
long_names = StackSample(("x" * _MAX_FRAME_NAME_BYTES,) * 128, 1)
invalid_inputs = (
    [],
    (),
    (StackSample([], 1),),
    (StackSample((), 1),),
    (StackSample(("x",), True),),
    (StackSample(("x",), 0),),
    (StackSample((1,), 1),),
    (StackSample(("",), 1),),
    (StackSample(("bad frame",), 1),),
    (StackSample(("bad;frame",), 1),),
    (StackSample(("\x7f",), 1),),
    (StackSample(("caf\N{LATIN SMALL LETTER E WITH ACUTE}",), 1),),
    (StackSample(("x" * (_MAX_FRAME_NAME_BYTES + 1),), 1),),
    (one_frame,) * (_MAX_SAMPLE_COUNT + 1),
    (many_frames,) * (_MAX_TOTAL_FRAMES // _MAX_FRAMES_PER_SAMPLE + 1),
    (long_names,) * (_MAX_TOTAL_FRAME_NAME_BYTES // (128 * _MAX_FRAME_NAME_BYTES) + 1),
    (
        StackSample(("x",), _MAX_TOTAL_WEIGHT),
        StackSample(("y",), 1),
    ),
)

assert profile == expected
assert profile == counter_oracle(samples)
assert checked_permutations == 24
assert profile.text.endswith("\n")
assert unsplit == profile
assert all(rejected(candidate) for candidate in invalid_inputs)
```

## Trade-offs and Limitations

Validation and aggregation take linear time in the supplied frame occurrences;
sorting the distinct paths adds `O(u log u)` tuple comparisons for `u` unique
paths. The function retains both the aggregate dictionary and the complete
rendered text, so it is intentionally limited to a 3 MiB output.

Sample counts describe how often collection observed each complete path. They
are not exact CPU duration, wall-clock duration, call count, or causal evidence,
and sampling frequency and bias remain properties of the upstream collector.
The result also discards timestamps, thread identity, sample order, and any
metadata not encoded in the frame tuple.

This deliberately narrow format rejects spaces, semicolons, control characters,
non-ASCII names, and all escaping conventions. Use a profiler's native format
when rich symbols, module paths, inline frames, per-thread views, timestamps, or
interoperability with a different collapsed-stack dialect are required.

## Related Snippets

<!-- catalog:related:start -->
- [Project a Bounded asyncio Call Graph Without Retaining Live Frames](project-a-bounded-asyncio-call-graph-without-retaining-live-frames.md)
- [Compute Inclusive and Exclusive Durations for a Bounded Nested Span Forest](compute-inclusive-and-exclusive-durations-for-a-bounded-nested-span-forest.md)
- [Count Values in Fixed Upper-Bound Bins](count-values-in-fixed-upper-bound-bins.md)
<!-- catalog:related:end -->
