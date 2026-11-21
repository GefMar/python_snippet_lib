---
title: "Measure Wall and Thread CPU Time for One Synchronous Operation"
snippet_type: recipe
use_cases:
  - observability
  - performance-optimization
tested_python:
  - "3.14"
dependencies: []
related:
  - measure-and-freeze-elapsed-time-in-a-context.md
  - compute-a-process-cpu-rate-from-two-linux-procfs-samples.md
---

# Measure Wall and Thread CPU Time for One Synchronous Operation

## Idea and Problem

Measure both the elapsed duration of a synchronous call and the CPU time used by its calling thread.

Take a difference on each clock. `perf_counter_ns` measures elapsed time,
including waiting. `thread_time_ns` measures the current thread's user and
system CPU time. Preserve the two durations as separate observations.

## When to Use

Use this for a synchronous operation on a platform with a thread CPU clock.
Run the callable and take both measurements in the same thread. This is
useful when investigating latency that may include waiting for other work.

## Implementation

```python
from collections.abc import Callable
from dataclasses import dataclass
from time import perf_counter_ns, thread_time_ns


@dataclass(frozen=True, slots=True)
class OperationTiming:
    elapsed_ns: int
    thread_cpu_ns: int


def measure_operation[ResultT](
    operation: Callable[[], ResultT],
) -> tuple[ResultT, OperationTiming]:
    wall_before = perf_counter_ns()
    cpu_before = thread_time_ns()
    result = operation()
    thread_cpu = thread_time_ns() - cpu_before
    elapsed = perf_counter_ns() - wall_before
    return result, OperationTiming(elapsed, thread_cpu)
```

## Example

```python
result, timing = measure_operation(lambda: sum(value * value for value in range(100)))

assert result == 328350 and timing.elapsed_ns >= 0 and timing.thread_cpu_ns >= 0
```

## Trade-offs and Limitations

Elapsed time includes scheduling delays, waits, and measurement overhead.
CPU work delegated to other threads or processes falls outside the calling
thread's CPU measurement. The clocks have independent resolutions, so use
the readings as diagnostics and account for noise on short operations.

The helper returns timings after successful calls. A callable's exception
propagates. Use repeated measurements and a benchmark design when comparing
small performance differences.

## Related Snippets

<!-- catalog:related:start -->
- [Measure and Freeze Elapsed Time in a Context](measure-and-freeze-elapsed-time-in-a-context.md)
- [Compute a Process CPU Rate from Two Linux procfs Samples](compute-a-process-cpu-rate-from-two-linux-procfs-samples.md)
<!-- catalog:related:end -->
