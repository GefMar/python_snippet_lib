---
title: "Derive Reciprocal Bid-Ask Quotes with Explicit Decimal Rounding"
snippet_type: recipe
use_cases:
  - data-transformation
  - resource-management
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - build-quantized-decimal-views-with-deltas-at-explicit-scales.md
  - project-a-bounded-document-snapshot-into-collision-audited-secondary-index-rows.md
  - ../configuration-serialization/convert-decimal-values-to-exact-minor-units.md
---

# Derive Reciprocal Bid-Ask Quotes with Explicit Decimal Rounding

## Idea and Problem

Derive a bounded reciprocal quote batch with exact Decimal arithmetic, explicit rounding, and collision-audited reverse identifiers.

For each input, the reciprocal bid is one divided by the original ask and the
reciprocal ask is one divided by the original bid. This cross-over preserves
the side ordering before rounding. Both divisions use one caller-declared local
decimal context, and both results are then quantized to the caller's scale.

## When to Use

Use this recipe for a finite in-memory tuple of already observed, two-sided
quotes represented as exact `Decimal` values. Each frozen quote carries an
opaque observation time and integer version; both are validated and copied
unchanged. A complete one-to-one mapping declares every reciprocal identifier,
while a separate frozen set reserves identifiers already occupied elsewhere.

The caller must choose a precision, one supported decimal rounding mode, and a
scale from zero through eighteen. Inputs are strictly positive and finite, with
bid no greater than ask. Quote values are limited to 64 coefficient digits and
decimal exponents from -64 through 64; identifiers and metadata are bounded as
well.

## Implementation

```python
import re
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
    Context,
    Decimal,
    DecimalException,
    DivisionByZero,
    InvalidOperation,
    Overflow,
    localcontext,
)

_MAX_QUOTES = 1_024
_MAX_OCCUPIED_IDS = 4_096
_MAX_ID_BYTES = 64
_MAX_TOTAL_ID_BYTES = 512 * 1_024
_MAX_OBSERVED_AT_BYTES = 64
_MAX_TOTAL_METADATA_BYTES = 128 * 1_024
_MAX_INPUT_DIGITS = 64
_MIN_INPUT_EXPONENT = -64
_MAX_INPUT_EXPONENT = 64
_MIN_SCALE = 0
_MAX_SCALE = 18
_MAX_PRECISION = 96
_CONTEXT_EXPONENT_LIMIT = 256
_MAX_VERSION = 2**63 - 1
_ID = re.compile(r"[A-Za-z0-9][A-Za-z0-9._-]{0,63}\Z", re.ASCII).fullmatch
_ROUNDING_MODES = frozenset(
    {
        ROUND_05UP,
        ROUND_CEILING,
        ROUND_DOWN,
        ROUND_FLOOR,
        ROUND_HALF_DOWN,
        ROUND_HALF_EVEN,
        ROUND_HALF_UP,
        ROUND_UP,
    }
)
_INPUT_QUANTA = tuple(
    (exponent, Decimal(f"1e{exponent}"))
    for exponent in range(_MIN_INPUT_EXPONENT, _MAX_INPUT_EXPONENT + 1)
)
_SCALE_QUANTA = tuple(Decimal(f"1e-{scale}") for scale in range(_MIN_SCALE, _MAX_SCALE + 1))


@dataclass(frozen=True, slots=True)
class TwoSidedQuote:
    quote_id: str
    bid: Decimal
    ask: Decimal
    observed_at: str
    version: int


@dataclass(frozen=True, slots=True)
class ReverseId:
    quote_id: str
    reciprocal_id: str


def _validated_id(value: object, *, field: str) -> tuple[str, int]:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if not value or len(value) > _MAX_ID_BYTES or _ID(value) is None:
        raise ValueError(f"{field} must be a bounded conservative ASCII ID")
    return value, len(value)


def _validated_observed_at(value: object) -> tuple[str, int]:
    if type(value) is not str:
        raise TypeError("observed_at must be an exact string")
    if not value or len(value) > _MAX_OBSERVED_AT_BYTES:
        raise ValueError("observed_at is empty or too long")
    try:
        encoded = value.encode("utf-8")
    except UnicodeEncodeError as error:
        raise ValueError("observed_at must be valid UTF-8 text") from error
    if len(encoded) > _MAX_OBSERVED_AT_BYTES or value != value.strip() or not value.isprintable():
        raise ValueError("observed_at is unstripped, unprintable, or too long")
    return value, len(encoded)


def _validated_version(value: object) -> int:
    if type(value) is not int:
        raise TypeError("version must be an exact integer")
    if not 0 <= value <= _MAX_VERSION:
        raise ValueError("version is outside the supported range")
    return value


def _validated_decimal(value: object, *, field: str) -> Decimal:
    if type(value) is not Decimal:
        raise TypeError(f"{field} must be an exact Decimal")
    if not value.is_finite() or value <= 0:
        raise ValueError(f"{field} must be strictly positive and finite")

    exponent = next(
        (candidate for candidate, quantum in _INPUT_QUANTA if value.same_quantum(quantum)),
        None,
    )
    if exponent is None:
        raise ValueError(f"{field} exponent is outside the supported range")
    digit_count = value.adjusted() - exponent + 1
    if digit_count > _MAX_INPUT_DIGITS:
        raise ValueError(f"{field} exceeds the supported digit count")
    return value


def derive_reciprocal_quotes(
    quotes: tuple[TwoSidedQuote, ...],
    reverse_ids: tuple[ReverseId, ...],
    *,
    occupied_ids: frozenset[str],
    scale: int,
    precision: int,
    rounding: str,
) -> tuple[TwoSidedQuote, ...]:
    if type(quotes) is not tuple:
        raise TypeError("quotes must be an exact tuple")
    if len(quotes) > _MAX_QUOTES:
        raise ValueError("quote count exceeds the supported limit")
    if type(reverse_ids) is not tuple:
        raise TypeError("reverse_ids must be an exact tuple")
    if len(reverse_ids) != len(quotes):
        raise ValueError("reverse_ids must have one entry per quote")
    if type(occupied_ids) is not frozenset:
        raise TypeError("occupied_ids must be an exact frozenset")
    if len(occupied_ids) > _MAX_OCCUPIED_IDS:
        raise ValueError("occupied ID count exceeds the supported limit")
    if type(scale) is not int:
        raise TypeError("scale must be an exact integer")
    if not _MIN_SCALE <= scale <= _MAX_SCALE:
        raise ValueError("scale is outside the supported range")
    if type(precision) is not int:
        raise TypeError("precision must be an exact integer")
    if not 1 <= precision <= _MAX_PRECISION:
        raise ValueError("precision is outside the supported range")
    if type(rounding) is not str or rounding not in _ROUNDING_MODES:
        raise ValueError("rounding must be a supported decimal rounding mode")

    identifier_bytes = 0
    metadata_bytes = 0

    def check_id(value: object, *, field: str) -> str:
        nonlocal identifier_bytes
        checked, size = _validated_id(value, field=field)
        identifier_bytes += size
        if identifier_bytes > _MAX_TOTAL_ID_BYTES:
            raise ValueError("identifiers exceed the aggregate byte limit")
        return checked

    checked_quotes: list[TwoSidedQuote] = []
    input_ids: set[str] = set()
    for index, raw_quote in enumerate(quotes):
        field = f"quotes[{index}]"
        if type(raw_quote) is not TwoSidedQuote:
            raise TypeError(f"{field} must be an exact TwoSidedQuote")
        quote_id = check_id(raw_quote.quote_id, field=f"{field}.quote_id")
        if quote_id in input_ids:
            raise ValueError("input quote IDs must be unique")
        input_ids.add(quote_id)

        bid = _validated_decimal(raw_quote.bid, field=f"{field}.bid")
        ask = _validated_decimal(raw_quote.ask, field=f"{field}.ask")
        if bid > ask:
            raise ValueError("an input bid exceeds its ask")
        observed_at, observed_size = _validated_observed_at(raw_quote.observed_at)
        metadata_bytes += observed_size
        if metadata_bytes > _MAX_TOTAL_METADATA_BYTES:
            raise ValueError("observation metadata exceeds the aggregate byte limit")
        version = _validated_version(raw_quote.version)
        checked_quotes.append(TwoSidedQuote(quote_id, bid, ask, observed_at, version))

    if any(type(value) is not str for value in occupied_ids):
        raise TypeError("occupied_ids must contain only exact strings")
    occupied = {check_id(value, field="an occupied ID") for value in sorted(occupied_ids)}

    reverse_by_input: dict[str, str] = {}
    output_ids: set[str] = set()
    for index, raw_mapping in enumerate(reverse_ids):
        field = f"reverse_ids[{index}]"
        if type(raw_mapping) is not ReverseId:
            raise TypeError(f"{field} must be an exact ReverseId")
        source = check_id(raw_mapping.quote_id, field=f"{field}.quote_id")
        target = check_id(
            raw_mapping.reciprocal_id,
            field=f"{field}.reciprocal_id",
        )
        if source in reverse_by_input:
            raise ValueError("reverse-ID input mappings must be unique")
        if target in output_ids:
            raise ValueError("reciprocal output IDs must be unique")
        if target in input_ids:
            raise ValueError("a reciprocal ID collides with an input quote ID")
        if target in occupied:
            raise ValueError("a reciprocal ID collides with an occupied ID")
        reverse_by_input[source] = target
        output_ids.add(target)

    if set(reverse_by_input) != input_ids:
        raise ValueError("reverse-ID mappings must cover exactly the input IDs")

    quantum = _SCALE_QUANTA[scale]
    work_context = Context(
        prec=precision,
        rounding=rounding,
        Emin=-_CONTEXT_EXPONENT_LIMIT,
        Emax=_CONTEXT_EXPONENT_LIMIT,
    )
    work_context.traps[InvalidOperation] = True
    work_context.traps[DivisionByZero] = True
    work_context.traps[Overflow] = True

    staged: list[TwoSidedQuote] = []
    try:
        with localcontext(work_context):
            one = Decimal(1)
            for quote in checked_quotes:
                reciprocal_bid = (one / quote.ask).quantize(quantum)
                reciprocal_ask = (one / quote.bid).quantize(quantum)
                if (
                    not reciprocal_bid.is_finite()
                    or not reciprocal_ask.is_finite()
                    or reciprocal_bid <= 0
                    or reciprocal_ask <= 0
                ):
                    raise ValueError("a rounded reciprocal is not positive and finite")
                if reciprocal_bid > reciprocal_ask:
                    raise ValueError("a rounded reciprocal bid exceeds its ask")
                staged.append(
                    TwoSidedQuote(
                        quote_id=reverse_by_input[quote.quote_id],
                        bid=reciprocal_bid,
                        ask=reciprocal_ask,
                        observed_at=quote.observed_at,
                        version=quote.version,
                    )
                )
    except DecimalException as error:
        raise ValueError("reciprocal arithmetic exceeds the declared context") from error

    return tuple(staged)
```

## Example

```python
quotes = (
    TwoSidedQuote(
        quote_id="quote-a",
        bid=Decimal("2.000"),
        ask=Decimal("2.500"),
        observed_at="2026-01-15T12:00:00Z",
        version=7,
    ),
    TwoSidedQuote(
        quote_id="quote-b",
        bid=Decimal("4.000"),
        ask=Decimal("5.000"),
        observed_at="2026-01-15T12:00:01Z",
        version=8,
    ),
)
result = derive_reciprocal_quotes(
    quotes,
    (
        ReverseId("quote-b", "reciprocal-b"),
        ReverseId("quote-a", "reciprocal-a"),
    ),
    occupied_ids=frozenset({"reserved-id"}),
    scale=4,
    precision=28,
    rounding=ROUND_HALF_EVEN,
)

assert result == (
    TwoSidedQuote(
        "reciprocal-a",
        Decimal("0.4000"),
        Decimal("0.5000"),
        "2026-01-15T12:00:00Z",
        7,
    ),
    TwoSidedQuote(
        "reciprocal-b",
        Decimal("0.2000"),
        Decimal("0.2500"),
        "2026-01-15T12:00:01Z",
        8,
    ),
)
```

## Trade-offs and Limitations

The explicit context can round a non-terminating reciprocal before quantization,
so a precision that is too small can introduce double rounding or make
quantization fail. Quantization can also collapse a narrow spread to equal bid
and ask; equality is accepted, but a rounded zero or reversed output is not.
Choose precision, scale, and rounding as part of the caller's data contract.
This numerical transformation is not market or execution advice.

Preflight validates the entire quote tuple, metadata, mapping, occupied set, and
configuration before arithmetic. Derived records are staged and returned only
after every result passes the post-rounding checks. The fixed count, digit,
exponent, identifier, metadata, scale, and context bounds deliberately reject
some valid decimals and large batches. The helper accepts no floats and uses no
storage, external feeds, clocks, project-specific identifiers, or provenance.

Tests should exercise zero, negative, infinite, and NaN operands; crossed
inputs; digit and exponent edges; boolean scale, precision, and version values;
missing, extra, duplicate, and colliding mappings; every rounding mode; rounded
zero; spread collapse; insufficient precision; empty batches; and the maximum
accepted counts. Assert that output order follows quote order rather than
mapping order and that observation metadata is unchanged.

## Related Snippets

<!-- catalog:related:start -->
- [Build Quantized Decimal Views with Deltas at Explicit Scales](build-quantized-decimal-views-with-deltas-at-explicit-scales.md)
- [Project a Bounded Document Snapshot into Collision-Audited Secondary Index Rows](project-a-bounded-document-snapshot-into-collision-audited-secondary-index-rows.md)
- [Convert Decimal Values to Exact Minor Units](../configuration-serialization/convert-decimal-values-to-exact-minor-units.md)
<!-- catalog:related:end -->
