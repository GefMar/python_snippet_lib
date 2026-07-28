---
title: "Acquire a Fixed Set of Keyed Thread Locks in Canonical Order"
snippet_type: pattern
use_cases:
  - concurrency-control
  - resource-management
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - guard-readers-with-a-writer-priority-read-write-lock.md
  - run-one-async-operation-with-a-bounded-resource-stack.md
  - prevent-overlapping-posix-jobs-with-a-nonblocking-file-lock.md
---

# Acquire a Fixed Set of Keyed Thread Locks in Canonical Order

## Idea and Problem

Acquire every lock for one multi-key critical section in a shared canonical order so competing threads cannot form a lock-order cycle.

One closed registry owns the locks and assigns their order from key declaration
positions. A hold validates its complete request before waiting, acquires nested
lock contexts in that order, and unwinds them in reverse whether acquisition or
the protected body raises.

## When to Use

Use this pattern when one process has a fixed, small set of keyed resources and
some synchronous operations must exclude concurrent access to several of them.
Every operation that can overlap those resources must use the same registry,
request its full key set in one non-nested hold, and enter and exit the context
in the same thread.

Prefer one ordinary lock when most operations need most keys or critical
sections are short enough that finer locking adds no useful concurrency. Use a
transaction, asynchronous lock, process lock, or distributed coordinator when
the protected boundary requires those semantics instead.

## Implementation

```python
import threading
from collections.abc import Iterator
from contextlib import contextmanager

_MAX_REGISTRY_KEYS = 256
_MAX_KEYS_PER_HOLD = 32
_MAX_KEY_CHARACTERS = 128
_MAX_KEY_BYTES = 128
_MAX_TOTAL_KEY_BYTES = 4_096


def _key_size(value: object, *, label: str) -> int:
    if type(value) is not str:
        raise TypeError(f"{label} must be an exact string")
    if not 1 <= len(value) <= _MAX_KEY_CHARACTERS:
        raise ValueError(f"{label} has an unsupported character count")
    try:
        encoded = value.encode("utf-8")
    except UnicodeEncodeError:
        raise ValueError(f"{label} must be valid UTF-8 text") from None
    if len(encoded) > _MAX_KEY_BYTES:
        raise ValueError(f"{label} exceeds the encoded byte limit")
    return len(encoded)


@contextmanager
def _acquire_in_order(
    locks: tuple[threading.Lock, ...],
    position: int = 0,
) -> Iterator[None]:
    if position == len(locks):
        yield
        return

    with locks[position]:
        with _acquire_in_order(locks, position + 1):
            yield


class FixedKeyLocks:
    """Own one closed set of process-local locks under declaration ordering."""

    __slots__ = (
        "_active_threads",
        "_keys",
        "_locks",
        "_positions",
        "_state_lock",
    )

    def __init__(self, keys: tuple[str, ...]) -> None:
        if type(keys) is not tuple:
            raise TypeError("keys must be an exact tuple")
        if not 1 <= len(keys) <= _MAX_REGISTRY_KEYS:
            raise ValueError("key count is outside the supported range")

        positions: dict[str, int] = {}
        total_bytes = 0
        for position, key in enumerate(keys):
            size = _key_size(key, label=f"keys[{position}]")
            if size > _MAX_TOTAL_KEY_BYTES - total_bytes:
                raise ValueError("registry keys exceed the aggregate byte limit")
            if key in positions:
                raise ValueError("registry keys must be unique")
            positions[key] = position
            total_bytes += size

        self._keys = keys
        self._positions = positions
        self._locks = tuple(threading.Lock() for _ in keys)
        self._state_lock = threading.Lock()
        self._active_threads: set[threading.Thread] = set()

    def _canonical_request(self, requested: object) -> tuple[str, ...]:
        if type(requested) is not tuple:
            raise TypeError("requested must be an exact tuple")
        if not 1 <= len(requested) <= _MAX_KEYS_PER_HOLD:
            raise ValueError("requested key count is outside the supported range")

        seen: set[str] = set()
        positions: list[int] = []
        total_bytes = 0
        for index, key in enumerate(requested):
            size = _key_size(key, label=f"requested[{index}]")
            if size > _MAX_TOTAL_KEY_BYTES - total_bytes:
                raise ValueError("requested keys exceed the aggregate byte limit")
            if key in seen:
                raise ValueError("requested keys must be unique")
            try:
                position = self._positions[key]
            except KeyError:
                raise ValueError("every requested key must be registered") from None
            seen.add(key)
            positions.append(position)
            total_bytes += size

        return tuple(self._keys[position] for position in sorted(positions))

    @contextmanager
    def hold(self, requested: tuple[str, ...]) -> Iterator[tuple[str, ...]]:
        """Hold validated keys canonically and yield that canonical key tuple."""
        canonical = self._canonical_request(requested)
        owner = threading.current_thread()
        registered = False

        try:
            with self._state_lock:
                if owner in self._active_threads:
                    raise RuntimeError("the registry does not support nested holds")
                registered = True
                self._active_threads.add(owner)

            ordered_locks = tuple(self._locks[self._positions[key]] for key in canonical)
            with _acquire_in_order(ordered_locks):
                yield canonical
        finally:
            if registered:
                with self._state_lock:
                    self._active_threads.discard(owner)
```

## Example

```python
balances = {"left": 100, "right": 0}
locks = FixedKeyLocks(("left", "right", "audit"))


def move_units() -> None:
    for _ in range(25):
        with locks.hold(("right", "left")) as canonical:
            assert canonical == ("left", "right")
            balances["left"] -= 1
            balances["right"] += 1


workers = [threading.Thread(target=move_units) for _ in range(4)]
for worker in workers:
    worker.start()
for worker in workers:
    worker.join()

nested_rejected = False
with locks.hold(("audit",)):
    try:
        with locks.hold(("left",)):
            pass
    except RuntimeError:
        nested_rejected = True


class StopProbe(BaseException):
    pass


try:
    with locks.hold(("right",)):
        raise StopProbe
except StopProbe:
    pass

with locks.hold(("right",)) as reusable:
    pass

assert (balances, nested_rejected, reusable) == (
    {"left": 0, "right": 100},
    True,
    ("right",),
)
```

## Trade-offs and Limitations

Registry construction takes time proportional to the admitted key bytes and
allocates one lock per key. Each hold validates `r` requested keys, sorts their
positions in `O(r log r)` time, and may then block indefinitely. Holding
several locks until the body exits can reduce concurrency, and a thread that
never leaves the body keeps every acquired lock unavailable.

Canonical ordering prevents a lock-order cycle only when every access to these
keys uses this one registry, requests every required key together, and avoids
nesting, and when no external lock participates in the ordering cycle. The
internal acquisition stack releases prior acquisitions in reverse order when a
later acquisition is interrupted by a signal or another `BaseException`, and it
does the same when the body raises. There is no timeout, FIFO fairness, bounded
waiting, reentrancy, dead-thread recovery, or transaction rollback.

The registry does not expose its locks and rejects same-registry nested use by
one thread. A hold context must not be handed to another thread for exit. The
keys are exact, case-sensitive strings without normalization, and the registry
cannot add keys after construction. This pattern is synchronous and
process-local; it provides no asyncio, inter-process, filesystem, or
distributed locking.

## Related Snippets

<!-- catalog:related:start -->
- [Guard Readers with a Writer-Priority Read-Write Lock](guard-readers-with-a-writer-priority-read-write-lock.md)
- [Run One Async Operation with a Bounded Resource Stack](run-one-async-operation-with-a-bounded-resource-stack.md)
- [Prevent Overlapping POSIX Jobs with a Nonblocking File Lock](prevent-overlapping-posix-jobs-with-a-nonblocking-file-lock.md)
<!-- catalog:related:end -->
