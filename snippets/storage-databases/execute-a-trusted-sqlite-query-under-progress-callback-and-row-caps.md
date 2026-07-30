---
title: "Execute a Trusted SQLite Query Under Progress-Callback and Row Caps"
snippet_type: recipe
use_cases:
  - resource-management
  - validation
  - performance-optimization
tested_python:
  - "3.14"
dependencies: []
related:
  - page-bounded-sqlite-rows-with-a-composite-keyset-cursor.md
  - scope-caller-owned-sqlite-work-with-an-explicit-savepoint.md
  - compile-a-bounded-t-string-into-sqlite-qmark-sql-and-parameters.md
---

# Execute a Trusted SQLite Query Under Progress-Callback and Row Caps

## Idea and Problem

Run one trusted SQLite row query while bounding virtual-machine progress callbacks, returned rows, columns, SQL size, parameters, and SQLite value length.

SQLite can invoke a progress handler after a configured number of virtual
machine instructions. Returning a non-zero value interrupts the statement, so
an exact callback budget provides a deterministic work guard for CPU-bound SQL.
Fetching one row beyond the declared row limit distinguishes a complete result
from a silently truncated one.

The progress handler is connection-global state. This recipe therefore requires
a dedicated, non-concurrent caller-owned connection and removes its handler in
`finally`. It deliberately avoids SQLite connection limits because those also
affect parsing of the database's stored schema.

## When to Use

Use this boundary for trusted parameterized `SELECT` statements whose complete
small result must fit explicit resource limits. It is useful in local tools,
bounded previews, and defensive internal query paths where an unexpectedly
expensive query should fail instead of monopolizing SQLite indefinitely.

Use a database-level statement timeout when a real elapsed-time guarantee is
required. Use SQLite's authorizer or a separate read-only connection to enforce
SQL permissions. A progress handler cannot preempt a blocking user-defined
function, bound memory precisely, or undo side effects from a statement.

## Implementation

```python
import math
import sqlite3

_MAX_SQL_BYTES = 65_536
_MAX_SQL_PARAMETERS = 256
_MAX_QUERY_ROWS = 10_000
_MAX_QUERY_COLUMNS = 256
_MAX_PROGRESS_INTERVAL = 1_000_000
_MAX_PROGRESS_CALLBACKS = 1_000_000
_MAX_SQLITE_LENGTH = 4 * 1_024 * 1_024
_MIN_SQLITE_INT = -(1 << 63)
_MAX_SQLITE_INT = (1 << 63) - 1


class SQLiteQueryBudgetExceeded(RuntimeError):
    """The installed progress handler interrupted the statement."""


class SQLiteRowLimitExceeded(RuntimeError):
    """The complete query result contains more rows than admitted."""


class SQLiteColumnLimitExceeded(RuntimeError):
    """The query result contains more columns than admitted."""


def _validated_query_limit(
    value: object,
    *,
    label: str,
    minimum: int,
    maximum: int,
) -> int:
    if type(value) is not int:
        raise TypeError(f"{label} must be an exact integer")
    if not minimum <= value <= maximum:
        raise ValueError(f"{label} is outside {minimum}..{maximum}")
    return value


def _validate_sqlite_value(value: object, *, label: str, max_bytes: int) -> None:
    if value is None:
        return
    if type(value) is int:
        if not _MIN_SQLITE_INT <= value <= _MAX_SQLITE_INT:
            raise ValueError(f"{label} is outside SQLite's integer range")
        return
    if type(value) is float:
        if not math.isfinite(value):
            raise ValueError(f"{label} must be finite")
        return
    if type(value) is str:
        if len(value) > max_bytes:
            raise ValueError(f"{label} exceeds the byte limit")
        try:
            encoded = value.encode("utf-8", errors="strict")
        except UnicodeEncodeError as error:
            raise ValueError(f"{label} must be valid UTF-8 text") from error
        if len(encoded) > max_bytes:
            raise ValueError(f"{label} exceeds the byte limit")
        return
    if type(value) is bytes:
        if len(value) > max_bytes:
            raise ValueError(f"{label} exceeds the byte limit")
        return
    raise TypeError(f"{label} has an unsupported exact SQLite value type")


def execute_trusted_sqlite_query(
    connection: sqlite3.Connection,
    sql: str,
    parameters: tuple[object, ...] = (),
    *,
    progress_interval: int,
    max_progress_callbacks: int,
    max_rows: int,
    max_columns: int = _MAX_QUERY_COLUMNS,
    max_value_bytes: int = _MAX_SQLITE_LENGTH,
) -> tuple[tuple[object, ...], ...]:
    """Return one complete bounded row result or raise a typed limit error."""
    if not isinstance(connection, sqlite3.Connection):
        raise TypeError("connection must be a sqlite3.Connection")
    if connection.row_factory is not None or connection.text_factory is not str:
        raise ValueError("connection must use default tuple rows and text decoding")
    if type(sql) is not str:
        raise TypeError("sql must be an exact string")
    if not 1 <= len(sql) <= _MAX_SQL_BYTES:
        raise ValueError("sql character length cannot fit the byte limit")
    try:
        sql_bytes = sql.encode("utf-8", errors="strict")
    except UnicodeEncodeError as error:
        raise ValueError("sql must be valid UTF-8 text") from error
    if not 1 <= len(sql_bytes) <= _MAX_SQL_BYTES or "\x00" in sql:
        raise ValueError("sql has an invalid byte length or contains NUL")

    if type(parameters) is not tuple:
        raise TypeError("parameters must be an exact tuple")
    if len(parameters) > _MAX_SQL_PARAMETERS:
        raise ValueError("parameter count exceeds the supported limit")

    progress_interval = _validated_query_limit(
        progress_interval,
        label="progress_interval",
        minimum=1,
        maximum=_MAX_PROGRESS_INTERVAL,
    )
    max_progress_callbacks = _validated_query_limit(
        max_progress_callbacks,
        label="max_progress_callbacks",
        minimum=1,
        maximum=_MAX_PROGRESS_CALLBACKS,
    )
    max_rows = _validated_query_limit(
        max_rows,
        label="max_rows",
        minimum=0,
        maximum=_MAX_QUERY_ROWS,
    )
    max_columns = _validated_query_limit(
        max_columns,
        label="max_columns",
        minimum=1,
        maximum=_MAX_QUERY_COLUMNS,
    )
    max_value_bytes = _validated_query_limit(
        max_value_bytes,
        label="max_value_bytes",
        minimum=1,
        maximum=_MAX_SQLITE_LENGTH,
    )
    for parameter_index, parameter in enumerate(parameters):
        _validate_sqlite_value(
            parameter,
            label=f"parameters[{parameter_index}]",
            max_bytes=max_value_bytes,
        )

    callback_count = 0
    interruption_requested = False

    def check_progress() -> int:
        nonlocal callback_count, interruption_requested
        callback_count += 1
        if callback_count >= max_progress_callbacks:
            interruption_requested = True
            return 1
        return 0

    cursor: sqlite3.Cursor | None = None
    try:
        connection.set_progress_handler(check_progress, progress_interval)

        cursor = connection.execute(sql, parameters)
        if cursor.description is None:
            raise ValueError("sql must produce a row result")
        if len(cursor.description) > max_columns:
            raise SQLiteColumnLimitExceeded(
                f"query returned more than {max_columns} columns"
            )
        rows = cursor.fetchmany(max_rows + 1)
        if len(rows) > max_rows:
            raise SQLiteRowLimitExceeded(
                f"query returned more than {max_rows} rows"
            )

        for row_index, row in enumerate(rows):
            if type(row) is not tuple:
                raise TypeError("connection returned a non-tuple row")
            for column_index, value in enumerate(row):
                _validate_sqlite_value(
                    value,
                    label=f"rows[{row_index}][{column_index}]",
                    max_bytes=max_value_bytes,
                )
        return tuple(rows)
    except sqlite3.DatabaseError as error:
        if (
            interruption_requested
            and getattr(error, "sqlite_errorcode", None) == sqlite3.SQLITE_INTERRUPT
        ):
            raise SQLiteQueryBudgetExceeded(
                f"query reached {callback_count} progress callbacks"
            ) from error
        raise
    finally:
        try:
            if cursor is not None:
                cursor.close()
        finally:
            connection.set_progress_handler(None, 0)
```

## Example

```python
connection = sqlite3.connect(":memory:")
try:
    rows = execute_trusted_sqlite_query(
        connection,
        "WITH data(x) AS (VALUES (?), (?), (?)) "
        "SELECT x, x * x FROM data ORDER BY x",
        (3, 1, 2),
        progress_interval=100,
        max_progress_callbacks=100,
        max_rows=3,
    )

    single_column_rows = execute_trusted_sqlite_query(
        connection,
        "SELECT 1",
        progress_interval=100,
        max_progress_callbacks=100,
        max_rows=1,
        max_columns=1,
    )

    column_limit_rejected = False
    try:
        execute_trusted_sqlite_query(
            connection,
            "SELECT 1, 2",
            progress_interval=100,
            max_progress_callbacks=100,
            max_rows=1,
            max_columns=1,
        )
    except SQLiteColumnLimitExceeded:
        column_limit_rejected = True

    budget_rejected = False
    try:
        execute_trusted_sqlite_query(
            connection,
            "WITH RECURSIVE numbers(x) AS ("
            "VALUES (0) UNION ALL SELECT x + 1 FROM numbers"
            ") SELECT x FROM numbers",
            progress_interval=100,
            max_progress_callbacks=1,
            max_rows=_MAX_QUERY_ROWS,
        )
    except SQLiteQueryBudgetExceeded:
        budget_rejected = True

    row_limit_rejected = False
    try:
        execute_trusted_sqlite_query(
            connection,
            "WITH RECURSIVE numbers(x) AS ("
            "VALUES (0) UNION ALL SELECT x + 1 FROM numbers WHERE x < 5"
            ") SELECT x FROM numbers ORDER BY x",
            progress_interval=1_000,
            max_progress_callbacks=1_000,
            max_rows=5,
        )
    except SQLiteRowLimitExceeded:
        row_limit_rejected = True

    syntax_preserved = False
    try:
        execute_trusted_sqlite_query(
            connection,
            "SELEC broken",
            progress_interval=100,
            max_progress_callbacks=100,
            max_rows=1,
        )
    except sqlite3.OperationalError as error:
        syntax_preserved = "syntax" in str(error).lower()

    cleanup_rows = connection.execute(
        "WITH RECURSIVE numbers(x) AS ("
        "VALUES (0) UNION ALL SELECT x + 1 FROM numbers WHERE x < 1000"
        ") SELECT sum(x) FROM numbers"
    ).fetchall()
finally:
    connection.close()

schema_source = sqlite3.connect(":memory:")
try:
    schema_source.execute(
        "CREATE TABLE schema_probe(value TEXT CHECK(length(value) < 1000))"
    )
    serialized_database = schema_source.serialize()
finally:
    schema_source.close()

fresh_connection = sqlite3.connect(":memory:")
try:
    fresh_connection.deserialize(serialized_database)
    fresh_schema_rows = execute_trusted_sqlite_query(
        fresh_connection,
        "SELECT 1",
        progress_interval=100,
        max_progress_callbacks=100,
        max_rows=1,
        max_value_bytes=16,
    )
finally:
    fresh_connection.close()

assert (
    rows == ((1, 1), (2, 4), (3, 9))
    and single_column_rows == ((1,),)
    and column_limit_rejected
    and budget_rejected
    and row_limit_rejected
    and syntax_preserved
    and cleanup_rows == [(500_500,)]
    and fresh_schema_rows == ((1,),)
)
```

## Trade-offs and Limitations

The progress guard requests interruption on the configured callback count,
after approximately `interval * callbacks` SQLite virtual-machine
instructions. It is not a wall-clock deadline: a blocking user-defined
function, filesystem stall, lock wait, or other work between callbacks can
still take arbitrarily long. Materializing a successful result uses
`O(rows * columns)` references plus retained cell payloads.

Progress handlers belong to the connection, so concurrent use could observe or
replace this temporary policy. The function requires a dedicated connection
with no prior handler, default tuple rows, and trusted read-only SQL. It always
removes its own handler, but cannot restore an unknown previous handler because
SQLite exposes no getter.

The helper does not lower SQLite's connection-global SQL or value limits:
SQLite also applies those limits while parsing stored schema, so changing them
for one query can reject an otherwise valid database. Input and returned cell
checks bound this API's copies, but SQLite may already have allocated result
values before the post-fetch validation runs.

This is not a SQL sandbox, authorization layer, memory limit, transaction
policy, or automatic rollback boundary. A side-effecting statement can change
state before failure. SQLite extensions, callbacks, custom row/text factories,
untrusted SQL, and concurrent connection use are outside the contract.

## Related Snippets

<!-- catalog:related:start -->
- [Page Bounded SQLite Rows with a Composite Keyset Cursor](page-bounded-sqlite-rows-with-a-composite-keyset-cursor.md)
- [Scope Caller-Owned SQLite Work with an Explicit Savepoint](scope-caller-owned-sqlite-work-with-an-explicit-savepoint.md)
- [Compile a Bounded T-String into SQLite Qmark SQL and Parameters](compile-a-bounded-t-string-into-sqlite-qmark-sql-and-parameters.md)
<!-- catalog:related:end -->
