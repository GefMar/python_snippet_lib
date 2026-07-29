---
title: "Inspect Bounded Named Class Attributes Without Invoking Descriptors"
snippet_type: testing-technique
use_cases:
  - testing
  - validation
  - automation
tested_python:
  - "3.14"
dependencies: []
related:
  - index-bounded-python-scope-bindings-with-symtable-without-execution.md
  - ../python-language/validate-reused-fields-with-a-data-descriptor.md
  - ../python-language/collect-decorated-methods-in-class-definition-order.md
---

# Inspect Bounded Named Class Attributes Without Invoking Descriptors

## Idea and Problem

Classify selected raw class attributes without binding methods, evaluating properties, invoking custom descriptor hooks, or retaining the inspected objects.

Generic descriptor detection needs more than `__get__`: set-only and
delete-only objects also participate in the descriptor protocol. Scanning
bounded class namespace mappings through direct `type.__getattribute__` calls
avoids a second dynamic lookup even when a raw value's type has a hostile
metaclass.

## When to Use

Use this for a trusted class in a documentation generator, plugin audit, test
assertion, or static inventory that requests a small closed set of member
names. It is suitable when the desired answer is descriptor shape rather than
the value normal attribute access would produce.

Do not use it to inspect untrusted live objects, reproduce `getattr`, resolve a
descriptor, or discover attributes synthesized dynamically. If bound values
are required, ordinary lookup is the correct operation and may execute user
code by design.

## Implementation

```python
import inspect
from dataclasses import dataclass
from enum import StrEnum
from types import FunctionType

_MAX_ATTRIBUTE_NAMES = 128
_MAX_ATTRIBUTE_NAME_BYTES = 128
_MAX_CLASS_MRO = 64
_DESCRIPTOR_HOOKS = ("__get__", "__set__", "__delete__")


class StaticAttributeLimitError(ValueError):
    """Raised when static inspection exceeds the closed profile."""


class StaticAttributeKind(StrEnum):
    MISSING = "missing"
    PLAIN = "plain"
    FUNCTION = "function"
    PROPERTY = "property"
    STATICMETHOD = "staticmethod"
    CLASSMETHOD = "classmethod"
    DESCRIPTOR = "descriptor"


@dataclass(frozen=True, slots=True)
class StaticAttribute:
    name: str
    kind: StaticAttributeKind


def _bounded_mro(cls: type) -> tuple[type, ...]:
    mro = type.__getattribute__(cls, "__mro__")
    if len(mro) > _MAX_CLASS_MRO:
        raise StaticAttributeLimitError("a class MRO exceeds the supported limit")
    return mro


def _has_descriptor_hook(raw_type: type) -> bool:
    for base in _bounded_mro(raw_type):
        namespace = type.__getattribute__(base, "__dict__")
        if any(name in namespace for name in _DESCRIPTOR_HOOKS):
            return True
    return False


def _classify_static_attribute(raw: object) -> StaticAttributeKind:
    raw_type = type(raw)
    if raw_type is FunctionType:
        return StaticAttributeKind.FUNCTION
    if raw_type is property:
        return StaticAttributeKind.PROPERTY
    if raw_type is staticmethod:
        return StaticAttributeKind.STATICMETHOD
    if raw_type is classmethod:
        return StaticAttributeKind.CLASSMETHOD
    if _has_descriptor_hook(raw_type):
        return StaticAttributeKind.DESCRIPTOR
    return StaticAttributeKind.PLAIN


def inspect_named_class_attributes(
    target: type,
    names: tuple[str, ...],
) -> tuple[StaticAttribute, ...]:
    """Classify named raw class members without resolving their values."""
    if type(target) is not type:
        raise TypeError("target must be a class with exact metaclass type")
    _bounded_mro(target)

    if type(names) is not tuple:
        raise TypeError("names must be an exact tuple")
    if not 1 <= len(names) <= _MAX_ATTRIBUTE_NAMES:
        raise StaticAttributeLimitError("names has an invalid item count")

    seen: set[str] = set()
    result: list[StaticAttribute] = []
    for index, name in enumerate(names):
        if type(name) is not str:
            raise TypeError(f"names[{index}] must be an exact str")
        if (
            not name.isascii()
            or not name.isidentifier()
            or not 1 <= len(name) <= _MAX_ATTRIBUTE_NAME_BYTES
        ):
            raise StaticAttributeLimitError(
                f"names[{index}] must be a bounded ASCII identifier"
            )
        if name in seen:
            raise StaticAttributeLimitError("attribute names must be unique")
        seen.add(name)

        try:
            raw = inspect.getattr_static(target, name)
        except AttributeError:
            kind = StaticAttributeKind.MISSING
        else:
            kind = _classify_static_attribute(raw)
        result.append(StaticAttribute(name=name, kind=kind))
    return tuple(result)
```

## Example

```python
events = {"get": 0, "set": 0, "delete": 0, "metaclass": 0}


class GetOnly:
    def __get__(self, instance: object, owner: type | None = None) -> object:
        events["get"] += 1
        raise RuntimeError("descriptor must not run")


class HostileMeta(type):
    def __getattribute__(self, name: str) -> object:
        events["metaclass"] += 1
        raise RuntimeError("metaclass lookup must not run")


class SetOnly(metaclass=HostileMeta):
    def __set__(self, instance: object, value: object) -> None:
        events["set"] += 1
        raise RuntimeError("descriptor must not run")


class DeleteOnly:
    def __delete__(self, instance: object) -> None:
        events["delete"] += 1
        raise RuntimeError("descriptor must not run")


class Base:
    inherited = 1


class Subject(Base):
    plain = 2
    opaque_marker = object()
    get_only = GetOnly()
    set_only = SetOnly()
    delete_only = DeleteOnly()

    @property
    def property_value(self) -> int:
        raise RuntimeError("property must not run")

    @staticmethod
    def static_method() -> None:
        pass

    @classmethod
    def class_method(cls) -> None:
        pass

    def function(self) -> None:
        pass


names = (
    "inherited",
    "plain",
    "opaque_marker",
    "get_only",
    "set_only",
    "delete_only",
    "property_value",
    "static_method",
    "class_method",
    "function",
    "missing",
)
records = inspect_named_class_attributes(Subject, names)


def class_with_mro_length(length: int) -> type:
    current: type = object
    for index in range(length - 1):
        current = type(f"Layer{length}_{index}", (current,), {})
    return current


mro_64 = class_with_mro_length(64)
mro_65 = class_with_mro_length(65)
target_mro_boundary = inspect_named_class_attributes(mro_64, ("missing",))


class RawTypeHolder:
    within_limit = mro_64()
    beyond_limit = mro_65()


raw_mro_boundary = inspect_named_class_attributes(
    RawTypeHolder,
    ("within_limit",),
)

limit_failures = 0
for target, requested in (
    (Subject, tuple(f"name_{index}" for index in range(129))),
    (mro_65, ("missing",)),
    (RawTypeHolder, ("beyond_limit",)),
):
    try:
        inspect_named_class_attributes(target, requested)
    except StaticAttributeLimitError:
        limit_failures += 1

name_boundary = inspect_named_class_attributes(
    Subject,
    tuple(f"name_{index}" for index in range(128)),
)

assert (
    tuple(record.kind for record in records)
    == (
        StaticAttributeKind.PLAIN,
        StaticAttributeKind.PLAIN,
        StaticAttributeKind.PLAIN,
        StaticAttributeKind.DESCRIPTOR,
        StaticAttributeKind.DESCRIPTOR,
        StaticAttributeKind.DESCRIPTOR,
        StaticAttributeKind.PROPERTY,
        StaticAttributeKind.STATICMETHOD,
        StaticAttributeKind.CLASSMETHOD,
        StaticAttributeKind.FUNCTION,
        StaticAttributeKind.MISSING,
    )
    and events == {"get": 0, "set": 0, "delete": 0, "metaclass": 0}
    and target_mro_boundary[0].kind is StaticAttributeKind.MISSING
    and raw_mro_boundary[0].kind is StaticAttributeKind.PLAIN
    and len(name_boundary) == 128
    and limit_failures == 3
)
```

## Trade-offs and Limitations

Static lookup intentionally differs from normal attribute access. It can find
raw descriptors that ordinary lookup would bind, and it can miss attributes
synthesized by `__getattr__`, `__getattribute__`, or a custom metaclass. The
target therefore requires exact metaclass `type`; raw value types may have
custom metaclasses because their MRO and class dictionaries are read through
the base `type` implementation directly.

Only names and closed classifications are returned. Raw values are neither
retained nor rendered, compared, resolved, or called. Target and raw-type MRO
limits bound lookup and descriptor-shape scanning for at most 128 names, though
the interpreter's static lookup details remain implementation-dependent. This
is an inspection aid for trusted classes, not a sandbox for hostile object
graphs.

## Related Snippets

<!-- catalog:related:start -->
- [Index Bounded Python Scope Bindings with symtable without Execution](index-bounded-python-scope-bindings-with-symtable-without-execution.md)
- [Validate Reused Fields with a Data Descriptor](../python-language/validate-reused-fields-with-a-data-descriptor.md)
- [Collect Decorated Methods in Class Definition Order](../python-language/collect-decorated-methods-in-class-definition-order.md)
<!-- catalog:related:end -->
