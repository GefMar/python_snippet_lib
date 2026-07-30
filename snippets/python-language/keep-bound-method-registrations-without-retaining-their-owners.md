---
title: "Keep Bound-Method Registrations without Retaining Their Owners"
snippet_type: pattern
use_cases:
  - lifecycle-management
  - resource-management
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - cache-one-zero-argument-method-result-per-weakly-referenced-instance.md
  - reuse-live-objects-by-exact-string-key-with-weakvaluedictionary.md
  - attach-best-effort-cleanup-with-weakref-finalize-without-retaining-the-owner.md
---

# Keep Bound-Method Registrations without Retaining Their Owners

## Idea and Problem

Keep a bounded, ordered registry of bound methods without making their owners live as long as the registry.

Each registration receives a distinct integer token and stores a
`weakref.WeakMethod` instead of the bound method itself. Its automatic cleanup
callback closes over only that token and a weak reference to the registry, so
neither the entry nor its callback creates a strong path back to the owner.
Registering the same method repeatedly intentionally creates independent
entries.

## When to Use

Use this pattern when a short-lived object advertises one ordinary synchronous
bound method to a longer-lived in-process registry and explicit deregistration
cannot be guaranteed. Examples include observational hooks and optional local
listeners where losing an entry with its owner is the correct lifecycle.

The consumer decides whether and how to invoke methods returned by `snapshot`.
Use an explicit strong registry when registration owns the listener lifetime,
or a purpose-built event system when callback failures, arguments, priorities,
async dispatch, or concurrent mutation need a defined policy.

## Implementation

```python
import gc
import weakref
from inspect import isasyncgenfunction, iscoroutinefunction
from types import MethodType

_MAX_WEAK_METHOD_REGISTRATIONS = 128


class WeakMethodRegistryCapacityError(RuntimeError):
    """Raised when all bounded registration slots are occupied."""


class WeakMethodRegistry:
    """Keep ordered weak registrations for ordinary synchronous methods."""

    __slots__ = ("__weakref__", "_entries", "_next_token")

    def __init__(self) -> None:
        self._entries: dict[int, weakref.WeakMethod] = {}
        self._next_token = 1

    def _prune_dead(self) -> None:
        dead_tokens = [
            token for token, reference in tuple(self._entries.items()) if reference() is None
        ]
        for token in dead_tokens:
            self._entries.pop(token, None)

    def register(self, method: MethodType) -> int:
        """Register one exact bound method and return its distinct token."""
        if type(method) is not MethodType:
            raise TypeError("method must be an exact bound method")
        if iscoroutinefunction(method) or isasyncgenfunction(method):
            raise TypeError("async methods are outside this registry contract")

        self._prune_dead()
        if len(self._entries) >= _MAX_WEAK_METHOD_REGISTRATIONS:
            raise WeakMethodRegistryCapacityError("the registry already contains 128 live entries")

        token = self._next_token
        self._next_token += 1
        registry_reference = weakref.ref(self)

        def discard_dead_method(
            _reference: weakref.ReferenceType[MethodType],
            *,
            registration_token: int = token,
            weak_registry: weakref.ReferenceType[WeakMethodRegistry] = (registry_reference),
        ) -> None:
            registry = weak_registry()
            if registry is not None:
                registry._entries.pop(registration_token, None)

        try:
            reference = weakref.WeakMethod(method, discard_dead_method)
        except TypeError:
            raise TypeError("the bound method owner must support weak references") from None
        self._entries[token] = reference
        return token

    def remove(self, token: int) -> bool:
        """Remove one token, returning whether it was still registered."""
        if type(token) is not int:
            raise TypeError("token must be an exact integer")
        return self._entries.pop(token, None) is not None

    def snapshot(self) -> tuple[MethodType, ...]:
        """Return live methods in registration order and prune dead entries."""
        live_methods: list[MethodType] = []
        dead_tokens: list[int] = []
        for token, reference in tuple(self._entries.items()):
            method = reference()
            if method is None:
                dead_tokens.append(token)
            else:
                live_methods.append(method)
        for token in dead_tokens:
            self._entries.pop(token, None)
        return tuple(live_methods)

    @property
    def entry_count(self) -> int:
        self._prune_dead()
        return len(self._entries)


```

## Example

```python
class Listener:
    def __init__(self, label: str) -> None:
        self.label = label

    def observe(self) -> str:
        return self.label

    async def observe_async(self) -> str:
        return self.label


def raises(error_type: type[Exception], operation) -> bool:
    try:
        operation()
    except error_type:
        return True
    return False


registry = WeakMethodRegistry()
listener = Listener("alpha")
first_token = registry.register(listener.observe)
second_token = registry.register(listener.observe)
retaining_snapshot = registry.snapshot()
listener_reference = weakref.ref(listener)

assert first_token != second_token
assert tuple(method() for method in retaining_snapshot) == ("alpha", "alpha")

del listener
gc.collect()
retained_by_snapshot = listener_reference() is not None

del retaining_snapshot
gc.collect()
collected_after_snapshot = listener_reference() is None
automatically_pruned = registry.snapshot() == () and registry.entry_count == 0

remaining = Listener("beta")
removed_token = registry.register(remaining.observe)
removed_once = registry.remove(removed_token)
removed_twice = registry.remove(removed_token)

capacity_registry = WeakMethodRegistry()
capacity_owner = Listener("capacity")
capacity_tokens = tuple(
    capacity_registry.register(capacity_owner.observe)
    for _ in range(_MAX_WEAK_METHOD_REGISTRATIONS)
)
capacity_rejected = raises(
    WeakMethodRegistryCapacityError,
    lambda: capacity_registry.register(capacity_owner.observe),
)
async_rejected = raises(TypeError, lambda: registry.register(remaining.observe_async))
standalone_rejected = raises(TypeError, lambda: registry.register(lambda: None))

assert (
    retained_by_snapshot
    and collected_after_snapshot
    and automatically_pruned
    and removed_once
    and not removed_twice
    and len(set(capacity_tokens)) == 128
    and capacity_registry.entry_count == 128
    and capacity_rejected
    and async_rejected
    and standalone_rejected
)
```

## Trade-offs and Limitations

Registration, removal, and callback cleanup use expected `O(1)` dictionary
operations. A capacity check or explicit snapshot may also scan at most 128
entries to prune dead references. Tokens increase monotonically for the
registry's lifetime and are never reused, preventing an old token from
removing a later registration.

A returned snapshot contains real bound methods, so it strongly retains their
owners until the snapshot and all methods taken from it are released. Garbage
collection timing remains implementation-dependent; the deterministic example
calls `gc.collect()` on the tested CPython runtimes. The registry does not
invoke callbacks and therefore defines no argument, return-value, ordering
beyond registration order, or exception policy. It rejects standalone and
async functions, and it is not thread-safe. Add synchronization around every
operation if registration, collection callbacks, removal, and snapshotting can
interleave across threads.

## Related Snippets

<!-- catalog:related:start -->
- [Cache One Zero-Argument Method Result per Weakly Referenced Instance](cache-one-zero-argument-method-result-per-weakly-referenced-instance.md)
- [Reuse Live Objects by Exact String Key with WeakValueDictionary](reuse-live-objects-by-exact-string-key-with-weakvaluedictionary.md)
- [Attach Best-Effort Cleanup with weakref.finalize Without Retaining the Owner](attach-best-effort-cleanup-with-weakref-finalize-without-retaining-the-owner.md)
<!-- catalog:related:end -->
