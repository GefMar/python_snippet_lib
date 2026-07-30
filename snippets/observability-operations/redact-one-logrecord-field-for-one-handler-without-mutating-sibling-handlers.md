---
title: "Redact One LogRecord Field for One Handler Without Mutating Sibling Handlers"
snippet_type: pattern
use_cases:
  - observability
  - security
tested_python:
  - "3.14"
dependencies: []
related:
  - format-log-records-as-json-with-explicit-extra-fields.md
  - scope-structured-log-fields-with-context-variables.md
  - ../security-privacy/redact-explicit-paths-in-bounded-json-like-data.md
---

# Redact One LogRecord Field for One Handler Without Mutating Sibling Handlers

## Idea and Problem

Replace one explicit logging extra value only for the handler that must emit a redacted view.

Since Python 3.12, a handler filter may return a replacement `LogRecord`.
Shallow-copying the record before replacing the complete field value keeps the
original record unchanged, so another handler can still apply its own policy.
Returning the original decision when the field is absent avoids copying every
record.

## When to Use

Use this pattern when one application-controlled handler may expose an
explicit extra field less widely than another handler. Configure the filter on
the handler, not on the logger, and choose a reserved marker that contains no
original value. Apply an appropriate redaction policy independently to every
handler or downstream sink that must not receive the original data.

Use structured transformation before logging when secrets may be nested,
embedded in text, spread across several fields, or hidden inside `msg`,
`args`, or exception data. Prefer never adding secret values to a log record
when no destination needs them.

## Implementation

```python
import copy
import logging

_MAX_RECORD_ATTRIBUTES = 128
_MAX_FIELD_NAME_BYTES = 64
_MAX_MARKER_BYTES = 128
_RESERVED_RECORD_FIELDS = frozenset(logging.makeLogRecord({}).__dict__) | {
    "asctime",
    "message",
}


class RedactExtraField(logging.Filter):
    def __init__(self, field_name: str, marker: str = "[redacted]") -> None:
        super().__init__()
        if type(field_name) is not str:
            raise TypeError("field_name must be an exact string")
        if not field_name.isascii() or not field_name.isidentifier():
            raise ValueError("field_name must be a non-empty ASCII identifier")
        if len(field_name.encode("ascii")) > _MAX_FIELD_NAME_BYTES:
            raise ValueError("field_name exceeds the byte limit")
        if field_name in _RESERVED_RECORD_FIELDS:
            raise ValueError("field_name is reserved by logging")

        if type(marker) is not str:
            raise TypeError("marker must be an exact string")
        marker_size = len(marker.encode("utf-8"))
        if not 1 <= marker_size <= _MAX_MARKER_BYTES:
            raise ValueError("marker length is outside the byte limits")

        self._field_name = field_name
        self._marker = marker

    def filter(
        self,
        record: logging.LogRecord,
    ) -> logging.LogRecord | bool:
        if len(record.__dict__) > _MAX_RECORD_ATTRIBUTES:
            raise ValueError("record has too many attributes")
        if self._field_name not in record.__dict__:
            return True

        replacement = copy.copy(record)
        replacement.__dict__[self._field_name] = self._marker
        return replacement
```

## Example

```python
class FieldCapture(logging.Handler):
    def __init__(self, field_name: str) -> None:
        super().__init__()
        self.field_name = field_name
        self.values: list[object] = []

    def emit(self, record: logging.LogRecord) -> None:
        self.values.append(record.__dict__[self.field_name])


redacted_handler = FieldCapture("customer_reference")
redacted_handler.addFilter(RedactExtraField("customer_reference", "<hidden>"))
private_handler = FieldCapture("customer_reference")

record = logging.makeLogRecord(
    {
        "name": "example.worker",
        "levelno": logging.INFO,
        "levelname": "INFO",
        "msg": "completed",
        "args": (),
        "customer_reference": "reference-42",
    }
)
before = dict(record.__dict__)

redacted_handler.handle(record)
private_handler.handle(record)

assert redacted_handler.values == ["<hidden>"]
assert private_handler.values == ["reference-42"]
assert record.__dict__ == before

missing = logging.makeLogRecord({"msg": "no configured extra", "args": ()})
assert RedactExtraField("customer_reference").filter(missing) is True

rejected_names = 0
for invalid_name in ("msg", "not-ascii-\u00e9", ""):
    try:
        RedactExtraField(invalid_name)
    except ValueError:
        rejected_names += 1
    else:
        raise AssertionError("invalid field name was accepted")

assert rejected_names == 3
```

## Trade-offs and Limitations

Filtering takes `O(A)` time to count and, when the field is present, shallowly
copy the record's `A` attributes. The replacement record also uses `O(A)`
temporary space. The 128-attribute cap bounds that work, but the values behind
all other attributes remain shared. This is whole-value replacement, not a
recursive copy, substring scrubber, regular-expression filter, or detector of
secret material.

Handler-local replacement deliberately means sibling handlers still receive
the original value. It does not protect records processed by logger filters,
queues, formatters, other handlers, fallback handlers, or external collectors.
The configured field must be an application-controlled ASCII identifier and
must not be a standard `LogRecord` field such as `msg`, `args`, or `exc_info`.
Exceptions raised by the filter are configuration or contract failures; this
pattern does not silently drop malformed records.

## Related Snippets

<!-- catalog:related:start -->
- [Format Log Records as JSON with Explicit Extra Fields](format-log-records-as-json-with-explicit-extra-fields.md)
- [Scope Structured Log Fields with Context Variables](scope-structured-log-fields-with-context-variables.md)
- [Redact Explicit Paths in Bounded JSON-Like Data](../security-privacy/redact-explicit-paths-in-bounded-json-like-data.md)
<!-- catalog:related:end -->
