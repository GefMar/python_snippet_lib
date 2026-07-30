---
title: "Parse One Canonical OpenSSH Ed25519 Public Key and Compute Its SHA-256 Fingerprint"
snippet_type: recipe
use_cases:
  - interoperability
  - parsing
  - security
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - ../configuration-serialization/encode-and-decode-a-canonical-size-capped-rfc-7468-textual-block-under-a-closed-profile.md
  - ../configuration-serialization/encode-and-decode-one-exact-size-big-endian-binary-record-with-struct.md
  - ../networking-protocols/verify-one-rfc-9530-sha-256-content-digest-under-a-closed-profile.md
---

# Parse One Canonical OpenSSH Ed25519 Public Key and Compute Its SHA-256 Fingerprint

## Idea and Problem

Parse one canonical OpenSSH Ed25519 public-key line, validate its SSH wire strings, and compute the familiar unpadded SHA-256 fingerprint.

The outer line identifies the algorithm and carries a standard-Base64 blob.
Inside that blob, RFC SSH strings use four-byte unsigned big-endian lengths;
the Ed25519 format contains the same algorithm name and exactly 32 key bytes.
Requiring both layers to agree prevents a plausible outer label from hiding a
different or malformed wire key.

## When to Use

Use this parser after isolating one public-key line from a controlled input when
a strict, reproducible representation and OpenSSH-style SHA-256 fingerprint are
needed for display, inventory, comparison, or test fixtures. The optional
comment remains data and is returned verbatim rather than interpreted as an
identity.

Use OpenSSH or a maintained SSH library when reading `authorized_keys`,
`known_hosts`, certificates, private keys, multiple algorithms, or tolerant
whitespace. Successful parsing confirms structure only; authorization and key
ownership require a separately authenticated policy.

## Implementation

```python
import base64
import binascii
import re
from dataclasses import dataclass
from hashlib import sha256

_ALGORITHM = b"ssh-ed25519"
_MAX_LINE_CHARACTERS = 256
_ENCODED_BLOB_CHARACTERS = 68
_BLOB_BYTES = 51
_KEY_BYTES = 32
_LINE = re.compile(
    r"ssh-ed25519 ([A-Za-z0-9+/]{68})"
    r"(?: ([!-~](?:[ -~]{0,126}[!-~])?))?",
    re.ASCII,
)


class OpenSshEd25519PublicKeyError(ValueError):
    """The line is outside the canonical Ed25519 public-key profile."""


@dataclass(frozen=True, slots=True)
class OpenSshEd25519PublicKey:
    blob: bytes
    key: bytes
    comment: str | None
    fingerprint: str


def _take_ssh_string(blob: bytes, offset: int) -> tuple[bytes, int]:
    if len(blob) - offset < 4:
        raise OpenSshEd25519PublicKeyError("an SSH string length is truncated")
    length = int.from_bytes(blob[offset : offset + 4], "big")
    value_start = offset + 4
    value_stop = value_start + length
    if value_stop > len(blob):
        raise OpenSshEd25519PublicKeyError("an SSH string value is truncated")
    return blob[value_start:value_stop], value_stop


def parse_openssh_ed25519_public_key(line: str) -> OpenSshEd25519PublicKey:
    """Parse one exact closed-profile line and compute its SHA-256 fingerprint."""
    if type(line) is not str:
        raise TypeError("line must be an exact string")
    if len(line) > _MAX_LINE_CHARACTERS:
        raise OpenSshEd25519PublicKeyError("line exceeds the character limit")
    if not line.isascii():
        raise OpenSshEd25519PublicKeyError("line must contain only ASCII characters")
    if "\r" in line or "\n" in line or "\t" in line:
        raise OpenSshEd25519PublicKeyError("line must not contain CR, LF, or TAB")

    match = _LINE.fullmatch(line)
    if match is None:
        raise OpenSshEd25519PublicKeyError("line is outside the closed outer grammar")
    encoded_blob, comment = match.groups()
    if len(encoded_blob) != _ENCODED_BLOB_CHARACTERS:
        raise AssertionError("the fixed Base64 grammar returned an unexpected length")

    try:
        blob = base64.b64decode(encoded_blob, validate=True)
    except (binascii.Error, ValueError) as error:
        raise OpenSshEd25519PublicKeyError("key blob is not strict Base64") from error
    if len(blob) != _BLOB_BYTES:
        raise OpenSshEd25519PublicKeyError("key blob must contain exactly 51 bytes")
    if base64.b64encode(blob).decode("ascii") != encoded_blob:
        raise OpenSshEd25519PublicKeyError("key blob Base64 is not canonical")

    algorithm, cursor = _take_ssh_string(blob, 0)
    if algorithm != _ALGORITHM:
        raise OpenSshEd25519PublicKeyError("outer and inner algorithms do not match")
    key, cursor = _take_ssh_string(blob, cursor)
    if len(key) != _KEY_BYTES:
        raise OpenSshEd25519PublicKeyError("Ed25519 public key must contain 32 bytes")
    if cursor != len(blob):
        raise OpenSshEd25519PublicKeyError("key blob contains trailing data")

    fingerprint = "SHA256:" + base64.b64encode(sha256(blob).digest()).decode("ascii").rstrip("=")
    return OpenSshEd25519PublicKey(
        blob=blob,
        key=key,
        comment=comment,
        fingerprint=fingerprint,
    )


```

## Example

```python
fixture_key = bytes(range(32))
fixture_blob = (
    len(b"ssh-ed25519").to_bytes(4, "big")
    + b"ssh-ed25519"
    + len(fixture_key).to_bytes(4, "big")
    + fixture_key
)
encoded_fixture = base64.b64encode(fixture_blob).decode("ascii")
line = f"ssh-ed25519 {encoded_fixture} fixture@example.invalid"

parsed = parse_openssh_ed25519_public_key(line)
without_comment = parse_openssh_ed25519_public_key(f"ssh-ed25519 {encoded_fixture}")

wrong_inner_blob = fixture_blob.replace(b"ssh-ed25519", b"ssh-ed25518", 1)
invalid_lines = (
    line + "\n",
    line.replace(" fixture@", "  fixture@"),
    f"ssh-ed25519 {encoded_fixture[:-1]}=",
    "ssh-ed25519 " + base64.b64encode(wrong_inner_blob).decode("ascii"),
)
rejected = 0
for invalid in invalid_lines:
    try:
        parse_openssh_ed25519_public_key(invalid)
    except OpenSshEd25519PublicKeyError:
        rejected += 1

assert parsed == OpenSshEd25519PublicKey(
    blob=fixture_blob,
    key=fixture_key,
    comment="fixture@example.invalid",
    fingerprint="SHA256:ZkAslGjFiUHdGf/WUL8rQvkib4PTvQatUV0OUQSncCA",
)
assert without_comment.comment is None
assert rejected == len(invalid_lines)
```

## Trade-offs and Limitations

Parsing and hashing are linear in the bounded input, while the fixed 256-character
line and 51-byte blob make time and auxiliary memory constant in practice. The
decoder accepts one exact ASCII spelling: one separator space, canonical
unpadded standard Base64, no line terminator, and an optional printable comment
whose first and last characters are not spaces.

The wire parser accepts any 32 key bytes. It does not validate Ed25519 point
encoding, verify a signature, prove possession of a private key, establish
trust, consult an authorization policy, or compare a host identity. The
SHA-256 fingerprint is a stable identifier for the complete SSH blob, not a
message authentication code or certificate.

This closed profile rejects `authorized_keys` options, `known_hosts` prefixes,
certificates, private keys, non-Ed25519 algorithms, tabs, repeated separators,
leading or trailing comment spaces, and otherwise tolerated file whitespace.
Comments can contain identifying text, and immutable strings and bytes are not
redaction or secure-memory mechanisms; avoid logging them unless policy allows
it.

## Related Snippets

<!-- catalog:related:start -->
- [Encode and Decode a Canonical Size-Capped RFC 7468 Textual Block Under a Closed Profile](../configuration-serialization/encode-and-decode-a-canonical-size-capped-rfc-7468-textual-block-under-a-closed-profile.md)
- [Encode and Decode One Exact-Size Big-Endian Binary Record with struct.Struct](../configuration-serialization/encode-and-decode-one-exact-size-big-endian-binary-record-with-struct.md)
- [Verify One RFC 9530 SHA-256 Content-Digest Under a Closed Profile](../networking-protocols/verify-one-rfc-9530-sha-256-content-digest-under-a-closed-profile.md)
<!-- catalog:related:end -->
