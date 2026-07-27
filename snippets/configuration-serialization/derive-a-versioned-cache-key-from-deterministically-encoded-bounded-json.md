---
title: "Derive a Versioned Cache Key from Deterministically Encoded Bounded JSON"
snippet_type: algorithm
use_cases:
  - caching
  - configuration
  - serialization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - derive-a-versioned-record-key-from-explicit-identity-fields.md
  - fingerprint-a-set-like-json-array-deterministically.md
  - ../storage-databases/store-bytes-by-their-content-digest.md
---

# Derive a Versioned Cache Key from Deterministically Encoded Bounded JSON

## Idea and Problem

Validate, copy, and deterministically encode one defaults-expanded JSON-like configuration before deriving a versioned BLAKE2b cache key.

The accepted value tree contains only exact built-in JSON-like types under
explicit structural and byte budgets. Top-level defaults fill absent keys,
configuration values win without recursive merging, mapping insertion order is
ignored, and list order remains significant. The returned encoded bytes make
the complete hashing contract inspectable.

## When to Use

Use this algorithm when a small in-memory configuration determines reusable
work and equivalent mappings must produce the same local cache key. Treat the
algorithm label, token grammars, defaults, bounds, and encoder settings as one
versioned contract. The two input roots count as depth one; node and mapping-key
budgets are cumulative across both trees, including overridden defaults.

Use an explicit identity projection when only selected fields define identity,
a set-like array fingerprint when array order should not matter, or a digest of
raw bytes when the original byte representation is the content contract. This
function derives key material only; cache storage and execution policy belong
outside it.

## Implementation

```python
import hashlib
import json
import math
import re
from dataclasses import dataclass
from typing import cast


_ALGORITHM = "deterministic-bounded-json-cache-key-v1"
_MAX_DEPTH = 8
_MAX_NODES = 512
_MAX_MAPPING_KEYS = 256
_MAX_TEXT_BYTES = 1_024
_MAX_ENCODED_BYTES = 65_536
_MIN_INTEGER = -(2**63)
_MAX_INTEGER = 2**63 - 1
_NAMESPACE = re.compile(r"[a-z][a-z0-9_-]{0,31}\Z", re.ASCII)
_REVISION = re.compile(r"v[1-9][0-9]{0,5}\Z", re.ASCII)


class CacheKeyValidationError(ValueError):
    pass


@dataclass(frozen=True, slots=True)
class DerivedCacheKey:
    encoded_bytes: bytes
    key: str


@dataclass(slots=True)
class _Budget:
    nodes: int = 0
    mapping_keys: int = 0


def _check_text(text: str, *, role: str) -> None:
    try:
        size = len(text.encode("utf-8"))
    except UnicodeEncodeError as error:
        raise CacheKeyValidationError(f"{role} must be valid UTF-8") from error
    if size > _MAX_TEXT_BYTES:
        raise CacheKeyValidationError(
            f"{role} cannot exceed {_MAX_TEXT_BYTES} UTF-8 bytes"
        )


def _normalize_json(
    value: object,
    *,
    container_depth: int,
    budget: _Budget,
    active: set[int],
) -> object:
    budget.nodes += 1
    if budget.nodes > _MAX_NODES:
        raise CacheKeyValidationError(
            f"inputs cannot exceed {_MAX_NODES} value occurrences"
        )

    if value is None or type(value) is bool:
        return value
    if type(value) is int:
        if not _MIN_INTEGER <= value <= _MAX_INTEGER:
            raise CacheKeyValidationError("integers must fit signed 64-bit range")
        return value
    if type(value) is float:
        if not math.isfinite(value):
            raise CacheKeyValidationError("floats must be finite")
        return value
    if type(value) is str:
        _check_text(value, role="a string value")
        return value
    if type(value) not in (list, dict):
        raise CacheKeyValidationError("values must use exact JSON-like built-in types")

    next_depth = container_depth + 1
    if next_depth > _MAX_DEPTH:
        raise CacheKeyValidationError(
            f"inputs cannot exceed container depth {_MAX_DEPTH}"
        )
    identity = id(value)
    if identity in active:
        raise CacheKeyValidationError("cyclic containers are not supported")
    active.add(identity)
    try:
        if type(value) is list:
            return [
                _normalize_json(
                    item,
                    container_depth=next_depth,
                    budget=budget,
                    active=active,
                )
                for item in value
            ]

        normalized: dict[str, object] = {}
        for key, item in value.items():
            if type(key) is not str:
                raise CacheKeyValidationError("mapping keys must be exact strings")
            budget.mapping_keys += 1
            if budget.mapping_keys > _MAX_MAPPING_KEYS:
                raise CacheKeyValidationError(
                    f"inputs cannot exceed {_MAX_MAPPING_KEYS} mapping-key occurrences"
                )
            _check_text(key, role="a mapping key")
            normalized[key] = _normalize_json(
                item,
                container_depth=next_depth,
                budget=budget,
                active=active,
            )
        return normalized
    finally:
        active.remove(identity)


def derive_cache_key(
    configuration: dict[str, object],
    defaults: dict[str, object],
    *,
    namespace: str,
    revision: str,
) -> DerivedCacheKey:
    if type(configuration) is not dict or type(defaults) is not dict:
        raise CacheKeyValidationError("configuration and defaults must be exact dictionaries")
    if type(namespace) is not str or _NAMESPACE.fullmatch(namespace) is None:
        raise CacheKeyValidationError("namespace must be a bounded ASCII token")
    if type(revision) is not str or _REVISION.fullmatch(revision) is None:
        raise CacheKeyValidationError("revision must be a bounded ASCII version token")

    budget = _Budget()
    normalized_configuration = cast(
        dict[str, object],
        _normalize_json(
            configuration,
            container_depth=0,
            budget=budget,
            active=set(),
        ),
    )
    normalized_defaults = cast(
        dict[str, object],
        _normalize_json(
            defaults,
            container_depth=0,
            budget=budget,
            active=set(),
        ),
    )

    normalized_config = dict(normalized_defaults)
    normalized_config.update(normalized_configuration)
    envelope = {
        "algorithm": _ALGORITHM,
        "namespace": namespace,
        "revision": revision,
        "configuration": normalized_config,
    }
    encoded_bytes = json.dumps(
        envelope,
        sort_keys=True,
        separators=(",", ":"),
        ensure_ascii=False,
        allow_nan=False,
    ).encode("utf-8")
    if len(encoded_bytes) > _MAX_ENCODED_BYTES:
        raise CacheKeyValidationError(
            f"encoded envelope cannot exceed {_MAX_ENCODED_BYTES} bytes"
        )

    key = hashlib.blake2b(encoded_bytes, digest_size=32).hexdigest()
    return DerivedCacheKey(encoded_bytes=encoded_bytes, key=key)
```

## Example

```python
import copy

configuration = {
    "limits": {"hard": 30},
    "features": ["cache", "audit"],
    "enabled": False,
}
defaults = {
    "enabled": True,
    "limits": {"soft": 10, "hard": 20},
    "region": "eu",
}
before = copy.deepcopy((configuration, defaults))
derived = derive_cache_key(
    configuration,
    defaults,
    namespace="worker",
    revision="v1",
)
expected_encoded_bytes = (
    b'{"algorithm":"deterministic-bounded-json-cache-key-v1",'
    b'"configuration":{"enabled":false,"features":["cache","audit"],'
    b'"limits":{"hard":30},"region":"eu"},"namespace":"worker",'
    b'"revision":"v1"}'
)

equivalent = derive_cache_key(
    {
        "region": "eu",
        "enabled": False,
        "limits": {"hard": 30},
        "features": ["cache", "audit"],
    },
    {
        "region": "eu",
        "limits": {"hard": 20, "soft": 10},
        "enabled": True,
    },
    namespace="worker",
    revision="v1",
)
list_changed = derive_cache_key(
    {**configuration, "features": ["audit", "cache"]},
    defaults,
    namespace="worker",
    revision="v1",
)

assert (configuration, defaults) == before
assert derived.encoded_bytes == expected_encoded_bytes
assert derived.key == "c9200b33eda0756b7c38d1a879e5ff8ff9fee20e557b3e005032c8bdc159a971"
assert equivalent == derived
assert list_changed.key != derived.key

try:
    derive_cache_key(
        {"ratio": float("nan")},
        {},
        namespace="worker",
        revision="v1",
    )
except CacheKeyValidationError:
    non_finite_rejected = True
else:
    non_finite_rejected = False
assert non_finite_rejected
```

## Trade-offs and Limitations

Both inputs are traversed and copied before encoding, so time and auxiliary
space are linear in the accepted occurrences. Shared aliases are copied and
charged once per occurrence, while only an alias on the active traversal path
is a cycle. Defaults apply at the top level only: an explicit configuration
mapping replaces, rather than recursively merges, a default mapping.

The deterministic `json.dumps` settings are a local Python serialization contract,
not a general cross-language canonical JSON standard. Strings are preserved
without Unicode normalization, and floating-point spelling follows Python's
encoder. The unkeyed digest reveals equality and does not authenticate data.
Unlike an explicit identity projection, every normalized configuration value
affects the key; unlike a set-like fingerprint, every list position matters.
Returning encoded bytes does not store either those bytes or a cache value.

## Related Snippets

<!-- catalog:related:start -->
- [Derive a Versioned Record Key from Explicit Identity Fields](derive-a-versioned-record-key-from-explicit-identity-fields.md)
- [Fingerprint a Set-Like JSON Array Deterministically](fingerprint-a-set-like-json-array-deterministically.md)
- [Store Bytes by Their Content Digest](../storage-databases/store-bytes-by-their-content-digest.md)
<!-- catalog:related:end -->
