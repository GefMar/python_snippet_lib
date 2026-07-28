---
title: "Decode Canonical Unpadded Base64url Under Encoded and Decoded Byte Limits"
snippet_type: recipe
use_cases:
  - interoperability
  - parsing
  - serialization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../security-privacy/authenticate-bounded-payloads-with-versioned-hmac-keys.md
  - ../security-privacy/verify-an-rfc-7636-s256-pkce-challenge.md
---

# Decode Canonical Unpadded Base64url Under Encoded and Decoded Byte Limits

## Idea and Problem

Decode one exact non-empty ASCII string only when it is the canonical unpadded base64url representation of bounded bytes.

Alphabet validation rejects padding and non-URL-safe symbols before the
standard-library decoder runs. The decoded length is calculated first, the
decoder receives internal padding and strict validation, and re-encoding the
result catches spellings with nonzero unused pad bits.

## When to Use

Use this recipe when an application format explicitly requires unpadded
base64url and both its textual and binary sizes must be bounded before a value
is accepted. The fixed limits are independent: a string can fit the encoded
limit while its predicted output exceeds the smaller decoded limit.

Use a format-specific implementation when a surrounding protocol defines
additional structure, length, or error-mapping rules. Decoding establishes a
byte representation only; integrity, authenticity, confidentiality, and
authorization require separate mechanisms.

## Implementation

```python
import base64
import binascii
import re
from enum import StrEnum

_MAX_ENCODED_BYTES = 4_096
_MAX_DECODED_BYTES = 2_048
_BASE64URL = re.compile(r"[A-Za-z0-9_-]+", re.ASCII)


class Base64UrlProblem(StrEnum):
    EMPTY_INPUT = "empty_input"
    ENCODED_LIMIT_EXCEEDED = "encoded_limit_exceeded"
    NON_ASCII = "non_ascii"
    PADDING_NOT_ALLOWED = "padding_not_allowed"
    INVALID_ALPHABET = "invalid_alphabet"
    INVALID_LENGTH = "invalid_length"
    DECODED_LIMIT_EXCEEDED = "decoded_limit_exceeded"
    INVALID_ENCODING = "invalid_encoding"
    NON_CANONICAL = "non_canonical"


class Base64UrlDecodeError(ValueError):
    def __init__(self, problem: Base64UrlProblem) -> None:
        self.problem = problem
        super().__init__(problem.value)


def _fail(problem: Base64UrlProblem) -> None:
    raise Base64UrlDecodeError(problem)


def decode_canonical_base64url(text: str) -> bytes:
    if type(text) is not str:
        raise TypeError("base64url input must be an exact string")
    if not text:
        _fail(Base64UrlProblem.EMPTY_INPUT)
    if len(text) > _MAX_ENCODED_BYTES:
        _fail(Base64UrlProblem.ENCODED_LIMIT_EXCEEDED)
    if not text.isascii():
        _fail(Base64UrlProblem.NON_ASCII)
    if "=" in text:
        _fail(Base64UrlProblem.PADDING_NOT_ALLOWED)
    if _BASE64URL.fullmatch(text) is None:
        _fail(Base64UrlProblem.INVALID_ALPHABET)

    remainder = len(text) % 4
    if remainder == 1:
        _fail(Base64UrlProblem.INVALID_LENGTH)
    decoded_length = (len(text) // 4) * 3
    if remainder == 2:
        decoded_length += 1
    elif remainder == 3:
        decoded_length += 2
    if decoded_length > _MAX_DECODED_BYTES:
        _fail(Base64UrlProblem.DECODED_LIMIT_EXCEEDED)

    encoded = text.encode("ascii")
    padded = encoded + b"=" * (-len(encoded) % 4)
    try:
        decoded = base64.b64decode(
            padded,
            altchars=b"-_",
            validate=True,
        )
    except (binascii.Error, ValueError):
        _fail(Base64UrlProblem.INVALID_ENCODING)

    if len(decoded) != decoded_length:
        _fail(Base64UrlProblem.INVALID_ENCODING)
    canonical = base64.urlsafe_b64encode(decoded).rstrip(b"=").decode("ascii")
    if canonical != text:
        _fail(Base64UrlProblem.NON_CANONICAL)
    return decoded
```

## Example

```python
decoded = tuple(
    decode_canonical_base64url(value)
    for value in ("SGVsbG8td29ybGQ", "AA", "_w")
)

invalid_values = ("", "Zg==", "a", "Zm$", "_x", "éA")
problems = []
for value in invalid_values:
    try:
        decode_canonical_base64url(value)
    except Base64UrlDecodeError as error:
        problems.append(error.problem)

assert (decoded, tuple(problems)) == (
    (b"Hello-world", b"\x00", b"\xff"),
    (
        Base64UrlProblem.EMPTY_INPUT,
        Base64UrlProblem.PADDING_NOT_ALLOWED,
        Base64UrlProblem.INVALID_LENGTH,
        Base64UrlProblem.INVALID_ALPHABET,
        Base64UrlProblem.NON_CANONICAL,
        Base64UrlProblem.NON_ASCII,
    ),
)
```

## Trade-offs and Limitations

This function accepts only one canonical, non-empty, unpadded base64url
profile. It rejects ordinary base64 symbols, padded forms, whitespace, and
otherwise decodable spellings whose unused pad bits are nonzero. The complete
input is held in memory, although the encoded and predicted decoded limits
bound both allocations. The returned bytes carry no type or content semantics.
In particular, this decoder is not a token parser or verifier and does not
provide integrity, authenticity, secrecy, freshness, or authorization.

## Related Snippets

<!-- catalog:related:start -->
- [Authenticate Bounded Payloads with Versioned HMAC Keys](../security-privacy/authenticate-bounded-payloads-with-versioned-hmac-keys.md)
- [Verify an RFC 7636 S256 PKCE Challenge](../security-privacy/verify-an-rfc-7636-s256-pkce-challenge.md)
<!-- catalog:related:end -->
