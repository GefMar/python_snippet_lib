---
title: "Submit a Callable with a Snapshot of the Current Context"
snippet_type: recipe
use_cases:
  - concurrency-control
  - interoperability
  - observability
tested_python:
  - "3.14"
dependencies: []
related:
  - collect-thread-pool-results-and-errors-as-futures-complete.md
  - ../observability-operations/scope-structured-log-fields-with-context-variables.md
  - gather-async-results-with-bounded-concurrency.md
---

# Submit a Callable with a Snapshot of the Current Context

## Idea and Problem

Capture the current context for each thread-pool submission so the worker sees the caller's context variables without custom globals or exception wrappers.

Threads have separate effective context stacks, and a plain
`ThreadPoolExecutor.submit()` does not copy the submitting thread's current
context. Taking a fresh snapshot and entering it around one bound call carries
request-scoped values into that call while leaving results, exceptions, and
cancellation behavior to the ordinary `Future`.

## When to Use

Use this helper when synchronous code is submitted directly to a caller-owned
`ThreadPoolExecutor` and it must observe context variables such as a trace or
request identifier. Capture a new context for every submission. Async callers
should usually prefer `asyncio.to_thread()`, which already propagates the
current context. Process pools need an explicitly serializable data model
instead. Interpreter pools are also excluded: although their executor is a
thread-pool subclass, the copied context and local wrapper cannot cross the
isolated-interpreter boundary.

## Implementation

```python
from concurrent.futures import (
    Future,
    InterpreterPoolExecutor,
    ThreadPoolExecutor,
)
from contextvars import copy_context
from collections.abc import Callable
from typing import ParamSpec, TypeVar


Parameters = ParamSpec("Parameters")
Result = TypeVar("Result")


def submit_with_context(
    executor: ThreadPoolExecutor,
    function: Callable[Parameters, Result],
    /,
    *args: Parameters.args,
    **kwargs: Parameters.kwargs,
) -> Future[Result]:
    if isinstance(executor, InterpreterPoolExecutor):
        raise TypeError("interpreter pools are not supported")
    if not isinstance(executor, ThreadPoolExecutor):
        raise TypeError("executor must be a ThreadPoolExecutor")
    if not callable(function):
        raise TypeError("function must be callable")

    context = copy_context()

    def invoke() -> Result:
        return context.run(function, *args, **kwargs)

    return executor.submit(invoke)
```

## Example

```python
from contextvars import ContextVar


request_id: ContextVar[str] = ContextVar(
    "request_id",
    default="not-set",
)


def worker_call() -> tuple[str, str]:
    captured = request_id.get()
    request_id.set("worker-local")
    return captured, request_id.get()


token = request_id.set("request-42")
try:
    with ThreadPoolExecutor(max_workers=1) as executor:
        future = submit_with_context(executor, worker_call)
        request_id.set("caller-changed")
        worker_values = future.result(timeout=1.0)
        caller_value = request_id.get()
finally:
    request_id.reset(token)

assert (worker_values, caller_value, request_id.get()) == (
    ("request-42", "worker-local"),
    "caller-changed",
    "not-set",
)
```

## Trade-offs and Limitations

The snapshot copies the context mapping, not the objects stored as values.
Mutable values can therefore still be shared and raced across threads. Changes
made by the worker remain in its copied context and are not merged back into
the caller. A separate snapshot per submission is essential because one
`Context` cannot be entered concurrently by multiple threads.

The helper neither owns nor shuts down the executor. The returned `Future`
preserves the callable's normal result, exception, and traceback; no diagnostic
stack is captured or rewritten. Cancelling a future can prevent queued work
from starting, but it cannot stop a call that is already running. Thread-local
state outside `ContextVar`, executor saturation, timeouts, and worker cleanup
remain separate concerns.

## Related Snippets

<!-- catalog:related:start -->
- [Collect Thread-Pool Results and Errors as Futures Complete](collect-thread-pool-results-and-errors-as-futures-complete.md)
- [Scope Structured Log Fields with Context Variables](../observability-operations/scope-structured-log-fields-with-context-variables.md)
- [Gather Async Results with Bounded Concurrency](gather-async-results-with-bounded-concurrency.md)
<!-- catalog:related:end -->
