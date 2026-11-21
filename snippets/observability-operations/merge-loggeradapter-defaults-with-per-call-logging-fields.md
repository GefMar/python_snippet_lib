---
title: "Merge LoggerAdapter Defaults with Per-Call Logging Fields"
snippet_type: recipe
use_cases:
  - configuration
  - observability
tested_python:
  - "3.14"
dependencies: []
related:
  - scope-structured-log-fields-with-context-variables.md
  - format-log-records-as-json-with-explicit-extra-fields.md
---

# Merge LoggerAdapter Defaults with Per-Call Logging Fields

## Idea and Problem

Keep shared logging fields on an adapter while adding or overriding fields for one event.

By default, `LoggerAdapter` supplies its own `extra` mapping in place of the
mapping passed to a log call. With `merge_extra=True`, both mappings
contribute fields to the record. The call's value wins when the same key
appears in both.

## When to Use

Use this in Python 3.13 or later when a logger adapter carries stable fields
for a job or operation and individual events need extra detail. Choose
field names the receiving formatter or handler understands and pass a
mapping when providing per-call fields.

## Implementation

```python
import logging

logger = logging.getLogger("example.inventory")
shared_fields = {"job": "inventory", "stage": "prepare"}
worker_log = logging.LoggerAdapter(logger, shared_fields, merge_extra=True)
status_log = logging.LoggerAdapter(logger, shared_fields, merge_extra=True)
```

## Example

```python
from io import StringIO

event_fields = {"stage": "store", "records": 4}
previous_level, previous_propagation = logger.level, logger.propagate
previous_handlers = tuple(logger.handlers)

with StringIO() as output:
    sink = logging.StreamHandler(output)
    sink.setFormatter(
        logging.Formatter("%(job)s %(stage)s %(records)s %(message)s", defaults={"records": "-"})
    )
    logger.setLevel(logging.INFO)
    logger.propagate = False
    logger.addHandler(sink)
    try:
        worker_log.info("stored", extra=event_fields)
        worker_log.info("waiting")
        status_log.info("heartbeat")
        observed = output.getvalue().splitlines()
    finally:
        logger.removeHandler(sink)
        sink.close()
        logger.setLevel(previous_level)
        logger.propagate = previous_propagation

restored = (logger.level, logger.propagate, tuple(logger.handlers))
assert (observed, shared_fields, event_fields, restored) == (
    ["inventory store 4 stored", "inventory prepare - waiting", "inventory prepare - heartbeat"],
    {"job": "inventory", "stage": "prepare"},
    {"stage": "store", "records": 4},
    (previous_level, previous_propagation, previous_handlers),
)
```

## Trade-offs and Limitations

The merge is shallow. An event replaces the entire value for a matching key,
and objects stored inside either mapping can still be shared. Prefer stable
scalar values for diagnostic fields. Reserved `LogRecord` attribute names
such as `levelname` are unavailable as extra keys.

Configure merging on each adapter that should combine fields. The merge
happens when the record is created, so every handler for that event can see
the combined fields. Use a handler-specific formatter when only the display
of a field should differ between destinations.

## Related Snippets

<!-- catalog:related:start -->
- [Scope Structured Log Fields with Context Variables](scope-structured-log-fields-with-context-variables.md)
- [Format Log Records as JSON with Explicit Extra Fields](format-log-records-as-json-with-explicit-extra-fields.md)
<!-- catalog:related:end -->
