---
title: "Load Bounded Trusted Extension Factories by One Entry Point"
snippet_type: recipe
use_cases:
  - interoperability
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - dispatch-named-strategies-with-an-explicit-function-mapping.md
  - load-text-templates-from-package-resources.md
---

# Load Bounded Trusted Extension Factories by One Entry Point

## Idea and Problem

Load one consistently named factory from each explicitly trusted module after validating the complete bounded request.

Each frozen specification separates its public name from its importable module
name. The loader validates the tuple, immutable allowlist, factory attribute,
name syntax, uniqueness, bounds, and every allowlist membership before the
first import. It then returns input-ordered frozen descriptors without calling
any factory.

## When to Use

Use this recipe at an application composition boundary when a small, fixed set
of installed extension modules is trusted and configuration selects which of
them to load. The caller must construct both the specification tuple and the
allowlist; neither is discovered from directories, packages, or user input.
Use direct imports when the module set never varies.

## Implementation

```python
import re
from collections.abc import Callable
from dataclasses import dataclass
from importlib import import_module


_PUBLIC_NAME = re.compile(
    r"[a-z][a-z0-9]*(?:[-_][a-z0-9]+)*\Z",
    flags=re.ASCII,
)
_MODULE_NAME = re.compile(
    r"[A-Za-z_][A-Za-z0-9_]{0,62}"
    r"(?:\.[A-Za-z_][A-Za-z0-9_]{0,62}){0,7}\Z",
    flags=re.ASCII,
)
_ATTRIBUTE_NAME = re.compile(
    r"[A-Za-z_][A-Za-z0-9_]{0,63}\Z",
    flags=re.ASCII,
)
_MAX_EXTENSIONS = 32
_MAX_PUBLIC_NAME_LENGTH = 64
_MAX_MODULE_NAME_LENGTH = 200


class ExtensionImportError(ImportError):
    pass


class InvalidExtensionFactory(Exception):
    pass


@dataclass(frozen=True, slots=True)
class ExtensionSpec:
    public_name: str
    module_name: str


@dataclass(frozen=True, slots=True)
class LoadedExtension:
    public_name: str
    module_name: str
    factory: Callable[..., object]


def _require_public_name(value: object, *, context: str) -> str:
    if (
        type(value) is not str
        or len(value) > _MAX_PUBLIC_NAME_LENGTH
        or _PUBLIC_NAME.fullmatch(value) is None
    ):
        raise ValueError(f"{context} must be a bounded lowercase ASCII name")
    return value


def _require_module_name(value: object, *, context: str) -> str:
    if (
        type(value) is not str
        or len(value) > _MAX_MODULE_NAME_LENGTH
        or _MODULE_NAME.fullmatch(value) is None
    ):
        raise ValueError(f"{context} must be a bounded absolute module name")
    return value


def _require_attribute_name(value: object) -> str:
    if type(value) is not str or _ATTRIBUTE_NAME.fullmatch(value) is None:
        raise ValueError("factory_attribute must be a bounded ASCII identifier")
    return value


def _validate_request(
    specs: object,
    factory_attribute: object,
    trusted_modules: object,
) -> tuple[ExtensionSpec, ...]:
    if type(specs) is not tuple:
        raise TypeError("specs must be a tuple")
    if not 1 <= len(specs) <= _MAX_EXTENSIONS:
        raise ValueError(f"specs must contain between 1 and {_MAX_EXTENSIONS} entries")
    _require_attribute_name(factory_attribute)

    if type(trusted_modules) is not frozenset:
        raise TypeError("trusted_modules must be a frozenset")
    if not 1 <= len(trusted_modules) <= _MAX_EXTENSIONS:
        raise ValueError(
            f"trusted_modules must contain between 1 and {_MAX_EXTENSIONS} names"
        )
    for module_name in trusted_modules:
        _require_module_name(module_name, context="trusted module name")

    public_names: set[str] = set()
    module_names: set[str] = set()
    for index, spec in enumerate(specs):
        if type(spec) is not ExtensionSpec:
            raise TypeError(f"specs[{index}] must be an ExtensionSpec")
        public_name = _require_public_name(
            spec.public_name,
            context=f"specs[{index}].public_name",
        )
        module_name = _require_module_name(
            spec.module_name,
            context=f"specs[{index}].module_name",
        )
        if public_name in public_names:
            raise ValueError(f"duplicate public name: {public_name!r}")
        if module_name in module_names:
            raise ValueError(f"duplicate module name: {module_name!r}")
        if module_name not in trusted_modules:
            raise ValueError(
                f"module for public extension {public_name!r} is not trusted"
            )
        public_names.add(public_name)
        module_names.add(module_name)

    return specs


def load_extension_factories(
    specs: tuple[ExtensionSpec, ...],
    *,
    factory_attribute: str,
    trusted_modules: frozenset[str],
) -> tuple[LoadedExtension, ...]:
    validated_specs = _validate_request(specs, factory_attribute, trusted_modules)

    loaded: list[LoadedExtension] = []
    for spec in validated_specs:
        try:
            module = import_module(spec.module_name)
        except Exception as error:
            raise ExtensionImportError(
                f"extension {spec.public_name!r} failed to import "
                f"from {spec.module_name!r}"
            ) from error

        try:
            factory = getattr(module, factory_attribute)
        except AttributeError as error:
            raise InvalidExtensionFactory(
                f"extension {spec.public_name!r} has no "
                f"{factory_attribute!r} factory"
            ) from error
        except Exception as error:
            raise InvalidExtensionFactory(
                f"extension {spec.public_name!r} factory lookup failed"
            ) from error
        if not callable(factory):
            raise InvalidExtensionFactory(
                f"extension {spec.public_name!r} factory must be callable"
            )

        loaded.append(
            LoadedExtension(
                public_name=spec.public_name,
                module_name=spec.module_name,
                factory=factory,
            )
        )
    return tuple(loaded)
```

## Example

```python
import sys
from types import ModuleType


factory_calls = []


def build_plain() -> object:
    factory_calls.append("plain")
    return object()


def build_boxed() -> object:
    factory_calls.append("boxed")
    return object()


module_names = ("plain_extension", "boxed_extension")
plain_module = ModuleType(module_names[0])
boxed_module = ModuleType(module_names[1])
plain_module.build = build_plain
boxed_module.build = build_boxed

previous_modules = {name: sys.modules.get(name) for name in module_names}
try:
    sys.modules[module_names[0]] = plain_module
    sys.modules[module_names[1]] = boxed_module
    extensions = load_extension_factories(
        (
            ExtensionSpec("plain", module_names[0]),
            ExtensionSpec("boxed", module_names[1]),
        ),
        factory_attribute="build",
        trusted_modules=frozenset(module_names),
    )
finally:
    for name, previous in previous_modules.items():
        if previous is None:
            sys.modules.pop(name, None)
        else:
            sys.modules[name] = previous

assert (
    tuple(extension.public_name for extension in extensions),
    tuple(extension.factory for extension in extensions),
    factory_calls,
) == (("plain", "boxed"), (build_plain, build_boxed), [])
```

## Trade-offs and Limitations

Allowlist membership is a configuration check, not proof that a module is
safe. Importing trusted Python code can perform irreversible side effects, and
a failure in a later module cannot undo earlier imports. Normal import
semantics also reuse entries from `sys.modules`, so repeated calls do not imply
fresh module execution. This recipe provides no discovery, installation,
user-controlled import paths, sandbox, dependency injection, factory
invocation, lifecycle management, hot reload, rollback, or mutable registry.

## Related Snippets

<!-- catalog:related:start -->
- [Dispatch Named Strategies with an Explicit Function Mapping](dispatch-named-strategies-with-an-explicit-function-mapping.md)
- [Load Text Templates from Package Resources](load-text-templates-from-package-resources.md)
<!-- catalog:related:end -->
