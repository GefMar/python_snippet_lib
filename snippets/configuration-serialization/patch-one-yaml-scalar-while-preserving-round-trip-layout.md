---
title: "Patch One YAML Scalar While Preserving Round-Trip Layout"
snippet_type: integration
use_cases:
  - configuration
  - serialization
  - validation
tested_python:
  - "3.14"
dependencies:
  - name: ruamel.yaml
    version: "0.19.1"
related:
  - get-nested-values-with-a-validated-dot-path.md
  - merge-nested-configuration-with-an-explicit-delete-sentinel.md
  - render-a-stable-unified-diff-for-nested-json-values.md
---

# Patch One YAML Scalar While Preserving Round-Trip Layout

## Idea and Problem

Update one existing scalar in a bounded YAML mapping while retaining ordinary comments, mapping order, and supported scalar style around that value.

The function accepts and returns text only. It uses `ruamel.yaml` round-trip
mode, requires an exact string path, and compares the current value with an
expected old passive scalar before mutation. Duplicate keys, extra documents,
anchors, aliases, and every explicit tag are rejected so the update cannot
quietly choose among ambiguous values or expand shared structures.

## When to Use

Use this integration at a trusted configuration-editing boundary when a small
YAML document must remain readable and one known scalar needs an optimistic
compare-and-set update. The expected old value makes concurrent or stale
assumptions visible rather than silently overwriting a changed setting.

Use a schema-aware configuration service when validation spans multiple
fields, and use a canonical serializer when byte stability matters. Parse and
write files outside this function with an atomic replacement strategy; no
filesystem path, stream, callback, environment lookup, or secret source is
accepted here.

## Implementation

```python
import io
import math

from ruamel.yaml import YAML
from ruamel.yaml.comments import CommentedMap, CommentedSeq
from ruamel.yaml.error import YAMLError
from ruamel.yaml.scalarstring import ScalarString
from ruamel.yaml.tokens import AliasToken, AnchorToken, TagToken


PassiveScalar = None | bool | int | float | str
_MAX_TEXT_CHARACTERS = 100_000
_MAX_TEXT_BYTES = 400_000
_MAX_OUTPUT_BYTES = 420_000
_MAX_TOKENS = 30_000
_MAX_NODES = 10_000
_MAX_DEPTH = 64
_MAX_PATH_PARTS = 16
_MAX_KEY_CHARACTERS = 256
_MAX_SCALAR_CHARACTERS = 4_096
_MAX_INTEGER = 2**63 - 1


def _utf8_size(value: str, *, field: str) -> int:
    try:
        return len(value.encode("utf-8"))
    except UnicodeEncodeError as error:
        raise ValueError(f"{field} must be valid UTF-8 text") from error


def _argument_scalar(value: object, *, field: str) -> PassiveScalar:
    if type(value) not in (type(None), bool, int, float, str):
        raise TypeError(f"{field} must be an exact passive scalar")
    if type(value) is int and abs(value) > _MAX_INTEGER:
        raise ValueError(f"{field} integer is outside the supported range")
    if type(value) is float and not math.isfinite(value):
        raise ValueError(f"{field} float must be finite")
    if type(value) is str:
        if len(value) > _MAX_SCALAR_CHARACTERS:
            raise ValueError(f"{field} text is outside the supported range")
        if _utf8_size(value, field=field) > _MAX_TEXT_BYTES:
            raise ValueError(f"{field} text is outside the supported range")
    return value


def _parsed_scalar(value: object) -> PassiveScalar:
    if value is None or type(value) is bool:
        return value
    if isinstance(value, int) and not isinstance(value, bool):
        normalized = int(value)
        if abs(normalized) <= _MAX_INTEGER:
            return normalized
    elif isinstance(value, float):
        normalized = float(value)
        if math.isfinite(normalized):
            return normalized
    elif isinstance(value, str):
        normalized = str(value)
        if (
            len(normalized) <= _MAX_SCALAR_CHARACTERS
            and _utf8_size(normalized, field="current scalar") <= _MAX_TEXT_BYTES
        ):
            return normalized
    raise TypeError("the selected YAML value is not a supported passive scalar")


def _path_parts(path: object) -> tuple[str, ...]:
    if type(path) is not tuple:
        raise TypeError("path must be an exact tuple")
    if not 1 <= len(path) <= _MAX_PATH_PARTS:
        raise ValueError("path length is outside the supported range")
    for part in path:
        if type(part) is not str:
            raise TypeError("path parts must be exact strings")
        if not part or len(part) > _MAX_KEY_CHARACTERS:
            raise ValueError("a path part is outside the supported range")
        if _utf8_size(part, field="path part") > _MAX_TEXT_BYTES:
            raise ValueError("a path part is outside the supported range")
    return path


def _validate_tree(root: CommentedMap) -> None:
    nodes: list[tuple[object, int]] = [(root, 0)]
    visited = 0
    while nodes:
        value, depth = nodes.pop()
        visited += 1
        if visited > _MAX_NODES:
            raise ValueError("YAML node count exceeds the supported limit")
        if depth > _MAX_DEPTH:
            raise ValueError("YAML nesting exceeds the supported limit")

        if type(value) is CommentedMap:
            if value.merge:
                raise ValueError("YAML merge keys are forbidden")
            for key, child in value.items():
                if not isinstance(key, str):
                    raise TypeError("all YAML mapping keys must be text")
                key_text = str(key)
                if not key_text or len(key_text) > _MAX_KEY_CHARACTERS:
                    raise ValueError("a YAML mapping key is outside the supported range")
                if _utf8_size(key_text, field="mapping key") > _MAX_TEXT_BYTES:
                    raise ValueError("a YAML mapping key is outside the supported range")
                nodes.append((child, depth + 1))
        elif type(value) is CommentedSeq:
            nodes.extend((child, depth + 1) for child in value)
        else:
            _parsed_scalar(value)


def patch_yaml_scalar(
    text: str,
    *,
    path: tuple[str, ...],
    expected_old: PassiveScalar,
    replacement: PassiveScalar,
) -> str:
    if type(text) is not str:
        raise TypeError("text must be an exact string")
    if len(text) > _MAX_TEXT_CHARACTERS:
        raise ValueError("YAML text exceeds the supported character limit")
    if _utf8_size(text, field="text") > _MAX_TEXT_BYTES:
        raise ValueError("YAML text exceeds the supported byte limit")

    parts = _path_parts(path)
    expected = _argument_scalar(expected_old, field="expected_old")
    new_value = _argument_scalar(replacement, field="replacement")

    yaml = YAML(typ="rt")
    yaml.allow_duplicate_keys = False
    yaml.max_depth = _MAX_DEPTH
    yaml.preserve_quotes = True
    yaml.width = _MAX_TEXT_CHARACTERS

    try:
        token_count = 0
        for token in yaml.scan(text):
            token_count += 1
            if token_count > _MAX_TOKENS:
                raise ValueError("YAML token count exceeds the supported limit")
            if isinstance(token, (AnchorToken, AliasToken, TagToken)):
                raise ValueError("anchors, aliases, and explicit tags are forbidden")

        documents = yaml.load_all(text)
        document = next(documents)
        try:
            next(documents)
        except StopIteration:
            pass
        else:
            raise ValueError("exactly one YAML document is required")
    except StopIteration as error:
        raise ValueError("exactly one YAML document is required") from error
    except (YAMLError, RecursionError) as error:
        raise ValueError("invalid or unsupported YAML") from error

    if type(document) is not CommentedMap:
        raise TypeError("the YAML document root must be a mapping")
    _validate_tree(document)

    parent = document
    for part in parts[:-1]:
        if part not in parent:
            raise KeyError(f"missing YAML path part: {part!r}")
        child = parent[part]
        if type(child) is not CommentedMap:
            raise TypeError(f"YAML path part is not a mapping: {part!r}")
        parent = child

    leaf = parts[-1]
    if leaf not in parent:
        raise KeyError(f"missing YAML path part: {leaf!r}")
    current_value = parent[leaf]
    current = _parsed_scalar(current_value)
    if type(current) is not type(expected) or current != expected:
        raise ValueError("current YAML scalar does not match expected_old")

    if isinstance(current_value, ScalarString) and type(new_value) is str:
        parent[leaf] = type(current_value)(new_value)
    else:
        parent[leaf] = new_value

    output = io.StringIO()
    yaml.dump(document, output)
    result = output.getvalue()
    if _utf8_size(result, field="output") > _MAX_OUTPUT_BYTES:
        raise ValueError("patched YAML exceeds the supported output limit")
    return result
```

## Example

```python
source = (
    "# deployment settings\n"
    "service:\n"
    '  mode: "safe" # reviewed\n'
    "  retries: 3\n"
)
patched = patch_yaml_scalar(
    source,
    path=("service", "mode"),
    expected_old="safe",
    replacement="strict",
)

invalid_documents = (
    "a: 1\na: 2\n",
    "a: &value 1\nb: *value\n",
    "a: !!str 1\n",
    "a: 1\nmerged:\n  <<: {inherited: 2}\n",
    "a: 1\ncreated: 2026-07-27\n",
    "---\na: 1\n---\na: 2\n",
)
rejected = 0
for invalid in invalid_documents:
    try:
        patch_yaml_scalar(
            invalid,
            path=("a",),
            expected_old=1,
            replacement=2,
        )
    except (TypeError, ValueError):
        rejected += 1

assert patched == (
    "# deployment settings\n"
    "service:\n"
    '  mode: "strict" # reviewed\n'
    "  retries: 3\n"
)
assert rejected == len(invalid_documents)
```

## Trade-offs and Limitations

Round-trip output is deliberately not byte-identical serialization. Comments,
key order, and quoted-string style are retained in ordinary cases, but line
wrapping, directives, indentation, numeric spelling, final newlines, and other
presentation details can be normalized by the pinned library. Replacing one
scalar with another scalar type can also require a different YAML spelling.

The parser and dumper still use memory proportional to the whole document.
Character, byte, token, node, depth, path, key, and scalar bounds constrain
that cost, while the strict string-key and passive-scalar tree rules intentionally
exclude some valid YAML. The expected-old check is not a file lock; callers
that persist the result still need atomic I/O and concurrency control. Schema
validation, secret handling, encryption, merge keys, references, tags, and
multi-document streams are outside this helper.

## Related Snippets

<!-- catalog:related:start -->
- [Get Nested Values with a Validated Dot Path](get-nested-values-with-a-validated-dot-path.md)
- [Merge Nested Configuration with an Explicit Delete Sentinel](merge-nested-configuration-with-an-explicit-delete-sentinel.md)
- [Render a Stable Unified Diff for Nested JSON Values](render-a-stable-unified-diff-for-nested-json-values.md)
<!-- catalog:related:end -->
