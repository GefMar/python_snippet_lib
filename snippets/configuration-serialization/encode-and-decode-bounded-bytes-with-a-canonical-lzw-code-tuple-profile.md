---
title: "Encode and Decode Bounded Bytes with a Canonical LZW Code-Tuple Profile"
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
  - encode-and-decode-a-bounded-byte-block-with-a-canonical-greedy-lz77-profile.md
  - derive-deterministic-huffman-code-lengths-from-bounded-byte-frequencies.md
  - transform-and-restore-bounded-bytes-with-a-canonical-move-to-front-profile.md
---

# Encode and Decode Bounded Bytes with a Canonical LZW Code-Tuple Profile

## Idea and Problem

Replace repeated byte phrases with codes from an evolving shared dictionary and restore only the canonical code tuple defined by one bounded LZW profile.

The initial codes `0..255` represent individual byte values. During encoding,
each absent transition from the current phrase through one byte receives the
next code, while the current phrase's existing code is emitted. Decoding grows
the same dictionary from earlier output and handles the one special case where
the next code describes the previous phrase followed by its first byte.

The dictionary stops growing after code 4,095. There are no clear, reset, or
end codes and no bit-packing convention, so the returned integer tuple is a
complete and deterministic abstract profile rather than an interchange format.

## When to Use

Use this profile for compression-algorithm fixtures, demonstrations of an
adaptive phrase dictionary, or tests that need stable abstract LZW codes and a
strict decoder. It is useful when the 64-KiB input/output cap and an explicit
non-interoperable code-tuple representation are acceptable.

Use a standard compression library for stored or exchanged data. A real file
format must also specify code widths, packing order, dictionary resets,
framing, versioning, and often checksums. Use LZ77 when backward distances and
lengths are the intended representation rather than persistent phrase codes.

## Implementation

```python
_MAX_LZW_BYTES = 65_536
_MAX_LZW_CODES = 65_536
_MAX_LZW_CODE = 4_095


def lzw_encode(data: bytes) -> tuple[int, ...]:
    """Return canonical LZW codes for one bounded byte string."""
    if type(data) is not bytes:
        raise TypeError("data must be exact bytes")
    if len(data) > _MAX_LZW_BYTES:
        raise ValueError("data exceeds 65,536 bytes")
    if not data:
        return ()

    transitions: dict[tuple[int, int], int] = {}
    next_code = 256
    current_code = data[0]
    codes: list[int] = []

    for value in data[1:]:
        candidate = transitions.get((current_code, value))
        if candidate is not None:
            current_code = candidate
            continue

        codes.append(current_code)
        if next_code <= _MAX_LZW_CODE:
            transitions[(current_code, value)] = next_code
            next_code += 1
        current_code = value

    codes.append(current_code)
    return tuple(codes)


def lzw_decode(codes: tuple[int, ...]) -> bytes:
    """Decode one complete canonical code tuple under the bounded profile."""
    if type(codes) is not tuple:
        raise TypeError("codes must be an exact tuple")
    if len(codes) > _MAX_LZW_CODES:
        raise ValueError("code count exceeds 65,536")
    for index, code in enumerate(codes):
        if type(code) is not int:
            raise TypeError(f"codes[{index}] must be an exact integer")
        if not 0 <= code <= _MAX_LZW_CODE:
            raise ValueError(f"codes[{index}] is outside 0..4,095")
    if not codes:
        return b""
    if codes[0] >= 256:
        raise ValueError("the first code must belong to the byte alphabet")

    dictionary = [bytes((value,)) for value in range(256)]
    previous = dictionary[codes[0]]
    output = bytearray(previous)

    for index, code in enumerate(codes[1:], start=1):
        if code < len(dictionary):
            phrase = dictionary[code]
        elif code == len(dictionary) and len(dictionary) <= _MAX_LZW_CODE:
            phrase = previous + previous[:1]
        else:
            raise ValueError(f"codes[{index}] references an undefined phrase")

        if len(output) + len(phrase) > _MAX_LZW_BYTES:
            raise ValueError("decoded output exceeds 65,536 bytes")
        output.extend(phrase)

        if len(dictionary) <= _MAX_LZW_CODE:
            dictionary.append(previous + phrase[:1])
        previous = phrase

    decoded = bytes(output)
    if lzw_encode(decoded) != codes:
        raise ValueError("codes are not canonical for their decoded bytes")
    return decoded
```

## Example

```python
from itertools import product


def slow_lzw_encode(data: bytes) -> tuple[tuple[int, ...], int]:
    if not data:
        return (), 256
    dictionary = {bytes((value,)): value for value in range(256)}
    next_code = 256
    phrase = data[:1]
    result: list[int] = []
    for value in data[1:]:
        candidate = phrase + bytes((value,))
        if candidate in dictionary:
            phrase = candidate
            continue
        result.append(dictionary[phrase])
        if next_code <= _MAX_LZW_CODE:
            dictionary[candidate] = next_code
            next_code += 1
        phrase = bytes((value,))
    result.append(dictionary[phrase])
    return tuple(result), next_code


checked = 0
for length in range(9):
    for values in product(range(3), repeat=length):
        sample = bytes(values)
        expected, _ = slow_lzw_encode(sample)
        actual = lzw_encode(sample)
        assert actual == expected
        assert lzw_decode(actual) == sample
        checked += 1

special = b"ABABABA"
special_codes = lzw_encode(special)
assert special_codes == (65, 66, 256, 258)
assert lzw_decode(special_codes) == special

maximum_data = bytes((index * 73 + index // 251) % 256 for index in range(_MAX_LZW_BYTES))
maximum_expected, maximum_next_code = slow_lzw_encode(maximum_data)
maximum_codes = lzw_encode(maximum_data)
maximum_decoded = lzw_decode(maximum_codes)


class TupleSubclass(tuple):
    pass


rejected = 0
invalid_actions = (
    lambda: lzw_encode(bytearray(b"a")),
    lambda: lzw_encode(b"x" * (_MAX_LZW_BYTES + 1)),
    lambda: lzw_decode(TupleSubclass((65,))),
    lambda: lzw_decode((True,)),
    lambda: lzw_decode((-1,)),
    lambda: lzw_decode((_MAX_LZW_CODE + 1,)),
    lambda: lzw_decode((256,)),
    lambda: lzw_decode((65, 300)),
    lambda: lzw_decode((65, 66, 65, 66)),
    lambda: lzw_decode((0,) * (_MAX_LZW_CODES + 1)),
)
for invalid_action in invalid_actions:
    try:
        invalid_action()
    except (TypeError, ValueError):
        rejected += 1

assert (
    checked == 9_841
    and lzw_encode(b"") == ()
    and lzw_decode(()) == b""
    and maximum_codes == maximum_expected
    and maximum_next_code == _MAX_LZW_CODE + 1
    and maximum_decoded == maximum_data
    and rejected == len(invalid_actions)
)
```

## Trade-offs and Limitations

Encoding performs one expected-constant-time transition lookup per input byte
and stores at most 3,840 integer-pair transitions. Decoding and its canonical
re-encoding visit the bounded output linearly. Materialized decoder phrases
can contain `O(D**2)` bytes in the worst case for dictionary size `D`, even
though the dictionary is capped at 4,096 entries and output at 65,536 bytes.

The decoder accepts only streams reproduced exactly by this encoder. It
rejects otherwise decodable alternative phrase decompositions, undefined
codes, output expansion beyond the cap, subclasses, and non-integer codes. The
`code == next_code` self-extension case is legal only before the dictionary is
full.

This profile does not pack codes, vary their bit width, reset a full
dictionary, emit control codes, stream across blocks, detect corruption, or
claim compatibility with GIF, TIFF, Unix `compress`, or another LZW format.

## Related Snippets

<!-- catalog:related:start -->
- [Encode and Decode a Bounded Byte Block with a Canonical Greedy LZ77 Profile](encode-and-decode-a-bounded-byte-block-with-a-canonical-greedy-lz77-profile.md)
- [Derive Deterministic Huffman Code Lengths from Bounded Byte Frequencies](derive-deterministic-huffman-code-lengths-from-bounded-byte-frequencies.md)
- [Transform and Restore Bounded Bytes with a Canonical Move-to-Front Profile](transform-and-restore-bounded-bytes-with-a-canonical-move-to-front-profile.md)
<!-- catalog:related:end -->
