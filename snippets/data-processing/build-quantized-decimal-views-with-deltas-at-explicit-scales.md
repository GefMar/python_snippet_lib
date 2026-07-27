---
title: "Build Quantized Decimal Views with Deltas at Explicit Scales"
snippet_type: recipe
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../configuration-serialization/convert-decimal-values-to-exact-minor-units.md
  - check-a-value-against-an-asymmetric-tolerance-band.md
---

# Build Quantized Decimal Views with Deltas at Explicit Scales

## Idea and Problem

Build immutable Decimal views at several explicit scales after applying one bounded rounding and delta contract to every view.

Quantizing the current and previous operands before subtraction makes each
reported delta agree with the displayed values. The relative delta divides the
signed absolute-unit delta by the magnitude of the quantized previous value,
so it remains a dimensionless ratio and has no implicit percentage conversion.

## When to Use

Use this recipe when one exact decimal measurement must be rendered or compared
at a small caller-selected set of scales. It fits finite in-memory values created
from exact text or integers, and preserves the requested scale order for tables,
audit records, or deterministic serialization.

Keep currency metadata, unit conversion, and presentation outside the helper.
Choose a domain-specific policy when a rounded-to-zero baseline should use a
fallback denominator instead of an unavailable relative delta.

## Implementation

```python
from dataclasses import dataclass
from decimal import Context, Decimal, ROUND_HALF_EVEN, localcontext


_MAX_INPUT_DIGITS = 64
_MIN_INPUT_EXPONENT = -64
_MAX_INPUT_EXPONENT = 64
_MAX_ABSOLUTE_VALUE = Decimal("1e32")
_MIN_SCALE = 0
_MAX_SCALE = 8
_MAX_SCALES = 8
_WORK_PRECISION = 96
_WORK_EXPONENT_LIMIT = 96
_SUPPORTED_QUANTA = tuple(
    (exponent, Decimal(f"1e{exponent}"))
    for exponent in range(_MIN_INPUT_EXPONENT, _MAX_INPUT_EXPONENT + 1)
)


@dataclass(frozen=True, slots=True)
class QuantizedDecimalView:
    scale: int
    current: Decimal
    previous: Decimal | None
    absolute_delta: Decimal | None
    relative_delta: Decimal | None


def _validated_decimal(value: object, *, name: str) -> Decimal:
    if type(value) is not Decimal:
        raise TypeError(f"{name} must be an exact Decimal")
    if not value.is_finite():
        raise ValueError(f"{name} must be finite")

    exponent = next(
        (
            candidate
            for candidate, quantum in _SUPPORTED_QUANTA
            if value.same_quantum(quantum)
        ),
        None,
    )
    if exponent is None:
        raise ValueError(f"{name} exponent is outside the supported range")
    digit_count = (
        1 if value.is_zero() else value.adjusted() - exponent + 1
    )
    if digit_count > _MAX_INPUT_DIGITS:
        raise ValueError(f"{name} exceeds the supported digit count")
    if value.copy_abs() > _MAX_ABSOLUTE_VALUE:
        raise ValueError(f"{name} magnitude exceeds the supported limit")
    return value


def _validated_scales(value: object) -> tuple[int, ...]:
    if type(value) is not tuple:
        raise TypeError("scales must be an exact tuple")
    if not 1 <= len(value) <= _MAX_SCALES:
        raise ValueError("scale count is outside the supported range")

    seen: set[int] = set()
    for scale in value:
        if type(scale) is not int:
            raise TypeError("scales must contain exact integers")
        if not _MIN_SCALE <= scale <= _MAX_SCALE:
            raise ValueError("a scale is outside the supported range")
        if scale in seen:
            raise ValueError("scales must be unique")
        seen.add(scale)
    return value


def build_quantized_decimal_views(
    current: Decimal,
    *,
    previous: Decimal | None = None,
    scales: tuple[int, ...],
) -> tuple[QuantizedDecimalView, ...]:
    current_value = _validated_decimal(current, name="current")
    previous_value = (
        None
        if previous is None
        else _validated_decimal(previous, name="previous")
    )
    requested_scales = _validated_scales(scales)

    views = []
    work_context = Context(
        prec=_WORK_PRECISION,
        rounding=ROUND_HALF_EVEN,
        Emin=-_WORK_EXPONENT_LIMIT,
        Emax=_WORK_EXPONENT_LIMIT,
    )
    with localcontext(work_context):
        for scale in requested_scales:
            quantum = Decimal(1).scaleb(-scale)
            current_view = current_value.quantize(quantum)

            if previous_value is None:
                previous_view = None
                absolute_delta = None
                relative_delta = None
            else:
                previous_view = previous_value.quantize(quantum)
                absolute_delta = current_view - previous_view
                relative_delta = (
                    None
                    if previous_view.is_zero()
                    else absolute_delta / previous_view.copy_abs()
                )

            views.append(
                QuantizedDecimalView(
                    scale=scale,
                    current=current_view,
                    previous=previous_view,
                    absolute_delta=absolute_delta,
                    relative_delta=relative_delta,
                )
            )
    return tuple(views)
```

## Example

```python
from decimal import Decimal


views = build_quantized_decimal_views(
    Decimal("12.345"),
    previous=Decimal("10.000"),
    scales=(2, 0, 3),
)
without_previous = build_quantized_decimal_views(
    Decimal("1.005"),
    scales=(2,),
)
rounded_zero_previous = build_quantized_decimal_views(
    Decimal("0.125"),
    previous=Decimal("0.004"),
    scales=(2,),
)

assert (views, without_previous, rounded_zero_previous) == (
    (
        QuantizedDecimalView(
            2,
            Decimal("12.34"),
            Decimal("10.00"),
            Decimal("2.34"),
            Decimal("0.234"),
        ),
        QuantizedDecimalView(
            0,
            Decimal("12"),
            Decimal("10"),
            Decimal("2"),
            Decimal("0.2"),
        ),
        QuantizedDecimalView(
            3,
            Decimal("12.345"),
            Decimal("10.000"),
            Decimal("2.345"),
            Decimal("0.2345"),
        ),
    ),
    (
        QuantizedDecimalView(2, Decimal("1.00"), None, None, None),
    ),
    (
        QuantizedDecimalView(
            2,
            Decimal("0.12"),
            Decimal("0.00"),
            Decimal("0.12"),
            None,
        ),
    ),
)
```

## Trade-offs and Limitations

The helper materializes at most eight records and performs every arithmetic
operation in a fresh 96-digit context. Relative division rounds to that
precision when its decimal expansion does not terminate. Digit, exponent,
magnitude, scale, and result-count limits intentionally reject otherwise valid
`Decimal` values so adversarial coefficients or exponents cannot make this
small view builder consume unbounded resources. Exponent validation performs at
most 129 `same_quantum()` comparisons, and the digit count is derived from the
adjusted position; rejection does not materialize a coefficient tuple or text
representation.

The absolute delta is signed, while the relative denominator is the magnitude
of the quantized previous value. Quantization can turn a nonzero input into
zero; in that case the absolute delta remains available but the relative delta
is `None`. A missing previous value makes both deltas `None`. The helper accepts
no floats, applies no unit rules, and never multiplies a ratio by 100; callers
must request percentage presentation explicitly.

## Related Snippets

<!-- catalog:related:start -->
- [Convert Decimal Values to Exact Minor Units](../configuration-serialization/convert-decimal-values-to-exact-minor-units.md)
- [Check a Value Against an Asymmetric Tolerance Band](check-a-value-against-an-asymmetric-tolerance-band.md)
<!-- catalog:related:end -->
