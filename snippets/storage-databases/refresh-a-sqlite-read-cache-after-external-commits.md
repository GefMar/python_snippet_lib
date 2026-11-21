---
title: "Refresh a SQLite Read Cache After External Commits"
snippet_type: pattern
use_cases:
  - caching
  - persistence
tested_python:
  - "3.14"
dependencies: []
related:
  - replace-a-complete-bounded-sqlite-cache-snapshot-with-a-generation.md
  - compare-and-swap-a-versioned-sqlite-setting-with-one-conditional-update.md
---

# Refresh a SQLite Read Cache After External Commits

## Idea and Problem

Reuse an in-memory snapshot of a small SQLite table until another connection commits a change to the database.

SQLite exposes `PRAGMA main.data_version` as a connection-local change signal.
Compare it with the value captured before the previous successful load. If a
commit lands between sampling the signal and reading the table, the next call
can refresh again. Sampling after the query could associate an older result
with a newer signal and let the stale result persist.

## When to Use

Use this for a small lookup table in `main` when a display or periodic job can
tolerate snapshot reads. Give the cache a dedicated connection opened with
`mode=ro` and `autocommit=True`, using the default result conversion. Keep
each cache with that exact connection and serialize its use on the owning
thread. The example expects a schema that enforces non-null text codes and
integer rates. Open a new cache when the connection is replaced.

## Implementation

```python
import sqlite3
from contextlib import closing

type RateRows = tuple[tuple[str, int], ...]


class RateCache:
    def __init__(self, reader: sqlite3.Connection) -> None:
        self._reader = reader
        self._snapshot: tuple[int, RateRows] | None = None

    def get(self) -> RateRows:
        if self._reader.autocommit is not True or self._reader.in_transaction:
            raise ValueError("the reader must be in autocommit outside a transaction")

        with closing(self._reader.cursor()) as cursor:
            (version,) = cursor.execute("PRAGMA main.data_version").fetchone()
            snapshot = self._snapshot
            if snapshot is not None and snapshot[0] == version:
                return snapshot[1]

            rows = tuple(cursor.execute("SELECT code, cents FROM main.rates ORDER BY code"))

        self._snapshot = (version, rows)
        return rows
```

## Example

```python
from pathlib import Path
from tempfile import TemporaryDirectory

with TemporaryDirectory() as directory:
    path = Path(directory).resolve() / "rates.sqlite3"
    with closing(sqlite3.connect(path, autocommit=False)) as writer:
        with writer, closing(writer.cursor()) as cursor:
            cursor.execute(
                "CREATE TABLE rates (code TEXT PRIMARY KEY, cents INTEGER NOT NULL) STRICT"
            )
            cursor.execute("INSERT INTO rates VALUES (?, ?)", ("day", 1200))

        uri = f"{path.as_uri()}?mode=ro"
        with closing(sqlite3.connect(uri, uri=True, autocommit=True)) as reader:
            cache = RateCache(reader)
            original = cache.get()
            with closing(writer.cursor()) as cursor:
                cursor.execute("UPDATE rates SET cents = ? WHERE code = ?", (1500, "day"))
                writer.rollback()
                reused_after_rollback = cache.get() is original

                cursor.execute("UPDATE rates SET cents = ? WHERE code = ?", (1800, "day"))
                writer.commit()
                refreshed = cache.get()
                reused_after_refresh = cache.get() is refreshed

assert (original, reused_after_rollback, refreshed, reused_after_refresh) == (
    (("day", 1200),),
    True,
    (("day", 1800),),
    True,
)
```

## Trade-offs and Limitations

Every access still queries SQLite for the change signal. Writes anywhere in
`main` may cause a full reload. Reloading costs time and memory proportional
to the result size, so keep the lookup small. A failed read propagates and
leaves the last complete snapshot installed for future refresh attempts.

The value cannot be compared across connections and does not track writes
made on the same connection. A successful call can return a snapshot taken
before a concurrent commit. Query inside the transaction that owns a
consistency-sensitive decision when it needs a particular database snapshot.

## Related Snippets

<!-- catalog:related:start -->
- [Replace a Complete Bounded SQLite Cache Snapshot with a Generation](replace-a-complete-bounded-sqlite-cache-snapshot-with-a-generation.md)
- [Compare and Swap a Versioned SQLite Setting with One Conditional Update](compare-and-swap-a-versioned-sqlite-setting-with-one-conditional-update.md)
<!-- catalog:related:end -->
