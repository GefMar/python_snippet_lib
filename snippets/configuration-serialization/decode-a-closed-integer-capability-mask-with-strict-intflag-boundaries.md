---
title: "Decode a Closed Integer Capability Mask with Strict IntFlag Boundaries"
snippet_type: recipe
use_cases:
  - configuration
  - interoperability
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - convert-a-weekday-bitmask-to-a-canonical-cron-schedule.md
  - pack-and-unpack-a-bounded-boolean-tuple-with-explicit-bit-length.md
  - ../networking-protocols/parse-and-format-a-strict-w3c-traceparent-version-00-value.md
---

# Decode a Closed Integer Capability Mask with Strict IntFlag Boundaries

## Idea and Problem

Decode one nonnegative integer from a closed external schema into canonical single-bit capability names while rejecting every unknown bit.

A fixed `IntFlag` with a strict boundary closes enum construction, and
definition checks catch accidental aliases or composites containing unnamed
bits. Explicit input checks are still required because negative `IntFlag`
values use complement semantics even under a strict boundary.

## When to Use

Use this at a configuration or protocol boundary whose producer and consumer
share one versioned bit assignment and where unknown bits must fail closed. The
named composite remains convenient inside the definition, while decoded output
always lists canonical atomic members in definition order.

Use a more forward-compatible representation when newer producers may add bits
that older consumers must preserve. Prefer a schema-specific enum rather than
turning this into a generic dynamic-enum wrapper.

## Implementation

```python
from enum import NAMED_FLAGS, STRICT, UNIQUE, IntFlag, verify


@verify(UNIQUE, NAMED_FLAGS)
class Capability(IntFlag, boundary=STRICT):
    READ = 0x0001
    WRITE = 0x0002
    EXPORT = 0x0004
    READ_WRITE = READ | WRITE


class CapabilityMaskError(ValueError):
    """Raised when an integer is outside the closed capability schema."""


def decode_capability_mask(mask: int) -> tuple[str, ...]:
    """Return canonical atomic capability names in definition order."""
    if type(mask) is not int:
        raise TypeError("mask must be an exact int")
    if not 0 <= mask <= 0xFFFF:
        raise CapabilityMaskError(
            "mask is outside the supported unsigned range"
        )

    try:
        decoded = Capability(mask)
    except ValueError:
        raise CapabilityMaskError(
            "mask contains an unknown capability bit"
        ) from None
    return tuple(member.name for member in decoded)
```

## Example

```python
decoded = {
    mask: decode_capability_mask(mask)
    for mask in range(8)
}

for mask, names in decoded.items():
    rebuilt = Capability(0)
    for name in names:
        rebuilt |= Capability[name]
    assert int(rebuilt) == mask

rejected = 0
for invalid in (
    -1,
    0x0008,
    True,
    0x1_0000,
    Capability.READ,
):
    try:
        decode_capability_mask(invalid)
    except (TypeError, CapabilityMaskError):
        rejected += 1

assert (
    decoded
    == {
        0: (),
        1: ("READ",),
        2: ("WRITE",),
        3: ("READ", "WRITE"),
        4: ("EXPORT",),
        5: ("READ", "EXPORT"),
        6: ("WRITE", "EXPORT"),
        7: ("READ", "WRITE", "EXPORT"),
    }
    and decode_capability_mask(
        int(Capability.READ_WRITE)
    )
    == ("READ", "WRITE")
    and Capability(-1) == Capability(7)
    and rejected == 5
)
```

## Trade-offs and Limitations

Atomic members are the canonical single-bit members: `READ`, `WRITE`, and
`EXPORT`. Iterating an `IntFlag` value returns those contained members in
definition order, so the named `READ_WRITE` composite decodes to two names
instead of becoming a second wire representation. With a fixed small enum,
decoding uses `O(F)` time and result space in the number of atomic flags.

`boundary=STRICT` rejects an unknown positive bit, but it does not replace
the explicit unsigned check: `Capability(-1)` maps to all known bits through
the enum's negative-complement behavior. Exact-`int` validation also rejects
Boolean values and `IntFlag` instances before conversion.

This is a closed-world decoder. Adding a bit requires coordinated schema
evolution, and older consumers reject the new mask. Most importantly, decoding
a capability name does not authenticate a sender or authorize the represented
operation; authorization remains a separate decision.

## Related Snippets

<!-- catalog:related:start -->
- [Convert a Weekday Bitmask to a Canonical Cron Schedule](convert-a-weekday-bitmask-to-a-canonical-cron-schedule.md)
- [Pack and Unpack a Bounded Boolean Tuple with Explicit Bit Length](pack-and-unpack-a-bounded-boolean-tuple-with-explicit-bit-length.md)
- [Parse and Format a Strict W3C traceparent Version 00 Value](../networking-protocols/parse-and-format-a-strict-w3c-traceparent-version-00-value.md)
<!-- catalog:related:end -->
