---
title: "Embed a Small Routing Hint in a Random UUIDv8"
snippet_type: recipe
use_cases:
  - interoperability
  - serialization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - append-a-fixed-width-crc-check-to-a-human-readable-identifier.md
  - ../security-privacy/authenticate-bounded-payloads-with-versioned-hmac-keys.md
  - ../storage-databases/store-bytes-by-their-content-digest.md
---

# Embed a Small Routing Hint in a Random UUIDv8

## Idea and Problem

Reserve one to eight custom bits in a UUIDv8 for a small routing hint while filling every remaining custom bit with cryptographically secure randomness.

UUID version 8 is intended for application-defined layouts. Python exposes its
122 custom bits as 48-, 12-, and 62-bit blocks; the version and variant remain
standardized. Placing the hint at the high end of the first custom block gives
creation and extraction one explicit layout without disguising customized data
as a UUIDv4.

## When to Use

Use this layout when an opaque identifier must remain a valid UUID and a tiny,
non-secret hint can avoid an initial routing lookup. Every producer and
consumer must agree on the hint width and meaning. Keep the width small and
stable, and preserve the full UUID as the actual identifier.

The hint is attacker-controlled metadata once an identifier crosses a trust
boundary. Validate routing through authoritative state and authorize the
operation independently. Prefer a normal structured field when the protocol
can carry one, or a standard UUID version when no custom layout is required.

## Implementation

```python
import secrets
from uuid import RFC_4122, UUID, uuid8


_CUSTOM_A_BITS = 48
_MIN_HINT_BITS = 1
_MAX_HINT_BITS = 8


def _validate_hint_bits(hint_bits: int) -> None:
    if isinstance(hint_bits, bool) or not isinstance(hint_bits, int):
        raise TypeError("hint_bits must be an integer")
    if not _MIN_HINT_BITS <= hint_bits <= _MAX_HINT_BITS:
        raise ValueError("hint_bits must be between one and eight")


def create_uuid8_with_hint(hint: int, *, hint_bits: int) -> UUID:
    _validate_hint_bits(hint_bits)
    if isinstance(hint, bool) or not isinstance(hint, int):
        raise TypeError("hint must be an integer")
    if not 0 <= hint < 1 << hint_bits:
        raise ValueError("hint does not fit in hint_bits")

    shift = _CUSTOM_A_BITS - hint_bits
    hint_mask = ((1 << hint_bits) - 1) << shift
    custom_a = secrets.randbits(_CUSTOM_A_BITS)
    custom_a = (custom_a & ~hint_mask) | (hint << shift)

    return uuid8(
        custom_a,
        secrets.randbits(12),
        secrets.randbits(62),
    )


def extract_uuid8_hint(value: UUID, *, hint_bits: int) -> int:
    _validate_hint_bits(hint_bits)
    if not isinstance(value, UUID):
        raise TypeError("value must be a UUID")
    if value.version != 8 or value.variant != RFC_4122:
        raise ValueError("value is not an RFC-variant UUIDv8")

    custom_a = int.from_bytes(value.bytes[:6], "big")
    return custom_a >> (_CUSTOM_A_BITS - hint_bits)
```

## Example

```python
identifier = create_uuid8_with_hint(5, hint_bits=3)
decoded_hint = extract_uuid8_hint(identifier, hint_bits=3)

assert (identifier.version, identifier.variant, decoded_hint) == (
    8,
    RFC_4122,
    5,
)
```

## Trade-offs and Limitations

Reserving `k` custom bits leaves `122 - k` random bits, so identifiers sharing
a hint have less collision resistance than fully random UUIDv8 values. The
remaining space is still large for a small hint, but capacity and collision
requirements must be evaluated for the real issuance rate. The hint also leaks
one small piece of topology or classification to every observer.

An extractor cannot infer the width, meaning, or trustworthiness of the hint.
Changing the layout requires an external versioning or migration strategy, and
random identifiers are not naturally ordered by creation time. Supplying no
arguments to `uuid8()` uses its own pseudo-random defaults, so this recipe
passes every custom block explicitly from `secrets` instead.

## Related Snippets

<!-- catalog:related:start -->
- [Append a Fixed-Width CRC Check to a Human-Readable Identifier](append-a-fixed-width-crc-check-to-a-human-readable-identifier.md)
- [Authenticate Bounded Payloads with Versioned HMAC Keys](../security-privacy/authenticate-bounded-payloads-with-versioned-hmac-keys.md)
- [Store Bytes by Their Content Digest](../storage-databases/store-bytes-by-their-content-digest.md)
<!-- catalog:related:end -->
