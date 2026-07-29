---
title: "Coalesce Concurrent Thread Calls by Exact Key with One Shared Result Cell"
snippet_type: pattern
use_cases:
  - concurrency-control
  - lifecycle-management
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - initialize-one-shared-resource-lazily-with-serialized-retries.md
  - reuse-one-pending-future-across-non-cancelling-poll-timeouts.md
  - ../reliability-resilience/cache-values-with-a-monotonic-ttl-and-early-jitter.md
---

# Coalesce Concurrent Thread Calls by Exact Key with One Shared Result Cell

## Idea and Problem

Let overlapping thread calls for one key share a single in-progress computation without turning its result into a cache.

The first caller creates a result cell and executes the callback. Followers
find that cell under a short state lock, release the lock, and wait on its
`threading.Event`. The leader publishes either the exact result object or the
exact exception object before waking every follower.

Removing the cell at completion is the admission boundary: a later call leads
a new computation. Different keys never share a cell and may execute at the
same time.

## When to Use

Use this pattern when an idempotent or otherwise safely shared operation is
expensive and concurrent duplicate requests should collapse only while the
first request is active. Typical keys are bounded identifiers for reads,
refreshes, or derivations whose callers accept the same object and failure.

Use a real cache when completed values should be replayed. Use an executor,
queue, or async primitive when execution ownership, scheduling, cancellation,
deadlines, or process coordination must be managed as part of the abstraction.

## Implementation

```python
from collections.abc import Callable
from dataclasses import dataclass, field
from threading import Event, Lock, get_ident
from typing import cast


@dataclass(slots=True)
class _ResultCell:
    owner_thread_id: int
    done: Event = field(default_factory=Event)
    follower_admitted: Event = field(default_factory=Event)
    result: object = None
    error: BaseException | None = None
    succeeded: bool = False


class KeyedSingleFlight[ResultT]:
    def __init__(self, max_in_flight: int) -> None:
        if type(max_in_flight) is not int:
            raise TypeError("max_in_flight must be an exact integer")
        if not 1 <= max_in_flight <= 1_024:
            raise ValueError("max_in_flight is outside the supported range")

        self._max_in_flight = max_in_flight
        self._lock = Lock()
        self._cells: dict[str, _ResultCell] = {}

    def run(
        self,
        key: str,
        operation: Callable[[], ResultT],
    ) -> ResultT:
        """Run or join the one active operation for an exact key."""
        if type(key) is not str:
            raise TypeError("key must be an exact string")
        if not 1 <= len(key) <= 128:
            raise ValueError("key length is outside the supported range")
        try:
            encoded_key = key.encode("utf-8")
        except UnicodeEncodeError as error:
            raise ValueError("key must contain valid Unicode scalar values") from error
        if len(encoded_key) > 512:
            raise ValueError("key exceeds the supported UTF-8 byte length")
        if not callable(operation):
            raise TypeError("operation must be callable")

        caller_thread_id = get_ident()
        with self._lock:
            cell = self._cells.get(key)
            if cell is None:
                if len(self._cells) >= self._max_in_flight:
                    raise RuntimeError("distinct in-flight key limit reached")
                cell = _ResultCell(owner_thread_id=caller_thread_id)
                self._cells[key] = cell
                is_leader = True
            else:
                if cell.owner_thread_id == caller_thread_id:
                    raise RuntimeError("same-thread same-key re-entry is not allowed")
                is_leader = False

        if not is_leader:
            cell.follower_admitted.set()
            cell.done.wait()
            if cell.error is not None:
                raise cell.error
            if not cell.succeeded:
                raise AssertionError("completed result cell has no outcome")
            return cast(ResultT, cell.result)

        try:
            result = operation()
        except BaseException as error:
            cell.error = error
            raise
        else:
            cell.result = result
            cell.succeeded = True
            return result
        finally:
            cell.done.set()
            with self._lock:
                if self._cells.get(key) is cell:
                    del self._cells[key]
```

## Example

```python
from concurrent.futures import ThreadPoolExecutor
from threading import Event

single_flight = KeyedSingleFlight[object](max_in_flight=2)
operation_started = Event()
release_operation = Event()
result_token = object()
call_count = 0


def load_once() -> object:
    global call_count
    call_count += 1
    operation_started.set()
    release_operation.wait()
    return result_token


with ThreadPoolExecutor(max_workers=2) as executor:
    leader = executor.submit(single_flight.run, "item", load_once)
    operation_started.wait()
    with single_flight._lock:
        active_cell = single_flight._cells["item"]
    follower = executor.submit(single_flight.run, "item", load_once)
    active_cell.follower_admitted.wait()
    release_operation.set()
    leader_result = leader.result()
    follower_result = follower.result()

later_token = object()
later_result = single_flight.run("item", lambda: later_token)

try:
    single_flight.run(
        "recursive",
        lambda: single_flight.run("recursive", lambda: None),
    )
except RuntimeError:
    reentry_rejected = True
else:
    reentry_rejected = False

assert call_count == 1
assert leader_result is result_token and follower_result is result_token
assert later_result is later_token
assert reentry_rejected
```

## Trade-offs and Limitations

Map admission is average `O(1)`. Completing one cell wakes its `w` blocked
followers in `O(w)` coordination work. State is `O(k + w)` for `k` active keys
and all waiting followers, excluding callback-owned objects and thread stacks.
The state lock is never held while a callback runs or a follower waits.

Result and exception objects are shared by identity. A follower re-raises the
same exception object with a follower traceback, while the leader uses a bare
re-raise. Catching `BaseException` ensures cleanup for cancellation-like
exceptions, but it does not make arbitrary callbacks safe or interruptible.
One instance has one declared result type, and every use of a given key must
also preserve the same operation meaning.

Only same-thread same-key recursion is detected. Callbacks that synchronously
request other keys, or spawn followers and wait for them, can create dependency
cycles and deadlock. There is no timeout, cancellation, completed-result cache,
fairness promise, async support, process sharing, or leader-recovery mechanism
if the process itself terminates.

## Related Snippets

<!-- catalog:related:start -->
- [Initialize One Shared Resource Lazily with Serialized Retries](initialize-one-shared-resource-lazily-with-serialized-retries.md)
- [Reuse One Pending Future Across Non-Cancelling Poll Timeouts](reuse-one-pending-future-across-non-cancelling-poll-timeouts.md)
- [Cache Values with a Monotonic TTL and Early Jitter](../reliability-resilience/cache-values-with-a-monotonic-ttl-and-early-jitter.md)
<!-- catalog:related:end -->
