---
title: "Guard Readers with a Writer-Priority Read-Write Lock"
snippet_type: pattern
use_cases:
  - concurrency-control
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - collect-thread-pool-results-and-errors-as-futures-complete.md
  - drain-bounded-deferred-writes-outside-the-queue-lock.md
  - ../testing-tooling/wait-for-named-queue-conditions-under-one-monotonic-deadline.md
---

# Guard Readers with a Writer-Priority Read-Write Lock

## Idea and Problem

Allow several threads to read shared state concurrently while ensuring that later readers cannot continually bypass a writer that is already waiting.

A `Condition` protects sets of reader and queued-writer ownership tokens plus
the active writer token. Readers wait whenever a writer is active or queued; writers wait until
both the active writer and all readers are gone. Context managers restore the
state even when a body or an acquisition is interrupted by `BaseException`.

## When to Use

Use this pattern for thread-shared, process-local state when reads are long
enough and concurrent enough to justify more machinery than one mutex. Every
access to the protected state must follow the same protocol. Measure the real
workload first: a plain `Lock` is easier to reason about and often faster for
short critical sections.

## Implementation

```python
import threading
from contextlib import contextmanager
from collections.abc import Iterator


class WriterPriorityReadWriteLock:
    def __init__(self) -> None:
        self._condition = threading.Condition(threading.Lock())
        self._reader_owners: set[threading.Thread] = set()
        self._writer_owner: threading.Thread | None = None
        self._waiting_writer_owners: set[threading.Thread] = set()
        self._writer_queued = threading.Event()

    @property
    def waiting_writers(self) -> int:
        with self._condition:
            return len(self._waiting_writer_owners)

    def _new_owner(self) -> threading.Thread:
        owner = threading.current_thread()
        with self._condition:
            if (
                owner in self._reader_owners
                or self._writer_owner is owner
                or owner in self._waiting_writer_owners
            ):
                raise RuntimeError("the lock is not reentrant or upgradeable")
        return owner

    @contextmanager
    def read(self) -> Iterator[None]:
        owner = self._new_owner()
        try:
            with self._condition:
                while self._writer_owner is not None or self._waiting_writer_owners:
                    self._condition.wait()
                self._reader_owners.add(owner)
            yield
        finally:
            with self._condition:
                self._reader_owners.discard(owner)
                if not self._reader_owners:
                    self._condition.notify_all()

    @contextmanager
    def write(self) -> Iterator[None]:
        owner = self._new_owner()
        try:
            with self._condition:
                self._waiting_writer_owners.add(owner)
                self._writer_queued.set()
                while self._writer_owner is not None or self._reader_owners:
                    self._condition.wait()
                self._writer_owner = owner
                self._waiting_writer_owners.discard(owner)
                if not self._waiting_writer_owners:
                    self._writer_queued.clear()
                self._condition.notify_all()
            yield
        finally:
            with self._condition:
                self._waiting_writer_owners.discard(owner)
                if not self._waiting_writer_owners:
                    self._writer_queued.clear()
                if self._writer_owner is owner:
                    self._writer_owner = None
                self._condition.notify_all()
```

## Example

```python
lock = WriterPriorityReadWriteLock()
writer_entered = threading.Event()
release_writer = threading.Event()
entry_order: list[str] = []


def writer() -> None:
    with lock.write():
        entry_order.append("writer")
        writer_entered.set()
        release_writer.wait()


def late_reader() -> None:
    with lock.read():
        entry_order.append("late-reader")


with lock.read():
    writer_thread = threading.Thread(target=writer)
    writer_thread.start()
    lock._writer_queued.wait()
    reader_thread = threading.Thread(target=late_reader)
    reader_thread.start()

writer_entered.wait()
release_writer.set()
writer_thread.join()
reader_thread.join()


class MarkerError(Exception):
    pass


try:
    with lock.read():
        raise MarkerError
except MarkerError:
    pass
with lock.write():
    write_after_reader_error = True

try:
    with lock.write():
        raise MarkerError
except MarkerError:
    pass
with lock.read():
    read_after_writer_error = True

assert (
    entry_order,
    lock.waiting_writers,
    write_after_reader_error,
    read_after_writer_error,
) == (["writer", "late-reader"], 0, True, True)
```

## Trade-offs and Limitations

Writer priority is not fairness: `Condition` makes no FIFO or bounded-wait
guarantee, and a continuous stream of writers can starve readers. The lock is
not reentrant and does not support read-to-write upgrades, acquisition
timeouts, asynchronous tasks, multiple processes, or recovery from a thread
that never leaves its context. The testing event is private observation state,
not part of the synchronization protocol. Correctness still depends on every
caller protecting the same mutable state with the appropriate context.

## Related Snippets

<!-- catalog:related:start -->
- [Collect Thread-Pool Results and Errors as Futures Complete](collect-thread-pool-results-and-errors-as-futures-complete.md)
- [Drain Bounded Deferred Writes Outside the Queue Lock](drain-bounded-deferred-writes-outside-the-queue-lock.md)
- [Wait for Named Queue Conditions Under One Monotonic Deadline](../testing-tooling/wait-for-named-queue-conditions-under-one-monotonic-deadline.md)
<!-- catalog:related:end -->
