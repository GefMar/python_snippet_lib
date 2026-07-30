---
title: "Decode a Bounded Uncompressed TAR into Regular-File Byte Records"
snippet_type: recipe
use_cases:
  - data-transformation
  - interoperability
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - build-a-deterministic-size-capped-ustar-archive-from-bytes.md
  - ../security-privacy/audit-bounded-relative-posix-archive-member-names-under-a-closed-policy.md
  - ../security-privacy/audit-a-bounded-zip-central-directory-without-decompressing-members.md
---

# Decode a Bounded Uncompressed TAR into Regular-File Byte Records

## Idea and Problem

Read a small closed-profile TAR archive into immutable name-and-bytes records without creating filesystem paths.

The TAR container supports links, devices, sparse files, multiple extension
formats, repeated names, and compressed variants. This decoder admits none of
those features. It requires USTAR-marked regular-file headers without base-256
numeric extensions, conservative relative ASCII names, explicit member and
payload budgets, exact payload reads, and a zero-only end marker.

Keeping the result in memory removes extraction-path, symlink, permission, and
partial-filesystem-state concerns. Sorting the frozen records by name makes the
result independent of archive member order.

## When to Use

Use this decoder for a small data-only interchange bundle whose complete
contents must fit memory and whose producer can emit the narrow uncompressed
USTAR profile. It is useful before validating a closed set of configuration,
fixture, or artifact bytes without preserving archive metadata.

Use a mature archive workflow when directories, links, permissions, sparse
files, PAX/GNU extensions, compression, or large streaming payloads are
required. Parsing limits reduce accidental resource use but do not authenticate
the archive or turn Python's TAR parser into a hostile-format sandbox.

## Implementation

```python
import re
import tarfile
from dataclasses import dataclass
from io import BytesIO

_TAR_BLOCK_SIZE = 512
_TAR_END_SIZE = 2 * _TAR_BLOCK_SIZE
_MAX_TAR_ARCHIVE_BYTES = 16 * 1_024 * 1_024
_MAX_TAR_MEMBERS = 64
_MAX_TAR_NAME_BYTES = 256
_MAX_TAR_NAME_DEPTH = 8
_MAX_TAR_MEMBER_BYTES = 4 * 1_024 * 1_024
_MAX_TAR_PAYLOAD_BYTES = 8 * 1_024 * 1_024
_USTAR_MAGIC = b"ustar\0"
_USTAR_VERSION = b"00"
_REGULAR_TYPE_FLAGS = (b"\0", b"0")
_USTAR_OCTAL_FIELDS = (
    (100, 108, False),
    (108, 116, False),
    (116, 124, False),
    (124, 136, False),
    (136, 148, False),
    (329, 337, True),
    (337, 345, True),
)
_SAFE_TAR_SEGMENT = re.compile(r"[A-Za-z0-9._-]+\Z")


@dataclass(frozen=True, slots=True)
class TarByteRecord:
    name: str
    data: bytes


def _is_canonical_ustar_octal(field: bytes, *, allow_empty: bool) -> bool:
    if allow_empty and not any(field):
        return True
    return field.endswith(b"\0") and all(48 <= byte <= 55 for byte in field[:-1])


def _has_canonical_ustar_numbers(header: bytes) -> bool:
    if not all(
        _is_canonical_ustar_octal(
            header[start:end],
            allow_empty=allow_empty,
        )
        for start, end, allow_empty in _USTAR_OCTAL_FIELDS
    ):
        return False
    checksum = header[148:156]
    return checksum[6:] == b"\0 " and all(48 <= byte <= 55 for byte in checksum[:6])


def _validated_tar_name(name: object, *, member_index: int) -> str:
    if type(name) is not str:
        raise TypeError(f"member {member_index} name must be an exact string")
    try:
        encoded = name.encode("ascii", errors="strict")
    except UnicodeEncodeError as error:
        raise ValueError(f"member {member_index} name must be ASCII") from error
    if not 1 <= len(encoded) <= _MAX_TAR_NAME_BYTES:
        raise ValueError(f"member {member_index} name has an invalid byte size")
    if name.startswith("/") or name.endswith("/") or "\\" in name:
        raise ValueError(f"member {member_index} name is not a relative POSIX file")
    segments = name.split("/")
    if not 1 <= len(segments) <= _MAX_TAR_NAME_DEPTH:
        raise ValueError(f"member {member_index} name is too deep")
    if any(
        segment in ("", ".", "..") or _SAFE_TAR_SEGMENT.fullmatch(segment) is None
        for segment in segments
    ):
        raise ValueError(f"member {member_index} name has an unsafe segment")
    return name


def decode_uncompressed_tar_records(
    archive_bytes: bytes,
) -> tuple[TarByteRecord, ...]:
    """Return sorted regular-file bytes from one closed-profile USTAR archive."""
    if type(archive_bytes) is not bytes:
        raise TypeError("archive_bytes must be exact bytes")
    if not _TAR_END_SIZE <= len(archive_bytes) <= _MAX_TAR_ARCHIVE_BYTES:
        raise ValueError("archive byte length is outside the supported range")
    if len(archive_bytes) % _TAR_BLOCK_SIZE != 0:
        raise ValueError("archive length must be a multiple of 512 bytes")

    records: list[TarByteRecord] = []
    seen_names: set[str] = set()
    payload_total = 0
    payload_end = 0

    try:
        with tarfile.open(fileobj=BytesIO(archive_bytes), mode="r:") as archive:
            for member_index, member in enumerate(archive):
                if member_index >= _MAX_TAR_MEMBERS:
                    raise ValueError("archive contains more than 64 members")

                header = archive_bytes[
                    member.offset : member.offset + _TAR_BLOCK_SIZE
                ]
                if len(header) != _TAR_BLOCK_SIZE:
                    raise ValueError(f"member {member_index} header is truncated")
                if (
                    header[257:263] != _USTAR_MAGIC
                    or header[263:265] != _USTAR_VERSION
                    or header[156:157] not in _REGULAR_TYPE_FLAGS
                    or not _has_canonical_ustar_numbers(header)
                ):
                    raise ValueError(
                        f"member {member_index} is not a supported USTAR regular file"
                    )
                if (
                    not member.isreg()
                    or member.sparse is not None
                    or member.pax_headers
                ):
                    raise ValueError(f"member {member_index} uses an unsupported type")

                name = _validated_tar_name(member.name, member_index=member_index)
                if name in seen_names:
                    raise ValueError(f"member {member_index} duplicates {name!r}")
                seen_names.add(name)

                if not 0 <= member.size <= _MAX_TAR_MEMBER_BYTES:
                    raise ValueError(f"member {member_index} payload is too large")
                payload_total += member.size
                if payload_total > _MAX_TAR_PAYLOAD_BYTES:
                    raise ValueError("aggregate TAR payload exceeds the supported limit")

                extracted = archive.extractfile(member)
                if extracted is None:
                    raise ValueError(f"member {member_index} has no readable payload")
                data = extracted.read(member.size + 1)
                if len(data) != member.size:
                    raise ValueError(f"member {member_index} payload is truncated")
                records.append(TarByteRecord(name=name, data=data))

                padded_size = (
                    (member.size + _TAR_BLOCK_SIZE - 1) // _TAR_BLOCK_SIZE
                ) * _TAR_BLOCK_SIZE
                payload_end = max(payload_end, member.offset_data + padded_size)
    except (tarfile.TarError, EOFError, OSError) as error:
        raise ValueError("archive is not a valid uncompressed TAR") from error

    trailer = archive_bytes[payload_end:]
    if len(trailer) < _TAR_END_SIZE or any(trailer):
        raise ValueError("archive must end with at least two zero blocks only")
    return tuple(sorted(records, key=lambda record: record.name))
```

## Example

```python
def build_ustar(
    entries: tuple[tuple[str, bytes, bytes], ...],
    *,
    archive_format: int = tarfile.USTAR_FORMAT,
) -> bytes:
    output = BytesIO()
    with tarfile.open(fileobj=output, mode="w:", format=archive_format) as archive:
        for name, data, member_type in entries:
            member = tarfile.TarInfo(name)
            member.type = member_type
            member.size = len(data) if member_type in _REGULAR_TYPE_FLAGS else 0
            member.mode = 0o600
            member.mtime = 0
            if member_type == tarfile.SYMTYPE:
                member.linkname = "target"
            archive.addfile(
                member,
                BytesIO(data) if member.isreg() and data else None,
            )
    return output.getvalue()


def raises_value(operation) -> bool:
    try:
        operation()
    except ValueError:
        return True
    return False


def refresh_first_header_checksum(archive_bytes: bytearray) -> None:
    archive_bytes[148:156] = b"        "
    checksum = sum(archive_bytes[:_TAR_BLOCK_SIZE])
    archive_bytes[148:156] = f"{checksum:06o}\0 ".encode("ascii")


valid = build_ustar(
    (
        ("zeta.bin", b"\x00\x01", tarfile.REGTYPE),
        ("config/app.toml", b"enabled = true\n", tarfile.REGTYPE),
        ("empty", b"", tarfile.REGTYPE),
    )
)
decoded = decode_uncompressed_tar_records(valid)

traversal = build_ustar((("../outside", b"bad", tarfile.REGTYPE),))
absolute = build_ustar((("/outside", b"bad", tarfile.REGTYPE),))
backslash = build_ustar((("folder\\file", b"bad", tarfile.REGTYPE),))
dot_segment = build_ustar((("folder/./file", b"bad", tarfile.REGTYPE),))
duplicate = build_ustar(
    (
        ("same", b"first", tarfile.REGTYPE),
        ("same", b"second", tarfile.REGTYPE),
    )
)
directory = build_ustar((("folder", b"", tarfile.DIRTYPE),))
symlink = build_ustar((("shortcut", b"", tarfile.SYMTYPE),))
device = build_ustar((("device", b"", tarfile.CHRTYPE),))
sparse = build_ustar((("sparse", b"", tarfile.GNUTYPE_SPARSE),))
pax = build_ustar(
    (("x" * 150, b"extended", tarfile.REGTYPE),),
    archive_format=tarfile.PAX_FORMAT,
)
gnu = build_ustar(
    (("gnu", b"extended", tarfile.REGTYPE),),
    archive_format=tarfile.GNU_FORMAT,
)
bad_checksum = bytearray(valid)
bad_checksum[0] ^= 1
base_256_size = bytearray(
    build_ustar((("base256", b"abc", tarfile.REGTYPE),))
)
base_256_size[124:136] = bytes([0x80]) + (3).to_bytes(11, "big")
refresh_first_header_checksum(base_256_size)
oversize_declaration = bytearray(
    build_ustar((("oversize", b"", tarfile.REGTYPE),))
)
oversize_declaration[124:136] = f"{_MAX_TAR_MEMBER_BYTES + 1:011o}\0".encode(
    "ascii"
)
refresh_first_header_checksum(oversize_declaration)

shared_payload = b"x" * (3 * 1_024 * 1_024)
aggregate_oversize = build_ustar(
    tuple(
        (f"part-{index}", shared_payload, tarfile.REGTYPE)
        for index in range(3)
    )
)
too_many_members = build_ustar(
    tuple(
        (f"member-{index}", b"", tarfile.REGTYPE)
        for index in range(_MAX_TAR_MEMBERS + 1)
    )
)

compressed_output = BytesIO()
with tarfile.open(
    fileobj=compressed_output,
    mode="w:gz",
    format=tarfile.USTAR_FORMAT,
) as compressed_archive:
    compressed_member = tarfile.TarInfo("compressed")
    compressed_member.size = 3
    compressed_archive.addfile(compressed_member, BytesIO(b"abc"))
compressed_raw = compressed_output.getvalue()
compressed_size = max(
    _TAR_END_SIZE,
    ((len(compressed_raw) + _TAR_BLOCK_SIZE - 1) // _TAR_BLOCK_SIZE)
    * _TAR_BLOCK_SIZE,
)
compressed = compressed_raw + b"\0" * (compressed_size - len(compressed_raw))
nonzero_trailer = bytearray(valid)
nonzero_trailer[-1] = 1

single = build_ustar((("one", b"abc", tarfile.REGTYPE),))
one_payload_end = 2 * _TAR_BLOCK_SIZE
truncated_trailer = single[: one_payload_end + _TAR_BLOCK_SIZE]

assert (
    decoded
    == (
        TarByteRecord("config/app.toml", b"enabled = true\n"),
        TarByteRecord("empty", b""),
        TarByteRecord("zeta.bin", b"\x00\x01"),
    )
    and decode_uncompressed_tar_records(b"\0" * 1_024) == ()
    and raises_value(lambda: decode_uncompressed_tar_records(traversal))
    and raises_value(lambda: decode_uncompressed_tar_records(absolute))
    and raises_value(lambda: decode_uncompressed_tar_records(backslash))
    and raises_value(lambda: decode_uncompressed_tar_records(dot_segment))
    and raises_value(lambda: decode_uncompressed_tar_records(duplicate))
    and raises_value(lambda: decode_uncompressed_tar_records(directory))
    and raises_value(lambda: decode_uncompressed_tar_records(symlink))
    and raises_value(lambda: decode_uncompressed_tar_records(device))
    and raises_value(lambda: decode_uncompressed_tar_records(sparse))
    and raises_value(lambda: decode_uncompressed_tar_records(pax))
    and raises_value(lambda: decode_uncompressed_tar_records(gnu))
    and raises_value(lambda: decode_uncompressed_tar_records(bytes(bad_checksum)))
    and raises_value(lambda: decode_uncompressed_tar_records(bytes(base_256_size)))
    and raises_value(
        lambda: decode_uncompressed_tar_records(bytes(oversize_declaration))
    )
    and raises_value(lambda: decode_uncompressed_tar_records(aggregate_oversize))
    and raises_value(lambda: decode_uncompressed_tar_records(too_many_members))
    and raises_value(lambda: decode_uncompressed_tar_records(compressed))
    and raises_value(
        lambda: decode_uncompressed_tar_records(bytes(nonzero_trailer))
    )
    and raises_value(lambda: decode_uncompressed_tar_records(truncated_trailer))
    and raises_value(lambda: decode_uncompressed_tar_records(valid + single))
)
```

## Trade-offs and Limitations

Reading headers and payloads costs `O(input bytes + payload bytes)`, sorting
`N` records costs `O(N log N)`, and the returned data uses `O(payload bytes)`
memory. The exact input is capped at 16 MiB, with at most 64 members, 4 MiB per
file, and 8 MiB of aggregate payload.

Only supported uncompressed USTAR regular files with conservative relative
ASCII names are admitted. Directories, links, devices, sparse files, duplicate
names, PAX/GNU extensions (including base-256 numeric fields), compression,
concatenation, and non-zero trailing data fail rather than being normalized.
Archive modes, owners, timestamps, and other metadata are discarded.

The function never touches the filesystem, but it still invokes Python's TAR
parser on bounded bytes. The limits are not authenticity, malware scanning,
format-parser isolation, or a security sandbox. Use a separately isolated and
audited archive service for genuinely hostile inputs.

## Related Snippets

<!-- catalog:related:start -->
- [Build a Deterministic Size-Capped USTAR Archive from Bytes](build-a-deterministic-size-capped-ustar-archive-from-bytes.md)
- [Audit Bounded Relative POSIX Archive Member Names Under a Closed Policy](../security-privacy/audit-bounded-relative-posix-archive-member-names-under-a-closed-policy.md)
- [Audit a Bounded ZIP Central Directory Without Decompressing Members](../security-privacy/audit-a-bounded-zip-central-directory-without-decompressing-members.md)
<!-- catalog:related:end -->
