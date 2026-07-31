---
title: "Pack and Restore a Bounded Strictly Increasing Integer Sequence with Elias–Fano"
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
  - pack-and-unpack-a-bounded-boolean-tuple-with-explicit-bit-length.md
  - ../algorithms-data-structures/maintain-a-bounded-integer-multiset-with-fenwick-rank-and-select.md
  - ../algorithms-data-structures/rank-and-unrank-index-combinations-in-itertools-combinations-order.md
---

# Pack and Restore a Bounded Strictly Increasing Integer Sequence with Elias–Fano

## Idea and Problem

Pack a bounded strictly increasing non-negative integer sequence by separating fixed-width lower bits from monotone upper bits, then restore it after validating the exact canonical layout.

For `N` values below the exclusive upper bound `U`, this profile chooses
`L = floor(log2(U / N))` lower bits. The one for value `values[i]` is placed at
`(values[i] >> L) + i`. Both payloads number bits from the least-significant bit
of byte zero, making the in-memory layout deterministic and reversible.

## When to Use

Use this layout when a bounded, sorted set of large non-negative integers needs
a compact immutable representation that can later be restored in full. It is
especially useful when the universe is substantially larger than the number of
stored values and byte-level fixtures must be reproducible.

Use a format-specific succinct data structure when rank, select, predecessor,
random access, streaming, or interoperability is required. This snippet defines
only one closed packing profile and full-sequence restoration; it is not a file
format or a search index.

## Implementation

```python
from dataclasses import dataclass

_MAX_ELIAS_FANO_ITEMS = 65_536
_MAX_ELIAS_FANO_VALUE = (1 << 63) - 1


@dataclass(frozen=True, slots=True)
class EliasFanoRecord:
    count: int
    upper_bound: int
    lower_bytes: bytes
    upper_bytes: bytes


def _packed_byte_length(bit_count: int) -> int:
    return (bit_count + 7) // 8


def _write_lsb_bits(
    payload: bytearray,
    start: int,
    value: int,
    width: int,
) -> None:
    for offset in range(width):
        if value & (1 << offset):
            bit_index = start + offset
            payload[bit_index // 8] |= 1 << (bit_index % 8)


def _read_lsb_bits(payload: bytes, start: int, width: int) -> int:
    value = 0
    for offset in range(width):
        bit_index = start + offset
        if payload[bit_index // 8] & (1 << (bit_index % 8)):
            value |= 1 << offset
    return value


def _require_zero_padding(payload: bytes, bit_count: int, field: str) -> None:
    used_in_last_byte = bit_count % 8
    if used_in_last_byte and payload[-1] >> used_in_last_byte:
        raise ValueError(f"{field} has non-zero padding bits")


def pack_elias_fano(values: tuple[int, ...]) -> EliasFanoRecord:
    """Pack one bounded strictly increasing unsigned integer tuple."""
    if type(values) is not tuple:
        raise TypeError("values must be an exact tuple")
    count = len(values)
    if count > _MAX_ELIAS_FANO_ITEMS:
        raise ValueError("values contains more than 65536 items")

    previous = -1
    for index, value in enumerate(values):
        if type(value) is not int:
            raise TypeError(f"values[{index}] must be an exact integer")
        if not 0 <= value <= _MAX_ELIAS_FANO_VALUE:
            raise ValueError(f"values[{index}] is outside the supported range")
        if value <= previous:
            raise ValueError("values must be strictly increasing")
        previous = value

    if count == 0:
        return EliasFanoRecord(0, 0, b"", b"")

    upper_bound = values[-1] + 1
    lower_width = max(0, (upper_bound // count).bit_length() - 1)
    lower_bit_count = count * lower_width
    upper_bit_count = ((upper_bound - 1) >> lower_width) + count
    lower_payload = bytearray(_packed_byte_length(lower_bit_count))
    upper_payload = bytearray(_packed_byte_length(upper_bit_count))
    lower_mask = (1 << lower_width) - 1

    for index, value in enumerate(values):
        _write_lsb_bits(
            lower_payload,
            index * lower_width,
            value & lower_mask,
            lower_width,
        )
        upper_position = (value >> lower_width) + index
        upper_payload[upper_position // 8] |= 1 << (upper_position % 8)

    return EliasFanoRecord(
        count=count,
        upper_bound=upper_bound,
        lower_bytes=bytes(lower_payload),
        upper_bytes=bytes(upper_payload),
    )


def restore_elias_fano(record: EliasFanoRecord) -> tuple[int, ...]:
    """Validate and restore one record produced by pack_elias_fano."""
    if type(record) is not EliasFanoRecord:
        raise TypeError("record must be an exact EliasFanoRecord")
    if type(record.count) is not int:
        raise TypeError("record.count must be an exact integer")
    if type(record.upper_bound) is not int:
        raise TypeError("record.upper_bound must be an exact integer")
    if type(record.lower_bytes) is not bytes:
        raise TypeError("record.lower_bytes must be exact bytes")
    if type(record.upper_bytes) is not bytes:
        raise TypeError("record.upper_bytes must be exact bytes")
    if not 0 <= record.count <= _MAX_ELIAS_FANO_ITEMS:
        raise ValueError("record.count is outside the supported range")

    if record.count == 0:
        if record != EliasFanoRecord(0, 0, b"", b""):
            raise ValueError("an empty record must use the canonical empty form")
        return ()

    if not 1 <= record.upper_bound <= _MAX_ELIAS_FANO_VALUE + 1:
        raise ValueError("record.upper_bound is outside the supported range")
    if record.upper_bound < record.count:
        raise ValueError("record.upper_bound cannot contain record.count values")

    lower_width = max(
        0,
        (record.upper_bound // record.count).bit_length() - 1,
    )
    lower_bit_count = record.count * lower_width
    upper_bit_count = ((record.upper_bound - 1) >> lower_width) + record.count
    if len(record.lower_bytes) != _packed_byte_length(lower_bit_count):
        raise ValueError("record.lower_bytes has the wrong length")
    if len(record.upper_bytes) != _packed_byte_length(upper_bit_count):
        raise ValueError("record.upper_bytes has the wrong length")
    _require_zero_padding(record.lower_bytes, lower_bit_count, "record.lower_bytes")
    _require_zero_padding(record.upper_bytes, upper_bit_count, "record.upper_bytes")

    restored: list[int] = []
    one_index = 0
    for position in range(upper_bit_count):
        if not record.upper_bytes[position // 8] & (1 << (position % 8)):
            continue
        if one_index == record.count:
            raise ValueError("record.upper_bytes contains too many one bits")

        upper_value = position - one_index
        lower_value = _read_lsb_bits(
            record.lower_bytes,
            one_index * lower_width,
            lower_width,
        )
        value = (upper_value << lower_width) | lower_value
        if value >= record.upper_bound:
            raise ValueError("record decodes a value outside its upper bound")
        if restored and value <= restored[-1]:
            raise ValueError("record does not decode to strictly increasing values")
        restored.append(value)
        one_index += 1

    if one_index != record.count:
        raise ValueError("record.upper_bytes does not contain record.count one bits")
    if restored[-1] + 1 != record.upper_bound:
        raise ValueError("record.upper_bound is not one past the final value")

    result = tuple(restored)
    if pack_elias_fano(result) != record:
        raise ValueError("record is not in canonical Elias-Fano form")
    return result
```

## Example

```python
def reference_elias_fano(values: tuple[int, ...]) -> EliasFanoRecord:
    if not values:
        return EliasFanoRecord(0, 0, b"", b"")

    count = len(values)
    upper_bound = values[-1] + 1
    lower_width = max(0, (upper_bound // count).bit_length() - 1)
    lower_mask = (1 << lower_width) - 1
    lower_word = sum(
        (value & lower_mask) << (index * lower_width) for index, value in enumerate(values)
    )
    upper_word = sum(1 << ((value >> lower_width) + index) for index, value in enumerate(values))
    lower_bit_count = count * lower_width
    upper_bit_count = ((upper_bound - 1) >> lower_width) + count
    return EliasFanoRecord(
        count,
        upper_bound,
        lower_word.to_bytes(_packed_byte_length(lower_bit_count), "little"),
        upper_word.to_bytes(_packed_byte_length(upper_bit_count), "little"),
    )


def exercise_elias_fano_examples() -> tuple[int, int, int]:
    checked = 0
    fingerprint = 0xCBF2_9CE4_8422_2325
    for selection_mask in range(1 << 9):
        values = tuple(value for value in range(9) if selection_mask & (1 << value))
        record = pack_elias_fano(values)
        assert record == reference_elias_fano(values)
        assert restore_elias_fano(record) == values
        record_word = (
            record.count
            ^ (record.upper_bound << 17)
            ^ (int.from_bytes(record.lower_bytes, "little") << 29)
            ^ (int.from_bytes(record.upper_bytes, "little") << 41)
        )
        fingerprint ^= record_word
        fingerprint = (fingerprint * 0x100_0000_01B3) & ((1 << 64) - 1)
        checked += 1

    sample = pack_elias_fano((1, 4, 8))
    assert sample == EliasFanoRecord(3, 9, b"\x01", b"I")
    assert restore_elias_fano(EliasFanoRecord(3, 9, b"\x01", b"E")) == (
        1,
        2,
        8,
    )

    for boundary_values in (
        (_MAX_ELIAS_FANO_VALUE,),
        (0, _MAX_ELIAS_FANO_VALUE),
        tuple(range(_MAX_ELIAS_FANO_ITEMS)),
    ):
        assert restore_elias_fano(pack_elias_fano(boundary_values)) == boundary_values

    rejected_inputs = 0
    for invalid_values in (
        [0],
        (True,),
        (-1,),
        (_MAX_ELIAS_FANO_VALUE + 1,),
        (0, 0),
        tuple(range(_MAX_ELIAS_FANO_ITEMS + 1)),
    ):
        try:
            pack_elias_fano(invalid_values)  # type: ignore[arg-type]
        except (TypeError, ValueError):
            rejected_inputs += 1
    assert rejected_inputs == 6

    malformed = (
        (object(), TypeError),
        (EliasFanoRecord(True, 0, b"", b""), TypeError),
        (EliasFanoRecord(1, True, b"", b""), TypeError),
        (EliasFanoRecord(_MAX_ELIAS_FANO_ITEMS + 1, 1, b"", b""), ValueError),
        (EliasFanoRecord(0, 1, b"", b""), ValueError),
        (EliasFanoRecord(1, 0, b"", b""), ValueError),
        (EliasFanoRecord(1, _MAX_ELIAS_FANO_VALUE + 2, b"", b""), ValueError),
        (EliasFanoRecord(3, 2, b"", b""), ValueError),
        (EliasFanoRecord(3, 9, bytearray(b"\x01"), b"I"), TypeError),
        (EliasFanoRecord(3, 9, b"\x01", bytearray(b"I")), TypeError),
        (EliasFanoRecord(3, 9, b"", b"I"), ValueError),
        (EliasFanoRecord(3, 9, b"\x81", b"I"), ValueError),
        (EliasFanoRecord(3, 9, b"\x01", b""), ValueError),
        (EliasFanoRecord(3, 9, b"\x01", b"\xc9"), ValueError),
        (EliasFanoRecord(3, 9, b"\x01", b"\x09"), ValueError),
        (EliasFanoRecord(3, 9, b"\x01", b"C"), ValueError),
        (EliasFanoRecord(3, 9, b"\x01", b")"), ValueError),
    )
    rejected = 0
    for malformed_record, expected_error in malformed:
        try:
            restore_elias_fano(malformed_record)
        except expected_error:
            rejected += 1

    return checked, rejected, fingerprint


assert exercise_elias_fano_examples() == (512, 17, 1_290_937_834_757_224_021)
```

## Trade-offs and Limitations

For `N` values, lower width `L`, and
`H = ((U - 1) >> L) + N`, packing and restoration use
`O(N * L + H)` bit work. The two payloads occupy exactly
`ceil(N * L / 8) + ceil(H / 8)` bytes; the choice of `L` guarantees
`H < 3N`. Restoration additionally returns `O(N)` Python integer objects, and
the record, tuple, byte arrays, and interpreter objects add overhead beyond the
packed payload sizes.

The input is restricted to at most 65,536 exact integers in
`0..2**63 - 1`. The layout stores `U` as one past the final value, validates
exact byte lengths and zero padding, and rejects records that do not reproduce
exactly after decoding and re-encoding. It provides no rank, select,
predecessor, incremental update, compression container, checksum, or
interoperability guarantee.

## Related Snippets

<!-- catalog:related:start -->
- [Pack and Unpack a Bounded Boolean Tuple with Explicit Bit Length](pack-and-unpack-a-bounded-boolean-tuple-with-explicit-bit-length.md)
- [Maintain a Bounded Integer Multiset with Fenwick Rank and Select](../algorithms-data-structures/maintain-a-bounded-integer-multiset-with-fenwick-rank-and-select.md)
- [Rank and Unrank Index Combinations in itertools.combinations Order](../algorithms-data-structures/rank-and-unrank-index-combinations-in-itertools-combinations-order.md)
<!-- catalog:related:end -->
