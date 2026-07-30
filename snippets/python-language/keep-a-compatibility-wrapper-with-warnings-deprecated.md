---
title: "Keep a Compatibility Wrapper with warnings.deprecated"
snippet_type: pattern
use_cases:
  - interoperability
  - lifecycle-management
  - testing
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - load-bounded-trusted-extension-factories-by-one-entry-point.md
  - dispatch-named-strategies-with-an-explicit-function-mapping.md
  - ../testing-tooling/verify-calls-to-a-standalone-function-with-create-autospec.md
---

# Keep a Compatibility Wrapper with warnings.deprecated

## Idea and Problem

Retain one old callable name temporarily while directing both runtime and static callers to its replacement.

Python's `warnings.deprecated` decorator records a PEP 702 deprecation for type
checkers and, by default, emits `DeprecationWarning` when the decorated function
is called. Applying it to a distinct compatibility wrapper keeps the preferred
function unmarked. Delegation also gives the old and new entry points one shared
implementation while migration is in progress.

## When to Use

Use this pattern when a public Python API has a replacement but existing callers
need one bounded compatibility interval. Keep the wrapper's old signature,
message, and delegation behavior explicit, and define the removal condition in
release policy outside the code.

Use a normal warning at a specific behavioral branch when an entire callable is
not deprecated. Use `category=None` only for a deliberately static-only signal.
Remove the wrapper after the published compatibility condition is met instead of
letting it become a permanent second API.

## Implementation

```python
from warnings import catch_warnings, deprecated, simplefilter

_MAX_LABEL_BYTES = 128
_MAX_PREFIX_BYTES = 32
_DEPRECATION_MESSAGE = (
    "use format_label(); legacy_label() will be removed in the next major release"
)


def _bounded_utf8(value: str, *, name: str, maximum: int, allow_empty: bool) -> str:
    if type(value) is not str:
        raise TypeError(f"{name} must be an exact string")
    try:
        size = len(value.encode("utf-8", errors="strict"))
    except UnicodeEncodeError as error:
        raise ValueError(f"{name} must be valid UTF-8 text") from error
    minimum = 0 if allow_empty else 1
    if not minimum <= size <= maximum:
        raise ValueError(f"{name} byte size is outside the supported range")
    return value


def format_label(value: str, *, prefix: str = "") -> str:
    value = _bounded_utf8(
        value,
        name="value",
        maximum=_MAX_LABEL_BYTES,
        allow_empty=False,
    )
    prefix = _bounded_utf8(
        prefix,
        name="prefix",
        maximum=_MAX_PREFIX_BYTES,
        allow_empty=True,
    )
    return f"{prefix}{value}"


@deprecated(_DEPRECATION_MESSAGE)
def legacy_label(value: str, *, prefix: str = "") -> str:
    return format_label(value, prefix=prefix)
```

## Example

```python
with catch_warnings(record=True) as legacy_warnings:
    simplefilter("always", DeprecationWarning)
    legacy_result = legacy_label("ready", prefix="state:")

with catch_warnings(record=True) as preferred_warnings:
    simplefilter("always", DeprecationWarning)
    preferred_result = format_label("ready", prefix="state:")

warning = legacy_warnings[0]
assert (
    legacy_result,
    preferred_result,
    len(legacy_warnings),
    warning.category,
    str(warning.message),
    legacy_label.__deprecated__,
    len(preferred_warnings),
) == (
    "state:ready",
    "state:ready",
    1,
    DeprecationWarning,
    _DEPRECATION_MESSAGE,
    _DEPRECATION_MESSAGE,
    0,
)
```

## Trade-offs and Limitations

`DeprecationWarning` is commonly hidden by default outside development and test
contexts, so runtime callers are not guaranteed to see it. Static diagnostics
depend on a type checker or editor that understands PEP 702. The decorator's
default `stacklevel=1` reports the direct caller; an extra public wrapper layer
may require a deliberately tested higher value. A warning filter can also turn
the warning into an exception before the compatibility wrapper delegates.

The decorator stores its message in `__deprecated__`, but it does not schedule,
enforce, or perform removal. A compatibility wrapper still expands the API that
must be documented and tested. Keep its message stable enough for tooling, and
coordinate the actual deadline through release policy.

Decorating an alias to the same function object would mark the preferred name
too, which is why this pattern uses a separate wrapper. The example covers one
ordinary function with the same explicit signature. Classes, overload-specific
deprecations, descriptors, signature translation, and asynchronous wrappers
need separate design and tests.

## Related Snippets

<!-- catalog:related:start -->
- [Load Bounded Trusted Extension Factories by One Entry Point](load-bounded-trusted-extension-factories-by-one-entry-point.md)
- [Dispatch Named Strategies with an Explicit Function Mapping](dispatch-named-strategies-with-an-explicit-function-mapping.md)
- [Verify Calls to a Standalone Function with Autospec](../testing-tooling/verify-calls-to-a-standalone-function-with-create-autospec.md)
<!-- catalog:related:end -->
