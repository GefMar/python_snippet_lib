---
title: "Derive Bounded Key Material with RFC 5869 HKDF-SHA-256"
snippet_type: recipe
use_cases:
  - interoperability
  - security
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - authenticate-bounded-payloads-with-versioned-hmac-keys.md
  - create-and-verify-a-short-lived-hmac-download-url.md
  - build-and-verify-a-bounded-sha-256-merkle-inclusion-proof.md
---

# Derive Bounded Key Material with RFC 5869 HKDF-SHA-256

## Idea and Problem

Derive a bounded amount of domain-separated key material with the RFC 5869 extract-then-expand construction.

[RFC 5869](https://www.rfc-editor.org/rfc/rfc5869.html) first extracts a
fixed-size pseudorandom key with HMAC-SHA-256. Expansion then chains HMAC
blocks containing the preceding block, application-defined `info`, and a
one-octet block counter.

Separate extract and expand functions make the boundary explicit. A
composition helper validates input-key material, salt, context, and requested
length before performing any HMAC work.

## When to Use

Use this closed SHA-256 profile when a protocol already supplies suitable
input-key material and specifies how derived keys are separated. Give distinct
logical keys distinct, unambiguous `info` values so their derivations cannot
collide accidentally.

Salt and `info` are not secret. Salt strengthens extraction when it is
available, while `info` binds output to non-secret protocol context. Use the
exact salt and context encoding required by the surrounding protocol.

## Implementation

```python
import hmac

_HASH_LENGTH = 32
_MAX_IKM_BYTES = 4_096
_MAX_SALT_BYTES = 4_096
_MAX_INFO_BYTES = 4_096
_MAX_OUTPUT_BYTES = 255 * _HASH_LENGTH


def _validate_bytes(
    name: str,
    value: object,
    *,
    minimum: int,
    maximum: int,
) -> bytes:
    if type(value) is not bytes:
        raise TypeError(f"{name} must be exact bytes")
    if not minimum <= len(value) <= maximum:
        raise ValueError(f"{name} size is outside the supported range")
    return value


def _validate_salt(salt: object) -> bytes | None:
    if salt is None:
        return None
    return _validate_bytes(
        "salt",
        salt,
        minimum=0,
        maximum=_MAX_SALT_BYTES,
    )


def _validate_length(length: object) -> int:
    if type(length) is not int:
        raise TypeError("length must be an exact integer")
    if not 1 <= length <= _MAX_OUTPUT_BYTES:
        raise ValueError("length is outside the supported range")
    return length


def _extract_unchecked(ikm: bytes, salt: bytes | None) -> bytes:
    effective_salt = b"\x00" * _HASH_LENGTH if salt is None else salt
    return hmac.digest(effective_salt, ikm, "sha256")


def _expand_unchecked(prk: bytes, info: bytes, length: int) -> bytes:
    output = bytearray()
    previous = b""
    block_count = (length + _HASH_LENGTH - 1) // _HASH_LENGTH
    for counter in range(1, block_count + 1):
        previous = hmac.digest(
            prk,
            previous + info + bytes((counter,)),
            "sha256",
        )
        output.extend(previous)
    return bytes(output[:length])


def hkdf_sha256_extract(ikm: bytes, salt: bytes | None = None) -> bytes:
    """Extract one 32-byte pseudorandom key."""
    ikm = _validate_bytes(
        "ikm",
        ikm,
        minimum=1,
        maximum=_MAX_IKM_BYTES,
    )
    salt = _validate_salt(salt)
    return _extract_unchecked(ikm, salt)


def hkdf_sha256_expand(
    prk: bytes,
    info: bytes,
    *,
    length: int,
) -> bytes:
    """Expand one established SHA-256 pseudorandom key."""
    prk = _validate_bytes(
        "prk",
        prk,
        minimum=_HASH_LENGTH,
        maximum=_HASH_LENGTH,
    )
    info = _validate_bytes(
        "info",
        info,
        minimum=0,
        maximum=_MAX_INFO_BYTES,
    )
    length = _validate_length(length)
    return _expand_unchecked(prk, info, length)


def derive_hkdf_sha256(
    ikm: bytes,
    *,
    salt: bytes | None = None,
    info: bytes = b"",
    length: int,
) -> bytes:
    """Extract and expand bounded key material after complete validation."""
    ikm = _validate_bytes(
        "ikm",
        ikm,
        minimum=1,
        maximum=_MAX_IKM_BYTES,
    )
    salt = _validate_salt(salt)
    info = _validate_bytes(
        "info",
        info,
        minimum=0,
        maximum=_MAX_INFO_BYTES,
    )
    length = _validate_length(length)

    prk = _extract_unchecked(ikm, salt)
    return _expand_unchecked(prk, info, length)
```

## Example

```python
rfc_ikm = bytes.fromhex("0b" * 22)
rfc_salt = bytes.fromhex("000102030405060708090a0b0c")
rfc_info = bytes.fromhex("f0f1f2f3f4f5f6f7f8f9")
rfc_prk = bytes.fromhex(
    "077709362c2e32df0ddc3f0dc47bba63"
    "90b6c73bb50f9c3122ec844ad7c2b3e5"
)
rfc_okm = bytes.fromhex(
    "3cb25f25faacd57a90434f64d0362f2a"
    "2d2d0a90cf1a5a4c5db02d56ecc4c5bf"
    "34007208d5b887185865"
)

assert hkdf_sha256_extract(rfc_ikm, rfc_salt) == rfc_prk
assert hkdf_sha256_expand(rfc_prk, rfc_info, length=42) == rfc_okm
assert derive_hkdf_sha256(
    rfc_ikm,
    salt=rfc_salt,
    info=rfc_info,
    length=42,
) == rfc_okm

absent_salt_okm = bytes.fromhex(
    "8da4e775a563c18f715f802a063c5a31"
    "b8a11f5c5ee1879ec3454e5f3c738d2d"
    "9d201395faa4b61a96c8"
)
assert derive_hkdf_sha256(rfc_ikm, length=42) == absent_salt_okm
assert derive_hkdf_sha256(rfc_ikm, salt=b"", length=42) == absent_salt_okm

long_output = derive_hkdf_sha256(
    b"bounded input key material",
    info=b"example protocol\x00traffic key",
    length=64,
)
assert derive_hkdf_sha256(
    b"bounded input key material",
    info=b"example protocol\x00traffic key",
    length=33,
) == long_output[:33]
```

## Trade-offs and Limitations

The composition helper performs one extract HMAC and
`ceil(length / 32)` expand HMACs, at most 256 HMAC operations in total.
Straightforward expansion uses `O(length)` output memory and copies the
bounded `info` value into each HMAC message.

HKDF redistributes existing entropy; it does not create entropy and is not a
password-hardening function. The caller must establish strong input-key
material, define collision-free context framing, assign derived-key purposes,
and follow the surrounding protocol's key schedule.

This code provides no entropy collection, password hashing, key splitting,
storage, rotation, erasure, algorithm agility, encryption, or authentication.
Python also cannot guarantee that temporary key material is erased from memory.
Calling expand directly is safe only with a suitable, correctly established
32-byte pseudorandom key.

## Related Snippets

<!-- catalog:related:start -->
- [Authenticate Bounded Payloads with Versioned HMAC Keys](authenticate-bounded-payloads-with-versioned-hmac-keys.md)
- [Create and Verify a Short-Lived HMAC Download URL](create-and-verify-a-short-lived-hmac-download-url.md)
- [Build and Verify a Bounded SHA-256 Merkle Inclusion Proof](build-and-verify-a-bounded-sha-256-merkle-inclusion-proof.md)
<!-- catalog:related:end -->
