---
title: "Replay a Randomized Test Case from a Generator Checkpoint"
snippet_type: testing-technique
use_cases:
  - testing
tested_python:
  - "3.14"
dependencies: []
related:
  - generate-a-seeded-metric-with-bounded-flapping-runs.md
---

# Replay a Randomized Test Case from a Generator Checkpoint

## Idea and Problem

Replay one randomized test case from the point where it started, without running the earlier draws again.

A seed starts the generator at the beginning of its sequence. When earlier
cases consume draws, capture the current state with `getstate()` before the
case you want to reproduce. Restore it into a separate `Random` instance so
debugging can repeat the case while the original generator continues its run.

## When to Use

Use this when a test owns its standard-library `Random` instance and generates
multiple cases in sequence. Save the state just before drawing the input for a
case. Keep the checkpoint in memory and restore it in the same runtime while
investigating that case.

## Implementation

```python
from random import Random


def generator_from_checkpoint(checkpoint: tuple[object, ...]) -> Random:
    replay = Random(0)
    replay.setstate(checkpoint)
    return replay


def draw_metric_case(rng: Random) -> tuple[int, float, str]:
    sample_count = rng.randrange(1, 32)
    noise = rng.gauss(0.0, 1.0)
    window = rng.choice(("short", "long"))
    return sample_count, noise, window
```

## Example

```python
original = Random(37)
original.gauss(0.0, 1.0)
original.randrange(100)

checkpoint = original.getstate()
case = draw_metric_case(original)
continuation = original.getstate()

replay = generator_from_checkpoint(checkpoint)
replayed_case = draw_metric_case(replay)
assert replay.getstate() == continuation

replay.random()
assert replay.getstate() != continuation
assert Random(37).getstate() != checkpoint
assert (replayed_case, original.getstate()) == (case, continuation)
```

## Trade-offs and Limitations

Treat the checkpoint as opaque state from a trusted `Random` instance. Use the
same generator implementation and case-generation logic, including the order
and parameters of calls. This recipe does not guarantee replay across Python
versions or implementations. Other inputs, clocks and shared generators need
their own controls.

Retaining a checkpoint for every case increases memory use. Keep only the
checkpoint needed to diagnose the current case. A separate instance prevents
replay draws from consuming the original sequence, but its output is
deliberately reproducible. Use a different source for security-sensitive work.

## Related Snippets

<!-- catalog:related:start -->
- [Generate a Seeded Metric with Bounded Flapping Runs](generate-a-seeded-metric-with-bounded-flapping-runs.md)
<!-- catalog:related:end -->
