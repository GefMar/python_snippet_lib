---
title: "Derive Collision-Audited Keyed Pseudonyms for Bounded Identifiers"
snippet_type: pattern
use_cases:
  - security
  - data-transformation
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - authenticate-bounded-payloads-with-versioned-hmac-keys.md
  - derive-bounded-key-material-with-rfc-5869-hkdf-sha-256.md
  - ../testing-tooling/build-a-collision-audited-artifact-copy-plan.md
---

# Derive Collision-Audited Keyed Pseudonyms for Bounded Identifiers

## Idea and Problem

Derive deterministic, purpose-separated pseudonyms for a bounded batch of text identifiers and reject any observed truncated-tag collision between distinct identifiers.

Each identifier is framed with a versioned domain, a one-byte purpose length,
and a four-byte identifier length before HMAC-SHA-256 is applied. The first 16
tag bytes become an unpadded base64url token with a `p1_` prefix. Input order is
preserved, and duplicate identifiers produce the same token at every position.

## When to Use

Use this pattern when an internal workflow needs stable opaque join keys but
should not carry the original bounded identifiers into every downstream table
or report. Keep the exact 32-byte key in a managed secret store, dedicate it to
this pseudonym scheme, and use a narrow purpose such as `billing_export` to
separate otherwise identical identifiers used in different domains.

This is pseudonymization, not anonymization. Deterministic output reveals
equality and frequency, a small identifier space can be enumerated by anyone
who obtains the key, and changing or losing the key breaks stable joins. Apply
access controls and retention rules to both the key and the resulting tokens.

## Implementation

```python
import base64
import hmac
import re

_DOMAIN = b"bounded-keyed-pseudonym-v1\x00"
_PURPOSE_PATTERN = re.compile(r"[a-z][a-z0-9_-]{0,31}\Z", re.ASCII)
_KEY_BYTES = 32
_TAG_BYTES = 16
_MAX_IDENTIFIERS = 1_024
_MAX_IDENTIFIER_BYTES = 1_024
_MAX_TOTAL_IDENTIFIER_BYTES = 64 * 1_024


class PseudonymCollisionError(RuntimeError):
    def __init__(self, first_index: int, second_index: int) -> None:
        self.first_index = first_index
        self.second_index = second_index
        super().__init__(
            "distinct identifiers produced one pseudonym at indices "
            f"{first_index} and {second_index}"
        )


def _pseudonym_frame(purpose: bytes, identifier: bytes) -> bytes:
    return b"".join(
        (
            _DOMAIN,
            len(purpose).to_bytes(1, "big"),
            purpose,
            len(identifier).to_bytes(4, "big"),
            identifier,
        )
    )


def _encode_pseudonym(tag: bytes) -> str:
    encoded = base64.urlsafe_b64encode(tag).rstrip(b"=").decode("ascii")
    return f"p1_{encoded}"


def derive_collision_audited_pseudonyms(
    key: bytes,
    purpose: str,
    identifiers: tuple[str, ...],
) -> tuple[str, ...]:
    """Return stable keyed pseudonyms in the identifiers' original order."""
    if type(key) is not bytes:
        raise TypeError("key must be exact bytes")
    if len(key) != _KEY_BYTES:
        raise ValueError("key must contain exactly 32 bytes")
    if type(purpose) is not str:
        raise TypeError("purpose must be an exact string")
    if _PURPOSE_PATTERN.fullmatch(purpose) is None:
        raise ValueError("purpose is outside the closed ASCII profile")
    if type(identifiers) is not tuple:
        raise TypeError("identifiers must be an exact tuple")
    if not 1 <= len(identifiers) <= _MAX_IDENTIFIERS:
        raise ValueError("identifier count is outside the supported range")

    encoded_identifiers: list[bytes] = []
    total_bytes = 0
    for index, identifier in enumerate(identifiers):
        if type(identifier) is not str:
            raise TypeError(f"identifiers[{index}] must be an exact string")
        try:
            encoded = identifier.encode("utf-8")
        except UnicodeEncodeError as error:
            raise ValueError(f"identifiers[{index}] is not valid UTF-8 text") from error
        if not 1 <= len(encoded) <= _MAX_IDENTIFIER_BYTES:
            message = f"identifiers[{index}] must contain 1-1024 UTF-8 bytes"
            raise ValueError(message)
        total_bytes += len(encoded)
        if total_bytes > _MAX_TOTAL_IDENTIFIER_BYTES:
            raise ValueError("aggregate identifier bytes exceed the batch limit")
        encoded_identifiers.append(encoded)

    purpose_bytes = purpose.encode("ascii")
    pseudonyms: list[str] = []
    first_by_tag: dict[bytes, tuple[int, str]] = {}
    for index, (identifier, encoded) in enumerate(
        zip(identifiers, encoded_identifiers, strict=True)
    ):
        tag = hmac.digest(
            key,
            _pseudonym_frame(purpose_bytes, encoded),
            "sha256",
        )[:_TAG_BYTES]
        previous = first_by_tag.get(tag)
        if previous is not None and previous[1] != identifier:
            raise PseudonymCollisionError(previous[0], index)
        if previous is None:
            first_by_tag[tag] = (index, identifier)
        pseudonyms.append(_encode_pseudonym(tag))

    return tuple(pseudonyms)


```

## Example

```python
# Repeated bytes keep the example deterministic; use a managed random key in practice.
example_key = b"k" * 32
identifiers = ("Case-17", "case-17", "Case-17", "café")

tokens = derive_collision_audited_pseudonyms(
    example_key,
    "support_export",
    identifiers,
)
other_purpose = derive_collision_audited_pseudonyms(
    example_key,
    "audit_export",
    identifiers,
)

assert tokens == (
    "p1_EdIh705DwGRtozJtrhEBBw",
    "p1_qS-lknCtWN1ONT_VztN4Hw",
    "p1_EdIh705DwGRtozJtrhEBBw",
    "p1_QQsQMWlkNr8IJTr26uze-Q",
)
assert tokens[0] == tokens[2]
assert tokens[0] != tokens[1]
assert tokens != other_purpose
assert all(token.startswith("p1_") and len(token) == 25 for token in tokens)

try:
    derive_collision_audited_pseudonyms(example_key, "Support", identifiers)
except ValueError:
    invalid_purpose_rejected = True
else:
    invalid_purpose_rejected = False

try:
    derive_collision_audited_pseudonyms(b"short", "support_export", identifiers)
except ValueError:
    invalid_key_rejected = True
else:
    invalid_key_rejected = False

assert invalid_purpose_rejected and invalid_key_rejected
```

## Trade-offs and Limitations

For `N` identifiers containing `B` UTF-8 bytes in total, framing, hashing, and
encoding take `O(B + N)` time. The prevalidated byte copies, output tokens, and
same-batch collision index use `O(B + N)` memory. Validation completes before
any tag is derived, and a detected collision raises one exception containing
indices but no identifier values or key material.

The 128-bit truncation keeps tokens compact but retains a nonzero collision
probability. The audit detects distinct identifiers that collide only within
one call; it cannot discover a collision with tokens created by an earlier
batch unless the caller performs a separately designed persistent uniqueness
check. The function treats text as case-sensitive UTF-8 and performs no Unicode
normalization, so visually similar spellings may remain distinct.

This pattern does not encrypt identifiers, conceal duplicate frequency, prove
consent, enforce deletion, rotate or escrow keys, provide cross-batch collision
storage, or turn pseudonymous data into anonymous data. Do not publish the key,
reuse it for unrelated cryptographic purposes, or treat the fixed key length as
evidence that the key was generated with sufficient entropy.

## Related Snippets

<!-- catalog:related:start -->
- [Authenticate Bounded Payloads with Versioned HMAC Keys](authenticate-bounded-payloads-with-versioned-hmac-keys.md)
- [Derive Bounded Key Material with RFC 5869 HKDF-SHA-256](derive-bounded-key-material-with-rfc-5869-hkdf-sha-256.md)
- [Build a Collision-Audited Artifact Copy Plan](../testing-tooling/build-a-collision-audited-artifact-copy-plan.md)
<!-- catalog:related:end -->
