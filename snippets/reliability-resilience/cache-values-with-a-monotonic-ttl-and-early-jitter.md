---
title: "Cache Values with a Monotonic TTL and Early Jitter"
snippet_type: recipe
use_cases:
  - caching
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - wait-for-a-predicate-until-a-monotonic-deadline.md
  - hold-a-switch-active-through-a-monotonic-cooldown.md
---

# Cache Values with a Monotonic TTL and Early Jitter

## Idea and Problem

Bound an in-process cache and spread its expirations before a monotonic freshness ceiling without making cached None values look like misses.

Each write subtracts a random fraction of the configured jitter from the base
TTL, so no entry outlives that TTL. An explicit lookup result separates a miss
from a hit whose value is `None`, while a hard entry cap prevents unbounded key
growth.

## When to Use

Use this recipe for a small, single-owner cache whose keys and values are safe
to retain in one process. Choose a TTL that represents the maximum acceptable
staleness, a smaller jitter window, and a capacity based on known workload
bounds. Inject a deterministic clock and fraction source in tests. Use a
concurrent cache with single-flight loading when multiple threads can request
the same missing key.

## Implementation

```python
import math
from collections.abc import Callable
from dataclasses import dataclass
from random import random
from time import monotonic
from typing import Generic, TypeVar


K = TypeVar("K")
V = TypeVar("V")


class CacheFullError(RuntimeError):
    pass


@dataclass(frozen=True, slots=True)
class CacheLookup(Generic[V]):
    hit: bool
    value: V | None = None


@dataclass(frozen=True, slots=True)
class _CacheEntry(Generic[V]):
    value: V
    expires_at: float


def _finite_number(value: int | float, *, name: str) -> float:
    if isinstance(value, bool) or not isinstance(value, (int, float)):
        raise TypeError(f"{name} must be an integer or float")
    try:
        converted = float(value)
    except OverflowError as error:
        raise ValueError(f"{name} must be representable as a float") from error
    if not math.isfinite(converted):
        raise ValueError(f"{name} must be finite")
    return converted


class JitteredTtlCache(Generic[K, V]):
    def __init__(
        self,
        *,
        ttl: int | float,
        max_jitter: int | float = 0,
        max_entries: int,
        clock: Callable[[], int | float] = monotonic,
        random_fraction: Callable[[], int | float] = random,
    ) -> None:
        ttl_value = _finite_number(ttl, name="ttl")
        jitter_value = _finite_number(max_jitter, name="max_jitter")
        if ttl_value <= 0:
            raise ValueError("ttl must be positive")
        if not 0 <= jitter_value < ttl_value:
            raise ValueError("max_jitter must be non-negative and smaller than ttl")
        if isinstance(max_entries, bool) or not isinstance(max_entries, int):
            raise TypeError("max_entries must be an integer")
        if max_entries <= 0:
            raise ValueError("max_entries must be positive")
        if not callable(clock) or not callable(random_fraction):
            raise TypeError("clock and random_fraction must be callable")

        self._ttl = ttl_value
        self._max_jitter = jitter_value
        self._max_entries = max_entries
        self._clock = clock
        self._random_fraction = random_fraction
        self._entries: dict[K, _CacheEntry[V]] = {}
        self._last_now: float | None = None

    def _now(self) -> float:
        now = _finite_number(self._clock(), name="clock result")
        if self._last_now is not None and now < self._last_now:
            raise ValueError("clock must not move backwards")
        self._last_now = now
        return now

    def _deadline(self, now: float) -> float:
        fraction = 0.0
        if self._max_jitter:
            fraction = _finite_number(
                self._random_fraction(),
                name="random_fraction result",
            )
            if not 0 <= fraction <= 1:
                raise ValueError("random_fraction result must be between zero and one")
        lifetime = self._ttl - self._max_jitter * fraction
        deadline = now + lifetime
        if not math.isfinite(deadline) or deadline <= now:
            raise OverflowError("expiration deadline is not representable")
        return deadline

    def _purge_expired(self, now: float) -> None:
        expired = [
            key
            for key, entry in self._entries.items()
            if now >= entry.expires_at
        ]
        for key in expired:
            del self._entries[key]

    def set(self, key: K, value: V) -> V:
        now = self._now()
        self._purge_expired(now)
        if key not in self._entries and len(self._entries) >= self._max_entries:
            raise CacheFullError("cache has reached max_entries")
        self._entries[key] = _CacheEntry(value, self._deadline(now))
        return value

    def lookup(self, key: K) -> CacheLookup[V]:
        now = self._now()
        entry = self._entries.get(key)
        if entry is None:
            return CacheLookup(hit=False)
        if now >= entry.expires_at:
            del self._entries[key]
            return CacheLookup(hit=False)
        return CacheLookup(hit=True, value=entry.value)

    def get_or_set(self, key: K, factory: Callable[[], V]) -> V:
        if not callable(factory):
            raise TypeError("factory must be callable")
        cached = self.lookup(key)
        if cached.hit:
            return cached.value
        if len(self._entries) >= self._max_entries:
            self._purge_expired(self._now())
            if len(self._entries) >= self._max_entries:
                raise CacheFullError("cache has reached max_entries")
        return self.set(key, factory())
```

## Example

```python
now = [100.0]
fractions = iter([0.5, 0.0])
cache: JitteredTtlCache[str, object | None] = JitteredTtlCache(
    ttl=10,
    max_jitter=4,
    max_entries=1,
    clock=lambda: now[0],
    random_fraction=lambda: next(fractions),
)
factory_calls = 0


def load_optional() -> None:
    global factory_calls
    factory_calls += 1
    return None


first = cache.get_or_set("optional", load_optional)
second = cache.get_or_set("optional", load_optional)
now[0] = 107.999
before_boundary = cache.lookup("optional")
now[0] = 108.0
at_boundary = cache.lookup("optional")
cache.set("replacement", 3)

try:
    cache.set("overflow", 4)
except CacheFullError:
    capacity_enforced = True
else:
    capacity_enforced = False

assert (
    first,
    second,
    factory_calls,
    before_boundary,
    at_boundary,
    capacity_enforced,
) == (
    None,
    None,
    1,
    CacheLookup(hit=True, value=None),
    CacheLookup(hit=False),
    True,
)
```

## Trade-offs and Limitations

Early jitter shortens some lifetimes; it spreads expirations but neither
prevents a cache stampede nor coordinates different processes. The cache is
not thread-safe and performs no single-flight loading, LRU eviction, size-based
accounting, refresh-ahead, persistence, or failure caching. `set()` scans all
entries to reclaim expired capacity, while `lookup()` removes only the queried
expired entry. Keys and values are held by strong reference until replacement,
expiration, or cache disposal. Monotonic deadlines are normally process-local
and should not be persisted across restarts.

## Related Snippets

<!-- catalog:related:start -->
- [Wait for a Predicate Until a Monotonic Deadline](wait-for-a-predicate-until-a-monotonic-deadline.md)
- [Hold a Switch Active Through a Monotonic Cooldown](hold-a-switch-active-through-a-monotonic-cooldown.md)
<!-- catalog:related:end -->
