---
title: "Parse a Size-Capped systemd Journal Export Stream into Immutable Entries"
snippet_type: recipe
use_cases:
  - observability
  - parsing
  - serialization
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - read-a-bounded-log-delta-across-one-rename-based-rotation.md
  - ../storage-databases/recover-a-verified-prefix-from-a-bounded-crc-framed-byte-log.md
  - ../data-processing/decode-bounded-fixed-width-ascii-records-across-arbitrary-byte-chunks.md
---

# Parse a Size-Capped systemd Journal Export Stream into Immutable Entries

## Idea and Problem

Parse one complete bounded systemd Journal Export byte stream while preserving field order, duplicate names, binary values, and transport metadata.

Journal Export uses readable `NAME=VALUE` lines for valid non-control UTF-8
values and length-framed records for other bytes. A blank LF line closes each
entry. Parsing the complete snapshot before returning an immutable result keeps
truncated framing or resource-limit failures from exposing a partial prefix.

## When to Use

Use this parser after a trusted acquisition boundary has captured one complete
`journalctl --output=export` stream of at most one MiB. It fits deterministic
fixtures, small diagnostic transfers, and adapters that must retain repeated
fields or distinguish their original text and binary encodings.

This is the Journal Export transfer format, not the native journal file format.
Use systemd's supported readers when files must be discovered, followed,
filtered, verified, or imported. Treat all returned values as untrusted bytes;
field names and double-underscore metadata do not make their contents genuine.

## Implementation

```python
import re
from dataclasses import dataclass
from enum import StrEnum

_MAX_STREAM_BYTES = 1 * 1024 * 1024
_MAX_VALUE_BYTES = 65_536
_MAX_FIELDS_PER_ENTRY = 128
_MAX_ENTRIES = 256
_BINARY_LENGTH_BYTES = 8
_LF = 0x0A
_FIELD_NAME = re.compile(rb"[A-Z_][A-Z0-9_]{0,63}", re.ASCII)


class JournalExportError(ValueError):
    """The bytes are outside the bounded Journal Export profile."""


class JournalFieldRepresentation(StrEnum):
    TEXT = "text"
    BINARY = "binary"


@dataclass(frozen=True, slots=True)
class JournalExportField:
    name: str
    value: bytes
    representation: JournalFieldRepresentation


@dataclass(frozen=True, slots=True)
class JournalExportEntry:
    fields: tuple[JournalExportField, ...]


def _field_name(raw_name: bytes) -> str:
    if _FIELD_NAME.fullmatch(raw_name) is None:
        raise JournalExportError("a field name is outside the closed safety profile")
    return raw_name.decode("ascii")


def _validate_text_value(value: bytes) -> None:
    if len(value) > _MAX_VALUE_BYTES:
        raise JournalExportError("a text value exceeds the supported byte limit")
    try:
        text = value.decode("utf-8", errors="strict")
    except UnicodeDecodeError as error:
        raise JournalExportError("a text value is not valid UTF-8") from error
    if any(character != "\t" and ord(character) < 0x20 for character in text):
        raise JournalExportError("a text value contains a disallowed control code point")


def parse_systemd_journal_export(
    data: bytes,
) -> tuple[JournalExportEntry, ...]:
    """Parse one complete bounded Journal Export snapshot."""
    if type(data) is not bytes:
        raise TypeError("data must be exact bytes")
    if len(data) > _MAX_STREAM_BYTES:
        raise JournalExportError("data exceeds the supported stream byte limit")
    if not data:
        return ()

    entries: list[JournalExportEntry] = []
    fields: list[JournalExportField] = []
    cursor = 0

    while cursor < len(data):
        line_end = data.find(b"\n", cursor)
        if line_end < 0:
            raise JournalExportError("a field header or entry terminator is truncated")
        header = data[cursor:line_end]
        cursor = line_end + 1

        if not header:
            if not fields:
                raise JournalExportError("empty or repeated entries are not allowed")
            entries.append(JournalExportEntry(tuple(fields)))
            if len(entries) > _MAX_ENTRIES:
                raise JournalExportError("entry count exceeds the supported limit")
            fields.clear()
            continue

        if len(entries) == _MAX_ENTRIES:
            raise JournalExportError("entry count exceeds the supported limit")
        if len(fields) == _MAX_FIELDS_PER_ENTRY:
            raise JournalExportError("field count exceeds the per-entry limit")

        equals_at = header.find(b"=")
        if equals_at >= 0:
            name = _field_name(header[:equals_at])
            value = header[equals_at + 1 :]
            _validate_text_value(value)
            fields.append(
                JournalExportField(
                    name=name,
                    value=value,
                    representation=JournalFieldRepresentation.TEXT,
                )
            )
            continue

        name = _field_name(header)
        if len(data) - cursor < _BINARY_LENGTH_BYTES:
            raise JournalExportError("a binary value length is truncated")
        value_length = int.from_bytes(
            data[cursor : cursor + _BINARY_LENGTH_BYTES],
            "little",
        )
        if value_length > _MAX_VALUE_BYTES:
            raise JournalExportError("a binary value exceeds the supported byte limit")
        cursor += _BINARY_LENGTH_BYTES

        value_end = cursor + value_length
        if value_end >= len(data):
            raise JournalExportError("a binary value or its terminating LF is truncated")
        if data[value_end] != _LF:
            raise JournalExportError("a binary value is not followed by LF")
        value = data[cursor:value_end]
        cursor = value_end + 1
        fields.append(
            JournalExportField(
                name=name,
                value=value,
                representation=JournalFieldRepresentation.BINARY,
            )
        )

    if fields:
        raise JournalExportError("the final entry is missing its closing blank LF")
    return tuple(entries)


```

## Example

```python
def binary_field(name: bytes, value: bytes) -> bytes:
    return name + b"\n" + len(value).to_bytes(8, "little") + value + b"\n"


stream = (
    b"__SYNTHETIC=opaque\n"
    b"MESSAGE=ready\tok\n"
    b"TAG=first\n"
    b"TAG=second\n"
    + binary_field(b"PAYLOAD", b"first line\nsecond line")
    + binary_field(b"DETAIL", b"printable but binary")
    + b"\n"
    + b"MESSAGE=complete\n\n"
)
entries = parse_systemd_journal_export(stream)

invalid_streams = (b"\n", stream[:-1], stream + b"\n")
rejected = 0
for invalid in invalid_streams:
    try:
        parse_systemd_journal_export(invalid)
    except JournalExportError:
        rejected += 1

assert entries == (
    JournalExportEntry(
        (
            JournalExportField("__SYNTHETIC", b"opaque", JournalFieldRepresentation.TEXT),
            JournalExportField("MESSAGE", b"ready\tok", JournalFieldRepresentation.TEXT),
            JournalExportField("TAG", b"first", JournalFieldRepresentation.TEXT),
            JournalExportField("TAG", b"second", JournalFieldRepresentation.TEXT),
            JournalExportField(
                "PAYLOAD",
                b"first line\nsecond line",
                JournalFieldRepresentation.BINARY,
            ),
            JournalExportField(
                "DETAIL",
                b"printable but binary",
                JournalFieldRepresentation.BINARY,
            ),
        )
    ),
    JournalExportEntry(
        (JournalExportField("MESSAGE", b"complete", JournalFieldRepresentation.TEXT),)
    ),
)
assert parse_systemd_journal_export(b"") == ()
assert rejected == len(invalid_streams)
```

## Trade-offs and Limitations

Parsing takes `O(B + F)` time for `B` admitted bytes and `F` fields. The input
snapshot, line and value slices, decoded text used for validation, and immutable
result can coexist temporarily, so peak memory is also `O(B)` under the fixed
one-MiB stream cap. Binary lengths are rejected before their declared bodies
are sliced, and no partial result accompanies an exception.

Every non-empty entry must end with its blank LF, including the final entry.
Text values may be empty and accept only strict UTF-8 code points equal to TAB
or at least U+0020. Binary values may contain arbitrary bytes and retain binary
representation even when the same value could use text form. Duplicate names,
field order, and unknown double-underscore metadata are deliberately preserved.

The byte, value, field, entry, and field-name limits are local safety policy,
not systemd wire-format limits. This parser does not read native journal files,
follow a live stream, interpret well-known fields, normalize metadata, redact
possibly sensitive values, verify provenance, authenticate input, or import
entries into the journal.

## Related Snippets

<!-- catalog:related:start -->
- [Read a Bounded Log Delta Across One Rename-Based Rotation](read-a-bounded-log-delta-across-one-rename-based-rotation.md)
- [Recover a Verified Prefix from a Bounded CRC-Framed Byte Log](../storage-databases/recover-a-verified-prefix-from-a-bounded-crc-framed-byte-log.md)
- [Decode Bounded Fixed-Width ASCII Records Across Arbitrary Byte Chunks](../data-processing/decode-bounded-fixed-width-ascii-records-across-arbitrary-byte-chunks.md)
<!-- catalog:related:end -->
