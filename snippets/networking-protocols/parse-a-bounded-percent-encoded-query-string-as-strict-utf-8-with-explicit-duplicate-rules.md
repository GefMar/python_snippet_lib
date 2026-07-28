---
title: "Parse a Bounded Percent-Encoded Query String as Strict UTF-8 with Explicit Duplicate Rules"
snippet_type: recipe
use_cases:
  - interoperability
  - networking
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - build-a-bounded-http-request-target-from-typed-segments.md
  - ../configuration-serialization/parse-a-bounded-component-options-expression.md
  - ../configuration-serialization/parse-bounded-utf-8-json-without-duplicate-object-names-or-non-finite-numbers.md
---

# Parse a Bounded Percent-Encoded Query String as Strict UTF-8 with Explicit Duplicate Rules

## Idea and Problem

Parse already isolated query-string bytes into an immutable ordered mapping under one strict percent-decoding and duplicate-name contract.

An ampersand separates fields, each field has exactly one literal equals-sign
separator, and percent-encoded equals signs remain component data. Every
percent escape is validated before decoding, plus signs become spaces, and
decoded bytes must form strict UTF-8 before names are compared.

## When to Use

Use this recipe when a boundary receives only the raw query component, without
a leading question mark, and all producers can follow this closed form-style
profile. Choose `REJECT` when every decoded name must occur once, or `COLLECT`
when repeated values are meaningful and their encounter order must be kept.
An empty input represents no fields; blank values use the explicit `name=`
form, while leading, trailing, or adjacent separators are errors.

Use a maintained URL or framework parser when extraction from a complete URL,
scheme-specific behavior, alternate separators, or another query convention
is required. Apply application schema rules only after this byte-to-text
boundary succeeds.

## Implementation

```python
from collections.abc import Iterator, Mapping
from dataclasses import dataclass
from enum import StrEnum
from types import MappingProxyType

_MAX_RAW_QUERY_BYTES = 16_384
_MAX_FIELDS = 128
_MAX_DECODED_NAME_BYTES = 256
_MAX_DECODED_VALUE_BYTES = 4_096
_MAX_DECODED_AGGREGATE_BYTES = 8_192
_HEX_DIGITS = frozenset(b"0123456789ABCDEFabcdef")


class DuplicatePolicy(StrEnum):
    REJECT = "reject"
    COLLECT = "collect"


class QueryProblem(StrEnum):
    RAW_LIMIT_EXCEEDED = "raw_limit_exceeded"
    LEADING_QUESTION_MARK = "leading_question_mark"
    TOO_MANY_FIELDS = "too_many_fields"
    EMPTY_FIELD = "empty_field"
    INVALID_FIELD_SEPARATOR = "invalid_field_separator"
    INVALID_PERCENT_ESCAPE = "invalid_percent_escape"
    NAME_LIMIT_EXCEEDED = "name_limit_exceeded"
    VALUE_LIMIT_EXCEEDED = "value_limit_exceeded"
    AGGREGATE_LIMIT_EXCEEDED = "aggregate_limit_exceeded"
    INVALID_UTF8 = "invalid_utf8"
    EMPTY_NAME = "empty_name"
    DUPLICATE_NAME = "duplicate_name"


class QueryStringError(ValueError):
    def __init__(self, problem: QueryProblem) -> None:
        self.problem = problem
        super().__init__(problem.value)


def _fail(problem: QueryProblem) -> None:
    raise QueryStringError(problem)


@dataclass(frozen=True, slots=True, eq=False)
class QueryParameters(Mapping[str, tuple[str, ...]]):
    _values: Mapping[str, tuple[str, ...]]

    def __init__(self, values: dict[str, list[str]]) -> None:
        frozen = {name: tuple(items) for name, items in values.items()}
        object.__setattr__(self, "_values", MappingProxyType(frozen))

    def __getitem__(self, name: str) -> tuple[str, ...]:
        return self._values[name]

    def __iter__(self) -> Iterator[str]:
        return iter(self._values)

    def __len__(self) -> int:
        return len(self._values)


def _validate_percent_escapes(data: bytes) -> None:
    position = 0
    while True:
        position = data.find(b"%", position)
        if position == -1:
            return
        if (
            position + 2 >= len(data)
            or data[position + 1] not in _HEX_DIGITS
            or data[position + 2] not in _HEX_DIGITS
        ):
            _fail(QueryProblem.INVALID_PERCENT_ESCAPE)
        position += 3


def _decode_component(
    raw: bytes,
    *,
    limit: int,
    limit_problem: QueryProblem,
) -> bytes:
    decoded = bytearray()
    position = 0
    while position < len(raw):
        byte = raw[position]
        if byte == 0x25:
            decoded.append(int(raw[position + 1 : position + 3], 16))
            position += 3
        else:
            decoded.append(0x20 if byte == 0x2B else byte)
            position += 1
        if len(decoded) > limit:
            _fail(limit_problem)
    return bytes(decoded)


def parse_query_string(
    data: bytes,
    *,
    duplicate_policy: DuplicatePolicy,
) -> QueryParameters:
    if type(data) is not bytes:
        raise TypeError("query string must be exact bytes")
    if type(duplicate_policy) is not DuplicatePolicy:
        raise TypeError("duplicate_policy must be an exact DuplicatePolicy")
    if len(data) > _MAX_RAW_QUERY_BYTES:
        _fail(QueryProblem.RAW_LIMIT_EXCEEDED)
    if data.startswith(b"?"):
        _fail(QueryProblem.LEADING_QUESTION_MARK)

    _validate_percent_escapes(data)
    if not data:
        return QueryParameters({})

    fields = data.split(b"&")
    if len(fields) > _MAX_FIELDS:
        _fail(QueryProblem.TOO_MANY_FIELDS)

    values: dict[str, list[str]] = {}
    aggregate_bytes = 0
    for field in fields:
        if not field:
            _fail(QueryProblem.EMPTY_FIELD)
        if field.count(b"=") != 1:
            _fail(QueryProblem.INVALID_FIELD_SEPARATOR)
        raw_name, raw_value = field.split(b"=", 1)

        name_bytes = _decode_component(
            raw_name,
            limit=_MAX_DECODED_NAME_BYTES,
            limit_problem=QueryProblem.NAME_LIMIT_EXCEEDED,
        )
        value_bytes = _decode_component(
            raw_value,
            limit=_MAX_DECODED_VALUE_BYTES,
            limit_problem=QueryProblem.VALUE_LIMIT_EXCEEDED,
        )
        aggregate_bytes += len(name_bytes) + len(value_bytes)
        if aggregate_bytes > _MAX_DECODED_AGGREGATE_BYTES:
            _fail(QueryProblem.AGGREGATE_LIMIT_EXCEEDED)

        try:
            name = name_bytes.decode("utf-8", errors="strict")
            value = value_bytes.decode("utf-8", errors="strict")
        except UnicodeDecodeError:
            _fail(QueryProblem.INVALID_UTF8)
        if not name:
            _fail(QueryProblem.EMPTY_NAME)

        previous = values.get(name)
        if previous is None:
            values[name] = [value]
        elif duplicate_policy is DuplicatePolicy.REJECT:
            _fail(QueryProblem.DUPLICATE_NAME)
        else:
            previous.append(value)

    return QueryParameters(values)
```

## Example

```python
parameters = parse_query_string(
    b"tag=one&empty=&tag=two%3D2&city=M%C3%A9xico+City",
    duplicate_policy=DuplicatePolicy.COLLECT,
)
empty = parse_query_string(b"", duplicate_policy=DuplicatePolicy.REJECT)

invalid_values = (
    (b"?a=1", QueryProblem.LEADING_QUESTION_MARK),
    (b"a", QueryProblem.INVALID_FIELD_SEPARATOR),
    (b"a=b=c", QueryProblem.INVALID_FIELD_SEPARATOR),
    (b"a=%GG", QueryProblem.INVALID_PERCENT_ESCAPE),
    (b"=blank", QueryProblem.EMPTY_NAME),
    (b"x=%FF", QueryProblem.INVALID_UTF8),
    (b"a=1&a=2", QueryProblem.DUPLICATE_NAME),
    (b"a=1&&b=2", QueryProblem.EMPTY_FIELD),
)
problems = []
for value, _expected_problem in invalid_values:
    try:
        parse_query_string(value, duplicate_policy=DuplicatePolicy.REJECT)
    except QueryStringError as error:
        problems.append(error.problem)

assert (
    tuple(parameters),
    parameters["tag"],
    parameters["empty"],
    parameters["city"],
    tuple(empty.items()),
    tuple(problems),
) == (
    ("tag", "empty", "city"),
    ("one", "two=2"),
    ("",),
    ("México City",),
    (),
    tuple(expected for _, expected in invalid_values),
)
```

## Trade-offs and Limitations

This is a closed query-component parser, not a URL parser. It does not locate a
query inside a URL, remove a leading `?`, interpret a fragment, or validate a
scheme or authority. It accepts uppercase and lowercase hex digits and does not
preserve whether a byte came from a literal, a plus sign, or a percent escape.
Decoded names are compared exactly without Unicode normalization or
case-folding. All values remain strings; the function applies no schema,
scalar coercion, parameter allowlist, or application authorization rule. It
materializes the bounded raw input, decoded components, and frozen result in
memory.

## Related Snippets

<!-- catalog:related:start -->
- [Build a Bounded HTTP Request Target from Typed Segments](build-a-bounded-http-request-target-from-typed-segments.md)
- [Parse a Bounded Component Options Expression](../configuration-serialization/parse-a-bounded-component-options-expression.md)
- [Parse Bounded UTF-8 JSON Without Duplicate Object Names or Non-Finite Numbers](../configuration-serialization/parse-bounded-utf-8-json-without-duplicate-object-names-or-non-finite-numbers.md)
<!-- catalog:related:end -->
