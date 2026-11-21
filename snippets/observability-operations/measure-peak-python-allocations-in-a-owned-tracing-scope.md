---
title: "Measure Peak Python Allocations in an Owned Tracing Scope"
snippet_type: testing-technique
use_cases:
  - observability
  - testing
tested_python:
  - "3.14"
dependencies: []
related:
  - report-a-bounded-positive-allocation-delta-from-filtered-tracemalloc-snapshots.md
  - ../testing-tooling/capture-executed-lines-for-one-code-object-with-sys-monitoring.md
---

# Measure Peak Python Allocations in an Owned Tracing Scope

## Idea and Problem

Measure how many traced Python allocation bytes remain after a call and how high usage rose while it ran.

A before-and-after snapshot can miss storage that was allocated and released
during the operation. Start a fresh tracing session and read both the current
and peak counters before stopping it.

## When to Use

Use this in a controlled test or diagnostic run with exclusive control of
`tracemalloc`. Other threads and unrelated work can contribute to the same
counters. Refuse to start if an existing tracing session is active.

## Implementation

```python
import tracemalloc
from collections.abc import Callable


def measure_allocations[T](operation: Callable[[], T]) -> tuple[T, int, int]:
    if tracemalloc.is_tracing():
        raise RuntimeError("another tracing session is active")

    tracemalloc.start()
    try:
        result = operation()
        current_bytes, peak_bytes = tracemalloc.get_traced_memory()
        return result, current_bytes, peak_bytes
    finally:
        tracemalloc.stop()
```

## Example

```python
def temporary_workspace() -> int:
    workspace = bytearray(200_000)
    return len(workspace)


def fail_operation() -> None:
    raise ValueError("operation failed")


size, current, peak = measure_allocations(temporary_workspace)
try:
    measure_allocations(fail_operation)
except ValueError:
    stopped_after_failure = not tracemalloc.is_tracing()
else:
    stopped_after_failure = False

tracemalloc.start()
try:
    try:
        measure_allocations(temporary_workspace)
    except RuntimeError:
        preserved_existing = tracemalloc.is_tracing()
    else:
        preserved_existing = False
finally:
    tracemalloc.stop()

assert size == 200_000 and peak >= size and current < peak
assert stopped_after_failure and preserved_existing
```

## Trade-offs and Limitations

The counters cover allocations tracked since tracing started, not process
RSS or all allocations made by native extensions. The reported current size
includes any tracked objects still held at the measurement point, including
the result. The exact totals vary between runs and interpreter builds.

The caller must serialize tracing configuration and give the measurement a
quiet process. The operation should finish its work before returning and leave
tracing settings alone. Use snapshot comparisons to identify allocation sites.

## Related Snippets

<!-- catalog:related:start -->
- [Report a Bounded Positive Allocation Delta from Filtered tracemalloc Snapshots](report-a-bounded-positive-allocation-delta-from-filtered-tracemalloc-snapshots.md)
- [Capture Executed Lines for One Code Object with sys.monitoring](../testing-tooling/capture-executed-lines-for-one-code-object-with-sys-monitoring.md)
<!-- catalog:related:end -->
