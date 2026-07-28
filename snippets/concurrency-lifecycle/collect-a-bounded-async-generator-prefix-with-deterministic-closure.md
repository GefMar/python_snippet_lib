---
title: "Collect a Bounded Async-Generator Prefix with Deterministic Closure"
snippet_type: pattern
use_cases:
  - concurrency-control
  - lifecycle-management
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - run-one-async-operation-with-a-bounded-resource-stack.md
  - guard-an-async-resource-with-explicit-lifecycle-states.md
  - ../networking-protocols/yield-bounded-sse-frames-with-serialized-comment-keepalives.md
---

# Collect a Bounded Async-Generator Prefix with Deterministic Closure

## Idea and Problem

Take ownership of one exact async generator, collect no more than a declared prefix, and close the generator on every exit path.

Stopping an `async for` loop after enough items does not by itself express who
owns the suspended generator. Wrapping the transferred generator in
`contextlib.aclosing` makes early completion, exhaustion, source failure, and
propagated cancellation pass through one awaited `aclose()` boundary. Calling
`anext()` explicitly also makes the no-overread rule visible: the collector
never requests a witness item after reaching the limit.

## When to Use

Use this pattern when a caller creates a native async generator solely for one
bounded collection operation and transfers ownership after both arguments have
validated. The source may hold resources across yields, and its cleanup must be
safe to run in the same task and context as iteration. A zero limit is useful
when policy can disable collection without entering the generator body.

Use a different abstraction when the source is a generic async iterator, must
remain reusable, is shared with another consumer, or requires a timeout or
shielded shutdown policy. If one extra pull is acceptable and detecting
truncation matters, design a separate result contract that explicitly requests
that witness item.

## Implementation

```python
from collections.abc import AsyncGenerator
from contextlib import aclosing
from types import AsyncGeneratorType
from typing import TypeVar

ItemT = TypeVar("ItemT")

_MAX_PREFIX_ITEMS = 4_096


async def collect_async_generator_prefix(
    source: AsyncGenerator[ItemT, None],
    limit: int,
) -> tuple[ItemT, ...]:
    if type(source) is not AsyncGeneratorType:
        raise TypeError("source must be an exact async generator")
    if type(limit) is not int:
        raise TypeError("limit must be an exact non-boolean integer")
    if not 0 <= limit <= _MAX_PREFIX_ITEMS:
        raise ValueError("limit is outside the supported range")

    items: list[ItemT] = []
    async with aclosing(source):
        for _ in range(limit):
            try:
                item = await anext(source)
            except StopAsyncIteration:
                break
            items.append(item)
    return tuple(items)
```

## Example

```python
import asyncio


async def run_example():
    prefix_events: list[str] = []

    async def many_values():
        prefix_events.append("entered")
        try:
            for value in range(4):
                prefix_events.append(f"yield:{value}")
                yield value
        finally:
            prefix_events.append("closed")

    prefix = await collect_async_generator_prefix(many_values(), 2)

    exhausted_events: list[str] = []

    async def one_value():
        exhausted_events.append("entered")
        try:
            exhausted_events.append("yield:only")
            yield "only"
        finally:
            exhausted_events.append("closed")

    exhausted = await collect_async_generator_prefix(one_value(), 3)

    never_started_events: list[str] = []

    async def never_started():
        never_started_events.append("entered")
        try:
            yield "unused"
        finally:
            never_started_events.append("closed")

    unopened = never_started()
    empty = await collect_async_generator_prefix(unopened, 0)
    try:
        await anext(unopened)
    except StopAsyncIteration:
        unopened_is_closed = True
    else:
        unopened_is_closed = False

    return (
        prefix,
        tuple(prefix_events),
        exhausted,
        tuple(exhausted_events),
        empty,
        tuple(never_started_events),
        unopened_is_closed,
    )


assert asyncio.run(run_example()) == (
    (0, 1),
    ("entered", "yield:0", "yield:1", "closed"),
    ("only",),
    ("entered", "yield:only", "closed"),
    (),
    (),
    True,
)
```

## Trade-offs and Limitations

For `k` returned items the tuple and temporary list retain `O(k)` references.
The function makes at most `limit` pull attempts. Exhaustion or failure before
the limit is observed by attempt `k + 1`; reaching the limit after `k` yields
does not make a witness pull. Source awaits and cleanup add source-defined time
and resource costs.

Once validation succeeds, the caller must not iterate or close the transferred
generator concurrently. Validation failures occur before `aclosing` is entered,
so the caller still owns and must close an otherwise valid generator in that
case. The returned tuple is immutable, but yielded objects are not copied or
deeply frozen.

When cancellation propagates from a source pull, normal async-context-manager
unwinding attempts and awaits `aclose()`. A generator that suppresses
`CancelledError` has source-defined behavior instead, and cleanup can raise,
never finish, or be interrupted by an additional cancellation. This pattern
therefore promises neither timeout nor shielding and makes no repeated-
cancellation guarantee. A cleanup error follows ordinary exception chaining
and can replace the active source or cancellation exception.

Closing a never-started async generator marks it closed without necessarily
entering its body or running a `finally` block. The function excludes generic
async iterators without `aclose()`, concurrent iteration, reusable ownership,
and any promise to distinguish exact exhaustion from truncation at the limit.

## Related Snippets

<!-- catalog:related:start -->
- [Run One Async Operation with a Bounded Resource Stack](run-one-async-operation-with-a-bounded-resource-stack.md)
- [Guard an Async Resource with Explicit Lifecycle States](guard-an-async-resource-with-explicit-lifecycle-states.md)
- [Yield Bounded SSE Frames with Serialized Comment Keepalives](../networking-protocols/yield-bounded-sse-frames-with-serialized-comment-keepalives.md)
<!-- catalog:related:end -->
