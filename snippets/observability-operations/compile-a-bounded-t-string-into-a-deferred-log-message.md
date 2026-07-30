---
title: "Compile a Bounded T-String into a Deferred Log Message"
snippet_type: pattern
use_cases:
  - observability
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - format-log-records-as-json-with-explicit-extra-fields.md
  - scope-structured-log-fields-with-context-variables.md
  - process-log-records-in-a-background-thread-with-queuelistener.md
---

# Compile a Bounded T-String into a Deferred Log Message

## Idea and Problem

Compile a trusted t-string into an immutable percent-style logging message whose values are formatted only when the logging system renders the record.

A t-string separates its static strings from captured interpolation values.
Escaping percent signs in the static strings and replacing each interpolation
with `%s` produces the `msg` and `args` pair expected by Python logging. A
literal-only template keeps an empty argument tuple, so its percent signs must
remain unchanged.

## When to Use

Use this pattern on Python 3.14 or newer when application-owned logging calls
need a small, closed set of scalar values and should avoid formatting those
values before a log record is rendered. Pass the returned format string and
arguments separately to a logger, for example with
`logger.info(message.format_string, *message.arguments)`.

Use a structured logging adapter when downstream systems need named fields or
typed values. Apply separate redaction and line-sanitization policies before
logging when values may contain secrets or control characters.

## Implementation

```python
import logging
import math
from dataclasses import dataclass
from string.templatelib import Interpolation, Template

_MAX_INTERPOLATIONS = 32
_MAX_STATIC_BYTES = 16 * 1_024
_MAX_STRING_BYTES = 1_024
_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1


@dataclass(frozen=True, slots=True)
class DeferredLogMessage:
    format_string: str
    arguments: tuple[object, ...]


def _utf8_size(value: str, *, name: str) -> int:
    try:
        return len(value.encode("utf-8"))
    except UnicodeEncodeError as error:
        raise ValueError(f"{name} must not contain Unicode surrogates") from error


def _validate_log_value(value: object) -> object:
    if value is None or type(value) is bool:
        return value
    if type(value) is int:
        if not _MIN_INT64 <= value <= _MAX_INT64:
            raise ValueError("integer interpolation is outside the signed 64-bit range")
        return value
    if type(value) is float:
        if not math.isfinite(value):
            raise ValueError("float interpolation must be finite")
        return value
    if type(value) is str:
        if _utf8_size(value, name="string interpolation") > _MAX_STRING_BYTES:
            raise ValueError("string interpolation exceeds the UTF-8 byte limit")
        return value
    raise TypeError("interpolations must contain supported exact scalar values")


def compile_deferred_log_message(template: Template) -> DeferredLogMessage:
    if type(template) is not Template:
        raise TypeError("template must be an exact string.templatelib.Template")

    has_interpolations = bool(template.interpolations)
    format_parts: list[str] = []
    arguments: list[object] = []
    static_bytes = 0

    for part in template:
        if type(part) is str:
            static_bytes += _utf8_size(part, name="static message")
            if static_bytes > _MAX_STATIC_BYTES:
                raise ValueError("static message exceeds the UTF-8 byte limit")
            format_parts.append(part.replace("%", "%%") if has_interpolations else part)
            continue

        if type(part) is not Interpolation:
            raise TypeError("template contains an unsupported exact part type")
        if len(arguments) >= _MAX_INTERPOLATIONS:
            raise ValueError("template exceeds the interpolation count limit")
        if part.conversion is not None:
            raise ValueError("interpolation conversions are not supported")
        if part.format_spec != "":
            raise ValueError("interpolation format specifications are not supported")

        arguments.append(_validate_log_value(part.value))
        format_parts.append("%s")

    return DeferredLogMessage("".join(format_parts), tuple(arguments))


```

## Example

```python
customer = "Ada"
attempts = 3
message = compile_deferred_log_message(t"processed 100% for {customer} after {attempts} attempts")
record = logging.LogRecord(
    "example.worker",
    logging.INFO,
    __file__,
    1,
    message.format_string,
    message.arguments,
    None,
)

literal = compile_deferred_log_message(t"utilization is 100%")
literal_record = logging.LogRecord(
    "example.worker",
    logging.INFO,
    __file__,
    2,
    literal.format_string,
    literal.arguments,
    None,
)

assert message.format_string == "processed 100%% for %s after %s attempts"
assert message.arguments == ("Ada", 3)
assert record.getMessage() == "processed 100% for Ada after 3 attempts"
assert literal.format_string == "utilization is 100%"
assert literal.arguments == ()
assert literal_record.getMessage() == "utilization is 100%"
```

## Trade-offs and Limitations

For `S` static UTF-8 bytes and `n` interpolations, compilation takes
`O(S + n)` time and returns `O(S + n)` state. The static byte, interpolation,
integer, and per-string limits bound that work. Static percent escaping can at
most double the number of percent characters in the format string.

Python evaluates every interpolation expression while it creates the
t-string, before this helper receives the `Template`. Only conversion to text
by logging remains deferred. The result is a conventional logging `msg` and
`args` pair, not structured data, and interpolation expression names are not
retained.

This helper does not redact secrets, sanitize newlines or other control
characters, make untrusted static template text safe, or guarantee that a log
record is emitted. It deliberately rejects conversions, format specifications,
non-finite floats, integer subclasses, and every value outside the closed
scalar profile.

## Related Snippets

<!-- catalog:related:start -->
- [Format Log Records as JSON with Explicit Extra Fields](format-log-records-as-json-with-explicit-extra-fields.md)
- [Scope Structured Log Fields with Context Variables](scope-structured-log-fields-with-context-variables.md)
- [Process Log Records in a Background Thread with QueueListener](process-log-records-in-a-background-thread-with-queuelistener.md)
<!-- catalog:related:end -->
