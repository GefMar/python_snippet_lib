---
title: "Model Newest-Wins Lookups with Bounded Immutable LSM Segments"
snippet_type: pattern
use_cases:
  - data-transformation
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - build-a-capacity-sized-bloom-filter.md
  - ../storage-databases/build-and-apply-a-deterministic-mapping-delta.md
---

# Model Newest-Wins Lookups with Bounded Immutable LSM Segments

## Idea and Problem

Model newest-wins lookup by flushing a bounded mutable table into immutable sorted segments and resolving each key from the newest state first.

A tombstone is a stored deletion marker rather than the absence of an entry,
so it can hide values in older segments. Full compaction visits segments from
oldest to newest, applies each value or tombstone, and emits at most one sorted
segment containing only the final live values.

## When to Use

Use this pattern to study segment ordering, tombstone visibility, and full
compaction on a small in-memory data set. It is also useful in deterministic
tests of code that consumes newest-wins snapshots. Keys must fit the
conservative ASCII format, and callers must explicitly flush or compact before
the fixed entry and segment limits are reached.

Use a maintained storage engine when data must survive process failure, when
multiple writers share state, or when incremental compaction and production
performance matter.

## Implementation

```python
import re
from bisect import bisect_left
from dataclasses import dataclass

_MAX_KEY_BYTES = 64
_MAX_VALUE_BYTES = 4_096
_MAX_MEMTABLE_ENTRIES = 64
_MAX_STORED_ENTRIES = 512
_MAX_SEGMENTS = 8
_KEY_PATTERN = re.compile(r"[A-Za-z0-9][A-Za-z0-9_.-]*")
_MISSING = object()


@dataclass(frozen=True, slots=True)
class SegmentEntry:
    key: str
    value: bytes | None


@dataclass(frozen=True, slots=True)
class Segment:
    entries: tuple[SegmentEntry, ...]


@dataclass(frozen=True, slots=True)
class LookupResult:
    key: str
    found: bool
    value: bytes | None


@dataclass(frozen=True, slots=True)
class StoreSnapshot:
    memtable: tuple[SegmentEntry, ...]
    segments: tuple[Segment, ...]


def _require_key(key: object) -> str:
    if type(key) is not str:
        raise TypeError("key must be an exact str")
    if (
        not key.isascii()
        or len(key.encode("ascii")) > _MAX_KEY_BYTES
        or _KEY_PATTERN.fullmatch(key) is None
    ):
        raise ValueError("key has an invalid format or exceeds the byte limit")
    return key


class BoundedSegmentStore:
    def __init__(self) -> None:
        self._memtable: dict[str, bytes | None] = {}
        self._segments: list[Segment] = []

    def _physical_entry_count(self) -> int:
        return len(self._memtable) + sum(
            len(segment.entries) for segment in self._segments
        )

    def _write(self, key: str, value: bytes | None) -> None:
        is_new_entry = key not in self._memtable
        if is_new_entry and len(self._memtable) >= _MAX_MEMTABLE_ENTRIES:
            raise ValueError("memtable entry limit reached; flush it first")
        if is_new_entry and self._physical_entry_count() >= _MAX_STORED_ENTRIES:
            raise ValueError("stored entry limit reached; compact segments first")
        self._memtable[key] = value

    def put(self, key: str, value: bytes) -> None:
        validated_key = _require_key(key)
        if type(value) is not bytes:
            raise TypeError("value must be exact bytes")
        if len(value) > _MAX_VALUE_BYTES:
            raise ValueError("value exceeds the byte limit")
        self._write(validated_key, value)

    def delete(self, key: str) -> None:
        self._write(_require_key(key), None)

    def lookup(self, key: str) -> LookupResult:
        validated_key = _require_key(key)
        memtable_value = self._memtable.get(validated_key, _MISSING)
        if memtable_value is not _MISSING:
            return LookupResult(
                key=validated_key,
                found=memtable_value is not None,
                value=memtable_value,
            )

        for segment in reversed(self._segments):
            index = bisect_left(
                segment.entries,
                validated_key,
                key=lambda entry: entry.key,
            )
            if (
                index < len(segment.entries)
                and segment.entries[index].key == validated_key
            ):
                value = segment.entries[index].value
                return LookupResult(
                    key=validated_key,
                    found=value is not None,
                    value=value,
                )

        return LookupResult(key=validated_key, found=False, value=None)

    def flush(self) -> Segment | None:
        if not self._memtable:
            return None
        if len(self._segments) >= _MAX_SEGMENTS:
            raise ValueError("segment limit reached; compact segments first")

        segment = Segment(
            tuple(
                SegmentEntry(key, self._memtable[key])
                for key in sorted(self._memtable)
            )
        )
        self._segments.append(segment)
        self._memtable.clear()
        return segment

    def compact(self) -> Segment | None:
        if not self._segments:
            return None

        live_values: dict[str, bytes] = {}
        for segment in self._segments:
            for entry in segment.entries:
                if entry.value is None:
                    live_values.pop(entry.key, None)
                else:
                    live_values[entry.key] = entry.value

        entries = tuple(
            SegmentEntry(key, live_values[key]) for key in sorted(live_values)
        )
        self._segments.clear()
        if not entries:
            return None

        compacted = Segment(entries)
        self._segments.append(compacted)
        return compacted

    def snapshot(self) -> StoreSnapshot:
        memtable = tuple(
            SegmentEntry(key, self._memtable[key])
            for key in sorted(self._memtable)
        )
        return StoreSnapshot(memtable=memtable, segments=tuple(self._segments))
```

## Example

```python
store = BoundedSegmentStore()
store.put("alpha", b"first")
store.put("beta", b"kept-for-now")
first_segment = store.flush()

store.put("alpha", b"second")
store.delete("beta")
store.put("gamma", b"")
second_segment = store.flush()

before_compaction = store.snapshot()
alpha_before = store.lookup("alpha")
beta_before = store.lookup("beta")
compacted = store.compact()
after_compaction = store.snapshot()

try:
    store.put("bad/key", b"value")
except ValueError:
    invalid_key_rejected = True
else:
    invalid_key_rejected = False

try:
    store.put("delta", b"x" * 4_097)
except ValueError:
    oversized_value_rejected = True
else:
    oversized_value_rejected = False

assert (
    first_segment,
    second_segment,
    alpha_before,
    beta_before,
    compacted,
    before_compaction.segments[1].entries[1],
    after_compaction,
    store.lookup("beta"),
    invalid_key_rejected,
    oversized_value_rejected,
) == (
    Segment(
        (
            SegmentEntry("alpha", b"first"),
            SegmentEntry("beta", b"kept-for-now"),
        )
    ),
    Segment(
        (
            SegmentEntry("alpha", b"second"),
            SegmentEntry("beta", None),
            SegmentEntry("gamma", b""),
        )
    ),
    LookupResult("alpha", True, b"second"),
    LookupResult("beta", False, None),
    Segment(
        (
            SegmentEntry("alpha", b"second"),
            SegmentEntry("gamma", b""),
        )
    ),
    SegmentEntry("beta", None),
    StoreSnapshot(
        memtable=(),
        segments=(
            Segment(
                (
                    SegmentEntry("alpha", b"second"),
                    SegmentEntry("gamma", b""),
                )
            ),
        ),
    ),
    LookupResult("beta", False, None),
    True,
    True,
)
```

## Trade-offs and Limitations

Lookup checks the memtable and then up to eight segments, using binary search
inside each sorted tuple. Flush sorts at most 64 entries. Full compaction scans
at most 512 stored entries and sorts the remaining distinct keys; it compacts
only flushed segments, so flush first when current memtable changes must be
included.

The fixed bounds intentionally make capacity failure part of the example. A
failed write or flush leaves the existing state unchanged, while compaction
can free superseded physical entries and segment slots. Tombstones may expose
an older value if callers discard newer segments, so segments are not
independent snapshots. This educational in-memory model has no files,
write-ahead log, durability, concurrency control, crash recovery, partial or
background compaction, Bloom filters, or production performance guarantees.

## Related Snippets

<!-- catalog:related:start -->
- [Build a Capacity-Sized Bloom Filter](build-a-capacity-sized-bloom-filter.md)
- [Build and Apply a Deterministic Mapping Delta](../storage-databases/build-and-apply-a-deterministic-mapping-delta.md)
<!-- catalog:related:end -->
