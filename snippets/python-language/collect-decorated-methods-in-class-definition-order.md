---
title: "Collect Decorated Methods in Class Definition Order"
snippet_type: pattern
use_cases:
  - automation
  - configuration
tested_python:
  - "3.14"
dependencies: []
related:
  - dispatch-named-strategies-with-an-explicit-function-mapping.md
  - validate-reused-fields-with-a-data-descriptor.md
---

# Collect Decorated Methods in Class Definition Order

## Idea and Problem

Build an immutable per-class registry from explicitly decorated instance methods without inspecting caller frames or maintaining global mutable state.

The decorator only attaches frozen metadata and returns the original function.
During class creation, `__init_subclass__` scans the ordered class namespace,
combines it with one inherited registry, and publishes a tuple. Decorated
subclass definitions replace a same-named inherited entry and append in the
subclass's definition order; undecorated overrides hide inherited entries.

## When to Use

Use this pattern when a class defines a small ordered set of commands,
renderers, checks, or handlers and declaration next to each method improves
readability. Registry classes must use single inheritance, and registered
members must be ordinary instance methods. Use an explicit function mapping
instead when mixins, multiple inheritance, or method binding add no value.

## Implementation

```python
import re
from dataclasses import dataclass
from types import FunctionType
from typing import ClassVar


_LABEL = re.compile(r"[a-z][a-z0-9_-]{0,63}\Z")
_MARKER = "__ordered_method_registration__"
_MAX_METHODS = 256


@dataclass(frozen=True, slots=True)
class MethodRegistration:
    label: str
    title: str
    method_name: str


@dataclass(frozen=True, slots=True)
class _Decoration:
    label: str
    title: str


def registered_method(*, label: str, title: str):
    if not isinstance(label, str) or _LABEL.fullmatch(label) is None:
        raise ValueError("label must be a canonical lowercase identifier")
    if not isinstance(title, str) or not 1 <= len(title) <= 100:
        raise ValueError("title must contain between 1 and 100 characters")

    decoration = _Decoration(label=label, title=title)

    def decorate(function: FunctionType) -> FunctionType:
        if not isinstance(function, FunctionType):
            raise TypeError("registered members must be plain functions")
        if hasattr(function, _MARKER):
            raise ValueError("a method can have only one registration")
        setattr(function, _MARKER, decoration)
        return function

    return decorate


class OrderedMethodRegistry:
    method_registry: ClassVar[tuple[MethodRegistration, ...]] = ()

    def __init_subclass__(cls, **kwargs: object) -> None:
        super().__init_subclass__(**kwargs)
        if len(cls.__bases__) != 1:
            raise TypeError("registry classes require single inheritance")
        if "method_registry" in cls.__dict__:
            raise TypeError("method_registry is reserved by the registry base")

        base = cls.__bases__[0]
        entries = list(base.__dict__.get("method_registry", ()))
        for method_name, member in cls.__dict__.items():
            if isinstance(member, (classmethod, staticmethod)):
                if hasattr(member.__func__, _MARKER):
                    raise TypeError("registered descriptors are not supported")
                entries = [entry for entry in entries if entry.method_name != method_name]
                continue
            if isinstance(member, property):
                accessors = (member.fget, member.fset, member.fdel)
                if any(
                    accessor is not None and hasattr(accessor, _MARKER)
                    for accessor in accessors
                ):
                    raise TypeError("registered properties are not supported")
                entries = [entry for entry in entries if entry.method_name != method_name]
                continue

            existing_index = next(
                (
                    index
                    for index, entry in enumerate(entries)
                    if entry.method_name == method_name
                ),
                None,
            )
            decoration = (
                getattr(member, _MARKER, None)
                if isinstance(member, FunctionType)
                else None
            )
            if decoration is None:
                if existing_index is not None:
                    entries.pop(existing_index)
                continue

            registration = MethodRegistration(
                label=decoration.label,
                title=decoration.title,
                method_name=method_name,
            )
            if existing_index is not None:
                entries.pop(existing_index)
            conflicting_index = next(
                (
                    index
                    for index, entry in enumerate(entries)
                    if entry.label == registration.label
                ),
                None,
            )
            if conflicting_index is not None:
                raise ValueError(f"duplicate effective label: {registration.label!r}")

            entries.append(registration)
            if len(entries) > _MAX_METHODS:
                raise ValueError(f"a registry cannot exceed {_MAX_METHODS} methods")

        cls.method_registry = tuple(entries)
```

## Example

```python
class Formatter(OrderedMethodRegistry):
    @registered_method(label="plain", title="Plain text")
    def plain(self, value: str) -> str:
        return value

    @registered_method(label="compact", title="Compact text")
    def compact(self, value: str) -> str:
        return value.replace(" ", "")


class DisplayFormatter(Formatter):
    def plain(self, value: str) -> str:
        return value.lower()

    @registered_method(label="compact", title="Uppercase compact text")
    def compact(self, value: str) -> str:
        return value.replace(" ", "").upper()

    @registered_method(label="boxed", title="Boxed text")
    def boxed(self, value: str) -> str:
        return f"[{value}]"


formatter = DisplayFormatter()
labels = [entry.label for entry in DisplayFormatter.method_registry]
outputs = [
    getattr(formatter, entry.method_name)("a b")
    for entry in DisplayFormatter.method_registry
]

try:
    class DuplicateLabels(OrderedMethodRegistry):
        @registered_method(label="same", title="First")
        def first(self) -> None:
            pass

        @registered_method(label="same", title="Second")
        def second(self) -> None:
            pass
except ValueError:
    duplicate_rejected = True
else:
    duplicate_rejected = False

try:
    class ReservedName(OrderedMethodRegistry):
        @registered_method(label="reserved", title="Reserved")
        def method_registry(self) -> None:
            pass
except TypeError:
    reserved_name_rejected = True
else:
    reserved_name_rejected = False

assert (labels, outputs, duplicate_rejected, reserved_name_rejected) == (
    ["compact", "boxed"],
    ["AB", "[a b]"],
    True,
    True,
)
```

## Trade-offs and Limitations

This is intentionally class-local metadata, not plugin discovery. It supports
only single inheritance and plain instance methods; `staticmethod`,
`classmethod`, properties, mixins, dynamic injection, and
more than 256 effective entries require a different contract. The marker is an
attribute on a function and can still be altered deliberately before class
creation. Importing the class executes validation, so malformed declarations
fail early. For externally supplied extensions, add explicit loading, trust,
compatibility, and error-isolation policies instead of relying on decorators.

## Related Snippets

<!-- catalog:related:start -->
- [Dispatch Named Strategies with an Explicit Function Mapping](dispatch-named-strategies-with-an-explicit-function-mapping.md)
- [Validate Reused Fields with a Data Descriptor](validate-reused-fields-with-a-data-descriptor.md)
<!-- catalog:related:end -->
