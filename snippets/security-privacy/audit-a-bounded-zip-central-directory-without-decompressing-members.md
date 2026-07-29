---
title: "Audit a Bounded ZIP Central Directory Without Decompressing Members"
snippet_type: recipe
use_cases:
  - parsing
  - resource-management
  - security
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - audit-bounded-relative-posix-archive-member-names-under-a-closed-policy.md
  - decompress-exactly-one-zlib-stream-under-input-and-output-limits.md
  - ../configuration-serialization/build-a-deterministic-size-capped-ustar-archive-from-bytes.md
---

# Audit a Bounded ZIP Central Directory Without Decompressing Members

## Idea and Problem

Audit the decoded central-directory metadata of one small in-memory ZIP while never opening, reading, testing, extracting, or decompressing a member.

The audit applies a closed name, kind, compression, declared-size, ratio, tree,
and local-header-offset policy to the records returned by
`ZipFile.infolist()`. It returns sorted immutable metadata only; payloads and
their CRC values remain unverified.

## When to Use

Use this as an early format-aware filter when a workflow must inventory a
captured ZIP before a separate stage decides whether and how to process it.
Choose smaller limits when the expected package shape allows them, and keep
the returned names private when archive metadata may itself be sensitive.

Do not treat success as permission to extract. A trusted extraction design
still needs destination isolation, race-resistant filesystem operations,
actual decompression limits, content checks, and a format-specific extraction
policy.

## Implementation

```python
import re
import stat
from dataclasses import dataclass
from enum import StrEnum
from io import BytesIO
from zipfile import (
    ZIP_DEFLATED,
    ZIP_STORED,
    BadZipFile,
    LargeZipFile,
    ZipFile,
    ZipInfo,
)

_MAX_ARCHIVE_BYTES = 65_536
_MAX_ENTRIES = 128
_MAX_NAME_BYTES = 256
_MAX_TOTAL_NAME_BYTES = 8_192
_MAX_COMPONENTS = 32
_MAX_COMPONENT_BYTES = 64
_MAX_FILE_BYTES = 1_048_576
_MAX_TOTAL_FILE_BYTES = 4_194_304
_MAX_EXPANSION_RATIO = 100
_LOCAL_HEADER_BYTES = 30
_COMPONENT = re.compile(r"[A-Za-z0-9._-]+", re.ASCII)
_ALLOWED_COMPRESSION = frozenset({ZIP_STORED, ZIP_DEFLATED})


class ZipAuditError(ValueError):
    """Raised when ZIP metadata violates the closed audit profile."""


class ZipEntryKind(StrEnum):
    FILE = "file"
    DIRECTORY = "directory"


@dataclass(frozen=True, slots=True)
class AuditedZipEntry:
    name: str
    kind: ZipEntryKind
    compression: int
    compressed_size: int
    declared_size: int
    crc: int


def _logical_zip_name(
    name: str,
) -> tuple[str, tuple[str, ...], ZipEntryKind]:
    if (
        not name
        or not name.isascii()
        or len(name) > _MAX_NAME_BYTES
    ):
        raise ZipAuditError(
            "an entry name violates the bounded ASCII profile"
        )
    if name.startswith("/") or "\\" in name:
        raise ZipAuditError(
            "an entry name is not a relative POSIX name"
        )

    is_directory = name.endswith("/")
    logical_name = name[:-1] if is_directory else name
    parts = tuple(logical_name.split("/"))
    if (
        not logical_name
        or len(parts) > _MAX_COMPONENTS
        or any(
            not part
            or part in {".", ".."}
            or len(part) > _MAX_COMPONENT_BYTES
            or _COMPONENT.fullmatch(part) is None
            for part in parts
        )
    ):
        raise ZipAuditError(
            "an entry name has an invalid path component"
        )
    kind = (
        ZipEntryKind.DIRECTORY
        if is_directory
        else ZipEntryKind.FILE
    )
    return logical_name, parts, kind


def _validate_zip_kind(
    info: ZipInfo,
    *,
    kind: ZipEntryKind,
) -> None:
    is_directory = kind is ZipEntryKind.DIRECTORY
    dos_directory = bool(info.external_attr & 0x10)
    if dos_directory and not is_directory:
        raise ZipAuditError(
            "directory metadata contradicts the entry name"
        )

    if info.create_system != 3:
        return
    unix_kind = stat.S_IFMT((info.external_attr >> 16) & 0xFFFF)
    if unix_kind not in {0, stat.S_IFREG, stat.S_IFDIR}:
        raise ZipAuditError(
            "a Unix entry is not a regular file or directory"
        )
    if unix_kind == stat.S_IFREG and is_directory:
        raise ZipAuditError(
            "Unix file metadata contradicts the entry name"
        )
    if unix_kind == stat.S_IFDIR and not is_directory:
        raise ZipAuditError(
            "Unix directory metadata contradicts the entry name"
        )


def audit_zip_central_directory(
    data: bytes,
) -> tuple[AuditedZipEntry, ...]:
    """Audit bounded declared ZIP metadata without reading member payloads."""
    if type(data) is not bytes:
        raise TypeError("data must be exact bytes")
    if not 1 <= len(data) <= _MAX_ARCHIVE_BYTES:
        raise ZipAuditError(
            "data size is outside the supported range"
        )

    try:
        with ZipFile(BytesIO(data), "r") as archive:
            infos = archive.infolist()
    except (
        BadZipFile,
        LargeZipFile,
        EOFError,
        NotImplementedError,
        ValueError,
    ):
        # UnicodeDecodeError is covered by ValueError.
        raise ZipAuditError(
            "data has no supported ZIP central directory"
        ) from None
    if len(infos) > _MAX_ENTRIES:
        raise ZipAuditError(
            "entry count exceeds the supported limit"
        )

    records: list[AuditedZipEntry] = []
    identities: dict[tuple[str, ...], ZipEntryKind] = {}
    offsets: set[int] = set()
    total_name_bytes = 0
    total_compressed_bytes = 0
    total_declared_bytes = 0

    for info in infos:
        logical_name, parts, kind = _logical_zip_name(info.filename)
        total_name_bytes += len(info.filename)
        if total_name_bytes > _MAX_TOTAL_NAME_BYTES:
            raise ZipAuditError(
                "entry names exceed the aggregate byte limit"
            )
        if parts in identities:
            raise ZipAuditError(
                "decoded logical entry names must be unique"
            )
        identities[parts] = kind

        _validate_zip_kind(info, kind=kind)
        if info.flag_bits & 0x01:
            raise ZipAuditError("encrypted entries are not supported")
        if info.compress_type not in _ALLOWED_COMPRESSION:
            raise ZipAuditError(
                "an entry uses an unsupported compression method"
            )

        offset = info.header_offset
        if (
            offset in offsets
            or not 0 <= offset <= len(data) - _LOCAL_HEADER_BYTES
        ):
            raise ZipAuditError(
                "local-header offsets are invalid or reused"
            )
        offsets.add(offset)

        compressed_size = info.compress_size
        declared_size = info.file_size
        if compressed_size < 0 or declared_size < 0:
            raise ZipAuditError("entry sizes must be nonnegative")

        if kind is ZipEntryKind.DIRECTORY:
            if compressed_size or declared_size:
                raise ZipAuditError(
                    "directory entries must have zero sizes"
                )
        else:
            if declared_size > _MAX_FILE_BYTES:
                raise ZipAuditError(
                    "an entry exceeds the declared-size limit"
                )
            if (
                info.compress_type == ZIP_STORED
                and compressed_size != declared_size
            ):
                raise ZipAuditError(
                    "a stored entry has inconsistent sizes"
                )
            if declared_size and compressed_size == 0:
                raise ZipAuditError(
                    "a nonempty entry has zero compressed size"
                )
            if (
                compressed_size
                and declared_size
                > compressed_size * _MAX_EXPANSION_RATIO
            ):
                raise ZipAuditError(
                    "an entry exceeds the declared expansion-ratio limit"
                )

        total_compressed_bytes += compressed_size
        total_declared_bytes += declared_size
        if total_compressed_bytes > len(data):
            raise ZipAuditError(
                "declared compressed bytes exceed archive size"
            )
        if total_declared_bytes > _MAX_TOTAL_FILE_BYTES:
            raise ZipAuditError(
                "entries exceed the aggregate declared-size limit"
            )

        records.append(
            AuditedZipEntry(
                name=logical_name,
                kind=kind,
                compression=info.compress_type,
                compressed_size=compressed_size,
                declared_size=declared_size,
                crc=info.CRC,
            )
        )

    for parts in identities:
        for length in range(1, len(parts)):
            if (
                identities.get(parts[:length])
                is ZipEntryKind.FILE
            ):
                raise ZipAuditError(
                    "a file entry cannot contain descendants"
                )
    return tuple(sorted(records, key=lambda record: record.name))
```

## Example

```python
def zip_info(
    name: str,
    *,
    directory: bool = False,
    compression: int = ZIP_STORED,
) -> ZipInfo:
    info = ZipInfo(name)
    info.create_system = 3
    mode = (
        stat.S_IFDIR | 0o755
        if directory
        else stat.S_IFREG | 0o644
    )
    info.external_attr = mode << 16
    if directory:
        info.external_attr |= 0x10
    info.compress_type = compression
    return info


def build_zip(
    entries: tuple[tuple[ZipInfo, bytes], ...],
) -> bytes:
    output = BytesIO()
    with ZipFile(output, "w") as archive:
        for info, payload in entries:
            archive.writestr(info, payload)
    return output.getvalue()


payload = build_zip(
    (
        (zip_info("docs/", directory=True), b""),
        (
            zip_info(
                "docs/readme.txt",
                compression=ZIP_DEFLATED,
            ),
            b"bounded metadata\n",
        ),
        (zip_info("data.bin"), b"1234"),
    )
)
records = audit_zip_central_directory(payload)

symlink = zip_info("link")
symlink.external_attr = (stat.S_IFLNK | 0o777) << 16
invalid_archives = (
    build_zip(
        (
            (zip_info("parent"), b""),
            (zip_info("parent/child.txt"), b"x"),
        )
    ),
    build_zip(((symlink, b"target"),)),
    build_zip(
        (
            (
                zip_info(
                    "dense.txt",
                    compression=ZIP_DEFLATED,
                ),
                b"A" * 20_000,
            ),
        )
    ),
    payload + b"x" * 65_536,
)

rejected = 0
for invalid in invalid_archives:
    try:
        audit_zip_central_directory(invalid)
    except ZipAuditError:
        rejected += 1

assert (
    tuple(
        (record.name, record.kind, record.declared_size)
        for record in records
    )
    == (
        ("data.bin", ZipEntryKind.FILE, 4),
        ("docs", ZipEntryKind.DIRECTORY, 0),
        ("docs/readme.txt", ZipEntryKind.FILE, 17),
    )
    and all(
        record.compression in {ZIP_STORED, ZIP_DEFLATED}
        for record in records
    )
    and rejected == 4
)
```

## Trade-offs and Limitations

Let `B` be the captured bytes and `E` the central entries. Parsing,
validation, name inspection, and output sorting use
`O(B + E log E + name bytes)` time and `O(B + E)` memory under the fixed
caps. `ZipFile` materializes central records before the entry-count check;
the 64-KiB input limit bounds that unavoidable work but is not a proactive
object-count guard.

Only stored and deflated entries are accepted. Empty compressed files may have
a nonzero representation and therefore have ratio zero; nonempty files with a
zero compressed size are rejected. Unix entries with an explicit type must be
regular files or directories. Other creator systems do not provide enough
portable metadata to prove that the producer did not intend a link or special
entry, so their logical kind follows the directory-name convention.

The documented `ZipInfo.filename` is already decoded and can hide details of
the raw central spelling, including truncation at a NUL. Central sizes, methods,
offsets, CRC values, and names can also lie or disagree with local headers.
The audit never reads payloads, verifies CRCs, measures actual expansion, or
prevents every archive bomb. Its conservative path checks complement, rather
than replace, a destination-aware extraction policy.

## Related Snippets

<!-- catalog:related:start -->
- [Audit Bounded Relative POSIX Archive Member Names Under a Closed Policy](audit-bounded-relative-posix-archive-member-names-under-a-closed-policy.md)
- [Decompress Exactly One Zlib Stream Under Input and Output Limits](decompress-exactly-one-zlib-stream-under-input-and-output-limits.md)
- [Build a Deterministic Size-Capped USTAR Archive from Bytes](../configuration-serialization/build-a-deterministic-size-capped-ustar-archive-from-bytes.md)
<!-- catalog:related:end -->
