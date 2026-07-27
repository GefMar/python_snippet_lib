---
title: "Authenticate Bounded Payloads with Versioned HMAC Keys"
snippet_type: pattern
use_cases:
  - security
  - serialization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - verify-an-rfc-7636-s256-pkce-challenge.md
  - ../reliability-resilience/assign-stable-schedule-slots-with-a-digest.md
  - ../storage-databases/store-bytes-by-their-content-digest.md
---

# Authenticate Bounded Payloads with Versioned HMAC Keys

## Idea and Problem

Authenticate a bounded payload with the key assigned to the current time epoch while retaining older versioned keys for verification.

The canonical token carries an integer key ID, visible payload, and fixed
HMAC-SHA-256 tag. The authenticated frame includes the key ID and explicit
lengths, so changing a version or moving byte boundaries invalidates the tag.
A verifier accepts retained past keys but not a pre-provisioned future key, and
a bounded coverage check reports missing keys before a future epoch begins.

## When to Use

Use this pattern for compact application tokens when all issuers and verifiers
share a pre-provisioned finite key schedule and the payload may remain visible.
Inject one trusted wall clock, choose an epoch duration that matches operational
rotation, provision independently generated high-entropy keys from a
cryptographically secure source before cutover, and retain old keys for as
long as their tokens must verify. Dedicate each registry to one token purpose;
other purposes need independent keys or a different authenticated domain.
Key length alone does not provide entropy.

Use a standard protocol or managed key service when keys must be fetched,
revoked, audited, shared across trust domains, or backed by hardware. Add
separate expiry and replay controls when token age or one-time use matters.

## Implementation

```python
import base64
import binascii
import hmac
import math
import time
from collections.abc import Callable, Mapping
from dataclasses import dataclass
from hashlib import sha256
from types import MappingProxyType


_DOMAIN = b"bounded-versioned-hmac-token-v1\x00"
_MAX_KEYS = 128
_MIN_KEY_BYTES = 32
_MAX_KEY_BYTES = 1024
_MAX_PAYLOAD_BYTES = 64 * 1024
_MAX_EPOCH = 9_999_999_999
_MAX_COVERAGE_EPOCHS = 128


class HMACConfigurationError(ValueError):
    pass


class MissingSigningKeyError(RuntimeError):
    pass


@dataclass(frozen=True, slots=True)
class KeyCoverage:
    first_epoch: int
    last_epoch: int
    missing_epochs: tuple[int, ...]

    @property
    def complete(self) -> bool:
        return not self.missing_epochs


def _encode_urlsafe(data: bytes) -> str:
    return base64.urlsafe_b64encode(data).rstrip(b"=").decode("ascii")


def _decode_urlsafe(segment: str) -> bytes:
    if not segment.isascii() or "=" in segment:
        raise ValueError("non-canonical base64url")
    padding = "=" * (-len(segment) % 4)
    try:
        decoded = base64.b64decode(
            segment + padding,
            altchars=b"-_",
            validate=True,
        )
    except (binascii.Error, ValueError) as error:
        raise ValueError("invalid base64url") from error
    if _encode_urlsafe(decoded) != segment:
        raise ValueError("non-canonical base64url")
    return decoded


def _authenticated_frame(epoch: int, payload: bytes) -> bytes:
    epoch_bytes = str(epoch).encode("ascii")
    return b"".join(
        (
            _DOMAIN,
            len(epoch_bytes).to_bytes(1, "big"),
            epoch_bytes,
            len(payload).to_bytes(4, "big"),
            payload,
        )
    )


class EpochHMACKeys:
    def __init__(
        self,
        keys: Mapping[int, bytes],
        *,
        epoch_seconds: int,
        clock: Callable[[], float] = time.time,
        max_payload_bytes: int = 4096,
    ) -> None:
        if not isinstance(keys, Mapping):
            raise TypeError("keys must be a mapping")
        if not 1 <= len(keys) <= _MAX_KEYS:
            raise HMACConfigurationError("key count is outside the supported range")
        if not callable(clock):
            raise TypeError("clock must be callable")
        for name, value, lower, upper in (
            ("epoch_seconds", epoch_seconds, 1, 31_536_000),
            ("max_payload_bytes", max_payload_bytes, 0, _MAX_PAYLOAD_BYTES),
        ):
            if isinstance(value, bool) or not isinstance(value, int):
                raise TypeError(f"{name} must be an integer")
            if not lower <= value <= upper:
                raise HMACConfigurationError(
                    f"{name} is outside the supported range"
                )

        copied: dict[int, bytes] = {}
        for epoch, key in keys.items():
            if isinstance(epoch, bool) or not isinstance(epoch, int):
                raise TypeError("key epochs must be integers")
            if not 0 <= epoch <= _MAX_EPOCH:
                raise HMACConfigurationError("a key epoch is outside the supported range")
            if not isinstance(key, bytes):
                raise TypeError("HMAC keys must be immutable bytes")
            if not _MIN_KEY_BYTES <= len(key) <= _MAX_KEY_BYTES:
                raise HMACConfigurationError("an HMAC key has an unsafe length")
            copied[epoch] = key

        self._keys = MappingProxyType(copied)
        self._epoch_seconds = epoch_seconds
        self._clock = clock
        self._max_payload_bytes = max_payload_bytes

    def _timestamp(self) -> float:
        now = self._clock()
        if isinstance(now, bool) or not isinstance(now, (int, float)):
            raise TypeError("clock must return a number")
        timestamp = float(now)
        if not math.isfinite(timestamp) or timestamp < 0:
            raise HMACConfigurationError("clock returned an invalid timestamp")
        return timestamp

    def _current_epoch(self) -> int:
        timestamp = self._timestamp()
        epoch = int(timestamp // self._epoch_seconds)
        if epoch > _MAX_EPOCH:
            raise HMACConfigurationError("current epoch is outside the token format")
        return epoch

    def sign(self, payload: bytes) -> str:
        if not isinstance(payload, bytes):
            raise TypeError("payload must be immutable bytes")
        if len(payload) > self._max_payload_bytes:
            raise ValueError("payload exceeds max_payload_bytes")
        epoch = self._current_epoch()
        key = self._keys.get(epoch)
        if key is None:
            raise MissingSigningKeyError("no signing key covers the current epoch")
        tag = hmac.digest(key, _authenticated_frame(epoch, payload), "sha256")
        return f"v1.{epoch}.{_encode_urlsafe(payload)}.{_encode_urlsafe(tag)}"

    def verify(self, token: object) -> bool:
        if not isinstance(token, str):
            return False
        maximum_token_length = 64 + (4 * self._max_payload_bytes + 2) // 3
        if not 1 <= len(token) <= maximum_token_length or not token.isascii():
            return False
        parts = token.split(".")
        if len(parts) != 4 or parts[0] != "v1":
            return False
        epoch_text, payload_text, tag_text = parts[1:]
        if (
            not epoch_text
            or not epoch_text.isdecimal()
            or len(epoch_text) > 10
        ):
            return False
        epoch = int(epoch_text)
        if str(epoch) != epoch_text or epoch > _MAX_EPOCH:
            return False
        if epoch > self._current_epoch():
            return False
        key = self._keys.get(epoch)
        if key is None:
            return False
        try:
            payload = _decode_urlsafe(payload_text)
            supplied_tag = _decode_urlsafe(tag_text)
        except ValueError:
            return False
        if len(payload) > self._max_payload_bytes or len(supplied_tag) != sha256().digest_size:
            return False
        expected_tag = hmac.digest(
            key,
            _authenticated_frame(epoch, payload),
            "sha256",
        )
        return hmac.compare_digest(expected_tag, supplied_tag)

    def coverage(self, *, foresight_seconds: int) -> KeyCoverage:
        if isinstance(foresight_seconds, bool) or not isinstance(
            foresight_seconds,
            int,
        ):
            raise TypeError("foresight_seconds must be an integer")
        if foresight_seconds < 0:
            raise ValueError("foresight_seconds must be non-negative")
        if foresight_seconds > self._epoch_seconds * _MAX_COVERAGE_EPOCHS:
            raise HMACConfigurationError("coverage window contains too many epochs")
        now = self._timestamp()
        first = int(now // self._epoch_seconds)
        last = int((now + foresight_seconds) // self._epoch_seconds)
        if last - first + 1 > _MAX_COVERAGE_EPOCHS:
            raise HMACConfigurationError("coverage window contains too many epochs")
        if last > _MAX_EPOCH:
            raise HMACConfigurationError("coverage exceeds the token format")
        missing = tuple(epoch for epoch in range(first, last + 1) if epoch not in self._keys)
        return KeyCoverage(first, last, missing)
```

## Example

```python
# Repeated bytes make this example deterministic; they are not production keys.
keys = {
    1: b"o" * 32,
    2: b"c" * 32,
    3: b"n" * 32,
}
current = EpochHMACKeys(keys, epoch_seconds=60, clock=lambda: 125.0)
previous = EpochHMACKeys(keys, epoch_seconds=60, clock=lambda: 65.0)
future = EpochHMACKeys(keys, epoch_seconds=60, clock=lambda: 185.0)

token = current.sign(b"sample")
old_token = previous.sign(b"earlier")
future_token = future.sign(b"later")
version, epoch, payload, tag = token.split(".")
tampered_payload = _encode_urlsafe(b"tamper")
tampered = ".".join((version, epoch, tampered_payload, tag))
unknown_key = ".".join((version, "9", payload, tag))
covered = current.coverage(foresight_seconds=60)
missing = current.coverage(foresight_seconds=130)

assert (
    current.verify(token),
    current.verify(old_token),
    current.verify(future_token),
    current.verify(tampered),
    current.verify(unknown_key),
    covered,
    missing,
) == (
    True,
    True,
    False,
    False,
    False,
    KeyCoverage(2, 3, ()),
    KeyCoverage(2, 4, (4,)),
)
```

## Trade-offs and Limitations

The payload and epoch are encoded, not encrypted. HMAC proves possession of a
shared key but does not provide confidentiality, asymmetric identity, expiry,
replay prevention, revocation, or secure key storage. Verification accepts any
retained epoch, so removal policy must account for the longest legitimate
token lifetime. Production keys must come from a cryptographically secure
random source and remain independent across epochs and token purposes. Reusing
the same keys with this fixed domain lets a valid token cross those purposes.
Unknown key IDs fail locally and never trigger unbounded key fetching.

Every signer and coverage monitor must agree on epoch duration and trusted
wall-clock behavior. Clock jumps can select the wrong signing key, and the
verifier rejects key epochs ahead of its own clock. Deployments must therefore
keep clocks synchronized or define a separately reviewed skew policy. The
coverage calculation rounds outward to whole epochs. A fixed epoch schedule
may be less operationally flexible than an explicit active key ID. Token
parsing is deliberately narrow and bounded; changing framing, algorithms, or
version syntax requires a new format version rather than a permissive parser.

## Related Snippets

<!-- catalog:related:start -->
- [Verify an RFC 7636 S256 PKCE Challenge](verify-an-rfc-7636-s256-pkce-challenge.md)
- [Assign Stable Schedule Slots with a Digest](../reliability-resilience/assign-stable-schedule-slots-with-a-digest.md)
- [Store Bytes by Their Content Digest](../storage-databases/store-bytes-by-their-content-digest.md)
<!-- catalog:related:end -->
