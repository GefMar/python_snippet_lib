---
title: "Inspect a Bounded SQLite Database Header from In-Memory Bytes"
snippet_type: recipe
use_cases:
  - parsing
  - persistence
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - open-a-verified-read-only-sqlite-connection-under-a-closed-hardening-profile.md
  - execute-a-trusted-sqlite-query-under-progress-callback-and-row-caps.md
  - recover-a-verified-prefix-from-a-bounded-crc-framed-byte-log.md
---

# Inspect a Bounded SQLite Database Header from In-Memory Bytes

## Idea and Problem

Decode selected fields from an exact 100-byte SQLite database header after checking the cross-field rules needed to interpret those fields safely.

The result makes a future write-version requirement explicit, distinguishes a
trusted in-header page count from an obsolete one, and preserves unsigned and
signed fields according to the file format. It inspects bytes already owned by
the caller and never opens a database or filesystem path.

## When to Use

Use this parser when a bounded ingestion or diagnostic boundary needs a small,
immutable summary before deciding whether a database deserves deeper
inspection. It is useful for format triage, fixtures, and metadata displays
where exactly the first 100 bytes have already been read.

Use SQLite itself plus application-specific integrity checks before trusting
database contents. A plausible header does not prove that later pages exist,
are internally consistent, or are safe to query.

## Implementation

```python
from dataclasses import dataclass
from enum import IntEnum

_SQLITE_MAGIC = b"SQLite format 3\x00"
_MAX_PAGE_NUMBER = 0xFFFFFFFE


class SQLiteHeaderError(ValueError):
    """The bytes are outside this bounded SQLite-header profile."""


class SQLiteTextEncoding(IntEnum):
    UTF_8 = 1
    UTF_16_LE = 2
    UTF_16_BE = 3


@dataclass(frozen=True, slots=True)
class SQLiteDatabaseHeader:
    page_size: int
    usable_page_size: int
    write_version: int
    read_version: int
    requires_read_only: bool
    reserved_bytes_per_page: int
    file_change_counter: int
    database_pages: int | None
    schema_cookie: int
    schema_format: int
    suggested_cache_pages: int
    largest_root_btree_page: int
    text_encoding: SQLiteTextEncoding | None
    user_version: int
    incremental_vacuum: bool
    application_id: int
    version_valid_for: int
    sqlite_version_number: int


def _unsigned_32(header: bytes, offset: int) -> int:
    return int.from_bytes(header[offset : offset + 4], "big")


def inspect_sqlite_database_header(header: bytes) -> SQLiteDatabaseHeader:
    """Decode one exact SQLite database-header image without reading a file."""
    if type(header) is not bytes:
        raise TypeError("header must be exact bytes")
    if len(header) != 100:
        raise SQLiteHeaderError("header must contain exactly 100 bytes")
    if header[:16] != _SQLITE_MAGIC:
        raise SQLiteHeaderError("SQLite header magic is invalid")

    page_size_code = int.from_bytes(header[16:18], "big")
    if page_size_code == 1:
        page_size = 65_536
    elif 512 <= page_size_code <= 32_768 and page_size_code & (page_size_code - 1) == 0:
        page_size = page_size_code
    else:
        raise SQLiteHeaderError("page size code is invalid")

    write_version = header[18]
    read_version = header[19]
    if write_version == 0:
        raise SQLiteHeaderError("write version must be nonzero")
    if read_version not in (1, 2):
        raise SQLiteHeaderError("read version is unsupported")

    reserved_bytes = header[20]
    usable_page_size = page_size - reserved_bytes
    if usable_page_size < 480:
        raise SQLiteHeaderError("usable page size is smaller than 480 bytes")
    if header[21:24] != bytes((64, 32, 32)):
        raise SQLiteHeaderError("payload fractions are invalid")

    file_change_counter = _unsigned_32(header, 24)
    raw_database_pages = _unsigned_32(header, 28)
    version_valid_for = _unsigned_32(header, 92)
    if raw_database_pages != 0 and file_change_counter == version_valid_for:
        if raw_database_pages > _MAX_PAGE_NUMBER:
            raise SQLiteHeaderError("trusted database page count is too large")
        database_pages: int | None = raw_database_pages
    else:
        database_pages = None

    schema_format = _unsigned_32(header, 44)
    if schema_format > 4:
        raise SQLiteHeaderError("schema format is invalid")

    raw_encoding = _unsigned_32(header, 56)
    allowed_encodings = range(4) if schema_format == 0 else range(1, 4)
    if raw_encoding not in allowed_encodings:
        raise SQLiteHeaderError("text encoding is inconsistent with schema format")
    text_encoding = None if raw_encoding == 0 else SQLiteTextEncoding(raw_encoding)

    largest_root_page = _unsigned_32(header, 52)
    raw_incremental_vacuum = _unsigned_32(header, 64)
    if largest_root_page == 0:
        if raw_incremental_vacuum != 0:
            raise SQLiteHeaderError(
                "incremental vacuum requires a largest root page",
            )
    else:
        if largest_root_page > _MAX_PAGE_NUMBER:
            raise SQLiteHeaderError("largest root page number is too large")
        if database_pages is not None and largest_root_page > database_pages:
            raise SQLiteHeaderError("largest root page exceeds trusted page count")

    if header[72:92] != bytes(20):
        raise SQLiteHeaderError("reserved header bytes must be zero")

    return SQLiteDatabaseHeader(
        page_size=page_size,
        usable_page_size=usable_page_size,
        write_version=write_version,
        read_version=read_version,
        requires_read_only=write_version > 2,
        reserved_bytes_per_page=reserved_bytes,
        file_change_counter=file_change_counter,
        database_pages=database_pages,
        schema_cookie=_unsigned_32(header, 40),
        schema_format=schema_format,
        suggested_cache_pages=int.from_bytes(header[48:52], "big", signed=True),
        largest_root_btree_page=largest_root_page,
        text_encoding=text_encoding,
        user_version=_unsigned_32(header, 60),
        incremental_vacuum=raw_incremental_vacuum != 0,
        application_id=_unsigned_32(header, 68),
        version_valid_for=version_valid_for,
        sqlite_version_number=_unsigned_32(header, 96),
    )
```

## Example

```python


def write_unsigned_32(buffer: bytearray, offset: int, value: int) -> None:
    buffer[offset : offset + 4] = value.to_bytes(4, "big")


image = bytearray(100)
image[:16] = _SQLITE_MAGIC
image[16:18] = (4096).to_bytes(2, "big")
image[18:24] = bytes((7, 2, 32, 64, 32, 32))
write_unsigned_32(image, 24, 9)
write_unsigned_32(image, 28, 4)
write_unsigned_32(image, 40, 8)
write_unsigned_32(image, 44, 4)
image[48:52] = (-2000).to_bytes(4, "big", signed=True)
write_unsigned_32(image, 52, 3)
write_unsigned_32(image, 56, SQLiteTextEncoding.UTF_8)
write_unsigned_32(image, 60, 5)
write_unsigned_32(image, 64, 1)
write_unsigned_32(image, 68, 0x53514C54)
write_unsigned_32(image, 92, 9)
write_unsigned_32(image, 96, 3_049_001)

summary = inspect_sqlite_database_header(bytes(image))

stale_count_image = bytearray(image)
write_unsigned_32(stale_count_image, 92, 10)
stale_count = inspect_sqlite_database_header(bytes(stale_count_image))

assert (
    summary.page_size,
    summary.usable_page_size,
    summary.requires_read_only,
    summary.database_pages,
    summary.suggested_cache_pages,
    summary.text_encoding,
    summary.incremental_vacuum,
    stale_count.database_pages,
) == (4096, 4064, True, 4, -2000, SQLiteTextEncoding.UTF_8, True, None)
```

## Trade-offs and Limitations

The parser performs constant work and allocates one fixed-size result. It
accepts write-version bytes above `2` so a caller can display metadata, but
marks every such result `requires_read_only`; it never claims that a current
SQLite library can write that format. A nonzero page count is exposed only
when the file-change counter equals the version-valid-for number.

Only the documented header relationships represented in the function are
checked. The result does not validate file length, page types, freelist state,
schema objects, checksums, journal or WAL state, or cross-page integrity. It
also does not prove that an untrusted page count is wrong: `None` means only
that the header cannot currently authenticate that field through its two
counters.

## Related Snippets

<!-- catalog:related:start -->
- [Open a Verified Read-Only SQLite Connection Under a Closed Hardening Profile](open-a-verified-read-only-sqlite-connection-under-a-closed-hardening-profile.md)
- [Execute a Trusted SQLite Query Under Progress-Callback and Row Caps](execute-a-trusted-sqlite-query-under-progress-callback-and-row-caps.md)
- [Recover a Verified Prefix from a Bounded CRC-Framed Byte Log](recover-a-verified-prefix-from-a-bounded-crc-framed-byte-log.md)
<!-- catalog:related:end -->
