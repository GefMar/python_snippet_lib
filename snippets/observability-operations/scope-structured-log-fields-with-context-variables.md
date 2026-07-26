---
title: "Scope Structured Log Fields with Context Variables"
snippet_type: pattern
use_cases:
  - concurrency-control
  - observability
tested_python:
  - "3.14"
dependencies: []
related: []
---

# Scope Structured Log Fields with Context Variables

## Idea and Problem

Attach nested structured fields to log events without passing the same identifiers through every function call.

A module-level `ContextVar` keeps a tuple of shallow, read-only field-name
mappings. Entering a scope appends one layer, inner fields override matching
outer fields, and the token returned by `set()` restores the exact previous
state. Context variables also isolate independently created asynchronous tasks
while preserving the context inherited when each task is created.

## When to Use

Use this pattern when request, operation, or job identifiers should accompany
many log messages across synchronous and asynchronous call stacks. The logging
adapter or formatter can call `current_log_fields()` immediately before it
emits a record. Keep the context small, bounded, safe to disclose, and
serializable by the chosen log format. Supplied values remain caller-owned and
must stay stable for the lifetime of the scope; immutable scalar values are the
safest default.

## Implementation

```python
from collections.abc import Iterator, Mapping
from contextlib import contextmanager
from contextvars import ContextVar
from types import MappingProxyType


LogLayer = Mapping[str, object]
_LOG_LAYERS: ContextVar[tuple[LogLayer, ...]] = ContextVar(
    "structured_log_layers",
    default=(),
)


@contextmanager
def log_field_scope(**fields: object) -> Iterator[None]:
    layer = MappingProxyType(dict(fields))
    token = _LOG_LAYERS.set((*_LOG_LAYERS.get(), layer))
    try:
        yield
    finally:
        _LOG_LAYERS.reset(token)


def current_log_fields() -> dict[str, object]:
    merged: dict[str, object] = {}
    for layer in _LOG_LAYERS.get():
        merged.update(layer)
    return merged
```

## Example

```python
import asyncio


async def capture_fields(request_id: str) -> tuple[dict[str, object], ...]:
    with log_field_scope(request_id=request_id, component="api"):
        await asyncio.sleep(0)
        outer = current_log_fields()
        try:
            with log_field_scope(component="storage", attempt=1):
                nested = current_log_fields()
                raise LookupError("simulated failure")
        except LookupError:
            restored_after_error = current_log_fields()
    empty_after_exit = current_log_fields()
    return outer, nested, restored_after_error, empty_after_exit


async def run_example() -> tuple[tuple[dict[str, object], ...], ...]:
    return tuple(
        await asyncio.gather(
            capture_fields("request-a"),
            capture_fields("request-b"),
        )
    )


captured = asyncio.run(run_example())

assert captured == (
    (
        {"request_id": "request-a", "component": "api"},
        {"request_id": "request-a", "component": "storage", "attempt": 1},
        {"request_id": "request-a", "component": "api"},
        {},
    ),
    (
        {"request_id": "request-b", "component": "api"},
        {"request_id": "request-b", "component": "storage", "attempt": 1},
        {"request_id": "request-b", "component": "api"},
        {},
    ),
)
```

## Trade-offs and Limitations

Context variables make dependencies less visible than explicit parameters, so
reserve them for cross-cutting telemetry rather than business data. New tasks
inherit the context present at task creation, which may be surprising when a
task intentionally outlives its parent operation. Only the field-name mapping
is copied and frozen: nested or otherwise mutable values are not copied, and a
caller mutation remains visible to later reads. Values are not validated or
serialized here; secrets, personal data, large objects, and uncontrolled
high-cardinality fields must stay out. Native threads do not automatically
inherit another thread's context.

## Related Snippets

<!-- catalog:related:start -->
No related snippets.
<!-- catalog:related:end -->
