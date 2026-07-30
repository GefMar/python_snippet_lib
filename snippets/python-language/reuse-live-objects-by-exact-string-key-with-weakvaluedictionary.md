---
title: "Reuse Live Objects by Exact String Key with WeakValueDictionary"
snippet_type: pattern
use_cases:
  - caching
  - resource-management
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - cache-one-zero-argument-method-result-per-weakly-referenced-instance.md
  - attach-best-effort-cleanup-with-weakref-finalize-without-retaining-the-owner.md
  - ../reliability-resilience/cache-values-with-a-monotonic-ttl-and-early-jitter.md
---

# Reuse Live Objects by Exact String Key with WeakValueDictionary

## Idea and Problem

Reuse one live object for each exact bounded string key without making the registry keep that object alive.

A `WeakValueDictionary` holds only weak references to produced objects. A
strong reference elsewhere therefore keeps a key reusable, while collection
automatically frees its registry slot. A separate creating-key set detects a
factory that recursively requests its own unfinished key.

Capacity is checked both before a factory call and after it returns. The second
check matters because a trusted factory may synchronously intern a different
key and fill the last slot while the outer value is being constructed.

## When to Use

Use this pattern for a bounded, process-local identity map when equal exact
string keys should share one currently live object, but unused objects should
not remain alive only because they were registered. Typical values include
weak-referenceable parsed descriptors or mutable runtime models whose identity
is useful while a caller owns them.

Use a normal bounded dictionary when entries must survive without external
owners. Use a lock-aware cache for concurrent access, a TTL cache for timed
expiry, or durable storage for cross-process reuse. The factory must be trusted,
synchronous, and safe to call once on a cache miss.

## Implementation

```python
import weakref
from collections.abc import Callable

_MAX_INTERN_KEY_BYTES = 256
_MAX_LIVE_INTERN_ENTRIES = 4_096
_MAX_ACTIVE_INTERN_CREATIONS = 64


class InternerCapacityError(RuntimeError):
    pass


class InternerReentryError(RuntimeError):
    pass


class WeakStringInterner[ValueT]:
    """Reuse weak-referenceable live values under bounded exact string keys."""

    def __init__(self, max_live_entries: int) -> None:
        if type(max_live_entries) is not int:
            raise TypeError("max_live_entries must be an exact integer")
        if not 1 <= max_live_entries <= _MAX_LIVE_INTERN_ENTRIES:
            raise ValueError("max_live_entries is outside 1..4096")

        self._max_live_entries = max_live_entries
        self._values: weakref.WeakValueDictionary[str, ValueT] = (
            weakref.WeakValueDictionary()
        )
        self._creating: set[str] = set()

    @staticmethod
    def _validated_key(key: object) -> str:
        if type(key) is not str:
            raise TypeError("key must be an exact string")
        if not 1 <= len(key) <= _MAX_INTERN_KEY_BYTES:
            raise ValueError("key length cannot fit the UTF-8 byte limit")
        try:
            encoded = key.encode("utf-8")
        except UnicodeEncodeError:
            raise ValueError("key must be UTF-8 encodable") from None
        if not 1 <= len(encoded) <= _MAX_INTERN_KEY_BYTES:
            raise ValueError("key UTF-8 length is outside 1..256 bytes")
        return key

    def intern(
        self,
        key: str,
        factory: Callable[[str], ValueT],
    ) -> ValueT:
        """Return the live value for key, constructing one bounded miss."""
        checked_key = self._validated_key(key)
        if not callable(factory):
            raise TypeError("factory must be callable")

        try:
            return self._values[checked_key]
        except KeyError:
            pass

        if checked_key in self._creating:
            raise InternerReentryError("factory re-entered its unfinished key")
        if len(self._values) >= self._max_live_entries:
            raise InternerCapacityError("live interner capacity is exhausted")
        if len(self._creating) >= _MAX_ACTIVE_INTERN_CREATIONS:
            raise InternerCapacityError("active factory nesting exceeds 64 keys")

        self._creating.add(checked_key)
        try:
            produced = factory(checked_key)
        finally:
            self._creating.remove(checked_key)

        try:
            weakref.ref(produced)
        except TypeError:
            raise TypeError("factory result must support weak references") from None

        if len(self._values) >= self._max_live_entries:
            raise InternerCapacityError(
                "a nested factory call exhausted live interner capacity"
            )
        self._values[checked_key] = produced
        return produced

    @property
    def live_entry_count(self) -> int:
        return len(self._values)
```

## Example

```python
class MutableRecord:
    __slots__ = ("__weakref__", "key", "revision")

    def __init__(self, key: str) -> None:
        self.key = key
        self.revision = 0


def raises(error_type: type[Exception], operation) -> bool:
    try:
        operation()
    except error_type:
        return True
    return False


def collect_garbage() -> None:
    import gc

    gc.collect()


created_keys: list[str] = []


def create_record(key: str) -> MutableRecord:
    created_keys.append(key)
    return MutableRecord(key)


interner = WeakStringInterner[MutableRecord](2)
first = interner.intern("alpha", create_record)
same_first = interner.intern("alpha", create_record)
first.revision = 7
second = interner.intern("beta", create_record)
identity_reused = first is same_first and same_first.revision == 7
capacity_rejected = raises(
    InternerCapacityError,
    lambda: interner.intern("gamma", create_record),
)

first_reference = weakref.ref(first)
del first, same_first
collect_garbage()
replacement = interner.intern("alpha", create_record)

retry_interner = WeakStringInterner[MutableRecord](1)
attempts = 0


def fail_once(key: str) -> MutableRecord:
    global attempts
    attempts += 1
    if attempts == 1:
        raise LookupError("temporary construction failure")
    return MutableRecord(key)


failure_propagated = raises(
    LookupError,
    lambda: retry_interner.intern("retry", fail_once),
)
retried = retry_interner.intern("retry", fail_once)

reentry_interner = WeakStringInterner[MutableRecord](1)


def reenter_same_key(key: str) -> MutableRecord:
    return reentry_interner.intern(key, reenter_same_key)


reentry_rejected = raises(
    InternerReentryError,
    lambda: reentry_interner.intern("recursive", reenter_same_key),
)
recovered_after_reentry = reentry_interner.intern("recursive", MutableRecord)

nested_interner = WeakStringInterner[MutableRecord](1)
nested_survivors: list[MutableRecord] = []


def create_with_nested_key(key: str) -> MutableRecord:
    nested_survivors.append(
        nested_interner.intern("inner", MutableRecord)
    )
    return MutableRecord(key)


nested_capacity_rechecked = raises(
    InternerCapacityError,
    lambda: nested_interner.intern("outer", create_with_nested_key),
)

depth_interner = WeakStringInterner[MutableRecord](1)


def create_at_next_depth(key: str) -> MutableRecord:
    return depth_interner.intern(str(int(key) + 1), create_at_next_depth)


factory_depth_rejected = raises(
    InternerCapacityError,
    lambda: depth_interner.intern("0", create_at_next_depth),
)

utf8_interner = WeakStringInterner[MutableRecord](1)
utf8_boundary = utf8_interner.intern("é" * 128, MutableRecord)

assert (
    identity_reused
    and capacity_rejected
    and first_reference() is None
    and replacement is not second
    and created_keys == ["alpha", "beta", "alpha"]
    and interner.live_entry_count == 2
    and failure_propagated
    and attempts == 2
    and retried.key == "retry"
    and reentry_rejected
    and recovered_after_reentry.key == "recursive"
    and nested_capacity_rechecked
    and nested_interner.live_entry_count == 1
    and nested_survivors[0].key == "inner"
    and factory_depth_rejected
    and depth_interner.live_entry_count == 0
    and utf8_boundary.key == "é" * 128
    and raises(ValueError, lambda: utf8_interner.intern("é" * 129, MutableRecord))
    and raises(ValueError, lambda: utf8_interner.intern("", MutableRecord))
    and raises(ValueError, lambda: utf8_interner.intern("\ud800", MutableRecord))
    and raises(TypeError, lambda: WeakStringInterner(1).intern("value", tuple))
    and raises(TypeError, lambda: WeakStringInterner(True))
    and raises(ValueError, lambda: WeakStringInterner(0))
)
```

## Trade-offs and Limitations

Validating a key costs `O(B)` for at most 256 UTF-8 bytes. Registry lookup,
insertion, and removal are expected `O(1)`, excluding factory work and garbage
collection. At most `K` live key entries and 64 active construction marks retain
`O(K)` registry memory.

Values must support weak references, but they need not be immutable. Mutating a
shared live value is visible to every caller that received it. Collection
timing is not promised; the Example explicitly calls `gc.collect()` on the
tested CPython runtime instead of treating prompt collection as a portable
lifecycle guarantee.

The implementation is not thread-safe and provides no TTL, persistence,
cross-process sharing, explicit eviction order, or factory-result ownership
transfer. It assumes exact built-in string keys and trusted synchronous
factories; hostile factories, recursive different-key cycles, and concurrent
mutation are outside the contract. Successful nested different-key insertions
are independent and are not rolled back if an outer factory later fails or
loses the post-factory capacity check. An acyclic different-key factory chain is
rejected when it attempts to exceed 64 simultaneously unfinished keys.

## Related Snippets

<!-- catalog:related:start -->
- [Cache One Zero-Argument Method Result per Weakly Referenced Instance](cache-one-zero-argument-method-result-per-weakly-referenced-instance.md)
- [Attach Best-Effort Cleanup with weakref.finalize Without Retaining the Owner](attach-best-effort-cleanup-with-weakref-finalize-without-retaining-the-owner.md)
- [Cache Values with a Monotonic TTL and Early Jitter](../reliability-resilience/cache-values-with-a-monotonic-ttl-and-early-jitter.md)
<!-- catalog:related:end -->
