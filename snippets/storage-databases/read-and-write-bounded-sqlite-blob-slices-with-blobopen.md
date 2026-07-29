---
title: "Read and Write Bounded SQLite BLOB Slices with blobopen"
snippet_type: pattern
use_cases:
  - persistence
  - resource-management
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - read-an-exact-posix-byte-range-with-os-pread-without-moving-the-file-descriptor-offset.md
  - commit-one-sqlite-item-mutation-and-its-checkpoint-atomically.md
  - scope-caller-owned-sqlite-work-with-an-explicit-savepoint.md
---

# Read and Write Bounded SQLite BLOB Slices with blobopen

## Idea and Problem

Transfer one bounded slice of a fixed-size SQLite BLOB without materializing the complete value or taking ownership of the caller's transaction.

`sqlite3.Connection.blobopen` exposes a seekable handle to one BLOB in a rowid
table. The helper opens only a fixed database, table, and column, validates the
stored BLOB size and complete slice bounds, then reads or overwrites exactly the
requested bytes. A context manager closes every handle on success or failure.

Incremental BLOB writes cannot change the stored length. Requiring an already
active transaction for writes makes commit and rollback decisions remain
explicitly caller-owned.

## When to Use

Use this pattern with the fixed `main.blob_items` schema when an application
needs a small range from a larger stored BLOB or must replace bytes in place.
It is useful for bounded headers, blocks, or checkpoints inside an otherwise
fixed-size binary value.

Use ordinary parameterized SQL for small values. Use an external object store
or a purpose-built streaming design when values must grow, shrink, be accessed
concurrently through long-lived handles, or move between database engines.

## Implementation

```python
import sqlite3

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_BLOB_BYTES = 1 << 20
_MAX_TRANSFER_BYTES = 65_536


def _validate_connection(connection: object) -> sqlite3.Connection:
    if type(connection) is not sqlite3.Connection:
        raise TypeError("connection must be an exact sqlite3.Connection")
    return connection


def _validate_row_id(row_id: object) -> int:
    if type(row_id) is not int:
        raise TypeError("row_id must be an exact integer")
    if not _MIN_INT64 <= row_id <= _MAX_INT64:
        raise ValueError("row_id is outside the signed 64-bit range")
    return row_id


def _validate_offset(offset: object) -> int:
    if type(offset) is not int:
        raise TypeError("offset must be an exact integer")
    if not 0 <= offset <= _MAX_BLOB_BYTES:
        raise ValueError("offset is outside the supported range")
    return offset


def _validate_read_length(length: object) -> int:
    if type(length) is not int:
        raise TypeError("length must be an exact integer")
    if not 0 <= length <= _MAX_TRANSFER_BYTES:
        raise ValueError("length is outside the supported transfer range")
    return length


def _validate_replacement(replacement: object) -> bytes:
    if type(replacement) is not bytes:
        raise TypeError("replacement must be exact bytes")
    if len(replacement) > _MAX_TRANSFER_BYTES:
        raise ValueError("replacement exceeds the supported transfer size")
    return replacement


def _validate_blob_bounds(
    blob: sqlite3.Blob,
    offset: int,
    transfer_length: int,
) -> int:
    blob_length = len(blob)
    if blob_length > _MAX_BLOB_BYTES:
        raise ValueError("stored BLOB exceeds the trusted schema limit")
    if offset > blob_length:
        raise ValueError("offset is beyond the end of the stored BLOB")
    if transfer_length > blob_length - offset:
        raise ValueError("transfer extends beyond the end of the stored BLOB")
    return blob_length


def read_sqlite_blob_slice(
    connection: sqlite3.Connection,
    row_id: int,
    *,
    offset: int,
    length: int,
) -> bytes:
    """Read one admitted slice from main.blob_items.payload."""
    connection = _validate_connection(connection)
    row_id = _validate_row_id(row_id)
    offset = _validate_offset(offset)
    length = _validate_read_length(length)

    with connection.blobopen(
        "blob_items",
        "payload",
        row_id,
        readonly=True,
        name="main",
    ) as blob:
        _validate_blob_bounds(blob, offset, length)
        blob.seek(offset)
        result = blob.read(length)
        if type(result) is not bytes or len(result) != length:
            raise RuntimeError("SQLite BLOB read did not return the exact slice")
        return result


def write_sqlite_blob_slice(
    connection: sqlite3.Connection,
    row_id: int,
    replacement: bytes,
    *,
    offset: int,
) -> None:
    """Overwrite one admitted slice inside a caller-owned transaction."""
    connection = _validate_connection(connection)
    row_id = _validate_row_id(row_id)
    offset = _validate_offset(offset)
    replacement = _validate_replacement(replacement)
    if not connection.in_transaction:
        raise RuntimeError("an active caller-owned transaction is required")

    with connection.blobopen(
        "blob_items",
        "payload",
        row_id,
        readonly=False,
        name="main",
    ) as blob:
        _validate_blob_bounds(blob, offset, len(replacement))
        blob.seek(offset)
        blob.write(replacement)
```

## Example

```python
connection = sqlite3.connect(":memory:", isolation_level=None)
connection.executescript(
    """
    CREATE TABLE blob_items (
        item_id INTEGER PRIMARY KEY,
        payload BLOB NOT NULL CHECK (
            typeof(payload) = 'blob'
            AND length(payload) <= 1048576
        )
    ) STRICT;
    """
)
original = bytes(range(32))
connection.execute(
    "INSERT INTO blob_items(item_id, payload) VALUES (?, ?)",
    (7, original),
)

assert read_sqlite_blob_slice(
    connection,
    7,
    offset=8,
    length=5,
) == bytes(range(8, 13))
assert read_sqlite_blob_slice(connection, 7, offset=32, length=0) == b""

connection.execute("BEGIN")
write_sqlite_blob_slice(connection, 7, b"ABCD", offset=10)
write_sqlite_blob_slice(connection, 7, b"", offset=32)
changed = read_sqlite_blob_slice(connection, 7, offset=8, length=8)
transaction_still_active = connection.in_transaction
connection.execute("ROLLBACK")
restored = read_sqlite_blob_slice(connection, 7, offset=0, length=32)
connection.close()

assert changed == bytes([8, 9]) + b"ABCD" + bytes([14, 15])
assert transaction_still_active
assert restored == original
```

## Trade-offs and Limitations

For transfer size `k`, SQLite and Python move `O(k)` bytes. Reads allocate
`O(k)` returned state; writes use `O(1)` additional Python state beyond the
replacement supplied by the caller. The database engine can still perform
page-cache, locking, journal, or WAL work not described by this byte count.

The contract is specific to
`main.blob_items(item_id INTEGER PRIMARY KEY, payload BLOB NOT NULL)` with its
type and one-mebibyte length checks. Incremental BLOB handles require a rowid
table, cannot resize a value, and can fail if the row is missing or database
state changes invalidate the handle.

The helper does not create the schema, allocate `zeroblob` values, choose
isolation, begin or end transactions, retry lock failures, schedule chunks,
coordinate concurrent handles, manage the connection, accept dynamic
identifiers, support `WITHOUT ROWID`, or provide cross-database portability.

## Related Snippets

<!-- catalog:related:start -->
- [Read an Exact POSIX Byte Range with os.pread Without Moving the File Descriptor Offset](read-an-exact-posix-byte-range-with-os-pread-without-moving-the-file-descriptor-offset.md)
- [Commit One SQLite Item Mutation and Its Checkpoint Atomically](commit-one-sqlite-item-mutation-and-its-checkpoint-atomically.md)
- [Scope Caller-Owned SQLite Work with an Explicit Savepoint](scope-caller-owned-sqlite-work-with-an-explicit-savepoint.md)
<!-- catalog:related:end -->
