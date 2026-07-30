---
title: "Open a Verified Read-Only SQLite Connection Under a Closed Hardening Profile"
snippet_type: recipe
use_cases:
  - persistence
  - resource-management
  - security
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - execute-a-trusted-sqlite-query-under-progress-callback-and-row-caps.md
  - page-bounded-sqlite-rows-with-a-composite-keyset-cursor.md
  - clone-a-bounded-in-memory-sqlite-fixture-with-connection-backup.md
---

# Open a Verified Read-Only SQLite Connection Under a Closed Hardening Profile

## Idea and Problem

Open one existing SQLite file through a dedicated read-only connection, apply a closed set of connection hardening controls, and verify every control before yielding the connection.

SQLite URI `mode=ro` prevents the connection from opening the main database for
normal writes. `PRAGMA query_only`, `SQLITE_DBCONFIG_DEFENSIVE`, and
`SQLITE_DBCONFIG_TRUSTED_SCHEMA` add independent connection-level restrictions.
This context manager treats every setting and read-back as mandatory: it closes
the connection and raises one typed error instead of silently using a weaker
profile.

## When to Use

Use this boundary when a local component needs to run reviewed read queries
against one known existing database file and must not retain the connection
after the operation. Pass an absolute platform `Path`; strings, relative paths,
path subclasses, missing paths, and non-regular-file targets are rejected.

Keep query review separate. The helper deliberately returns a normal
`sqlite3.Connection`, so callers remain responsible for the statements they
execute and for validating any returned values.

## Implementation

```python
import sqlite3
from collections.abc import Iterator
from contextlib import contextmanager
from pathlib import Path
from tempfile import TemporaryDirectory

_PLATFORM_PATH_TYPE = type(Path())


class SQLiteHardeningError(RuntimeError):
    """The required SQLite connection profile could not be verified."""


def _required_sqlite_dbconfig(name: str) -> int:
    option = getattr(sqlite3, name, None)
    if type(option) is not int:
        raise SQLiteHardeningError(f"required SQLite option {name} is unavailable")
    return option


def _apply_verified_sqlite_profile(connection: sqlite3.Connection) -> None:
    defensive = _required_sqlite_dbconfig("SQLITE_DBCONFIG_DEFENSIVE")
    trusted_schema = _required_sqlite_dbconfig(
        "SQLITE_DBCONFIG_TRUSTED_SCHEMA",
    )
    set_config = getattr(connection, "setconfig", None)
    get_config = getattr(connection, "getconfig", None)
    if not callable(set_config) or not callable(get_config):
        raise SQLiteHardeningError(
            "SQLite connection configuration methods are unavailable",
        )

    try:
        connection.execute("PRAGMA query_only = ON").close()
        cursor = connection.execute("PRAGMA query_only")
        try:
            query_only_row = cursor.fetchone()
            trailing_row = cursor.fetchone()
        finally:
            cursor.close()

        set_config(defensive, True)
        defensive_enabled = get_config(defensive)
        set_config(trusted_schema, False)
        trusted_schema_enabled = get_config(trusted_schema)
    except (sqlite3.Error, TypeError, ValueError) as error:
        raise SQLiteHardeningError(
            "the SQLite hardening profile could not be applied",
        ) from error

    if query_only_row != (1,) or trailing_row is not None:
        raise SQLiteHardeningError("PRAGMA query_only read-back failed")
    if defensive_enabled is not True:
        raise SQLiteHardeningError("defensive-mode read-back failed")
    if trusted_schema_enabled is not False:
        raise SQLiteHardeningError("trusted-schema read-back failed")


@contextmanager
def open_verified_read_only_sqlite(
    database_path: Path,
) -> Iterator[sqlite3.Connection]:
    """Yield one owned connection only after verifying the closed profile."""
    if type(database_path) is not _PLATFORM_PATH_TYPE:
        raise TypeError("database_path must be an exact platform Path")
    if not database_path.is_absolute():
        raise ValueError("database_path must be absolute")
    if not database_path.is_file():
        raise FileNotFoundError("database_path must name an existing regular file")

    database_uri = database_path.as_uri() + "?mode=ro"
    connection = sqlite3.connect(
        database_uri,
        uri=True,
        autocommit=True,
        check_same_thread=True,
    )
    try:
        _apply_verified_sqlite_profile(connection)
        yield connection
    finally:
        connection.close()
```

## Example

```python


with TemporaryDirectory() as temporary_directory:
    database_path = Path(temporary_directory) / "catalog.sqlite3"
    setup_connection = sqlite3.connect(database_path, autocommit=True)
    try:
        setup_connection.execute(
            "CREATE TABLE item(id INTEGER PRIMARY KEY, value TEXT NOT NULL)",
        )
        setup_connection.executemany(
            "INSERT INTO item(value) VALUES (?)",
            (("alpha",), ("beta",)),
        )
    finally:
        setup_connection.close()

    with open_verified_read_only_sqlite(database_path) as connection:
        rows = connection.execute(
            "SELECT id, value FROM item ORDER BY id",
        ).fetchall()
        query_only = connection.execute("PRAGMA query_only").fetchone()
        defensive = connection.getconfig(sqlite3.SQLITE_DBCONFIG_DEFENSIVE)
        trusted_schema = connection.getconfig(
            sqlite3.SQLITE_DBCONFIG_TRUSTED_SCHEMA,
        )

    try:
        connection.execute("SELECT 1")
    except sqlite3.ProgrammingError:
        connection_was_closed = True
    else:
        connection_was_closed = False

assert (
    rows,
    query_only,
    defensive,
    trusted_schema,
    connection_was_closed,
) == (
    [(1, "alpha"), (2, "beta")],
    (1,),
    True,
    False,
    True,
)
```

## Trade-offs and Limitations

The exact platform-path check intentionally rejects `PurePath` values and
custom `Path` subclasses. `is_file()` and the later SQLite open are separate
filesystem operations. A rename, replacement, or symlink change between them
can change what is opened, and an external writer can change the database while
the connection is active. This helper neither pins a filesystem identity nor
creates an immutable database snapshot.

The profile depends on Python exposing both database-configuration constants
and the linked SQLite library supporting them. Unsupported builds fail closed
with `SQLiteHardeningError`. The three controls reduce writable and
schema-trust behavior on this connection; they do not validate database
contents, authorize arbitrary SQL, cap query work or result size, or replace
application-level query review. `immutable=1` is intentionally absent because
the helper cannot prove that the file will remain unchanged.

The yielded object is a normal mutable connection, so trusted caller code can
turn these controls off again. The helper verifies the entry profile; it does
not make later reconfiguration impossible.

The context owns a fresh connection and always closes it, including when
profile setup, read-back, the caller's block, or normal block exit raises. Do
not retain the yielded object or share it with concurrent work.

## Related Snippets

<!-- catalog:related:start -->
- [Execute a Trusted SQLite Query Under Progress-Callback and Row Caps](execute-a-trusted-sqlite-query-under-progress-callback-and-row-caps.md)
- [Page Bounded SQLite Rows with a Composite Keyset Cursor](page-bounded-sqlite-rows-with-a-composite-keyset-cursor.md)
- [Clone a Bounded In-Memory SQLite Fixture with Connection.backup](clone-a-bounded-in-memory-sqlite-fixture-with-connection-backup.md)
<!-- catalog:related:end -->
