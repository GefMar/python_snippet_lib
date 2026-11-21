---
title: "Restore Thread-Local State Between Jobs on a Reused Worker"
snippet_type: recipe
use_cases:
  - concurrency-control
  - lifecycle-management
tested_python:
  - "3.14"
dependencies: []
related:
  - start-one-thread-with-an-explicit-contextvar-inheritance-policy.md
  - submit-a-callable-with-a-snapshot-of-the-current-context.md
---

# Restore Thread-Local State Between Jobs on a Reused Worker

## Idea and Problem

Keep one job's temporary thread-local value from leaking into the next job on the same worker.

A pool can reuse a thread with its local attributes still set. Save the
previous binding when entering a scope and restore it in `finally`. An
attribute that was absent needs to be removed, not set to `None`.

## When to Use

Use this at a synchronous job boundary when legacy code reads a value from a
plain `threading.local` object you control. Enter and exit the scope on the
worker itself. Nested calls can override the same attribute and then return
to the outer value.

## Implementation

```python
from collections.abc import Iterator
from contextlib import contextmanager
from threading import local

worker_state = local()


@contextmanager
def worker_label(label: str | None) -> Iterator[None]:
    state = vars(worker_state)
    missing = object()
    previous = state.get("label", missing)
    state["label"] = label
    try:
        yield
    finally:
        if previous is missing:
            state.pop("label", None)
        else:
            state["label"] = previous
```

## Example

```python
from concurrent.futures import ThreadPoolExecutor
from threading import current_thread


def nested_job():
    with worker_label(None):
        with worker_label("review"):
            inner_label = worker_state.label
        outer_state = vars(worker_state).copy()
    return current_thread(), inner_label, outer_state


def failing_job() -> None:
    with worker_label("broken job"):
        raise LookupError(worker_state.label)


with worker_label("caller"), ThreadPoolExecutor(max_workers=1) as pool:
    first_thread, inner_label, restored_outer = pool.submit(nested_job).result(timeout=3)
    try:
        pool.submit(failing_job).result(timeout=3)
    except LookupError as error:
        failed_with_label = error.args == ("broken job",)
    else:
        failed_with_label = False

    later_thread, has_label = pool.submit(
        lambda: (current_thread(), hasattr(worker_state, "label"))
    ).result(timeout=3)
    caller_label = worker_state.label

assert (
    first_thread is later_thread
    and inner_label == "review"
    and restored_outer == {"label": None}
    and failed_with_label
    and not has_label
    and caller_label == "caller"
)
```

## Trade-offs and Limitations

This restores one binding, not mutations to a value it referenced or other
side effects. Do not transfer the scope or its thread-local dictionary to
another thread. Thread-local storage also does not distinguish coroutines
running on the same thread; use `ContextVar` for task-local context instead.

## Related Snippets

<!-- catalog:related:start -->
- [Start One Thread with an Explicit ContextVar Inheritance Policy](start-one-thread-with-an-explicit-contextvar-inheritance-policy.md)
- [Submit a Callable with a Snapshot of the Current Context](submit-a-callable-with-a-snapshot-of-the-current-context.md)
<!-- catalog:related:end -->
