---
title: "Reuse One Pending Future Across Non-Cancelling Poll Timeouts"
snippet_type: pattern
use_cases:
  - concurrency-control
  - lifecycle-management
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - collect-a-bounded-thread-pool-batch-under-one-deadline.md
  - collect-thread-pool-results-and-errors-as-futures-complete.md
  - submit-a-callable-with-a-snapshot-of-the-current-context.md
---

# Reuse One Pending Future Across Non-Cancelling Poll Timeouts

## Idea and Problem

Poll one owned future repeatedly without starting duplicate work when a bounded wait expires before the future finishes.

`Future.result(timeout=...)` uses `TimeoutError` both for an expired wait and
for a completed operation that raised that exception. The helper retains the
same unfinished future only in the first case. Every terminal outcome clears
the slot, while explicit `Pending` and `Ready` values keep an unfinished call
distinct from a successful `None` payload.

## When to Use

Use this pattern when one lifecycle owner starts at most one exact
`concurrent.futures.Future` at a time and checks it through short synchronous
waits. The factory must transfer ownership of a newly created future to the
helper, and a new future should represent new work only after the previous one
has reached or left terminal handling.

Keep executor shutdown and callable-specific stopping policy outside this
class. The nonblocking lifecycle lock detects overlapping calls; it is a guard
against accidental concurrent polling, not a multi-owner scheduling policy.

## Implementation

```python
import math
from collections.abc import Callable
from concurrent.futures import Future
from dataclasses import dataclass
from threading import Lock, TIMEOUT_MAX
from typing import Generic, TypeVar


ResultT = TypeVar("ResultT")


@dataclass(frozen=True, slots=True)
class Pending:
    pass


@dataclass(frozen=True, slots=True)
class Ready(Generic[ResultT]):
    value: ResultT


class PendingFuturePoller(Generic[ResultT]):
    def __init__(
        self,
        create_future: Callable[[], Future[ResultT]],
    ) -> None:
        if not callable(create_future):
            raise TypeError("create_future must be callable")
        self._create_future = create_future
        self._future: Future[ResultT] | None = None
        self._lifecycle_lock = Lock()

    @staticmethod
    def _validate_timeout(timeout: int | float) -> float:
        if isinstance(timeout, bool) or not isinstance(timeout, (int, float)):
            raise TypeError("timeout must be an integer or float")
        try:
            timeout_seconds = float(timeout)
        except OverflowError as error:
            raise ValueError(
                "timeout must be representable as a float"
            ) from error
        if not math.isfinite(timeout_seconds) or timeout_seconds <= 0:
            raise ValueError("timeout must be finite and positive")
        if timeout_seconds > TIMEOUT_MAX:
            raise ValueError("timeout exceeds the platform wait limit")
        return timeout_seconds

    def _consume_done(
        self,
        future: Future[ResultT],
    ) -> Ready[ResultT]:
        try:
            value = future.result()
        finally:
            self._future = None
        return Ready(value)

    def poll(
        self,
        *,
        timeout: int | float,
    ) -> Pending | Ready[ResultT]:
        timeout_seconds = self._validate_timeout(timeout)
        if not self._lifecycle_lock.acquire(blocking=False):
            raise RuntimeError("another lifecycle operation is in progress")

        try:
            if self._future is None:
                created = self._create_future()
                if type(created) is not Future:
                    raise TypeError(
                        "create_future must return an exact Future"
                    )
                self._future = created
            future = self._future

            try:
                value = future.result(timeout=timeout_seconds)
            except TimeoutError:
                if not future.done():
                    return Pending()
                return self._consume_done(future)
            except BaseException:
                self._future = None
                raise
            else:
                self._future = None
                return Ready(value)
        finally:
            self._lifecycle_lock.release()

    def cancel_and_release(self) -> Future[ResultT] | None:
        if not self._lifecycle_lock.acquire(blocking=False):
            raise RuntimeError("another lifecycle operation is in progress")
        try:
            future = self._future
            self._future = None
            if future is not None:
                future.cancel()
            return future
        finally:
            self._lifecycle_lock.release()
```

## Example

```python
future: Future[None] = Future()
created: list[Future[None]] = []


def create_future() -> Future[None]:
    created.append(future)
    return future


poller = PendingFuturePoller(create_future)
first = poller.poll(timeout=0.001)
future.set_result(None)
second = poller.poll(timeout=0.001)

terminal = Future()
terminal.set_exception(TimeoutError("operation timed out"))
terminal_poller = PendingFuturePoller(lambda: terminal)
try:
    terminal_poller.poll(timeout=0.001)
except TimeoutError:
    terminal_timeout_raised = True
else:
    terminal_timeout_raised = False

assert (
    type(first),
    type(second),
    second.value,
    len(created),
    created[0] is future,
    terminal_timeout_raised,
    terminal_poller.cancel_and_release(),
) == (Pending, Ready, None, 1, True, True, None)
```

## Trade-offs and Limitations

The helper calls the factory and waits while holding a nonblocking lock. A
second `poll()` or `cancel_and_release()` therefore fails immediately instead
of sharing the wait. Factory exceptions leave the slot empty. Success,
cancellation, and any stored `BaseException` clear the slot before returning
or propagating; a later poll may consequently create new work.

After a wait raises `TimeoutError`, `done()` separates a still-pending future
from a terminal one. If completion races with that check, a done future is read
again without a timeout so its real value, cancellation, or exception wins.
This makes a terminal operation-raised `TimeoutError` propagate and clear just
like any other terminal failure.

`cancel_and_release()` always clears the helper, attempts `Future.cancel()`,
and returns the exact handle. Cancellation can prevent work that has not
started, but it cannot stop work already running. If the returned future is not
cancelled, the caller has explicitly taken ownership and must observe or clean
up its eventual outcome before deciding whether overlapping work is safe.

This class does not own an executor, force thread termination, impose an
operation deadline, or coordinate several owners. Timeouts bound individual
waits only; they neither cancel the future nor change the underlying work.

## Related Snippets

<!-- catalog:related:start -->
- [Collect a Bounded Thread-Pool Batch Under One Deadline](collect-a-bounded-thread-pool-batch-under-one-deadline.md)
- [Collect Thread-Pool Results and Errors as Futures Complete](collect-thread-pool-results-and-errors-as-futures-complete.md)
- [Submit a Callable with a Snapshot of the Current Context](submit-a-callable-with-a-snapshot-of-the-current-context.md)
<!-- catalog:related:end -->
