---
title: "Compute Linux PSI Stall Deltas from Two Bounded System Snapshots"
snippet_type: algorithm
use_cases:
  - data-transformation
  - observability
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - compute-a-process-cpu-rate-from-two-linux-procfs-samples.md
  - measure-cache-hit-ratios-from-monotonic-counter-snapshots.md
  - compute-a-validated-delta-between-cumulative-histogram-snapshots.md
---

# Compute Linux PSI Stall Deltas from Two Bounded System Snapshots

## Idea and Problem

Turn two recorded Linux system pressure snapshots into exact stall-counter deltas over an explicit monotonic observation interval.

Each PSI lane carries recent percentage averages plus a cumulative microsecond
counter. Parsing the closed two-line system format and subtracting monotonic
totals separates the measured interval from the kernel-provided rolling
averages. An exact `Fraction` relates each counter delta to the caller's
nanosecond observation interval without first rounding to `float`.

## When to Use

Use this algorithm after a Linux-specific boundary has captured complete
`/proc/pressure/cpu`, `memory`, or `io` bytes together with monotonic
timestamps. It is useful for deterministic offline checks, custom scrape
intervals, or alert inputs that need both the newest kernel averages and exact
cumulative-counter deltas.

Use a maintained system monitor when files must be read, cgroup pressure must
be combined, resets must span host lifetimes, or collection and alerting need
coordination. Keep capture, scheduling, resource discovery, and policy outside
this pure comparison.

## Implementation

```python
import re
from dataclasses import dataclass
from enum import StrEnum
from fractions import Fraction

_MAX_PAYLOAD_BYTES = 512
_MAX_COUNTER = (1 << 64) - 1
_MAX_OBSERVED_NS = (1 << 63) - 1
_PERCENT_TOKEN = rb"((?:[0-9]|[1-9][0-9]|100)\.[0-9]{2})"
_LINE_PATTERN = re.compile(
    rb"(some|full) avg10="
    + _PERCENT_TOKEN
    + rb" avg60="
    + _PERCENT_TOKEN
    + rb" avg300="
    + _PERCENT_TOKEN
    + rb" total=(0|[1-9][0-9]{0,19})",
    re.ASCII,
)


class PressureResource(StrEnum):
    CPU = "cpu"
    MEMORY = "memory"
    IO = "io"


@dataclass(frozen=True, slots=True)
class SystemPressureReading:
    resource: PressureResource
    observed_ns: int
    payload: bytes


@dataclass(frozen=True, slots=True)
class PressureLane:
    avg10_basis_points: int
    avg60_basis_points: int
    avg300_basis_points: int
    total_us: int


@dataclass(frozen=True, slots=True)
class PressureLaneDelta:
    delta_us: int
    latest_avg10_basis_points: int
    latest_avg60_basis_points: int
    latest_avg300_basis_points: int
    elapsed_ratio: Fraction


@dataclass(frozen=True, slots=True)
class SystemPressureDelta:
    resource: PressureResource
    elapsed_ns: int
    some: PressureLaneDelta
    full: PressureLaneDelta | None


def _basis_points(token: bytes) -> int:
    whole, fractional = token.split(b".", 1)
    value = int(whole) * 100 + int(fractional)
    if value > 10_000:
        raise ValueError("a PSI average exceeds 100.00 percent")
    return value


def _parse_lane(line: bytes, *, expected_name: bytes) -> PressureLane:
    match = _LINE_PATTERN.fullmatch(line)
    if match is None or match.group(1) != expected_name:
        raise ValueError("payload does not match the closed PSI line profile")
    total = int(match.group(5))
    if total > _MAX_COUNTER:
        raise ValueError("a PSI total exceeds the unsigned 64-bit range")
    return PressureLane(
        avg10_basis_points=_basis_points(match.group(2)),
        avg60_basis_points=_basis_points(match.group(3)),
        avg300_basis_points=_basis_points(match.group(4)),
        total_us=total,
    )


def _validated_reading(
    value: object,
    *,
    name: str,
) -> tuple[PressureResource, int, PressureLane, PressureLane]:
    if type(value) is not SystemPressureReading:
        raise TypeError(f"{name} must be an exact SystemPressureReading")
    if type(value.resource) is not PressureResource:
        raise TypeError(f"{name}.resource must be an exact PressureResource")
    if type(value.observed_ns) is not int:
        raise TypeError(f"{name}.observed_ns must be an exact integer")
    if not 0 <= value.observed_ns <= _MAX_OBSERVED_NS:
        raise ValueError(f"{name}.observed_ns is outside the supported range")
    if type(value.payload) is not bytes:
        raise TypeError(f"{name}.payload must be exact bytes")
    if not 1 <= len(value.payload) <= _MAX_PAYLOAD_BYTES:
        raise ValueError(f"{name}.payload length is outside the supported range")

    lines = value.payload.split(b"\n")
    if len(lines) != 3 or lines[-1] != b"":
        raise ValueError(f"{name}.payload must contain exactly two final-newline lines")
    some = _parse_lane(lines[0], expected_name=b"some")
    full = _parse_lane(lines[1], expected_name=b"full")

    some_values = (
        some.avg10_basis_points,
        some.avg60_basis_points,
        some.avg300_basis_points,
        some.total_us,
    )
    full_values = (
        full.avg10_basis_points,
        full.avg60_basis_points,
        full.avg300_basis_points,
        full.total_us,
    )
    if value.resource is PressureResource.CPU:
        if any(full_values):
            raise ValueError("system CPU full values must all be zero")
    elif any(
        full_value > some_value
        for full_value, some_value in zip(full_values, some_values, strict=True)
    ):
        raise ValueError("full pressure values must not exceed matching some values")

    return value.resource, value.observed_ns, some, full


def _lane_delta(
    earlier: PressureLane,
    later: PressureLane,
    *,
    elapsed_ns: int,
    name: str,
) -> PressureLaneDelta:
    if later.total_us < earlier.total_us:
        raise ValueError(f"{name} total decreased")
    delta_us = later.total_us - earlier.total_us
    return PressureLaneDelta(
        delta_us=delta_us,
        latest_avg10_basis_points=later.avg10_basis_points,
        latest_avg60_basis_points=later.avg60_basis_points,
        latest_avg300_basis_points=later.avg300_basis_points,
        elapsed_ratio=Fraction(delta_us * 1_000, elapsed_ns),
    )


def compute_system_pressure_delta(
    earlier: SystemPressureReading,
    later: SystemPressureReading,
) -> SystemPressureDelta:
    first_resource, first_ns, first_some, first_full = _validated_reading(
        earlier,
        name="earlier",
    )
    second_resource, second_ns, second_some, second_full = _validated_reading(
        later,
        name="later",
    )
    if first_resource is not second_resource:
        raise ValueError("readings must describe the same pressure resource")
    elapsed_ns = second_ns - first_ns
    if elapsed_ns <= 0:
        raise ValueError("observation time must increase")

    some_delta = _lane_delta(
        first_some,
        second_some,
        elapsed_ns=elapsed_ns,
        name="some",
    )
    full_delta = None
    if first_resource is not PressureResource.CPU:
        full_delta = _lane_delta(
            first_full,
            second_full,
            elapsed_ns=elapsed_ns,
            name="full",
        )
    return SystemPressureDelta(
        resource=first_resource,
        elapsed_ns=elapsed_ns,
        some=some_delta,
        full=full_delta,
    )


```

## Example

```python
earlier = SystemPressureReading(
    resource=PressureResource.MEMORY,
    observed_ns=1_000_000_000,
    payload=(
        b"some avg10=1.25 avg60=2.50 avg300=3.75 total=100000\n"
        b"full avg10=0.25 avg60=0.50 avg300=0.75 total=20000\n"
    ),
)
later = SystemPressureReading(
    resource=PressureResource.MEMORY,
    observed_ns=1_500_000_000,
    payload=(
        b"some avg10=4.00 avg60=3.00 avg300=2.00 total=350000\n"
        b"full avg10=1.00 avg60=0.80 avg300=0.60 total=70000\n"
    ),
)
measured = compute_system_pressure_delta(earlier, later)

cpu = compute_system_pressure_delta(
    SystemPressureReading(
        PressureResource.CPU,
        2_000_000_000,
        b"some avg10=1.00 avg60=1.00 avg300=1.00 total=10000\n"
        b"full avg10=0.00 avg60=0.00 avg300=0.00 total=0\n",
    ),
    SystemPressureReading(
        PressureResource.CPU,
        2_100_000_000,
        b"some avg10=2.00 avg60=1.50 avg300=1.25 total=30000\n"
        b"full avg10=0.00 avg60=0.00 avg300=0.00 total=0\n",
    ),
)

assert measured == SystemPressureDelta(
    resource=PressureResource.MEMORY,
    elapsed_ns=500_000_000,
    some=PressureLaneDelta(250_000, 400, 300, 200, Fraction(1, 2)),
    full=PressureLaneDelta(50_000, 100, 80, 60, Fraction(1, 10)),
)
assert cpu.some.elapsed_ratio == Fraction(1, 5)
assert cpu.full is None
```

## Trade-offs and Limitations

Parsing takes constant work over at most 512 bytes per reading, and comparison
uses constant additional memory. Basis points preserve the two-decimal average
spelling, while `Fraction` preserves the exact relationship between a
microsecond counter delta and the supplied nanosecond interval.

The observation timestamp is captured outside the kernel text read, so
collection latency or skew can make an elapsed ratio slightly exceed one. The
function reports that evidence instead of clamping it. It copies the later
10-, 60-, and 300-second averages but does not recompute them. A counter
decrease is an incomparable reset, not a wrap to guess.

This closed profile accepts only system-level two-line snapshots. System CPU
`full` is undefined and exported as zero, so all four of its fields must be
zero and the result uses `None`. The algorithm does not read procfs, support
cgroup PSI, register pressure triggers, poll, choose thresholds, correlate
resources, or provide a portability abstraction.

## Related Snippets

<!-- catalog:related:start -->
- [Compute a Process CPU Rate from Two Linux procfs Samples](compute-a-process-cpu-rate-from-two-linux-procfs-samples.md)
- [Measure Cache Hit Ratios from Monotonic Counter Snapshots](measure-cache-hit-ratios-from-monotonic-counter-snapshots.md)
- [Compute a Validated Delta Between Cumulative Histogram Snapshots](compute-a-validated-delta-between-cumulative-histogram-snapshots.md)
<!-- catalog:related:end -->
