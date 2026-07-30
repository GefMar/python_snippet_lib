---
title: "Resolve Bounded Configuration Layers with Winner Provenance"
snippet_type: recipe
use_cases:
  - configuration
  - data-transformation
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - merge-nested-configuration-with-an-explicit-delete-sentinel.md
  - normalize-bounded-named-options-with-explicit-default-semantics.md
  - ../reliability-resilience/resolve-incoming-configuration-with-last-known-good-values.md
---

# Resolve Bounded Configuration Layers with Winner Provenance

## Idea and Problem

Overlay a small ordered set of flat configuration mappings while retaining the exact layer that supplied every winning value.

Each layer is validated and copied before resolution begins. Later layers have
higher priority, and membership rather than truthiness decides whether a value
overrides an earlier one. The result therefore preserves explicit `None`,
`False`, zero, and empty text while detaching both values and provenance from
the caller's mutable dictionaries.

## When to Use

Use this recipe when several already parsed configuration sources share one
flat key namespace and diagnostics must explain which source won. Declare the
layers from lowest to highest priority, keep their names stable, and apply a
separate schema after resolution when individual keys need different types or
cross-field rules.

Use a recursive merge when nested mappings have structural meaning. Use an
explicit deletion sentinel when a higher layer must remove a key rather than
set it to `None`. Parse files, environment variables, command-line text, and
secrets before they reach this in-memory boundary.

## Implementation

```python
import math
import re
from dataclasses import dataclass

_MAX_LAYERS = 16
_MAX_TOTAL_PAIRS = 512
_MAX_LAYER_NAME_BYTES = 64
_MAX_KEY_BYTES = 128
_MAX_STRING_VALUE_BYTES = 4_096
_MAX_TOTAL_PAYLOAD_BYTES = 65_536
_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_LAYER_NAME = re.compile(r"[a-z][a-z0-9_-]{0,63}", re.ASCII).fullmatch
_KEY = re.compile(r"[a-z][a-z0-9_.-]{0,127}", re.ASCII).fullmatch


@dataclass(frozen=True, slots=True)
class ConfigurationLayer:
    name: str
    values: dict[str, bool | int | float | str | None]


@dataclass(frozen=True, slots=True)
class ResolvedSetting:
    key: str
    value: bool | int | float | str | None
    source_layer: str


def _scalar_payload_size(value: object, *, location: str) -> int:
    if value is None:
        return 0
    if type(value) is bool:
        return 1
    if type(value) is int:
        if not _MIN_INT64 <= value <= _MAX_INT64:
            raise ValueError(f"{location} integer is outside the signed 64-bit range")
        return 8
    if type(value) is float:
        if not math.isfinite(value):
            raise ValueError(f"{location} float must be finite")
        return 8
    if type(value) is str:
        if len(value) > _MAX_STRING_VALUE_BYTES:
            raise ValueError(f"{location} text exceeds the UTF-8 byte limit")
        try:
            encoded = value.encode("utf-8", errors="strict")
        except UnicodeEncodeError as error:
            raise ValueError(f"{location} text must be valid UTF-8") from error
        if len(encoded) > _MAX_STRING_VALUE_BYTES:
            raise ValueError(f"{location} text exceeds the UTF-8 byte limit")
        return len(encoded)
    raise TypeError(f"{location} has an unsupported exact scalar type")


def _snapshot_layers(
    layers: tuple[ConfigurationLayer, ...],
) -> tuple[tuple[str, dict[str, bool | int | float | str | None]], ...]:
    if type(layers) is not tuple:
        raise TypeError("layers must be an exact tuple")
    if not 1 <= len(layers) <= _MAX_LAYERS:
        raise ValueError("layer count is outside the supported range")

    names: set[str] = set()
    pair_count = 0
    payload_bytes = 0
    snapshots: list[tuple[str, dict[str, bool | int | float | str | None]]] = []

    for layer_index, layer in enumerate(layers):
        if type(layer) is not ConfigurationLayer:
            raise TypeError("layers must contain exact ConfigurationLayer values")
        if type(layer.name) is not str or _LAYER_NAME(layer.name) is None:
            raise ValueError("layer names must be bounded lowercase ASCII names")
        if layer.name in names:
            raise ValueError("layer names must be unique")
        names.add(layer.name)
        payload_bytes += len(layer.name)
        if payload_bytes > _MAX_TOTAL_PAYLOAD_BYTES:
            raise ValueError("configuration payload exceeds the byte limit")

        if type(layer.values) is not dict:
            raise TypeError("layer values must be exact dictionaries")
        copied: dict[str, bool | int | float | str | None] = {}
        for key, value in layer.values.items():
            pair_count += 1
            if pair_count > _MAX_TOTAL_PAIRS:
                raise ValueError("configuration pairs exceed the supported count")
            if type(key) is not str or _KEY(key) is None:
                raise ValueError("configuration keys must be bounded lowercase ASCII keys")
            key_bytes = len(key)
            value_bytes = _scalar_payload_size(
                value,
                location=f"layers[{layer_index}].values[{key!r}]",
            )
            payload_bytes += key_bytes + value_bytes
            if payload_bytes > _MAX_TOTAL_PAYLOAD_BYTES:
                raise ValueError("configuration payload exceeds the byte limit")
            copied[key] = value
        snapshots.append((layer.name, copied))

    return tuple(snapshots)


def resolve_configuration_layers(
    layers: tuple[ConfigurationLayer, ...],
) -> tuple[ResolvedSetting, ...]:
    """Return key-sorted winning values from low-to-high priority layers."""
    snapshots = _snapshot_layers(layers)
    winners: dict[str, ResolvedSetting] = {}
    for layer_name, values in snapshots:
        for key, value in values.items():
            winners[key] = ResolvedSetting(key, value, layer_name)
    return tuple(winners[key] for key in sorted(winners))
```

## Example

```python


base_values: dict[str, bool | int | float | str | None] = {
    "enabled": True,
    "label": "standard",
    "optional": "fallback",
    "retries": 3,
}
site_values: dict[str, bool | int | float | str | None] = {
    "enabled": False,
    "optional": None,
    "retries": 0,
}
request_values: dict[str, bool | int | float | str | None] = {
    "label": "",
}

resolved = resolve_configuration_layers(
    (
        ConfigurationLayer("base", base_values),
        ConfigurationLayer("site", site_values),
        ConfigurationLayer("request", request_values),
    )
)

base_values["enabled"] = True
site_values["retries"] = 99
request_values["label"] = "changed"

try:
    resolve_configuration_layers(
        (
            ConfigurationLayer("same", {}),
            ConfigurationLayer("same", {}),
        )
    )
except ValueError:
    duplicate_layer_rejected = True
else:
    duplicate_layer_rejected = False

try:
    resolve_configuration_layers((ConfigurationLayer("invalid", {"ratio": float("nan")}),))
except ValueError:
    non_finite_rejected = True
else:
    non_finite_rejected = False

assert (
    resolved,
    duplicate_layer_rejected,
    non_finite_rejected,
) == (
    (
        ResolvedSetting("enabled", False, "site"),
        ResolvedSetting("label", "", "request"),
        ResolvedSetting("optional", None, "site"),
        ResolvedSetting("retries", 0, "site"),
    ),
    True,
    True,
)
```

## Trade-offs and Limitations

For `P` admitted pairs and `K` distinct keys, validation and overlay use
`O(P)` work, final ordering uses `O(K log K)` work, and snapshots plus the
result use `O(P + K)` state under fixed count and byte limits. Scalar values
are immutable exact built-in values, so the result retains no caller-owned
container. The original dictionaries must still be quiescent while they are
being copied.

Layer names report runtime configuration provenance, not trust or
authorization. The resolver does not retain losing values or a complete
override history, distinguish an absent key from a later deletion, redact
secrets, parse text, normalize key spelling, interpolate values, recurse into
containers, or validate relationships between settings. Sorting keys makes the
output deterministic but deliberately discards source insertion order.

## Related Snippets

<!-- catalog:related:start -->
- [Merge Nested Configuration with an Explicit Delete Sentinel](merge-nested-configuration-with-an-explicit-delete-sentinel.md)
- [Normalize Bounded Named Options with Explicit Default Semantics](normalize-bounded-named-options-with-explicit-default-semantics.md)
- [Resolve Incoming Configuration with Last-Known-Good Values](../reliability-resilience/resolve-incoming-configuration-with-last-known-good-values.md)
<!-- catalog:related:end -->
