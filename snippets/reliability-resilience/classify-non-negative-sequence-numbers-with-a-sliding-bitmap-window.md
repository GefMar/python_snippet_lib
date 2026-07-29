---
title: "Classify Non-Negative Sequence Numbers with a Sliding Bitmap Window"
snippet_type: algorithm
use_cases:
  - retry-recovery
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - suppress-stale-keyed-events-with-strictly-increasing-sequence-numbers.md
  - ../networking-protocols/unwrap-one-uint32-serial-around-an-explicit-absolute-reference.md
  - ../testing-tooling/generate-integer-boundary-probes-around-closed-cut-points.md
---

# Classify Non-Negative Sequence Numbers with a Sliding Bitmap Window

## Idea and Problem

Classify each bounded non-negative sequence as accepted, duplicate, or too old while retaining only a fixed numeric window in one integer bitmap.

The highest observed sequence anchors bit zero. Each older sequence within the
window maps to a bit at its distance below that high-water mark. A set bit is a
known duplicate, a clear bit is an unseen value that can still be accepted, and
a value below the represented window has an unknowable history.

## When to Use

Use this reducer when one sequence space is trustworthy, bounded reordering is
allowed, and retaining every accepted value would grow without limit. It fits
an already authenticated stream whose non-negative sequence numbers never wrap
and whose deduplication horizon is naturally expressed as a numeric distance.

Keep the immutable state in the same serialized update path as the action it
guards. Use persistent event identities when duplicates must be recognized
forever, a time-indexed structure when expiry is temporal, or a protocol-aware
serial-number comparison when the sequence space wraps.

## Implementation

```python
from dataclasses import dataclass
from enum import StrEnum

_MAX_INT63 = (1 << 63) - 1
_MAX_WINDOW_WIDTH = 4_096


class SequenceStatus(StrEnum):
    ACCEPTED = "accepted"
    DUPLICATE = "duplicate"
    TOO_OLD = "too_old"


@dataclass(frozen=True, slots=True)
class SequenceWindow:
    width: int
    highest: int | None = None
    bitmap: int = 0

    def __post_init__(self) -> None:
        if type(self.width) is not int:
            raise TypeError("width must be an exact integer")
        if not 1 <= self.width <= _MAX_WINDOW_WIDTH:
            raise ValueError("width is outside the supported range")
        if type(self.bitmap) is not int:
            raise TypeError("bitmap must be an exact integer")

        if self.highest is None:
            if self.bitmap != 0:
                raise ValueError("an empty window must have a zero bitmap")
            return

        if type(self.highest) is not int:
            raise TypeError("highest must be an exact integer or None")
        if not 0 <= self.highest <= _MAX_INT63:
            raise ValueError("highest is outside the supported range")
        if self.bitmap <= 0:
            raise ValueError("a non-empty window must have a positive bitmap")
        if self.bitmap & 1 == 0:
            raise ValueError("bit zero must represent highest")
        maximum_bits = min(self.width, self.highest + 1)
        if self.bitmap.bit_length() > maximum_bits:
            raise ValueError("bitmap contains an impossible sequence position")


@dataclass(frozen=True, slots=True)
class SequenceDecision:
    window: SequenceWindow
    status: SequenceStatus


def classify_sequence(
    window: SequenceWindow,
    sequence: int,
) -> SequenceDecision:
    """Classify one sequence and return the next immutable window."""
    if type(window) is not SequenceWindow:
        raise TypeError("window must be an exact SequenceWindow")
    if type(sequence) is not int:
        raise TypeError("sequence must be an exact integer")
    if not 0 <= sequence <= _MAX_INT63:
        raise ValueError("sequence is outside the supported range")

    if window.highest is None:
        return SequenceDecision(
            SequenceWindow(window.width, sequence, 1),
            SequenceStatus.ACCEPTED,
        )

    if sequence > window.highest:
        distance = sequence - window.highest
        if distance >= window.width:
            next_bitmap = 1
        else:
            retained_bits = window.width - distance
            retained_mask = (1 << retained_bits) - 1
            next_bitmap = ((window.bitmap & retained_mask) << distance) | 1
        return SequenceDecision(
            SequenceWindow(window.width, sequence, next_bitmap),
            SequenceStatus.ACCEPTED,
        )

    distance = window.highest - sequence
    if distance >= window.width:
        return SequenceDecision(window, SequenceStatus.TOO_OLD)

    sequence_bit = 1 << distance
    if window.bitmap & sequence_bit:
        return SequenceDecision(window, SequenceStatus.DUPLICATE)

    return SequenceDecision(
        SequenceWindow(
            window.width,
            window.highest,
            window.bitmap | sequence_bit,
        ),
        SequenceStatus.ACCEPTED,
    )
```

## Example

```python
window = SequenceWindow(width=4)
statuses: list[SequenceStatus] = []
for sequence in (10, 8, 8, 6, 14, 10, 15):
    decision = classify_sequence(window, sequence)
    window = decision.window
    statuses.append(decision.status)

try:
    SequenceWindow(width=8, highest=0, bitmap=0b11)
except ValueError:
    ghost_bit_rejected = True
else:
    ghost_bit_rejected = False

maximum = classify_sequence(SequenceWindow(width=1), _MAX_INT63)

assert (tuple(statuses), window, ghost_bit_rejected, maximum) == (
    (
        SequenceStatus.ACCEPTED,
        SequenceStatus.ACCEPTED,
        SequenceStatus.DUPLICATE,
        SequenceStatus.TOO_OLD,
        SequenceStatus.ACCEPTED,
        SequenceStatus.TOO_OLD,
        SequenceStatus.ACCEPTED,
    ),
    SequenceWindow(width=4, highest=15, bitmap=0b11),
    True,
    SequenceDecision(
        SequenceWindow(width=1, highest=_MAX_INT63, bitmap=1),
        SequenceStatus.ACCEPTED,
    ),
)
```

## Trade-offs and Limitations

Bitmap transitions perform bounded operations on an integer of at most `width`
bits; sequence comparisons and subtraction use fixed 63-bit non-negative
inputs. The bitmap cost therefore scales with its machine-word count, while the
immutable state uses `O(width)` bits. A jump of at least `width` resets directly
to one bit before any shift, so a very large valid sequence does not request a
very large temporary integer.

The window covers numeric positions, not the last `width` arrivals. Advancing
the high-water mark discards older knowledge; a later value below that boundary
is `too_old`, not a proven duplicate. Missing values within the window remain
acceptable, and a large jump intentionally forgets the entire previous bitmap.

This state has no wraparound arithmetic, clocks, expiry, persistence,
synchronization, authentication, or replay-security guarantee. Concurrent
callers still need one atomic owner for reading and replacing the state, and a
protocol with signed or wrapping serials needs its own ordering rules.

## Related Snippets

<!-- catalog:related:start -->
- [Suppress Stale Keyed Events with Strictly Increasing Sequence Numbers](suppress-stale-keyed-events-with-strictly-increasing-sequence-numbers.md)
- [Unwrap One uint32 Serial Around an Explicit Absolute Reference](../networking-protocols/unwrap-one-uint32-serial-around-an-explicit-absolute-reference.md)
- [Generate Integer Boundary Probes Around Closed Cut Points](../testing-tooling/generate-integer-boundary-probes-around-closed-cut-points.md)
<!-- catalog:related:end -->
