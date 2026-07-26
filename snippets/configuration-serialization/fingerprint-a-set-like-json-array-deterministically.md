---
title: "Fingerprint a Set-Like JSON Array Deterministically"
snippet_type: algorithm
use_cases:
  - serialization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../storage-databases/store-bytes-by-their-content-digest.md
  - substitute-typed-values-into-a-json-like-template.md
---

# Fingerprint a Set-Like JSON Array Deterministically

## Idea and Problem

Fingerprint a strict JSON array as an unordered multiset while preserving duplicate counts and the order of every nested array.

Each top-level element is serialized with one explicit Python JSON contract.
The encoded elements are sorted, length-framed, and fed once into a domain-
separated SHA-256 digest. Mapping insertion order does not matter, but only the
root array is set-like; nested arrays retain ordinary ordered semantics.

## When to Use

Use this fingerprint for local cache keys, change detection, or deduplication
when the root collection is semantically unordered and duplicates still
matter. Freeze the exact encoder and domain prefix as a versioned application
contract. Keep ordinary ordered JSON serialization when root order carries
meaning, and use a formal cross-language canonicalization standard when other
implementations must reproduce the digest.

## Implementation

```python
import hashlib
import json
import math


_DOMAIN = b"json-multiset-v1\x00"


def _validate_json(value: object, active: set[int]) -> None:
    if value is None or type(value) in (bool, int, str):
        return
    if type(value) is float:
        if not math.isfinite(value):
            raise ValueError("JSON numbers must be finite")
        return
    if type(value) is list:
        identity = id(value)
        if identity in active:
            raise ValueError("cyclic lists are not valid JSON")
        active.add(identity)
        try:
            for item in value:
                _validate_json(item, active)
        finally:
            active.remove(identity)
        return
    if type(value) is dict:
        identity = id(value)
        if identity in active:
            raise ValueError("cyclic mappings are not valid JSON")
        if any(type(key) is not str for key in value):
            raise TypeError("JSON object keys must be strings")
        active.add(identity)
        try:
            for item in value.values():
                _validate_json(item, active)
        finally:
            active.remove(identity)
        return
    raise TypeError(f"unsupported JSON value: {type(value).__name__}")


def _canonical_element(value: object) -> bytes:
    return json.dumps(
        value,
        allow_nan=False,
        ensure_ascii=False,
        separators=(",", ":"),
        sort_keys=True,
    ).encode("utf-8")


def fingerprint_json_multiset(items: list[object]) -> str:
    if type(items) is not list:
        raise TypeError("items must be a JSON array")
    _validate_json(items, set())

    encoded_items = sorted(_canonical_element(item) for item in items)
    digest = hashlib.sha256(_DOMAIN)
    digest.update(len(encoded_items).to_bytes(8, byteorder="big"))
    for encoded in encoded_items:
        digest.update(len(encoded).to_bytes(8, byteorder="big"))
        digest.update(encoded)
    return digest.hexdigest()
```

## Example

```python
first = [
    {"name": "alpha", "values": [1, 2]},
    {"enabled": True, "name": "beta"},
    {"name": "alpha", "values": [1, 2]},
]
reordered = [
    {"name": "alpha", "values": [1, 2]},
    {"name": "beta", "enabled": True},
    {"values": [1, 2], "name": "alpha"},
]
nested_order_changed = [
    {"name": "alpha", "values": [2, 1]},
    {"enabled": True, "name": "beta"},
    {"name": "alpha", "values": [1, 2]},
]
one_duplicate_removed = first[:-1]
fingerprint = fingerprint_json_multiset(first)

assert (
    fingerprint == fingerprint_json_multiset(reordered),
    fingerprint != fingerprint_json_multiset(nested_order_changed),
    fingerprint != fingerprint_json_multiset(one_duplicate_removed),
    fingerprint,
) == (
    True,
    True,
    True,
    "ce8bf3f85404ab8680df24cb97036e9cbac6de05a8db9f754ce15d78c97b8f76",
)
```

## Trade-offs and Limitations

Encoding and sorting require `O(n)` element buffers and `O(n log n)` bytewise
comparisons in addition to traversing the input. The Python JSON encoder's
number formatting, Unicode handling, and key ordering are part of this local
contract; strings are not Unicode-normalized. Values such as `1`, `1.0`,
`true`, and `"1"` remain distinct encodings. The unkeyed digest detects
changes but does not authenticate data, prevent chosen-input attacks, or prove
origin. Changing the domain prefix or encoder settings changes every digest.

## Related Snippets

<!-- catalog:related:start -->
- [Store Bytes by Their Content Digest](../storage-databases/store-bytes-by-their-content-digest.md)
- [Substitute Typed Values into a JSON-Like Template](substitute-typed-values-into-a-json-like-template.md)
<!-- catalog:related:end -->
