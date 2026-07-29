---
title: "Divide Bounded Decimals under an Explicit Context and Report Signals"
snippet_type: pattern
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - derive-reciprocal-bid-ask-quotes-with-explicit-decimal-rounding.md
  - build-quantized-decimal-views-with-deltas-at-explicit-scales.md
  - ../configuration-serialization/render-a-finite-decimal-in-canonical-plain-notation.md
---

# Divide Bounded Decimals under an Explicit Context and Report Signals

## Idea and Problem

Perform bounded Decimal division in a fresh explicit context and return the arithmetic signals with the result.

Decimal rounding can be successful while still setting `Inexact`, `Rounded`,
`Subnormal`, or `Underflow`. Those flags are sticky inside a context, so using
the ambient context can make a result depend on unrelated prior work. A fresh
context makes precision, exponent range, rounding, traps, and signal capture
part of one local operation.

## When to Use

Use this pattern when a data pipeline or calculation boundary must distinguish
an exact quotient from a rounded or subnormal one without changing the
process's ambient Decimal policy. Both operands and the desired rounding mode
must already belong to the caller's explicit numeric contract.

Use domain-specific quantization when the required output scale is fixed, and
use a broader numerical policy object when several operations must share one
context deliberately. Signal reporting describes arithmetic conditions; it is
not a substitute for business validation or an error budget.

## Implementation

```python
from dataclasses import dataclass
from decimal import (
    ROUND_05UP,
    ROUND_CEILING,
    ROUND_DOWN,
    ROUND_FLOOR,
    ROUND_HALF_DOWN,
    ROUND_HALF_EVEN,
    ROUND_HALF_UP,
    ROUND_UP,
    Clamped,
    Context,
    Decimal,
    DivisionByZero,
    Inexact,
    InvalidOperation,
    Overflow,
    Rounded,
    Subnormal,
    Underflow,
)
from enum import StrEnum

_MAX_COEFFICIENT_DIGITS = 256
_MIN_ADJUSTED_EXPONENT = -999
_MAX_ADJUSTED_EXPONENT = 999
_MIN_PRECISION = 1
_MAX_PRECISION = 100


class DecimalRounding(StrEnum):
    CEILING = ROUND_CEILING
    DOWN = ROUND_DOWN
    FLOOR = ROUND_FLOOR
    HALF_DOWN = ROUND_HALF_DOWN
    HALF_EVEN = ROUND_HALF_EVEN
    HALF_UP = ROUND_HALF_UP
    UP = ROUND_UP
    ZERO_FIVE_UP = ROUND_05UP


class ArithmeticSignal(StrEnum):
    CLAMPED = "clamped"
    INEXACT = "inexact"
    ROUNDED = "rounded"
    SUBNORMAL = "subnormal"
    UNDERFLOW = "underflow"


@dataclass(frozen=True, slots=True)
class DecimalDivision:
    result: Decimal
    signals: tuple[ArithmeticSignal, ...]


class DecimalDivisionError(ArithmeticError):
    pass


_REPORTED_SIGNALS = (
    (Clamped, ArithmeticSignal.CLAMPED),
    (Inexact, ArithmeticSignal.INEXACT),
    (Rounded, ArithmeticSignal.ROUNDED),
    (Subnormal, ArithmeticSignal.SUBNORMAL),
    (Underflow, ArithmeticSignal.UNDERFLOW),
)
_TRAPPED_SIGNALS = (InvalidOperation, DivisionByZero, Overflow)


def _validate_operand(value: Decimal, *, name: str, allow_zero: bool) -> None:
    if type(value) is not Decimal:
        raise TypeError(f"{name} must be an exact Decimal")
    if not value.is_finite():
        raise ValueError(f"{name} must be finite")
    if not allow_zero and value.is_zero():
        raise ValueError(f"{name} must not be zero")
    if len(value.as_tuple().digits) > _MAX_COEFFICIENT_DIGITS:
        raise ValueError(f"{name} exceeds the supported coefficient digits")
    if not _MIN_ADJUSTED_EXPONENT <= value.adjusted() <= _MAX_ADJUSTED_EXPONENT:
        raise ValueError(f"{name} adjusted exponent is outside the supported range")


def divide_decimals(
    numerator: Decimal,
    denominator: Decimal,
    *,
    precision: int,
    rounding: DecimalRounding,
) -> DecimalDivision:
    _validate_operand(numerator, name="numerator", allow_zero=True)
    _validate_operand(denominator, name="denominator", allow_zero=False)
    if type(precision) is not int:
        raise TypeError("precision must be an exact integer")
    if not _MIN_PRECISION <= precision <= _MAX_PRECISION:
        raise ValueError("precision is outside the supported range")
    if type(rounding) is not DecimalRounding:
        raise TypeError("rounding must be an exact DecimalRounding")

    context = Context(
        prec=precision,
        rounding=rounding.value,
        Emin=_MIN_ADJUSTED_EXPONENT,
        Emax=_MAX_ADJUSTED_EXPONENT,
        capitals=1,
        clamp=0,
    )
    for signal in context.traps:
        context.traps[signal] = False
    for signal in _TRAPPED_SIGNALS:
        context.traps[signal] = True
    context.clear_flags()

    try:
        result = context.divide(numerator, denominator)
    except _TRAPPED_SIGNALS as error:
        raise DecimalDivisionError(
            f"decimal division trapped {type(error).__name__}"
        ) from error
    if not result.is_finite():
        raise ArithmeticError("decimal division produced a non-finite result")

    signals = tuple(
        reported
        for signal, reported in _REPORTED_SIGNALS
        if context.flags[signal]
    )
    return DecimalDivision(result=result, signals=signals)
```

## Example

```python
from decimal import getcontext


def context_snapshot(context: Context) -> tuple[object, ...]:
    signal_types = sorted(context.flags, key=lambda signal: signal.__name__)
    return (
        context.prec,
        context.rounding,
        context.Emin,
        context.Emax,
        context.capitals,
        context.clamp,
        tuple((signal.__name__, context.traps[signal]) for signal in signal_types),
        tuple((signal.__name__, context.flags[signal]) for signal in signal_types),
    )


before = context_snapshot(getcontext())

exact = divide_decimals(
    Decimal("1"),
    Decimal("8"),
    precision=3,
    rounding=DecimalRounding.HALF_EVEN,
)
repeating = divide_decimals(
    Decimal("1"),
    Decimal("7"),
    precision=3,
    rounding=DecimalRounding.HALF_EVEN,
)
rounded_zeroes = divide_decimals(
    Decimal("1.2300"),
    Decimal("1"),
    precision=3,
    rounding=DecimalRounding.HALF_EVEN,
)
tiny = divide_decimals(
    Decimal("1e-999"),
    Decimal("3"),
    precision=3,
    rounding=DecimalRounding.HALF_EVEN,
)

assert exact == DecimalDivision(Decimal("0.125"), ())
assert repeating == DecimalDivision(
    Decimal("0.143"),
    (ArithmeticSignal.INEXACT, ArithmeticSignal.ROUNDED),
)
assert rounded_zeroes == DecimalDivision(
    Decimal("1.23"),
    (ArithmeticSignal.ROUNDED,),
)
assert tiny == DecimalDivision(
    Decimal("3.3e-1000"),
    (
        ArithmeticSignal.INEXACT,
        ArithmeticSignal.ROUNDED,
        ArithmeticSignal.SUBNORMAL,
        ArithmeticSignal.UNDERFLOW,
    ),
)

try:
    divide_decimals(
        Decimal("1e999"),
        Decimal("1e-999"),
        precision=3,
        rounding=DecimalRounding.HALF_EVEN,
    )
except DecimalDivisionError as error:
    assert isinstance(error.__cause__, Overflow)
else:
    raise AssertionError("overflow must be trapped")

assert context_snapshot(getcontext()) == before
```

## Trade-offs and Limitations

Operand validation uses `O(D)` work for the bounded coefficient digits.
Decimal division time and state depend on the interpreter's arbitrary-precision
implementation, but the operand, exponent, and precision caps bound the
admitted operation.

Signals are returned in one fixed order, not in the order the arithmetic
engine happened to set them. Decimal signals are hierarchical: an underflowed
inexact result can also report subnormal, rounded, and inexact conditions.
`Rounded` without `Inexact` is possible when only insignificant trailing zeroes
are discarded.

The fresh context ignores ambient precision, rounding, traps, and sticky
flags, and direct `Context.divide` leaves them unchanged. The helper performs
one operation only; it does not quantize to a business scale, accumulate a
multi-step audit, choose a precision, or decide which reported signals a
particular domain should accept.

## Related Snippets

<!-- catalog:related:start -->
- [Derive Reciprocal Bid-Ask Quotes with Explicit Decimal Rounding](derive-reciprocal-bid-ask-quotes-with-explicit-decimal-rounding.md)
- [Build Quantized Decimal Views with Deltas at Explicit Scales](build-quantized-decimal-views-with-deltas-at-explicit-scales.md)
- [Render a Finite Decimal in Canonical Plain Notation](../configuration-serialization/render-a-finite-decimal-in-canonical-plain-notation.md)
<!-- catalog:related:end -->
