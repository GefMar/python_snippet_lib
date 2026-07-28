---
title: "Replace a Complete Bounded SQLite Cache Snapshot with a Generation"
snippet_type: pattern
use_cases:
  - caching
  - persistence
  - retry-recovery
tested_python:
  - "3.14"
dependencies: []
related:
  - build-and-apply-a-deterministic-mapping-delta.md
  - validate-a-bounded-stage-verify-pointer-switch-log.md
  - decide-whether-to-restore-a-versioned-snapshot.md
---

# Replace a Complete Bounded SQLite Cache Snapshot with a Generation

## Idea and Problem

Atomically replace a local SQLite lookup cache only after its complete bounded snapshot has been fetched and validated.

Deleting by wall-clock age can retain records that disappeared recently, and
deleting before ingestion can empty the cache when a later page fails. Give
every successful refresh a transactionally allocated generation, mark each
upserted row with it, and delete all older generations in the same transaction.
Readers then see either the prior complete snapshot or the new one.

## When to Use

Use this pattern when an upstream source can provide a complete finite snapshot
and a local application needs fast keyed reads. Fetch and paginate outside the
function, prove that every page succeeded, then pass one immutable collection.
An explicitly complete empty collection is allowed to clear the cache.

Choose a delta feed or staged pointer switch when a full snapshot is too large
for one write transaction. Do not use this approach when the source can return
partial success without a reliable completeness signal.

## Implementation

```python
import re
import sqlite3
from dataclasses import dataclass

_KEY = re.compile(r"[a-z][a-z0-9._:-]{0,63}", re.ASCII).fullmatch
_MAX_ROWS = 4_096
_MAX_EXISTING_ROWS = 8_192
_MAX_PAYLOAD_BYTES = 16 * 1_024
_MAX_TOTAL_PAYLOAD_BYTES = 8 * 1_024 * 1_024
_SIGNED_64_MAX = (1 << 63) - 1


@dataclass(frozen=True, slots=True)
class CacheSnapshotEntry:
    key: str
    payload: bytes


@dataclass(frozen=True, slots=True)
class CacheSnapshotReplacement:
    generation: int
    row_count: int
    deleted_rows: int


def initialize_generation_cache(connection: sqlite3.Connection) -> None:
    if not isinstance(connection, sqlite3.Connection):
        raise TypeError("connection must be a sqlite3.Connection")
    if connection.in_transaction:
        raise RuntimeError("connection must be idle")
    connection.execute(
        """
        CREATE TABLE IF NOT EXISTS cache_generation (
            singleton INTEGER PRIMARY KEY CHECK (singleton = 1),
            current_generation INTEGER NOT NULL CHECK (current_generation >= 0)
        ) STRICT
        """
    )
    connection.execute(
        """
        INSERT OR IGNORE INTO cache_generation(singleton, current_generation)
        VALUES (1, 0)
        """
    )
    connection.execute(
        """
        CREATE TABLE IF NOT EXISTS cache_entries (
            entry_key TEXT PRIMARY KEY,
            payload BLOB NOT NULL,
            generation INTEGER NOT NULL CHECK (generation > 0)
        ) STRICT
        """
    )
    connection.execute(
        "CREATE INDEX IF NOT EXISTS cache_entries_generation ON cache_entries(generation)"
    )


def _validated_snapshot(
    value: object,
) -> tuple[CacheSnapshotEntry, ...]:
    if type(value) is not tuple:
        raise TypeError("entries must be an exact tuple")
    if len(value) > _MAX_ROWS:
        raise ValueError("snapshot row count exceeds the supported limit")

    validated: list[CacheSnapshotEntry] = []
    keys: set[str] = set()
    total_payload_bytes = 0
    for entry in value:
        if type(entry) is not CacheSnapshotEntry:
            raise TypeError("entries must contain exact CacheSnapshotEntry values")
        if type(entry.key) is not str or _KEY(entry.key) is None:
            raise ValueError("entry keys must be conservative ASCII tokens")
        if entry.key in keys:
            raise ValueError("entry keys must be unique")
        if type(entry.payload) is not bytes:
            raise TypeError("entry payloads must be exact bytes")
        if len(entry.payload) > _MAX_PAYLOAD_BYTES:
            raise ValueError("entry payload exceeds the supported byte limit")
        total_payload_bytes += len(entry.payload)
        if total_payload_bytes > _MAX_TOTAL_PAYLOAD_BYTES:
            raise ValueError("aggregate payload exceeds the supported byte limit")
        keys.add(entry.key)
        validated.append(CacheSnapshotEntry(entry.key, entry.payload))
    return tuple(sorted(validated, key=lambda entry: entry.key))


def replace_complete_cache_snapshot(
    connection: sqlite3.Connection,
    entries: tuple[CacheSnapshotEntry, ...],
) -> CacheSnapshotReplacement:
    if not isinstance(connection, sqlite3.Connection):
        raise TypeError("connection must be a sqlite3.Connection")
    if connection.in_transaction:
        raise RuntimeError("connection must be idle")
    snapshot = _validated_snapshot(entries)

    connection.execute("BEGIN IMMEDIATE")
    try:
        generation_row = connection.execute(
            "SELECT current_generation FROM cache_generation WHERE singleton = 1"
        ).fetchone()
        if generation_row is None:
            raise RuntimeError("cache generation record is missing")
        current_generation = generation_row[0]
        if type(current_generation) is not int or not 0 <= current_generation < _SIGNED_64_MAX:
            raise RuntimeError("stored cache generation is invalid or exhausted")

        existing_rows, maximum_row_generation = connection.execute(
            "SELECT COUNT(*), COALESCE(MAX(generation), 0) FROM cache_entries"
        ).fetchone()
        if existing_rows > _MAX_EXISTING_ROWS:
            raise RuntimeError("existing cache exceeds the supported row limit")
        if (
            type(maximum_row_generation) is not int
            or not 0 <= maximum_row_generation <= current_generation
        ):
            raise RuntimeError("stored cache row generation is inconsistent")
        generation = current_generation + 1

        connection.executemany(
            """
            INSERT INTO cache_entries(entry_key, payload, generation)
            VALUES (?, ?, ?)
            ON CONFLICT(entry_key) DO UPDATE SET
                payload = excluded.payload,
                generation = excluded.generation
            """,
            ((entry.key, entry.payload, generation) for entry in snapshot),
        )
        deleted_rows = connection.execute(
            "DELETE FROM cache_entries WHERE generation <> ?",
            (generation,),
        ).rowcount
        connection.execute(
            """
            UPDATE cache_generation
            SET current_generation = ?
            WHERE singleton = 1
            """,
            (generation,),
        )
        connection.execute("COMMIT")
        return CacheSnapshotReplacement(
            generation=generation,
            row_count=len(snapshot),
            deleted_rows=deleted_rows,
        )
    except BaseException:
        if connection.in_transaction:
            connection.execute("ROLLBACK")
        raise
```

## Example

```python
connection = sqlite3.connect(":memory:", isolation_level=None)
initialize_generation_cache(connection)

first = replace_complete_cache_snapshot(
    connection,
    (
        CacheSnapshotEntry("item-a", b"first"),
        CacheSnapshotEntry("item-b", b"second"),
    ),
)
second = replace_complete_cache_snapshot(
    connection,
    (
        CacheSnapshotEntry("item-b", b"updated"),
        CacheSnapshotEntry("item-c", b"third"),
    ),
)
visible_after_second = connection.execute(
    "SELECT entry_key, payload, generation FROM cache_entries ORDER BY entry_key"
).fetchall()

connection.execute(
    """
    CREATE TRIGGER reject_one_cache_entry
    BEFORE INSERT ON cache_entries
    WHEN NEW.entry_key = 'item-d'
    BEGIN
        SELECT RAISE(ABORT, 'synthetic snapshot failure');
    END
    """
)
try:
    replace_complete_cache_snapshot(
        connection,
        (CacheSnapshotEntry("item-d", b"rejected"),),
    )
except sqlite3.IntegrityError:
    rollback_preserved = connection.execute(
        "SELECT entry_key, payload, generation FROM cache_entries ORDER BY entry_key"
    ).fetchall() == visible_after_second
else:
    rollback_preserved = False

cleared = replace_complete_cache_snapshot(connection, ())
remaining_rows = connection.execute("SELECT COUNT(*) FROM cache_entries").fetchone()[0]
connection.close()

assert (
    first,
    second,
    visible_after_second,
    rollback_preserved,
    cleared,
    remaining_rows,
) == (
    CacheSnapshotReplacement(1, 2, 0),
    CacheSnapshotReplacement(2, 2, 1),
    [("item-b", b"updated", 2), ("item-c", b"third", 2)],
    True,
    CacheSnapshotReplacement(3, 0, 2),
    0,
)
```

## Trade-offs and Limitations

The complete snapshot is copied, validated, and sorted before SQLite work, so
memory use is linear in at most 4,096 rows and 8 MiB of payload. The write then
touches every current row and holds a write transaction through all upserts and
cleanup. Very large catalogs need staging tables, chunked verification, or a
versioned pointer instead.

A generation proves membership in one committed local snapshot, not upstream
freshness or correctness. The function cannot tell whether the caller omitted
a failed page, so its most important precondition is an externally established
complete fetch. It also assumes its schema and connection are owned; triggers,
foreign keys, concurrent writers, and the configured SQLite busy timeout can
still make replacement fail, in which case the prior generation remains.

## Related Snippets

<!-- catalog:related:start -->
- [Build and Apply a Deterministic Mapping Delta](build-and-apply-a-deterministic-mapping-delta.md)
- [Validate a Bounded Stage-Verify-Pointer-Switch Log](validate-a-bounded-stage-verify-pointer-switch-log.md)
- [Decide Whether to Restore a Versioned Snapshot](decide-whether-to-restore-a-versioned-snapshot.md)
<!-- catalog:related:end -->
