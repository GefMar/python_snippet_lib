---
title: "Extract Bounded Singleton HTTP Headers with Explicit Rules"
snippet_type: recipe
use_cases:
  - networking
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - parse-a-bounded-ascii-media-type-value.md
  - ../python-language/build-a-read-only-mapping-with-canonical-text-keys.md
  - ../python-language/dispatch-named-strategies-with-an-explicit-function-mapping.md
---

# Extract Bounded Singleton HTTP Headers with Explicit Rules

## Idea and Problem

Extract selected singleton HTTP fields from a bounded raw header tuple only after validating every field and every frozen parsing rule.

A raw tuple retains repeated fields that a dictionary would hide. Rules separate
the case-insensitive wire name from the output name, state whether absence is an
error, and provide the trusted parser for a present value. The returned tuple
is ordered like the rules and uses `None` for an absent optional field.

## When to Use

Use this recipe after an HTTP parser has produced an exact tuple of at most 64
ASCII name-value pairs and a small, fixed set of application fields must be
converted into typed values. Keep the tuple form until selection is complete so
that differently cased occurrences of a selected singleton field remain
detectable.

Create rules at a trusted composition boundary. A parser callback runs only
after the complete rule and raw-header structures, selected duplicates, and
required-field presence have passed validation. Unknown fields are still
syntax-checked but otherwise ignored.

## Implementation

```python
from collections.abc import Callable
from dataclasses import dataclass
from typing import TypeAlias


_MAX_RAW_HEADERS = 64
_MAX_RULES = 16
_MAX_NAME_LENGTH = 64
_MAX_VALUE_LENGTH = 1_024
_HTTP_TOKEN_CHARACTERS = frozenset(
    "!#$%&'*+-.^_`|~0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"
)

HeaderParser: TypeAlias = Callable[[str], object]
RawHeader: TypeAlias = tuple[str, str]


class HeaderSelectionError(ValueError):
    pass


class HeaderParseError(ValueError):
    pass


@dataclass(frozen=True, slots=True)
class SingletonHeaderRule:
    output_name: str
    source_name: str
    required: bool
    parser: HeaderParser


@dataclass(frozen=True, slots=True)
class HeaderResult:
    output_name: str
    value: object | None


def _require_token(value: object, *, field: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be text")
    if not 1 <= len(value) <= _MAX_NAME_LENGTH:
        raise HeaderSelectionError(f"{field} length is outside the supported range")
    if any(character not in _HTTP_TOKEN_CHARACTERS for character in value):
        raise HeaderSelectionError(f"{field} must contain only ASCII HTTP token characters")
    return value


def _require_field_value(value: object, *, index: int) -> str:
    if type(value) is not str:
        raise TypeError(f"raw header {index} value must be text")
    if len(value) > _MAX_VALUE_LENGTH:
        raise HeaderSelectionError(f"raw header {index} value is too long")
    if any(not 0x20 <= ord(character) <= 0x7E for character in value):
        raise HeaderSelectionError(
            f"raw header {index} value must contain only printable ASCII"
        )
    return value


def _validate_rules(rules: object) -> tuple[str, ...]:
    if type(rules) is not tuple:
        raise TypeError("rules must be an exact tuple")
    if not 1 <= len(rules) <= _MAX_RULES:
        raise HeaderSelectionError("rule count is outside the supported range")

    output_names: set[str] = set()
    source_names: set[str] = set()
    normalized_sources: list[str] = []
    for index, rule in enumerate(rules):
        if type(rule) is not SingletonHeaderRule:
            raise TypeError(f"rule {index} must be a SingletonHeaderRule")
        output_name = _require_token(rule.output_name, field=f"rule {index} output name")
        source_name = _require_token(rule.source_name, field=f"rule {index} source name")
        if type(rule.required) is not bool:
            raise TypeError(f"rule {index} required flag must be bool")
        if not callable(rule.parser):
            raise TypeError(f"rule {index} parser must be callable")

        normalized_source = source_name.lower()
        if output_name in output_names:
            raise HeaderSelectionError("rule output names must be unique")
        if normalized_source in source_names:
            raise HeaderSelectionError(
                "rule source names must be unique ignoring ASCII case"
            )
        output_names.add(output_name)
        source_names.add(normalized_source)
        normalized_sources.append(normalized_source)
    return tuple(normalized_sources)


def _collect_selected_values(
    raw_headers: object,
    *,
    selected_sources: frozenset[str],
) -> dict[str, str]:
    if type(raw_headers) is not tuple:
        raise TypeError("raw headers must be an exact tuple")
    if len(raw_headers) > _MAX_RAW_HEADERS:
        raise HeaderSelectionError("raw header count exceeds the supported limit")

    selected_values: dict[str, str] = {}
    for index, pair in enumerate(raw_headers):
        if type(pair) is not tuple:
            raise TypeError(f"raw header {index} must be an exact tuple")
        if len(pair) != 2:
            raise HeaderSelectionError(f"raw header {index} must contain two items")
        source_name = _require_token(pair[0], field=f"raw header {index} name")
        source_value = _require_field_value(pair[1], index=index)
        normalized_source = source_name.lower()
        if normalized_source not in selected_sources:
            continue
        if normalized_source in selected_values:
            raise HeaderSelectionError(
                "a selected singleton header occurs more than once"
            )
        selected_values[normalized_source] = source_value
    return selected_values


def extract_singleton_headers(
    raw_headers: tuple[RawHeader, ...],
    rules: tuple[SingletonHeaderRule, ...],
) -> tuple[HeaderResult, ...]:
    normalized_sources = _validate_rules(rules)
    selected_values = _collect_selected_values(
        raw_headers,
        selected_sources=frozenset(normalized_sources),
    )

    for rule, normalized_source in zip(rules, normalized_sources, strict=True):
        if rule.required and normalized_source not in selected_values:
            raise HeaderSelectionError(
                f"required header {rule.source_name!r} is missing"
            )

    results: list[HeaderResult] = []
    for rule, normalized_source in zip(rules, normalized_sources, strict=True):
        if normalized_source not in selected_values:
            results.append(HeaderResult(rule.output_name, None))
            continue
        try:
            parsed = rule.parser(selected_values[normalized_source])
        except (TypeError, ValueError) as error:
            raise HeaderParseError(
                f"parser failed for output {rule.output_name!r} "
                f"from header {rule.source_name!r}"
            ) from error
        results.append(HeaderResult(rule.output_name, parsed))
    return tuple(results)
```

## Example

```python
parse_calls = []


def parse_limit(value: str) -> int:
    parse_calls.append("limit")
    parsed = int(value)
    if parsed < 0:
        raise ValueError("a non-negative decimal is required")
    return parsed


rules = (
    SingletonHeaderRule("limit", "X-Limit", True, parse_limit),
    SingletonHeaderRule("label", "X-Label", False, str.strip),
    SingletonHeaderRule("checksum", "X-Checksum", False, str.strip),
)
raw_headers = (
    ("x-limit", "12"),
    ("X-LABEL", " compact "),
    ("X-Ignored", "first"),
    ("x-ignored", "second"),
)
selected = extract_singleton_headers(raw_headers, rules)


def rejects_structure(raw: object, candidate_rules: object) -> bool:
    try:
        extract_singleton_headers(raw, candidate_rules)
    except (TypeError, HeaderSelectionError):
        return True
    return False


required_missing = rejects_structure((), rules)
selected_duplicate = rejects_structure(
    (("X-Limit", "1"), ("x-limit", "2")),
    rules,
)
invalid_raw_cases = (
    ((b"X-Limit", "1"),),
    (("X-Limit", b"1"),),
    (("X-Limit", "line\r\nbreak"),),
    (("X-Limit", "x" * 1_025),),
    (("X" * 65, "1"),),
    (("X-Noise", "ok"),) * 65,
)
invalid_raw_rejections = sum(
    rejects_structure(candidate, rules) for candidate in invalid_raw_cases
)

duplicate_rule_sets = (
    (
        SingletonHeaderRule("same", "X-One", False, str.strip),
        SingletonHeaderRule("same", "X-Two", False, str.strip),
    ),
    (
        SingletonHeaderRule("one", "X-Same", False, str.strip),
        SingletonHeaderRule("two", "x-same", False, str.strip),
    ),
)
duplicate_rule_rejections = sum(
    rejects_structure((), candidate) for candidate in duplicate_rule_sets
)

prevalidation_calls = []


def observed_parser(value: str) -> str:
    prevalidation_calls.append(value)
    return value


prevalidation_cases = (
    (
        (("X-First", "one"),),
        (
            SingletonHeaderRule("first", "X-First", False, observed_parser),
            SingletonHeaderRule("second", "X-Second", True, observed_parser),
        ),
    ),
    (
        (("X-First", "one"), ("X-Other", "bad\nvalue")),
        (SingletonHeaderRule("first", "X-First", True, observed_parser),),
    ),
    (
        (("X-First", "one"), ("x-first", "two")),
        (SingletonHeaderRule("first", "X-First", True, observed_parser),),
    ),
)
prevalidation_rejections = sum(
    rejects_structure(candidate_raw, candidate_rules)
    for candidate_raw, candidate_rules in prevalidation_cases
)


def value_failure(value: str) -> object:
    raise ValueError("decimal expected")


def type_failure(value: str) -> object:
    raise TypeError("parser contract mismatch")


parser_failures = []
for parser, cause_type in ((value_failure, ValueError), (type_failure, TypeError)):
    hidden_value = "do-not-report-this-value"
    try:
        extract_singleton_headers(
            (("X-Number", hidden_value),),
            (SingletonHeaderRule("number", "X-Number", True, parser),),
        )
    except HeaderParseError as error:
        parser_failures.append(
            (
                type(error.__cause__) is cause_type,
                hidden_value not in str(error),
                str(error),
            )
        )

assert (
    selected,
    parse_calls,
    required_missing,
    selected_duplicate,
    invalid_raw_rejections,
    duplicate_rule_rejections,
    prevalidation_rejections,
    prevalidation_calls,
    parser_failures,
) == (
    (
        HeaderResult("limit", 12),
        HeaderResult("label", "compact"),
        HeaderResult("checksum", None),
    ),
    ["limit"],
    True,
    True,
    6,
    2,
    3,
    [],
    [
        (True, True, "parser failed for output 'number' from header 'X-Number'"),
        (True, True, "parser failed for output 'number' from header 'X-Number'"),
    ],
)
```

## Trade-offs and Limitations

Validation and selection are linear in the bounded input. Unknown fields do not
appear in the result, but they still consume the header budget and must satisfy
the same conservative ASCII syntax. The recipe does not implement general RFC
field combination; repeated selected fields are rejected even when a particular
HTTP field specification would permit multiple values.

Parser callbacks are trusted code. They can block, mutate external state, run
expensive or unsafe regular expressions, or return mutable objects. The result
tuple and its entries are frozen, but callback-owned return values are not
recursively copied or frozen. The wrapper adds field context when a parser
raises `TypeError` or `ValueError` without placing the raw header value in its
own message; the original exception remains available as the cause.

This is not an HTTP wire parser, authentication mechanism, framework adapter,
or network client. It deliberately does not normalize arbitrary field values,
combine multi-value fields, schedule callbacks, or control callback state.

## Related Snippets

<!-- catalog:related:start -->
- [Parse a Bounded ASCII Media Type Value](parse-a-bounded-ascii-media-type-value.md)
- [Build a Read-Only Mapping with Canonical Text Keys](../python-language/build-a-read-only-mapping-with-canonical-text-keys.md)
- [Dispatch Named Strategies with an Explicit Function Mapping](../python-language/dispatch-named-strategies-with-an-explicit-function-mapping.md)
<!-- catalog:related:end -->
