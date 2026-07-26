---
title: "Bypass an LRU Cache with a Per-Call Predicate"
snippet_type: pattern
use_cases:
  - caching
  - performance-optimization
tested_python:
  - "3.14"
dependencies: []
related:
  - ../reliability-resilience/cache-values-with-a-monotonic-ttl-and-early-jitter.md
  - make-a-defensive-copy-at-a-mutable-input-boundary.md
---

# Bypass an LRU Cache with a Per-Call Predicate

## Idea and Problem

Wrap a bounded LRU cache with a predicate that can send one call directly to the underlying function without reading or updating cached state.

The predicate receives the same arguments as the function and runs exactly
once for every invocation, including cache hits. A true result calls the
original function; a false result delegates to `functools.lru_cache`. This
keeps refresh or diagnostic policy outside the cached computation while
preserving its name, documentation, and `__wrapped__` reference.

## When to Use

Use this decorator for a synchronous, deterministic function when most calls
benefit from ordinary bounded memoization but a documented flag or input class
must bypass both cache reads and writes. Cached calls still require hashable
arguments. Prefer an explicit cache object when callers need TTLs, per-key
invalidation, distributed state, or more than one cache policy.

## Implementation

```python
from collections.abc import Callable
from functools import lru_cache, wraps
from typing import ParamSpec, Protocol, TypeVar, cast


P = ParamSpec("P")
ResultT = TypeVar("ResultT")


class _CacheDecorator(Protocol[P]):
    def __call__(
        self,
        function: Callable[P, ResultT],
        /,
    ) -> Callable[P, ResultT]: ...


def bypassable_lru_cache(
    *,
    maxsize: int,
    bypass: Callable[P, bool],
) -> _CacheDecorator[P]:
    if isinstance(maxsize, bool) or not isinstance(maxsize, int):
        raise TypeError("maxsize must be an integer")
    if maxsize <= 0:
        raise ValueError("maxsize must be positive")
    if not callable(bypass):
        raise TypeError("bypass must be callable")

    def decorate(function: Callable[P, ResultT]) -> Callable[P, ResultT]:
        if not callable(function):
            raise TypeError("the decorated value must be callable")

        cached = cast(
            Callable[P, ResultT],
            lru_cache(maxsize=maxsize)(function),
        )

        @wraps(function)
        def wrapper(*args: P.args, **kwargs: P.kwargs) -> ResultT:
            if bypass(*args, **kwargs):
                return function(*args, **kwargs)
            return cached(*args, **kwargs)

        return wrapper

    return decorate
```

## Example

```python
function_calls: list[str] = []
bypass_calls: list[tuple[str, bool]] = []


def force_refresh(key: str, *, refresh: bool = False) -> bool:
    bypass_calls.append((key, refresh))
    return refresh


@bypassable_lru_cache(maxsize=2, bypass=force_refresh)
def render(key: str, *, refresh: bool = False) -> str:
    function_calls.append(key)
    return f"{key}:{len(function_calls)}"


first = render("alpha")
cached = render("alpha")
refreshed = render("alpha", refresh=True)
still_cached = render("alpha")
render("beta")
render("gamma")
after_eviction = render("alpha")

assert (
    first,
    cached,
    refreshed,
    still_cached,
    after_eviction,
    function_calls,
    bypass_calls,
    render.__name__,
) == (
    "alpha:1",
    "alpha:1",
    "alpha:2",
    "alpha:1",
    "alpha:5",
    ["alpha", "alpha", "beta", "gamma", "alpha"],
    [
        ("alpha", False),
        ("alpha", False),
        ("alpha", True),
        ("alpha", False),
        ("beta", False),
        ("gamma", False),
        ("alpha", False),
    ],
    "render",
)
```

## Trade-offs and Limitations

The predicate adds work to every invocation and its exceptions propagate
before either function path runs. A bypass does not invalidate or refresh an
existing entry, so a later ordinary call can still return an older cached
value. The LRU cache retains arguments and results, shares mutable results,
and may treat different keyword ordering as distinct calls. Its internal
mapping is thread-safe, but concurrent misses can still compute more than
once. Unhashable arguments work only when the predicate bypasses the cache.
Do not apply this synchronous decorator to generators, coroutines, impure
functions, or calls that must produce a distinct object every time.

## Related Snippets

<!-- catalog:related:start -->
- [Cache Values with a Monotonic TTL and Early Jitter](../reliability-resilience/cache-values-with-a-monotonic-ttl-and-early-jitter.md)
- [Make a Defensive Copy at a Mutable Input Boundary](make-a-defensive-copy-at-a-mutable-input-boundary.md)
<!-- catalog:related:end -->
