---
title: "Resolve Incoming Configuration with Last-Known-Good Values"
snippet_type: pattern
use_cases:
  - configuration
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../storage-databases/replace-a-file-atomically-with-a-sibling-temporary-file.md
  - ../configuration-serialization/merge-nested-mappings-without-mutating-inputs.md
  - ../configuration-serialization/merge-nested-configuration-with-an-explicit-delete-sentinel.md
---

# Resolve Incoming Configuration with Last-Known-Good Values

## Idea and Problem

Resolve each incoming configuration entry independently while making every same-key fallback and rejection visible to the caller.

A valid incoming value wins. When validation raises the one declared failure
type, a cached value for that key is revalidated before use; otherwise the key
is reported as rejected. Keeping this step pure separates fallback policy from
file I/O, freshness decisions, and activation side effects.

## When to Use

Use this pattern when an incoming mapping is a complete desired key set and one
bad entry should not invalidate unrelated entries. The cached mapping must be a
candidate fallback, not trusted indefinitely: validate it again and let the
caller apply age, revocation, or version rules. Use transaction-wide validation
instead when entries have cross-key invariants and cannot be activated
independently.

## Implementation

```python
from collections.abc import Callable, Mapping
from dataclasses import dataclass
from typing import Generic, TypeVar


V = TypeVar("V")


class InvalidConfiguration(ValueError):
    pass


@dataclass(frozen=True, slots=True)
class ConfigurationResolution(Generic[V]):
    values: dict[str, V]
    fallback_keys: tuple[str, ...]
    rejected_keys: tuple[str, ...]


def _require_text_keys(mapping: Mapping[object, object], *, name: str) -> None:
    if any(not isinstance(key, str) or not key for key in mapping):
        raise ValueError(f"{name} keys must be non-empty text")


def resolve_last_known_good(
    incoming: Mapping[str, V],
    cached: Mapping[str, V],
    *,
    validate: Callable[[str, V], None],
) -> ConfigurationResolution[V]:
    if not isinstance(incoming, Mapping) or not isinstance(cached, Mapping):
        raise TypeError("incoming and cached must be mappings")
    if not callable(validate):
        raise TypeError("validate must be callable")
    _require_text_keys(incoming, name="incoming")
    _require_text_keys(cached, name="cached")

    values: dict[str, V] = {}
    fallback_keys: list[str] = []
    rejected_keys: list[str] = []

    for key, incoming_value in incoming.items():
        try:
            validate(key, incoming_value)
        except InvalidConfiguration:
            if key not in cached:
                rejected_keys.append(key)
                continue
            cached_value = cached[key]
            try:
                validate(key, cached_value)
            except InvalidConfiguration:
                rejected_keys.append(key)
            else:
                values[key] = cached_value
                fallback_keys.append(key)
        else:
            values[key] = incoming_value

    return ConfigurationResolution(
        values=values,
        fallback_keys=tuple(sorted(fallback_keys)),
        rejected_keys=tuple(sorted(rejected_keys)),
    )
```

## Example

```python
incoming = {
    "current": {"revision": 2},
    "fallback": {},
    "rejected": {},
    "disabled": False,
}
cached = {
    "fallback": {"revision": 1},
    "rejected": {},
    "cached-only": {"revision": 9},
}


def validate_entry(_key: str, value: object) -> None:
    if value is False:
        return
    if not isinstance(value, dict) or "revision" not in value:
        raise InvalidConfiguration("revision is required")


resolution = resolve_last_known_good(
    incoming,
    cached,
    validate=validate_entry,
)

assert resolution == ConfigurationResolution(
    values={
        "current": {"revision": 2},
        "fallback": {"revision": 1},
        "disabled": False,
    },
    fallback_keys=("fallback",),
    rejected_keys=("rejected",),
)
```

## Trade-offs and Limitations

Missing incoming keys are deliberately omitted, so this assumes the incoming
mapping describes the complete desired set rather than a patch. Values are
shared by reference with the inputs, and a validator can still mutate them;
copy under a separate ownership policy when necessary. Reusing a valid cached
entry may violate freshness, revocation, rollout, or cross-entry constraints
that this local decision cannot see. The function catches only
`InvalidConfiguration`; programming errors propagate. It returns an empty
result when nothing is usable and leaves fail-closed versus fail-open behavior,
persistence, locking, atomic publication, and audit logging to the caller.

## Related Snippets

<!-- catalog:related:start -->
- [Replace a File Atomically with a Sibling Temporary File](../storage-databases/replace-a-file-atomically-with-a-sibling-temporary-file.md)
- [Merge Nested Mappings Without Mutating Inputs](../configuration-serialization/merge-nested-mappings-without-mutating-inputs.md)
- [Merge Nested Configuration with an Explicit Delete Sentinel](../configuration-serialization/merge-nested-configuration-with-an-explicit-delete-sentinel.md)
<!-- catalog:related:end -->
