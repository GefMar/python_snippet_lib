---
title: "Unwrap One uint32 Serial Around an Explicit Absolute Reference"
snippet_type: algorithm
use_cases:
  - interoperability
  - networking
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../reliability-resilience/suppress-stale-keyed-events-with-strictly-increasing-sequence-numbers.md
  - read-and-write-size-capped-varint-frames.md
  - ../configuration-serialization/encode-and-decode-one-exact-size-big-endian-binary-record-with-struct.md
---

# Unwrap One uint32 Serial Around an Explicit Absolute Reference

## Idea and Problem

Map one wrapping 32-bit serial to the unique congruent integer nearest an explicit absolute reference, while exposing the exact half-range ambiguity.

A wire serial identifies a residue modulo `2**32`, not one absolute epoch. The
reference supplies the missing epoch information. Reducing the forward modular
distance and interpreting values above half the modulus as negative selects the
nearest representative without guessing across the one distance at which two
representatives are equally near.

## When to Use

Use this calculation when a protocol exposes an unsigned 32-bit serial but the
caller already has a trustworthy absolute reference less than half the serial
space from the intended value. It is useful at a narrow wire-to-domain boundary
before ordinary integer comparisons or storage.

Keep history and acceptance policy outside this function. A stateful receiver
must still decide how to handle reordering, replay, loss, resets, or a reference
that may be more than half a serial space away. Apply the rules of the actual
protocol rather than assuming that nearest-value unwrapping is sufficient.

## Implementation

```python
_UINT32_MODULUS = 1 << 32
_UINT32_MASK = _UINT32_MODULUS - 1
_UINT32_HALF_RANGE = 1 << 31
_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1


def unwrap_uint32_serial(
    wire_value: int,
    absolute_reference: int,
) -> int | None:
    """Return the unique nearest congruent integer, or None at half range."""
    if type(wire_value) is not int:
        raise TypeError("wire_value must be an exact integer")
    if not 0 <= wire_value <= _UINT32_MASK:
        raise ValueError("wire_value is outside the uint32 range")
    if type(absolute_reference) is not int:
        raise TypeError("absolute_reference must be an exact integer")
    if not _MIN_INT64 <= absolute_reference <= _MAX_INT64:
        raise ValueError("absolute_reference is outside the signed 64-bit range")

    reference_residue = absolute_reference & _UINT32_MASK
    forward_distance = (wire_value - reference_residue) & _UINT32_MASK
    if forward_distance == _UINT32_HALF_RANGE:
        return None

    signed_distance = (
        forward_distance
        if forward_distance < _UINT32_HALF_RANGE
        else forward_distance - _UINT32_MODULUS
    )
    return absolute_reference + signed_distance
```

## Example

```python
modulus = 1 << 32

forward_across_wrap = unwrap_uint32_serial(1, modulus - 2)
backward_across_wrap = unwrap_uint32_serial(modulus - 2, modulus + 1)
from_negative_epoch = unwrap_uint32_serial(1, -1)
ambiguous = unwrap_uint32_serial(1 << 31, 0)

assert (
    forward_across_wrap,
    backward_across_wrap,
    from_negative_epoch,
    ambiguous,
) == (
    modulus + 1,
    modulus - 2,
    1,
    None,
)
```

## Trade-offs and Limitations

The calculation uses `O(1)` time and memory. Both inputs are fixed-width
bounded integers, while Python keeps the addition exact. A result can lie up to
`2**31 - 1` below the signed-64-bit minimum or above its maximum because the
reference, rather than the result, owns that input bound.

Congruent representatives are `2**32` apart. Except at an exact distance of
`2**31`, exactly one representative is strictly closer to the reference than
all others. At that half-range tie the function returns `None`; choosing either
epoch would add policy that the two inputs do not contain.

This operation is stateless. It does not infer missing wraps, compare a stream
of serials, detect resets, reject replays, tolerate arbitrary loss, or define
TCP, RTP, or another protocol's ordering rules. Its result is reliable only
when the caller's reference is already close enough to identify the intended
epoch.

## Related Snippets

<!-- catalog:related:start -->
- [Suppress Stale Keyed Events with Strictly Increasing Sequence Numbers](../reliability-resilience/suppress-stale-keyed-events-with-strictly-increasing-sequence-numbers.md)
- [Read and Write Size-Capped Varint Frames](read-and-write-size-capped-varint-frames.md)
- [Encode and Decode One Exact-Size Big-Endian Binary Record with struct.Struct](../configuration-serialization/encode-and-decode-one-exact-size-big-endian-binary-record-with-struct.md)
<!-- catalog:related:end -->
