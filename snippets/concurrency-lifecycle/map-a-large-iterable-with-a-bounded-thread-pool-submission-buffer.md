---
title: "Map a Large Iterable with a Bounded Thread-Pool Submission Buffer"
snippet_type: recipe
use_cases:
  - concurrency-control
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - collect-thread-pool-results-and-errors-as-futures-complete.md
  - run-bounded-thread-work-by-priority-and-submission-order.md
  - ../data-processing/batch-items-by-estimated-byte-size.md
---

# Map a Large Iterable with a Bounded Thread-Pool Submission Buffer

## Idea and Problem

Use Python 3.14's thread-pool map buffer to consume a large iterable gradually while yielding results in input order and raising worker failures at their input positions.

Without `buffersize`, executor mapping collects input eagerly. A positive
buffer limits submitted calls whose results have not yet been yielded, so input
consumption pauses when the buffer is full. A small generator wrapper adds
validated limits and shuts the pool down when it is exhausted or explicitly
closed, without custom queues, daemon threads, or adaptive worker creation.

## When to Use

Use this recipe for a finite or long synchronous iterable of independent,
I/O-bound calls when `map` ordering is part of the consumer contract. The
worker and its arguments must be safe to use concurrently. A worker exception
is raised when iteration reaches that input position, after every earlier
result has been yielded.

Use completion-order futures when slow earlier work must not block ready later
results. Use an async concurrency primitive for native async work and a
process or interpreter pool only after accounting for its serialization and
startup requirements.

## Implementation

```python
from collections.abc import Callable, Generator, Iterable
from concurrent.futures import ThreadPoolExecutor
from typing import TypeVar


ItemT = TypeVar("ItemT")
ResultT = TypeVar("ResultT")
_MAX_WORKERS = 64
_MAX_BUFFER_SIZE = 1_024


def map_threaded_ordered(
    items: Iterable[ItemT],
    worker: Callable[[ItemT], ResultT],
    *,
    max_workers: int,
    buffer_size: int,
) -> Generator[ResultT, None, None]:
    if not callable(worker):
        raise TypeError("worker must be callable")
    for name, value, upper in (
        ("max_workers", max_workers, _MAX_WORKERS),
        ("buffer_size", buffer_size, _MAX_BUFFER_SIZE),
    ):
        if isinstance(value, bool) or not isinstance(value, int):
            raise TypeError(f"{name} must be an integer")
        if not 1 <= value <= upper:
            raise ValueError(f"{name} is outside the supported range")

    executor = ThreadPoolExecutor(
        max_workers=max_workers,
        thread_name_prefix="bounded-map",
    )
    try:
        results = executor.map(worker, items, buffersize=buffer_size)
        yield from results
    finally:
        executor.shutdown(wait=True, cancel_futures=True)
```

## Example

```python
from contextlib import closing
from time import sleep


def transform(value: int) -> int:
    sleep(0.001 * (4 - value))
    if value == 2:
        raise ValueError("invalid item")
    return value * 10


with closing(
    map_threaded_ordered(
        range(4),
        transform,
        max_workers=3,
        buffer_size=3,
    )
) as results:
    first = next(results)
    second = next(results)
    try:
        next(results)
    except ValueError as error:
        replayed_error = str(error)

assert (first, second, replayed_error) == (0, 10, "invalid item")
```

## Trade-offs and Limitations

Ordered retrieval creates head-of-line blocking: a slow early call delays every
later value even if those calls have finished. The buffer bounds submitted
results that have not been yielded, not the memory retained by one item or one
result. `chunksize` does not change `ThreadPoolExecutor` behavior.

Exhausting or explicitly closing the generator cancels calls that have not
started, but shutdown waits for running calls because Python threads cannot be
stopped safely. A consumer that might break early must use `contextlib.closing`
or call `close()` in `finally`; merely retaining a partly consumed generator
also retains its executor. Every worker still needs its own timeout and
cooperative resource cleanup. Threads do not remove the GIL for CPU-bound pure
Python work, and this wrapper provides no retry, deadline, rate, or
completion-order policy.

## Related Snippets

<!-- catalog:related:start -->
- [Collect Thread-Pool Results and Errors as Futures Complete](collect-thread-pool-results-and-errors-as-futures-complete.md)
- [Run Bounded Thread Work by Priority and Submission Order](run-bounded-thread-work-by-priority-and-submission-order.md)
- [Batch Items by Estimated Byte Size](../data-processing/batch-items-by-estimated-byte-size.md)
<!-- catalog:related:end -->
