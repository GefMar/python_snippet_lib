---
title: "Assert Structured LogRecord Fields Without Matching Rendered Text"
snippet_type: testing-technique
use_cases:
  - observability
  - testing
tested_python:
  - "3.14"
dependencies: []
related:
  - ../observability-operations/capture-a-scoped-tail-of-log-messages-for-a-failing-test.md
  - ../observability-operations/scope-structured-log-fields-with-context-variables.md
---

# Assert Structured LogRecord Fields Without Matching Rendered Text

## Idea and Problem

Assert the fields on captured log records without depending on message wording or formatting.

`unittest.TestCase.assertLogs()` provides both formatted output and a list
of `LogRecord` objects. Check `records` when the contract is a severity or a
structured field passed through `extra`, rather than a rendered string.

## When to Use

Use this when a test controls a dedicated logger and needs to verify a
structured event. For example, check that a retry retains its job identifier
and attempt number even if someone changes the human-readable message.

## Implementation

```python
import logging


def report_retry(logger: logging.Logger, job_id: str, attempt: int) -> None:
    logger.warning("retrying job", extra={"job_id": job_id, "attempt": attempt})
```

## Example

```python
from io import StringIO
from unittest import TestCase

logger = logging.getLogger("snippet.retry-test")
saved_level, saved_propagation = logger.level, logger.propagate
saved_handlers = tuple(logger.handlers)
with StringIO() as output:
    handler = logging.StreamHandler(output)
    logger.addHandler(handler)
    logger.setLevel(logging.INFO)
    logger.propagate = True
    expected_handlers = tuple(logger.handlers)
    try:
        with TestCase().assertLogs(logger, level=logging.WARNING) as captured:
            logger.info("work started")
            report_retry(logger, "sample", 2)

        (record,) = captured.records
        assert (record.levelno, record.__dict__["job_id"], record.__dict__["attempt"]) == (
            logging.WARNING,
            "sample",
            2,
        )
        assert not output.getvalue()
        assert (logger.level, logger.propagate, tuple(logger.handlers)) == (
            logging.INFO,
            True,
            expected_handlers,
        )
    finally:
        logger.setLevel(saved_level)
        logger.propagate = saved_propagation
        logger.removeHandler(handler)
        handler.close()

assert (logger.level, logger.propagate, tuple(logger.handlers), output.closed) == (
    saved_level,
    saved_propagation,
    saved_handlers,
    True,
)
```

## Trade-offs and Limitations

`assertLogs()` requires at least one matching record. It temporarily replaces
the selected logger's handlers, changes its level, and stops propagation.
Use a logger reserved for the test, not the root or a shared logger used by
concurrent tests. This verifies emitted records, not what the regular
handlers would format, filter, or write.

Captured records can retain the objects supplied through `extra`. Keep those
values small and synthetic in tests, and do not rely on capture to redact
them.

## Related Snippets

<!-- catalog:related:start -->
- [Capture a Scoped Tail of Log Messages for a Failing Test](../observability-operations/capture-a-scoped-tail-of-log-messages-for-a-failing-test.md)
- [Scope Structured Log Fields with Context Variables](../observability-operations/scope-structured-log-fields-with-context-variables.md)
<!-- catalog:related:end -->
