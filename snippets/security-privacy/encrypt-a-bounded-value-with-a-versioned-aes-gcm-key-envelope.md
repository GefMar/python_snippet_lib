---
title: "Encrypt a Bounded Value with a Versioned AES-GCM Key Envelope"
snippet_type: integration
use_cases:
  - security
  - serialization
  - validation
tested_python:
  - "3.14"
dependencies:
  - name: cryptography
    version: "49.0.0"
related:
  - authenticate-bounded-payloads-with-versioned-hmac-keys.md
  - redact-a-printable-ascii-secret-with-a-bounded-visible-tail.md
  - ../configuration-serialization/append-a-fixed-width-crc-check-to-a-human-readable-identifier.md
---

# Encrypt a Bounded Value with a Versioned AES-GCM Key Envelope

## Idea and Problem

Encrypt one bounded byte value in a canonical envelope that identifies a retained key while authenticating the format version, purpose, and key ID as associated data.

AES-GCM provides confidentiality and integrity, but an application still owns
the surrounding format and key lifecycle. One active key encrypts new values;
a small immutable key ring keeps older values readable during rotation. The
public key ID selects a candidate key, while authenticated associated data
prevents substituting another valid ID, version, or purpose.

## When to Use

Use this pattern for small values that must be stored or transported as text
and decrypted locally from a finite key set. Keep encryption keys in a real
secret manager, supply a stable purpose for one data domain, and rotate the
active key under an issuance policy that keeps nonce-collision risk acceptable.
Assign each raw key to exactly one key ID and purpose; never alias its bytes
across another ring or encryption domain.

Use a protocol or storage system with built-in envelope encryption when one is
available. Use a streaming construction for large content, and use a password
KDF before AES-GCM when the only secret is a human password. Review the
[`AESGCM` documentation](https://cryptography.io/en/stable/hazmat/primitives/aead/)
before changing nonce, tag, or associated-data behavior.

## Implementation

```python
import base64
import binascii
import re
import secrets
from dataclasses import dataclass, field

from cryptography.exceptions import InvalidTag
from cryptography.hazmat.primitives.ciphers.aead import AESGCM


_VERSION = "v1"
_MAX_KEYS = 8
_MAX_PLAINTEXT_BYTES = 64 * 1024
_MAX_ENVELOPE_CHARACTERS = 100_000
_KEY_ID = re.compile(r"[a-z0-9][a-z0-9_-]{0,31}", re.ASCII)
_PURPOSE = re.compile(r"[a-z][a-z0-9._-]{0,63}", re.ASCII)
_BASE64URL = re.compile(r"[A-Za-z0-9_-]+", re.ASCII)


class EnvelopeFormatError(ValueError):
    pass


class EnvelopeAuthenticationError(ValueError):
    pass


@dataclass(frozen=True, slots=True)
class AesKey:
    key_id: str
    key: bytes = field(repr=False)


@dataclass(frozen=True, slots=True)
class AesKeyRing:
    purpose: str
    active_key_id: str
    keys: tuple[AesKey, ...] = field(repr=False)


def _validated_key_map(key_ring: object) -> dict[str, bytes]:
    if not isinstance(key_ring, AesKeyRing):
        raise TypeError("key_ring must be an AesKeyRing")
    if _PURPOSE.fullmatch(key_ring.purpose) is None:
        raise ValueError("purpose is outside the canonical format")
    if _KEY_ID.fullmatch(key_ring.active_key_id) is None:
        raise ValueError("active_key_id is outside the canonical format")
    if not isinstance(key_ring.keys, tuple):
        raise TypeError("keys must be a tuple")
    if not 1 <= len(key_ring.keys) <= _MAX_KEYS:
        raise ValueError("key count is outside the supported range")

    result: dict[str, bytes] = {}
    key_material: set[bytes] = set()
    for item in key_ring.keys:
        if not isinstance(item, AesKey):
            raise TypeError("keys must contain AesKey values")
        if _KEY_ID.fullmatch(item.key_id) is None:
            raise ValueError("a key ID is outside the canonical format")
        if not isinstance(item.key, bytes):
            raise TypeError("AES keys must be immutable bytes")
        if len(item.key) not in (16, 24, 32):
            raise ValueError("AES keys must contain 128, 192, or 256 bits")
        if item.key_id in result:
            raise ValueError("key IDs must be unique")
        if item.key in key_material:
            raise ValueError("AES key material must be unique within a key ring")
        result[item.key_id] = item.key
        key_material.add(item.key)
    if key_ring.active_key_id not in result:
        raise ValueError("active_key_id is not present in keys")
    return result


def _associated_data(purpose: str, key_id: str) -> bytes:
    return b"bounded-aes-gcm\x00" + _VERSION.encode("ascii") + b"\x00" + (
        purpose.encode("ascii") + b"\x00" + key_id.encode("ascii")
    )


def _encode_base64url(data: bytes) -> str:
    return base64.urlsafe_b64encode(data).rstrip(b"=").decode("ascii")


def _decode_base64url(token: str, *, maximum_bytes: int) -> bytes:
    maximum_characters = (maximum_bytes * 4 + 2) // 3
    if (
        not 1 <= len(token) <= maximum_characters
        or _BASE64URL.fullmatch(token) is None
    ):
        raise EnvelopeFormatError("an envelope token is outside the format")
    padded = token + "=" * (-len(token) % 4)
    try:
        decoded = base64.b64decode(
            padded,
            altchars=b"-_",
            validate=True,
        )
    except (binascii.Error, ValueError):
        raise EnvelopeFormatError("an envelope token is not valid base64url") from None
    if len(decoded) > maximum_bytes or _encode_base64url(decoded) != token:
        raise EnvelopeFormatError("an envelope token is not canonical")
    return decoded


def encrypt_value(key_ring: AesKeyRing, plaintext: bytes) -> str:
    keys = _validated_key_map(key_ring)
    if not isinstance(plaintext, bytes):
        raise TypeError("plaintext must be immutable bytes")
    if len(plaintext) > _MAX_PLAINTEXT_BYTES:
        raise ValueError("plaintext exceeds the supported size")

    key_id = key_ring.active_key_id
    nonce = secrets.token_bytes(12)
    encrypted = AESGCM(keys[key_id]).encrypt(
        nonce,
        plaintext,
        _associated_data(key_ring.purpose, key_id),
    )
    envelope = ".".join(
        (_VERSION, key_id, _encode_base64url(nonce), _encode_base64url(encrypted))
    )
    if len(envelope) > _MAX_ENVELOPE_CHARACTERS:
        raise AssertionError("validated plaintext produced an oversized envelope")
    return envelope


def decrypt_value(key_ring: AesKeyRing, envelope: str) -> bytes:
    keys = _validated_key_map(key_ring)
    if not isinstance(envelope, str):
        raise TypeError("envelope must be text")
    if not 1 <= len(envelope) <= _MAX_ENVELOPE_CHARACTERS or not envelope.isascii():
        raise EnvelopeFormatError("envelope size or encoding is invalid")

    fields = envelope.split(".")
    if len(fields) != 4:
        raise EnvelopeFormatError("envelope must contain exactly four fields")
    version, key_id, nonce_token, encrypted_token = fields
    if version != _VERSION or _KEY_ID.fullmatch(key_id) is None:
        raise EnvelopeFormatError("envelope version or key ID is invalid")
    key = keys.get(key_id)
    if key is None:
        raise EnvelopeFormatError("envelope references an unavailable key")

    nonce = _decode_base64url(nonce_token, maximum_bytes=12)
    if len(nonce) != 12:
        raise EnvelopeFormatError("AES-GCM nonce must contain 12 bytes")
    encrypted = _decode_base64url(
        encrypted_token,
        maximum_bytes=_MAX_PLAINTEXT_BYTES + 16,
    )
    if not 16 <= len(encrypted) <= _MAX_PLAINTEXT_BYTES + 16:
        raise EnvelopeFormatError("encrypted value size is invalid")

    try:
        return AESGCM(key).decrypt(
            nonce,
            encrypted,
            _associated_data(key_ring.purpose, key_id),
        )
    except InvalidTag:
        raise EnvelopeAuthenticationError("envelope authentication failed") from None
```

## Example

```python
old_key = AESGCM.generate_key(bit_length=128)
current_key = AESGCM.generate_key(bit_length=256)
ring = AesKeyRing(
    purpose="profile-field",
    active_key_id="current-2",
    keys=(
        AesKey("retired-1", old_key),
        AesKey("current-2", current_key),
    ),
)

envelope = encrypt_value(ring, b"confidential value")
round_trip = decrypt_value(ring, envelope)
previous_ring = AesKeyRing(
    purpose=ring.purpose,
    active_key_id="retired-1",
    keys=ring.keys,
)
retired_envelope = encrypt_value(previous_ring, b"retained value")
retained_round_trip = decrypt_value(ring, retired_envelope)

fields = envelope.split(".")
encrypted = fields[-1]
fields[-1] = ("A" if encrypted[0] != "A" else "B") + encrypted[1:]
try:
    decrypt_value(ring, ".".join(fields))
except EnvelopeAuthenticationError:
    tampering_rejected = True
else:
    tampering_rejected = False

wrong_purpose = AesKeyRing(
    purpose="another-field",
    active_key_id=ring.active_key_id,
    keys=ring.keys,
)
try:
    decrypt_value(wrong_purpose, envelope)
except EnvelopeAuthenticationError:
    cross_purpose_rejected = True
else:
    cross_purpose_rejected = False

assert (
    round_trip,
    retained_round_trip,
    envelope.startswith("v1.current-2."),
    tampering_rejected,
    cross_purpose_rejected,
) == (
    b"confidential value",
    b"retained value",
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

Reusing a nonce with the same key can destroy AES-GCM security. Random 96-bit
nonces make collisions unlikely only under a bounded issuance volume; systems
must count that volume across every use of the actual key, not separately by
key ID or purpose. Systems with very high volume need a durable per-key
nonce-allocation strategy or a different approved construction. Rotation must
retain old keys for as long as old envelopes remain readable, while compromise
response may require making some values permanently unavailable.

The version and key ID are visible, and ciphertext length reveals plaintext
length plus the fixed tag. Unknown-key and malformed-format errors are separate
from authentication failure because the key ID is public; avoid exposing them
as a high-rate remote oracle. This recipe does not erase immutable key bytes
from memory, stream large values, derive keys from passwords, manage secrets,
prevent rollback to an older valid envelope, or authorize decryption.

## Related Snippets

<!-- catalog:related:start -->
- [Authenticate Bounded Payloads with Versioned HMAC Keys](authenticate-bounded-payloads-with-versioned-hmac-keys.md)
- [Redact a Printable ASCII Secret with a Bounded Visible Tail](redact-a-printable-ascii-secret-with-a-bounded-visible-tail.md)
- [Append a Fixed-Width CRC Check to a Human-Readable Identifier](../configuration-serialization/append-a-fixed-width-crc-check-to-a-human-readable-identifier.md)
<!-- catalog:related:end -->
