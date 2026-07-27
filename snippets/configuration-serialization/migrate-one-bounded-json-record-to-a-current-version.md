---
title: "Migrate One Bounded JSON Record to a Current Version"
snippet_type: recipe
use_cases:
  - data-transformation
  - serialization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - fingerprint-a-set-like-json-array-deterministically.md
  - render-a-stable-unified-diff-for-nested-json-values.md
---

# Migrate One Bounded JSON Record to a Current Version

## Idea and Problem

Decode one size-capped JSON record under either of two closed schemas and return one frozen current representation.

The decoder rejects malformed UTF-8 and JSON, duplicate object keys,
non-finite numbers, excessive structure, unknown versions, and every value
outside the exact v1 or v2 schema. A v1 value follows one deterministic path
to v2 fields, while source-version metadata records whether migration occurred.

## When to Use

Use this recipe when a small in-memory JSON document has one legacy shape and
one current shape, and both can be specified completely. Freeze the field
names, limits, and canonical encoder as part of the format contract. Use a
dedicated schema library when the document is large, deeply nested, or has
many independently evolving versions.

## Implementation

```python
import json
import math
import re
from dataclasses import dataclass


_NAME = re.compile(r"[a-z][a-z0-9-]{0,31}\Z", flags=re.ASCII)
_STATUSES = frozenset({"enabled", "disabled"})
_MAX_INPUT_BYTES = 32 * 1024
_MAX_NODES = 64
_MAX_DEPTH = 8
_MAX_ALIASES = 8
_MAX_INTEGER_DIGITS = 20
_V1_KEYS = frozenset({"version", "name", "enabled", "level"})
_V2_KEYS = frozenset({"version", "name", "status", "level", "aliases"})


class RecordFormatError(ValueError):
    pass


class UnsupportedRecordVersion(RecordFormatError):
    pass


def _require_name(value: object, *, context: str) -> str:
    if type(value) is not str or _NAME.fullmatch(value) is None:
        raise RecordFormatError(f"{context} must be a canonical ASCII name")
    return value


def _require_status(value: object) -> str:
    if type(value) is not str or value not in _STATUSES:
        raise RecordFormatError("status must be 'enabled' or 'disabled'")
    return value


def _require_level(value: object) -> int:
    if type(value) is not int or not 0 <= value <= 100:
        raise RecordFormatError("level must be an integer from 0 through 100")
    return value


def _require_aliases(value: object, *, name: str, container_type: type) -> tuple[str, ...]:
    if type(value) is not container_type:
        label = "array" if container_type is list else "tuple"
        raise RecordFormatError(f"aliases must be a {label}")
    if len(value) > _MAX_ALIASES:
        raise RecordFormatError(f"aliases cannot exceed {_MAX_ALIASES} entries")

    aliases: list[str] = []
    seen: set[str] = set()
    for index, alias in enumerate(value):
        valid_alias = _require_name(alias, context=f"aliases[{index}]")
        if valid_alias == name or valid_alias in seen:
            raise RecordFormatError("aliases must be unique and differ from name")
        aliases.append(valid_alias)
        seen.add(valid_alias)
    return tuple(aliases)


def _validate_current_values(
    *,
    name: object,
    status: object,
    level: object,
    aliases: object,
    source_version: object,
) -> None:
    valid_name = _require_name(name, context="name")
    _require_status(status)
    _require_level(level)
    _require_aliases(aliases, name=valid_name, container_type=tuple)
    if type(source_version) is not int or source_version not in (1, 2):
        raise RecordFormatError("source_version must be 1 or 2")


@dataclass(frozen=True, slots=True)
class CurrentRecord:
    name: str
    status: str
    level: int
    aliases: tuple[str, ...]
    source_version: int

    def __post_init__(self) -> None:
        _validate_current_values(
            name=self.name,
            status=self.status,
            level=self.level,
            aliases=self.aliases,
            source_version=self.source_version,
        )


def _unique_object(pairs: list[tuple[str, object]]) -> dict[str, object]:
    result: dict[str, object] = {}
    for key, value in pairs:
        if key in result:
            raise RecordFormatError(f"duplicate object key: {key!r}")
        result[key] = value
    return result


def _reject_constant(text: str) -> object:
    raise RecordFormatError(f"non-finite JSON constant is not allowed: {text}")


def _parse_float(text: str) -> float:
    value = float(text)
    if not math.isfinite(value):
        raise RecordFormatError("non-finite JSON numbers are not allowed")
    return value


def _parse_integer(text: str) -> int:
    digits = text.removeprefix("-")
    if len(digits) > _MAX_INTEGER_DIGITS:
        raise RecordFormatError("JSON integer is too large")
    return int(text)


def _validate_tree_bounds(root: object) -> None:
    nodes = 0
    stack = [(root, 0)]
    while stack:
        value, depth = stack.pop()
        nodes += 1
        if nodes > _MAX_NODES:
            raise RecordFormatError(f"JSON cannot exceed {_MAX_NODES} value nodes")
        if depth > _MAX_DEPTH:
            raise RecordFormatError(f"JSON cannot exceed depth {_MAX_DEPTH}")
        if type(value) is dict:
            stack.extend((child, depth + 1) for child in value.values())
        elif type(value) is list:
            stack.extend((child, depth + 1) for child in value)


def _require_keys(record: dict[str, object], expected: frozenset[str]) -> None:
    if record.keys() != expected:
        fields = ", ".join(sorted(expected))
        raise RecordFormatError(f"record must contain exactly these fields: {fields}")


def migrate_json_record(payload: bytes) -> CurrentRecord:
    if type(payload) is not bytes:
        raise TypeError("payload must be bytes")
    if not 1 <= len(payload) <= _MAX_INPUT_BYTES:
        raise RecordFormatError(
            f"payload must contain between 1 and {_MAX_INPUT_BYTES} bytes"
        )
    try:
        text = payload.decode("utf-8", errors="strict")
    except UnicodeDecodeError as error:
        raise RecordFormatError("payload is not valid UTF-8") from error

    try:
        document = json.loads(
            text,
            object_pairs_hook=_unique_object,
            parse_constant=_reject_constant,
            parse_float=_parse_float,
            parse_int=_parse_integer,
        )
    except RecordFormatError:
        raise
    except (json.JSONDecodeError, RecursionError, ValueError) as error:
        raise RecordFormatError("payload is not valid JSON") from error

    _validate_tree_bounds(document)
    if type(document) is not dict:
        raise RecordFormatError("record must be a JSON object")

    version = document.get("version")
    if type(version) is not int:
        raise RecordFormatError("version must be an integer")
    if version == 1:
        _require_keys(document, _V1_KEYS)
        name = _require_name(document["name"], context="name")
        enabled = document["enabled"]
        if type(enabled) is not bool:
            raise RecordFormatError("enabled must be a boolean")
        return CurrentRecord(
            name=name,
            status="enabled" if enabled else "disabled",
            level=_require_level(document["level"]),
            aliases=(),
            source_version=1,
        )
    if version != 2:
        raise UnsupportedRecordVersion(f"unsupported record version: {version}")

    _require_keys(document, _V2_KEYS)
    name = _require_name(document["name"], context="name")
    aliases = _require_aliases(document["aliases"], name=name, container_type=list)
    return CurrentRecord(
        name=name,
        status=_require_status(document["status"]),
        level=_require_level(document["level"]),
        aliases=aliases,
        source_version=2,
    )


def encode_current_record(record: CurrentRecord) -> bytes:
    if type(record) is not CurrentRecord:
        raise TypeError("record must be a CurrentRecord")
    _validate_current_values(
        name=record.name,
        status=record.status,
        level=record.level,
        aliases=record.aliases,
        source_version=record.source_version,
    )
    document = {
        "version": 2,
        "name": record.name,
        "status": record.status,
        "level": record.level,
        "aliases": list(record.aliases),
    }
    encoded = json.dumps(
        document,
        ensure_ascii=True,
        allow_nan=False,
        sort_keys=True,
        separators=(",", ":"),
    ).encode("utf-8")
    if len(encoded) > _MAX_INPUT_BYTES:
        raise RecordFormatError("canonical record exceeds the input size limit")
    return encoded
```

## Example

```python
legacy = b'{"level":4,"enabled":true,"name":"cedar","version":1}'
migrated = migrate_json_record(legacy)
canonical = encode_current_record(migrated)
reloaded = migrate_json_record(canonical)

invalid_payloads = (
    b'{"version":1,"version":2,"name":"cedar","status":"enabled",'
    b'"level":4,"aliases":[]}',
    b'{"version":3,"name":"cedar","status":"enabled","level":4,'
    b'"aliases":[]}',
)
rejected = 0
for invalid_payload in invalid_payloads:
    try:
        migrate_json_record(invalid_payload)
    except RecordFormatError:
        rejected += 1

assert (
    migrated,
    canonical,
    reloaded,
    encode_current_record(reloaded),
    rejected,
) == (
    CurrentRecord("cedar", "enabled", 4, (), 1),
    b'{"aliases":[],"level":4,"name":"cedar","status":"enabled","version":2}',
    CurrentRecord("cedar", "enabled", 4, (), 2),
    canonical,
    2,
)
```

## Trade-offs and Limitations

This format has exactly one direct v1-to-v2 migration; adding another version
requires an explicit code and contract change. Decoding materializes the whole
document before applying the node and depth limits, although the byte cap
constrains input size. The canonical form is a local convention based on
sorted keys, ASCII escapes, compact separators, and Python's integer encoding,
not a general cross-language canonicalization standard. Source-version
metadata is deliberately omitted from encoded v2 bytes. The recipe performs
no I/O, identifier generation, encryption, authentication, or clock handling.

## Related Snippets

<!-- catalog:related:start -->
- [Fingerprint a Set-Like JSON Array Deterministically](fingerprint-a-set-like-json-array-deterministically.md)
- [Render a Stable Unified Diff for Nested JSON Values](render-a-stable-unified-diff-for-nested-json-values.md)
<!-- catalog:related:end -->
