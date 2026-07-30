---
title: "Store Bounded Values by Weak Object Identity Without Equality or Hashing"
snippet_type: pattern
use_cases:
  - caching
  - lifecycle-management
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - cache-one-zero-argument-method-result-per-weakly-referenced-instance.md
  - reuse-live-objects-by-exact-string-key-with-weakvaluedictionary.md
  - attach-best-effort-cleanup-with-weakref-finalize-without-retaining-the-owner.md
---

# Store Bounded Values by Weak Object Identity Without Equality or Hashing

## Idea and Problem

Associate a bounded number of strong values with live weak-referenceable objects by exact identity, even when the objects are unhashable or define surprising equality.

The mapping indexes entries by `id(key)` and confirms every match with `is`.
It never compares or hashes either a key or a weak reference. A guarded weak
callback removes an entry only when both its integer identity and its exact
stored reference still match, so a delayed callback cannot delete a newer
entry after identity reuse.

## When to Use

Use this pattern for a small, single-owner side table whose entries should
usually disappear with their keys, but whose key classes cannot satisfy the
equality and hashing requirements of `weakref.WeakKeyDictionary`. It is useful
for attaching derived state to unhashable model objects without modifying
those objects.

Prefer an ordinary dictionary when value equality is the intended key
semantics, or store state on the object when its class is under your control.
Use explicit lifecycle management when prompt removal matters more than
garbage-collection fallback.

## Implementation

```python
import gc
import weakref
from dataclasses import dataclass


@dataclass(frozen=True, slots=True)
class IdentityLookup[ValueT]:
    found: bool
    value: ValueT | None


@dataclass(frozen=True, slots=True, eq=False)
class IdentityAssociation[ValueT]:
    key: object
    value: ValueT


@dataclass(frozen=True, slots=True, eq=False)
class _IdentityEntry[ValueT]:
    reference: weakref.ReferenceType[object]
    value: ValueT


class WeakIdentityValueMap[ValueT]:
    __slots__ = ("__weakref__", "_entries", "_max_entries")

    def __init__(self, *, max_entries: int) -> None:
        if type(max_entries) is not int:
            raise TypeError("max_entries must be an exact integer")
        if not 1 <= max_entries <= 4_096:
            raise ValueError("max_entries must be between 1 and 4096")
        self._max_entries = max_entries
        self._entries: dict[int, _IdentityEntry[ValueT]] = {}

    @staticmethod
    def _require_weak_reference(key: object) -> None:
        try:
            weakref.ref(key)
        except TypeError:
            raise TypeError("key must support weak references") from None

    def _discard_dead_entries(self) -> None:
        for identity, entry in tuple(self._entries.items()):
            if entry.reference() is not None:
                continue
            current = self._entries.get(identity)
            if current is not None and current.reference is entry.reference:
                del self._entries[identity]

    def set(self, key: object, value: ValueT) -> None:
        self._require_weak_reference(key)
        identity = id(key)
        current = self._entries.get(identity)
        if current is not None:
            current_key = current.reference()
            if current_key is key:
                self._entries[identity] = _IdentityEntry(current.reference, value)
                return
            if current_key is not None:
                raise RuntimeError("distinct live objects unexpectedly share an identity")
            del self._entries[identity]

        self._discard_dead_entries()
        if len(self._entries) >= self._max_entries:
            raise ValueError("weak identity map is at capacity")

        owner_reference = weakref.ref(self)

        def discard(dead_reference: weakref.ReferenceType[object]) -> None:
            owner = owner_reference()
            if owner is None:
                return
            entry = owner._entries.get(identity)
            if entry is not None and entry.reference is dead_reference:
                del owner._entries[identity]

        reference = weakref.ref(key, discard)
        self._entries[identity] = _IdentityEntry(reference, value)

    def lookup(self, key: object) -> IdentityLookup[ValueT]:
        self._require_weak_reference(key)
        entry = self._entries.get(id(key))
        if entry is not None and entry.reference() is key:
            return IdentityLookup(True, entry.value)
        return IdentityLookup(False, None)

    def delete(self, key: object) -> bool:
        self._require_weak_reference(key)
        identity = id(key)
        entry = self._entries.get(identity)
        if entry is None or entry.reference() is not key:
            return False
        del self._entries[identity]
        return True

    def snapshot(self) -> tuple[IdentityAssociation[ValueT], ...]:
        associations: list[IdentityAssociation[ValueT]] = []
        for entry in tuple(self._entries.values()):
            key = entry.reference()
            if key is not None:
                associations.append(IdentityAssociation(key, entry.value))
        return tuple(associations)


```

## Example

```python
class EqualButUnhashable:
    __hash__ = None
    equality_calls = 0

    def __init__(self, name: str) -> None:
        self.name = name

    def __eq__(self, other: object) -> bool:
        type(self).equality_calls += 1
        return isinstance(other, EqualButUnhashable)


values: WeakIdentityValueMap[str | None] = WeakIdentityValueMap(max_entries=2)
first = EqualButUnhashable("first")
second = EqualButUnhashable("second")

values.set(first, None)
values.set(second, "second value")
stored_none = values.lookup(first)
values.set(first, "replacement")

first_result = values.lookup(first)
second_result = values.lookup(second)
ordered = values.snapshot()
assert stored_none.found and stored_none.value is None
assert first_result == IdentityLookup(True, "replacement")
assert second_result == IdentityLookup(True, "second value")
assert ordered[0].key is first and ordered[1].key is second
assert EqualButUnhashable.equality_calls == 0

first_reference = weakref.ref(first)
del ordered
del first
gc.collect()

remaining = values.snapshot()
assert first_reference() is None
assert len(remaining) == 1 and remaining[0].key is second
assert values.delete(second) and not values.lookup(second).found
```

## Trade-offs and Limitations

The map holds values strongly. A value that directly or indirectly retains its
key prevents collection and therefore prevents automatic removal. A returned
snapshot strongly retains every key it contains for as long as that snapshot
is kept. Collection and weak-callback timing are implementation-dependent, so
automatic removal is not a prompt cleanup guarantee.

Replacing the value for the same live object preserves its insertion position,
including when the map is full. A new association instead fails at capacity;
there is no eviction policy. Non-weak-referenceable keys are rejected before
mutation. The explicit `found` flag distinguishes a stored `None` from a miss.

The implementation is deliberately single-owner and not thread-safe. Object
finalizers, value destruction, or other lifetime changes can re-enter Python
while an operation is in progress, and concurrent mutation can invalidate the
capacity or snapshot assumptions. It offers identity association only: no
locking, iteration view, TTL, size accounting for retained values, or durable
lifecycle guarantee.

## Related Snippets

<!-- catalog:related:start -->
- [Cache One Zero-Argument Method Result per Weakly Referenced Instance](cache-one-zero-argument-method-result-per-weakly-referenced-instance.md)
- [Reuse Live Objects by Exact String Key with WeakValueDictionary](reuse-live-objects-by-exact-string-key-with-weakvaluedictionary.md)
- [Attach Best-Effort Cleanup with weakref.finalize Without Retaining the Owner](attach-best-effort-cleanup-with-weakref-finalize-without-retaining-the-owner.md)
<!-- catalog:related:end -->
