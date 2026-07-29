---
title: "Encode and Decode Canonical Bencode Under Structural Limits"
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
  - parse-bounded-utf-8-json-without-duplicate-object-names-or-non-finite-numbers.md
  - encode-a-bounded-signed-integer-in-its-shortest-big-endian-twos-complement-byte-string.md
  - ../networking-protocols/read-and-write-size-capped-varint-frames.md
---

# Encode and Decode Canonical Bencode Under Structural Limits

## Idea and Problem

Round-trip a small immutable bencode model while rejecting ambiguous spellings and structurally excessive inputs.

[Bencode](https://www.bittorrent.org/beps/bep_0003.html) represents byte
strings with decimal lengths, integers between `i` and `e`, lists between `l`
and `e`, and dictionaries between `d` and `e`. Canonical dictionary keys are
raw bytes in strictly increasing lexicographic order.

The decoder accepts only canonical tokens and complete input. The encoder sorts
dictionary items, applies the same byte, node, and depth limits, and rejects
duplicate keys or active container cycles.

## When to Use

Use this codec for a closed interoperability boundary that needs exact bytes,
signed integers, immutable lists, and byte-key dictionaries. The explicit
model avoids silently guessing whether application bytes contain text.

Use a maintained protocol library when streaming, large values, extension
types, schema validation, or protocol-specific metadata is required. Use JSON,
CBOR, MessagePack, or another format when bencode interoperability is not the
actual requirement.

## Implementation

```python
from dataclasses import dataclass

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_ENCODED_BYTES = 1_048_576
_MAX_BYTE_STRING_BYTES = 65_536
_MAX_NODES = 10_000
_MAX_DEPTH = 32


@dataclass(frozen=True, slots=True)
class BencodeList:
    values: tuple[BencodeValue, ...]


@dataclass(frozen=True, slots=True)
class BencodeDictionary:
    items: tuple[tuple[bytes, BencodeValue], ...]


type BencodeValue = int | bytes | BencodeList | BencodeDictionary


class _BencodeParser:
    def __init__(self, encoded: bytes) -> None:
        self.encoded = encoded
        self.position = 0
        self.node_count = 0

    def _claim_node(self, depth: int) -> None:
        if depth > _MAX_DEPTH:
            raise ValueError("bencode nesting exceeds the supported depth")
        self.node_count += 1
        if self.node_count > _MAX_NODES:
            raise ValueError("bencode node count exceeds the supported limit")

    def _read_byte_string(self, depth: int) -> bytes:
        self._claim_node(depth)
        colon = self.encoded.find(b":", self.position)
        if colon < 0:
            raise ValueError("byte string length has no colon")
        length_token = self.encoded[self.position : colon]
        if not length_token:
            raise ValueError("byte string length is empty")
        if length_token == b"0":
            length = 0
        else:
            if not 49 <= length_token[0] <= 57 or any(
                byte < 48 or byte > 57 for byte in length_token[1:]
            ):
                raise ValueError("byte string length is not canonical")
            if len(length_token) > 5:
                raise ValueError("byte string exceeds the supported size")
            length = int(length_token)
        if length > _MAX_BYTE_STRING_BYTES:
            raise ValueError("byte string exceeds the supported size")

        value_start = colon + 1
        value_end = value_start + length
        if value_end > len(self.encoded):
            raise ValueError("byte string payload is truncated")
        self.position = value_end
        return self.encoded[value_start:value_end]

    def _read_integer(self, depth: int) -> int:
        self._claim_node(depth)
        end = self.encoded.find(b"e", self.position + 1)
        if end < 0:
            raise ValueError("integer has no terminator")
        payload = self.encoded[self.position + 1 : end]

        if payload == b"0":
            value = 0
        else:
            digits = payload[1:] if payload.startswith(b"-") else payload
            if (
                not digits
                or not 49 <= digits[0] <= 57
                or any(byte < 48 or byte > 57 for byte in digits[1:])
            ):
                raise ValueError("integer payload is not canonical")
            if len(digits) > 19:
                raise ValueError("integer is outside the signed 64-bit range")
            value = int(payload)
        if not _MIN_INT64 <= value <= _MAX_INT64:
            raise ValueError("integer is outside the signed 64-bit range")
        self.position = end + 1
        return value

    def read_value(self, depth: int) -> BencodeValue:
        if self.position >= len(self.encoded):
            raise ValueError("bencode value is truncated")
        marker = self.encoded[self.position]

        if 48 <= marker <= 57:
            return self._read_byte_string(depth)
        if marker == ord("i"):
            return self._read_integer(depth)
        if marker == ord("l"):
            self._claim_node(depth)
            self.position += 1
            values: list[BencodeValue] = []
            while True:
                if self.position >= len(self.encoded):
                    raise ValueError("list has no terminator")
                if self.encoded[self.position] == ord("e"):
                    self.position += 1
                    return BencodeList(tuple(values))
                values.append(self.read_value(depth + 1))
        if marker == ord("d"):
            self._claim_node(depth)
            self.position += 1
            items: list[tuple[bytes, BencodeValue]] = []
            previous_key: bytes | None = None
            while True:
                if self.position >= len(self.encoded):
                    raise ValueError("dictionary has no terminator")
                if self.encoded[self.position] == ord("e"):
                    self.position += 1
                    return BencodeDictionary(tuple(items))
                if not 48 <= self.encoded[self.position] <= 57:
                    raise ValueError("dictionary key must be a byte string")
                key = self._read_byte_string(depth + 1)
                if previous_key is not None and key <= previous_key:
                    raise ValueError("dictionary keys must be strictly increasing")
                value = self.read_value(depth + 1)
                items.append((key, value))
                previous_key = key

        raise ValueError("unknown bencode marker")


def decode_canonical_bencode(encoded: bytes) -> BencodeValue:
    """Decode exactly one canonical, structurally bounded value."""
    if type(encoded) is not bytes:
        raise TypeError("encoded must be exact bytes")
    if not 1 <= len(encoded) <= _MAX_ENCODED_BYTES:
        raise ValueError("encoded size is outside the supported range")

    parser = _BencodeParser(encoded)
    value = parser.read_value(depth=1)
    if parser.position != len(encoded):
        raise ValueError("trailing bytes follow the bencode value")
    return value


class _BencodeEncoder:
    def __init__(self) -> None:
        self.output = bytearray()
        self.node_count = 0
        self.active_containers: set[int] = set()

    def _claim_node(self, depth: int) -> None:
        if depth > _MAX_DEPTH:
            raise ValueError("bencode nesting exceeds the supported depth")
        self.node_count += 1
        if self.node_count > _MAX_NODES:
            raise ValueError("bencode node count exceeds the supported limit")

    def _append(self, encoded_part: bytes) -> None:
        if len(self.output) + len(encoded_part) > _MAX_ENCODED_BYTES:
            raise ValueError("encoded bencode exceeds the supported size")
        self.output.extend(encoded_part)

    def _write_byte_string(self, value: bytes, depth: int) -> None:
        self._claim_node(depth)
        if type(value) is not bytes:
            raise TypeError("byte strings and dictionary keys must be exact bytes")
        if len(value) > _MAX_BYTE_STRING_BYTES:
            raise ValueError("byte string exceeds the supported size")
        self._append(str(len(value)).encode("ascii") + b":" + value)

    def write_value(self, value: object, depth: int) -> None:
        if type(value) is int:
            self._claim_node(depth)
            if not _MIN_INT64 <= value <= _MAX_INT64:
                raise ValueError("integer is outside the signed 64-bit range")
            self._append(b"i" + str(value).encode("ascii") + b"e")
            return
        if type(value) is bytes:
            self._write_byte_string(value, depth)
            return
        if type(value) is BencodeList:
            self._claim_node(depth)
            if type(value.values) is not tuple:
                raise TypeError("BencodeList.values must be an exact tuple")
            container_id = id(value)
            if container_id in self.active_containers:
                raise ValueError("active bencode container cycle detected")
            self.active_containers.add(container_id)
            try:
                self._append(b"l")
                for item in value.values:
                    self.write_value(item, depth + 1)
                self._append(b"e")
            finally:
                self.active_containers.remove(container_id)
            return
        if type(value) is BencodeDictionary:
            self._claim_node(depth)
            if type(value.items) is not tuple:
                raise TypeError("BencodeDictionary.items must be an exact tuple")
            remaining_nodes = _MAX_NODES - self.node_count
            if len(value.items) > remaining_nodes // 2:
                raise ValueError("bencode node count exceeds the supported limit")
            container_id = id(value)
            if container_id in self.active_containers:
                raise ValueError("active bencode container cycle detected")
            self.active_containers.add(container_id)
            try:
                prepared: list[tuple[bytes, object]] = []
                minimum_additional_bytes = 2
                for item in value.items:
                    if type(item) is not tuple or len(item) != 2:
                        raise TypeError("dictionary items must be exact key-value tuples")
                    key, child = item
                    if type(key) is not bytes:
                        raise TypeError("dictionary keys must be exact bytes")
                    if len(key) > _MAX_BYTE_STRING_BYTES:
                        raise ValueError("dictionary key exceeds the supported size")
                    minimum_additional_bytes += len(str(len(key))) + 3 + len(key)
                    if len(self.output) + minimum_additional_bytes > _MAX_ENCODED_BYTES:
                        raise ValueError("encoded bencode exceeds the supported size")
                    prepared.append((key, child))
                prepared.sort(key=lambda item: item[0])
                if any(
                    prepared[index - 1][0] == prepared[index][0]
                    for index in range(1, len(prepared))
                ):
                    raise ValueError("dictionary keys must be unique")

                self._append(b"d")
                for key, child in prepared:
                    self._write_byte_string(key, depth + 1)
                    self.write_value(child, depth + 1)
                self._append(b"e")
            finally:
                self.active_containers.remove(container_id)
            return

        raise TypeError("value is outside the closed bencode model")


def encode_canonical_bencode(value: BencodeValue) -> bytes:
    """Encode one value canonically under the decoder's structural limits."""
    encoder = _BencodeEncoder()
    encoder.write_value(value, depth=1)
    return bytes(encoder.output)
```

## Example

```python
document = BencodeDictionary(
    (
        (b"version", 1),
        (b"items", BencodeList((b"alpha", -7))),
    )
)
expected = b"d5:itemsl5:alphai-7ee7:versioni1ee"

encoded = encode_canonical_bencode(document)
decoded = decode_canonical_bencode(encoded)

assert encoded == expected
assert decoded == BencodeDictionary(
    (
        (b"items", BencodeList((b"alpha", -7))),
        (b"version", 1),
    )
)
assert encode_canonical_bencode(decoded) == expected

try:
    decode_canonical_bencode(b"i-0e")
except ValueError:
    negative_zero_rejected = True
else:
    negative_zero_rejected = False

assert negative_zero_rejected
```

## Trade-offs and Limitations

Decoding is `O(B)` for `B` input bytes. Encoding is conservatively
`O(B log N)` in the worst case because sorting variable-length dictionary keys
may compare common byte prefixes repeatedly across `N` structural nodes. Both
directions retain `O(B + N)` result and traversal state.

The root is depth one. Every scalar or container occurrence and every
dictionary key token consumes one node; a shared acyclic subtree therefore
counts again at each occurrence. Cycle detection follows only the active
container recursion stack, so ordinary sharing is allowed.

This closed model intentionally narrows bencode integers to signed 64-bit and
all byte strings to 65,536 bytes. It provides no Unicode conversion, floats,
mutable list/dictionary output, streaming, recovery from malformed input,
arbitrary recursion, extension types, or interpretation of higher-level
protocol metadata.

## Related Snippets

<!-- catalog:related:start -->
- [Parse Bounded UTF-8 JSON Without Duplicate Object Names or Non-Finite Numbers](parse-bounded-utf-8-json-without-duplicate-object-names-or-non-finite-numbers.md)
- [Encode a Bounded Signed Integer in Its Shortest Big-Endian Two's-Complement Byte String](encode-a-bounded-signed-integer-in-its-shortest-big-endian-twos-complement-byte-string.md)
- [Read and Write Size-Capped Varint Frames](../networking-protocols/read-and-write-size-capped-varint-frames.md)
<!-- catalog:related:end -->
