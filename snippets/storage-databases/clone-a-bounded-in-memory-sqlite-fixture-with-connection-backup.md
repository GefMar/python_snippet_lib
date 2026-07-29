---
title: "Clone a Bounded In-Memory SQLite Fixture with Connection.backup"
snippet_type: recipe
use_cases:
  - persistence
  - resource-management
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - scope-caller-owned-sqlite-work-with-an-explicit-savepoint.md
  - replace-a-complete-bounded-sqlite-cache-snapshot-with-a-generation.md
  - read-and-write-bounded-sqlite-blob-slices-with-blobopen.md
---

# Clone a Bounded In-Memory SQLite Fixture with Connection.backup

## Idea and Problem

Clone a small, caller-owned in-memory SQLite fixture into a new independent in-memory connection while enforcing page, byte, and observed progress-call budgets.

`Connection.backup()` copies a coherent database through SQLite's backup API.
The `pages` argument controls pages attempted per step; it is not a total-work
limit. This wrapper first inspects the source, then checks the total page count
again in a fixed progress callback and closes the newly owned destination on
every failure.

## When to Use

Use this to fan out a trusted, read-only test template into isolated in-memory
fixtures. The caller must have created the source as a private pure-memory
database, own the connection exclusively, commit all setup work, and keep the
source quiescent for the entire clone.

Use a temporary file, serialized fixture, migration tool, or disposable
database service when provenance, cross-process sharing, large datasets, or
production restoration matters. Never treat copying as schema validation,
malware inspection, redaction, or a trust boundary for an untrusted database.

## Implementation

```python
import sqlite3

_MAX_PAGE_COUNT = 4_096
_MAX_PAGE_BYTES = 16_777_216
_BACKUP_PAGES_PER_STEP = 64
_MAX_PROGRESS_CALLS = 128


class FixtureCloneError(RuntimeError):
    """Raised when a source or backup exceeds the fixture profile."""


def clone_memory_fixture(source: sqlite3.Connection) -> sqlite3.Connection:
    """Return a newly owned in-memory clone of a quiescent fixture."""
    if type(source) is not sqlite3.Connection:
        raise TypeError("source must be an exact sqlite3.Connection")

    try:
        if source.in_transaction:
            raise FixtureCloneError("source must not have an active transaction")
        if source.text_factory is not str:
            raise FixtureCloneError("source must use the default text factory")

        inspection = source.cursor()
        inspection.row_factory = None
        try:
            databases = inspection.execute("PRAGMA database_list").fetchall()
            page_count_row = inspection.execute(
                "PRAGMA main.page_count"
            ).fetchone()
            page_size_row = inspection.execute(
                "PRAGMA main.page_size"
            ).fetchone()
        finally:
            inspection.close()
    except sqlite3.Error:
        raise FixtureCloneError(
            "source metadata could not be inspected"
        ) from None

    if databases != [(0, "main", "")]:
        raise FixtureCloneError(
            "source must expose only an unnamed main database"
        )
    if page_count_row is None or page_size_row is None:
        raise FixtureCloneError("source did not report page metadata")

    page_count = page_count_row[0]
    page_size = page_size_row[0]
    if (
        type(page_count) is not int
        or type(page_size) is not int
        or page_count < 0
        or page_size <= 0
    ):
        raise FixtureCloneError("source reported invalid page metadata")
    if (
        page_count > _MAX_PAGE_COUNT
        or page_size > _MAX_PAGE_BYTES
        or page_count * page_size > _MAX_PAGE_BYTES
    ):
        raise FixtureCloneError("source exceeds the fixture size profile")

    destination = sqlite3.connect(":memory:")
    progress_calls = 0

    def check_progress(
        _status: int,
        _remaining: int,
        total: int,
    ) -> None:
        nonlocal progress_calls
        progress_calls += 1
        if progress_calls > _MAX_PROGRESS_CALLS:
            raise FixtureCloneError(
                "backup exceeded the progress-call budget"
            )
        if (
            type(total) is not int
            or total < 0
            or total > _MAX_PAGE_COUNT
            or total * page_size > _MAX_PAGE_BYTES
        ):
            raise FixtureCloneError(
                "source grew beyond the fixture size profile"
            )

    try:
        source.backup(
            destination,
            pages=_BACKUP_PAGES_PER_STEP,
            progress=check_progress,
            name="main",
            sleep=0.0,
        )
        copied_count = destination.execute(
            "PRAGMA main.page_count"
        ).fetchone()[0]
        copied_size = destination.execute(
            "PRAGMA main.page_size"
        ).fetchone()[0]
        if (
            copied_count > _MAX_PAGE_COUNT
            or copied_count * copied_size > _MAX_PAGE_BYTES
        ):
            raise FixtureCloneError("clone exceeds the fixture size profile")
    except sqlite3.Error:
        destination.close()
        raise FixtureCloneError("fixture backup failed") from None
    except BaseException:
        destination.close()
        raise

    return destination
```

## Example

```python
source = sqlite3.connect(":memory:")
try:
    source.execute(
        "CREATE TABLE item(id INTEGER PRIMARY KEY, value TEXT NOT NULL)"
    )
    source.executemany(
        "INSERT INTO item(value) VALUES (?)",
        (("alpha",), ("beta",)),
    )
    source.commit()

    clone = clone_memory_fixture(source)
    try:
        cloned_rows = clone.execute(
            "SELECT id, value FROM item ORDER BY id"
        ).fetchall()
        clone.execute(
            "UPDATE item SET value = ? WHERE id = ?",
            ("changed", 1),
        )
        clone.commit()
        source_value = source.execute(
            "SELECT value FROM item WHERE id = ?",
            (1,),
        ).fetchone()
    finally:
        clone.close()

    source.execute("INSERT INTO item(value) VALUES (?)", ("pending",))
    try:
        clone_memory_fixture(source)
    except FixtureCloneError:
        active_transaction_rejected = True
    else:
        active_transaction_rejected = False
    source.rollback()

    source.execute("ATTACH DATABASE ':memory:' AS extra")
    try:
        clone_memory_fixture(source)
    except FixtureCloneError:
        attachment_rejected = True
    else:
        attachment_rejected = False
    source.execute("DETACH DATABASE extra")
finally:
    source.close()

assert (
    cloned_rows == [(1, "alpha"), (2, "beta")]
    and source_value == ("alpha",)
    and active_transaction_rejected
    and attachment_rejected
)
```

## Trade-offs and Limitations

The preflight checks use page count multiplied by page size, so free pages are
part of the 16 MiB profile. Copying is `O(P)` in the admitted database pages
and the returned connection owns a separate in-memory database. The caller
must close that connection.

`pages=64` limits work requested per backup step, not total work. Every
progress callback, including a BUSY or LOCKED status, counts toward 128 calls,
and `sleep=0` prevents Python's backup loop from adding a retry sleep. Those
rules do not impose a wall-clock bound on one underlying SQLite call. A source
that can be locked, changed, or used concurrently violates the quiescence
precondition and may still block below the callback boundary.

Runtime inspection cannot prove that an already handed-in connection was
created with `:memory:`. `PRAGMA database_list` reports an empty main filename
for both a pure-memory database and an empty-name temporary database, and the
temporary form may use disk. The empty filename and attachment checks are
necessary guards only; pure-memory provenance remains a caller assertion.

The source must also use the default text factory so inspection has stable
types. Backup copies the trusted schema and data as-is; it does not validate
triggers, virtual tables, extensions, sensitive values, or later queries.

## Related Snippets

<!-- catalog:related:start -->
- [Scope Caller-Owned SQLite Work with an Explicit Savepoint](scope-caller-owned-sqlite-work-with-an-explicit-savepoint.md)
- [Replace a Complete Bounded SQLite Cache Snapshot with a Generation](replace-a-complete-bounded-sqlite-cache-snapshot-with-a-generation.md)
- [Read and Write Bounded SQLite BLOB Slices with blobopen](read-and-write-bounded-sqlite-blob-slices-with-blobopen.md)
<!-- catalog:related:end -->
