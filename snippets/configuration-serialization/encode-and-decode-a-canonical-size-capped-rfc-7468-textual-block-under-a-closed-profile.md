---
title: "Encode and Decode a Canonical Size-Capped RFC 7468 Textual Block Under a Closed Profile"
snippet_type: recipe
use_cases:
  - interoperability
  - parsing
  - serialization
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - decode-canonical-unpadded-base64url-under-encoded-and-decoded-byte-limits.md
  - encode-and-decode-a-bounded-full-block-z85-frame.md
  - encode-and-decode-one-exact-size-big-endian-binary-record-with-struct.md
---

# Encode and Decode a Canonical Size-Capped RFC 7468 Textual Block Under a Closed Profile

## Idea and Problem

Encode or decode one non-empty bounded payload using a deliberately canonical RFC 7468 textual block with an explicitly expected label.

The encoder emits matching boundaries, standard padded Base64 wrapped at 64
characters, LF line endings, and exactly one terminal LF. The decoder checks the
derived binary size before codec work and accepts only bytes that the encoder
would reproduce exactly.

## When to Use

Use this recipe when a controlled producer and consumer need a deterministic
textual envelope for one PKIX, PKCS, or CMS object, and the caller already knows
which of the nine supported labels belongs at that boundary. The exact output is
useful for fixtures, stable files, and syntax-level interchange.

Use a format-aware cryptography or PKI library when the enclosed object must be
interpreted or trusted. Use a tolerant RFC 7468 receiver when interoperability
requires alternate line endings, whitespace, surrounding explanatory text, or
other labels that this closed profile intentionally rejects.

## Implementation

```python
import base64
import binascii

_MAX_PAYLOAD_BYTES = 65_536
_MAX_BLOCK_BYTES = 90_000
_LINE_WIDTH = 64
_LABELS = frozenset(
    {
        "ATTRIBUTE CERTIFICATE",
        "CERTIFICATE",
        "CERTIFICATE REQUEST",
        "CMS",
        "ENCRYPTED PRIVATE KEY",
        "PKCS7",
        "PRIVATE KEY",
        "PUBLIC KEY",
        "X509 CRL",
    }
)


class Rfc7468ProfileError(ValueError):
    """The value is outside this canonical textual-block profile."""


def _boundaries(label: str) -> tuple[bytes, bytes]:
    if type(label) is not str:
        raise TypeError("label must be an exact string")
    if label not in _LABELS:
        raise Rfc7468ProfileError("label is outside the supported profile")

    encoded_label = label.encode("ascii")
    return (
        b"-----BEGIN " + encoded_label + b"-----\n",
        b"-----END " + encoded_label + b"-----\n",
    )


def encode_rfc7468_block(payload: bytes, label: str) -> bytes:
    if type(payload) is not bytes:
        raise TypeError("payload must be exact bytes")
    if not 1 <= len(payload) <= _MAX_PAYLOAD_BYTES:
        raise Rfc7468ProfileError("payload size is outside the supported range")

    begin, end = _boundaries(label)
    encoded = base64.b64encode(payload)
    lines = (
        encoded[offset : offset + _LINE_WIDTH] for offset in range(0, len(encoded), _LINE_WIDTH)
    )
    block = begin + b"\n".join(lines) + b"\n" + end
    if len(block) > _MAX_BLOCK_BYTES:
        raise Rfc7468ProfileError("encoded block exceeds the supported size")
    return block


def decode_rfc7468_block(block: bytes, expected_label: str) -> bytes:
    if type(block) is not bytes:
        raise TypeError("block must be exact bytes")
    if not 1 <= len(block) <= _MAX_BLOCK_BYTES:
        raise Rfc7468ProfileError("block size is outside the supported range")

    begin, end = _boundaries(expected_label)
    if b"\r" in block:
        raise Rfc7468ProfileError("only LF line endings are canonical")
    if not block.startswith(begin) or not block.endswith(end):
        raise Rfc7468ProfileError("block boundaries do not match the expected label")

    encoded_section = block[len(begin) : -len(end)]
    if not encoded_section.endswith(b"\n"):
        raise Rfc7468ProfileError("encoded body is not followed by a line feed")
    encoded_lines = encoded_section[:-1].split(b"\n")
    if not encoded_lines or not encoded_lines[-1]:
        raise Rfc7468ProfileError("encoded body must not be empty")
    if any(len(line) != _LINE_WIDTH for line in encoded_lines[:-1]):
        raise Rfc7468ProfileError("non-final Base64 lines must be 64 characters")
    if not 4 <= len(encoded_lines[-1]) <= _LINE_WIDTH:
        raise Rfc7468ProfileError("final Base64 line length is outside the profile")
    if len(encoded_lines[-1]) % 4:
        raise Rfc7468ProfileError("final Base64 line must end on a complete quantum")

    encoded = b"".join(encoded_lines)
    padding = len(encoded) - len(encoded.rstrip(b"="))
    if padding > 2 or b"=" in encoded[: len(encoded) - padding]:
        raise Rfc7468ProfileError("Base64 padding is outside the canonical profile")
    decoded_length = (len(encoded) // 4) * 3 - padding
    if not 1 <= decoded_length <= _MAX_PAYLOAD_BYTES:
        raise Rfc7468ProfileError("derived payload size is outside the supported range")

    try:
        payload = base64.b64decode(encoded, validate=True)
    except (binascii.Error, ValueError) as error:
        raise Rfc7468ProfileError("body is not strict standard Base64") from error
    if len(payload) != decoded_length:
        raise Rfc7468ProfileError("decoded payload length is inconsistent")
    if base64.b64encode(payload) != encoded:
        raise Rfc7468ProfileError("Base64 body is not canonically padded")
    if encode_rfc7468_block(payload, expected_label) != block:
        raise Rfc7468ProfileError("block is not in canonical form")
    return payload
```

## Example

```python


payload = bytes(range(60))
block = encode_rfc7468_block(payload, "PUBLIC KEY")
body_lines = block.split(b"\n")[1:-2]
round_trip = decode_rfc7468_block(block, "PUBLIC KEY")

noncanonical_pad_bits = b"-----BEGIN PUBLIC KEY-----\n/x==\n-----END PUBLIC KEY-----\n"
invalid_blocks = (
    block.replace(b"\n", b"\r\n"),
    b"explanation\n" + block,
    block + block,
    block.replace(b"PUBLIC KEY", b"PRIVATE KEY", 1),
    noncanonical_pad_bits,
)
rejected = []
for invalid_block in invalid_blocks:
    try:
        decode_rfc7468_block(invalid_block, "PUBLIC KEY")
    except Rfc7468ProfileError:
        rejected.append(invalid_block)

assert (
    round_trip,
    tuple(map(len, body_lines)),
    tuple(rejected),
) == (payload, (64, 16), invalid_blocks)
```

## Trade-offs and Limitations

Encoding and decoding are linear in the bounded input size and hold the complete
binary and textual forms in memory. The decoder rejects CRLF, blank lines,
headers, surrounding text, multiple blocks, mismatched labels, alternate line
wrapping, malformed padding, and Base64 spellings with nonzero unused pad bits.
This is intentionally stricter than a general RFC 7468 receiver.

The helper validates only textual framing and Base64 syntax. It does not parse
ASN.1 or DER, distinguish object versions, validate certificate paths, inspect
key parameters, decrypt encrypted keys, verify signatures, or establish trust.
The expected label comes from the caller and does not prove that the payload has
the corresponding structure.

Private-key and certificate material can be sensitive. Returning immutable
`bytes` does not provide secure memory, zeroization, access control, encrypted
storage, or protection against logging and copies. Those concerns require a
separate secret-handling design.

## Related Snippets

<!-- catalog:related:start -->
- [Decode Canonical Unpadded Base64url Under Encoded and Decoded Byte Limits](decode-canonical-unpadded-base64url-under-encoded-and-decoded-byte-limits.md)
- [Encode and Decode a Bounded Full-Block Z85 Frame](encode-and-decode-a-bounded-full-block-z85-frame.md)
- [Encode and Decode One Exact-Size Big-Endian Binary Record with struct.Struct](encode-and-decode-one-exact-size-big-endian-binary-record-with-struct.md)
<!-- catalog:related:end -->
