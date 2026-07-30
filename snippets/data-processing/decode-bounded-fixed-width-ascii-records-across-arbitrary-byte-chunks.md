---
title: "Decode Bounded Fixed-Width ASCII Records Across Arbitrary Byte Chunks"
snippet_type: pattern
use_cases:
  - data-transformation
  - parsing
  - resource-management
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - decode-bounded-json-lines-across-arbitrary-text-chunks.md
  - assemble-fixed-width-feature-slots-from-bounded-named-sources.md
  - ../testing-tooling/enumerate-every-contiguous-chunking-of-bounded-bytes-for-stream-tests.md
---

# Decode Bounded Fixed-Width ASCII Records Across Arbitrary Byte Chunks

## Idea and Problem

Decode a complete bounded fixed-width ASCII document into immutable rows without making record boundaries depend on the supplied byte chunks.

The schema fixes field widths and ASCII-space trimming policies. The decoder
validates the complete schema and chunk envelope, preflights result size, then
carries less than one record between chunks instead of joining the full input.
Every accepted chunking of the same bytes therefore produces the same table.

## When to Use

Use this pattern for bounded in-memory imports whose records have no delimiters
and whose field positions are defined by a trusted fixed-width schema. It fits
legacy exports, deterministic fixtures, and compact interchange files that use
printable ASCII and byte widths rather than Unicode display columns.

Use a streaming row consumer when the complete immutable result is too large,
or a protocol-specific parser when fields contain binary data, character-set
encodings, newlines, numeric conversions, optional layouts, or checksums.

## Implementation

```python
import re
from dataclasses import dataclass
from enum import StrEnum

_MAX_FIELDS = 64
_MAX_FIELD_WIDTH = 1_024
_MAX_RECORD_WIDTH = 4_096
_MAX_CHUNKS = 4_096
_MAX_CHUNK_BYTES = 1 * 1_024 * 1_024
_MAX_INPUT_BYTES = 64 * 1_024 * 1_024
_MAX_RECORDS = 100_000
_MAX_CELLS = 1_000_000
_FIELD_NAME = re.compile(r"[a-z][a-z0-9_]{0,63}", flags=re.ASCII)


class TrimMode(StrEnum):
    PRESERVE = "preserve"
    LEFT_SPACES = "left_spaces"
    RIGHT_SPACES = "right_spaces"
    BOTH_SPACES = "both_spaces"


@dataclass(frozen=True, slots=True)
class FieldSpec:
    name: str
    width: int
    trim: TrimMode


@dataclass(frozen=True, slots=True)
class FixedWidthTable:
    field_names: tuple[str, ...]
    rows: tuple[tuple[str, ...], ...]


class FixedWidthDecodeError(ValueError):
    def __init__(self, message: str, *, byte_offset: int) -> None:
        self.byte_offset = byte_offset
        super().__init__(f"{message} at byte offset {byte_offset}")


def _validated_layout(
    schema: object,
) -> tuple[tuple[str, int, int, TrimMode], ...]:
    if type(schema) is not tuple:
        raise TypeError("schema must be an exact tuple")
    if not 1 <= len(schema) <= _MAX_FIELDS:
        raise ValueError("field count is outside the supported range")

    names: set[str] = set()
    layout: list[tuple[str, int, int, TrimMode]] = []
    record_width = 0
    for field in schema:
        if type(field) is not FieldSpec:
            raise TypeError("schema must contain exact FieldSpec instances")
        if type(field.name) is not str:
            raise TypeError("field names must be exact strings")
        if _FIELD_NAME.fullmatch(field.name) is None:
            raise ValueError("field names must match [a-z][a-z0-9_]{0,63}")
        if field.name in names:
            raise ValueError("field names must be unique")
        names.add(field.name)
        if type(field.width) is not int:
            raise TypeError("field widths must be exact integers")
        if not 1 <= field.width <= _MAX_FIELD_WIDTH:
            raise ValueError("field width is outside the supported range")
        if type(field.trim) is not TrimMode:
            raise TypeError("field trim modes must be exact TrimMode values")

        start = record_width
        record_width += field.width
        if record_width > _MAX_RECORD_WIDTH:
            raise ValueError("record width exceeds the supported cap")
        layout.append((field.name, start, record_width, field.trim))
    return tuple(layout)


def _validated_chunks(chunks: object) -> tuple[tuple[bytes, ...], int]:
    if type(chunks) is not tuple:
        raise TypeError("chunks must be an exact tuple")
    if len(chunks) > _MAX_CHUNKS:
        raise ValueError("chunk count exceeds the supported cap")

    total_bytes = 0
    for chunk in chunks:
        if type(chunk) is not bytes:
            raise TypeError("chunks must contain exact bytes")
        if len(chunk) > _MAX_CHUNK_BYTES:
            raise ValueError("one chunk exceeds the supported byte cap")
        total_bytes += len(chunk)
        if total_bytes > _MAX_INPUT_BYTES:
            raise ValueError("aggregate input exceeds the supported byte cap")
    return chunks, total_bytes


def _trim_ascii_spaces(value: bytes, mode: TrimMode) -> bytes:
    if mode is TrimMode.PRESERVE:
        return value
    if mode is TrimMode.LEFT_SPACES:
        return value.lstrip(b" ")
    if mode is TrimMode.RIGHT_SPACES:
        return value.rstrip(b" ")
    return value.strip(b" ")


def decode_fixed_width_ascii_records(
    schema: tuple[FieldSpec, ...],
    chunks: tuple[bytes, ...],
) -> FixedWidthTable:
    """Decode one complete, bounded document independently of chunking."""
    layout = _validated_layout(schema)
    checked_chunks, total_bytes = _validated_chunks(chunks)
    record_width = layout[-1][2]

    record_count, partial_width = divmod(total_bytes, record_width)
    if partial_width:
        raise FixedWidthDecodeError(
            "incomplete fixed-width record",
            byte_offset=record_count * record_width,
        )
    if record_count > _MAX_RECORDS:
        raise ValueError("record count exceeds the supported cap")
    if record_count * len(layout) > _MAX_CELLS:
        raise ValueError("result cell count exceeds the supported cap")

    rows: list[tuple[str, ...]] = []
    carry = b""
    next_record_offset = 0
    for chunk in checked_chunks:
        block = carry + chunk
        cursor = 0
        while len(block) - cursor >= record_width:
            record = block[cursor : cursor + record_width]
            for local_offset, byte in enumerate(record):
                if not 0x20 <= byte <= 0x7E:
                    raise FixedWidthDecodeError(
                        "non-printable ASCII byte",
                        byte_offset=next_record_offset + local_offset,
                    )

            row: list[str] = []
            for _, start, stop, trim in layout:
                raw_field = record[start:stop]
                row.append(_trim_ascii_spaces(raw_field, trim).decode("ascii"))
            rows.append(tuple(row))
            cursor += record_width
            next_record_offset += record_width
        carry = block[cursor:]

    if carry:
        raise AssertionError("preflight accepted a partial record")
    return FixedWidthTable(
        field_names=tuple(name for name, _, _, _ in layout),
        rows=tuple(rows),
    )


```

## Example

```python
schema = (
    FieldSpec("left", 5, TrimMode.LEFT_SPACES),
    FieldSpec("right", 5, TrimMode.RIGHT_SPACES),
    FieldSpec("both", 5, TrimMode.BOTH_SPACES),
    FieldSpec("raw", 5, TrimMode.PRESERVE),
)
payload = b"  abcabc   ab   x y    12done   z       "
expected = FixedWidthTable(
    field_names=("left", "right", "both", "raw"),
    rows=(
        ("abc", "abc", "ab", " x y "),
        ("12", "done", "z", "     "),
    ),
)

one_chunk = decode_fixed_width_ascii_records(schema, (payload,))
one_byte_chunks = decode_fixed_width_ascii_records(
    schema,
    tuple(payload[index : index + 1] for index in range(len(payload))),
)
uneven_chunks = decode_fixed_width_ascii_records(
    schema,
    (b"", payload[:3], b"", payload[3:27], payload[27:], b""),
)
assert one_chunk == one_byte_chunks == uneven_chunks == expected
assert decode_fixed_width_ascii_records(schema, ()).rows == ()

malformed = payload[:7] + b"\n" + payload[8:]
try:
    decode_fixed_width_ascii_records(schema, (malformed[:4], malformed[4:]))
except FixedWidthDecodeError as error:
    assert error.byte_offset == 7
else:
    raise AssertionError("a non-printable byte was accepted")

try:
    decode_fixed_width_ascii_records(schema, (payload[:-1],))
except FixedWidthDecodeError as error:
    assert error.byte_offset == 20
else:
    raise AssertionError("a partial record was accepted")
assert one_chunk.rows == expected.rows
```

## Trade-offs and Limitations

Schema and envelope validation take `O(F + C)` time for `F` fields and `C`
chunks. Parsing takes `O(B + C * W + R * F)` time for `B` bytes, maximum record
width `W`, and `R` records: joining each chunk to the carry can recopy up to one
partial record. The immutable result uses `O(B + R * F)` memory. Apart from the
result, parsing retains less than one record between chunks and constructs a
temporary block no larger than one accepted chunk plus that carry. Prefer
larger chunks, or an incrementally filled mutable record buffer when a strict
`O(B + R * F)` bound is required under adversarial fragmentation.

The decoder accepts only printable ASCII bytes from `0x20` through `0x7e` and
trims only byte `SP`. It has no newline handling, Unicode or display-width
semantics, field conversion, alternate layouts, or stateful public interface.
The schema is limited to 64 fields and 4,096 bytes per record; the document is
also bounded by chunk, byte, record, and result-cell caps.

## Related Snippets

<!-- catalog:related:start -->
- [Decode Bounded JSON Lines Across Arbitrary Text Chunks](decode-bounded-json-lines-across-arbitrary-text-chunks.md)
- [Assemble Fixed-Width Feature Slots from Bounded Named Sources](assemble-fixed-width-feature-slots-from-bounded-named-sources.md)
- [Enumerate Every Contiguous Chunking of Bounded Bytes for Stream Tests](../testing-tooling/enumerate-every-contiguous-chunking-of-bounded-bytes-for-stream-tests.md)
<!-- catalog:related:end -->
