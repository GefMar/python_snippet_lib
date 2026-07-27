---
title: "Append a Fixed-Width CRC Check to a Human-Readable Identifier"
snippet_type: recipe
use_cases:
  - interoperability
  - serialization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - fingerprint-a-set-like-json-array-deterministically.md
  - ../security-privacy/verify-an-rfc-7636-s256-pkce-challenge.md
  - ../storage-databases/store-bytes-by-their-content-digest.md
---

# Append a Fixed-Width CRC Check to a Human-Readable Identifier

## Idea and Problem

Append four canonical base-32 symbols derived from CRC-32 so common transcription errors can be detected before an identifier is accepted.

The payload uses an uppercase alphabet that omits visually ambiguous letters.
The checksum keeps only 20 bits and always emits four symbols, including
leading zeroes, so the complete representation has one canonical spelling.

## When to Use

Use this format when people must copy short, already assigned identifiers and
an accidental substitution or omission should fail early. All producers and
consumers must agree on the alphabet, payload length, separator, CRC variant,
bit truncation, and symbol order. Use a cryptographic message authentication
code or digital signature when an attacker may deliberately alter a value.

## Implementation

```python
import hmac
import zlib


_BASE32_ALPHABET = "0123456789ABCDEFGHJKMNPQRSTVWXYZ"
_BASE32_SYMBOLS = frozenset(_BASE32_ALPHABET)
_CHECK_MASK = (1 << 20) - 1


def _is_canonical_payload(payload: str) -> bool:
    return 1 <= len(payload) <= 64 and all(
        character in _BASE32_SYMBOLS for character in payload
    )


def _crc_check_symbols(payload: str) -> str:
    checksum = zlib.crc32(payload.encode("ascii")) & _CHECK_MASK
    return "".join(
        _BASE32_ALPHABET[(checksum >> shift) & 31]
        for shift in (15, 10, 5, 0)
    )


def build_checked_identifier(payload: str) -> str:
    if not isinstance(payload, str):
        raise TypeError("payload must be text")
    if not _is_canonical_payload(payload):
        raise ValueError("payload is not a canonical base-32 value")
    return f"{payload}-{_crc_check_symbols(payload)}"


def verify_checked_identifier(value: str) -> bool:
    if not isinstance(value, str):
        raise TypeError("value must be text")
    if not 6 <= len(value) <= 69:
        return False

    payload, separator, supplied_check = value.partition("-")
    if not separator:
        return False
    if not _is_canonical_payload(payload):
        return False
    if len(supplied_check) != 4 or any(
        character not in _BASE32_SYMBOLS for character in supplied_check
    ):
        return False
    return hmac.compare_digest(supplied_check, _crc_check_symbols(payload))
```

## Example

```python
identifier = build_checked_identifier("ENTRY7")
leading_zero_check = build_checked_identifier("8")

assert (
    identifier,
    leading_zero_check,
    verify_checked_identifier(identifier),
    verify_checked_identifier("ENTRY8-FZBD"),
    verify_checked_identifier("ENTRY7-FZBE"),
    verify_checked_identifier("entry7-FZBD"),
) == (
    "ENTRY7-FZBD",
    "8-0NRK",
    True,
    False,
    False,
    False,
)
```

## Trade-offs and Limitations

A 20-bit truncated checksum has collisions and provides error detection only:
it does not create unique IDs, hide the payload, prove who produced a value,
or prevent intentional forgery. A CRC is also weaker than a content digest for
detecting arbitrary corruption. The 64-character payload bound and restricted
alphabet exclude otherwise valid identifiers, while the separator and suffix
increase display length. Version the surrounding format before changing any
encoding rule because even symbol order is part of interoperability.

## Related Snippets

<!-- catalog:related:start -->
- [Fingerprint a Set-Like JSON Array Deterministically](fingerprint-a-set-like-json-array-deterministically.md)
- [Verify an RFC 7636 S256 PKCE Challenge](../security-privacy/verify-an-rfc-7636-s256-pkce-challenge.md)
- [Store Bytes by Their Content Digest](../storage-databases/store-bytes-by-their-content-digest.md)
<!-- catalog:related:end -->
