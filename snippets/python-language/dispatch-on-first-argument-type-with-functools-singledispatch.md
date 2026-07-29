---
title: "Dispatch on First-Argument Type with functools.singledispatch"
snippet_type: idiom
use_cases:
  - automation
  - interoperability
tested_python:
  - "3.14"
dependencies: []
related:
  - dispatch-on-an-exact-tuple-of-argument-types.md
  - dispatch-named-strategies-with-an-explicit-function-mapping.md
  - type-a-narrow-structural-interface-with-protocol.md
---

# Dispatch on First-Argument Type with functools.singledispatch

## Idea and Problem

Extend one operation by the runtime type of its first argument while delegating inheritance-aware resolution to the standard library.

`functools.singledispatch` turns the default implementation into a generic
function. Registered concrete-type handlers keep their ordinary signatures,
while calls choose the most specific registered implementation available
through the first argument's method resolution order.

A specific `bool` registration matters because `bool` inherits from `int`.
The `dispatch()` attribute makes the selected implementation auditable without
calling it. Each handler normalizes a supported subclass through the concrete
built-in type before validation, so overridden subclass methods cannot bypass
the advertised bounds or replace the rendered value.

## When to Use

Use this idiom when independently maintained handlers implement one operation,
the first argument's runtime class is the complete selection key, and inherited
fallback is desirable. Populate the closed registry at a visible composition
point before concurrent calls begin.

Use an explicit mapping for named strategies, an ordinary conditional for a
tiny permanently closed choice, or a multiple-dispatch design when later
arguments also determine behavior. Static annotations of container elements do
not create runtime dispatch keys.

## Implementation

```python
from dataclasses import dataclass
from functools import singledispatch

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_RENDERED_TEXT_LENGTH = 256
_MAX_RENDERED_BYTE_LENGTH = 128


@dataclass(frozen=True, slots=True)
class ScalarRendering:
    kind: str
    text: str


def _finish_rendering(
    kind: str,
    payload: str,
    *,
    bracketed: bool,
) -> ScalarRendering:
    if type(bracketed) is not bool:
        raise TypeError("bracketed must be an exact bool")
    text = f"[{payload}]" if bracketed else payload
    return ScalarRendering(kind, text)


@singledispatch
def render_scalar(
    value: object,
    *,
    bracketed: bool = False,
) -> ScalarRendering:
    """Render one supported scalar selected by its first-argument type."""
    if type(bracketed) is not bool:
        raise TypeError("bracketed must be an exact bool")
    raise TypeError(f"unsupported scalar type: {type(value).__name__}")


@render_scalar.register
def _render_text(
    value: str,
    *,
    bracketed: bool = False,
) -> ScalarRendering:
    plain_text = str.__getitem__(value, slice(None))
    if len(plain_text) > _MAX_RENDERED_TEXT_LENGTH:
        raise ValueError("text exceeds the supported length")
    return _finish_rendering("text", plain_text, bracketed=bracketed)


@render_scalar.register
def _render_bytes(
    value: bytes,
    *,
    bracketed: bool = False,
) -> ScalarRendering:
    plain_bytes = bytes.__getitem__(value, slice(None))
    if len(plain_bytes) > _MAX_RENDERED_BYTE_LENGTH:
        raise ValueError("bytes exceeds the supported length")
    return _finish_rendering("bytes", plain_bytes.hex(), bracketed=bracketed)


@render_scalar.register
def _render_integer(
    value: int,
    *,
    bracketed: bool = False,
) -> ScalarRendering:
    plain_integer = int.__index__(value)
    if not _MIN_INT64 <= plain_integer <= _MAX_INT64:
        raise ValueError("integer is outside the signed 64-bit range")
    return _finish_rendering("integer", str(plain_integer), bracketed=bracketed)


@render_scalar.register
def _render_bool(
    value: bool,
    *,
    bracketed: bool = False,
) -> ScalarRendering:
    payload = "true" if value else "false"
    return _finish_rendering("bool", payload, bracketed=bracketed)
```

## Example

```python
class Label(str):
    pass


class MisleadingInteger(int):
    def __ge__(self, other: object) -> bool:
        return True

    def __le__(self, other: object) -> bool:
        return True

    def __str__(self) -> str:
        return "0"


plain_text = render_scalar(Label("ready"))
bracketed_text = render_scalar(Label("ready"), bracketed=True)
binary = render_scalar(b"\x00\xff")
integer = render_scalar(-42)
boolean = render_scalar(True)

assert (plain_text, bracketed_text, binary, integer, boolean) == (
    ScalarRendering("text", "ready"),
    ScalarRendering("text", "[ready]"),
    ScalarRendering("bytes", "00ff"),
    ScalarRendering("integer", "-42"),
    ScalarRendering("bool", "true"),
)
assert render_scalar.dispatch(Label) is _render_text
assert render_scalar.dispatch(bool) is _render_bool
assert render_scalar.dispatch(int) is _render_integer
assert render_scalar.dispatch(Label) is render_scalar.dispatch(type(Label("x")))

try:
    render_scalar(MisleadingInteger(1 << 100))
except ValueError:
    subclass_cannot_bypass_bounds = True
else:
    subclass_cannot_bypass_bounds = False

try:
    render_scalar(1.5)
except TypeError:
    unsupported_rejected = True
else:
    unsupported_rejected = False

assert subclass_cannot_bypass_bounds and unsupported_rejected
```

## Trade-offs and Limitations

Dispatch considers only the runtime type of the first positional argument.
`bracketed` changes formatting but never selects a handler. Resolution follows
the registered inheritance relationships; the standard dispatcher caches
repeated runtime-type lookups, while rendering itself costs
`O(output_length)`.

Registrations remain mutable in the underlying API, but this pattern treats the
registry as fixed before use. Import-order-dependent plugin registration can
make behavior difficult to audit. This closed example deliberately excludes
ABC and union registrations, whose overlapping relationships need a separately
designed ambiguity policy.

All handlers return `ScalarRendering`, but `singledispatch` does not enforce
that agreement. It also does not inspect later arguments, generic parameters,
list element types, values, or protocols. The bounded rendering is a diagnostic
representation, not a reversible or escaped serialization format.

Calling the built-in descriptors directly deliberately discards overridden
conversion, slicing, length, and formatting behavior from supported subclasses.
Preserve those custom semantics only with explicit registrations and contracts
for those subclasses.

## Related Snippets

<!-- catalog:related:start -->
- [Dispatch on an Exact Tuple of Argument Types](dispatch-on-an-exact-tuple-of-argument-types.md)
- [Dispatch Named Strategies with an Explicit Function Mapping](dispatch-named-strategies-with-an-explicit-function-mapping.md)
- [Type a Narrow Structural Interface with Protocol](type-a-narrow-structural-interface-with-protocol.md)
<!-- catalog:related:end -->
