---
title: "Compute a Bounded RFC 6298 RTO Estimate from Explicit RTT Samples"
snippet_type: algorithm
use_cases:
  - interoperability
  - networking
  - retry-recovery
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - ../observability-operations/measure-and-freeze-elapsed-time-in-a-context.md
  - ../reliability-resilience/poll-with-deterministic-capped-backoff-under-one-monotonic-deadline.md
  - unwrap-one-uint32-serial-around-an-explicit-absolute-reference.md
---

# Compute a Bounded RFC 6298 RTO Estimate from Explicit RTT Samples

## Idea and Problem

Apply the RFC 6298 smoothing equations to a bounded sequence of positive round-trip-time samples without losing intermediate precision.

The first sample initializes the smoothed RTT to `R` and its variation to
`R / 2`. Each later sample updates the variation against the previous smoothed
RTT before updating the smoothed RTT itself. `fractions.Fraction` preserves
those ordered recurrences exactly. The raw estimate is
`SRTT + max(G, 4 * RTTVAR)`; only the final operational value is rounded up to
an integer microsecond and clamped to this profile's one-to-sixty-second range.

## When to Use

Use this estimator kernel when a small protocol implementation or test oracle
already has trustworthy RTT observations expressed as whole microseconds and
needs a reproducible view of the RFC 6298 state. It is useful for checking
another estimator, replaying a fixed trace, or separating numerical estimation
from timer and retransmission policy.

Pass the effective clock granularity explicitly. Keep sample acquisition,
connection state, and retransmission decisions outside the function. The local
60-second ceiling is an RFC-permitted maximum, not a universal choice for every
transport or application.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction

_MAX_RTT_SAMPLE_COUNT = 256
_MAX_RTT_MICROSECONDS = 60_000_000
_MAX_GRANULARITY_MICROSECONDS = 1_000_000
_MIN_RTO_MICROSECONDS = 1_000_000
_MAX_RTO_MICROSECONDS = 60_000_000
_ALPHA = Fraction(1, 8)
_BETA = Fraction(1, 4)
_VARIATION_MULTIPLIER = 4


@dataclass(frozen=True, slots=True)
class RtoEstimate:
    sample_count: int
    srtt_microseconds: Fraction
    rttvar_microseconds: Fraction
    unclamped_rto_microseconds: Fraction
    rto_microseconds: int


def estimate_rfc_6298_rto(
    rtt_samples_microseconds: tuple[int, ...],
    *,
    clock_granularity_microseconds: int,
) -> RtoEstimate:
    """Return exact estimator state and one bounded integer RTO."""
    if type(rtt_samples_microseconds) is not tuple:
        raise TypeError("rtt_samples_microseconds must be an exact tuple")
    if not 1 <= len(rtt_samples_microseconds) <= _MAX_RTT_SAMPLE_COUNT:
        raise ValueError("RTT sample count is outside the supported range")

    for index, sample in enumerate(rtt_samples_microseconds):
        if type(sample) is not int:
            raise TypeError(f"rtt_samples_microseconds[{index}] must be an exact integer")
        if not 1 <= sample <= _MAX_RTT_MICROSECONDS:
            raise ValueError(f"rtt_samples_microseconds[{index}] is outside the supported range")

    if type(clock_granularity_microseconds) is not int:
        raise TypeError("clock_granularity_microseconds must be an exact integer")
    if not 1 <= clock_granularity_microseconds <= _MAX_GRANULARITY_MICROSECONDS:
        raise ValueError("clock_granularity_microseconds is outside the supported range")

    smoothed_rtt = Fraction(rtt_samples_microseconds[0])
    rtt_variation = smoothed_rtt / 2

    for sample in rtt_samples_microseconds[1:]:
        previous_smoothed_rtt = smoothed_rtt
        rtt_variation = (1 - _BETA) * rtt_variation + _BETA * abs(previous_smoothed_rtt - sample)
        smoothed_rtt = (1 - _ALPHA) * previous_smoothed_rtt + _ALPHA * sample

    unclamped_rto = smoothed_rtt + max(
        Fraction(clock_granularity_microseconds),
        _VARIATION_MULTIPLIER * rtt_variation,
    )
    rounded_rto = (
        unclamped_rto.numerator + unclamped_rto.denominator - 1
    ) // unclamped_rto.denominator
    bounded_rto = min(
        max(rounded_rto, _MIN_RTO_MICROSECONDS),
        _MAX_RTO_MICROSECONDS,
    )

    return RtoEstimate(
        sample_count=len(rtt_samples_microseconds),
        srtt_microseconds=smoothed_rtt,
        rttvar_microseconds=rtt_variation,
        unclamped_rto_microseconds=unclamped_rto,
        rto_microseconds=bounded_rto,
    )


```

## Example

```python
estimate = estimate_rfc_6298_rto(
    (1_000_001, 1_000_002),
    clock_granularity_microseconds=1_000,
)
lower_clamped = estimate_rfc_6298_rto(
    (100_000,),
    clock_granularity_microseconds=1_000,
)
upper_clamped = estimate_rfc_6298_rto(
    (60_000_000,),
    clock_granularity_microseconds=1_000,
)

assert (
    estimate,
    lower_clamped.rto_microseconds,
    upper_clamped.rto_microseconds,
) == (
    RtoEstimate(
        sample_count=2,
        srtt_microseconds=Fraction(8_000_009, 8),
        rttvar_microseconds=Fraction(3_000_005, 8),
        unclamped_rto_microseconds=Fraction(20_000_029, 8),
        rto_microseconds=2_500_004,
    ),
    1_000_000,
    60_000_000,
)
```

## Trade-offs and Limitations

Validation and estimation take `O(N)` time for at most 256 samples and use
`O(1)` estimator state beyond the input. Fraction arithmetic is exact but not
constant-time: numerator and denominator bit lengths may grow as samples are
combined. The function rounds upward exactly once, after forming the final raw
estimate, and returns that exact raw value even when the operational integer is
clamped.

This is a stateless estimator kernel, not a complete RFC 6298 retransmission
implementation. It does not maintain an empty initial state, apply Karn's
algorithm, back off after a timeout, restart a timer, choose which RTT samples
are admissible, or interact with a socket. The one-second floor and local
60-second maximum are explicit profile policy. Different protocols, clock
units, or standards may require other bounds and update rules.

## Related Snippets

<!-- catalog:related:start -->
- [Measure and Freeze Elapsed Time in a Context](../observability-operations/measure-and-freeze-elapsed-time-in-a-context.md)
- [Poll with Deterministic Capped Backoff Under One Monotonic Deadline](../reliability-resilience/poll-with-deterministic-capped-backoff-under-one-monotonic-deadline.md)
- [Unwrap One uint32 Serial Around an Explicit Absolute Reference](unwrap-one-uint32-serial-around-an-explicit-absolute-reference.md)
<!-- catalog:related:end -->
