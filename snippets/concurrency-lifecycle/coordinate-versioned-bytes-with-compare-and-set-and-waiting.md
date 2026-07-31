---
title: "Coordinate Versioned Bytes with Compare-and-Set and Waiting"
snippet_type: pattern
use_cases:
  - concurrency-control
  - resource-management
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - coalesce-concurrent-thread-calls-by-exact-key-with-one-shared-result-cell.md
  - track-current-and-peak-scoped-work-with-a-synchronized-counter.md
  - ../reliability-resilience/wait-for-a-predicate-until-a-monotonic-deadline.md
---

# Coordinate Versioned Bytes with Compare-and-Set and Waiting

## Idea and Problem

Publish immutable process-local state only from the revision a writer actually observed, while letting threads wait for any strictly newer snapshot without a lost-wakeup race.

One `threading.Condition` owns the `(revision, value)` pair. Compare-and-set
checks and replaces that pair under the condition lock, then notifies every
waiter before releasing it. A waiter checks the revision predicate under the
same lock before sleeping and after every wake. Consequently, an update just
before the wait is observed immediately instead of being missed between a
separate check and notification subscription.

Exact immutable bytes make returned snapshots safe to share. A successful
same-value publication still advances the revision: revisions describe
publications, not byte inequality, and therefore avoid an ABA-by-value
ambiguity for callers.

## When to Use

Use this cell when a bounded number of threads in one process share a latest
configuration blob, serialized state, or checkpoint and writers must reject
work based on a stale snapshot. Readers that only need the latest state can
wait on its revision instead of polling.

Use a queue when every intermediate update must be delivered. Use a database,
file protocol, or distributed coordination service when state must survive a
process or coordinate different machines. Use an asynchronous condition when
the callers are tasks on an event loop rather than operating-system threads.

## Implementation

```python
import math
import time
from dataclasses import dataclass
from threading import Condition, Lock

_MAX_CELL_VALUE_BYTES = 1_048_576
_MAX_CELL_REVISION = (1 << 63) - 1
_MAX_CELL_WAIT_SECONDS = 86_400.0


@dataclass(frozen=True, slots=True)
class VersionedBytes:
    revision: int
    value: bytes


def _next_cell_revision(revision: int) -> int:
    if revision == _MAX_CELL_REVISION:
        raise OverflowError("the versioned cell revision is exhausted")
    return revision + 1


class VersionedBytesCell:
    def __init__(self, initial: bytes, *, max_value_bytes: int = 65_536) -> None:
        if type(max_value_bytes) is not int:
            raise TypeError("max_value_bytes must be an exact integer")
        if not 1 <= max_value_bytes <= _MAX_CELL_VALUE_BYTES:
            raise ValueError("max_value_bytes is outside 1..1,048,576")
        self._validate_value(initial, max_value_bytes)

        self._max_value_bytes = max_value_bytes
        self._condition = Condition(Lock())
        self._current = VersionedBytes(revision=0, value=initial)

    @staticmethod
    def _validate_value(value: bytes, maximum: int) -> None:
        if type(value) is not bytes:
            raise TypeError("cell values must be exact immutable bytes")
        if len(value) > maximum:
            raise ValueError("cell value exceeds max_value_bytes")

    @staticmethod
    def _validate_revision(revision: int) -> None:
        if type(revision) is not int:
            raise TypeError("revision must be an exact integer")
        if not 0 <= revision <= _MAX_CELL_REVISION:
            raise ValueError("revision is outside the signed 63-bit range")

    def snapshot(self) -> VersionedBytes:
        with self._condition:
            return self._current

    def compare_and_set(
        self,
        expected_revision: int,
        value: bytes,
    ) -> VersionedBytes | None:
        """Publish value from the current revision, or return None on a miss."""
        self._validate_revision(expected_revision)
        self._validate_value(value, self._max_value_bytes)

        with self._condition:
            if self._current.revision != expected_revision:
                return None
            next_revision = _next_cell_revision(self._current.revision)
            self._current = VersionedBytes(next_revision, value)
            self._condition.notify_all()
            return self._current

    def wait_for_newer(
        self,
        revision: int,
        *,
        timeout: float,
    ) -> VersionedBytes | None:
        """Return a newer snapshot, or None after the finite wait budget."""
        self._validate_revision(revision)
        if type(timeout) not in (int, float):
            raise TypeError("timeout must be an exact integer or float")
        try:
            duration = float(timeout)
        except OverflowError:
            raise ValueError("timeout must be finite") from None
        if not math.isfinite(duration) or not 0.0 <= duration <= _MAX_CELL_WAIT_SECONDS:
            raise ValueError("timeout must be finite and within 0..86,400 seconds")

        deadline = time.monotonic() + duration
        with self._condition:
            if revision > self._current.revision:
                raise ValueError("revision is newer than the cell")
            while self._current.revision <= revision:
                remaining = deadline - time.monotonic()
                if remaining <= 0.0:
                    return None
                self._condition.wait(remaining)
            return self._current
```

## Example

```python
from threading import Barrier, Thread

cell = VersionedBytesCell(b"initial")
assert cell.snapshot() == VersionedBytes(0, b"initial")

# Several waiters cannot miss the publication, even if it wins the scheduling
# race and happens before an individual waiter enters Condition.wait().
waiter_gate = Barrier(4)
observed: list[VersionedBytes | None] = [None, None, None]


def observe(index: int) -> None:
    waiter_gate.wait()
    observed[index] = cell.wait_for_newer(0, timeout=2.0)


waiters = [Thread(target=observe, args=(index,)) for index in range(3)]
for waiter in waiters:
    waiter.start()
waiter_gate.wait()
published = cell.compare_and_set(0, b"ready")
for waiter in waiters:
    waiter.join(timeout=2.0)

assert published == VersionedBytes(1, b"ready")
assert observed == [published, published, published]
assert all(not waiter.is_alive() for waiter in waiters)

# Two writers based on revision one race; exactly one may publish revision two.
writer_gate = Barrier(3)
outcomes: list[VersionedBytes | None] = [None, None]


def publish(index: int) -> None:
    writer_gate.wait()
    outcomes[index] = cell.compare_and_set(1, f"value-{index}".encode())


writers = [Thread(target=publish, args=(index,)) for index in range(2)]
for writer in writers:
    writer.start()
writer_gate.wait()
for writer in writers:
    writer.join(timeout=2.0)

successes = [outcome for outcome in outcomes if outcome is not None]
assert len(successes) == 1
assert successes[0].revision == 2
assert cell.snapshot() == successes[0]
assert cell.wait_for_newer(1, timeout=0) == successes[0]
assert cell.wait_for_newer(2, timeout=0) is None

# Same bytes are still a new publication, while an old expected revision misses.
same_value = cell.compare_and_set(2, successes[0].value)
assert same_value == VersionedBytes(3, successes[0].value)
assert cell.compare_and_set(2, b"stale") is None


def raises(error_type: type[BaseException], operation: object) -> bool:
    try:
        operation()  # type: ignore[operator]
    except error_type:
        return True
    return False


assert raises(ValueError, lambda: cell.wait_for_newer(4, timeout=0))
assert raises(TypeError, lambda: cell.compare_and_set(True, b"bad"))
assert raises(ValueError, lambda: VersionedBytesCell(b"xx", max_value_bytes=1))
assert raises(OverflowError, lambda: _next_cell_revision(_MAX_CELL_REVISION))
```

## Trade-offs and Limitations

This is a latest-state primitive, not an event log. Several rapid successful
publications may coalesce from a waiter's perspective; the returned snapshot
is merely the first newer state that thread observes after reacquiring the
lock. `notify_all()` also wakes every waiter, so callers must bound thread
counts and accept the resulting contention.

The timeout is a maximum condition-wait budget measured against one monotonic
deadline. The method may return after that deadline because lock reacquisition
and thread scheduling latency are outside the wait budget. A failed
compare-and-set does not notify, and an exhausted revision raises before
changing state.

The cell offers no history, callbacks, fairness, persistence, interprocess
coordination, distributed consensus, fencing token for an external resource,
or authentication of the bytes. Callbacks should run after the method returns,
never while the condition lock is held.

## Related Snippets

<!-- catalog:related:start -->
- [Coalesce Concurrent Thread Calls by Exact Key with One Shared Result Cell](coalesce-concurrent-thread-calls-by-exact-key-with-one-shared-result-cell.md)
- [Track Current and Peak Scoped Work with a Synchronized Counter](track-current-and-peak-scoped-work-with-a-synchronized-counter.md)
- [Wait for a Predicate Until a Monotonic Deadline](../reliability-resilience/wait-for-a-predicate-until-a-monotonic-deadline.md)
<!-- catalog:related:end -->
