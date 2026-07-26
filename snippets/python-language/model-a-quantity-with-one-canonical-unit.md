---
title: "Model a Quantity with One Canonical Unit"
snippet_type: pattern
use_cases:
  - validation
  - data-transformation
tested_python:
  - "3.14"
dependencies: []
related:
  - validate-reused-fields-with-a-data-descriptor.md
  - apply-partial-dataclass-updates-with-an-omitted-value-sentinel.md
  - ../configuration-serialization/convert-decimal-values-to-exact-minor-units.md
---

# Model a Quantity with One Canonical Unit

## Idea and Problem

Represent a small quantity with one immutable canonical value while exposing named factories and read-only properties for alternate units.

A single stored representation prevents different unit fields from drifting
apart. Central validation applies equally whether callers construct the
canonical unit directly or use a conversion factory.

## When to Use

Use this pattern for a small domain value with a few well-defined conversions,
one natural canonical unit, and validation that belongs with the value. The
temperature example stores kelvin and derives Celsius and Fahrenheit. Use a
specialized units library when dimensional analysis, compound units, large
unit catalogs, uncertainty, or high-precision scientific arithmetic is
required.

## Implementation

```python
import math
from dataclasses import dataclass


def _finite_number(value: float, *, name: str) -> float:
    if isinstance(value, bool) or not isinstance(value, (int, float)):
        raise TypeError(f"{name} must be numeric")
    normalized = float(value)
    if not math.isfinite(normalized):
        raise ValueError(f"{name} must be finite")
    return normalized


@dataclass(frozen=True, slots=True)
class Temperature:
    kelvin: float

    _CELSIUS_OFFSET = 273.15
    _FAHRENHEIT_OFFSET = 459.67

    def __post_init__(self) -> None:
        kelvin = _finite_number(self.kelvin, name="kelvin")
        if kelvin < 0.0:
            raise ValueError("kelvin cannot be below absolute zero")
        object.__setattr__(self, "kelvin", kelvin)

    @classmethod
    def from_celsius(cls, value: float) -> "Temperature":
        celsius = _finite_number(value, name="celsius")
        return cls(celsius + cls._CELSIUS_OFFSET)

    @classmethod
    def from_fahrenheit(cls, value: float) -> "Temperature":
        fahrenheit = _finite_number(value, name="fahrenheit")
        kelvin = (fahrenheit + cls._FAHRENHEIT_OFFSET) * (5.0 / 9.0)
        return cls(kelvin)

    @property
    def celsius(self) -> float:
        return self.kelvin - self._CELSIUS_OFFSET

    @property
    def fahrenheit(self) -> float:
        converted = self.kelvin * (9.0 / 5.0) - self._FAHRENHEIT_OFFSET
        if not math.isfinite(converted):
            raise OverflowError("fahrenheit conversion is not representable")
        return converted
```

## Example

```python
from dataclasses import FrozenInstanceError


freezing = Temperature.from_celsius(0)
room = Temperature.from_fahrenheit(68)
absolute_zero_c = Temperature.from_celsius(-273.15)
absolute_zero_f = Temperature.from_fahrenheit(-459.67)

try:
    Temperature.from_celsius(-273.151)
except ValueError:
    below_absolute_zero_rejected = True
else:
    below_absolute_zero_rejected = False

try:
    freezing.kelvin = 1
except FrozenInstanceError:
    mutation_rejected = True
else:
    mutation_rejected = False

assert (
    math.isclose(freezing.kelvin, 273.15),
    math.isclose(freezing.fahrenheit, 32.0),
    math.isclose(room.celsius, 20.0),
    absolute_zero_c.kelvin,
    absolute_zero_f.kelvin,
    below_absolute_zero_rejected,
    mutation_rejected,
) == (True, True, True, 0.0, 0.0, True, True)
```

## Trade-offs and Limitations

Binary floating point introduces small round-trip errors, so comparisons need
a deliberate tolerance. Immutability prevents in-place updates but creates a
new object for every change. This example validates only the absolute lower
bound and does not model uncertainty, measurement provenance, intervals, or
unit algebra. A finite canonical value can still overflow an alternate-unit
conversion, which is reported explicitly. Financial, protocol-critical, or
high-precision scientific quantities may require integers, `Decimal`, rational
arithmetic, or a mature units package instead.

## Related Snippets

<!-- catalog:related:start -->
- [Validate Reused Fields with a Data Descriptor](validate-reused-fields-with-a-data-descriptor.md)
- [Apply Partial Dataclass Updates with an Omitted-Value Sentinel](apply-partial-dataclass-updates-with-an-omitted-value-sentinel.md)
- [Convert Decimal Values to Exact Minor Units](../configuration-serialization/convert-decimal-values-to-exact-minor-units.md)
<!-- catalog:related:end -->
