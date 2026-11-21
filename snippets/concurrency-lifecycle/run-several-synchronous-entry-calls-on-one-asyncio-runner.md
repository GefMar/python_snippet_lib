---
title: "Run Several Synchronous Entry Calls on One asyncio Runner"
snippet_type: recipe
use_cases:
  - interoperability
  - lifecycle-management
tested_python:
  - "3.14"
dependencies: []
related:
  - run-one-async-operation-with-a-bounded-resource-stack.md
  - submit-a-callable-with-a-snapshot-of-the-current-context.md
---

# Run Several Synchronous Entry Calls on One asyncio Runner

## Idea and Problem

Call several asynchronous stages from synchronous code while keeping one event loop.

Repeated calls to `asyncio.run()` each create and close a loop. By contrast,
`asyncio.Runner` keeps one loop and a default context across sequential
`run()` calls. Exiting its context closes the loop.

## When to Use

Use this at an application entry point that must hand control back to
synchronous code between stages. For example, an asynchronous preparation
step can set a context variable that a later reporting step reads. The caller
owns the runner and calls it sequentially from one synchronous thread.

## Implementation

```python
import asyncio
from contextvars import ContextVar

batch_name = ContextVar[str]("batch_name", default="unassigned")


async def prepare_batch(name: str) -> asyncio.AbstractEventLoop:
    batch_name.set(name)
    return asyncio.get_running_loop()


async def report_batch(item_count: int) -> tuple[asyncio.AbstractEventLoop, str]:
    return asyncio.get_running_loop(), f"{batch_name.get()}: {item_count} items"
```

## Example

```python
with asyncio.Runner() as runner:
    preparation_loop = runner.run(prepare_batch("inventory"))

    # The synchronous caller can do work between asynchronous stages.
    item_count = len(("pen", "pad", "clip"))
    report_loop, report = runner.run(report_batch(item_count))
    caller_name = batch_name.get()

assert (
    preparation_loop is report_loop
    and report == "inventory: 3 items"
    and caller_name == "unassigned"
    and preparation_loop.is_closed()
)
```

## Trade-offs and Limitations

This is a synchronous boundary, not nested event-loop support. In code that
already runs in an asyncio loop, await the stages directly. The runner's
loop does not run in the background between calls. Finish loop-bound work
before closing the scope rather than letting resources escape it.

The default context deliberately carries values forward. If calls need
separate contexts, pass an explicit fresh context to each `run()`. If
synchronous coordination is unnecessary, a single asynchronous entry point
passed to `asyncio.run()` is simpler.

## Related Snippets

<!-- catalog:related:start -->
- [Run One Async Operation with a Bounded Resource Stack](run-one-async-operation-with-a-bounded-resource-stack.md)
- [Submit a Callable with a Snapshot of the Current Context](submit-a-callable-with-a-snapshot-of-the-current-context.md)
<!-- catalog:related:end -->
