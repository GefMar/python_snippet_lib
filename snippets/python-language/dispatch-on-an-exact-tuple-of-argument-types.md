---
title: "Dispatch on an Exact Tuple of Argument Types"
snippet_type: pattern
use_cases:
  - data-transformation
  - interoperability
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - dispatch-named-strategies-with-an-explicit-function-mapping.md
  - type-a-narrow-structural-interface-with-protocol.md
  - collect-decorated-methods-in-class-definition-order.md
---

# Dispatch on an Exact Tuple of Argument Types

## Idea and Problem

Select one handler from the exact ordered runtime types of several positional arguments, using a bounded registry assembled at an explicit composition point.

The dispatcher copies all registrations into a read-only mapping only after
validating their common arity, type objects, handlers, and uniqueness. Lookup
uses `tuple(type(argument) ...)`, so it has no implicit subclass, abstract base
class, protocol, or best-match behavior.

## When to Use

Use this pattern when a small set of operations genuinely depends on the exact
combination of two or more stable runtime types. It can fit conversion or
interoperability boundaries where subclass matching would be surprising and
all supported signatures are known during composition.

Prefer direct branching for a few fixed cases. Use `functools.singledispatch`
when only the first argument controls dispatch and MRO-based subclass matching
is useful. Use a static overload or a purpose-built visitor when compile-time
type checking is the main requirement.

## Implementation

```python
from collections.abc import Callable, Iterable, Mapping
from dataclasses import dataclass
from itertools import islice
from types import MappingProxyType


_MAX_REGISTRATIONS = 128
_MIN_ARITY = 2
_MAX_ARITY = 8


class NoExactTypeMatchError(LookupError):
    pass


@dataclass(frozen=True, slots=True)
class ExactTypeRegistration:
    argument_types: tuple[type, ...]
    handler: Callable[..., object]


@dataclass(frozen=True, slots=True, init=False)
class ExactTypeDispatcher:
    _arity: int
    _handlers: Mapping[tuple[type, ...], Callable[..., object]]

    def __init__(
        self,
        registrations: Iterable[ExactTypeRegistration],
    ) -> None:
        if isinstance(registrations, (str, bytes)):
            raise TypeError("registrations must be a non-text iterable")
        try:
            provided = tuple(islice(registrations, _MAX_REGISTRATIONS + 1))
        except TypeError as error:
            raise TypeError("registrations must be iterable") from error
        if not provided:
            raise ValueError("at least one registration is required")
        if len(provided) > _MAX_REGISTRATIONS:
            raise ValueError("registration count exceeds the supported limit")

        handlers: dict[tuple[type, ...], Callable[..., object]] = {}
        arity: int | None = None
        for registration in provided:
            if type(registration) is not ExactTypeRegistration:
                raise TypeError(
                    "every registration must be an ExactTypeRegistration"
                )
            signature = registration.argument_types
            if type(signature) is not tuple:
                raise TypeError("argument_types must be a tuple")
            if not _MIN_ARITY <= len(signature) <= _MAX_ARITY:
                raise ValueError("signature arity is outside the supported range")
            if any(not isinstance(item, type) for item in signature):
                raise TypeError("every signature item must be a type object")
            try:
                hash(signature)
            except TypeError as error:
                raise TypeError("signature type objects must be hashable") from error
            if arity is None:
                arity = len(signature)
            elif len(signature) != arity:
                raise ValueError("all registrations must have the same arity")
            if not callable(registration.handler):
                raise TypeError("every registered handler must be callable")
            if signature in handlers:
                raise ValueError("exact type signatures must be unique")
            handlers[signature] = registration.handler

        object.__setattr__(self, "_arity", arity)
        object.__setattr__(self, "_handlers", MappingProxyType(handlers))

    @property
    def arity(self) -> int:
        return self._arity

    @property
    def signature_count(self) -> int:
        return len(self._handlers)

    def __call__(self, *arguments: object) -> object:
        if len(arguments) != self._arity:
            raise TypeError(f"dispatcher requires {self._arity} arguments")
        signature = tuple(type(argument) for argument in arguments)
        try:
            handler = self._handlers[signature]
        except KeyError as error:
            names = ", ".join(item.__name__ for item in signature)
            raise NoExactTypeMatchError(
                f"no handler for exact argument types: {names}"
            ) from error
        return handler(*arguments)
```

## Example

```python
def repeat_text(value: str, count: int) -> str:
    return value * count


def repeat_bytes(value: bytes, count: int) -> bytes:
    return value * count


def raise_from_handler(value: str, marker: bytes) -> object:
    raise KeyError((value, marker))


dispatcher = ExactTypeDispatcher(
    (
        ExactTypeRegistration((str, int), repeat_text),
        ExactTypeRegistration((bytes, int), repeat_bytes),
        ExactTypeRegistration((str, bytes), raise_from_handler),
    )
)

try:
    dispatcher(type("TextSubclass", (str,), {})('x'), 2)
except NoExactTypeMatchError:
    subclass_rejected = True
else:
    subclass_rejected = False

try:
    dispatcher("x", b"marker")
except KeyError as error:
    handler_error_preserved = error.args == (("x", b"marker"),)
else:
    handler_error_preserved = False

assert (
    dispatcher("ab", 2),
    dispatcher(b"ab", 2),
    dispatcher.arity,
    dispatcher.signature_count,
    subclass_rejected,
    handler_error_preserved,
) == ("abab", b"abab", 2, 3, True, True)
```

## Trade-offs and Limitations

Registry construction is `O(r)` for bounded registrations; dispatch builds an
arity-bounded type tuple and performs an average `O(1)` mapping lookup. The
mapping is immutable, but registered handler objects may retain mutable state
and their execution cost is not bounded by the dispatcher.

Exact matching deliberately ignores inheritance, virtual subclasses,
protocols, unions, coercions, keyword arguments, and ambiguity resolution. It
also gives a static type checker only a broad callable result unless the caller
adds a separate typed facade. Handler exceptions propagate unchanged; only a
missing registry key becomes `NoExactTypeMatchError`.

## Related Snippets

<!-- catalog:related:start -->
- [Dispatch Named Strategies with an Explicit Function Mapping](dispatch-named-strategies-with-an-explicit-function-mapping.md)
- [Type a Narrow Structural Interface with Protocol](type-a-narrow-structural-interface-with-protocol.md)
- [Collect Decorated Methods in Class Definition Order](collect-decorated-methods-in-class-definition-order.md)
<!-- catalog:related:end -->
