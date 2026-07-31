---
title: "Encode and Decode One Size-Capped COBS Byte Block"
snippet_type: algorithm
use_cases:
  - data-transformation
  - serialization
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - encode-and-decode-a-bounded-full-block-z85-frame.md
  - ../networking-protocols/encode-and-decode-one-canonical-size-capped-netstring.md
  - ../networking-protocols/read-one-bounded-async-byte-field-with-either-line-feed-or-nul-terminator.md
---

# Encode and Decode One Size-Capped COBS Byte Block

## Idea and Problem

Represent arbitrary bounded bytes as one canonical zero-free block so a separate zero byte can delimit it on a transport.

Consistent Overhead Byte Stuffing divides input into runs of at most 254
nonzero bytes. Each run is preceded by a code equal to one plus its length;
code `0xff` means a full 254-byte run without an implied zero. Shorter blocks
imply an original zero when another code follows.

The codec here handles only the delimiter-free block. Choosing the shortest
valid spelling, including omission of a redundant empty block after a final
254-byte run, gives every raw payload one canonical representation.

## When to Use

Use COBS when a byte-oriented link or small embedded protocol reserves zero as
an unambiguous frame delimiter and bounded deterministic overhead matters. The
canonical decoder is also useful for fixtures and validation at a closed
serialization boundary.

Add and remove the transport delimiter outside this helper. Use a standard
framing library when partial reads, multiple frames, resynchronization, error
reporting, or protocol interoperability is required. Add a separately defined
integrity mechanism when corrupted data must be detected.

## Implementation

```python
from itertools import product
from random import Random

_MAX_COBS_RAW_BYTES = 1_048_576
_MAX_COBS_ENCODED_BYTES = 1_052_705


def encode_canonical_cobs(data: bytes) -> bytes:
    """Encode exact bytes as one shortest delimiter-free COBS block."""
    if type(data) is not bytes:
        raise TypeError("data must be exact bytes")
    if len(data) > _MAX_COBS_RAW_BYTES:
        raise ValueError("data exceeds 1048576 bytes")

    encoded = bytearray((0,))
    code_index = 0
    code = 1
    for value in data:
        if value == 0:
            encoded[code_index] = code
            code_index = len(encoded)
            encoded.append(0)
            code = 1
            continue
        encoded.append(value)
        code += 1
        if code == 0xFF:
            encoded[code_index] = 0xFF
            code_index = len(encoded)
            encoded.append(0)
            code = 1

    if code_index == len(encoded) - 1 and code == 1 and data and data[-1] != 0:
        encoded.pop()
    else:
        encoded[code_index] = code
    return bytes(encoded)


def decode_canonical_cobs(encoded: bytes) -> bytes:
    """Decode one canonical delimiter-free COBS block."""
    if type(encoded) is not bytes:
        raise TypeError("encoded must be exact bytes")
    if not encoded:
        raise ValueError("encoded block must be nonempty")
    if len(encoded) > _MAX_COBS_ENCODED_BYTES:
        raise ValueError("encoded block exceeds 1052705 bytes")
    if 0 in encoded:
        raise ValueError("a delimiter-free COBS block cannot contain zero")

    decoded = bytearray()
    index = 0
    while index < len(encoded):
        code = encoded[index]
        block_end = index + code
        if block_end > len(encoded):
            raise ValueError("COBS code overruns the encoded block")
        block_length = code - 1
        if len(decoded) + block_length > _MAX_COBS_RAW_BYTES:
            raise ValueError("decoded data exceeds 1048576 bytes")
        decoded.extend(encoded[index + 1 : block_end])
        index = block_end
        if code != 0xFF and index < len(encoded):
            if len(decoded) == _MAX_COBS_RAW_BYTES:
                raise ValueError("decoded data exceeds 1048576 bytes")
            decoded.append(0)

    result = bytes(decoded)
    if encode_canonical_cobs(result) != encoded:
        raise ValueError("encoded block is not the shortest canonical COBS spelling")
    return result
```

## Example

```python
def zero_run_oracle(data: bytes) -> bytes:
    if not data:
        return b"\x01"
    encoded = bytearray()
    cursor = 0
    while cursor < len(data):
        zero_index = data.find(b"\x00", cursor)
        run_end = len(data) if zero_index == -1 else zero_index
        run = data[cursor:run_end]
        while len(run) >= 254:
            encoded.append(0xFF)
            encoded.extend(run[:254])
            run = run[254:]
        if zero_index == -1:
            if run or not encoded:
                encoded.append(len(run) + 1)
                encoded.extend(run)
            return bytes(encoded)
        encoded.append(len(run) + 1)
        encoded.extend(run)
        cursor = zero_index + 1
    encoded.append(1)
    return bytes(encoded)


for length in range(9):
    for values in product((0x00, 0x01, 0xFF), repeat=length):
        raw = bytes(values)
        encoded = encode_canonical_cobs(raw)
        assert encoded == zero_run_oracle(raw)
        assert 0 not in encoded
        assert decode_canonical_cobs(encoded) == raw

for run_length in (253, 254, 255, 508):
    for raw in (b"x" * run_length, b"x" * run_length + b"\x00"):
        encoded = encode_canonical_cobs(raw)
        assert encoded == zero_run_oracle(raw)
        assert decode_canonical_cobs(encoded) == raw

rng = Random(0xC0_B5)
for _ in range(2_000):
    raw = rng.randbytes(rng.randrange(1_025))
    encoded = encode_canonical_cobs(raw)
    assert decode_canonical_cobs(encoded) == raw

rejected = 0
for malformed in (b"", b"\x00", b"\x02", b"\xff" + b"x" * 253, b"\xff" + b"x" * 254 + b"\x01"):
    try:
        decode_canonical_cobs(malformed)
    except ValueError:
        rejected += 1

assert rejected == 5
```

## Trade-offs and Limitations

Encoding and decoding take `O(N)` time and retain `O(N)` output memory. Every
254 raw nonzero bytes add at most one code byte, so the expansion is small and
bounded. Canonical re-encoding gives the decoder a second linear pass in
exchange for rejecting redundant spellings.

The returned bytes do not include a trailing zero delimiter and the decoder
does not process a stream or recover synchronization after damage. COBS is not
compression, encryption, authentication, a checksum, or corruption recovery.
This contract implements canonical ordinary COBS, not COBS/R or another
wire-compatible variant.

## Related Snippets

<!-- catalog:related:start -->
- [Encode and Decode a Bounded Full-Block Z85 Frame](encode-and-decode-a-bounded-full-block-z85-frame.md)
- [Encode and Decode One Canonical Size-Capped Netstring](../networking-protocols/encode-and-decode-one-canonical-size-capped-netstring.md)
- [Read One Bounded Async Byte Field with Either Line Feed or NUL Terminator](../networking-protocols/read-one-bounded-async-byte-field-with-either-line-feed-or-nul-terminator.md)
<!-- catalog:related:end -->
