---
title: "Encode and Decode a Bounded Big-Endian Complex Vector with struct"
snippet_type: recipe
use_cases:
  - interoperability
  - serialization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - encode-and-decode-one-exact-size-big-endian-binary-record-with-struct.md
  - ../networking-protocols/read-and-write-size-capped-varint-frames.md
  - ../data-processing/extract-a-finite-2d-bounding-box-from-bounded-wkb.md
---

# Encode and Decode a Bounded Big-Endian Complex Vector with struct

## Idea and Problem

Encode one bounded vector of finite complex numbers in an explicit big-endian binary profile with a precision tag and exact framing.

The seven-byte `>4scH` header carries the magic `CV01`, the `F` or `D`
precision code, and an unsigned sample count. The payload then repeats that
same `struct` complex format: `F` stores two binary32 components per sample,
while `D` stores two binary64 components. Exact length checks prevent a prefix,
truncated payload, or concatenated record from being mistaken for one vector.

## When to Use

Use this recipe when two Python 3.14 boundaries explicitly share this small
in-memory profile, the complete message is available as exact `bytes`, and a
fixed choice between compact binary32 and binary64 components is sufficient.
The frozen result preserves both the decoded precision and sample order.

Use a maintained array or serialization format when dimensions, metadata,
schema evolution, streaming, checksums, cross-language compatibility testing,
or zero-copy access matter. This private fixed layout is not a general file
format.

## Implementation

```python
from dataclasses import dataclass
from enum import StrEnum
from math import copysign, isfinite
from struct import Struct
from struct import error as StructError

_MAGIC = b"CV01"
_HEADER = Struct(">4scH")
_MIN_SAMPLES = 1
_MAX_SAMPLES = 4_096


class ComplexVectorError(ValueError):
    """Raised when data violates the bounded complex-vector profile."""


class ComplexPrecision(StrEnum):
    BINARY32 = "F"
    BINARY64 = "D"


@dataclass(frozen=True, slots=True)
class ComplexVector:
    precision: ComplexPrecision
    samples: tuple[complex, ...]


def _payload_struct(
    precision: ComplexPrecision,
    count: int,
) -> Struct:
    return Struct(f">{count}{precision.value}")


def _validate_samples(samples: tuple[complex, ...]) -> None:
    if type(samples) is not tuple:
        raise TypeError("samples must be an exact tuple")
    if not _MIN_SAMPLES <= len(samples) <= _MAX_SAMPLES:
        raise ValueError("samples must contain from 1 through 4096 values")
    for sample in samples:
        if type(sample) is not complex:
            raise TypeError("every sample must be an exact complex value")
        if not isfinite(sample.real) or not isfinite(sample.imag):
            raise ValueError("every complex component must be finite")


def encode_complex_vector(
    precision: ComplexPrecision,
    samples: tuple[complex, ...],
) -> bytes:
    """Encode one validated complex vector into the fixed profile."""
    if type(precision) is not ComplexPrecision:
        raise TypeError("precision must be an exact ComplexPrecision")
    _validate_samples(samples)

    payload_format = _payload_struct(precision, len(samples))
    try:
        payload = payload_format.pack(*samples)
    except (OverflowError, StructError) as error:
        raise ComplexVectorError("a component is outside the selected binary precision") from error

    packed_samples = payload_format.unpack(payload)
    if any(not isfinite(sample.real) or not isfinite(sample.imag) for sample in packed_samples):
        raise ComplexVectorError("packing produced a non-finite component")

    header = _HEADER.pack(
        _MAGIC,
        precision.value.encode("ascii"),
        len(samples),
    )
    return header + payload


def decode_complex_vector(data: bytes) -> ComplexVector:
    """Decode exactly one bounded complex vector."""
    if type(data) is not bytes:
        raise TypeError("data must be exact bytes")
    if len(data) < _HEADER.size:
        raise ComplexVectorError("data is shorter than the vector header")

    magic, precision_tag, count = _HEADER.unpack_from(data)
    if magic != _MAGIC:
        raise ComplexVectorError("data has unsupported vector magic")
    try:
        precision = ComplexPrecision(precision_tag.decode("ascii"))
    except (UnicodeDecodeError, ValueError) as error:
        raise ComplexVectorError("data has an unsupported precision tag") from error
    if not _MIN_SAMPLES <= count <= _MAX_SAMPLES:
        raise ComplexVectorError("sample count is outside the supported range")

    payload_format = _payload_struct(precision, count)
    expected_size = _HEADER.size + payload_format.size
    if len(data) != expected_size:
        raise ComplexVectorError("data length does not match its sample count")

    samples = payload_format.unpack_from(data, _HEADER.size)
    if any(not isfinite(sample.real) or not isfinite(sample.imag) for sample in samples):
        raise ComplexVectorError("decoded vector contains a non-finite component")
    return ComplexVector(precision=precision, samples=samples)
```

## Example

```python


binary64_samples = (
    complex(1.25, -2.5),
    complex(-0.0, 0.0),
    complex(0.0, -0.0),
)
binary64_data = encode_complex_vector(
    ComplexPrecision.BINARY64,
    binary64_samples,
)
binary64_result = decode_complex_vector(binary64_data)

assert binary64_data[:7] == bytes.fromhex("43563031440003")
assert len(binary64_data) == 7 + 3 * 16
assert binary64_result.samples == binary64_samples
assert copysign(1.0, binary64_result.samples[1].real) == -1.0
assert copysign(1.0, binary64_result.samples[2].imag) == -1.0

binary32_data = encode_complex_vector(
    ComplexPrecision.BINARY32,
    (complex(1.1, 2.2),),
)
binary32_result = decode_complex_vector(binary32_data)
assert len(binary32_data) == 7 + 8
assert binary32_result.samples == (complex(1.100000023841858, 2.200000047683716),)

try:
    encode_complex_vector(
        ComplexPrecision.BINARY32,
        (complex(3.5e38, 0.0),),
    )
except ComplexVectorError:
    binary32_overflow_rejected = True
else:
    binary32_overflow_rejected = False

try:
    decode_complex_vector(binary64_data + b"trailing")
except ComplexVectorError:
    trailing_bytes_rejected = True
else:
    trailing_bytes_rejected = False

assert binary32_overflow_rejected and trailing_bytes_rejected
```

## Trade-offs and Limitations

Validation, packing, and unpacking use `O(N)` time and memory for `N` bounded
samples. Binary32 intentionally rounds accepted components and rejects packing
overflow under this profile. Binary64 preserves Python complex components
exactly, including signed zero, on the tested CPython 3.14 runtime; other
implementations need their own interoperability tests.

The magic, count, and exact byte length identify only this layout. They provide
no integrity, authentication, confidentiality, corruption recovery, or schema
evolution. A changed finite payload can decode successfully, and the decoder
deliberately rejects buffers, streams, prefixes, trailing bytes, empty vectors,
unknown tags, and non-finite components.

## Related Snippets

<!-- catalog:related:start -->
- [Encode and Decode One Exact-Size Big-Endian Binary Record with struct.Struct](encode-and-decode-one-exact-size-big-endian-binary-record-with-struct.md)
- [Read and Write Size-Capped Varint Frames](../networking-protocols/read-and-write-size-capped-varint-frames.md)
- [Extract a Finite 2D Bounding Box from Bounded WKB](../data-processing/extract-a-finite-2d-bounding-box-from-bounded-wkb.md)
<!-- catalog:related:end -->
