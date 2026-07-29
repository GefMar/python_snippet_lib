---
title: "Combine zlib CRC-32 Values for Concatenated Byte Segments"
snippet_type: algorithm
use_cases:
  - data-transformation
  - persistence
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - recover-a-verified-prefix-from-a-bounded-crc-framed-byte-log.md
  - read-an-exact-posix-byte-range-with-os-pread-without-moving-the-file-descriptor-offset.md
  - fingerprint-a-bounded-flat-file-set-with-framed-sha-256.md
---

# Combine zlib CRC-32 Values for Concatenated Byte Segments

## Idea and Problem

Derive the zlib CRC-32 of two concatenated byte segments from their individual checksums and the suffix length, without reading either segment again.

Appending a zero byte is a linear transformation over `GF(2)`. A fixed
32-column matrix represents that transformation for the reflected CRC-32
polynomial. Binary exponentiation applies the matrix for the complete suffix
length in logarithmic time, after which the suffix checksum is combined with
an exclusive-or.

The inputs and output use the unsigned finalized values returned by
`zlib.crc32`.

## When to Use

Use this operation when separately checksummed byte chunks are concatenated
logically and recomputing a checksum from their payload bytes would require
unnecessary I/O. Typical examples include validating chunk assembly or
combining cached checksums in a storage tree.

This is compatible specifically with the CRC-32 convention used by zlib. See
the [zlib manual](https://zlib.net/manual.html) for the library checksum API and
the upstream [`crc32.c`](https://github.com/madler/zlib/blob/develop/crc32.c)
for its combine implementation.

## Implementation

```python
_CRC32_POLYNOMIAL = 0xEDB88320
_CRC32_WIDTH = 32
_UINT32_MAX = (1 << _CRC32_WIDTH) - 1
_MAX_SUFFIX_LENGTH = (1 << 63) - 1


def _gf2_matrix_times(matrix: tuple[int, ...], vector: int) -> int:
    product = 0
    column = 0
    while vector:
        if vector & 1:
            product ^= matrix[column]
        vector >>= 1
        column += 1
    return product


def _gf2_matrix_square(matrix: tuple[int, ...]) -> tuple[int, ...]:
    return tuple(
        _gf2_matrix_times(matrix, column_image)
        for column_image in matrix
    )


def _one_zero_byte_operator() -> tuple[int, ...]:
    one_zero_bit = [_CRC32_POLYNOMIAL]
    basis = 1
    for _ in range(1, _CRC32_WIDTH):
        one_zero_bit.append(basis)
        basis <<= 1

    two_zero_bits = _gf2_matrix_square(tuple(one_zero_bit))
    four_zero_bits = _gf2_matrix_square(two_zero_bits)
    return _gf2_matrix_square(four_zero_bits)


def combine_zlib_crc32(
    prefix_crc32: int,
    suffix_crc32: int,
    suffix_length: int,
) -> int:
    """Return zlib.crc32(prefix + suffix) from bounded checksum metadata."""
    for name, checksum in (
        ("prefix_crc32", prefix_crc32),
        ("suffix_crc32", suffix_crc32),
    ):
        if type(checksum) is not int:
            raise TypeError(f"{name} must be an exact integer")
        if not 0 <= checksum <= _UINT32_MAX:
            raise ValueError(f"{name} must be an unsigned 32-bit value")
    if type(suffix_length) is not int:
        raise TypeError("suffix_length must be an exact integer")
    if not 0 <= suffix_length <= _MAX_SUFFIX_LENGTH:
        raise ValueError("suffix_length is outside the supported range")
    if suffix_length == 0:
        if suffix_crc32 != 0:
            raise ValueError("an empty suffix must have the zlib CRC-32 value zero")
        return prefix_crc32

    transformed_prefix = prefix_crc32
    byte_operator = _one_zero_byte_operator()
    remaining = suffix_length
    while remaining:
        if remaining & 1:
            transformed_prefix = _gf2_matrix_times(
                byte_operator,
                transformed_prefix,
            )
        remaining >>= 1
        if remaining:
            byte_operator = _gf2_matrix_square(byte_operator)

    return (transformed_prefix ^ suffix_crc32) & _UINT32_MAX
```

## Example

```python
import zlib

prefix = b"bounded-prefix:"
middle = b"structured-middle:"
suffix = b"verified-suffix"

prefix_crc = zlib.crc32(prefix)
middle_crc = zlib.crc32(middle)
suffix_crc = zlib.crc32(suffix)

prefix_middle_crc = combine_zlib_crc32(
    prefix_crc,
    middle_crc,
    len(middle),
)
combined_crc = combine_zlib_crc32(
    prefix_middle_crc,
    suffix_crc,
    len(suffix),
)

assert combined_crc == zlib.crc32(prefix + middle + suffix)

middle_suffix_crc = combine_zlib_crc32(
    middle_crc,
    suffix_crc,
    len(suffix),
)
assert combine_zlib_crc32(
    prefix_crc,
    middle_suffix_crc,
    len(middle) + len(suffix),
) == combined_crc

assert combine_zlib_crc32(prefix_crc, 0, 0) == prefix_crc
```

## Trade-offs and Limitations

The function performs `O(log L)` fixed-width matrix operations for suffix
length `L` and uses `O(1)` auxiliary space. The matrices always contain 32
unsigned polynomial vectors; running time does not depend on segment contents.

CRC-32 detects many accidental transmission and storage errors, but it is not
collision resistant, keyed, or suitable for authenticating untrusted data.
The function does not prove that a supplied checksum or length describes the
claimed bytes.

Only the reflected zlib CRC-32 polynomial and checksum convention are
supported. This snippet does not implement custom polynomials, recover missing
payload bytes, stream data, combine other checksum families, or replace a
cryptographic digest or message authentication code.

## Related Snippets

<!-- catalog:related:start -->
- [Recover a Verified Prefix from a Bounded CRC-Framed Byte Log](recover-a-verified-prefix-from-a-bounded-crc-framed-byte-log.md)
- [Read an Exact POSIX Byte Range with os.pread Without Moving the File Descriptor Offset](read-an-exact-posix-byte-range-with-os-pread-without-moving-the-file-descriptor-offset.md)
- [Fingerprint a Bounded Flat File Set with Framed SHA-256](fingerprint-a-bounded-flat-file-set-with-framed-sha-256.md)
<!-- catalog:related:end -->
