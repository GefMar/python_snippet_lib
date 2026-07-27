---
title: "Cache One Zero-Argument Method Result per Weakly Referenced Instance"
snippet_type: pattern
use_cases:
  - caching
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - bypass-an-lru-cache-with-a-per-call-predicate.md
  - ../reliability-resilience/cache-values-with-a-monotonic-ttl-and-early-jitter.md
---

# Cache One Zero-Argument Method Result per Weakly Referenced Instance

## Idea and Problem

Cache one successful zero-argument method result per object without letting the cache's key keep that object alive.

A descriptor stores values in `WeakKeyDictionary`, so an entry normally
disappears when its owner is collected. The deliberately narrow contract
caches `None`, leaves exceptions uncached, and accepts only owners with ordinary
identity equality and hashing so distinct objects cannot alias one cache entry.

## When to Use

Use this pattern when a synchronous calculation is stable for an object's
entire lifetime, has no call arguments, and repeated work is materially more
expensive than retaining one result. Owners must support weak references and
use `object` identity equality and hashing. Prefer `functools.cached_property`
when storing the value on the instance is acceptable, or a bounded ordinary
cache when entries need arguments, expiry, invalidation, or cross-instance
sharing.

## Implementation

```python
import weakref
from collections.abc import Callable
from typing import Generic, Self, TypeVar, overload


OwnerT = TypeVar("OwnerT")
ValueT = TypeVar("ValueT")


class weak_cached_method(Generic[OwnerT, ValueT]):
    def __init__(self, method: Callable[[OwnerT], ValueT]) -> None:
        if not callable(method):
            raise TypeError("method must be callable")
        self._method = method
        self._values: weakref.WeakKeyDictionary[OwnerT, ValueT] = (
            weakref.WeakKeyDictionary()
        )

    @overload
    def __get__(self, instance: None, owner: type[OwnerT]) -> Self: ...

    @overload
    def __get__(
        self,
        instance: OwnerT,
        owner: type[OwnerT] | None = None,
    ) -> Callable[[], ValueT]: ...

    def __get__(
        self,
        instance: OwnerT | None,
        owner: type[OwnerT] | None = None,
    ) -> Self | Callable[[], ValueT]:
        if instance is None:
            return self

        instance_type = type(instance)
        if (
            instance_type.__eq__ is not object.__eq__
            or instance_type.__hash__ is not object.__hash__
        ):
            raise TypeError("cached owners must use identity equality and hashing")
        try:
            weakref.ref(instance)
        except TypeError:
            raise TypeError("cached owners must support weak references") from None

        def read_or_compute() -> ValueT:
            try:
                return self._values[instance]
            except KeyError:
                value = self._method(instance)
                self._values[instance] = value
                return value

        return read_or_compute

    @property
    def cached_instance_count(self) -> int:
        return len(self._values)
```

## Example

```python
import gc


class Calculation:
    def __init__(self, factor: int) -> None:
        self.factor = factor
        self.calls = 0

    @weak_cached_method
    def result(self) -> int:
        self.calls += 1
        return self.factor * 2


class OptionalCalculation:
    def __init__(self) -> None:
        self.calls = 0

    @weak_cached_method
    def result(self) -> None:
        self.calls += 1
        return None


first = Calculation(3)
second = Calculation(7)
first_result = (first.result(), first.result())
second_result = second.result()
optional = OptionalCalculation()
optional_result = (optional.result(), optional.result())

first_reference = weakref.ref(first)
del first
gc.collect()

assert (
    first_result,
    second_result,
    second.calls,
    optional_result,
    optional.calls,
    first_reference(),
    Calculation.result.cached_instance_count,
) == ((6, 6), 14, 1, (None, None), 1, None, 1)
```

## Trade-offs and Limitations

`WeakKeyDictionary` is not thread-safe, so concurrent first calls can compute
more than once or race during mutation. There is no TTL, explicit invalidation,
argument support, or exception caching, and every live owner can add one entry.
The returned value is shared, so callers can corrupt later reads if it is
mutable. Most importantly, a cached value that directly or indirectly retains
its owner prevents collection and defeats the weak-key benefit. A retained
bound wrapper also keeps its instance alive just like an ordinary bound method.
Classes with value equality, custom hashing, or no weak-reference support are
rejected rather than given surprising cache identity semantics.

## Related Snippets

<!-- catalog:related:start -->
- [Bypass an LRU Cache with a Per-Call Predicate](bypass-an-lru-cache-with-a-per-call-predicate.md)
- [Cache Values with a Monotonic TTL and Early Jitter](../reliability-resilience/cache-values-with-a-monotonic-ttl-and-early-jitter.md)
<!-- catalog:related:end -->
