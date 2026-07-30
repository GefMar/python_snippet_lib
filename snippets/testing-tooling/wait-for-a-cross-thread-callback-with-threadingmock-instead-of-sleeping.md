---
title: "Wait for a Cross-Thread Callback with ThreadingMock Instead of Sleeping"
snippet_type: testing-technique
use_cases:
  - concurrency-control
  - testing
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - verify-calls-to-a-standalone-function-with-create-autospec.md
  - wait-for-named-queue-conditions-under-one-monotonic-deadline.md
  - ../concurrency-lifecycle/stop-a-polling-worker-cooperatively-with-an-event.md
---

# Wait for a Cross-Thread Callback with ThreadingMock Instead of Sleeping

## Idea and Problem

Test one finite worker thread by waiting for an exact mock call and its completion under one monotonic deadline instead of guessing a sleep duration.

`ThreadingMock` records calls from another thread and exposes a blocking wait
for an invocation. A start gate keeps the worker from using the callback until
setup is complete. After a call is observed, the remaining deadline is spent
joining the worker; only then are its exact arguments and the complete one-call
contract asserted, so a late second call cannot race with verification.

## When to Use

Use this technique on Python 3.13 or later in a unit test for exactly one owned,
non-daemon worker whose operation is guaranteed to finish. Give the mock a
finite per-instance timeout from 0.001 through 5 seconds, define one exact
expected positional and keyword call, and return a meaningful worker result.

Use an `Event`, `Condition`, queue, future, or application protocol for
production synchronization. Use async test primitives for coroutine code and
a larger harness when several workers, repeated callbacks, cancellation, or
load behavior are part of the contract.

## Implementation

```python
import math
import time
from collections.abc import Callable
from threading import Event, Thread
from unittest.mock import ThreadingMock

_MIN_TIMEOUT_SECONDS = 0.001
_MAX_TIMEOUT_SECONDS = 5.0


class WorkerFailed(AssertionError):
    def __init__(self, error: BaseException) -> None:
        self.error = error
        super().__init__(f"worker raised {type(error).__name__}")


def run_one_worker_and_expect_callback[ResultT](
    work: Callable[[ThreadingMock], ResultT],
    *,
    expected_args: tuple[object, ...],
    expected_kwargs: dict[str, object],
    timeout: int | float,
) -> ResultT:
    if not callable(work):
        raise TypeError("work must be callable")
    if type(expected_args) is not tuple:
        raise TypeError("expected_args must be an exact tuple")
    if type(expected_kwargs) is not dict:
        raise TypeError("expected_kwargs must be an exact dict")
    if not all(type(name) is str for name in expected_kwargs):
        raise TypeError("expected keyword names must be exact strings")
    if isinstance(timeout, bool) or not isinstance(timeout, (int, float)):
        raise TypeError("timeout must be numeric")
    try:
        seconds = float(timeout)
    except OverflowError:
        raise ValueError("timeout must be representable as a float") from None
    if (
        not math.isfinite(seconds)
        or not _MIN_TIMEOUT_SECONDS <= seconds <= _MAX_TIMEOUT_SECONDS
    ):
        raise ValueError("timeout is outside the supported range")

    callback = ThreadingMock(timeout=seconds)
    keyword_arguments = expected_kwargs.copy()
    gate = Event()
    results: list[ResultT] = []
    failures: list[BaseException] = []

    def target() -> None:
        gate.wait()
        try:
            results.append(work(callback))
        except BaseException as error:
            failures.append(error)

    deadline = time.monotonic() + seconds
    worker = Thread(target=target, name="callback-test-worker", daemon=False)
    started = False
    wait_error: AssertionError | None = None
    try:
        worker.start()
        started = True
        gate.set()
        try:
            callback.wait_until_called(
                timeout=max(0.0, deadline - time.monotonic()),
            )
        except AssertionError as error:
            wait_error = error
    finally:
        gate.set()
        if started:
            worker.join(max(0.0, deadline - time.monotonic()))

    if worker.is_alive():
        raise AssertionError("worker did not finish before the deadline")
    if failures:
        raise WorkerFailed(failures[0]) from failures[0]
    if wait_error is not None:
        raise AssertionError(
            "expected callback was not observed before the deadline"
        ) from wait_error

    callback.assert_called_once_with(*expected_args, **keyword_arguments)
    if len(results) != 1:
        raise AssertionError("worker did not produce exactly one result")
    return results[0]
```

## Example

```python
def successful_worker(callback: ThreadingMock) -> str:
    callback("ready", item=7)
    return "processed"


result = run_one_worker_and_expect_callback(
    successful_worker,
    expected_args=("ready",),
    expected_kwargs={"item": 7},
    timeout=1,
)


def silent_worker(callback: ThreadingMock) -> str:
    del callback
    return "skipped"


try:
    run_one_worker_and_expect_callback(
        silent_worker,
        expected_args=("ready",),
        expected_kwargs={},
        timeout=0.01,
    )
except AssertionError:
    missing_call_reported = True
else:
    missing_call_reported = False


assert (result, missing_call_reported) == ("processed", True)
```

## Trade-offs and Limitations

`ThreadingMock` records a call before running its `side_effect`. Therefore a
successful wait proves only that invocation was recorded; it does not prove
that a side effect completed or that its exception reached the waiting thread.
This helper closes that gap for the worker itself by capturing its failure or
result and joining before the final assertion. It does not install a callback
side effect, and it leaves `ThreadingMock.DEFAULT_TIMEOUT` unchanged.

The gate is always released and the join uses only time remaining before the
single monotonic deadline. A missing or mismatched call, an unfinished worker,
or a late extra call raises `AssertionError`; a worker exception is retained as
the cause of `WorkerFailed`. Expected-value equality and the worker body must be
finite because Python cannot forcibly stop a thread. A worker still alive after
the bounded join remains owned by the test process.

This is a deterministic unit-test synchronization pattern, not production
coordination, cancellation, async support, high-volume callback verification,
or a scheduler guarantee. Thread scheduling can consume the timeout, so choose
the smallest bound that remains realistic on the test environment rather than
using a delay to make a race less likely.

## Related Snippets

<!-- catalog:related:start -->
- [Verify Calls to a Standalone Function with Autospec](verify-calls-to-a-standalone-function-with-create-autospec.md)
- [Wait for Named Queue Conditions Under One Monotonic Deadline](wait-for-named-queue-conditions-under-one-monotonic-deadline.md)
- [Stop a Polling Worker Cooperatively with an Event](../concurrency-lifecycle/stop-a-polling-worker-cooperatively-with-an-event.md)
<!-- catalog:related:end -->
