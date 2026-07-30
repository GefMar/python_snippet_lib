---
title: "Verify One RFC 9530 SHA-256 Content-Digest Under a Closed Profile"
snippet_type: recipe
use_cases:
  - networking
  - parsing
  - security
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - extract-bounded-singleton-http-headers-with-explicit-rules.md
  - ../security-privacy/verify-a-bounded-byte-stream-before-returning-its-payload.md
  - ../storage-databases/store-bytes-by-their-content-digest.md
---

# Verify One RFC 9530 SHA-256 Content-Digest Under a Closed Profile

## Idea and Problem

Verify bounded HTTP message content against one exact RFC 9530 SHA-256 Content-Digest field value.

The closed profile accepts only the canonical 54-byte ASCII spelling
`sha-256=:<44-character padded standard Base64>:`. It decodes exactly 32 digest
bytes, re-encodes them to reject non-canonical Base64 pad bits, hashes the
supplied content, and compares the two byte strings without data-dependent
early exit. Malformed fields, oversized content, and valid-but-mismatched
digests have distinct exceptions.

## When to Use

Use this recipe when an HTTP integration has already selected the exact message
content bytes and extracted exactly one field value under a deliberately closed
SHA-256 contract. Both inputs are already materialized, and content from zero
through 1,048,576 bytes fits the receiving boundary's memory policy. Empty
content is valid.

Keep header multiplicity and content-selection rules outside this helper. The
caller must reject repeated field lines before passing one value and must know
which bytes RFC 9530 defines as the message content at its protocol layer.
Successful verification provides integrity only relative to the supplied
digest; it does not authenticate the sender.

## Implementation

```python
import hmac
from base64 import b64decode, b64encode
from binascii import Error as Base64Error
from hashlib import sha256

MAX_HTTP_CONTENT_BYTES = 1_048_576
_CONTENT_DIGEST_BYTES = 32
_CONTENT_DIGEST_BASE64_BYTES = 44
_CONTENT_DIGEST_FIELD_BYTES = 54
_CONTENT_DIGEST_PREFIX = b"sha-256=:"


class ContentDigestError(ValueError):
    pass


class MalformedContentDigestError(ContentDigestError):
    pass


class ContentDigestSizeError(ContentDigestError):
    pass


class ContentDigestMismatchError(ContentDigestError):
    pass


def verify_sha256_content_digest(content: bytes, field_value: str) -> None:
    """Verify exact content against one canonical closed-profile field value."""
    if type(content) is not bytes:
        raise TypeError("content must be exact bytes")
    if len(content) > MAX_HTTP_CONTENT_BYTES:
        raise ContentDigestSizeError("content exceeds the supported byte limit")
    if type(field_value) is not str:
        raise TypeError("field_value must be exact text")
    if len(field_value) != _CONTENT_DIGEST_FIELD_BYTES:
        raise MalformedContentDigestError(
            "field_value must contain exactly 54 characters"
        )

    try:
        raw_field = field_value.encode("ascii")
    except UnicodeEncodeError as error:
        raise MalformedContentDigestError("field_value must contain only ASCII") from error

    if len(raw_field) != _CONTENT_DIGEST_FIELD_BYTES:
        raise MalformedContentDigestError("field_value must be exactly 54 ASCII bytes")
    if not raw_field.startswith(_CONTENT_DIGEST_PREFIX) or not raw_field.endswith(b":"):
        raise MalformedContentDigestError("field_value is outside the closed SHA-256 profile")

    encoded_digest = raw_field[len(_CONTENT_DIGEST_PREFIX) : -1]
    if (
        len(encoded_digest) != _CONTENT_DIGEST_BASE64_BYTES
        or encoded_digest[-1:] != b"="
    ):
        raise MalformedContentDigestError("SHA-256 digest must use padded standard Base64")
    try:
        expected_digest = b64decode(encoded_digest, validate=True)
    except Base64Error as error:
        raise MalformedContentDigestError("SHA-256 digest is not strict Base64") from error
    if len(expected_digest) != _CONTENT_DIGEST_BYTES:
        raise MalformedContentDigestError("SHA-256 digest must decode to exactly 32 bytes")
    if b64encode(expected_digest) != encoded_digest:
        raise MalformedContentDigestError("SHA-256 digest must use canonical Base64")

    if not hmac.compare_digest(sha256(content).digest(), expected_digest):
        raise ContentDigestMismatchError("content does not match the supplied SHA-256 digest")
```

## Example

```python
rfc_content = b'{"hello": "world"}\n'
rfc_field = "sha-256=:RK/0qy18MlBSVnWgjwz6lZEWjP/lF5HF9bvEF8FabDg=:"
rfc_result = verify_sha256_content_digest(rfc_content, rfc_field)

empty_field = (
    "sha-256=:" + b64encode(sha256(b"").digest()).decode("ascii") + ":"
)
empty_result = verify_sha256_content_digest(b"", empty_field)

malformed_values = (
    "",
    rfc_field + " ",
    "sha-512=" + rfc_field.removeprefix("sha-256="),
    "sha-256=:" + "A" * 42 + "!=:",
    "sha-256=:" + "A" * 42 + "B=:",
)
malformed_rejections = 0
for malformed_value in malformed_values:
    try:
        verify_sha256_content_digest(rfc_content, malformed_value)
    except MalformedContentDigestError:
        malformed_rejections += 1

try:
    verify_sha256_content_digest(b"x" * (MAX_HTTP_CONTENT_BYTES + 1), rfc_field)
except ContentDigestSizeError:
    oversized_rejected = True
else:
    oversized_rejected = False

try:
    verify_sha256_content_digest(rfc_content[:-1], rfc_field)
except ContentDigestMismatchError:
    mismatch_rejected = True
else:
    mismatch_rejected = False

assert (
    rfc_result,
    empty_result,
    len(empty_field.encode("ascii")),
    malformed_rejections,
    oversized_rejected,
    mismatch_rejected,
) == (None, None, 54, len(malformed_values), True, True)
```

## Trade-offs and Limitations

The function accepts exact `bytes` and exact `str` only. It allocates a small
decoded digest and hashes the complete in-memory content in `O(n)` time. The
one-mebibyte cap is fixed policy, not an RFC limit. A mismatch uses
`hmac.compare_digest`, but parsing and size failures are intentionally visible
and are not timing-oblivious.

This is not a general Structured Fields dictionary parser. It rejects multiple
algorithms, parameters, optional whitespace, combined values, and alternate
spellings. It does not acquire or merge header or trailer field lines, handle
`Repr-Digest`, stream content, choose the applicable content bytes, or acquire
an HTTP message. Those concerns belong at a protocol boundary that can enforce
field multiplicity and framing.

An unkeyed SHA-256 digest is not sender authentication. If an attacker can
replace both the content and field value, verification still succeeds. Use a
reviewed authenticated or signed protocol when provenance matters.

## Related Snippets

<!-- catalog:related:start -->
- [Extract Bounded Singleton HTTP Headers with Explicit Rules](extract-bounded-singleton-http-headers-with-explicit-rules.md)
- [Verify a Bounded Byte Stream Before Returning Its Payload](../security-privacy/verify-a-bounded-byte-stream-before-returning-its-payload.md)
- [Store Bytes by Their Content Digest](../storage-databases/store-bytes-by-their-content-digest.md)
<!-- catalog:related:end -->
