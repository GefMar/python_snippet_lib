---
title: "Resolve Bounded Configuration Through Dependent Adapters"
snippet_type: pattern
use_cases:
  - configuration
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - normalize-bounded-named-options-with-explicit-default-semantics.md
  - ../algorithms-data-structures/resolve-stable-ordering-constraints-with-topological-sort.md
  - ../python-language/make-a-defensive-copy-at-a-mutable-input-boundary.md
---

# Resolve Bounded Configuration Through Dependent Adapters

## Idea and Problem

Activate a closed set of configuration adapters only when their declared option dependencies have already been produced, then publish one immutable mapping after complete success.

Static cycle validation is necessary but not sufficient. An active upstream
adapter may legitimately omit an optional value, so a downstream adapter can
be topologically reachable without being runnable. A deterministic fixed-point
pass separates that conditional activation from ordinary graph ordering, while
a private value cache ensures that every dependency comes from an already
active adapter and is resolved only once.

## When to Use

Use this pattern for a small, trusted, in-memory configuration boundary whose
adapters are cooperative functions returning exact scalar values. It fits
conditional sources such as a regional override that should activate only when
an earlier source actually supplies a region. Adapter and option names are
public schema data; option values may contain secrets and are never rendered in
an error.

Parse untrusted text, perform authorization, and arrange I/O timeouts before
this boundary. Use a schema library for nested data or coercion, and use a
dedicated dependency-injection system when adapters own long-lived resources
or require cleanup.

## Implementation

```python
import math
import re
from collections.abc import Mapping
from dataclasses import dataclass
from types import MappingProxyType
from typing import Protocol

ConfigValue = bool | int | float | str
_NO_DEFAULT = object()
_MAX_OPTIONS = 32
_MAX_ADAPTERS = 16
_MAX_DEPENDENCIES = 8
_MAX_TEXT_BYTES = 4_096
_MAX_AGGREGATE_TEXT_BYTES = 65_536
_NAME = re.compile(r"[a-z][a-z0-9_-]{0,63}", re.ASCII).fullmatch
_ERROR_KIND = re.compile(r"[A-Za-z_][A-Za-z0-9_]{0,63}", re.ASCII).fullmatch
_VALUE_TYPES = (bool, int, float, str)


class ConfigurationSpecError(ValueError):
    pass


class ConfigurationResolutionError(ValueError):
    pass


class AdapterCallbackError(RuntimeError):
    def __init__(self, adapter_name: str, exception_kind: str) -> None:
        self.adapter_name = adapter_name
        self.exception_kind = exception_kind
        super().__init__(f"adapter {adapter_name!r} callback raised {exception_kind}")


class AdapterContractError(TypeError):
    def __init__(self, adapter_name: str, failure_kind: str) -> None:
        self.adapter_name = adapter_name
        self.failure_kind = failure_kind
        super().__init__(f"adapter {adapter_name!r} returned {failure_kind}")


class AdapterResolver(Protocol):
    def __call__(
        self,
        dependencies: Mapping[str, ConfigValue],
        /,
    ) -> dict[str, object]: ...


@dataclass(frozen=True, slots=True)
class OptionSpec:
    name: str
    adapter: str
    value_type: type
    required: bool
    default: object = _NO_DEFAULT


@dataclass(frozen=True, slots=True)
class AdapterSpec:
    name: str
    dependencies: tuple[str, ...]
    required: bool
    resolve: AdapterResolver


@dataclass(frozen=True, slots=True)
class _CheckedSpecs:
    options: tuple[OptionSpec, ...]
    adapters: tuple[AdapterSpec, ...]
    options_by_name: dict[str, OptionSpec]
    outputs_by_adapter: dict[str, tuple[OptionSpec, ...]]


def _text_size(value: str) -> int:
    if len(value) > _MAX_TEXT_BYTES:
        raise ValueError("a text value exceeds 4 KiB")
    try:
        size = len(value.encode("utf-8"))
    except UnicodeEncodeError:
        raise ValueError("a text value is not valid UTF-8") from None
    if size > _MAX_TEXT_BYTES:
        raise ValueError("a text value exceeds 4 KiB")
    return size


def _checked_value(value: object, expected_type: type) -> tuple[ConfigValue, int]:
    if type(value) is not expected_type:
        raise TypeError("a value does not have its declared exact type")
    if type(value) is float and not math.isfinite(value):
        raise ValueError("a float value must be finite")
    text_bytes = _text_size(value) if type(value) is str else 0
    return value, text_bytes


def _reject_adapter_cycles(
    adapters: tuple[AdapterSpec, ...],
    options_by_name: dict[str, OptionSpec],
) -> None:
    dependencies_by_adapter = {
        adapter.name: {options_by_name[dependency].adapter for dependency in adapter.dependencies}
        for adapter in adapters
    }
    ordered_names = tuple(adapter.name for adapter in adapters)
    completed: set[str] = set()
    while len(completed) < len(adapters):
        phase = tuple(
            name
            for name in ordered_names
            if name not in completed and dependencies_by_adapter[name] <= completed
        )
        if not phase:
            raise ConfigurationSpecError("adapter dependencies contain a cycle")
        completed.update(phase)


def _validate_specs(
    options: tuple[OptionSpec, ...],
    adapters: tuple[AdapterSpec, ...],
) -> _CheckedSpecs:
    if type(options) is not tuple or not 1 <= len(options) <= _MAX_OPTIONS:
        raise ConfigurationSpecError("options must contain 1 to 32 records")
    if type(adapters) is not tuple or not 1 <= len(adapters) <= _MAX_ADAPTERS:
        raise ConfigurationSpecError("adapters must contain 1 to 16 records")

    adapter_names: set[str] = set()
    for adapter in adapters:
        if type(adapter) is not AdapterSpec:
            raise TypeError("adapters must contain exact AdapterSpec records")
        if type(adapter.name) is not str or _NAME(adapter.name) is None:
            raise ConfigurationSpecError("an adapter name is invalid")
        if adapter.name in adapter_names:
            raise ConfigurationSpecError("adapter names must be unique")
        adapter_names.add(adapter.name)
        if type(adapter.dependencies) is not tuple:
            raise TypeError("adapter dependencies must be an exact tuple")
        if len(adapter.dependencies) > _MAX_DEPENDENCIES:
            raise ConfigurationSpecError("an adapter has more than 8 dependencies")
        seen_dependencies: set[str] = set()
        for dependency in adapter.dependencies:
            if type(dependency) is not str or _NAME(dependency) is None:
                raise ConfigurationSpecError("an adapter dependency name is invalid")
            if dependency in seen_dependencies:
                raise ConfigurationSpecError("adapter dependencies must be unique")
            seen_dependencies.add(dependency)
        if type(adapter.required) is not bool:
            raise TypeError("adapter required flags must be exact booleans")
        if not callable(adapter.resolve):
            raise TypeError("every adapter resolver must be callable")

    options_by_name: dict[str, OptionSpec] = {}
    outputs: dict[str, list[OptionSpec]] = {adapter.name: [] for adapter in adapters}
    default_text_bytes = 0
    for option in options:
        if type(option) is not OptionSpec:
            raise TypeError("options must contain exact OptionSpec records")
        if type(option.name) is not str or _NAME(option.name) is None:
            raise ConfigurationSpecError("an option name is invalid")
        if option.name in options_by_name:
            raise ConfigurationSpecError("option names must be unique")
        if type(option.adapter) is not str or _NAME(option.adapter) is None:
            raise ConfigurationSpecError("an option adapter name is invalid")
        if option.adapter not in adapter_names:
            raise ConfigurationSpecError("an option names an unknown adapter")
        if not any(option.value_type is candidate for candidate in _VALUE_TYPES):
            raise TypeError("option value types must be bool, int, float, or str")
        if type(option.required) is not bool:
            raise TypeError("option required flags must be exact booleans")
        if option.required and option.default is not _NO_DEFAULT:
            raise ConfigurationSpecError("a required option cannot have a default")
        if option.default is not _NO_DEFAULT:
            _, size = _checked_value(option.default, option.value_type)
            default_text_bytes += size
            if default_text_bytes > _MAX_AGGREGATE_TEXT_BYTES:
                raise ConfigurationSpecError("option defaults exceed 64 KiB of aggregate text")
        options_by_name[option.name] = option
        outputs[option.adapter].append(option)

    if any(not owned for owned in outputs.values()):
        raise ConfigurationSpecError("every adapter must own at least one option")
    for adapter in adapters:
        if any(name not in options_by_name for name in adapter.dependencies):
            raise ConfigurationSpecError("an adapter names an unknown dependency")
    _reject_adapter_cycles(adapters, options_by_name)
    return _CheckedSpecs(
        options=options,
        adapters=adapters,
        options_by_name=options_by_name,
        outputs_by_adapter={name: tuple(owned) for name, owned in outputs.items()},
    )


def _exception_kind(error: Exception) -> str:
    name = type(error).__name__
    return name if _ERROR_KIND(name) is not None else "AdapterError"


def _invoke_adapter(
    adapter: AdapterSpec,
    dependencies: Mapping[str, ConfigValue],
) -> tuple[object, str | None]:
    try:
        return adapter.resolve(dependencies), None
    except Exception as error:
        return None, _exception_kind(error)


def _adapter_values(
    adapter: AdapterSpec,
    checked: _CheckedSpecs,
    resolved: dict[str, ConfigValue],
    resolved_text_bytes: int,
) -> tuple[dict[str, ConfigValue], int]:
    dependency_values = MappingProxyType({name: resolved[name] for name in adapter.dependencies})
    raw_values, exception_kind = _invoke_adapter(adapter, dependency_values)
    if exception_kind is not None:
        raise AdapterCallbackError(adapter.name, exception_kind)

    owned = {option.name: option for option in checked.outputs_by_adapter[adapter.name]}
    if type(raw_values) is not dict or len(raw_values) > len(owned):
        raise AdapterContractError(adapter.name, "an invalid result mapping")

    copied: dict[str, ConfigValue] = {}
    added_text_bytes = 0
    for name, value in raw_values.items():
        if type(name) is not str or name not in owned:
            raise AdapterContractError(adapter.name, "an undeclared option")
        try:
            checked_value, size = _checked_value(value, owned[name].value_type)
        except (TypeError, ValueError):
            raise AdapterContractError(
                adapter.name,
                "an invalid option value",
            ) from None
        copied[name] = checked_value
        added_text_bytes += size

    new_text_bytes = resolved_text_bytes + added_text_bytes
    if new_text_bytes > _MAX_AGGREGATE_TEXT_BYTES:
        raise AdapterContractError(adapter.name, "too much aggregate text")
    return copied, new_text_bytes


def resolve_dependent_configuration(
    options: tuple[OptionSpec, ...],
    adapters: tuple[AdapterSpec, ...],
) -> Mapping[str, ConfigValue]:
    checked = _validate_specs(options, adapters)
    resolved: dict[str, ConfigValue] = {}
    active: set[str] = set()
    resolved_text_bytes = 0

    zero_dependency_adapters = tuple(
        adapter for adapter in checked.adapters if not adapter.dependencies
    )
    for adapter in zero_dependency_adapters:
        values, resolved_text_bytes = _adapter_values(
            adapter,
            checked,
            resolved,
            resolved_text_bytes,
        )
        resolved.update(values)
        active.add(adapter.name)

    for _phase in range(_MAX_ADAPTERS):
        ready = tuple(
            adapter
            for adapter in checked.adapters
            if adapter.name not in active and all(name in resolved for name in adapter.dependencies)
        )
        if not ready:
            break
        for adapter in ready:
            values, resolved_text_bytes = _adapter_values(
                adapter,
                checked,
                resolved,
                resolved_text_bytes,
            )
            resolved.update(values)
            active.add(adapter.name)

    inactive = tuple(adapter for adapter in checked.adapters if adapter.name not in active)
    if any(adapter.required for adapter in inactive):
        raise ConfigurationResolutionError("a required adapter has unresolved dependencies")
    if any(all(name in resolved for name in adapter.dependencies) for adapter in inactive):
        raise AssertionError("an inactive optional adapter has no missing dependency")

    published: dict[str, ConfigValue] = {}
    published_text_bytes = 0
    for option in checked.options:
        if option.name in resolved:
            value = resolved[option.name]
        elif option.default is not _NO_DEFAULT:
            value, _ = _checked_value(option.default, option.value_type)
        elif option.required:
            raise ConfigurationResolutionError("a required option is unresolved")
        else:
            continue
        if type(value) is str:
            published_text_bytes += _text_size(value)
            if published_text_bytes > _MAX_AGGREGATE_TEXT_BYTES:
                raise ConfigurationResolutionError(
                    "resolved configuration exceeds 64 KiB of aggregate text"
                )
        published[option.name] = value

    return MappingProxyType(published)
```

## Example

```python
def read_environment(
    _dependencies: Mapping[str, ConfigValue],
) -> dict[str, object]:
    return {"region": "central", "port": 8443}


def build_address(
    dependencies: Mapping[str, ConfigValue],
) -> dict[str, object]:
    return {"endpoint": (f"{dependencies['region']}.example.test:{dependencies['port']}")}


def choose_workers(
    dependencies: Mapping[str, ConfigValue],
) -> dict[str, object]:
    return {"workers": 8 if dependencies["profile"] == "burst" else 2}


option_specs = (
    OptionSpec("region", "environment", str, required=True),
    OptionSpec("port", "environment", int, required=True),
    OptionSpec("profile", "environment", str, required=False),
    OptionSpec("endpoint", "address", str, required=True),
    OptionSpec("workers", "tuning", int, required=False, default=3),
    OptionSpec("label", "environment", str, required=False, default="steady"),
)
adapter_specs = (
    AdapterSpec("address", ("region", "port"), True, build_address),
    AdapterSpec("environment", (), True, read_environment),
    AdapterSpec("tuning", ("profile",), False, choose_workers),
)

configuration = resolve_dependent_configuration(option_specs, adapter_specs)
try:
    configuration["region"] = "changed"
except TypeError:
    immutable = True
else:
    immutable = False


def reject_adapter(
    _dependencies: Mapping[str, ConfigValue],
) -> dict[str, object]:
    raise OSError("token=private-value")


try:
    resolve_dependent_configuration(
        (OptionSpec("token", "rejected", str, required=True),),
        (AdapterSpec("rejected", (), True, reject_adapter),),
    )
except AdapterCallbackError as error:
    sanitized_failure = (
        error.exception_kind,
        "private-value" not in str(error),
    )
else:
    sanitized_failure = ("missing", False)

try:
    resolve_dependent_configuration(
        (
            OptionSpec(
                "ratio",
                "finite",
                float,
                required=False,
                default=float("inf"),
            ),
        ),
        (AdapterSpec("finite", (), False, lambda _dependencies: {}),),
    )
except ValueError:
    non_finite_default_rejected = True
else:
    non_finite_default_rejected = False

assert (
    dict(configuration),
    immutable,
    sanitized_failure,
    non_finite_default_rejected,
) == (
    {
        "region": "central",
        "port": 8443,
        "endpoint": "central.example.test:8443",
        "workers": 3,
        "label": "steady",
    },
    True,
    ("OSError", True),
    True,
)
```

## Trade-offs and Limitations

The complete schema is checked before any resolver runs: counts, names,
ownership, defaults, unknown dependencies, and every static cycle are rejected
up front. Zero-dependency adapters run first in declaration order. Each of at
most 16 later phases takes one readiness snapshot, so values produced within a
phase become dependencies only in the next phase. Every resolver runs at most
once, and its dependency mapping is a read-only snapshot of cached immutable
scalars.

An omitted adapter result is a missing value, not an exception. It can leave an
optional downstream adapter inactive only when one of that adapter's declared
dependencies is absent. A callback exception instead aborts resolution with a
sanitized type name; exception messages and configuration values are not
retained in the raised error. `Exception` subclasses are sanitized, while
process-control `BaseException` subclasses propagate.

Exact scalar types make the final mapping deeply safe without recursive
copying; defaults are checked and reused without conversion, and exact floats
must be finite. This excludes `None`, containers, subclasses, and custom value
objects. Resolver callbacks can still perform side effects or block, so
production adapters need their own failure-atomicity and timeout contracts.
Defaults are publication fallbacks; they never satisfy adapter dependencies.
No partially resolved mapping escapes after an error.

## Related Snippets

<!-- catalog:related:start -->
- [Normalize Bounded Named Options with Explicit Default Semantics](normalize-bounded-named-options-with-explicit-default-semantics.md)
- [Resolve Stable Ordering Constraints with Topological Sort](../algorithms-data-structures/resolve-stable-ordering-constraints-with-topological-sort.md)
- [Make a Defensive Copy at a Mutable Input Boundary](../python-language/make-a-defensive-copy-at-a-mutable-input-boundary.md)
<!-- catalog:related:end -->
