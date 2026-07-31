---
title: "Encode and Decode a Bounded Byte Block with a Canonical Greedy LZ77 Profile"
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
  - transform-and-restore-bounded-bytes-with-a-sentinel-free-burrows-wheeler-transform.md
  - derive-deterministic-huffman-code-lengths-from-bounded-byte-frequencies.md
  - ../data-processing/aggregate-consecutive-values-into-weighted-runs.md
---

# Encode and Decode a Bounded Byte Block with a Canonical Greedy LZ77 Profile

## Idea and Problem

Replace repeated byte substrings with deterministic backward-reference tokens and restore them under one small, fully bounded LZ77 profile.

At each input position, the encoder searches the preceding 4,096 bytes for the
longest match of length 3 through 258. Equal-length matches choose the smallest
backward distance, so the same byte block always produces the same abstract
token tuple. A literal carries one byte when no qualifying match exists.

A reference may overlap the bytes it is producing. Copying one byte at a time
makes a short history such as one `a` expand into a long run, matching the
defined relation `data[position + k] == data[position + k - distance]`.

## When to Use

Use this profile to demonstrate or test dictionary-style parsing, inspect
repeated-substring choices, build deterministic compression fixtures, or feed
an explicitly separate token serializer. It is most useful when the 64-KiB
input cap and deliberately simple worst-case search are acceptable.

Use a standard compression library for stored or exchanged data. A production
format must define bit packing, end markers, versioning, and usually entropy
coding in addition to match selection. Use a suffix structure or specialized
match finder when large inputs need predictable encoding performance.

## Implementation

```python
from dataclasses import dataclass
from itertools import product
from random import Random

_MAX_LZ77_BYTES = 65_536
_MAX_LZ77_TOKENS = 65_536
_LZ77_WINDOW = 4_096
_LZ77_MIN_MATCH = 3
_LZ77_MAX_MATCH = 258


@dataclass(frozen=True, slots=True)
class Lz77Literal:
    value: int


@dataclass(frozen=True, slots=True)
class Lz77BackReference:
    distance: int
    length: int


Lz77Token = Lz77Literal | Lz77BackReference


def encode_greedy_lz77(data: bytes) -> tuple[Lz77Token, ...]:
    """Return deterministic abstract tokens for one bounded byte block."""
    if type(data) is not bytes:
        raise TypeError("data must be exact bytes")
    if len(data) > _MAX_LZ77_BYTES:
        raise ValueError("data exceeds 65,536 bytes")

    prefix_positions: dict[bytes, list[int]] = {}
    tokens: list[Lz77Token] = []
    position = 0
    while position < len(data):
        maximum_length = min(_LZ77_MAX_MATCH, len(data) - position)
        best_length = 0
        best_distance = 0

        if maximum_length >= _LZ77_MIN_MATCH:
            prefix = data[position : position + _LZ77_MIN_MATCH]
            for candidate in reversed(prefix_positions.get(prefix, ())):
                distance = position - candidate
                if distance > _LZ77_WINDOW:
                    break
                length = _LZ77_MIN_MATCH
                while (
                    length < maximum_length
                    and data[position + length] == data[position + length - distance]
                ):
                    length += 1
                if length > best_length:
                    best_length = length
                    best_distance = distance
                    if length == maximum_length:
                        break

        if best_length >= _LZ77_MIN_MATCH:
            tokens.append(Lz77BackReference(best_distance, best_length))
            consumed = best_length
        else:
            tokens.append(Lz77Literal(data[position]))
            consumed = 1

        for indexed_position in range(position, position + consumed):
            if indexed_position + _LZ77_MIN_MATCH <= len(data):
                prefix = data[indexed_position : indexed_position + _LZ77_MIN_MATCH]
                prefix_positions.setdefault(prefix, []).append(indexed_position)
        position += consumed

    return tuple(tokens)


def decode_greedy_lz77(tokens: tuple[Lz77Token, ...]) -> bytes:
    """Validate and decode abstract tokens without returning partial output."""
    if type(tokens) is not tuple:
        raise TypeError("tokens must be an exact tuple")
    if len(tokens) > _MAX_LZ77_TOKENS:
        raise ValueError("token count exceeds 65,536")

    output = bytearray()
    for index, token in enumerate(tokens):
        if type(token) is Lz77Literal:
            if type(token.value) is not int:
                raise TypeError(f"tokens[{index}].value must be an exact integer")
            if not 0 <= token.value <= 255:
                raise ValueError(f"tokens[{index}].value is outside byte range")
            if len(output) == _MAX_LZ77_BYTES:
                raise ValueError("decoded output exceeds 65,536 bytes")
            output.append(token.value)
        elif type(token) is Lz77BackReference:
            if type(token.distance) is not int or type(token.length) is not int:
                raise TypeError(f"tokens[{index}] fields must be exact integers")
            if not 1 <= token.distance <= _LZ77_WINDOW:
                raise ValueError(f"tokens[{index}].distance is outside 1..4,096")
            if not _LZ77_MIN_MATCH <= token.length <= _LZ77_MAX_MATCH:
                raise ValueError(f"tokens[{index}].length is outside 3..258")
            if token.distance > len(output):
                raise ValueError(f"tokens[{index}] references unavailable history")
            if len(output) + token.length > _MAX_LZ77_BYTES:
                raise ValueError("decoded output exceeds 65,536 bytes")
            for _ in range(token.length):
                output.append(output[-token.distance])
        else:
            raise TypeError(f"tokens[{index}] has an unknown exact token type")
    return bytes(output)
```

## Example

```python
def brute_greedy_tokens(data: bytes) -> tuple[Lz77Token, ...]:
    tokens: list[Lz77Token] = []
    position = 0
    while position < len(data):
        maximum_length = min(_LZ77_MAX_MATCH, len(data) - position)
        best_length = 0
        best_distance = 0
        for distance in range(1, min(_LZ77_WINDOW, position) + 1):
            length = 0
            while (
                length < maximum_length
                and data[position + length] == data[position + length - distance]
            ):
                length += 1
            if length > best_length:
                best_length = length
                best_distance = distance
        if best_length >= _LZ77_MIN_MATCH:
            tokens.append(Lz77BackReference(best_distance, best_length))
            position += best_length
        else:
            tokens.append(Lz77Literal(data[position]))
            position += 1
    return tuple(tokens)


exhaustive_blocks = 0
for length in range(13):
    for symbols in product((0, 1), repeat=length):
        data = bytes(symbols)
        encoded = encode_greedy_lz77(data)
        assert encoded == brute_greedy_tokens(data)
        assert decode_greedy_lz77(encoded) == data
        exhaustive_blocks += 1

nearest_tie_data = b"abcXabcYabc"
nearest_tie_tokens = encode_greedy_lz77(nearest_tie_data)
overlapping_run = b"a" * 1_024
overlapping_tokens = encode_greedy_lz77(overlapping_run)

rng = Random(31)
for _ in range(500):
    data = bytes(rng.randrange(8) for _ in range(rng.randrange(513)))
    assert decode_greedy_lz77(encode_greedy_lz77(data)) == data

malformed_streams = (
    (Lz77BackReference(1, 3),),
    (Lz77Literal(256),),
    (Lz77Literal(0), Lz77BackReference(0, 3)),
    (Lz77Literal(0), Lz77BackReference(1, 2)),
    (object(),),
    (Lz77Literal(0),) + (Lz77BackReference(1, 258),) * 255,
)
for malformed in malformed_streams:
    try:
        decode_greedy_lz77(malformed)
    except (TypeError, ValueError):
        pass
    else:
        raise AssertionError("accepted malformed or oversized tokens")

maximum_data = bytes(range(256)) * 256
maximum_tokens = encode_greedy_lz77(maximum_data)

assert (
    nearest_tie_tokens[-1],
    overlapping_tokens[:2],
    decode_greedy_lz77(overlapping_tokens) == overlapping_run,
    exhaustive_blocks,
    len(maximum_data),
    decode_greedy_lz77(maximum_tokens) == maximum_data,
) == (
    Lz77BackReference(4, 3),
    (Lz77Literal(ord("a")), Lz77BackReference(1, 258)),
    True,
    8_191,
    _MAX_LZ77_BYTES,
    True,
)
```

## Trade-offs and Limitations

The bounded match finder retains three-byte prefix positions but may still
compare every candidate in the window for up to the maximum match length. Its
conservative worst-case bound is therefore `O(N*W*M)` time for input length
`N`, window `W`, and match cap `M`; tokens and prefix positions use `O(N)`
state. The decoder is linear in decoded bytes and uses the complete output
buffer as history.

Greedy longest matches do not guarantee the fewest tokens or the smallest
eventual encoded representation. The nearest-distance tie rule is merely this
profile's deterministic choice. Keeping every indexed position also favors
clarity and auditability over the bounded-memory match structures used by
production compressors.

Tokens are abstract Python values, not a byte format. There is no bit packing,
entropy coding, checksum, streaming state, dictionary reset, interoperability,
DEFLATE compatibility, or guarantee that the representation is smaller than
the input. Untrusted serialized data still needs a separately specified parser
before these tokens can be constructed.

## Related Snippets

<!-- catalog:related:start -->
- [Transform and Restore Bounded Bytes with a Sentinel-Free Burrows-Wheeler Transform](transform-and-restore-bounded-bytes-with-a-sentinel-free-burrows-wheeler-transform.md)
- [Derive Deterministic Huffman Code Lengths from Bounded Byte Frequencies](derive-deterministic-huffman-code-lengths-from-bounded-byte-frequencies.md)
- [Aggregate Consecutive Values into Weighted Runs](../data-processing/aggregate-consecutive-values-into-weighted-runs.md)
<!-- catalog:related:end -->
