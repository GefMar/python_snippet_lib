---
title: "Format Log Records as JSON with Explicit Extra Fields"
snippet_type: recipe
use_cases:
  - observability
  - serialization
tested_python:
  - "3.14"
dependencies: []
related:
  - scope-structured-log-fields-with-context-variables.md
  - ../data-processing/split-quoted-and-bracketed-log-fields.md
---

# Format Log Records as JSON with Explicit Extra Fields

## Idea and Problem

Serialize each log record to one strict JSON object while keeping caller-supplied extra fields separate from standard logging metadata.

A neutral schema makes the formatted message, source metadata, UTC timestamp,
and exception text predictable. Unknown record attributes become explicit
structured fields, while unsupported values fail visibly instead of being
silently converted to strings.

## When to Use

Use this formatter when a log collector expects JSON Lines and applications
attach already-redacted, JSON-compatible context with `extra`. Define the
schema at the application boundary and configure the formatter on a handler.
Use a platform-specific formatter when a collector requires ECS,
OpenTelemetry, or another prescribed event schema.

## Implementation

```python
import json
import logging
import math
from datetime import datetime, timezone


_STANDARD_RECORD_FIELDS = frozenset(logging.makeLogRecord({}).__dict__) | {
    "asctime",
    "message",
}


def _validate_json_value(
    value: object,
    *,
    path: str,
    active_containers: set[int] | None = None,
) -> None:
    if value is None or isinstance(value, (str, bool, int)):
        return
    if isinstance(value, float):
        if not math.isfinite(value):
            raise ValueError(f"{path} must contain only finite numbers")
        return

    if active_containers is None:
        active_containers = set()
    if isinstance(value, (list, dict)):
        identity = id(value)
        if identity in active_containers:
            raise ValueError(f"{path} contains a reference cycle")
        active_containers.add(identity)
        try:
            if isinstance(value, list):
                for index, item in enumerate(value):
                    _validate_json_value(
                        item,
                        path=f"{path}[{index}]",
                        active_containers=active_containers,
                    )
            else:
                for key, item in value.items():
                    if not isinstance(key, str):
                        raise TypeError(f"{path} object keys must be strings")
                    _validate_json_value(
                        item,
                        path=f"{path}.{key}",
                        active_containers=active_containers,
                    )
        finally:
            active_containers.remove(identity)
        return

    raise TypeError(f"{path} contains a value outside the JSON data model")


class StrictJsonFormatter(logging.Formatter):
    def format(self, record: logging.LogRecord) -> str:
        fields = {}
        for name, value in record.__dict__.items():
            if name in _STANDARD_RECORD_FIELDS:
                continue
            if not isinstance(name, str):
                raise TypeError("extra field names must be strings")
            _validate_json_value(value, path=f"fields.{name}")
            fields[name] = value

        timestamp = (
            datetime.fromtimestamp(record.created, timezone.utc)
            .isoformat(timespec="milliseconds")
            .replace("+00:00", "Z")
        )
        payload = {
            "timestamp": timestamp,
            "level": record.levelname,
            "logger": record.name,
            "message": record.getMessage(),
            "source": {
                "module": record.module,
                "function": record.funcName,
                "line": record.lineno,
            },
            "process": record.process,
            "thread": record.thread,
            "fields": fields,
        }

        if record.exc_info is not None:
            payload["exception"] = self.formatException(record.exc_info)
        elif record.exc_text:
            payload["exception"] = record.exc_text

        return json.dumps(
            payload,
            ensure_ascii=False,
            allow_nan=False,
            separators=(",", ":"),
            sort_keys=True,
        )
```

## Example

```python
import sys


formatter = StrictJsonFormatter()
record = logging.LogRecord(
    "catalog.worker",
    logging.INFO,
    "/tmp/worker.py",
    17,
    "processed %s",
    ("café",),
    None,
    "run",
)
record.created = 0.0
record.request_id = "request-7"
before = dict(record.__dict__)
payload = json.loads(formatter.format(record))

try:
    1 / 0
except ZeroDivisionError:
    exception_record = logging.LogRecord(
        "catalog.worker",
        logging.ERROR,
        "/tmp/worker.py",
        23,
        "failed",
        (),
        sys.exc_info(),
        "run",
    )

exception_payload = json.loads(formatter.format(exception_record))

invalid_extras = (object(), {1: "integer key"})
rejected_extra_count = 0
for invalid_extra in invalid_extras:
    unsupported = logging.makeLogRecord({"msg": "bad extra", "args": ()})
    unsupported.context = invalid_extra
    try:
        formatter.format(unsupported)
    except TypeError:
        rejected_extra_count += 1

assert (
    payload["timestamp"],
    payload["message"],
    payload["fields"],
    before == record.__dict__,
    "ZeroDivisionError" in exception_payload["exception"],
    rejected_extra_count,
) == (
    "1970-01-01T00:00:00.000Z",
    "processed café",
    {"request_id": "request-7"},
    True,
    True,
    2,
)
```

## Trade-offs and Limitations

All extra values are recursively checked against the JSON data model: object
keys must be strings, arrays must be lists, and unsupported objects or
non-finite floats raise instead of being coerced. This surfaces schema drift
but can also make a handler report a formatting error, so validate event fields
before production rollout. The formatter performs no redaction,
sampling, buffering, size limiting, or transport. Standard logging rejects
`extra` names that collide with reserved record attributes before formatting,
and this schema is intentionally not a substitute for a collector-specific
standard.

## Related Snippets

<!-- catalog:related:start -->
- [Scope Structured Log Fields with Context Variables](scope-structured-log-fields-with-context-variables.md)
- [Split Quoted and Bracketed Log Fields](../data-processing/split-quoted-and-bracketed-log-fields.md)
<!-- catalog:related:end -->
