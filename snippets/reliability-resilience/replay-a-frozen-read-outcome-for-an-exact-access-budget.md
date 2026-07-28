---
title: "Replay a Frozen Read Outcome for an Exact Access Budget"
snippet_type: "pattern"
use_cases:
  - "caching"
  - "concurrency-control"
  - "retry-recovery"
tested_python:
  - "3.14"
dependencies: []
related:
  - "cache-values-with-a-monotonic-ttl-and-early-jitter.md"
  - "../python-language/cache-one-zero-argument-method-result-per-weakly-referenced-instance.md"
  - "../concurrency-lifecycle/initialize-one-shared-resource-lazily-with-serialized-retries.md"
---

# Replay a Frozen Read Outcome for an Exact Access Budget

## Idea and Problem

Cache one immutable outcome from an idempotent provider for an exact access budget while a per-key lock preserves call cadence without blocking unrelated keys.

Freeze successful values as `bytes` and reduce provider exceptions to small immutable records. Each failed access raises a new wrapper, so the cache never retains an exception, traceback, or frame from provider code.

## When to Use

Use this for a small, fixed set of non-personalized reads when access count is a better refresh signal than elapsed time. It fits process-local adapters whose reads are idempotent and whose byte results have a firm size limit.

Do not use it for writes, personalized responses, unbounded key spaces, or data that must become fresh after a wall-clock deadline.

## Implementation

```python
import re
import threading
from collections.abc import Callable
from dataclasses import dataclass
from types import MappingProxyType

_KEY_PATTERN = re.compile(r"[a-z][a-z0-9._:-]{0,63}", re.ASCII)
_ERROR_PATTERN = re.compile(r"[A-Za-z_][A-Za-z0-9_]{0,63}", re.ASCII)
_MAX_SOURCES = 32
_MAX_REPLAY_ACCESSES = 10_000
_MAX_VALUE_BYTES = 1_048_576


@dataclass(frozen=True, slots=True)
class FrozenReadSource:
    key: str
    provider: Callable[[], bytes]


@dataclass(frozen=True, slots=True)
class ReadSuccess:
    value: bytes


@dataclass(frozen=True, slots=True)
class ReadFailure:
    error_kind: str


FrozenReadOutcome = ReadSuccess | ReadFailure


class FrozenReadError(RuntimeError):
    def __init__(self, key: str, failure: ReadFailure) -> None:
        self.key = key
        self.failure = failure
        super().__init__(f"read {key!r} failed with {failure.error_kind}")


@dataclass(slots=True)
class _ReadEntry:
    source: FrozenReadSource
    lock: threading.Lock
    outcome: FrozenReadOutcome | None = None
    replay_accesses_left: int = 0


def _bounded_int(value: object, label: str, lower: int, upper: int) -> int:
    if type(value) is not int or not lower <= value <= upper:
        raise ValueError(f"{label} must be an integer from {lower} through {upper}")
    return value


def _canonical_key(value: object) -> str:
    if type(value) is not str or _KEY_PATTERN.fullmatch(value) is None:
        raise ValueError("keys must match [a-z][a-z0-9._:-]{0,63}")
    return value


def _failure_from(error: Exception) -> ReadFailure:
    name = type(error).__name__
    kind = name if _ERROR_PATTERN.fullmatch(name) is not None else "ProviderError"
    return ReadFailure(kind)


class AccessBudgetReadCache:
    def __init__(
        self,
        sources: tuple[FrozenReadSource, ...],
        *,
        replay_accesses: int,
        max_value_bytes: int = 65_536,
    ) -> None:
        if type(sources) is not tuple or not 1 <= len(sources) <= _MAX_SOURCES:
            raise ValueError(f"sources must be a tuple containing 1 to {_MAX_SOURCES} items")
        replay_budget = _bounded_int(
            replay_accesses,
            "replay_accesses",
            0,
            _MAX_REPLAY_ACCESSES,
        )
        value_budget = _bounded_int(
            max_value_bytes,
            "max_value_bytes",
            1,
            _MAX_VALUE_BYTES,
        )

        checked: list[FrozenReadSource] = []
        seen: set[str] = set()
        for source in sources:
            if type(source) is not FrozenReadSource:
                raise TypeError("every source must be a FrozenReadSource")
            key = _canonical_key(source.key)
            if key in seen:
                raise ValueError(f"duplicate source key: {key!r}")
            if not callable(source.provider):
                raise TypeError(f"provider for {key!r} must be callable")
            seen.add(key)
            checked.append(source)

        entries = {source.key: _ReadEntry(source, threading.Lock()) for source in checked}
        self._entries = MappingProxyType(entries)
        self._replay_accesses = replay_budget
        self._max_value_bytes = value_budget

    def read(self, key: str) -> bytes:
        canonical = _canonical_key(key)
        try:
            entry = self._entries[canonical]
        except KeyError:
            raise KeyError(f"unknown read key: {canonical!r}") from None

        with entry.lock:
            if entry.outcome is None or entry.replay_accesses_left == 0:
                entry.outcome = self._refresh(entry.source)
                entry.replay_accesses_left = self._replay_accesses
            else:
                entry.replay_accesses_left -= 1
            outcome = entry.outcome

        if isinstance(outcome, ReadSuccess):
            return outcome.value
        raise FrozenReadError(canonical, outcome) from None

    def _refresh(self, source: FrozenReadSource) -> FrozenReadOutcome:
        try:
            value = source.provider()
        except Exception as error:
            return _failure_from(error)
        if type(value) is not bytes:
            return ReadFailure("InvalidBytesResult")
        if len(value) > self._max_value_bytes:
            return ReadFailure("ValueTooLarge")
        return ReadSuccess(value)
```

## Example

```python
calls = {"document": 0, "unavailable": 0}


def read_document() -> bytes:
    calls["document"] += 1
    return f"revision-{calls['document']}".encode()


def read_unavailable() -> bytes:
    calls["unavailable"] += 1
    raise OSError("private provider detail")


cache = AccessBudgetReadCache(
    (
        FrozenReadSource("document", read_document),
        FrozenReadSource("unavailable", read_unavailable),
    ),
    replay_accesses=2,
)

documents = tuple(cache.read("document") for _ in range(4))
failures: list[FrozenReadError] = []
for _ in range(4):
    try:
        cache.read("unavailable")
    except FrozenReadError as error:
        failures.append(error)

assert documents == (b"revision-1", b"revision-1", b"revision-1", b"revision-2")
assert calls == {"document": 2, "unavailable": 2}
assert len({id(error) for error in failures}) == 4
assert {error.failure.error_kind for error in failures} == {"OSError"}
assert all("private provider detail" not in str(error) for error in failures)
```

## Trade-offs and Limitations

- A live provider call blocks other readers of the same key, while different keys can refresh independently.
- The budget measures completed accesses, not time. A quiet key can remain old indefinitely.
- A cached failure delays another provider attempt until its replay budget is consumed.
- The key set is fixed, with no eviction or explicit invalidation, and all successful values are held in memory.
- Failure records deliberately omit messages and traceback details. Add separate bounded observability at the provider boundary when diagnostics are required.
- The locks coordinate threads in one process; they do not coordinate separate processes or hosts.

## Related Snippets

<!-- catalog:related:start -->
- [Cache Values with a Monotonic TTL and Early Jitter](cache-values-with-a-monotonic-ttl-and-early-jitter.md)
- [Cache One Zero-Argument Method Result per Weakly Referenced Instance](../python-language/cache-one-zero-argument-method-result-per-weakly-referenced-instance.md)
- [Initialize One Shared Resource Lazily with Serialized Retries](../concurrency-lifecycle/initialize-one-shared-resource-lazily-with-serialized-retries.md)
<!-- catalog:related:end -->
