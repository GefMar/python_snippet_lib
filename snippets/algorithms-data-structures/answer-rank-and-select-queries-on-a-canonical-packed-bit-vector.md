---
title: "Answer Rank and Select Queries on a Canonical Packed Bit Vector"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - maintain-a-bounded-integer-multiset-with-fenwick-rank-and-select.md
  - decide-bounded-non-negative-subset-sum-reachability-with-an-integer-bitset.md
  - ../configuration-serialization/pack-and-restore-a-bounded-strictly-increasing-integer-sequence-with-elias-fano.md
---

# Answer Rank and Select Queries on a Canonical Packed Bit Vector

## Idea and Problem

Index one immutable packed bit vector so rank queries count one bits before a position and select queries locate one bits by zero-based ordinal.

The payload stores meaningful bits most-significant first. Its byte length must
be exactly the ceiling of the declared bit length, and unused low bits in the
last byte must be zero. A cumulative one-count per byte then answers rank with
one prefix lookup and one masked byte. Select binary-searches those cumulative
counts and scans only the located byte.

## When to Use

Use this for a bounded, static bitmap that is already available as canonical
bytes and receives a batch of positional and occurrence queries. It fits
compact presence maps, immutable feature flags, and validation or decoding
steps that need to move between a one bit's position and ordinal.

Use a mutable Fenwick or segment tree when bits change between queries. Use a
plain integer and `bit_count` for very small or infrequently queried bitmaps.
Use a dedicated succinct-bit-vector implementation when auxiliary-space bounds
or stronger select guarantees matter more than a short standard-library-only
index.

## Implementation

```python
from bisect import bisect_left
from dataclasses import dataclass

_MAX_BIT_LENGTH = 1_048_576
_MAX_BIT_QUERY_COUNT = 65_536


@dataclass(frozen=True, slots=True)
class BitRankSelectAnswers:
    rank_counts: tuple[int, ...]
    select_positions: tuple[int | None, ...]


def _validated_bit_queries(
    queries: object,
    bit_length: int,
    field: str,
) -> tuple[int, ...]:
    if type(queries) is not tuple:
        raise TypeError(f"{field} must be an exact tuple")
    if len(queries) > _MAX_BIT_QUERY_COUNT:
        raise ValueError(f"{field} contains more than 65,536 items")
    for index, query in enumerate(queries):
        if type(query) is not int:
            raise TypeError(f"{field}[{index}] must be an exact integer")
        if not 0 <= query <= bit_length:
            raise ValueError(f"{field}[{index}] is outside 0..bit_length")
    return queries


def answer_bit_rank_select(
    payload: bytes,
    bit_length: int,
    rank_stops: tuple[int, ...],
    select_ordinals: tuple[int, ...],
) -> BitRankSelectAnswers:
    """Answer rank1 and select1 queries on one canonical packed bitmap."""
    if type(payload) is not bytes:
        raise TypeError("payload must be exact bytes")
    if type(bit_length) is not int:
        raise TypeError("bit_length must be an exact integer")
    if not 0 <= bit_length <= _MAX_BIT_LENGTH:
        raise ValueError("bit_length is outside 0..1,048,576")
    if len(payload) != (bit_length + 7) // 8:
        raise ValueError("payload has the wrong byte length")
    used_bits = bit_length % 8
    if used_bits and payload[-1] & ((1 << (8 - used_bits)) - 1):
        raise ValueError("payload has non-zero low padding bits")

    checked_rank_stops = _validated_bit_queries(rank_stops, bit_length, "rank_stops")
    checked_select_ordinals = _validated_bit_queries(
        select_ordinals,
        bit_length,
        "select_ordinals",
    )

    byte_prefix = [0]
    for value in payload:
        byte_prefix.append(byte_prefix[-1] + value.bit_count())

    rank_counts: list[int] = []
    for stop in checked_rank_stops:
        full_bytes, partial_bits = divmod(stop, 8)
        count = byte_prefix[full_bytes]
        if partial_bits:
            count += (payload[full_bytes] >> (8 - partial_bits)).bit_count()
        rank_counts.append(count)

    total_ones = byte_prefix[-1]
    select_positions: list[int | None] = []
    for ordinal in checked_select_ordinals:
        target_count = ordinal + 1
        if target_count > total_ones:
            select_positions.append(None)
            continue

        byte_stop = bisect_left(byte_prefix, target_count)
        byte_index = byte_stop - 1
        remaining = target_count - byte_prefix[byte_index]
        value = payload[byte_index]
        for bit_offset in range(8):
            if value & (1 << (7 - bit_offset)):
                remaining -= 1
                if remaining == 0:
                    select_positions.append(byte_index * 8 + bit_offset)
                    break
        else:  # The cumulative counts make this branch unreachable.
            raise AssertionError("byte prefix and payload disagree")

    return BitRankSelectAnswers(tuple(rank_counts), tuple(select_positions))
```

## Example

```python
from itertools import product


def pack_bool_oracle(bits: tuple[bool, ...]) -> bytes:
    packed = bytearray((len(bits) + 7) // 8)
    for position, bit in enumerate(bits):
        if bit:
            packed[position // 8] |= 1 << (7 - position % 8)
    return bytes(packed)


def bool_query_oracle(
    bits: tuple[bool, ...],
    rank_stops: tuple[int, ...],
    select_ordinals: tuple[int, ...],
) -> BitRankSelectAnswers:
    one_positions = tuple(position for position, bit in enumerate(bits) if bit)
    return BitRankSelectAnswers(
        tuple(sum(bits[:stop]) for stop in rank_stops),
        tuple(
            one_positions[ordinal] if ordinal < len(one_positions) else None
            for ordinal in select_ordinals
        ),
    )


exhaustive_checked = 0
for size in range(15):
    rank_stops = tuple(range(size + 1))
    select_ordinals = tuple(range(size + 1))
    for bits in product((False, True), repeat=size):
        actual = answer_bit_rank_select(
            pack_bool_oracle(bits),
            size,
            rank_stops,
            select_ordinals,
        )
        assert actual == bool_query_oracle(bits, rank_stops, select_ordinals)
        for ordinal, position in enumerate(actual.select_positions):
            if position is not None:
                assert actual.rank_counts[position] == ordinal
                assert actual.rank_counts[position + 1] == ordinal + 1
        exhaustive_checked += 1

known = answer_bit_rank_select(
    b"\xb2\x80",
    9,
    (0, 1, 4, 8, 9),
    (0, 1, 2, 3, 4, 5, 9),
)
empty = answer_bit_rank_select(b"", 0, (0,), (0,))
all_zero = answer_bit_rank_select(b"\x00\x00", 16, (0, 8, 16), (0, 16))
all_one = answer_bit_rank_select(b"\xff\xff", 16, (0, 1, 16), (0, 15, 16))

maximum_payload = b"\xff" * (_MAX_BIT_LENGTH // 8)
maximum_queries = answer_bit_rank_select(
    maximum_payload,
    _MAX_BIT_LENGTH,
    (_MAX_BIT_LENGTH,) * _MAX_BIT_QUERY_COUNT,
    (0,) * _MAX_BIT_QUERY_COUNT,
)
maximum_edges = answer_bit_rank_select(
    maximum_payload,
    _MAX_BIT_LENGTH,
    (),
    (_MAX_BIT_LENGTH - 1, _MAX_BIT_LENGTH),
)


def rejects(
    payload: object,
    bit_length: object,
    rank_stops: object,
    select_ordinals: object,
) -> bool:
    try:
        answer_bit_rank_select(  # type: ignore[arg-type]
            payload,
            bit_length,
            rank_stops,
            select_ordinals,
        )
    except (TypeError, ValueError):
        return True
    return False


invalid_calls = (
    (bytearray(), 0, (), ()),
    (b"", True, (), ()),
    (b"", -1, (), ()),
    (b"", _MAX_BIT_LENGTH + 1, (), ()),
    (b"", 1, (), ()),
    (b"\x00", 0, (), ()),
    (b"\x40", 1, (), ()),
    (b"\x80", 1, [], ()),
    (b"\x80", 1, (), []),
    (b"\x80", 1, (True,), ()),
    (b"\x80", 1, (), (True,)),
    (b"\x80", 1, (-1,), ()),
    (b"\x80", 1, (2,), ()),
    (b"\x80", 1, (), (-1,)),
    (b"\x80", 1, (), (2,)),
    (b"", 0, (0,) * (_MAX_BIT_QUERY_COUNT + 1), ()),
    (b"", 0, (), (0,) * (_MAX_BIT_QUERY_COUNT + 1)),
)
rejected = sum(rejects(*arguments) for arguments in invalid_calls)

assert (
    exhaustive_checked,
    known,
    empty,
    all_zero,
    all_one,
    len(maximum_queries.rank_counts),
    maximum_queries.rank_counts[0],
    len(maximum_queries.select_positions),
    maximum_queries.select_positions[0],
    maximum_edges,
    rejected,
) == (
    32_767,
    BitRankSelectAnswers((0, 1, 3, 4, 5), (0, 2, 3, 6, 8, None, None)),
    BitRankSelectAnswers((0,), (None,)),
    BitRankSelectAnswers((0, 0, 0), (None, None)),
    BitRankSelectAnswers((0, 1, 16), (0, 15, None)),
    _MAX_BIT_QUERY_COUNT,
    _MAX_BIT_LENGTH,
    _MAX_BIT_QUERY_COUNT,
    0,
    BitRankSelectAnswers((), (_MAX_BIT_LENGTH - 1, None)),
    len(invalid_calls),
)
```

## Trade-offs and Limitations

For `B` payload bytes, `R` rank queries, and `S` select queries, building the
prefix counts takes `O(B)` time. Rank takes `O(1)` per query. Select takes
`O(log(B + 1))` for binary search plus a scan of at most eight bits. Total time
is `O(B + R + S log(B + 1))`; auxiliary and returned state together use
`O(B + R + S)` integers or positions.

The byte-prefix table is deliberately simple, not succinct. It can use much
more space than a sampled or compressed rank/select index, and select is not
constant time. Repeated calls rebuild the table; retain a purpose-built index
when the same payload serves many batches.

The profile is MSB-first and rejects alternate byte lengths or non-zero low
padding. It is immutable and supports only one-bit rank and select. It does not
provide zero-bit queries, mutation, streaming, dynamic updates, bitwise
combination, or serialization from an unpacked Boolean tuple.

## Related Snippets

<!-- catalog:related:start -->
- [Maintain a Bounded Integer Multiset with Fenwick Rank and Select](maintain-a-bounded-integer-multiset-with-fenwick-rank-and-select.md)
- [Decide Bounded Non-Negative Subset-Sum Reachability with an Integer Bitset](decide-bounded-non-negative-subset-sum-reachability-with-an-integer-bitset.md)
- [Pack and Restore a Bounded Strictly Increasing Integer Sequence with Elias–Fano](../configuration-serialization/pack-and-restore-a-bounded-strictly-increasing-integer-sequence-with-elias-fano.md)
<!-- catalog:related:end -->
