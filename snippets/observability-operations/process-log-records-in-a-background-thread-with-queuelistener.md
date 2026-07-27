---
title: "Process Log Records in a Background Thread with QueueListener"
snippet_type: recipe
use_cases:
  - concurrency-control
  - lifecycle-management
  - observability
tested_python:
  - "3.14"
dependencies: []
related:
  - format-log-records-as-json-with-explicit-extra-fields.md
  - scope-structured-log-fields-with-context-variables.md
  - ../concurrency-lifecycle/stop-a-polling-worker-cooperatively-with-an-event.md
---

# Process Log Records in a Background Thread with QueueListener

## Idea and Problem

Move target-handler work from producer threads to one standard-library QueueListener thread with an explicit scoped lifecycle.

`QueueHandler` prepares each record on the producer thread and enqueues it,
while `QueueListener` passes it to the caller-owned target handler. When that
target stays responsive and does not raise, teardown first detaches the producer
handler and then stops the listener, placing its sentinel after every record
already in the FIFO queue so those records are processed before return.

## When to Use

Use this recipe when in-process threads share a target handler whose formatting
or I/O should not run on their hot paths; message preparation still runs there.
Finish and join every producer before leaving the scope, and choose target-handler
levels deliberately. Use a durable logging transport when records must survive
process failure; this queue is only an in-memory handoff and does not acknowledge
storage.

## Implementation

```python
import logging
from collections.abc import Iterator
from contextlib import contextmanager
from logging.handlers import QueueHandler, QueueListener
from queue import SimpleQueue


@contextmanager
def process_logs_in_background(
    logger: logging.Logger,
    target_handler: logging.Handler,
) -> Iterator[None]:
    if not isinstance(logger, logging.Logger):
        raise TypeError("logger must be a logging.Logger")
    if not isinstance(target_handler, logging.Handler):
        raise TypeError("target_handler must be a logging.Handler")

    record_queue: SimpleQueue[logging.LogRecord] = SimpleQueue()
    producer_handler = QueueHandler(record_queue)
    listener = QueueListener(
        record_queue,
        target_handler,
        respect_handler_level=True,
    )

    attached = False
    started = False
    try:
        listener.start()
        started = True
        logger.addHandler(producer_handler)
        attached = True
        yield
    finally:
        if attached:
            logger.removeHandler(producer_handler)
        try:
            if started:
                listener.stop()
        finally:
            producer_handler.close()
```

## Example

```python
import threading


class MessageCollector(logging.Handler):
    def __init__(self) -> None:
        super().__init__(level=logging.INFO)
        self.messages: list[str] = []

    def emit(self, record: logging.LogRecord) -> None:
        self.messages.append(record.getMessage())


logger = logging.Logger("queue-example", level=logging.DEBUG)
logger.propagate = False
collector = MessageCollector()
original_handlers = tuple(logger.handlers)


def produce() -> None:
    logger.debug("filtered")
    logger.info("first")
    logger.warning("second")


with process_logs_in_background(logger, collector):
    producer = threading.Thread(target=produce)
    producer.start()
    producer.join()

try:
    with process_logs_in_background(logger, collector):
        logger.error("before-error")
        raise RuntimeError("body failure")
except RuntimeError:
    body_error_propagated = True
else:
    body_error_propagated = False

assert (
    collector.messages,
    tuple(logger.handlers),
    body_error_propagated,
) == (["first", "second", "before-error"], original_handlers, True)
```

## Trade-offs and Limitations

`SimpleQueue` is unbounded, so a sustained producer/sink imbalance can exhaust
memory. A bounded `Queue` needs a separate, explicit blocking or dropping
policy because the default `QueueHandler` uses a non-blocking enqueue. Listener
shutdown can wait forever for a stuck target, and target exceptions can stop
the listener without reporting failure to producers. Process crashes can lose
queued records, and FIFO enqueue order across threads is not causal event order.
The base handler prepares records before queuing and can remove argument or
exception details needed by specialized downstream formatters. Existing logger
handlers still run normally, and using the same multiprocessing queue for its
own internal logs can recurse or deadlock.

## Related Snippets

<!-- catalog:related:start -->
- [Format Log Records as JSON with Explicit Extra Fields](format-log-records-as-json-with-explicit-extra-fields.md)
- [Scope Structured Log Fields with Context Variables](scope-structured-log-fields-with-context-variables.md)
- [Stop a Polling Worker Cooperatively with an Event](../concurrency-lifecycle/stop-a-polling-worker-cooperatively-with-an-event.md)
<!-- catalog:related:end -->
