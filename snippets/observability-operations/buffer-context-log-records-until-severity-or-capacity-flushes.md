---
title: "Buffer Context Log Records Until Severity or Capacity Flushes"
snippet_type: recipe
use_cases:
  - lifecycle-management
  - observability
tested_python:
  - "3.14"
dependencies: []
related:
  - capture-a-scoped-tail-of-log-messages-for-a-failing-test.md
  - process-log-records-in-a-background-thread-with-queuelistener.md
---

# Buffer Context Log Records Until Severity or Capacity Flushes

## Idea and Problem

Collect recent log records and send them to a handler when an error arrives or the buffer fills.

`MemoryHandler` forwards the buffered records, including the triggering
record, when it reaches the configured severity or record count. Set the
close policy explicitly to decide whether the pending tail should also be
delivered at the end of the operation.

## When to Use

Use this for an operation with a dedicated logger and a caller-owned target
handler. Route events to the target through this buffer. Finish the emitters
before closing the scope, and keep the target open through that close.

## Implementation

```python
import logging
from collections.abc import Iterator
from contextlib import contextmanager
from logging.handlers import MemoryHandler


@contextmanager
def buffer_context_logs(
    logger: logging.Logger,
    target: logging.Handler,
    *,
    capacity: int,
    flush_pending: bool,
) -> Iterator[None]:
    if type(capacity) is not int or capacity <= 0:
        raise ValueError("capacity must be a positive record count")

    handler = MemoryHandler(
        capacity=capacity,
        flushLevel=logging.ERROR,
        target=target,
        flushOnClose=flush_pending,
    )
    logger.addHandler(handler)
    try:
        yield
    finally:
        logger.removeHandler(handler)
        handler.close()
```

## Example

```python
from io import StringIO

logger = logging.getLogger("example.import-buffer")
previous_level, previous_propagation = logger.level, logger.propagate
previous_handlers = tuple(logger.handlers)

with StringIO() as output:
    target = logging.StreamHandler(output)
    logger.setLevel(logging.DEBUG)
    logger.propagate = False
    try:
        with buffer_context_logs(logger, target, capacity=3, flush_pending=False):
            logger.debug("read header")
            before_error = output.getvalue()
            logger.error("invalid field")
            after_error = output.getvalue().splitlines()

            logger.info("queued first")
            logger.info("queued second")
            after_two = output.getvalue().splitlines()
            logger.info("queued third")
            after_full = output.getvalue().splitlines()
            logger.debug("quiet end")
        after_close = output.getvalue().splitlines()

        with buffer_context_logs(logger, target, capacity=3, flush_pending=True):
            logger.info("keep tail")
        after_flush_close = output.getvalue().splitlines()
    finally:
        target.close()
        logger.setLevel(previous_level)
        logger.propagate = previous_propagation

assert (
    before_error == ""
    and after_error == after_two == ["read header", "invalid field"]
    and after_full == after_close == [*after_error, "queued first", "queued second", "queued third"]
    and after_flush_close == [*after_full, "keep tail"]
    and tuple(logger.handlers) == previous_handlers
    and logger.level == previous_level
    and logger.propagate == previous_propagation
)
```

## Trade-offs and Limitations

Capacity counts records. Buffered records can hold large messages and
references to mutable arguments. The flush runs synchronously and forwards
each record directly to the target's `handle()` method. The target's level is
not checked on that path, but its filters still run. Configure the input
logger or buffering handler if low-level records should not enter the buffer.

A full buffer forwards its contents even if every record is below the error
level. With `flush_pending=False`, closing skips delivery of the remaining
records. Use a separate rolling capture when only the newest messages should
be retained for inspection.

## Related Snippets

<!-- catalog:related:start -->
- [Capture a Scoped Tail of Log Messages for a Failing Test](capture-a-scoped-tail-of-log-messages-for-a-failing-test.md)
- [Process Log Records in a Background Thread with QueueListener](process-log-records-in-a-background-thread-with-queuelistener.md)
<!-- catalog:related:end -->
