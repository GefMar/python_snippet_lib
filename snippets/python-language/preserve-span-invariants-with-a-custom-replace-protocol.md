---
title: "Preserve Span Invariants with a Custom Replace Protocol"
snippet_type: pattern
use_cases:
  - data-transformation
  - interoperability
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - apply-partial-dataclass-updates-with-an-omitted-value-sentinel.md
  - build-an-immutable-slice-aware-sequence.md
  - pass-constructor-only-context-with-initvar.md
---

# Preserve Span Invariants with a Custom Replace Protocol

## Idea and Problem

Opt a custom immutable value into copy.replace without letting partial updates bypass its cross-field invariants.

Python 3.13 added a common replacement operation for named tuples,
dataclasses, and classes that implement `__replace__`. A custom value should
combine the requested fields with its current state and then call the same
validating constructor used for initial creation. That keeps a replacement from
producing a state that the original type would reject.

## When to Use

Use this pattern for a small immutable value that is not a dataclass or named
tuple but should participate in generic named-field replacement. The example
models a half-open byte span with exact non-negative integer positions and the
invariant `start <= stop`.

Keep explicit domain operations when a change needs authorization, I/O,
version checks, event emission, or different validation from construction.
Prefer a frozen dataclass when generated initialization, comparison, and
replacement already express the complete value contract.

## Implementation

```python
from copy import replace
from typing import Self, final

_MAX_POSITION = (1 << 63) - 1
_REPLACEABLE_FIELDS = frozenset({"start", "stop"})


@final
class ByteSpan:
    __slots__ = ("_start", "_stop")

    def __init__(self, start: int, stop: int) -> None:
        if type(start) is not int or type(stop) is not int:
            raise TypeError("span positions must be exact integers")
        if not 0 <= start <= stop <= _MAX_POSITION:
            raise ValueError("span must satisfy 0 <= start <= stop <= 2^63 - 1")
        object.__setattr__(self, "_start", start)
        object.__setattr__(self, "_stop", stop)

    def __setattr__(self, name: str, value: object) -> None:
        raise AttributeError("ByteSpan values are immutable")

    @property
    def start(self) -> int:
        return self._start

    @property
    def stop(self) -> int:
        return self._stop

    def __replace__(self, /, **changes: object) -> Self:
        if type(self) is not ByteSpan:
            raise TypeError("replacement requires an exact ByteSpan value")
        unknown = changes.keys() - _REPLACEABLE_FIELDS
        if unknown:
            name = min(unknown)
            raise TypeError(f"unsupported replacement field: {name}")
        return ByteSpan(
            changes.get("start", self.start),
            changes.get("stop", self.stop),
        )

    def __eq__(self, other: object) -> bool:
        if type(other) is not ByteSpan:
            return NotImplemented
        return (self.start, self.stop) == (other.start, other.stop)

    def __hash__(self) -> int:
        return hash((self.start, self.stop))

    def __repr__(self) -> str:
        return f"ByteSpan(start={self.start}, stop={self.stop})"


```

## Example

```python
original = ByteSpan(4, 9)
same_value = replace(original)
extended = replace(original, stop=12)
shifted = replace(original, start=5, stop=10)


class DerivedByteSpan(ByteSpan):
    __slots__ = ()


failures = []
for operation in (
    lambda: replace(original, start=10),
    lambda: replace(original, owner="other"),
    lambda: replace(original, stop=True),
    lambda: replace(DerivedByteSpan(1, 2), stop=3),
    lambda: setattr(original, "_start", 0),
):
    try:
        operation()
    except Exception as error:
        failures.append(type(error) in {AttributeError, TypeError, ValueError})
    else:
        failures.append(False)

assert (
    original,
    same_value,
    same_value is original,
    extended,
    shifted,
    extended is original,
    tuple(failures),
) == (
    ByteSpan(4, 9),
    ByteSpan(4, 9),
    False,
    ByteSpan(4, 12),
    ByteSpan(5, 10),
    False,
    (True, True, True, True, True),
)
```

## Trade-offs and Limitations

Each replacement validates at most two named changes and constructs one new
object, so its work and additional state are constant. The original remains
untouched, and even an empty replacement creates a distinct value as required
by this closed contract.

`copy.replace` is a shallow named-field operation. It does not recursively copy
mutable members, merge nested values, authorize fields, or provide optimistic
concurrency control. This example avoids those issues by storing only exact
integers. A type with mutable members must define and document its ownership
policy separately.

The field allowlist and constructor call must evolve together. Silently
ignoring an unknown name would hide caller errors, while assigning internal
slots directly would bypass validation. The runtime exact-type guard rejects
subclass instances; `final` communicates the same closed hierarchy to static
tools.

## Related Snippets

<!-- catalog:related:start -->
- [Apply Partial Dataclass Updates with an Omitted-Value Sentinel](apply-partial-dataclass-updates-with-an-omitted-value-sentinel.md)
- [Build an Immutable Slice-Aware Sequence](build-an-immutable-slice-aware-sequence.md)
- [Pass Constructor-Only Context with dataclasses.InitVar](pass-constructor-only-context-with-initvar.md)
<!-- catalog:related:end -->
