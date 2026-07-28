---
title: "Resolve a Bounded Configuration Fragment Graph"
snippet_type: algorithm
use_cases:
  - configuration
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - merge-nested-mappings-without-mutating-inputs.md
  - normalize-a-bounded-json-copy-before-standard-schema-validation.md
  - ../algorithms-data-structures/resolve-stable-ordering-constraints-with-topological-sort.md
---

# Resolve a Bounded Configuration Fragment Graph

## Idea and Problem

Resolve a closed graph of already-parsed configuration fragments only after validating the entire graph and a conservative expansion-work budget.

Every fragment has a normalized name, a declared-order tuple of included
fragment names, and a local mapping body. Includes are resolved recursively and
deep-merged from left to right: a later include overrides an earlier include,
then the local body overrides every include. Two colliding mappings merge
recursively; every scalar or array replaces the earlier value whole.

Unlike an ordinary two-mapping merge, graph shape is part of the trust
boundary. The preflight rejects unknown references, cycles, excessive reference
depth, and a diamond graph whose repeated logical expansion exceeds a fixed
node-work budget before constructing a result.

## When to Use

Use this algorithm for a small, closed, in-memory configuration graph after a
parser has produced exact JSON-like `dict` and `list` values. It fits reusable
defaults and ordered overlays when references and all fragment bodies arrive
together and deterministic precedence matters.

Keep parsing, files, paths, URLs, environment access, interpolation, callbacks,
and schema-specific coercion outside this boundary. Use a schema validator as a
separate step when field names, required values, or domain constraints also
need validation.

## Implementation

```python
import math
import re
import unicodedata
from collections.abc import Mapping
from dataclasses import dataclass
from types import MappingProxyType
from typing import TypeAlias

JSONValue: TypeAlias = None | bool | int | float | str | list["JSONValue"] | dict[str, "JSONValue"]
FrozenJSONValue: TypeAlias = (
    None
    | bool
    | int
    | float
    | str
    | tuple["FrozenJSONValue", ...]
    | Mapping[str, "FrozenJSONValue"]
)

_NAME = re.compile(r"[a-z][a-z0-9_-]{0,63}", re.ASCII).fullmatch
_MAX_FRAGMENTS = 64
_MAX_INCLUDES = 8
_MAX_REFERENCE_DEPTH = 16
_MAX_JSON_DEPTH = 12
_MAX_TOTAL_NODES = 2_048
_MAX_CONTAINER_ITEMS = 64
_MAX_STRING_BYTES = 4_096
_MAX_TOTAL_TEXT_BYTES = 128 * 1_024
_MAX_NAME_BYTES = 64
_MIN_INTEGER = -(1 << 63)
_MAX_INTEGER = (1 << 63) - 1
_MAX_EXPANSION_WORK = 65_536


@dataclass(frozen=True, slots=True)
class Fragment:
    name: str
    includes: tuple[str, ...]
    body: dict[str, JSONValue]


@dataclass(frozen=True, slots=True)
class _CheckedFragment:
    includes: tuple[str, ...]
    body: dict[str, JSONValue]
    body_nodes: int


def _preflight(
    fragments: object,
    root: object,
) -> tuple[dict[str, _CheckedFragment], str]:
    if type(fragments) is not tuple:
        raise TypeError("fragments must be an exact tuple")
    if not 1 <= len(fragments) <= _MAX_FRAGMENTS:
        raise ValueError("fragment count must be from 1 through 64")

    total_nodes = 0
    total_text_bytes = 0
    mutable_ids: set[int] = set()

    def count_text(text: str) -> int:
        nonlocal total_text_bytes
        if len(text) > _MAX_STRING_BYTES:
            raise ValueError("a string exceeds the 4 KiB limit")
        try:
            size = len(text.encode("utf-8"))
        except UnicodeEncodeError:
            raise ValueError("text must be valid UTF-8") from None
        if size > _MAX_STRING_BYTES:
            raise ValueError("a string exceeds the 4 KiB limit")
        total_text_bytes += size
        if total_text_bytes > _MAX_TOTAL_TEXT_BYTES:
            raise ValueError("aggregate text exceeds the 128 KiB limit")
        return size

    def canonical_name(value: object) -> str:
        if type(value) is not str:
            raise TypeError("fragment and reference names must be exact strings")
        if count_text(value) > _MAX_NAME_BYTES:
            raise ValueError("a fragment or reference name exceeds 64 bytes")
        normalized = unicodedata.normalize("NFKC", value).casefold()
        if _NAME(normalized) is None:
            raise ValueError("a normalized fragment or reference name is invalid")
        return normalized

    def copy_json(value: object, depth: int) -> JSONValue:
        nonlocal total_nodes
        total_nodes += 1
        if total_nodes > _MAX_TOTAL_NODES:
            raise ValueError("fragment bodies exceed the 2048-node limit")
        if depth > _MAX_JSON_DEPTH:
            raise ValueError("a fragment body exceeds JSON depth 12")

        value_type = type(value)
        if value is None or value_type is bool:
            return value
        if value_type is int:
            if not _MIN_INTEGER <= value <= _MAX_INTEGER:
                raise ValueError("an integer is outside the signed 64-bit range")
            return value
        if value_type is float:
            if not math.isfinite(value):
                raise ValueError("a float must be finite")
            return value
        if value_type is str:
            count_text(value)
            return value
        if value_type not in (list, dict):
            raise TypeError("fragment bodies must contain only exact JSON-like types")

        identity = id(value)
        if identity in mutable_ids:
            raise ValueError("a mutable list or dict object is reused")
        mutable_ids.add(identity)
        if len(value) > _MAX_CONTAINER_ITEMS:
            raise ValueError("a list or mapping exceeds 64 items")

        if value_type is list:
            return [copy_json(item, depth + 1) for item in value]

        copied: dict[str, JSONValue] = {}
        for key, item in value.items():
            if type(key) is not str:
                raise TypeError("mapping keys must be exact strings")
            count_text(key)
            copied[key] = copy_json(item, depth + 1)
        return copied

    checked: dict[str, _CheckedFragment] = {}
    for fragment in fragments:
        if type(fragment) is not Fragment:
            raise TypeError("fragments must contain exact Fragment records")
        name = canonical_name(fragment.name)
        if name in checked:
            raise ValueError("fragment names collide after normalization")
        if type(fragment.includes) is not tuple:
            raise TypeError("fragment includes must be an exact tuple")
        if len(fragment.includes) > _MAX_INCLUDES:
            raise ValueError("a fragment includes more than 8 fragments")
        includes = tuple(canonical_name(item) for item in fragment.includes)
        if len(set(includes)) != len(includes):
            raise ValueError("a fragment repeats an include after normalization")
        if type(fragment.body) is not dict:
            raise TypeError("a fragment body must be an exact dict")
        before = total_nodes
        body = copy_json(fragment.body, 0)
        checked[name] = _CheckedFragment(includes, body, total_nodes - before)

    root_name = canonical_name(root)
    if root_name not in checked:
        raise ValueError("root names an unknown fragment")
    for fragment in checked.values():
        if any(name not in checked for name in fragment.includes):
            raise ValueError("an include names an unknown fragment")

    state: dict[str, int] = {}
    depths: dict[str, int] = {}
    sizes: dict[str, int] = {}
    work: dict[str, int] = {}

    def inspect(name: str) -> tuple[int, int, int]:
        if state.get(name) == 1:
            raise ValueError("fragment includes contain a cycle")
        if state.get(name) == 2:
            return depths[name], sizes[name], work[name]
        state[name] = 1
        fragment = checked[name]
        depth = 1
        expanded_size = fragment.body_nodes
        expanded_work = fragment.body_nodes
        for dependency in fragment.includes:
            child_depth, child_size, child_work = inspect(dependency)
            depth = max(depth, child_depth + 1)
            expanded_size += child_size
            expanded_work += child_work + child_size
            if expanded_size > _MAX_EXPANSION_WORK or expanded_work > _MAX_EXPANSION_WORK:
                raise ValueError("fragment expansion exceeds the 65536-node work limit")
        if depth > _MAX_REFERENCE_DEPTH:
            raise ValueError("fragment references exceed depth 16")
        state[name] = 2
        depths[name] = depth
        sizes[name] = expanded_size
        work[name] = expanded_work
        return depth, expanded_size, expanded_work

    for name in checked:
        inspect(name)
    return checked, root_name


def _clone_json(value: JSONValue) -> JSONValue:
    if type(value) is dict:
        return {key: _clone_json(item) for key, item in value.items()}
    if type(value) is list:
        return [_clone_json(item) for item in value]
    return value


def _merge_into(target: dict[str, JSONValue], overlay: dict[str, JSONValue]) -> None:
    for key, overlay_value in overlay.items():
        current = target.get(key)
        if type(current) is dict and type(overlay_value) is dict:
            _merge_into(current, overlay_value)
        else:
            target[key] = _clone_json(overlay_value)


def _freeze(value: JSONValue) -> FrozenJSONValue:
    if type(value) is dict:
        return MappingProxyType({key: _freeze(item) for key, item in value.items()})
    if type(value) is list:
        return tuple(_freeze(item) for item in value)
    return value


def resolve_fragment_graph(
    fragments: tuple[Fragment, ...],
    root: str,
) -> Mapping[str, FrozenJSONValue]:
    checked, root_name = _preflight(fragments, root)
    cache: dict[str, dict[str, JSONValue]] = {}

    def resolve(name: str) -> dict[str, JSONValue]:
        if name in cache:
            return cache[name]
        merged: dict[str, JSONValue] = {}
        fragment = checked[name]
        for dependency in fragment.includes:
            _merge_into(merged, resolve(dependency))
        _merge_into(merged, fragment.body)
        cache[name] = merged
        return merged

    frozen = _freeze(resolve(root_name))
    assert isinstance(frozen, Mapping)
    return frozen
```

## Example

```python
fragments = (
    Fragment(
        "defaults",
        (),
        {"service": {"host": "old", "ports": [8000]}, "mode": "safe"},
    ),
    Fragment("region", ("defaults",), {"service": {"host": "eu"}}),
    Fragment("diagnostics", (), {"service": {"trace": False}, "mode": ["debug"]}),
    Fragment(
        "application",
        ("region", "diagnostics"),
        {"service": {"trace": True}},
    ),
)

resolved = resolve_fragment_graph(fragments, "APPLICATION")

assert dict(resolved["service"]) == {
    "host": "eu",
    "ports": (8000,),
    "trace": True,
}
assert resolved["mode"] == ("debug",)
assert fragments[0].body["service"]["ports"] == [8000]

try:
    resolved["service"]["trace"] = False
except TypeError:
    pass
else:
    raise AssertionError("the resolved graph must be immutable")

assert resolved["service"]["trace"] is True
```

## Trade-offs and Limitations

Preflight accepts 1 to 64 exact `Fragment` records, at most eight includes per
fragment, reference depth at most 16, and exact JSON-like bodies with depth at
most 12. Bodies may contain 2,048 value nodes total; every list or mapping may
hold 64 items; every string is limited to 4 KiB of UTF-8; aggregate names,
references, keys, and string values are limited to 128 KiB. Integers use the
signed 64-bit range. Names are NFKC-normalized and case-folded into conservative
ASCII identifiers of at most 64 bytes, and normalized collisions are errors.

The 65,536-node work calculation conservatively charges each include occurrence
for both recursive construction and merging. It is evaluated for every
fragment, so repeated paths in a diamond cannot hide behind the resolver's
cache. Exact `dict` and `list` identities may occur only once across all bodies;
this rejects cycles and intentional sharing as well as accidental aliases, but
makes node accounting and ownership unambiguous. Any invalid exact type or
shape raises `TypeError`; invalid names, bounds, numeric values, aliases, or
graph structure raise `ValueError`. Resolution starts only after the complete
preflight succeeds, and no partial result is returned.

The result is detached from mutable inputs. Private dictionaries are exposed
through read-only mapping proxies and arrays become tuples recursively. It is
therefore immutable but not necessarily hashable. Standard JSON encoders do not
accept mapping proxies directly, so serialization requires an explicit thawing
step; some consumers also require arrays to be exact lists. Key order follows
first insertion, and replacement does not move an existing key. The algorithm
records no provenance and supports no deletion marker, list concatenation, lazy
lookup, concurrent input mutation, or schema-aware conflict policy.

## Related Snippets

<!-- catalog:related:start -->
- [Merge Nested Mappings Without Mutating Inputs](merge-nested-mappings-without-mutating-inputs.md)
- [Normalize a Bounded JSON Copy Before Standard Schema Validation](normalize-a-bounded-json-copy-before-standard-schema-validation.md)
- [Resolve Stable Ordering Constraints with Topological Sort](../algorithms-data-structures/resolve-stable-ordering-constraints-with-topological-sort.md)
<!-- catalog:related:end -->
