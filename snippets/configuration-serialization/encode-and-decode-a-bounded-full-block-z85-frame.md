---
title: "Encode and Decode a Bounded Full-Block Z85 Frame"
snippet_type: recipe
use_cases:
  - interoperability
  - serialization
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - decode-canonical-unpadded-base64url-under-encoded-and-decoded-byte-limits.md
  - encode-and-decode-one-exact-size-big-endian-binary-record-with-struct.md
  - ../security-privacy/authenticate-bounded-payloads-with-versioned-hmac-keys.md
---

# Encode and Decode a Bounded Full-Block Z85 Frame

## Idea and Problem

Encode and decode one bounded Z85 frame while accepting only complete binary and textual blocks.

Z85 represents each 4-byte binary block with exactly 5 printable ASCII
characters and uses no padding. Python's codec can process a trailing partial
group, so a format that requires standard Z85 block arithmetic must enforce
divisibility before calling it. Re-encoding decoded bytes also pins acceptance
to one canonical spelling.

## When to Use

Use this recipe when a closed application format explicitly selects Z85 for a
small non-empty binary field and requires complete blocks. The binary side is
limited to 65,536 bytes, and the corresponding text side is limited to 81,920
ASCII characters before codec work begins.

Choose a different encoding when existing consumers require base64, hexadecimal
text, another Base85 alphabet, padding, arbitrary byte lengths, or URL-safe
characters. Add an authentication mechanism when altered or forged content must
be detected.

## Implementation

```python
import base64

_MAX_Z85_BINARY_BYTES = 64 * 1024
_MAX_Z85_TEXT_CHARACTERS = (_MAX_Z85_BINARY_BYTES // 4) * 5


class Z85ProfileError(ValueError):
    pass


def encode_z85_frame(payload: bytes) -> str:
    if type(payload) is not bytes:
        raise TypeError("payload must be exact bytes")
    if not 4 <= len(payload) <= _MAX_Z85_BINARY_BYTES:
        raise Z85ProfileError("payload size is outside the supported range")
    if len(payload) % 4:
        raise Z85ProfileError("payload must contain complete four-byte blocks")

    encoded = base64.z85encode(payload)
    expected_size = (len(payload) // 4) * 5
    if len(encoded) != expected_size:
        raise Z85ProfileError("codec returned an unexpected text size")
    return encoded.decode("ascii")


def decode_z85_frame(text: str) -> bytes:
    if type(text) is not str:
        raise TypeError("text must be an exact string")
    if not 5 <= len(text) <= _MAX_Z85_TEXT_CHARACTERS:
        raise Z85ProfileError("text size is outside the supported range")
    if len(text) % 5:
        raise Z85ProfileError("text must contain complete five-character blocks")
    if not text.isascii():
        raise Z85ProfileError("text must contain only ASCII characters")

    encoded = text.encode("ascii")
    try:
        payload = base64.z85decode(encoded)
    except ValueError as error:
        raise Z85ProfileError("text is not a valid Z85 frame") from error

    expected_size = (len(encoded) // 5) * 4
    if len(payload) != expected_size:
        raise Z85ProfileError("codec returned an unexpected payload size")
    if base64.z85encode(payload) != encoded:
        raise Z85ProfileError("text is not the canonical Z85 spelling")
    return payload
```

## Example

```python
official_payload = bytes.fromhex("864fd26fb559f75b")
official_text = "HelloWorld"

encoded = encode_z85_frame(official_payload)
decoded = decode_z85_frame(official_text)

invalid_values = []
for operation in (
    lambda: encode_z85_frame(b"abc"),
    lambda: decode_z85_frame("Hell"),
    lambda: decode_z85_frame("#####"),
    lambda: decode_z85_frame("Hellé"),
):
    try:
        operation()
    except Z85ProfileError:
        invalid_values.append(True)
    else:
        invalid_values.append(False)

assert (
    encoded,
    decoded,
    encode_z85_frame(decoded),
    tuple(invalid_values),
) == (official_text, official_payload, official_text, (True, True, True, True))
```

## Trade-offs and Limitations

This closed profile rejects empty input and every partial 4-byte or 5-character
group even though the Python codec can transform some partial groups. It also
rejects non-ASCII text, invalid Z85 characters, overflowing five-character
chunks, and any spelling that does not round-trip canonically. Complete input
and output values are held in memory within fixed caps.

Z85 is an encoding, not encryption, authentication, integrity protection, or
compression. Its alphabet is not promised to be safe without escaping in URLs,
shell commands, markup, or another surrounding grammar. Z85 is also not
interchangeable with Ascii85, Git-style Base85, or another alphabet marketed as
Base85.

The size checks describe this format profile, not every possible use of
`base64.z85encode()` and `base64.z85decode()`. A surrounding protocol may need
stricter fixed lengths, typed fields, versioning, or authenticated framing.

## Related Snippets

<!-- catalog:related:start -->
- [Decode Canonical Unpadded Base64url Under Encoded and Decoded Byte Limits](decode-canonical-unpadded-base64url-under-encoded-and-decoded-byte-limits.md)
- [Encode and Decode One Exact-Size Big-Endian Binary Record with struct.Struct](encode-and-decode-one-exact-size-big-endian-binary-record-with-struct.md)
- [Authenticate Bounded Payloads with Versioned HMAC Keys](../security-privacy/authenticate-bounded-payloads-with-versioned-hmac-keys.md)
<!-- catalog:related:end -->
