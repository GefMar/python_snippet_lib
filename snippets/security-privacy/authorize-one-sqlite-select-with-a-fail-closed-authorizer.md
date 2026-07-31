---
title: "Authorize One SQLite SELECT with a Fail-Closed Authorizer"
snippet_type: recipe
use_cases:
  - persistence
  - security
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - ../storage-databases/execute-a-trusted-sqlite-query-under-progress-callback-and-row-caps.md
  - ../storage-databases/open-a-verified-read-only-sqlite-connection-under-a-closed-hardening-profile.md
  - ../storage-databases/compile-a-bounded-t-string-into-sqlite-qmark-sql-and-parameters.md
---

# Authorize One SQLite SELECT with a Fail-Closed Authorizer

## Idea and Problem

Execute one bounded SQLite SELECT while a fail-closed authorizer permits only named main-database columns and optional named functions.

SQLite calls an authorizer while it prepares operations. The callback below
returns `SQLITE_OK` only for `SQLITE_SELECT`, explicitly listed
`SQLITE_READ` pairs, and explicitly listed `SQLITE_FUNCTION` names. It returns
`SQLITE_DENY` for every other action and never uses `SQLITE_IGNORE`, so an
unknown operation cannot silently produce a partial or altered result.

The callback is connection-global state. It is installed before
`Connection.execute`, remains installed while the complete bounded result is
fetched, and is removed with `set_authorizer(None)` in `finally`. Input and
result limits keep the helper's own retained SQL, parameters, rows, columns,
and SQLite scalar values bounded.

## When to Use

Use this recipe for one direct read over a trusted SQLite schema when a closed
table-column policy is more appropriate than reviewing every possible query
shape. The caller must own a dedicated, non-concurrent connection with no
active transaction, no previous authorizer, and the default tuple-row and text
factories. The function also verifies that `main` is the connection's only
database, excluding pre-attached and temporary schemas whose synthetic
no-column read events SQLite cannot always distinguish by database name.

An empty column name can authorize a table read such as `count(*)` over an
explicitly `main`-qualified table, but only when that exact `(table, "")` pair
is listed. SQLite also emits a synthetic no-column event after some permitted
`INTEGER PRIMARY KEY` reads; the callback admits that event only after a named
column from the same main table has already passed the policy.

The connection must also have a trusted schema and no loaded extensions,
application-defined SQL functions, virtual-table implementations, custom
collations, adapters, or converters that can run code outside this policy.
Use a verified read-only connection when database mutation must also be
prevented, a progress handler when trusted SQL needs a work budget, and bound
parameters rather than interpolated SQL for runtime values.

## Implementation

```python
import math
import re
import sqlite3

_MAX_SQL_BYTES = 65_536
_MAX_PARAMETERS = 256
_MAX_ALLOWED_READS = 512
_MAX_ALLOWED_FUNCTIONS = 64
_MAX_IDENTIFIER_BYTES = 255
_MAX_FUNCTION_NAME_BYTES = 64
_MAX_ROWS = 10_000
_MAX_COLUMNS = 256
_MAX_VALUE_BYTES = 4 * 1_024 * 1_024
_MIN_SQLITE_INT = -(1 << 63)
_MAX_SQLITE_INT = (1 << 63) - 1
_SELECT_PREFIX = re.compile(
    r"\A[ \t\r\n]*SELECT(?![A-Za-z0-9_])",
    flags=re.ASCII | re.IGNORECASE,
)
_FUNCTION_NAME = re.compile(
    r"[A-Za-z_][A-Za-z0-9_]*\Z",
    flags=re.ASCII,
)


class SQLiteSelectAuthorizationError(PermissionError):
    """The closed SQLite authorizer rejected an operation."""


class SQLiteSelectRowLimitExceeded(RuntimeError):
    """The complete SELECT result contains too many rows."""


class SQLiteSelectColumnLimitExceeded(RuntimeError):
    """The SELECT result contains too many columns."""


def _bounded_select_limit(
    value: object,
    *,
    name: str,
    minimum: int,
    maximum: int,
) -> int:
    if type(value) is not int:
        raise TypeError(f"{name} must be an exact integer")
    if not minimum <= value <= maximum:
        raise ValueError(f"{name} is outside {minimum}..{maximum}")
    return value


def _bounded_sqlite_text(
    value: object,
    *,
    name: str,
    allow_empty: bool,
    max_bytes: int,
) -> str:
    if type(value) is not str:
        raise TypeError(f"{name} must be an exact string")
    if not allow_empty and not value:
        raise ValueError(f"{name} must not be empty")
    if "\x00" in value or len(value) > max_bytes:
        raise ValueError(f"{name} has an invalid character length or contains NUL")
    try:
        encoded = value.encode("utf-8", errors="strict")
    except UnicodeEncodeError as error:
        raise ValueError(f"{name} must be valid UTF-8 text") from error
    if len(encoded) > max_bytes:
        raise ValueError(f"{name} exceeds the supported UTF-8 byte count")
    return value


def _validated_allowed_reads(
    allowed_reads: object,
) -> frozenset[tuple[str, str]]:
    if type(allowed_reads) is not tuple:
        raise TypeError("allowed_reads must be an exact tuple")
    if len(allowed_reads) > _MAX_ALLOWED_READS:
        raise ValueError("allowed_reads exceeds the supported entry count")

    validated: set[tuple[str, str]] = set()
    for index, entry in enumerate(allowed_reads):
        if type(entry) is not tuple:
            raise TypeError(f"allowed_reads[{index}] must be an exact tuple")
        if len(entry) != 2:
            raise ValueError(f"allowed_reads[{index}] must contain two items")
        table = _bounded_sqlite_text(
            entry[0],
            name=f"allowed_reads[{index}][0]",
            allow_empty=False,
            max_bytes=_MAX_IDENTIFIER_BYTES,
        )
        column = _bounded_sqlite_text(
            entry[1],
            name=f"allowed_reads[{index}][1]",
            allow_empty=True,
            max_bytes=_MAX_IDENTIFIER_BYTES,
        )
        pair = (table, column)
        if pair in validated:
            raise ValueError("allowed_reads must not contain duplicates")
        validated.add(pair)
    return frozenset(validated)


def _validated_allowed_functions(
    allowed_functions: object,
) -> frozenset[str]:
    if type(allowed_functions) is not tuple:
        raise TypeError("allowed_functions must be an exact tuple")
    if len(allowed_functions) > _MAX_ALLOWED_FUNCTIONS:
        raise ValueError("allowed_functions exceeds the supported entry count")

    validated: set[str] = set()
    for index, value in enumerate(allowed_functions):
        if type(value) is not str:
            raise TypeError(f"allowed_functions[{index}] must be an exact string")
        if len(value) > _MAX_FUNCTION_NAME_BYTES or _FUNCTION_NAME.fullmatch(value) is None:
            raise ValueError(f"allowed_functions[{index}] must be a bounded ASCII function name")
        canonical = value.lower()
        if canonical in validated:
            raise ValueError("allowed_functions must not contain case-insensitive duplicates")
        validated.add(canonical)
    return frozenset(validated)


def _validate_sqlite_value(value: object, *, name: str, max_bytes: int) -> None:
    if value is None:
        return
    if type(value) is int:
        if not _MIN_SQLITE_INT <= value <= _MAX_SQLITE_INT:
            raise ValueError(f"{name} is outside SQLite's integer range")
        return
    if type(value) is float:
        if not math.isfinite(value):
            raise ValueError(f"{name} must be finite")
        return
    if type(value) is str:
        if len(value) > max_bytes:
            raise ValueError(f"{name} exceeds the supported byte count")
        try:
            encoded = value.encode("utf-8", errors="strict")
        except UnicodeEncodeError as error:
            raise ValueError(f"{name} must be valid UTF-8 text") from error
        if len(encoded) > max_bytes:
            raise ValueError(f"{name} exceeds the supported byte count")
        return
    if type(value) is bytes:
        if len(value) > max_bytes:
            raise ValueError(f"{name} exceeds the supported byte count")
        return
    raise TypeError(f"{name} has an unsupported exact SQLite value type")


def execute_authorized_sqlite_select(
    connection: sqlite3.Connection,
    sql: str,
    parameters: tuple[object, ...] = (),
    *,
    allowed_reads: tuple[tuple[str, str], ...],
    allowed_functions: tuple[str, ...] = (),
    max_rows: int,
    max_columns: int = _MAX_COLUMNS,
    max_value_bytes: int = _MAX_VALUE_BYTES,
) -> tuple[tuple[object, ...], ...]:
    """Return one complete result admitted by a closed SQLite read policy."""
    if not isinstance(connection, sqlite3.Connection):
        raise TypeError("connection must be a sqlite3.Connection")
    if connection.row_factory is not None or connection.text_factory is not str:
        raise ValueError("connection must use default tuple rows and text decoding")
    if connection.in_transaction:
        raise ValueError("connection must not have an active transaction")

    database_cursor = connection.execute("PRAGMA database_list")
    try:
        database_names = tuple(row[1] for row in database_cursor.fetchall())
    finally:
        database_cursor.close()
    if database_names != ("main",):
        raise ValueError("connection must expose only the main database")

    sql = _bounded_sqlite_text(
        sql,
        name="sql",
        allow_empty=False,
        max_bytes=_MAX_SQL_BYTES,
    )
    if _SELECT_PREFIX.match(sql) is None:
        raise ValueError("sql must start directly with SELECT")

    if type(parameters) is not tuple:
        raise TypeError("parameters must be an exact tuple")
    if len(parameters) > _MAX_PARAMETERS:
        raise ValueError("parameter count exceeds the supported limit")

    checked_max_rows = _bounded_select_limit(
        max_rows,
        name="max_rows",
        minimum=0,
        maximum=_MAX_ROWS,
    )
    checked_max_columns = _bounded_select_limit(
        max_columns,
        name="max_columns",
        minimum=1,
        maximum=_MAX_COLUMNS,
    )
    checked_max_value_bytes = _bounded_select_limit(
        max_value_bytes,
        name="max_value_bytes",
        minimum=1,
        maximum=_MAX_VALUE_BYTES,
    )
    for index, parameter in enumerate(parameters):
        _validate_sqlite_value(
            parameter,
            name=f"parameters[{index}]",
            max_bytes=checked_max_value_bytes,
        )

    read_policy = _validated_allowed_reads(allowed_reads)
    function_policy = _validated_allowed_functions(allowed_functions)
    authorized_named_tables: set[str] = set()
    denied = False

    def authorize(
        action_code: int,
        argument_one: str | None,
        argument_two: str | None,
        database_name: str | None,
        source: str | None,
    ) -> int:
        nonlocal denied

        permitted = False
        if source is None:
            if action_code == sqlite3.SQLITE_SELECT:
                permitted = argument_one is None and argument_two is None and database_name is None
            elif action_code == sqlite3.SQLITE_READ:
                if type(argument_one) is str and type(argument_two) is str:
                    if database_name == "main" and (argument_one, argument_two) in read_policy:
                        permitted = True
                        if argument_two:
                            authorized_named_tables.add(argument_one)
                    elif (
                        database_name is None
                        and argument_two == ""
                        and (
                            (argument_one, "") in read_policy
                            or argument_one in authorized_named_tables
                        )
                    ):
                        permitted = True
            elif action_code == sqlite3.SQLITE_FUNCTION:
                permitted = (
                    argument_one is None
                    and type(argument_two) is str
                    and argument_two.isascii()
                    and database_name is None
                    and argument_two.lower() in function_policy
                )

        if permitted:
            return sqlite3.SQLITE_OK
        denied = True
        return sqlite3.SQLITE_DENY

    cursor: sqlite3.Cursor | None = None
    try:
        connection.set_authorizer(authorize)
        cursor = connection.execute(sql, parameters)
        if cursor.description is None:
            raise ValueError("sql must produce a row result")
        column_count = len(cursor.description)
        if not 1 <= column_count <= checked_max_columns:
            raise SQLiteSelectColumnLimitExceeded(
                f"SELECT returned more than {checked_max_columns} columns"
            )

        rows = cursor.fetchmany(checked_max_rows + 1)
        if len(rows) > checked_max_rows:
            raise SQLiteSelectRowLimitExceeded(f"SELECT returned more than {checked_max_rows} rows")
        for row_index, row in enumerate(rows):
            if type(row) is not tuple or len(row) != column_count:
                raise TypeError("connection returned a non-default row shape")
            for column_index, value in enumerate(row):
                _validate_sqlite_value(
                    value,
                    name=f"rows[{row_index}][{column_index}]",
                    max_bytes=checked_max_value_bytes,
                )
        return tuple(rows)
    except sqlite3.DatabaseError as error:
        if denied:
            raise SQLiteSelectAuthorizationError(
                "the SQLite authorizer denied the SELECT"
            ) from error
        raise
    finally:
        try:
            if cursor is not None:
                cursor.close()
        finally:
            connection.set_authorizer(None)


```

## Example

```python
connection = sqlite3.connect(":memory:")
try:
    connection.execute(
        "CREATE TABLE account("
        "id INTEGER PRIMARY KEY, tenant TEXT, name TEXT, balance INTEGER, secret TEXT)"
    )
    connection.executemany(
        "INSERT INTO account VALUES (?, ?, ?, ?, ?)",
        (
            (1, "north", "Ada", 40, "red"),
            (2, "south", "Grace", 90, "blue"),
            (3, "north", "Lin", 60, "green"),
        ),
    )
    connection.commit()

    rows = execute_authorized_sqlite_select(
        connection,
        "SELECT upper(name), balance FROM account WHERE tenant = ? ORDER BY id",
        ("north",),
        allowed_reads=(
            ("account", "name"),
            ("account", "balance"),
            ("account", "tenant"),
            ("account", "id"),
        ),
        allowed_functions=("upper",),
        max_rows=2,
        max_columns=2,
        max_value_bytes=32,
    )
    count_rows = execute_authorized_sqlite_select(
        connection,
        "SELECT count(*) FROM main.account",
        allowed_reads=(("account", ""),),
        allowed_functions=("count",),
        max_rows=1,
        max_columns=1,
        max_value_bytes=32,
    )
    id_rows = execute_authorized_sqlite_select(
        connection,
        "SELECT id FROM account ORDER BY id",
        allowed_reads=(("account", "id"),),
        max_rows=3,
        max_columns=1,
        max_value_bytes=32,
    )

    secret_denied = False
    try:
        execute_authorized_sqlite_select(
            connection,
            "SELECT secret FROM account ORDER BY id",
            allowed_reads=(("account", "id"),),
            max_rows=3,
            max_columns=1,
            max_value_bytes=32,
        )
    except SQLiteSelectAuthorizationError:
        secret_denied = True

    rows_after_cleanup = connection.execute("SELECT secret FROM account ORDER BY id").fetchall()
finally:
    connection.close()

assert (rows, count_rows, id_rows, secret_denied, rows_after_cleanup) == (
    (("ADA", 40), ("LIN", 60)),
    ((3,),),
    ((1,), (2,), (3,)),
    True,
    [("red",), ("blue",), ("green",)],
)
```

## Trade-offs and Limitations

For `R` returned rows and `C` columns, the successful result retains
`O(R * C)` scalar references and their bounded payloads. Fetching one extra
row proves that a successful result is complete rather than silently
truncated. SQLite may allocate values before this post-fetch validation, so
the byte limit bounds what this helper accepts and retains, not every internal
allocation made by SQLite.

The policy authorizes column access, not row access. Predicates, aggregates,
and allowed functions can reveal information derived from every permitted
column value; this is not row-level security. A function-name allowlist is
safe here only because extensions and application-defined functions are
excluded by the connection precondition. Custom collations and hostile or
virtual-table schemas can execute code through mechanisms that this closed
opcode list does not model.

An authorizer is not a CPU, elapsed-time, recursion, temporary-storage, or
memory sandbox. This helper deliberately has no progress handler or SQLite
connection hardening profile; those are separate policies taught by the
related snippets. It also accepts only a direct `SELECT` prefix, so comments,
`WITH`, `EXPLAIN`, and other top-level forms are outside the contract.

SQLite exposes no getter for a connection's current authorizer. The caller
must therefore supply a connection with no prior authorizer: this function
overwrites that unobservable state and always clears it rather than pretending
to restore it. Dedicated ownership and absence of concurrent use are likewise
caller-enforced preconditions. The explicit database-list check rejects
pre-attached and temporary schemas so SQLite's database-less synthetic read
event cannot cross a schema boundary.

## Related Snippets

<!-- catalog:related:start -->
- [Execute a Trusted SQLite Query Under Progress-Callback and Row Caps](../storage-databases/execute-a-trusted-sqlite-query-under-progress-callback-and-row-caps.md)
- [Open a Verified Read-Only SQLite Connection Under a Closed Hardening Profile](../storage-databases/open-a-verified-read-only-sqlite-connection-under-a-closed-hardening-profile.md)
- [Compile a Bounded T-String into SQLite Qmark SQL and Parameters](../storage-databases/compile-a-bounded-t-string-into-sqlite-qmark-sql-and-parameters.md)
<!-- catalog:related:end -->
