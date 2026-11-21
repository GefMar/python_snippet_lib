---
title: "Complete One asyncio Future from a Worker-Thread Callback"
snippet_type: pattern
use_cases:
  - concurrency-control
  - interoperability
tested_python:
  - "3.14"
dependencies: []
related:
  - ../testing-tooling/wait-for-a-cross-thread-callback-with-threadingmock-instead-of-sleeping.md
  - reuse-one-pending-future-across-non-cancelling-poll-timeouts.md
---

# Complete One asyncio Future from a Worker-Thread Callback

## Idea and Problem

Turn a callback from a worker thread into one awaitable result on the event loop that owns the future.

Schedule delivery with `call_soon_threadsafe`. Check the future's state and
set its result in that same loop callback. This lets the first delivered
value complete the future and ignores later values, including results
delivered after cancellation.

## When to Use

Use this to adapt a trusted callback interface that supplies one result from
another thread. Build the adapter on the future's running loop and pass only
the callback to the worker. The caller owns the future and the producer's
lifetime.

## Implementation

```python
import asyncio
from collections.abc import Callable


def thread_result_callback[T](future: asyncio.Future[T]) -> Callable[[T], None]:
    loop = asyncio.get_running_loop()
    if future.get_loop() is not loop:
        raise ValueError("the future must belong to the running loop")

    def deliver(value: T) -> None:
        if not future.done():
            future.set_result(value)

    def callback(value: T) -> None:
        loop.call_soon_threadsafe(deliver, value)

    return callback
```

## Example

```python
async def demonstrate() -> tuple[str, bool, list[str]]:
    loop = asyncio.get_running_loop()
    result: asyncio.Future[str] = loop.create_future()
    cancelled: asyncio.Future[str] = loop.create_future()
    report_result = thread_result_callback(result)
    report_cancelled = thread_result_callback(cancelled)
    cancelled.cancel()

    drained = asyncio.Event()
    errors: list[str] = []
    loop.set_exception_handler(lambda _, context: errors.append(context["message"]))

    def worker() -> None:
        report_result("ready")
        report_result("duplicate")
        report_cancelled("late")
        loop.call_soon_threadsafe(drained.set)

    await asyncio.to_thread(worker)
    await drained.wait()
    return await result, cancelled.cancelled(), errors


assert asyncio.run(demonstrate()) == ("ready", True, [])
```

## Trade-offs and Limitations

Cancellation leaves the producer running. Shut it down or unregister its
callback before closing the loop. Scheduling on a closed loop raises
`RuntimeError`. The callback schedules a value for delivery, so the producer
should leave mutable payloads unchanged after handing them off.

Every invocation adds a callback to the loop even when its value ultimately
gets discarded. Use this for one completion with occasional duplicate or
late callbacks. Streams need a queue and backpressure. If a producer also
reports failures, define an error-delivery policy on the loop.

## Related Snippets

<!-- catalog:related:start -->
- [Wait for a Cross-Thread Callback with ThreadingMock Instead of Sleeping](../testing-tooling/wait-for-a-cross-thread-callback-with-threadingmock-instead-of-sleeping.md)
- [Reuse One Pending Future Across Non-Cancelling Poll Timeouts](reuse-one-pending-future-across-non-cancelling-poll-timeouts.md)
<!-- catalog:related:end -->
