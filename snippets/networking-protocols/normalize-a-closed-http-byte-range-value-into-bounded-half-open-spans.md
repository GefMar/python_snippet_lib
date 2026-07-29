---
title: "Normalize a Closed HTTP Byte-Range Value into Bounded Half-Open Spans"
snippet_type: recipe
use_cases:
  - networking
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - resume-a-bounded-http-byte-stream-with-validated-range-responses.md
  - parse-one-bounded-printable-ascii-http-1-field-section-with-exact-crlf-framing.md
  - ../algorithms-data-structures/coalesce-bounded-half-open-integer-intervals-under-an-explicit-adjacency-policy.md
---

# Normalize a Closed HTTP Byte-Range Value into Bounded Half-Open Spans

## Idea and Problem

Parse one deliberately closed canonical HTTP byte-range value and resolve its satisfiable members into one bounded union of half-open spans.

The profile recognizes only canonical closed, open-ended, and suffix members.
It distinguishes malformed syntax, a valid request with no satisfiable member,
and a union that exceeds the selected-byte budget. Resolution clamps members to
the known representation length before sorting and coalescing overlaps or
adjacency, so the result intentionally loses request order and multipart
boundaries.

## When to Use

Use this recipe at a controlled HTTP boundary that has already selected a
positive known representation length and explicitly wants one canonical byte
union rather than multipart response semantics. Apply method, precondition,
`If-Range`, response-status, and response-header policy outside this parser.

Use an HTTP implementation with complete range-field support when optional
whitespace, combined field lines, extension range units, multipart rendering,
or preservation of member order matters.

## Implementation

```python
from dataclasses import dataclass

_MAX_INT64 = (1 << 63) - 1
_MAX_INT64_TEXT = str(_MAX_INT64)
_MAX_FIELD_BYTES = 4_096
_MAX_MEMBERS = 16
_PREFIX = "bytes="


class ByteRangeError(ValueError):
    pass


class MalformedByteRangeError(ByteRangeError):
    pass


class UnsatisfiableByteRangeError(ByteRangeError):
    pass


class ByteRangeBudgetError(ByteRangeError):
    pass


@dataclass(frozen=True, slots=True, order=True)
class ByteSpan:
    start: int
    stop: int


@dataclass(frozen=True, slots=True)
class NormalizedByteRanges:
    spans: tuple[ByteSpan, ...]
    total: int


def _positive_int64(value: object, *, name: str) -> int:
    if type(value) is not int:
        raise TypeError(f"{name} must be an exact integer")
    if not 1 <= value <= _MAX_INT64:
        raise ValueError(f"{name} is outside the positive signed 64-bit range")
    return value


def _decimal(field_value: str, start: int, stop: int) -> int:
    digit_count = stop - start
    if not 1 <= digit_count <= 19:
        raise MalformedByteRangeError("decimal tokens must contain 1 to 19 digits")
    if digit_count > 1 and field_value[start] == "0":
        raise MalformedByteRangeError("decimal tokens must not have leading zeroes")
    if any(not "0" <= field_value[position] <= "9" for position in range(start, stop)):
        raise MalformedByteRangeError("decimal tokens must contain only ASCII digits")

    token = field_value[start:stop]
    if digit_count == 19 and token > _MAX_INT64_TEXT:
        raise MalformedByteRangeError("a decimal token exceeds signed 64-bit range")
    return int(token)


def _resolve_member(
    field_value: str,
    start: int,
    stop: int,
    *,
    length: int,
) -> ByteSpan | None:
    hyphen = field_value.find("-", start, stop)
    if hyphen < 0 or field_value.find("-", hyphen + 1, stop) >= 0:
        raise MalformedByteRangeError("each member must contain one hyphen")

    if hyphen == start:
        suffix = _decimal(field_value, hyphen + 1, stop)
        if suffix == 0:
            return None
        return ByteSpan(max(0, length - suffix), length)

    first = _decimal(field_value, start, hyphen)
    if hyphen + 1 == stop:
        if first >= length:
            return None
        return ByteSpan(first, length)

    last = _decimal(field_value, hyphen + 1, stop)
    if first > last:
        raise MalformedByteRangeError("a first position must not exceed its last position")
    if first >= length:
        return None
    return ByteSpan(first, min(last, length - 1) + 1)


def _coalesce(spans: list[ByteSpan]) -> tuple[ByteSpan, ...]:
    ordered = sorted(spans)
    merged: list[ByteSpan] = []
    current_start = ordered[0].start
    current_stop = ordered[0].stop

    for span in ordered[1:]:
        if span.start <= current_stop:
            current_stop = max(current_stop, span.stop)
            continue
        merged.append(ByteSpan(current_start, current_stop))
        current_start = span.start
        current_stop = span.stop

    merged.append(ByteSpan(current_start, current_stop))
    return tuple(merged)


def normalize_http_byte_ranges(
    field_value: str,
    *,
    length: int,
    selected_byte_budget: int,
) -> NormalizedByteRanges:
    length = _positive_int64(length, name="length")
    selected_byte_budget = _positive_int64(
        selected_byte_budget,
        name="selected_byte_budget",
    )
    if type(field_value) is not str:
        raise TypeError("field_value must be exact text")
    if len(field_value) > _MAX_FIELD_BYTES:
        raise MalformedByteRangeError("field_value exceeds the supported byte limit")
    if any(not 0x20 <= ord(character) <= 0x7E for character in field_value):
        raise MalformedByteRangeError("field_value must contain printable ASCII")
    if " " in field_value:
        raise MalformedByteRangeError("whitespace is not accepted")
    if not field_value.startswith(_PREFIX):
        raise MalformedByteRangeError("field_value must start with lowercase bytes=")

    spans: list[ByteSpan] = []
    member_count = 0
    position = len(_PREFIX)

    while True:
        member_count += 1
        if member_count > _MAX_MEMBERS:
            raise MalformedByteRangeError("field_value contains too many members")
        member_stop = field_value.find(",", position)
        is_last = member_stop < 0
        if is_last:
            member_stop = len(field_value)
        if member_stop == position:
            raise MalformedByteRangeError("range members must not be empty")

        span = _resolve_member(field_value, position, member_stop, length=length)
        if span is not None:
            spans.append(span)
        if is_last:
            break
        position = member_stop + 1

    if not spans:
        raise UnsatisfiableByteRangeError("no range member selects representation bytes")

    canonical = _coalesce(spans)
    total = sum(span.stop - span.start for span in canonical)
    if total > selected_byte_budget:
        raise ByteRangeBudgetError("canonical range union exceeds selected_byte_budget")
    return NormalizedByteRanges(canonical, total)
```

## Example

```python
normalized = normalize_http_byte_ranges(
    "bytes=8-12,0-2,3-4,-2",
    length=10,
    selected_byte_budget=7,
)
reordered = normalize_http_byte_ranges(
    "bytes=-2,3-4,0-2,8-12",
    length=10,
    selected_byte_budget=7,
)
edge = normalize_http_byte_ranges(
    "bytes=9223372036854775806-9223372036854775807",
    length=_MAX_INT64,
    selected_byte_budget=1,
)


def error_type(field_value: str, *, budget: int = 10) -> type[ByteRangeError]:
    try:
        normalize_http_byte_ranges(
            field_value,
            length=10,
            selected_byte_budget=budget,
        )
    except ByteRangeError as error:
        return type(error)
    raise AssertionError("the example expected a byte-range error")


assert (
    normalized,
    reordered == normalized,
    edge,
    error_type("bytes=4-3"),
    error_type("bytes=-0,10-"),
    error_type("bytes=0-9", budget=9),
) == (
    NormalizedByteRanges((ByteSpan(0, 5), ByteSpan(8, 10)), 7),
    True,
    NormalizedByteRanges((ByteSpan(_MAX_INT64 - 1, _MAX_INT64),), 1),
    MalformedByteRangeError,
    UnsatisfiableByteRangeError,
    ByteRangeBudgetError,
)
```

## Trade-offs and Limitations

Parsing `P` field characters and resolving at most `r = 16` members takes
`O(P + r log r)` time; the resolved and canonical span lists use `O(r)`
auxiliary memory. Decimal length and signed-64 checks happen before integer
conversion. A satisfiable closed member resolves as
`[first, min(last, length - 1) + 1)`, so the implementation never evaluates an
overflowing raw `last + 1`. Open and positive suffix members resolve as
`[first, length)` and `[max(0, length - suffix), length)` respectively.

This is a closed canonical union profile, not a complete implementation of HTTP
range processing. It rejects zero-length representations, whitespace,
noncanonical decimal spellings, other range units, and more than 16 members.
Suffix zero and starts at or beyond the known length are valid but
unsatisfiable; malformed members still reject the complete field. Sorting and
coalescing intentionally discard original member order, multiplicity, and
multipart boundaries. The selected-byte budget is applied only after that
canonical union. Method and precondition handling, `If-Range`, status choice,
response headers, and multipart rendering remain outside this function.

## Related Snippets

<!-- catalog:related:start -->
- [Resume a Bounded HTTP Byte Stream with Validated Range Responses](resume-a-bounded-http-byte-stream-with-validated-range-responses.md)
- [Parse One Bounded Printable-ASCII HTTP/1 Field Section with Exact CRLF Framing](parse-one-bounded-printable-ascii-http-1-field-section-with-exact-crlf-framing.md)
- [Coalesce Bounded Half-Open Integer Intervals Under an Explicit Adjacency Policy](../algorithms-data-structures/coalesce-bounded-half-open-integer-intervals-under-an-explicit-adjacency-policy.md)
<!-- catalog:related:end -->
