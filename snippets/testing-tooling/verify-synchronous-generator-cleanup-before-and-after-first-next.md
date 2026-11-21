---
title: "Verify Synchronous Generator Cleanup Before and After First Next"
snippet_type: testing-technique
use_cases:
  - lifecycle-management
  - testing
tested_python:
  - "3.14"
dependencies: []
related:
  - ../concurrency-lifecycle/collect-a-bounded-async-generator-prefix-with-deterministic-closure.md
  - capture-unraisable-exception-types-in-a-scoped-test.md
---

# Verify Synchronous Generator Cleanup Before and After First Next

## Idea and Problem

Test whether closing a synchronous generator runs cleanup after iteration begins and skips it before the body starts.

Calling a generator function creates an iterator without entering its body.
Closing it at that point cannot run a `finally` block it never entered.
After `next()` reaches a `yield` inside the `try` block, closing the
suspended generator unwinds the block and executes its `finally` clause.

## When to Use

Use this for a test of a generator that opens a resource inside its body and
can be abandoned after a partial read. Observe both the generator state and
a cleanup effect. A closed state by itself does not tell you whether any
cleanup was needed or performed.

## Implementation

```python
from collections.abc import Generator


def observed_rows(events: list[str]) -> Generator[str]:
    events.append("entered")
    try:
        yield "name,count"
        yield "blue,2"
    finally:
        events.append("cleanup")
```

## Example

```python
from inspect import GEN_CLOSED, GEN_CREATED, GEN_SUSPENDED, getgeneratorstate


unstarted_events: list[str] = []
unstarted = observed_rows(unstarted_events)
created = getgeneratorstate(unstarted)
unstarted.close()
never_entered = getgeneratorstate(unstarted)

started_events: list[str] = []
started = observed_rows(started_events)
try:
    first = next(started)
    paused = getgeneratorstate(started)
finally:
    started.close()

started.close()

assert (created, never_entered, unstarted_events) == (GEN_CREATED, GEN_CLOSED, [])
assert (first, paused, getgeneratorstate(started), started_events) == (
    "name,count",
    GEN_SUSPENDED,
    GEN_CLOSED,
    ["entered", "cleanup"],
)
```

## Trade-offs and Limitations

This fixture records lifecycle transitions instead of opening a real
resource. A resource acquired before iteration needs its own owner because
closing an unstarted generator will not reach its body. Explicitly close
partially consumed generators while the test can still observe cleanup.

`close()` raises `GeneratorExit` inside a suspended generator. Other errors
from cleanup can propagate, and trying to yield another value during that
close raises `RuntimeError`. These checks apply to synchronous generators.

## Related Snippets

<!-- catalog:related:start -->
- [Collect a Bounded Async-Generator Prefix with Deterministic Closure](../concurrency-lifecycle/collect-a-bounded-async-generator-prefix-with-deterministic-closure.md)
- [Capture Unraisable Exception Types in a Scoped Test](capture-unraisable-exception-types-in-a-scoped-test.md)
<!-- catalog:related:end -->
