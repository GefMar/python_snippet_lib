---
title: "Report Missing Declared Protocol Members Without Binding Descriptors"
snippet_type: testing-technique
use_cases:
  - interoperability
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../python-language/type-a-narrow-structural-interface-with-protocol.md
  - inspect-bounded-named-class-attributes-without-invoking-descriptors.md
---

# Report Missing Declared Protocol Members Without Binding Descriptors

## Idea and Problem

List the protocol member names a candidate lacks under static attribute lookup.

`get_protocol_members()` supplies the required names, including inherited
protocol members. `inspect.getattr_static()` looks them up without binding
methods or evaluating properties. A sentinel distinguishes an absent name
from an attribute whose stored value happens to be `None`.

## When to Use

Use this in a test or developer diagnostic for a trusted adapter instance
and a declared `Protocol`. It helps identify absent names when accessing an
ordinary property might perform work. Use it as a presence report alongside
type checking or behavior tests.

## Implementation

```python
from inspect import getattr_static
from typing import get_protocol_members


def missing_protocol_members(candidate: object, protocol: type) -> tuple[str, ...]:
    missing = object()
    return tuple(
        name
        for name in sorted(get_protocol_members(protocol))
        if getattr_static(candidate, name, missing) is missing
    )
```

## Example

```python
from typing import Protocol


class Labeled(Protocol):
    @property
    def label(self) -> str: ...


class Reporter(Labeled, Protocol):
    def publish(self, message: str) -> None: ...

    def flush(self) -> None: ...


class LabelOnly:
    @property
    def label(self) -> str:
        raise AssertionError("the diagnostic must not evaluate this property")


class MethodsOnly:
    def publish(self, message: str) -> None:
        pass

    def flush(self) -> None:
        pass


class CompleteReporter(LabelOnly, MethodsOnly):
    pass


assert (
    missing_protocol_members(LabelOnly(), Reporter),
    missing_protocol_members(MethodsOnly(), Reporter),
    missing_protocol_members(CompleteReporter(), Reporter),
) == (("flush", "publish"), ("label",), ())
```

## Trade-offs and Limitations

The result describes names only. A found property can still raise when
read, a slot can exist without a value, and a found method can have the
wrong signature. Attributes synthesized by dynamic lookup may appear
absent to this diagnostic.

Pass a protocol class, not a parameterized alias. The standard-library
member lookup requires Python 3.13 or newer. An empty report means static
lookup found all the names. Check conformance separately when it matters.

## Related Snippets

<!-- catalog:related:start -->
- [Type a Narrow Structural Interface with Protocol](../python-language/type-a-narrow-structural-interface-with-protocol.md)
- [Inspect Bounded Named Class Attributes Without Invoking Descriptors](inspect-bounded-named-class-attributes-without-invoking-descriptors.md)
<!-- catalog:related:end -->
