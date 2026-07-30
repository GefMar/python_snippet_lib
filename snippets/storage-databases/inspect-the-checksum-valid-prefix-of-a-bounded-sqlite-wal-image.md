---
title: "Inspect the Checksum-Valid Prefix of a Bounded SQLite WAL Image"
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
  - inspect-a-bounded-sqlite-database-header-from-in-memory-bytes.md
  - recover-a-verified-prefix-from-a-bounded-crc-framed-byte-log.md
  - open-a-verified-read-only-sqlite-connection-under-a-closed-hardening-profile.md
---

# Inspect the Checksum-Valid Prefix of a Bounded SQLite WAL Image

## Idea and Problem

Locate the consecutive checksum-valid and last committed prefixes of one bounded immutable SQLite write-ahead-log image without retaining page bodies.

A WAL can contain valid committed frames followed by valid uncommitted work, a
partially written tail, or complete stale slots left after an invalid frame.
This parser follows SQLite's cumulative checksum chain, stops permanently at
the first invalid complete frame, and reports both the checksum-valid boundary
and the last commit boundary.

## When to Use

Use this recipe for offline diagnostics, fixtures, and bounded ingestion after
capturing an immutable WAL image while no writer can change it. It is useful
when callers need frame locations and prefix lengths without copying database
pages into a second representation.

Use SQLite itself for recovery and for reading a live database. A WAL image
must be captured together with the database under an application-appropriate
quiescence or snapshot protocol; parsing a changing `-wal` file cannot produce
a coherent view.

## Implementation

```python
from dataclasses import dataclass
from enum import Enum

_WAL_MAGIC_LITTLE_CHECKSUM = 0x377F0682
_WAL_MAGIC_BIG_CHECKSUM = 0x377F0683
_WAL_VERSION = 3_007_000
_WAL_HEADER_BYTES = 32
_FRAME_HEADER_BYTES = 24
_MIN_PAGE_SIZE = 512
_MAX_PAGE_SIZE = 65_536
_MAX_PAGE_NUMBER = 0xFFFF_FFFE
_MAX_FRAMES = 256
_MAX_WAL_BYTES = 16_783_392
_UINT32_MASK = 0xFFFF_FFFF


class SQLiteWalError(ValueError):
    """The bytes are outside this bounded SQLite-WAL profile."""


class SQLiteWalChecksumByteOrder(Enum):
    LITTLE = "little"
    BIG = "big"


class SQLiteWalStopReason(Enum):
    END_OF_IMAGE = "end-of-image"
    PARTIAL_TAIL = "partial-tail"
    SALT_MISMATCH = "salt-mismatch"
    INVALID_PAGE_NUMBER = "invalid-page-number"
    INVALID_DATABASE_SIZE = "invalid-database-size"
    CHECKSUM_MISMATCH = "checksum-mismatch"


@dataclass(frozen=True, slots=True)
class SQLiteWalHeader:
    magic: int
    version: int
    page_size: int
    checkpoint_sequence: int
    salt_1: int
    salt_2: int
    checksum_1: int
    checksum_2: int
    checksum_byte_order: SQLiteWalChecksumByteOrder


@dataclass(frozen=True, slots=True)
class SQLiteWalFrame:
    frame_number: int
    offset: int
    page_number: int
    database_size_pages: int


@dataclass(frozen=True, slots=True)
class SQLiteWalInspection:
    header: SQLiteWalHeader
    valid_frames: tuple[SQLiteWalFrame, ...]
    checksum_valid_prefix_length: int
    committed_prefix_length: int
    valid_frame_count: int
    last_committed_frame_number: int | None
    last_committed_database_pages: int | None
    valid_uncommitted_frame_count: int
    stop_reason: SQLiteWalStopReason
    first_invalid_frame_number: int | None
    nonprefix_complete_slot_count: int
    partial_tail_offset: int
    partial_tail_length: int


def _unsigned_32_be(image: bytes, offset: int) -> int:
    return int.from_bytes(image[offset : offset + 4], "big")


def _extend_wal_checksum(
    image: memoryview,
    start: int,
    stop: int,
    byte_order: SQLiteWalChecksumByteOrder,
    state: tuple[int, int],
) -> tuple[int, int]:
    if start < 0 or stop > len(image) or (stop - start) % 8 != 0:
        raise AssertionError("checksum span must be an in-bounds multiple of 8 bytes")

    checksum_1, checksum_2 = state
    for offset in range(start, stop, 8):
        word_1 = int.from_bytes(image[offset : offset + 4], byte_order.value)
        word_2 = int.from_bytes(image[offset + 4 : offset + 8], byte_order.value)
        checksum_1 = (checksum_1 + word_1 + checksum_2) & _UINT32_MASK
        checksum_2 = (checksum_2 + word_2 + checksum_1) & _UINT32_MASK
    return checksum_1, checksum_2


def inspect_sqlite_wal_image(image: bytes) -> SQLiteWalInspection:
    """Inspect one exact immutable WAL image without retaining page bodies."""
    if type(image) is not bytes:
        raise TypeError("image must be exact bytes")
    if not _WAL_HEADER_BYTES <= len(image) <= _MAX_WAL_BYTES:
        raise SQLiteWalError("image length is outside the bounded WAL profile")

    magic = _unsigned_32_be(image, 0)
    if magic == _WAL_MAGIC_LITTLE_CHECKSUM:
        checksum_byte_order = SQLiteWalChecksumByteOrder.LITTLE
    elif magic == _WAL_MAGIC_BIG_CHECKSUM:
        checksum_byte_order = SQLiteWalChecksumByteOrder.BIG
    else:
        raise SQLiteWalError("WAL magic is invalid")

    version = _unsigned_32_be(image, 4)
    if version != _WAL_VERSION:
        raise SQLiteWalError("WAL format version is unsupported")

    page_size = _unsigned_32_be(image, 8)
    if not _MIN_PAGE_SIZE <= page_size <= _MAX_PAGE_SIZE or page_size & (page_size - 1) != 0:
        raise SQLiteWalError("WAL page size must be a supported power of two")

    view = memoryview(image)
    checksum_state = _extend_wal_checksum(
        view,
        0,
        24,
        checksum_byte_order,
        (0, 0),
    )
    stored_header_checksum = (
        _unsigned_32_be(image, 24),
        _unsigned_32_be(image, 28),
    )
    if checksum_state != stored_header_checksum:
        raise SQLiteWalError("WAL header checksum is invalid")

    frame_size = _FRAME_HEADER_BYTES + page_size
    complete_slot_count, partial_tail_length = divmod(
        len(image) - _WAL_HEADER_BYTES,
        frame_size,
    )
    if complete_slot_count > _MAX_FRAMES:
        raise SQLiteWalError("WAL contains more than 256 complete frame slots")

    header = SQLiteWalHeader(
        magic=magic,
        version=version,
        page_size=page_size,
        checkpoint_sequence=_unsigned_32_be(image, 12),
        salt_1=_unsigned_32_be(image, 16),
        salt_2=_unsigned_32_be(image, 20),
        checksum_1=stored_header_checksum[0],
        checksum_2=stored_header_checksum[1],
        checksum_byte_order=checksum_byte_order,
    )
    header_salt = image[16:24]
    valid_frames: list[SQLiteWalFrame] = []
    last_committed_frame_number: int | None = None
    last_committed_database_pages: int | None = None
    first_invalid_frame_number: int | None = None
    stop_reason = (
        SQLiteWalStopReason.PARTIAL_TAIL
        if partial_tail_length
        else SQLiteWalStopReason.END_OF_IMAGE
    )

    for frame_number in range(1, complete_slot_count + 1):
        offset = _WAL_HEADER_BYTES + (frame_number - 1) * frame_size
        if image[offset + 8 : offset + 16] != header_salt:
            stop_reason = SQLiteWalStopReason.SALT_MISMATCH
            first_invalid_frame_number = frame_number
            break

        page_number = _unsigned_32_be(image, offset)
        if not 1 <= page_number <= _MAX_PAGE_NUMBER:
            stop_reason = SQLiteWalStopReason.INVALID_PAGE_NUMBER
            first_invalid_frame_number = frame_number
            break

        database_size_pages = _unsigned_32_be(image, offset + 4)
        if database_size_pages > _MAX_PAGE_NUMBER:
            stop_reason = SQLiteWalStopReason.INVALID_DATABASE_SIZE
            first_invalid_frame_number = frame_number
            break

        candidate_state = _extend_wal_checksum(
            view,
            offset,
            offset + 8,
            checksum_byte_order,
            checksum_state,
        )
        candidate_state = _extend_wal_checksum(
            view,
            offset + _FRAME_HEADER_BYTES,
            offset + frame_size,
            checksum_byte_order,
            candidate_state,
        )
        stored_frame_checksum = (
            _unsigned_32_be(image, offset + 16),
            _unsigned_32_be(image, offset + 20),
        )
        if candidate_state != stored_frame_checksum:
            stop_reason = SQLiteWalStopReason.CHECKSUM_MISMATCH
            first_invalid_frame_number = frame_number
            break

        checksum_state = candidate_state
        valid_frames.append(
            SQLiteWalFrame(
                frame_number=frame_number,
                offset=offset,
                page_number=page_number,
                database_size_pages=database_size_pages,
            )
        )
        if database_size_pages != 0:
            last_committed_frame_number = frame_number
            last_committed_database_pages = database_size_pages

    valid_frame_count = len(valid_frames)
    committed_frame_count = last_committed_frame_number or 0
    partial_tail_offset = _WAL_HEADER_BYTES + complete_slot_count * frame_size
    return SQLiteWalInspection(
        header=header,
        valid_frames=tuple(valid_frames),
        checksum_valid_prefix_length=(_WAL_HEADER_BYTES + valid_frame_count * frame_size),
        committed_prefix_length=(_WAL_HEADER_BYTES + committed_frame_count * frame_size),
        valid_frame_count=valid_frame_count,
        last_committed_frame_number=last_committed_frame_number,
        last_committed_database_pages=last_committed_database_pages,
        valid_uncommitted_frame_count=valid_frame_count - committed_frame_count,
        stop_reason=stop_reason,
        first_invalid_frame_number=first_invalid_frame_number,
        nonprefix_complete_slot_count=complete_slot_count - valid_frame_count,
        partial_tail_offset=partial_tail_offset,
        partial_tail_length=partial_tail_length,
    )
```

## Example

```python


def _write_unsigned_32_be(buffer: bytearray, offset: int, value: int) -> None:
    buffer[offset : offset + 4] = value.to_bytes(4, "big")


def _build_wal_image(
    byte_order: SQLiteWalChecksumByteOrder,
    frames: tuple[tuple[int, int, bytes], ...],
    page_size: int = 512,
) -> bytes:
    magic = (
        _WAL_MAGIC_LITTLE_CHECKSUM
        if byte_order is SQLiteWalChecksumByteOrder.LITTLE
        else _WAL_MAGIC_BIG_CHECKSUM
    )
    header = bytearray(_WAL_HEADER_BYTES)
    for offset, value in (
        (0, magic),
        (4, _WAL_VERSION),
        (8, page_size),
        (12, 7),
        (16, 0x12345678),
        (20, 0x9ABCDEF0),
    ):
        _write_unsigned_32_be(header, offset, value)

    checksum_state = _extend_wal_checksum(
        memoryview(header),
        0,
        24,
        byte_order,
        (0, 0),
    )
    _write_unsigned_32_be(header, 24, checksum_state[0])
    _write_unsigned_32_be(header, 28, checksum_state[1])
    image = bytearray(header)

    for page_number, database_size_pages, page in frames:
        if len(page) != page_size:
            raise ValueError("each page must match the WAL page size")
        frame = bytearray(_FRAME_HEADER_BYTES + page_size)
        _write_unsigned_32_be(frame, 0, page_number)
        _write_unsigned_32_be(frame, 4, database_size_pages)
        frame[8:16] = header[16:24]
        frame[_FRAME_HEADER_BYTES:] = page
        frame_view = memoryview(frame)
        candidate_state = _extend_wal_checksum(
            frame_view,
            0,
            8,
            byte_order,
            checksum_state,
        )
        candidate_state = _extend_wal_checksum(
            frame_view,
            _FRAME_HEADER_BYTES,
            len(frame),
            byte_order,
            candidate_state,
        )
        _write_unsigned_32_be(frame, 16, candidate_state[0])
        _write_unsigned_32_be(frame, 20, candidate_state[1])
        image.extend(frame)
        checksum_state = candidate_state

    return bytes(image)


frames = (
    (1, 0, bytes(512)),
    (2, 3, b"A" * 512),
    (3, 0, b"B" * 512),
)
little_image = _build_wal_image(SQLiteWalChecksumByteOrder.LITTLE, frames)
big_image = _build_wal_image(SQLiteWalChecksumByteOrder.BIG, frames)

little = inspect_sqlite_wal_image(little_image)
big = inspect_sqlite_wal_image(big_image)
assert little.valid_frame_count == big.valid_frame_count == 3
assert little.last_committed_frame_number == 2
assert little.last_committed_database_pages == 3
assert little.valid_uncommitted_frame_count == 1

frame_size = _FRAME_HEADER_BYTES + 512
corrupted = bytearray(little_image)
corrupted[_WAL_HEADER_BYTES + 2 * frame_size + _FRAME_HEADER_BYTES] ^= 1
stopped = inspect_sqlite_wal_image(bytes(corrupted))
assert stopped.stop_reason is SQLiteWalStopReason.CHECKSUM_MISMATCH
assert stopped.valid_frame_count == 2
assert stopped.first_invalid_frame_number == 3
assert stopped.committed_prefix_length == _WAL_HEADER_BYTES + 2 * frame_size

partial = inspect_sqlite_wal_image(little_image[:-5])
assert partial.stop_reason is SQLiteWalStopReason.PARTIAL_TAIL
assert partial.valid_frame_count == 2
assert partial.partial_tail_length == frame_size - 5
```

## Trade-offs and Limitations

The parser scans only the consecutive valid prefix and returns at most 256
small immutable frame records. The input is capped at 16,783,392 bytes, the
size of a 32-byte header plus 256 maximum-size frames. It validates both
checksum byte orders, modulo-32-bit wraparound, salt equality, page-number and
commit-size ranges, but the checksum is an accidental-corruption check rather
than authentication.

A partial tail is reported separately from complete slots. Once any complete
frame is invalid, all complete slots from that frame onward are non-prefix
data and are never reconsidered. A nonzero database-size field marks a commit,
so later checksum-valid frames can remain uncommitted. The result does not
retain page bodies or tail bytes and does not repair, truncate, recover, or
interpret a database.

The field layout and checksum recurrence follow the
[SQLite WAL file-format documentation](https://sqlite.org/fileformat.html#the_write_ahead_log).
This bounded inspection does not validate page contents, transaction meaning,
the WAL-index, or consistency with a main database, and it must not be applied
to a live changing file.

## Related Snippets

<!-- catalog:related:start -->
- [Inspect a Bounded SQLite Database Header from In-Memory Bytes](inspect-a-bounded-sqlite-database-header-from-in-memory-bytes.md)
- [Recover a Verified Prefix from a Bounded CRC-Framed Byte Log](recover-a-verified-prefix-from-a-bounded-crc-framed-byte-log.md)
- [Open a Verified Read-Only SQLite Connection Under a Closed Hardening Profile](open-a-verified-read-only-sqlite-connection-under-a-closed-hardening-profile.md)
<!-- catalog:related:end -->
