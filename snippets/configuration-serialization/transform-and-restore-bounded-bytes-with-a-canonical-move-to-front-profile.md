---
title: "Transform and Restore Bounded Bytes with a Canonical Move-to-Front Profile"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - serialization
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - transform-and-restore-bounded-bytes-with-a-sentinel-free-burrows-wheeler-transform.md
  - derive-deterministic-huffman-code-lengths-from-bounded-byte-frequencies.md
  - encode-and-decode-bounded-bytes-with-a-canonical-packbits-profile.md
---

# Transform and Restore Bounded Bytes with a Canonical Move-to-Front Profile

## Idea and Problem

Replace each byte with its current recency rank under one completely specified, reversible move-to-front transform.

The alphabet begins in numeric byte order from 0 through 255. Encoding emits
the zero-based position of the next symbol, then moves that symbol to the front
of the alphabet. Decoding interprets each input byte as a position and performs
the same move after recovering the symbol.

Recently repeated symbols therefore become small ranks. That pattern can help
a later entropy or run-length coder, especially after a transform that groups
similar contexts, while the move-to-front step itself remains lossless.

## When to Use

Use this closed profile as an in-memory preprocessing stage when both sides
agree on the fixed full-byte alphabet and another stage will encode the rank
distribution. It is useful for experiments, deterministic fixtures, and small
compression pipelines paired with a Burrows-Wheeler transform.

Do not use the output as if it were compressed: it has exactly the same byte
length as the input. A custom alphabet, reset marker, streaming state, or
external wire format needs its own explicit profile. For latency-sensitive
large data, use a native or specialized implementation rather than repeated
Python list movement.

## Implementation

```python
_MAX_MOVE_TO_FRONT_BYTES = 65_536


def _validate_move_to_front_bytes(value: object, field: str) -> bytes:
    if type(value) is not bytes:
        raise TypeError(f"{field} must be exact bytes")
    if len(value) > _MAX_MOVE_TO_FRONT_BYTES:
        raise ValueError(f"{field} exceeds 65,536 bytes")
    return value


def move_to_front_encode(data: bytes) -> bytes:
    """Encode bytes as ranks in the fixed 0..255 move-to-front alphabet."""
    checked_data = _validate_move_to_front_bytes(data, "data")
    alphabet = list(range(256))
    encoded = bytearray()
    for symbol in checked_data:
        rank = alphabet.index(symbol)
        encoded.append(rank)
        if rank:
            alphabet.pop(rank)
            alphabet.insert(0, symbol)
    return bytes(encoded)


def move_to_front_decode(indices: bytes) -> bytes:
    """Decode ranks under the fixed 0..255 move-to-front alphabet."""
    checked_indices = _validate_move_to_front_bytes(indices, "indices")
    alphabet = list(range(256))
    decoded = bytearray()
    for rank in checked_indices:
        symbol = alphabet[rank]
        decoded.append(symbol)
        if rank:
            alphabet.pop(rank)
            alphabet.insert(0, symbol)
    return bytes(decoded)
```

## Example

```python
from itertools import product
from random import Random


def literal_encode(data: bytes) -> bytes:
    ordering = tuple(range(256))
    ranks: list[int] = []
    for symbol in data:
        rank = ordering.index(symbol)
        ranks.append(rank)
        ordering = (symbol, *ordering[:rank], *ordering[rank + 1 :])
    return bytes(ranks)


def literal_decode(indices: bytes) -> bytes:
    ordering = tuple(range(256))
    symbols: list[int] = []
    for rank in indices:
        symbol = ordering[rank]
        symbols.append(symbol)
        ordering = (symbol, *ordering[:rank], *ordering[rank + 1 :])
    return bytes(symbols)


exhaustive_checked = 0
for size in range(8):
    for values in product(range(4), repeat=size):
        data = bytes(values)
        expected = literal_encode(data)
        assert move_to_front_encode(data) == expected
        assert move_to_front_decode(expected) == literal_decode(expected) == data
        exhaustive_checked += 1

generator = Random(31_337)
random_checked = 0
for _ in range(300):
    data = generator.randbytes(generator.randrange(513))
    encoded = move_to_front_encode(data)
    assert encoded == literal_encode(data)
    assert move_to_front_decode(encoded) == data
    random_checked += 1

all_bytes = bytes(range(256))
maximum_input = all_bytes * 256
maximum_encoded = move_to_front_encode(maximum_input)


def rejects_encode(value: object) -> bool:
    try:
        move_to_front_encode(value)  # type: ignore[arg-type]
    except (TypeError, ValueError):
        return True
    return False


def rejects_decode(value: object) -> bool:
    try:
        move_to_front_decode(value)  # type: ignore[arg-type]
    except (TypeError, ValueError):
        return True
    return False


assert (
    exhaustive_checked,
    random_checked,
    move_to_front_encode(b"bananaaa"),
    move_to_front_decode(bytes.fromhex("62626e0101010000")),
    move_to_front_encode(b"aaaa"),
    len(maximum_encoded),
    move_to_front_decode(maximum_encoded) == maximum_input,
    rejects_encode(bytearray()),
    rejects_decode(memoryview(b"")),
    rejects_encode(b"\x00" * (_MAX_MOVE_TO_FRONT_BYTES + 1)),
) == (
    21_845,
    300,
    bytes.fromhex("62626e0101010000"),
    b"bananaaa",
    b"a\x00\x00\x00",
    _MAX_MOVE_TO_FRONT_BYTES,
    True,
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

For `N` bytes, both directions perform `O(256*N)` bounded alphabet search or
movement and use `O(N + 256)` space including the result. Because the alphabet
has a fixed size of 256, this is asymptotically `O(N)`, but the Python-level
list operations still have a meaningful constant cost.

Every possible byte in the encoded input is a valid rank, because the alphabet
always contains exactly 256 symbols. The decoder therefore has no malformed
rank spelling to reject; only the exact input type and size cap can be invalid.
The transform has no checksum, framing, corruption detection, or recovery.

This profile never resets or changes its initial numeric alphabet. It does not
pack small ranks into fewer bits and does not compress by itself. Streaming,
custom alphabets, entropy coding, and interoperability with another
move-to-front convention are deliberately outside its scope.

## Related Snippets

<!-- catalog:related:start -->
- [Transform and Restore Bounded Bytes with a Sentinel-Free Burrows-Wheeler Transform](transform-and-restore-bounded-bytes-with-a-sentinel-free-burrows-wheeler-transform.md)
- [Derive Deterministic Huffman Code Lengths from Bounded Byte Frequencies](derive-deterministic-huffman-code-lengths-from-bounded-byte-frequencies.md)
- [Encode and Decode Bounded Bytes with a Canonical PackBits Profile](encode-and-decode-bounded-bytes-with-a-canonical-packbits-profile.md)
<!-- catalog:related:end -->
