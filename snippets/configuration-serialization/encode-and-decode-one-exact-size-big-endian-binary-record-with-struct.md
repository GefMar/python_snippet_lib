---
title: "Encode and Decode One Exact-Size Big-Endian Binary Record with struct.Struct"
snippet_type: recipe
use_cases:
  - interoperability
  - parsing
  - serialization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../networking-protocols/read-and-write-size-capped-varint-frames.md
  - ../data-processing/extract-a-finite-2d-bounding-box-from-bounded-wkb.md
  - decode-canonical-unpadded-base64url-under-encoded-and-decoded-byte-limits.md
---

# Encode and Decode One Exact-Size Big-Endian Binary Record with struct.Struct

## Idea and Problem

Compile one fixed binary layout once, validate its complete value ranges, and require exactly one record on decode.

The `>4sHIq` layout contains the fixed magic `BR01`, an unsigned 16-bit code,
an unsigned 32-bit sequence, and a signed 64-bit value. The `>` prefix fixes
big-endian byte order and standard field sizes without native alignment, so the
record is always 18 bytes on every supported platform.

## When to Use

Use this recipe when two boundaries explicitly share this small immutable wire
format and exactly one complete record is already available in memory. The
magic distinguishes this profile from unrelated bytes, while strict input
types and sizes prevent implicit coercion or trailing-record acceptance.

Use a maintained serialization format when fields must evolve independently,
unknown versions must be skipped, records are streamed in batches, or schemas
must be described outside Python. A fixed struct is deliberately not a general
message protocol.

## Implementation

```python
from dataclasses import dataclass
from struct import Struct

_MAGIC = b"BR01"
_RECORD_STRUCT = Struct(">4sHIq")
_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_UINT16 = (1 << 16) - 1
_MAX_UINT32 = (1 << 32) - 1


class BinaryRecordError(ValueError):
    """Raised when bytes do not contain this exact binary record profile."""


@dataclass(frozen=True, slots=True)
class BinaryRecord:
    code: int
    sequence: int
    value: int


def _bounded_exact_integer(
    value: object,
    *,
    name: str,
    minimum: int,
    maximum: int,
) -> int:
    if type(value) is not int:
        raise TypeError(f"{name} must be an exact integer")
    if not minimum <= value <= maximum:
        raise ValueError(f"{name} is outside the supported range")
    return value


def encode_binary_record(record: BinaryRecord) -> bytes:
    """Encode one validated record into the fixed 18-byte representation."""
    if type(record) is not BinaryRecord:
        raise TypeError("record must be an exact BinaryRecord")
    code = _bounded_exact_integer(
        record.code,
        name="record.code",
        minimum=0,
        maximum=_MAX_UINT16,
    )
    sequence = _bounded_exact_integer(
        record.sequence,
        name="record.sequence",
        minimum=0,
        maximum=_MAX_UINT32,
    )
    value = _bounded_exact_integer(
        record.value,
        name="record.value",
        minimum=_MIN_INT64,
        maximum=_MAX_INT64,
    )
    return _RECORD_STRUCT.pack(_MAGIC, code, sequence, value)


def decode_binary_record(data: bytes) -> BinaryRecord:
    """Decode exactly one record and reject other sizes or format magic."""
    if type(data) is not bytes:
        raise TypeError("data must be exact bytes")
    if len(data) != _RECORD_STRUCT.size:
        raise BinaryRecordError("data is not exactly one binary record")

    magic, code, sequence, value = _RECORD_STRUCT.unpack(data)
    if magic != _MAGIC:
        raise BinaryRecordError("data has unsupported binary record magic")
    return BinaryRecord(code=code, sequence=sequence, value=value)
```

## Example

```python
record = BinaryRecord(
    code=0x1234,
    sequence=0x01020304,
    value=-2,
)
encoded = encode_binary_record(record)
golden = bytes.fromhex("42523031123401020304fffffffffffffffe")
decoded = decode_binary_record(encoded)

try:
    decode_binary_record(b"BAD!" + encoded[4:])
except BinaryRecordError:
    wrong_magic_rejected = True
else:
    wrong_magic_rejected = False

assert (
    _RECORD_STRUCT.size,
    encoded,
    decoded,
    wrong_magic_rejected,
) == (
    18,
    golden,
    record,
    True,
)
```

## Trade-offs and Limitations

Encoding and decoding each use fixed `O(1)` time and memory because every record
is exactly 18 bytes. A compiled `Struct` avoids repeatedly parsing the format,
and the explicit big-endian standard layout avoids platform-dependent native
sizes and padding.

The magic and exact byte count identify this one layout but provide no checksum,
authentication, confidentiality, or corruption recovery. Changed field bytes
can still decode as valid integers. The decoder intentionally rejects
`bytearray`, `memoryview`, prefixes, trailing bytes, other magic values, and
multiple concatenated records. Field meaning, version migration, streaming,
and durable storage framing require separate contracts.

## Related Snippets

<!-- catalog:related:start -->
- [Read and Write Size-Capped Varint Frames](../networking-protocols/read-and-write-size-capped-varint-frames.md)
- [Extract a Finite 2D Bounding Box from Bounded WKB](../data-processing/extract-a-finite-2d-bounding-box-from-bounded-wkb.md)
- [Decode Canonical Unpadded Base64url Under Encoded and Decoded Byte Limits](decode-canonical-unpadded-base64url-under-encoded-and-decoded-byte-limits.md)
<!-- catalog:related:end -->
