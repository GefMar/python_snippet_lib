---
title: "Compute Linux cgroup v2 CPU Deltas from Two Bounded cpu.stat Snapshots"
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
  - compute-linux-psi-stall-deltas-from-two-bounded-system-snapshots.md
  - compute-a-validated-delta-between-cumulative-histogram-snapshots.md
---

# Compute Linux cgroup v2 CPU Deltas from Two Bounded cpu.stat Snapshots

## Idea and Problem

Parse two recorded Linux cgroup v2 cpu.stat snapshots under one closed profile and compute exact monotonic counter deltas over an explicit observation interval.

The kernel exposes cumulative microsecond and event counters rather than
per-interval rates. Canonical key ordering in the immutable result makes the
comparison independent of line order, while `Fraction` preserves the exact
average number of CPU cores consumed during the caller-supplied interval.

## When to Use

Use this algorithm after a Linux-specific collector has captured complete
`cpu.stat` bytes together with monotonic nanosecond timestamps. It fits small
diagnostic snapshots, deterministic fixtures, and custom telemetry boundaries
that need cgroup-scoped CPU deltas without consulting a file or clock during
calculation.

The accepted keys form a deliberately local current-and-LTS compatibility
profile. Unknown future fields are rejected instead of silently discarded.
Use a maintained cgroup monitor when kernel-version discovery, controller
changes, cgroup lifecycle, collection, or alert policy must be coordinated.

## Implementation

```python
import re
from dataclasses import dataclass
from fractions import Fraction

_MAX_PAYLOAD_BYTES = 1_024
_MAX_LINES = 16
_MAX_COUNTER = (1 << 64) - 1
_MAX_OBSERVED_NS = (1 << 63) - 1
_LINE_PATTERN = re.compile(
    rb"([a-z][a-z0-9_.]{0,63}) (0|[1-9][0-9]{0,19})",
    re.ASCII,
)
_COUNTER_ORDER = (
    "usage_usec",
    "user_usec",
    "system_usec",
    "nice_usec",
    "core_sched.force_idle_usec",
    "nr_periods",
    "nr_throttled",
    "throttled_usec",
    "nr_bursts",
    "burst_usec",
)
_KNOWN_KEYS = frozenset(_COUNTER_ORDER)
_REQUIRED_KEYS = frozenset(("usage_usec", "user_usec", "system_usec"))
_THROTTLE_KEYS = frozenset(("nr_periods", "nr_throttled", "throttled_usec"))
_BURST_KEYS = frozenset(("nr_bursts", "burst_usec"))


class CgroupCpuStatError(ValueError):
    """Raised when a snapshot is outside the closed cpu.stat profile."""


@dataclass(frozen=True, slots=True)
class CgroupCpuStatSnapshot:
    payload: bytes
    observed_ns: int


@dataclass(frozen=True, slots=True)
class CgroupCpuStatDelta:
    elapsed_ns: int
    counter_deltas: tuple[tuple[str, int], ...]
    average_cpu_cores: Fraction
    throttled_period_fraction: Fraction | None


def _parse_payload(payload: object, *, field: str) -> dict[str, int]:
    if type(payload) is not bytes:
        raise TypeError(f"{field}.payload must be exact bytes")
    if not 1 <= len(payload) <= _MAX_PAYLOAD_BYTES:
        raise CgroupCpuStatError(f"{field}.payload length is outside 1..1024")
    if not payload.endswith(b"\n"):
        raise CgroupCpuStatError(f"{field}.payload must end with one LF")

    lines = payload.split(b"\n")
    if lines[-1] != b"" or not 1 <= len(lines) - 1 <= _MAX_LINES:
        raise CgroupCpuStatError(f"{field}.payload has an invalid line count")

    counters: dict[str, int] = {}
    for line_number, line in enumerate(lines[:-1], start=1):
        match = _LINE_PATTERN.fullmatch(line)
        if match is None:
            raise CgroupCpuStatError(
                f"{field}.payload line {line_number} is outside the line grammar"
            )
        key = match.group(1).decode("ascii")
        if key not in _KNOWN_KEYS:
            raise CgroupCpuStatError(f"{field}.payload contains unknown key {key!r}")
        if key in counters:
            raise CgroupCpuStatError(f"{field}.payload contains duplicate key {key!r}")
        value = int(match.group(2))
        if value > _MAX_COUNTER:
            raise CgroupCpuStatError(f"{field}.payload counter {key!r} exceeds uint64")
        counters[key] = value

    keys = counters.keys()
    if not _REQUIRED_KEYS <= keys:
        raise CgroupCpuStatError(f"{field}.payload is missing a required base counter")
    throttle_count = len(_THROTTLE_KEYS & keys)
    if throttle_count not in (0, len(_THROTTLE_KEYS)):
        raise CgroupCpuStatError(
            f"{field}.payload must contain all or none of the throttle counters"
        )
    burst_count = len(_BURST_KEYS & keys)
    if burst_count not in (0, len(_BURST_KEYS)):
        raise CgroupCpuStatError(f"{field}.payload must contain both or neither burst counters")
    if burst_count and not throttle_count:
        raise CgroupCpuStatError(f"{field}.payload burst counters require the throttle counters")
    return counters


def _validated_snapshot(
    value: object,
    *,
    field: str,
) -> tuple[int, dict[str, int]]:
    if type(value) is not CgroupCpuStatSnapshot:
        raise TypeError(f"{field} must be an exact CgroupCpuStatSnapshot")
    if type(value.observed_ns) is not int:
        raise TypeError(f"{field}.observed_ns must be an exact integer")
    if not 0 <= value.observed_ns <= _MAX_OBSERVED_NS:
        raise ValueError(f"{field}.observed_ns is outside the supported range")
    return value.observed_ns, _parse_payload(value.payload, field=field)


def compute_cgroup_cpu_stat_delta(
    earlier: CgroupCpuStatSnapshot,
    later: CgroupCpuStatSnapshot,
) -> CgroupCpuStatDelta:
    """Return canonical counter deltas for two snapshots with identical keys."""
    earlier_ns, earlier_counters = _validated_snapshot(earlier, field="earlier")
    later_ns, later_counters = _validated_snapshot(later, field="later")
    if earlier_counters.keys() != later_counters.keys():
        raise CgroupCpuStatError("snapshots must contain exactly the same keys")

    elapsed_ns = later_ns - earlier_ns
    if elapsed_ns <= 0:
        raise ValueError("later.observed_ns must be greater than earlier.observed_ns")

    deltas: list[tuple[str, int]] = []
    for key in _COUNTER_ORDER:
        if key not in earlier_counters:
            continue
        delta = later_counters[key] - earlier_counters[key]
        if delta < 0:
            raise CgroupCpuStatError(f"counter {key!r} decreased between snapshots")
        deltas.append((key, delta))

    delta_by_key = dict(deltas)
    periods = delta_by_key.get("nr_periods", 0)
    throttled_period_fraction = None
    if periods:
        throttled_period_fraction = Fraction(
            delta_by_key["nr_throttled"],
            periods,
        )

    return CgroupCpuStatDelta(
        elapsed_ns=elapsed_ns,
        counter_deltas=tuple(deltas),
        average_cpu_cores=Fraction(
            delta_by_key["usage_usec"] * 1_000,
            elapsed_ns,
        ),
        throttled_period_fraction=throttled_period_fraction,
    )


```

## Example

```python
earlier = CgroupCpuStatSnapshot(
    payload=(
        b"nr_throttled 10\n"
        b"usage_usec 100000\n"
        b"nr_periods 40\n"
        b"system_usec 30000\n"
        b"user_usec 60000\n"
        b"throttled_usec 50000\n"
        b"nice_usec 10000\n"
        b"nr_bursts 2\n"
        b"burst_usec 7000\n"
        b"core_sched.force_idle_usec 1000\n"
    ),
    observed_ns=1_000_000_000,
)
later = CgroupCpuStatSnapshot(
    payload=(
        b"core_sched.force_idle_usec 21000\n"
        b"burst_usec 87000\n"
        b"nr_bursts 4\n"
        b"nice_usec 110000\n"
        b"throttled_usec 350000\n"
        b"user_usec 1060000\n"
        b"system_usec 830000\n"
        b"nr_periods 60\n"
        b"usage_usec 2500000\n"
        b"nr_throttled 15\n"
    ),
    observed_ns=3_000_000_000,
)

delta = compute_cgroup_cpu_stat_delta(earlier, later)

assert delta == CgroupCpuStatDelta(
    elapsed_ns=2_000_000_000,
    counter_deltas=(
        ("usage_usec", 2_400_000),
        ("user_usec", 1_000_000),
        ("system_usec", 800_000),
        ("nice_usec", 100_000),
        ("core_sched.force_idle_usec", 20_000),
        ("nr_periods", 20),
        ("nr_throttled", 5),
        ("throttled_usec", 300_000),
        ("nr_bursts", 2),
        ("burst_usec", 80_000),
    ),
    average_cpu_cores=Fraction(6, 5),
    throttled_period_fraction=Fraction(1, 4),
)

try:
    compute_cgroup_cpu_stat_delta(
        CgroupCpuStatSnapshot(b"usage_usec 2\nuser_usec 1\nsystem_usec 1\n", 0),
        CgroupCpuStatSnapshot(b"usage_usec 1\nuser_usec 1\nsystem_usec 1\n", 1),
    )
except CgroupCpuStatError:
    reset_rejected = True
else:
    reset_rejected = False

assert reset_rejected
```

## Trade-offs and Limitations

Parsing and comparison take linear time in at most 1,024 bytes and allocate at
most sixteen parsed lines plus the immutable result. Input order does not affect
output order. Integer subtraction is exact, and both derived ratios remain
exact until a caller deliberately converts them.

The unsigned 64-bit counter limit and accepted-key set are local bounds rather
than promises made by every Linux kernel. `throttled_period_fraction` is a ratio
of event-counter deltas, not a duration percentage; it is `None` when the
throttle group is absent or no periods elapsed. The algorithm does not enforce
relationships among usage, user, system, nice, throttling, or burst counters.

A counter decrease is rejected because it can indicate cgroup recreation,
counter reset, or mismatched capture identity. Task migration can change which
work contributes to each snapshot, and a replacement cgroup whose counters
happen to be no smaller cannot be detected from these two payloads. The result
does not infer CPU quota, capacity, saturation, controller availability, or
cause, and it does not read files, choose an observation interval, or
coordinate collection.

## Related Snippets

<!-- catalog:related:start -->
- [Compute a Process CPU Rate from Two Linux procfs Samples](compute-a-process-cpu-rate-from-two-linux-procfs-samples.md)
- [Compute Linux PSI Stall Deltas from Two Bounded System Snapshots](compute-linux-psi-stall-deltas-from-two-bounded-system-snapshots.md)
- [Compute a Validated Delta Between Cumulative Histogram Snapshots](compute-a-validated-delta-between-cumulative-histogram-snapshots.md)
<!-- catalog:related:end -->
