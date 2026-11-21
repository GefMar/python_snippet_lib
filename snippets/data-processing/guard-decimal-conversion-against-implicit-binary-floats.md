---
title: "Guard Decimal Conversion Against Implicit Binary Floats"
snippet_type: recipe
use_cases:
  - validation
  - data-transformation
tested_python:
  - "3.14"
dependencies: []
related:
  - divide-bounded-decimals-under-an-explicit-context-and-report-signals.md
  - ../configuration-serialization/convert-decimal-values-to-exact-minor-units.md
---

# Guard Decimal Conversion Against Implicit Binary Floats

## Idea and Problem

Reject accidental conversions from binary floats without changing the caller's Decimal settings.

Use the decimal context's float-operation trap inside a short-lived local context. Explicit text and integer conversion still work, but a float passed directly to the constructor raises.

## When to Use

Use this at a conversion boundary where the caller is supposed to supply decimal text or integers and a binary float may already have rounded the intended decimal value.

## Implementation

```python
from collections.abc import Iterable
from decimal import Decimal, FloatOperation, localcontext


def strict_decimals(values: Iterable[str | int | float]) -> tuple[Decimal, ...]:
    with localcontext() as context:
        context.traps[FloatOperation] = True
        return tuple(Decimal(value) for value in values)
```

## Example

```python
from decimal import getcontext

before = getcontext().traps[FloatOperation]
try:
    strict_decimals(["0.10", 0.1])
except FloatOperation:
    rejected_float = True
else:
    rejected_float = False

assert (
    strict_decimals(["0.10", 2]),
    rejected_float,
    getcontext().traps[FloatOperation],
) == ((Decimal("0.10"), Decimal(2)), True, before)
```

## Trade-offs and Limitations

This guards the constructor call, not all numeric interactions. Explicit conversion with Decimal.from_float still works and preserves the exact stored binary value. The trap does not enforce scale, range, finiteness, or a business rule about rounding.

## Related Snippets

<!-- catalog:related:start -->
- [Divide Bounded Decimals under an Explicit Context and Report Signals](divide-bounded-decimals-under-an-explicit-context-and-report-signals.md)
- [Convert Decimal Values to Exact Minor Units](../configuration-serialization/convert-decimal-values-to-exact-minor-units.md)
<!-- catalog:related:end -->
