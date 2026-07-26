---
title: "Build a Read-Only Mapping with Canonical Text Keys"
snippet_type: recipe
use_cases:
  - interoperability
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../algorithms-data-structures/build-a-canonical-unicode-caseless-comparison-key.md
---

# Build a Read-Only Mapping with Canonical Text Keys

## Idea and Problem

Build a read-only mapping once so construction, lookup, membership, get, and iteration all share one Unicode caseless key invariant.

The wrapper stores only canonical keys in a private copied dictionary and
inherits the remaining read-only behavior from `collections.abc.Mapping`.
Rejecting collisions during construction prevents two distinct input spellings
from silently overwriting one logical key.

## When to Use

Use this recipe for a fixed lookup table whose text keys should match across
case and canonical Unicode composition differences. It fits protocol labels or
imported reference data when locale-independent caseless matching is the
declared rule. Keep original spellings separately when they must be displayed,
and use schema-specific identifier validation for security-sensitive names.

## Implementation

```python
from collections.abc import Iterable, Iterator, Mapping
from types import MappingProxyType
from typing import Generic, TypeVar
from unicodedata import normalize


V = TypeVar("V")


class CanonicalKeyCollision(ValueError):
    pass


def canonical_caseless_key(text: str) -> str:
    if not isinstance(text, str):
        raise TypeError("key must be text")
    decomposed = normalize("NFD", text)
    return normalize("NFD", decomposed.casefold())


class CanonicalTextMapping(Mapping[str, V], Generic[V]):
    def __init__(
        self,
        items: Mapping[str, V] | Iterable[tuple[str, V]],
    ) -> None:
        values: dict[str, V] = {}
        pairs = items.items() if isinstance(items, Mapping) else items
        for pair in pairs:
            if not isinstance(pair, tuple) or len(pair) != 2:
                raise TypeError("items must contain key-value pairs")
            source_key, value = pair
            canonical = canonical_caseless_key(source_key)
            if canonical in values:
                raise CanonicalKeyCollision(
                    f"more than one input key canonicalizes to {canonical!r}",
                )
            values[canonical] = value
        self._values: Mapping[str, V] = MappingProxyType(values)

    def __getitem__(self, key: str) -> V:
        return self._values[canonical_caseless_key(key)]

    def __iter__(self) -> Iterator[str]:
        return iter(self._values)

    def __len__(self) -> int:
        return len(self._values)
```

## Example

```python
source = {"Straße": 7, "é": 8}
mapping = CanonicalTextMapping(source)
source.clear()

try:
    CanonicalTextMapping([("Straße", 1), ("STRASSE", 2)])
except CanonicalKeyCollision:
    collision_rejected = True
else:
    collision_rejected = False

try:
    mapping["new"] = 9
except TypeError:
    assignment_rejected = True
else:
    assignment_rejected = False

assert (
    mapping["STRASSE"],
    mapping.get("e\u0301"),
    "strasse" in mapping,
    tuple(mapping),
    collision_rejected,
    assignment_rejected,
) == (
    7,
    8,
    True,
    (canonical_caseless_key("Straße"), canonical_caseless_key("é")),
    True,
    True,
)
```

## Trade-offs and Limitations

Construction materializes all entries and rejects even repeated identical
source keys after canonicalization. Iteration exposes decomposed canonical keys,
not original spellings, while values are copied only by reference and may
remain mutable. The normalizer is locale-independent: it neither performs
language-aware collation nor catches visually confusable characters. Distinct
text can intentionally share a caseless key, so collision rejection may be too
strict for data that needs a multi-value or explicit winner policy.

## Related Snippets

<!-- catalog:related:start -->
- [Build a Canonical Unicode Caseless Comparison Key](../algorithms-data-structures/build-a-canonical-unicode-caseless-comparison-key.md)
<!-- catalog:related:end -->
