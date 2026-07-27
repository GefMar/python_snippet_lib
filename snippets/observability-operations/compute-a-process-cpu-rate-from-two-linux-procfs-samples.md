---
title: "Compute a Process CPU Rate from Two Linux procfs Samples"
snippet_type: algorithm
use_cases:
  - observability
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - report-partition-offsets-behind-a-fixed-checkpoint.md
  - classify-required-health-stamps-by-freshness.md
  - ../data-processing/measure-time-in-a-state-within-a-half-open-window.md
---

# Compute a Process CPU Rate from Two Linux procfs Samples

## Idea and Problem

Compute a Linux process CPU rate from two explicit procfs stat samples without reading a file, consulting a clock, or maintaining a cache.

The cumulative user and system tick counters become useful only as deltas over
an observed monotonic interval. The process start counter is part of the
identity check, because the operating system can reuse a PID between samples.
The result is measured in CPU-core equivalents: `1.0` means one full core over
the interval, and a multi-threaded process may legitimately exceed `1.0`.

## When to Use

Use this algorithm after a Linux-specific boundary has captured the stat bytes,
a monotonic observation timestamp, and the host's clock-tick frequency. Keeping
capture outside the calculation makes recorded samples deterministic to test
and lets the caller choose its own file, cache, locking, and scheduling policy.

This is not a portable system monitor. Use an operating-system abstraction when
the same code must run outside Linux, when child-process accounting is needed,
or when per-thread attribution and container CPU quotas must be interpreted.

## Implementation

```python
from dataclasses import dataclass


_MAX_STAT_BYTES = 4_096
_MAX_PID = (1 << 31) - 1
_MAX_COUNTER = (1 << 64) - 1
_MAX_OBSERVED_NS = (1 << 63) - 1
_MAX_CLOCK_TICKS_PER_SECOND = 1_000_000


@dataclass(frozen=True, slots=True)
class ProcessCpuSample:
    pid: int
    user_ticks: int
    system_ticks: int
    process_start_ticks: int
    observed_ns: int


@dataclass(frozen=True, slots=True)
class ProcessCpuRate:
    pid: int
    elapsed_ns: int
    user_tick_delta: int
    system_tick_delta: int
    cpu_seconds: float
    core_rate: float


def _bounded_integer(
    value: object,
    *,
    field: str,
    minimum: int,
    maximum: int,
) -> int:
    if type(value) is not int:
        raise TypeError(f"{field} must be an exact integer")
    if not minimum <= value <= maximum:
        raise ValueError(f"{field} is outside the supported range")
    return value


def _decimal_token(token: bytes) -> bool:
    if not token or len(token) > 21:
        return False
    digits = token[1:] if token.startswith(b"-") else token
    return bool(digits) and digits.isdigit()


def _unsigned_field(
    token: bytes,
    *,
    field: str,
    minimum: int = 0,
    maximum: int,
) -> int:
    if not token or len(token) > 20 or not token.isdigit():
        raise ValueError(f"{field} is not a supported unsigned decimal")
    value = int(token)
    if not minimum <= value <= maximum:
        raise ValueError(f"{field} is outside the supported range")
    return value


def parse_linux_process_stat(
    stat_bytes: bytes,
    *,
    observed_ns: int,
) -> ProcessCpuSample:
    if type(stat_bytes) is not bytes:
        raise TypeError("stat_bytes must be exact bytes")
    if not 1 <= len(stat_bytes) <= _MAX_STAT_BYTES:
        raise ValueError("stat_bytes length is outside the supported range")
    try:
        stat_bytes.decode("ascii")
    except UnicodeDecodeError as error:
        raise ValueError("stat_bytes must contain only ASCII") from error
    if b"\x00" in stat_bytes:
        raise ValueError("stat_bytes must not contain NUL bytes")

    timestamp = _bounded_integer(
        observed_ns,
        field="observed_ns",
        minimum=0,
        maximum=_MAX_OBSERVED_NS,
    )
    first_space = stat_bytes.find(b" ")
    if (
        first_space <= 0
        or first_space + 1 >= len(stat_bytes)
        or stat_bytes[first_space + 1 : first_space + 2] != b"("
    ):
        raise ValueError("stat_bytes has an invalid process prefix")

    closing = stat_bytes.rfind(b") ")
    if closing < first_space + 2:
        raise ValueError("stat_bytes has an invalid command field")
    tail = stat_bytes[closing + 2 :].split()
    if len(tail) < 20:
        raise ValueError("stat_bytes does not contain the required fields")
    if len(tail[0]) != 1 or not tail[0].isalpha():
        raise ValueError("stat_bytes has an invalid process state")
    if any(not _decimal_token(token) for token in tail[1:20]):
        raise ValueError("stat_bytes has a malformed numeric field")

    return ProcessCpuSample(
        pid=_unsigned_field(
            stat_bytes[:first_space],
            field="pid",
            minimum=1,
            maximum=_MAX_PID,
        ),
        user_ticks=_unsigned_field(
            tail[11],
            field="user ticks",
            maximum=_MAX_COUNTER,
        ),
        system_ticks=_unsigned_field(
            tail[12],
            field="system ticks",
            maximum=_MAX_COUNTER,
        ),
        process_start_ticks=_unsigned_field(
            tail[19],
            field="process start ticks",
            maximum=_MAX_COUNTER,
        ),
        observed_ns=timestamp,
    )


def _validated_sample(value: object, *, field: str) -> ProcessCpuSample:
    if type(value) is not ProcessCpuSample:
        raise TypeError(f"{field} must be an exact ProcessCpuSample")
    return ProcessCpuSample(
        pid=_bounded_integer(
            value.pid,
            field=f"{field} pid",
            minimum=1,
            maximum=_MAX_PID,
        ),
        user_ticks=_bounded_integer(
            value.user_ticks,
            field=f"{field} user ticks",
            minimum=0,
            maximum=_MAX_COUNTER,
        ),
        system_ticks=_bounded_integer(
            value.system_ticks,
            field=f"{field} system ticks",
            minimum=0,
            maximum=_MAX_COUNTER,
        ),
        process_start_ticks=_bounded_integer(
            value.process_start_ticks,
            field=f"{field} process start ticks",
            minimum=0,
            maximum=_MAX_COUNTER,
        ),
        observed_ns=_bounded_integer(
            value.observed_ns,
            field=f"{field} observed_ns",
            minimum=0,
            maximum=_MAX_OBSERVED_NS,
        ),
    )


def compute_process_cpu_rate(
    earlier: ProcessCpuSample,
    later: ProcessCpuSample,
    *,
    clock_ticks_per_second: int,
) -> ProcessCpuRate:
    first = _validated_sample(earlier, field="earlier")
    second = _validated_sample(later, field="later")
    tick_frequency = _bounded_integer(
        clock_ticks_per_second,
        field="clock_ticks_per_second",
        minimum=1,
        maximum=_MAX_CLOCK_TICKS_PER_SECOND,
    )

    if (
        first.pid != second.pid
        or first.process_start_ticks != second.process_start_ticks
    ):
        raise ValueError("samples do not describe the same process lifetime")
    elapsed_ns = second.observed_ns - first.observed_ns
    if elapsed_ns <= 0:
        raise ValueError("sample observation time must increase")
    if second.user_ticks < first.user_ticks:
        raise ValueError("user tick counter decreased")
    if second.system_ticks < first.system_ticks:
        raise ValueError("system tick counter decreased")

    user_delta = second.user_ticks - first.user_ticks
    system_delta = second.system_ticks - first.system_ticks
    cpu_seconds = (user_delta + system_delta) / tick_frequency
    core_rate = cpu_seconds / (elapsed_ns / 1_000_000_000)
    return ProcessCpuRate(
        pid=first.pid,
        elapsed_ns=elapsed_ns,
        user_tick_delta=user_delta,
        system_tick_delta=system_delta,
        cpu_seconds=cpu_seconds,
        core_rate=core_rate,
    )
```

## Example

```python
def sample_bytes(*, user_ticks: int, system_ticks: int, start_ticks: int) -> bytes:
    fields_4_through_22 = [
        1,
        2,
        3,
        4,
        5,
        6,
        7,
        8,
        9,
        10,
        user_ticks,
        system_ticks,
        13,
        14,
        15,
        16,
        17,
        18,
        start_ticks,
    ]
    tail = " ".join(str(value) for value in fields_4_through_22)
    return f"321 (worker ) pool) R {tail}\n".encode("ascii")


earlier_bytes = sample_bytes(user_ticks=100, system_ticks=50, start_ticks=900)
later_bytes = sample_bytes(user_ticks=300, system_ticks=150, start_ticks=900)
earlier = parse_linux_process_stat(earlier_bytes, observed_ns=1_000_000_000)
later = parse_linux_process_stat(later_bytes, observed_ns=3_000_000_000)
rate = compute_process_cpu_rate(
    earlier,
    later,
    clock_ticks_per_second=100,
)

try:
    compute_process_cpu_rate(
        earlier,
        ProcessCpuSample(321, 300, 150, 901, 3_000_000_000),
        clock_ticks_per_second=100,
    )
except ValueError:
    reused_pid_rejected = True
else:
    reused_pid_rejected = False

assert (rate, reused_pid_rejected) == (
    ProcessCpuRate(
        pid=321,
        elapsed_ns=2_000_000_000,
        user_tick_delta=200,
        system_tick_delta=100,
        cpu_seconds=3.0,
        core_rate=1.5,
    ),
    True,
)
```

## Trade-offs and Limitations

Parsing and rate calculation use constant memory and linear work in at most
4 KiB of input. The parser deliberately accepts only ASCII and the stable field
prefix through the process start counter. It ignores the command text after
locating its final closing delimiter, so spaces and right parentheses inside
that field do not shift the numeric positions.

The calculation does not read procfs, discover the system tick frequency,
choose a sampling interval, persist state, lock a cache, normalize by available
CPUs, or account for container quotas. Counter rollback and a changed process
start value are rejected rather than interpreted. The caller must capture both
samples from the same Linux host and supply timestamps from one monotonic clock.

## Related Snippets

<!-- catalog:related:start -->
- [Report Partition Offsets Behind a Fixed Checkpoint](report-partition-offsets-behind-a-fixed-checkpoint.md)
- [Classify Required Health Stamps by Freshness](classify-required-health-stamps-by-freshness.md)
- [Measure Time in a State Within a Half-Open Window](../data-processing/measure-time-in-a-state-within-a-half-open-window.md)
<!-- catalog:related:end -->
