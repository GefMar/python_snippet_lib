---
title: "Start One Thread with an Explicit ContextVar Inheritance Policy"
snippet_type: recipe
use_cases:
  - concurrency-control
  - observability
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - submit-a-callable-with-a-snapshot-of-the-current-context.md
  - ../observability-operations/scope-structured-log-fields-with-context-variables.md
  - ../reliability-resilience/propagate-a-monotonic-deadline-with-contextvar.md
---

# Start One Thread with an Explicit ContextVar Inheritance Policy

## Idea and Problem

Start one owned thread with either a snapshot of the caller's current context or a deliberately empty context, without depending on build-specific inheritance defaults.

Python 3.14 lets `Thread` receive a `context` explicitly. Creating a fresh
`copy_context()` carries the values visible at the helper boundary, while a
fresh `Context()` starts the worker from each variable's default. Passing one
of them directly makes the policy visible and consistent across regular and
free-threaded builds.

## When to Use

Use this helper for one trusted, finite, zero-argument synchronous callable
that must run in its own non-daemon thread. Choose inheritance for work that
needs caller-scoped values such as a trace identifier. Choose isolation for
background work that must not receive those ambient values.

Use a caller-owned executor for repeated submissions, `asyncio.to_thread()`
from async code, and explicit function arguments when a value is part of the
business contract. The callable must terminate cooperatively because the
helper waits for it without a timeout.

## Implementation

```python
from collections.abc import Callable
from contextvars import Context, ContextVar, copy_context
from threading import Thread


def run_with_thread_context[ResultT](
    work: Callable[[], ResultT],
    *,
    inherit_context: bool,
) -> ResultT:
    if not callable(work):
        raise TypeError("work must be callable")
    if type(inherit_context) is not bool:
        raise TypeError("inherit_context must be an exact boolean")

    context = copy_context() if inherit_context else Context()
    results: list[ResultT] = []
    failures: list[BaseException] = []

    def target() -> None:
        try:
            results.append(work())
        except BaseException as error:
            failures.append(error)

    worker = Thread(
        target=target,
        name="explicit-context-worker",
        daemon=False,
        context=context,
    )
    worker.start()
    worker.join()

    if failures:
        error = failures.pop()
        try:
            raise error
        finally:
            del error
    if len(results) != 1:
        raise RuntimeError("worker produced no result")
    return results.pop()
```

## Example

```python
request_label: ContextVar[str] = ContextVar(
    "request_label",
    default="not-set",
)


def observe_and_change_label() -> tuple[str, str]:
    observed = request_label.get()
    request_label.set("worker-local")
    return observed, request_label.get()


token = request_label.set("request-42")
try:
    inherited = run_with_thread_context(
        observe_and_change_label,
        inherit_context=True,
    )
    isolated = run_with_thread_context(
        observe_and_change_label,
        inherit_context=False,
    )
    caller_value = request_label.get()
finally:
    request_label.reset(token)

assert (inherited, isolated, caller_value, request_label.get()) == (
    ("request-42", "worker-local"),
    ("not-set", "worker-local"),
    "request-42",
    "not-set",
)
```

## Trade-offs and Limitations

The inherited snapshot is taken before `Thread.start()`. It copies the context
mapping, not the objects stored as values, so mutable values can still be
shared and raced. Worker changes stay in its context and are not merged back
into the caller. A new `Context` is created for every invocation because the
same context cannot be entered concurrently from multiple threads.

The helper catches a worker exception only to transfer it after `join()` and
then clears the temporary cell; the original exception and traceback are
re-raised. It does not rewrite failures, copy thread-local storage, control OS
thread state, or propagate later caller changes into the already captured
snapshot.

Joining without a timeout preserves simple ownership: success means the
non-daemon worker has finished and no live thread is abandoned. The cost and
termination of `work` are otherwise unbounded. This is not cancellation,
parallelism control, an executor, deep isolation, or a security boundary.

## Related Snippets

<!-- catalog:related:start -->
- [Submit a Callable with a Snapshot of the Current Context](submit-a-callable-with-a-snapshot-of-the-current-context.md)
- [Scope Structured Log Fields with Context Variables](../observability-operations/scope-structured-log-fields-with-context-variables.md)
- [Propagate a Monotonic Deadline with ContextVar](../reliability-resilience/propagate-a-monotonic-deadline-with-contextvar.md)
<!-- catalog:related:end -->
