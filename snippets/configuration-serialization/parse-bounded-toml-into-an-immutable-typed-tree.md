---
title: "Parse Bounded TOML into an Immutable Typed Tree"
snippet_type: recipe
use_cases:
  - configuration
  - parsing
  - serialization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - parse-a-bounded-ini-document-under-a-closed-no-interpolation-schema.md
  - parse-bounded-utf-8-json-without-duplicate-object-names-or-non-finite-numbers.md
  - normalize-a-bounded-json-copy-before-standard-schema-validation.md
---

# Parse Bounded TOML into an Immutable Typed Tree

## Idea and Problem

Parse a size-capped TOML document while preserving typed scalars, exact decimal floats, and an immutable canonical table representation.

`tomllib` handles TOML grammar, duplicate definitions, dates, times, arrays,
and tables. A custom `parse_float` hook produces `Decimal` rather than binary
floating point and rejects non-finite or excessively large decimal profiles.

A second pass validates signed integers and structural budgets. Lists become
tuples, while dictionaries become `FrozenTable` values with lexically sorted
keys, so no mutable parser container escapes and equivalent table declaration
order does not change equality.

## When to Use

Use this boundary for small trusted-shape configuration documents when TOML's
typed dates, times, arrays, and tables are useful but consumers should receive
an immutable value graph. Exact decimal parsing is helpful when configuration
numbers must avoid an initial binary-float conversion.

Apply a separate application schema after parsing. Use a TOML editing library
when comments, whitespace, key spelling, or round-trip layout must be
preserved; `tomllib` is intentionally read-only.

## Implementation

```python
import tomllib
from dataclasses import dataclass
from datetime import date, datetime, time
from decimal import Decimal

_MAX_INPUT_BYTES = 65_536
_MAX_DEPTH = 32
_MAX_NODES = 5_000
_MAX_TEXT_BYTES = 1 << 20
_MAX_FLOAT_SOURCE = 256
_MAX_FLOAT_DIGITS = 128
_MIN_FLOAT_ADJUSTED_EXPONENT = -1_000
_MAX_FLOAT_ADJUSTED_EXPONENT = 1_000
_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1

type FrozenTomlValue = (
    str
    | int
    | bool
    | Decimal
    | date
    | time
    | datetime
    | tuple[FrozenTomlValue, ...]
    | FrozenTable
)


@dataclass(frozen=True, slots=True)
class FrozenTable:
    items: tuple[tuple[str, FrozenTomlValue], ...]

    def __getitem__(self, key: str) -> FrozenTomlValue:
        if type(key) is not str:
            raise TypeError("table key must be an exact string")
        for item_key, value in self.items:
            if item_key == key:
                return value
        raise KeyError(key)


def _validate_decimal(value: Decimal) -> Decimal:
    if not value.is_finite():
        raise ValueError("TOML float must be finite")
    if len(value.as_tuple().digits) > _MAX_FLOAT_DIGITS:
        raise ValueError("TOML float has too many coefficient digits")
    if not (
        _MIN_FLOAT_ADJUSTED_EXPONENT
        <= value.adjusted()
        <= _MAX_FLOAT_ADJUSTED_EXPONENT
    ):
        raise ValueError("TOML float exponent is outside the supported range")
    return value


def _parse_decimal(source: str) -> Decimal:
    if type(source) is not str:
        raise TypeError("TOML float source must be an exact string")
    if len(source) > _MAX_FLOAT_SOURCE:
        raise ValueError("TOML float source exceeds the supported length")
    return _validate_decimal(Decimal(source))


def _freeze_toml_tree(root: object) -> FrozenTable:
    nodes = 0
    text_bytes = 0

    def claim_text(kind: str, value: str) -> None:
        nonlocal text_bytes

        try:
            encoded_length = len(value.encode("utf-8"))
        except UnicodeEncodeError as error:
            raise ValueError(f"TOML {kind} contains a Unicode surrogate") from error
        text_bytes += encoded_length
        if text_bytes > _MAX_TEXT_BYTES:
            raise ValueError("TOML key and string text exceeds the supported limit")

    def freeze(value: object, depth: int) -> FrozenTomlValue:
        nonlocal nodes

        if depth > _MAX_DEPTH:
            raise ValueError("TOML tree exceeds the supported depth")
        nodes += 1
        if nodes > _MAX_NODES:
            raise ValueError("TOML tree exceeds the supported node count")

        if type(value) is str:
            claim_text("string", value)
            return value
        if type(value) is bool:
            return value
        if type(value) is int:
            if not _MIN_INT64 <= value <= _MAX_INT64:
                raise ValueError("TOML integer is outside the signed 64-bit range")
            return value
        if type(value) is Decimal:
            return _validate_decimal(value)
        if type(value) in (date, time, datetime):
            return value
        if type(value) is list:
            return tuple(freeze(item, depth + 1) for item in value)
        if type(value) is dict:
            frozen_items: list[tuple[str, FrozenTomlValue]] = []
            for key in sorted(value):
                if type(key) is not str:
                    raise TypeError("TOML table keys must be exact strings")
                claim_text("key", key)
                frozen_items.append((key, freeze(value[key], depth + 1)))
            return FrozenTable(tuple(frozen_items))
        raise TypeError("parsed TOML contains a value outside the closed model")

    frozen = freeze(root, 1)
    if type(frozen) is not FrozenTable:
        raise RuntimeError("tomllib did not return a top-level table")
    return frozen


def parse_bounded_toml(document: bytes) -> FrozenTable:
    """Parse exact UTF-8 TOML bytes and return an immutable typed tree."""
    if type(document) is not bytes:
        raise TypeError("document must be exact bytes")
    if len(document) > _MAX_INPUT_BYTES:
        raise ValueError("TOML document exceeds the supported byte size")
    if document.startswith(b"\xef\xbb\xbf"):
        raise ValueError("TOML document must not start with a UTF-8 byte-order mark")
    try:
        text = document.decode("utf-8")
    except UnicodeDecodeError as error:
        raise ValueError("TOML document must be valid UTF-8") from error

    parsed = tomllib.loads(text, parse_float=_parse_decimal)
    return _freeze_toml_tree(parsed)
```

## Example

```python
document = b"""
title = "bounded service"
released = 2026-07-29

[service]
price = 1_000.50
enabled = true
windows = [08:30:00, 17:45:00]
"""
configuration = parse_bounded_toml(document)
service = configuration["service"]

assert configuration["title"] == "bounded service"
assert configuration["released"] == date(2026, 7, 29)
assert type(service) is FrozenTable
assert service["price"] == Decimal("1000.50")
assert service["enabled"] is True
assert service["windows"] == (time(8, 30), time(17, 45))
assert tuple(key for key, _ in service.items) == (
    "enabled",
    "price",
    "windows",
)

first = parse_bounded_toml(b"point = { z = 2, a = 1 }")
second = parse_bounded_toml(b"point = { a = 1, z = 2 }")
assert first == second
assert parse_bounded_toml(b"") == FrozenTable(())

try:
    parse_bounded_toml(b"value = inf")
except ValueError:
    non_finite_rejected = True
else:
    non_finite_rejected = False

assert non_finite_rejected
```

## Trade-offs and Limitations

For `B` input bytes and `N` bounded parsed nodes and text, parsing and freezing
use `O(B + N log N)` work in the worst case because every table sorts its
keys. Live parsing plus immutable output uses `O(B + N)` state. The byte cap is
enforced before `tomllib`; depth, node, expanded text, integer, and typed-value
limits are necessarily checked after grammar parsing has constructed a tree.

Table keys are sorted and arrays preserve source order. This creates a stable
logical value, not a canonical TOML text: comments, whitespace, numeric
spelling, table declaration style, key order, source locations, and provenance
are intentionally discarded. Dates, times, and datetimes remain distinct
standard-library values.

The function does not write TOML, preserve layout, apply an application schema,
interpolate values, process includes, merge environment variables, resolve
paths, authenticate configuration, conceal secrets, or make untrusted
configuration safe for arbitrary downstream behavior.

## Related Snippets

<!-- catalog:related:start -->
- [Parse a Bounded INI Document Under a Closed No-Interpolation Schema](parse-a-bounded-ini-document-under-a-closed-no-interpolation-schema.md)
- [Parse Bounded UTF-8 JSON Without Duplicate Object Names or Non-Finite Numbers](parse-bounded-utf-8-json-without-duplicate-object-names-or-non-finite-numbers.md)
- [Normalize a Bounded JSON Copy Before Standard Schema Validation](normalize-a-bounded-json-copy-before-standard-schema-validation.md)
<!-- catalog:related:end -->
