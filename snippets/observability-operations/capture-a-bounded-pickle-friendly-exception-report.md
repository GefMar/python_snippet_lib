---
title: "Capture a Bounded Pickle-Friendly Exception Report"
snippet_type: recipe
use_cases:
  - interoperability
  - observability
  - serialization
tested_python:
  - "3.14"
dependencies: []
related:
  - format-log-records-as-json-with-explicit-extra-fields.md
  - ../concurrency-lifecycle/collect-thread-pool-results-and-errors-as-futures-complete.md
  - ../security-privacy/redact-a-printable-ascii-secret-with-a-bounded-visible-tail.md
---

# Capture a Bounded Pickle-Friendly Exception Report

## Idea and Problem

Flatten one exception and its native cause or context chain into bounded immutable diagnostic data instead of trying to serialize live exception state.

`TracebackException` can format ordinary chains and exception groups without
retaining frames. Capturing with locals disabled, bounding the combined public
text, and storing only strings plus explicit status flags produces a report that can cross a
trusted same-environment process boundary without custom exception pickling.

## When to Use

Use this report immediately inside a worker or task exception boundary when a
parent process needs readable diagnostics but cannot receive live traceback
objects. Keep the transport trusted and short-lived, and apply a domain-specific
redaction policy before wider publication. Use structured error codes or a
JSON-compatible schema for untrusted, cross-language, or long-term storage.

## Implementation

```python
import traceback
from dataclasses import dataclass


_MAX_FRAME_LIMIT = 100
_MAX_REPORT_CHARACTERS = 65_536
_MAX_MESSAGE_CHARACTERS = 2048
_TRUNCATION_MARKER = "\n... [truncated]\n"


@dataclass(frozen=True, slots=True)
class ExceptionReport:
    exception_type: str
    message: str
    traceback_text: str
    text_truncated: bool
    formatting_error_caught: bool


def _truncate_text(text: str, limit: int) -> tuple[str, bool]:
    if len(text) <= limit:
        return text, False
    retained = limit - len(_TRUNCATION_MARKER)
    return text[:retained] + _TRUNCATION_MARKER, True


def _safe_exception_message(error: BaseException) -> tuple[str, bool]:
    try:
        return str(error), False
    except Exception as formatting_error:
        return (
            "<message formatting raised "
            f"{type(formatting_error).__name__}>"
        ), True


def capture_exception_report(
    error: BaseException,
    *,
    max_frames: int = 20,
    max_characters: int = 16_000,
) -> ExceptionReport:
    if not isinstance(error, BaseException):
        raise TypeError("error must be an exception instance")
    if isinstance(max_frames, bool) or not isinstance(max_frames, int):
        raise TypeError("max_frames must be an integer")
    if not 1 <= max_frames <= _MAX_FRAME_LIMIT:
        raise ValueError("max_frames is outside the supported range")
    if isinstance(max_characters, bool) or not isinstance(max_characters, int):
        raise TypeError("max_characters must be an integer")
    if not 256 <= max_characters <= _MAX_REPORT_CHARACTERS:
        raise ValueError("max_characters is outside the supported range")

    qualified_type = (
        f"{type(error).__module__}.{type(error).__qualname__}"
    )
    type_limit = min(256, max_characters // 4)
    exception_type, type_truncated = _truncate_text(
        qualified_type,
        type_limit,
    )
    message_limit = min(_MAX_MESSAGE_CHARACTERS, max_characters // 4)
    raw_message, message_formatting_failed = _safe_exception_message(error)
    message, message_truncated = _truncate_text(
        raw_message,
        message_limit,
    )

    traceback_formatting_failed = False
    try:
        snapshot = traceback.TracebackException.from_exception(
            error,
            limit=-max_frames,
            lookup_lines=False,
            capture_locals=False,
            compact=True,
            max_group_width=10,
            max_group_depth=5,
        )
        rendered_traceback = "".join(snapshot.format(chain=True))
    except Exception as formatting_error:
        traceback_formatting_failed = True
        rendered_traceback = (
            "<traceback formatting raised "
            f"{type(formatting_error).__name__}>\n"
        )
    limit_notice = (
        f"[limits: last {max_frames} frames per stack; "
        "exception-group width 10, depth 5]\n"
    )
    traceback_budget = max_characters - len(exception_type) - len(message)
    traceback_text, traceback_truncated = _truncate_text(
        limit_notice + rendered_traceback,
        traceback_budget,
    )

    return ExceptionReport(
        exception_type=exception_type,
        message=message,
        traceback_text=traceback_text,
        text_truncated=(
            type_truncated
            or message_truncated
            or traceback_truncated
        ),
        formatting_error_caught=(
            message_formatting_failed or traceback_formatting_failed
        ),
    )
```

## Example

```python
import pickle


def fail_with_context() -> None:
    private_value = "local-value-must-not-be-captured"
    try:
        int("not-an-integer")
    except ValueError as cause:
        if private_value:
            raise RuntimeError("conversion failed") from cause


try:
    fail_with_context()
except RuntimeError as error:
    report = capture_exception_report(
        error,
        max_frames=8,
        max_characters=4000,
    )
else:
    raise AssertionError("expected the example failure")

restored = pickle.loads(pickle.dumps(report))

assert (
    restored == report,
    "ValueError" in report.traceback_text,
    "RuntimeError" in report.traceback_text,
    "local-value-must-not-be-captured" not in report.traceback_text,
    sum(
        len(value)
        for value in (
            report.exception_type,
            report.message,
            report.traceback_text,
        )
    ) <= 4000,
) == (True, True, True, True, True)
```

## Trade-offs and Limitations

The report cannot restore a live traceback or re-raise the original exception.
Formatting can expose exception messages, source lines, module names, and file
paths even though local variable values are disabled; generic redaction cannot
infer which of those strings are sensitive. The frame limit keeps the most
recent frames of each formatted stack, while exception-group width and depth
use the fixed bounds shown above; every report includes a notice describing
those structural limits. `text_truncated` reports only character-budget
truncation, while `formatting_error_caught` reports an ordinary formatting
exception caught by this helper. The standard-library formatter can replace a
failure in a nested exception message with its own placeholder without exposing
that failure to the helper, so the flag can remain false in that case.
Structural limits can still omit diagnostics when `text_truncated` is false.
Formatting text may change across Python versions, and character truncation can
remove the most useful line. Pickle is Python-specific, requires the report
class to be importable in the receiving environment, and must never be loaded from an
untrusted or tampered source because unpickling can execute code.

## Related Snippets

<!-- catalog:related:start -->
- [Format Log Records as JSON with Explicit Extra Fields](format-log-records-as-json-with-explicit-extra-fields.md)
- [Collect Thread-Pool Results and Errors as Futures Complete](../concurrency-lifecycle/collect-thread-pool-results-and-errors-as-futures-complete.md)
- [Redact a Printable ASCII Secret with a Bounded Visible Tail](../security-privacy/redact-a-printable-ascii-secret-with-a-bounded-visible-tail.md)
<!-- catalog:related:end -->
