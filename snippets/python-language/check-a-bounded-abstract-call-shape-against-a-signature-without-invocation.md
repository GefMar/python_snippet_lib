---
title: "Check a Bounded Abstract Call Shape Against a Signature Without Invocation"
snippet_type: recipe
use_cases:
  - automation
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - dispatch-on-an-exact-tuple-of-argument-types.md
  - ../configuration-serialization/normalize-bounded-named-options-with-explicit-default-semantics.md
  - ../security-privacy/authorize-a-handler-through-explicit-request-extractors.md
---

# Check a Bounded Abstract Call Shape Against a Signature Without Invocation

## Idea and Problem

Check whether a positional-argument count and a tuple of keyword names can bind to one explicit, bounded inspect.Signature without inspecting or invoking a callable.

Private marker objects stand in for values only while `Signature.bind`
applies Python's call grammar. Discarding the resulting `BoundArguments`
keeps the answer value-free: the helper reports only whether the abstract shape
binds.

## When to Use

Use this for a call planner, adapter, test generator, or declarative dispatcher
that knows how many positional arguments and which keyword names a later call
would contain. Construct the signature explicitly in trusted configuration and
choose limits that match the supported interface.

Do not derive a signature from an arbitrary live callable at this boundary:
callable introspection can consult hooks. Binding also says nothing about
argument values, annotations, overloads, types, default suitability, or what a
callable would do when invoked.

## Implementation

```python
from collections.abc import Iterable
from inspect import Parameter, Signature

_MAX_PARAMETERS = 64
_MAX_POSITIONAL_ARGUMENTS = 64
_MAX_KEYWORDS = 64
_MAX_NAME_BYTES = 128
_MAX_TOTAL_NAME_BYTES = 4_096
_MARKER = object()


class CallShapeProfileError(ValueError):
    """Raised when a signature or requested shape exceeds the fixed profile."""


def _validate_names(names: Iterable[str], *, field: str) -> None:
    total_bytes = 0
    for name in names:
        if type(name) is not str:
            raise TypeError(f"{field} names must be exact strings")
        try:
            name_bytes = len(name.encode("utf-8"))
        except UnicodeEncodeError:
            raise CallShapeProfileError(
                f"{field} names must be UTF-8 encodable"
            ) from None
        if name_bytes > _MAX_NAME_BYTES:
            raise CallShapeProfileError(
                f"a {field} name exceeds the byte limit"
            )
        total_bytes += name_bytes
        if total_bytes > _MAX_TOTAL_NAME_BYTES:
            raise CallShapeProfileError(
                f"{field} names exceed the aggregate byte limit"
            )


def call_shape_matches(
    signature: Signature,
    positional_count: int,
    keyword_names: tuple[str, ...],
) -> bool:
    """Return whether an abstract argument shape binds to an exact signature."""
    if type(signature) is not Signature:
        raise TypeError("signature must be an exact inspect.Signature")

    parameters = signature.parameters.values()
    if len(signature.parameters) > _MAX_PARAMETERS:
        raise CallShapeProfileError("signature has too many parameters")
    if any(type(parameter) is not Parameter for parameter in parameters):
        raise TypeError("signature parameters must be exact inspect.Parameter objects")
    _validate_names(
        (parameter.name for parameter in parameters),
        field="parameter",
    )

    if type(positional_count) is not int:
        raise TypeError("positional_count must be an exact int")
    if not 0 <= positional_count <= _MAX_POSITIONAL_ARGUMENTS:
        raise CallShapeProfileError("positional_count is outside the profile")

    if type(keyword_names) is not tuple:
        raise TypeError("keyword_names must be an exact tuple")
    if len(keyword_names) > _MAX_KEYWORDS:
        raise CallShapeProfileError("keyword_names has too many items")
    _validate_names(keyword_names, field="keyword")
    if len(set(keyword_names)) != len(keyword_names):
        raise CallShapeProfileError("keyword_names must be unique")

    positional_markers = (_MARKER,) * positional_count
    keyword_markers = dict.fromkeys(keyword_names, _MARKER)
    try:
        signature.bind(*positional_markers, **keyword_markers)
    except TypeError:
        return False
    return True
```

## Example

```python
signature = Signature(
    (
        Parameter("head", Parameter.POSITIONAL_ONLY),
        Parameter(
            "tail",
            Parameter.POSITIONAL_OR_KEYWORD,
            default=0,
        ),
        Parameter("extras", Parameter.VAR_POSITIONAL),
        Parameter("required", Parameter.KEYWORD_ONLY),
        Parameter("options", Parameter.VAR_KEYWORD),
    )
)

cases = (
    ((0, ("required",)), False),
    ((1, ()), False),
    ((1, ("required",)), True),
    ((2, ("tail", "required")), False),
    ((2, ("required",)), True),
    ((4, ("required", "extra")), True),
    ((1, ("head", "required")), True),
    ((1, ("not-an-identifier", "required")), True),
)
observed = tuple(
    call_shape_matches(signature, *shape)
    for shape, _ in cases
)

profile_failures = 0
for shape in (
    (1, ("required", "required")),
    (65, ("required",)),
    (1, tuple(f"option-{index}" for index in range(65))),
    (1, ("x" * 129,)),
    (1, ("\ud800",)),
):
    try:
        call_shape_matches(signature, *shape)
    except CallShapeProfileError:
        profile_failures += 1

try:
    call_shape_matches(signature, True, ("required",))
except TypeError:
    boolean_count_rejected = True
else:
    boolean_count_rejected = False

assert (
    observed == tuple(expected for _, expected in cases)
    and all(type(result) is bool for result in observed)
    and profile_failures == 5
    and boolean_count_rejected
)
```

## Trade-offs and Limitations

Let `P` be the parameter count, `A` the positional count, `K` the
keyword count, and `B` the inspected name bytes. Validation and binding use
`O(P + A + K + B)` time and `O(A + K)` temporary marker storage under
the fixed limits. Defaults, annotations, and a signature's return annotation
remain referenced by the supplied signature but are never evaluated or
returned.

The helper distinguishes a malformed profile from a well-formed shape that
does not bind: bad types, duplicate requested names, and limit violations raise,
while ordinary binding conflicts return `False`. A positional-only parameter
name can legitimately appear in the keyword mapping when `**kwargs` captures
it. The result validates Python call syntax only; it is not authorization and
does not predict a successful or side-effect-free invocation.

## Related Snippets

<!-- catalog:related:start -->
- [Dispatch on an Exact Tuple of Argument Types](dispatch-on-an-exact-tuple-of-argument-types.md)
- [Normalize Bounded Named Options with Explicit Default Semantics](../configuration-serialization/normalize-bounded-named-options-with-explicit-default-semantics.md)
- [Authorize a Handler Through Explicit Request Extractors](../security-privacy/authorize-a-handler-through-explicit-request-extractors.md)
<!-- catalog:related:end -->
