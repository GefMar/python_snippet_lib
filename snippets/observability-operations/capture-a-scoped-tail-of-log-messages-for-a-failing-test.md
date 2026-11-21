---
title: "Capture a Scoped Tail of Log Messages for a Failing Test"
snippet_type: testing-technique
use_cases:
  - lifecycle-management
  - observability
  - testing
tested_python:
  - "3.14"
dependencies: []
related:
  - redact-one-logrecord-field-for-one-handler-without-mutating-sibling-handlers.md
  - process-log-records-in-a-background-thread-with-queuelistener.md
---

# Capture a Scoped Tail of Log Messages for a Failing Test

## Idea and Problem

Keep the latest log messages from one test operation available for diagnosis without replacing its existing handlers.

A handler can save each rendered message in a `deque` with a fixed capacity.
When a message arrives at a full tail, the oldest one drops out. Attach the
handler only for the test operation and detach it in `finally`, so the tail
still describes that operation when its code raises.

## When to Use

Use this when a controlled test needs a small amount of recent diagnostic
context, such as the messages before an export fails. Attach it to a logger
whose lifecycle the test controls. The existing logger settings determine
which records reach your handler.

## Implementation

```python
import logging
from collections import deque
from collections.abc import Iterator
from contextlib import contextmanager


class MessageTail(logging.Handler):
    def __init__(self, max_messages: int) -> None:
        if type(max_messages) is not int or max_messages <= 0:
            raise ValueError("max_messages must be a positive integer")
        super().__init__()
        self._messages: deque[str] = deque(maxlen=max_messages)

    def emit(self, record: logging.LogRecord) -> None:
        self._messages.append(record.getMessage())

    def snapshot(self) -> tuple[str, ...]:
        self.acquire()
        try:
            return tuple(self._messages)
        finally:
            self.release()


@contextmanager
def capture_message_tail(logger: logging.Logger, *, max_messages: int) -> Iterator[MessageTail]:
    handler = MessageTail(max_messages)
    logger.addHandler(handler)
    try:
        yield handler
    finally:
        logger.removeHandler(handler)
        handler.close()
```

## Example

```python
from io import StringIO


class ExportFailed(RuntimeError):
    pass


logger = logging.getLogger("example.export-test")
original_level, original_propagate = logger.level, logger.propagate
original_handlers = tuple(logger.handlers)
with StringIO() as normal_output:
    normal_handler = logging.StreamHandler(normal_output)
    logger.setLevel(logging.INFO)
    logger.propagate = False
    logger.addHandler(normal_handler)
    try:
        try:
            with capture_message_tail(logger, max_messages=2) as tail:
                logger.info("started export")
                logger.info("rendered %d rows", 3)
                logger.error("validation failed")
                raise ExportFailed
        except ExportFailed:
            pass
        else:
            raise AssertionError("the test error must propagate")

        logger.info("after export")
        normal_messages = normal_output.getvalue().splitlines()
        assert tuple(logger.handlers) == (*original_handlers, normal_handler)
    finally:
        logger.removeHandler(normal_handler)
        normal_handler.close()
        logger.setLevel(original_level)
        logger.propagate = original_propagate

assert (tail.snapshot(), normal_messages) == (
    ("rendered 3 rows", "validation failed"),
    ["started export", "rendered 3 rows", "validation failed", "after export"],
)
```

## Trade-offs and Limitations

The limit counts messages, so a long message can still use significant
memory. The tail keeps message text without retaining `LogRecord` objects,
timestamps, or exception tracebacks. It assumes valid logging arguments and
does not redact the captured text.

The snapshot uses the handler lock that normal logging also takes around
`emit()`. The scope does not isolate callers or threads that share the
logger. Use a dedicated test logger and finish any producers before leaving
the scope if you need an exact final tail.

## Related Snippets

<!-- catalog:related:start -->
- [Redact One LogRecord Field for One Handler Without Mutating Sibling Handlers](redact-one-logrecord-field-for-one-handler-without-mutating-sibling-handlers.md)
- [Process Log Records in a Background Thread with QueueListener](process-log-records-in-a-background-thread-with-queuelistener.md)
<!-- catalog:related:end -->
