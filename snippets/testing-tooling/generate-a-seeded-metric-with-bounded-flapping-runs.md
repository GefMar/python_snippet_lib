---
title: "Generate a Seeded Metric with Bounded Flapping Runs"
snippet_type: testing-technique
use_cases:
  - testing
  - observability
tested_python:
  - "3.14"
dependencies: []
related:
  - ../data-processing/sample-stream-items-independently-with-a-fixed-probability.md
  - ../reliability-resilience/hold-a-switch-active-through-a-monotonic-cooldown.md
---

# Generate a Seeded Metric with Bounded Flapping Runs

## Idea and Problem

Produce deterministic test metric values whose flapped state persists for a bounded randomly chosen run.

A start draw occurs only when no run is active. If it succeeds, the same call
emits the first flapped sample and an inclusive integer draw chooses the whole
run length. Later calls consume that run without another start draw. Every
sample exposes its state and the number of flapped ticks left after the emitted
value, making state transitions directly testable.

## When to Use

Use this mutable generator in a single-threaded test that needs reproducible
bursts rather than independent noise. Inject a dedicated `random.Random`
instance with a known seed, or a compatible deterministic test double that
implements `random()` and `randint()`. Keep the generator local to one test so
its draw sequence cannot be changed by unrelated consumers.

This is synthetic input, not telemetry instrumentation or a model of a real
failure process. Prefer a fixed table of samples when an exact scenario is
more important than varied seeded cases, and use an explicit stochastic model
when run lengths or transition probabilities must match observed data.

## Implementation

```python
import math
from dataclasses import dataclass
from enum import StrEnum
from typing import Protocol


_MAX_RUN_LENGTH = 10_000


class RandomSource(Protocol):
    def random(self) -> float: ...

    def randint(self, start: int, stop: int) -> int: ...


class MetricState(StrEnum):
    NORMAL = "normal"
    FLAPPED = "flapped"


@dataclass(frozen=True, slots=True)
class FlappingMetricPolicy:
    normal_value: float
    flapped_value: float
    start_probability: float
    minimum_run_length: int
    maximum_run_length: int


@dataclass(frozen=True, slots=True)
class MetricSample:
    value: float
    state: MetricState
    ticks_left: int


def _finite_float(value: object, *, field: str) -> float:
    if type(value) not in (int, float):
        raise TypeError(f"{field} must be an integer or float")
    try:
        result = float(value)
    except OverflowError as error:
        raise ValueError(f"{field} must fit in a finite float") from error
    if not math.isfinite(result):
        raise ValueError(f"{field} must be finite")
    return result


class FlappingMetricGenerator:
    """Mutable and deliberately not thread-safe."""

    def __init__(self, policy: FlappingMetricPolicy, *, rng: RandomSource) -> None:
        if type(policy) is not FlappingMetricPolicy:
            raise TypeError("policy must be a FlappingMetricPolicy")
        normal_value = _finite_float(policy.normal_value, field="normal_value")
        flapped_value = _finite_float(policy.flapped_value, field="flapped_value")
        probability = _finite_float(
            policy.start_probability,
            field="start_probability",
        )
        if not 0.0 <= probability <= 1.0:
            raise ValueError("start_probability must be between zero and one")
        for field, value in (
            ("minimum_run_length", policy.minimum_run_length),
            ("maximum_run_length", policy.maximum_run_length),
        ):
            if type(value) is not int:
                raise TypeError(f"{field} must be an integer")
        if not (
            1
            <= policy.minimum_run_length
            <= policy.maximum_run_length
            <= _MAX_RUN_LENGTH
        ):
            raise ValueError("run-length limits are outside the supported range")
        if not callable(getattr(rng, "random", None)) or not callable(
            getattr(rng, "randint", None)
        ):
            raise TypeError("rng must provide random() and randint()")

        self._policy = FlappingMetricPolicy(
            normal_value=normal_value,
            flapped_value=flapped_value,
            start_probability=probability,
            minimum_run_length=policy.minimum_run_length,
            maximum_run_length=policy.maximum_run_length,
        )
        self._rng = rng
        self._ticks_left = 0

    @property
    def ticks_left(self) -> int:
        return self._ticks_left

    def _start_draw(self) -> float:
        draw = self._rng.random()
        if type(draw) is not float:
            raise TypeError("rng.random() must return a float")
        if not math.isfinite(draw) or not 0.0 <= draw < 1.0:
            raise ValueError("rng.random() returned a value outside [0, 1)")
        return draw

    def _run_length_draw(self) -> int:
        length = self._rng.randint(
            self._policy.minimum_run_length,
            self._policy.maximum_run_length,
        )
        if type(length) is not int:
            raise TypeError("rng.randint() must return an integer")
        if not (
            self._policy.minimum_run_length
            <= length
            <= self._policy.maximum_run_length
        ):
            raise ValueError("rng.randint() returned a value outside the requested range")
        return length

    def next_sample(self) -> MetricSample:
        if self._ticks_left > 0:
            self._ticks_left -= 1
            return MetricSample(
                self._policy.flapped_value,
                MetricState.FLAPPED,
                self._ticks_left,
            )

        if self._start_draw() < self._policy.start_probability:
            self._ticks_left = self._run_length_draw() - 1
            return MetricSample(
                self._policy.flapped_value,
                MetricState.FLAPPED,
                self._ticks_left,
            )

        return MetricSample(
            self._policy.normal_value,
            MetricState.NORMAL,
            0,
        )
```

## Example

```python
import random


policy = FlappingMetricPolicy(
    normal_value=10.0,
    flapped_value=2.0,
    start_probability=0.4,
    minimum_run_length=2,
    maximum_run_length=3,
)
metric = FlappingMetricGenerator(policy, rng=random.Random(7))

samples = tuple(metric.next_sample() for _ in range(10))

assert tuple((sample.value, sample.state, sample.ticks_left) for sample in samples) == (
    (2.0, MetricState.FLAPPED, 1),
    (2.0, MetricState.FLAPPED, 0),
    (2.0, MetricState.FLAPPED, 1),
    (2.0, MetricState.FLAPPED, 0),
    (2.0, MetricState.FLAPPED, 1),
    (2.0, MetricState.FLAPPED, 0),
    (2.0, MetricState.FLAPPED, 1),
    (2.0, MetricState.FLAPPED, 0),
    (10.0, MetricState.NORMAL, 0),
    (2.0, MetricState.FLAPPED, 1),
)
```

## Trade-offs and Limitations

Seeded output is reproducible only when the seed, call order, generator
implementation, and policy stay fixed. An immediately restarted run can appear
as one longer flapped interval in the values alone; inspect `ticks_left` when a
test needs to distinguish the transition. A probability of one can restart on
every normal-state call, while zero disables all runs.

Each instance is mutable and intentionally not thread-safe. Sharing either the
instance or its random source introduces order-dependent results. The helper
does not sleep, publish metrics, access a clock, use module-global randomness,
or simulate gradual changes and correlated background noise.

## Related Snippets

<!-- catalog:related:start -->
- [Sample Stream Items Independently with a Fixed Probability](../data-processing/sample-stream-items-independently-with-a-fixed-probability.md)
- [Hold a Switch Active Through a Monotonic Cooldown](../reliability-resilience/hold-a-switch-active-through-a-monotonic-cooldown.md)
<!-- catalog:related:end -->
