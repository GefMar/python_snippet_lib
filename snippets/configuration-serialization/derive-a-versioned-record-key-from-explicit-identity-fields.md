---
title: "Derive a Versioned Record Key from Explicit Identity Fields"
snippet_type: algorithm
use_cases:
  - serialization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - fingerprint-a-set-like-json-array-deterministically.md
  - ../algorithms-data-structures/build-a-canonical-unicode-caseless-comparison-key.md
  - ../storage-databases/store-bytes-by-their-content-digest.md
---

# Derive a Versioned Record Key from Explicit Identity Fields

## Idea and Problem

Hash only an explicit bounded identity projection into a stable record key while keeping mutable metadata outside the key contract.

A versioned canonicalizer makes the equality rule inspectable. Field names are
sorted, text values use one Unicode, case, and whitespace policy, integer and
text values carry different tags, and every byte string is length-framed before
SHA-256. The domain and format version prevent accidental reuse across record
types or later normalization rules.

## When to Use

Use this algorithm when several representations of one logical record need the
same opaque upsert, deletion, or deduplication key. The caller must know which
small fields define identity and must accept case-insensitive, whitespace-folded
text equality. All identity fields and their encoded bytes must fit the fixed
bounds.

Use a database-assigned identifier when the database owns identity. Use a
content digest when every byte defines identity, and preserve case or spacing
when those differences are meaningful. A format change requires a new version
and an explicit migration rather than silently changing existing keys.

## Implementation

```python
import hashlib
import re
import unicodedata
from collections.abc import Iterable
from dataclasses import dataclass
from itertools import islice


IdentityValue = str | int
_MAX_FIELDS = 16
_MAX_TEXT_BYTES = 512
_MAX_FRAMED_BYTES = 8 * 1024
_MIN_INTEGER = -(2**63)
_MAX_INTEGER = 2**63 - 1
_DOMAIN = re.compile(r"[a-z][a-z0-9_-]{0,31}", re.ASCII)
_VERSION = re.compile(r"v[1-9][0-9]{0,5}", re.ASCII)
_FIELD_NAME = re.compile(r"[a-z][a-z0-9_]{0,31}", re.ASCII)


@dataclass(frozen=True, slots=True)
class IdentityField:
    name: str
    value: IdentityValue


def _frame(tag: bytes, payload: bytes) -> bytes:
    return tag + len(payload).to_bytes(4, "big") + payload


def _canonical_value(value: object) -> tuple[bytes, bytes]:
    if isinstance(value, bool):
        raise TypeError("Boolean identity values are not supported")
    if type(value) is int:
        if not _MIN_INTEGER <= value <= _MAX_INTEGER:
            raise ValueError("an integer identity value is outside the supported range")
        return b"I", str(value).encode("ascii")
    if not isinstance(value, str):
        raise TypeError("identity values must be text or integers")

    normalized = " ".join(unicodedata.normalize("NFC", value).casefold().split())
    if not normalized:
        raise ValueError("a text identity value is empty after normalization")
    encoded = normalized.encode("utf-8")
    if len(encoded) > _MAX_TEXT_BYTES:
        raise ValueError("a text identity value exceeds the supported size")
    return b"S", encoded


def derive_record_key(
    domain: str,
    version: str,
    fields: Iterable[IdentityField],
) -> str:
    if not isinstance(domain, str) or _DOMAIN.fullmatch(domain) is None:
        raise ValueError("domain is outside the canonical format")
    if not isinstance(version, str) or _VERSION.fullmatch(version) is None:
        raise ValueError("version is outside the canonical format")
    if isinstance(fields, (str, bytes)):
        raise TypeError("fields must be a non-text iterable")

    snapshot = tuple(islice(fields, _MAX_FIELDS + 1))
    if not 1 <= len(snapshot) <= _MAX_FIELDS:
        raise ValueError("field count is outside the supported range")

    canonical_fields: list[tuple[str, bytes, bytes]] = []
    seen_names: set[str] = set()
    for field in snapshot:
        if not isinstance(field, IdentityField):
            raise TypeError("fields must contain IdentityField values")
        if not isinstance(field.name, str) or _FIELD_NAME.fullmatch(field.name) is None:
            raise ValueError("an identity field name is outside the canonical format")
        if field.name in seen_names:
            raise ValueError("identity field names must be unique")
        seen_names.add(field.name)
        value_tag, value_bytes = _canonical_value(field.value)
        canonical_fields.append((field.name, value_tag, value_bytes))

    framed = bytearray(b"record-key\x00")
    framed.extend(_frame(b"D", domain.encode("ascii")))
    framed.extend(_frame(b"V", version.encode("ascii")))
    for name, value_tag, value_bytes in sorted(canonical_fields):
        framed.extend(_frame(b"N", name.encode("ascii")))
        framed.extend(_frame(value_tag, value_bytes))
        if len(framed) > _MAX_FRAMED_BYTES:
            raise ValueError("framed identity exceeds the supported size")

    digest = hashlib.sha256(framed).hexdigest()
    return f"{version}.{digest}"
```

## Example

```python
first = derive_record_key(
    "contact",
    "v1",
    (
        IdentityField("tenant", 7),
        IdentityField("label", "  Caf\u00e9  NORTH "),
    ),
)
same_identity = derive_record_key(
    "contact",
    "v1",
    (
        IdentityField("label", "CAFE\u0301 north"),
        IdentityField("tenant", 7),
    ),
)
text_seven = derive_record_key(
    "contact",
    "v1",
    (IdentityField("tenant", "7"), IdentityField("label", "caf\u00e9 north")),
)

assert (
    first == same_identity,
    first != text_seven,
    first.startswith("v1."),
) == (True, True, True)
```

## Trade-offs and Limitations

Canonicalization deliberately makes some distinct inputs equal. Case folding,
Unicode normalization, and whitespace collapse are appropriate only when that
is the identity policy. Field removal, field addition, or a policy change needs
a new version and may require dual lookup during migration.

The digest is deterministic, not secret. It reveals equality and can be tested
against guessed low-entropy identities, so it is not an access token or a
password hash. SHA-256 makes accidental collisions impractical, but it cannot
repair an incomplete identity projection: omitting a real identity field merges
records, while including volatile metadata creates unnecessary new keys.

## Related Snippets

<!-- catalog:related:start -->
- [Fingerprint a Set-Like JSON Array Deterministically](fingerprint-a-set-like-json-array-deterministically.md)
- [Build a Canonical Unicode Caseless Comparison Key](../algorithms-data-structures/build-a-canonical-unicode-caseless-comparison-key.md)
- [Store Bytes by Their Content Digest](../storage-databases/store-bytes-by-their-content-digest.md)
<!-- catalog:related:end -->
