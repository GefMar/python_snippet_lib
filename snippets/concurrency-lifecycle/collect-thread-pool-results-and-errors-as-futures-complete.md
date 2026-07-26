---
title: "Collect Thread-Pool Results and Errors as Futures Complete"
snippet_type: recipe
use_cases:
  - concurrency-control
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - gather-async-results-with-bounded-concurrency.md
  - ../python-language/keep-exception-handlers-narrow-with-try-else.md
---

# Collect Thread-Pool Results and Errors as Futures Complete

## Idea and Problem

Run independent blocking calls in a thread pool while retaining both successful values and item-specific failures as each future finishes.

Mapping every future back to its input index and item preserves enough context
to diagnose failures or reconstruct input order later. Returning a structured
report also avoids discarding successful work merely because another item
failed.

## When to Use

Use this recipe for a finite batch of independent I/O-bound or extension-backed
operations when completion-order processing is useful and ordinary failures
should be collected rather than raised immediately. The worker must be safe to
call concurrently. Use an async concurrency primitive for native async work,
and consider a process pool for CPU-bound pure Python work whose callables and
arguments are safely serializable.

## Implementation

```python
from collections.abc import Callable, Iterable
from concurrent.futures import ThreadPoolExecutor, as_completed
from dataclasses import dataclass
from typing import Generic, TypeVar


ItemT = TypeVar("ItemT")
ResultT = TypeVar("ResultT")


@dataclass(frozen=True, slots=True)
class BatchSuccess(Generic[ItemT, ResultT]):
    index: int
    item: ItemT
    value: ResultT


@dataclass(frozen=True, slots=True)
class BatchFailure(Generic[ItemT]):
    index: int
    item: ItemT
    error: Exception


def collect_thread_results(
    items: Iterable[ItemT],
    worker: Callable[[ItemT], ResultT],
    *,
    max_workers: int,
) -> list[BatchSuccess[ItemT, ResultT] | BatchFailure[ItemT]]:
    if isinstance(max_workers, bool) or not isinstance(max_workers, int):
        raise TypeError("max_workers must be an integer")
    if max_workers <= 0:
        raise ValueError("max_workers must be positive")

    completed: list[BatchSuccess[ItemT, ResultT] | BatchFailure[ItemT]] = []
    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        pending = {
            executor.submit(worker, item): (index, item)
            for index, item in enumerate(items)
        }
        for future in as_completed(pending):
            index, item = pending[future]
            try:
                value = future.result()
            except Exception as error:
                completed.append(BatchFailure(index, item, error))
            else:
                completed.append(BatchSuccess(index, item, value))

    return completed
```

## Example

```python
from time import sleep


def reciprocal(value: int) -> float:
    sleep(0.001 * (3 - value))
    return 12 / value


report = collect_thread_results([1, 0, 2], reciprocal, max_workers=3)
successes = sorted(
    (entry.index, entry.item, entry.value)
    for entry in report
    if isinstance(entry, BatchSuccess)
)
failures = [entry for entry in report if isinstance(entry, BatchFailure)]

assert (
    successes,
    len(failures),
    failures[0].index,
    failures[0].item,
    type(failures[0].error),
) == (
    [(0, 1, 12.0), (2, 2, 6.0)],
    1,
    1,
    0,
    ZeroDivisionError,
)
```

## Trade-offs and Limitations

The function eagerly consumes and submits the complete iterable, so memory and
queued work are `O(n)` and the recipe is unsuitable for an unbounded stream.
An exception while consuming the iterable or submitting work propagates
without returning a partial report.
Completion order is intentionally nondeterministic; sort by `index` when input
order is required. Only `Exception` subclasses are recorded: process-control
exceptions such as `KeyboardInterrupt` and `SystemExit` propagate. Leaving the
executor context waits for all submitted work, including queued calls, and
Python threads cannot forcibly stop a hung worker. This is collection policy,
not retry, per-call timeout, rate limiting, or cancellation policy.

## Related Snippets

<!-- catalog:related:start -->
- [Gather Async Results with Bounded Concurrency](gather-async-results-with-bounded-concurrency.md)
- [Keep Exception Handlers Narrow with try/else](../python-language/keep-exception-handlers-narrow-with-try-else.md)
<!-- catalog:related:end -->
