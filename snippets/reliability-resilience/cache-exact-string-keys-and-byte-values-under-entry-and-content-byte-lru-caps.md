---
title: "Cache Exact String Keys and Byte Values Under Entry and Content-Byte LRU Caps"
snippet_type: pattern
use_cases:
  - caching
  - resource-management
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - cache-values-with-a-monotonic-ttl-and-early-jitter.md
  - replay-a-frozen-read-outcome-for-an-exact-access-budget.md
  - ../python-language/bypass-an-lru-cache-with-a-per-call-predicate.md
---

# Cache Exact String Keys and Byte Values Under Entry and Content-Byte LRU Caps

## Idea and Problem

Keep immutable byte values in least-recently-used order while enforcing both an entry count and an explicit retained-content byte budget.

An `OrderedDict` stores entries from least to most recently used. A hit or
replacement moves its key to the newest end; insertion evicts from the oldest
end until both caps hold. Content accounting uses each key's UTF-8 length plus
its value length, rather than claiming to measure Python object memory.

## When to Use

Use this cache for one process-local, blocking owner when keys are bounded text,
values are already immutable bytes, and eviction by recent access is the desired
policy. Returning `None` on a miss keeps a cached empty byte string distinct
from absence.

Choose another component when entries need expiration, concurrent request
coalescing, cross-process access, durable recovery, or a bound on actual Python
object memory. Serialize all access externally if several threads can reach the
same instance.

## Implementation

```python
from collections import OrderedDict

_HARD_MAX_ENTRIES = 10_000
_HARD_MAX_CONTENT_BYTES = 64 * 1024 * 1024
_MAX_KEY_CHARACTERS = 1_024


def _positive_bounded_limit(value: object, *, name: str, maximum: int) -> int:
    if type(value) is not int:
        raise TypeError(f"{name} must be an exact integer")
    if not 1 <= value <= maximum:
        raise ValueError(f"{name} is outside the supported range")
    return value


class BoundedContentLru:
    """Own exact string-to-bytes entries under count and content-byte caps."""

    __slots__ = (
        "_content_bytes",
        "_entries",
        "_max_content_bytes",
        "_max_entries",
    )

    def __init__(self, *, max_entries: int, max_content_bytes: int) -> None:
        self._max_entries = _positive_bounded_limit(
            max_entries,
            name="max_entries",
            maximum=_HARD_MAX_ENTRIES,
        )
        self._max_content_bytes = _positive_bounded_limit(
            max_content_bytes,
            name="max_content_bytes",
            maximum=_HARD_MAX_CONTENT_BYTES,
        )
        self._entries: OrderedDict[str, bytes] = OrderedDict()
        self._content_bytes = 0

    def _key_size(self, key: object) -> int:
        if type(key) is not str:
            raise TypeError("key must be an exact string")
        if len(key) > _MAX_KEY_CHARACTERS:
            raise ValueError("key has too many characters")
        try:
            encoded_key = key.encode("utf-8")
        except UnicodeEncodeError:
            raise ValueError("key must be valid UTF-8 text") from None
        if len(encoded_key) > self._max_content_bytes:
            raise ValueError("the encoded key exceeds max_content_bytes")
        return len(encoded_key)

    def get(self, key: str) -> bytes | None:
        """Return and promote a hit, or return None without reordering a miss."""
        self._key_size(key)
        value = self._entries.get(key)
        if value is None:
            return None
        self._entries.move_to_end(key)
        return value

    def put(self, key: str, value: bytes) -> tuple[str, ...]:
        """Insert or replace one entry and return evicted keys in LRU order."""
        key_size = self._key_size(key)
        if type(value) is not bytes:
            raise TypeError("value must be exact bytes")
        entry_size = key_size + len(value)
        if entry_size > self._max_content_bytes:
            raise ValueError("the entry exceeds max_content_bytes")

        previous = self._entries.pop(key, None)
        if previous is not None:
            self._content_bytes -= key_size + len(previous)

        self._entries[key] = value
        self._content_bytes += entry_size

        evicted: list[str] = []
        while (
            len(self._entries) > self._max_entries or self._content_bytes > self._max_content_bytes
        ):
            evicted_key, evicted_value = self._entries.popitem(last=False)
            self._content_bytes -= len(evicted_key.encode("utf-8")) + len(evicted_value)
            evicted.append(evicted_key)
        return tuple(evicted)

    def snapshot(self) -> tuple[tuple[str, bytes], ...]:
        """Return immutable entries in least-to-most-recently-used order."""
        return tuple(self._entries.items())
```

## Example

```python
cache = BoundedContentLru(max_entries=3, max_content_bytes=12)
cache.put("a", b"12")
cache.put("é", b"")
cache.put("b", b"345")
initial = cache.snapshot()

hit = cache.get("a")
miss = cache.get("missing")
evicted = cache.put("é", b"6789")
after_replacement = cache.snapshot()

try:
    cache.put("a", b"x" * 12)
except ValueError:
    oversized_rejected = cache.snapshot() == after_replacement
else:
    oversized_rejected = False

assert (initial, hit, miss, evicted, after_replacement, oversized_rejected) == (
    (("a", b"12"), ("é", b""), ("b", b"345")),
    b"12",
    None,
    ("b",),
    (("a", b"12"), ("é", b"6789")),
    True,
)
```

## Trade-offs and Limitations

Key validation takes `O(c)` time and temporary space for `c` UTF-8 bytes.
Ordered-dictionary lookup, promotion, insertion, and removal are average
`O(1)` operations; one `put` additionally takes `O(e)` time for `e` evictions.
A snapshot takes `O(n)` time and tuple space for `n` retained entries.

The content counter covers encoded key bytes plus value bytes only. It excludes
Python string, bytes, dictionary and tuple overhead, allocator behavior, and
temporary encoding copies, so it is not a process-memory limit. The cache has
no TTL, locks, single-flight loading, admission policy, persistence, encryption,
or cross-process coherence. Snapshots expose the retained values and should not
be treated as redacted diagnostics.

## Related Snippets

<!-- catalog:related:start -->
- [Cache Values with a Monotonic TTL and Early Jitter](cache-values-with-a-monotonic-ttl-and-early-jitter.md)
- [Replay a Frozen Read Outcome for an Exact Access Budget](replay-a-frozen-read-outcome-for-an-exact-access-budget.md)
- [Bypass an LRU Cache with a Per-Call Predicate](../python-language/bypass-an-lru-cache-with-a-per-call-predicate.md)
<!-- catalog:related:end -->
