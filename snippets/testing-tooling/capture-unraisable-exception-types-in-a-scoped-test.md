---
title: "Capture Unraisable Exception Types in a Scoped Test"
snippet_type: testing-technique
use_cases:
  - lifecycle-management
  - observability
  - testing
tested_python:
  - "3.14"
dependencies: []
related:
  - ../python-language/attach-best-effort-cleanup-with-weakref-finalize-without-retaining-the-owner.md
  - ../observability-operations/capture-a-bounded-pickle-friendly-exception-report.md
---

# Capture Unraisable Exception Types in a Scoped Test

## Idea and Problem

Assert which exception types Python reports from an operation whose error cannot propagate to the caller.

Errors in a destructor reach `sys.unraisablehook` instead of a surrounding
`try` block. A test can temporarily replace the hook and record type names.
Keeping only names avoids holding the exception, traceback, or object that
Python passes to the hook.

## When to Use

Use this in an isolated, single-threaded test of a known cleanup path, such
as a destructor supplied by a dependency. Keep the operation short and
control when finalization runs. Test explicit cleanup directly when it can
report ordinary errors to its caller.

## Implementation

```python
import sys
from collections.abc import Iterator
from contextlib import contextmanager


@contextmanager
def capture_unraisable_types() -> Iterator[list[str | None]]:
    names: list[str | None] = []
    previous = sys.unraisablehook

    def record(event) -> None:
        error_type = event.exc_type
        names.append(error_type.__name__ if error_type is not None else None)

    sys.unraisablehook = record
    try:
        yield names
    finally:
        sys.unraisablehook = previous
```

## Example

```python
import gc
from weakref import ref


class BrokenFinalizer:
    def __del__(self) -> None:
        raise ValueError("test fixture cleanup failed")


class EndProbe(RuntimeError):
    pass


original_hook = sys.unraisablehook
try:
    with capture_unraisable_types() as observed:
        instance = BrokenFinalizer()
        instance_ref = ref(instance)
        del instance
        gc.collect()
        raise EndProbe
except EndProbe:
    pass
else:
    raise AssertionError("the body exception must propagate")

assert (observed, instance_ref(), sys.unraisablehook is original_hook) == (
    ["ValueError"],
    None,
    True,
)
```

## Trade-offs and Limitations

Replacing `sys.unraisablehook` affects the whole interpreter and interrupts
its normal reporting during the scope. Other finalizers can add entries.
Run this with exclusive control of the hook and avoid concurrent workload.

A type name identifies only the kind of error and may be shared by several
classes. The test discards messages and traces. Avoid storing live hook
arguments: retaining a traceback can keep frames alive, and retaining the
reported object can interfere with finalization. The example uses a simple
CPython object with no reference cycle.

## Related Snippets

<!-- catalog:related:start -->
- [Attach Best-Effort Cleanup with weakref.finalize Without Retaining the Owner](../python-language/attach-best-effort-cleanup-with-weakref-finalize-without-retaining-the-owner.md)
- [Capture a Bounded Pickle-Friendly Exception Report](../observability-operations/capture-a-bounded-pickle-friendly-exception-report.md)
<!-- catalog:related:end -->
