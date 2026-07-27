---
title: "Verify a Bounded Byte Stream Before Returning Its Payload"
snippet_type: recipe
use_cases:
  - resource-management
  - security
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - authenticate-bounded-payloads-with-versioned-hmac-keys.md
  - ../storage-databases/store-bytes-by-their-content-digest.md
---

# Verify a Bounded Byte Stream Before Returning Its Payload

## Idea and Problem

Verify a bounded iterable of exact byte chunks against a trusted length and canonical SHA-256 digest before returning any payload bytes.

Chunk count, individual chunk size, and cumulative byte limits are enforced as
the iterable is consumed in order. The function retains bytes privately while
hashing and returns one immutable `bytes` value only after both the length and
digest match. Distinct exceptions separate malformed chunks, resource-limit
overflow, length mismatch, and digest mismatch.

## When to Use

Use this recipe at an in-memory boundary where a producer supplies chunks but
the caller independently knows the exact byte length and a trusted SHA-256
digest. The complete payload must fit the chosen total limit, and consuming the
iterable once must be acceptable.

Set all three resource limits from the receiving boundary's policy. An
unkeyed digest detects deviation from a trusted expected value; a digest
obtained from the same untrusted source does not establish authenticity.

## Implementation

```python
import hmac
import re
from collections.abc import Iterable
from hashlib import sha256


_SHA256_HEX = re.compile(r"[0-9a-f]{64}", re.ASCII)
_MAX_CHUNKS = 100_000
_MAX_CHUNK_BYTES = 8 * 1024 * 1024
_MAX_TOTAL_BYTES = 128 * 1024 * 1024


class ByteStreamVerificationError(ValueError):
    pass


class MalformedByteStreamError(ByteStreamVerificationError):
    pass


class ByteStreamLimitExceeded(ByteStreamVerificationError):
    pass


class ByteStreamLengthMismatch(ByteStreamVerificationError):
    pass


class ByteStreamDigestMismatch(ByteStreamVerificationError):
    pass


def _bounded_stream_limit(
    value: int,
    *,
    name: str,
    minimum: int,
    maximum: int,
) -> int:
    if type(value) is not int:
        raise TypeError(f"{name} must be an exact integer")
    if not minimum <= value <= maximum:
        raise ValueError(f"{name} is outside the supported range")
    return value


def verify_bounded_byte_stream(
    chunks: Iterable[bytes],
    *,
    expected_length: int,
    expected_sha256: str,
    max_chunks: int,
    max_chunk_bytes: int,
    max_total_bytes: int,
) -> bytes:
    expected_length = _bounded_stream_limit(
        expected_length,
        name="expected_length",
        minimum=0,
        maximum=_MAX_TOTAL_BYTES,
    )
    max_chunks = _bounded_stream_limit(
        max_chunks,
        name="max_chunks",
        minimum=0,
        maximum=_MAX_CHUNKS,
    )
    max_chunk_bytes = _bounded_stream_limit(
        max_chunk_bytes,
        name="max_chunk_bytes",
        minimum=1,
        maximum=_MAX_CHUNK_BYTES,
    )
    max_total_bytes = _bounded_stream_limit(
        max_total_bytes,
        name="max_total_bytes",
        minimum=0,
        maximum=_MAX_TOTAL_BYTES,
    )
    if expected_length > max_total_bytes:
        raise ByteStreamLimitExceeded("expected length exceeds max_total_bytes")
    if type(expected_sha256) is not str:
        raise TypeError("expected_sha256 must be exact text")
    if _SHA256_HEX.fullmatch(expected_sha256) is None:
        raise MalformedByteStreamError(
            "expected_sha256 must be 64 lowercase hexadecimal characters"
        )

    expected_digest = bytes.fromhex(expected_sha256)
    digest = sha256()
    payload = bytearray()
    chunk_count = 0
    total_bytes = 0

    for chunk in chunks:
        chunk_count += 1
        if chunk_count > max_chunks:
            raise ByteStreamLimitExceeded("max_chunks was exceeded")
        if type(chunk) is not bytes:
            raise MalformedByteStreamError("chunks must contain exact bytes")
        if len(chunk) > max_chunk_bytes:
            raise ByteStreamLimitExceeded("max_chunk_bytes was exceeded")
        if len(chunk) > max_total_bytes - total_bytes:
            raise ByteStreamLimitExceeded("max_total_bytes was exceeded")

        total_bytes += len(chunk)
        if total_bytes > expected_length:
            raise ByteStreamLengthMismatch(
                "stream contains more bytes than expected"
            )
        digest.update(chunk)
        payload.extend(chunk)

    if total_bytes != expected_length:
        raise ByteStreamLengthMismatch("stream length does not match expected_length")
    if not hmac.compare_digest(digest.digest(), expected_digest):
        raise ByteStreamDigestMismatch("stream digest does not match expected_sha256")
    return bytes(payload)
```

## Example

```python
expected_payload = b"north\x00south"
expected_digest = sha256(expected_payload).hexdigest()
verified = verify_bounded_byte_stream(
    (b"north", b"", b"\x00south"),
    expected_length=len(expected_payload),
    expected_sha256=expected_digest,
    max_chunks=4,
    max_chunk_bytes=8,
    max_total_bytes=16,
)

try:
    verify_bounded_byte_stream(
        (expected_payload,),
        expected_length=len(expected_payload),
        expected_sha256="0" * 64,
        max_chunks=1,
        max_chunk_bytes=16,
        max_total_bytes=16,
    )
except ByteStreamDigestMismatch:
    digest_mismatch_rejected = True
else:
    digest_mismatch_rejected = False

assert (verified, type(verified), digest_mismatch_rejected) == (
    expected_payload,
    bytes,
    True,
)
```

## Trade-offs and Limitations

The function consumes the iterable once and retains the accepted content in a
byte array. Converting that array to immutable bytes briefly needs a second
payload-sized allocation. A producer has already allocated each chunk before
the helper can reject an oversized one, so upstream code should also bound
chunk creation. Empty chunks are valid and count toward `max_chunks`, which
prevents an endless stream of empty values from bypassing the count budget.

An overlong stream fails as soon as it passes the trusted expected length; a
short stream can be identified only when iteration ends. Digest comparison
happens only after exact length validation and uses constant-time byte
comparison. Exceptions raised while obtaining the iterator or advancing it
are not wrapped, so producer failures propagate unchanged. No partial payload
is attached to any verification exception.

SHA-256 is an unkeyed integrity check, not proof of origin. The expected digest
must arrive through a trustworthy mechanism; use a reviewed keyed or signed
scheme when an adversary can replace both content and expectation. This helper
does not acquire content, retain verified values, or manage recovery policy.

## Related Snippets

<!-- catalog:related:start -->
- [Authenticate Bounded Payloads with Versioned HMAC Keys](authenticate-bounded-payloads-with-versioned-hmac-keys.md)
- [Store Bytes by Their Content Digest](../storage-databases/store-bytes-by-their-content-digest.md)
<!-- catalog:related:end -->
