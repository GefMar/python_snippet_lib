---
title: "Encode and Decode RFC 8949 Core Deterministic CBOR Under Structural Limits"
snippet_type: recipe
use_cases:
  - interoperability
  - parsing
  - serialization
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - encode-and-decode-canonical-bencode-under-structural-limits.md
  - parse-bounded-utf-8-json-without-duplicate-object-names-or-non-finite-numbers.md
  - derive-a-versioned-cache-key-from-deterministically-encoded-bounded-json.md
---

# Encode and Decode RFC 8949 Core Deterministic CBOR Under Structural Limits

## Idea and Problem

Round-trip a closed immutable CBOR model while accepting only one bounded core-deterministic encoding.

[RFC 8949 Section 4.2.1](https://www.rfc-editor.org/rfc/rfc8949.html#section-4.2.1)
requires preferred shortest arguments, forbids indefinite-length items, and
orders map keys by the bytewise lexicographic order of their deterministic
encodings. A decoder that accepts alternate spellings or loses duplicate map
keys before checking them cannot establish that contract.

This profile contains null, booleans, basic CBOR integers, byte strings, UTF-8
text, arrays, and maps with unique text keys. Explicit array and map classes
keep the returned tree immutable and avoid guessing whether a tuple represents
an array or a sequence of pairs.

## When to Use

Use this codec when a small interoperability value, signed input, fixture, or
cache-key component must have one inspectable CBOR representation. Both sides
must agree on this exact text-keyed profile and its resource limits.

Use a maintained CBOR implementation for streams, CBOR sequences, tags,
bignums, floating-point values, arbitrary map-key types, schema integration,
or application protocols with a broader data model. Deterministic encoding
does not authenticate bytes or make decoded content safe to act upon.

## Implementation

```python
from dataclasses import dataclass

_MIN_CBOR_INTEGER = -(1 << 64)
_MAX_CBOR_INTEGER = (1 << 64) - 1
_MAX_ENCODED_BYTES = 1_048_576
_MAX_STRING_BYTES = 262_144
_MAX_CUMULATIVE_STRING_BYTES = 524_288
_MAX_NODES = 10_000
_MAX_CONTAINER_ITEMS = 4_096
_MAX_DEPTH = 32


class DeterministicCborError(ValueError):
    """Raised when a value is outside this deterministic CBOR profile."""


@dataclass(frozen=True, slots=True)
class CborArray:
    values: tuple["CborValue", ...]


@dataclass(frozen=True, slots=True)
class CborMap:
    items: tuple[tuple[str, "CborValue"], ...]


type CborValue = None | bool | int | bytes | str | CborArray | CborMap


@dataclass(slots=True)
class _Budget:
    nodes: int = 0
    string_bytes: int = 0

    def claim_node(self, depth: int) -> None:
        if depth > _MAX_DEPTH:
            raise DeterministicCborError("CBOR nesting exceeds the supported depth")
        self.nodes += 1
        if self.nodes > _MAX_NODES:
            raise DeterministicCborError("CBOR node count exceeds the supported limit")

    def claim_string(self, length: int) -> None:
        if length > _MAX_STRING_BYTES:
            raise DeterministicCborError("CBOR string exceeds the supported size")
        self.string_bytes += length
        if self.string_bytes > _MAX_CUMULATIVE_STRING_BYTES:
            raise DeterministicCborError("cumulative CBOR string bytes exceed the supported limit")


class _CborParser:
    def __init__(self, encoded: bytes) -> None:
        self.encoded = encoded
        self.position = 0
        self.budget = _Budget()

    def _read_argument(self, additional: int) -> int:
        if additional <= 23:
            return additional
        if additional == 24:
            width, minimum = 1, 24
        elif additional == 25:
            width, minimum = 2, 1 << 8
        elif additional == 26:
            width, minimum = 4, 1 << 16
        elif additional == 27:
            width, minimum = 8, 1 << 32
        elif additional == 31:
            raise DeterministicCborError("indefinite-length items are not accepted")
        else:
            raise DeterministicCborError("reserved CBOR additional information")

        end = self.position + width
        if end > len(self.encoded):
            raise DeterministicCborError("CBOR argument is truncated")
        argument = int.from_bytes(self.encoded[self.position : end], "big")
        self.position = end
        if argument < minimum:
            raise DeterministicCborError("CBOR argument is not in its shortest form")
        return argument

    def _read_string(self, additional: int, *, text: bool) -> bytes | str:
        length = self._read_argument(additional)
        self.budget.claim_string(length)
        end = self.position + length
        if end > len(self.encoded):
            raise DeterministicCborError("CBOR string payload is truncated")
        payload = self.encoded[self.position : end]
        self.position = end
        if not text:
            return payload
        try:
            return payload.decode("utf-8")
        except UnicodeDecodeError as error:
            raise DeterministicCborError("CBOR text is not valid UTF-8") from error

    def read_value(self, depth: int) -> CborValue:
        if self.position >= len(self.encoded):
            raise DeterministicCborError("CBOR value is truncated")
        self.budget.claim_node(depth)

        initial = self.encoded[self.position]
        self.position += 1
        major = initial >> 5
        additional = initial & 0x1F

        if major == 0:
            return self._read_argument(additional)
        if major == 1:
            return -1 - self._read_argument(additional)
        if major == 2:
            return self._read_string(additional, text=False)
        if major == 3:
            return self._read_string(additional, text=True)
        if major == 4:
            count = self._read_argument(additional)
            if count > _MAX_CONTAINER_ITEMS:
                raise DeterministicCborError("CBOR array exceeds the item limit")
            if count > _MAX_NODES - self.budget.nodes:
                raise DeterministicCborError("CBOR node count exceeds the supported limit")
            return CborArray(tuple(self.read_value(depth + 1) for _ in range(count)))
        if major == 5:
            pair_count = self._read_argument(additional)
            if pair_count > _MAX_CONTAINER_ITEMS:
                raise DeterministicCborError("CBOR map exceeds the pair limit")
            if pair_count * 2 > _MAX_NODES - self.budget.nodes:
                raise DeterministicCborError("CBOR node count exceeds the supported limit")

            items: list[tuple[str, CborValue]] = []
            seen: set[str] = set()
            previous_key_encoding: bytes | None = None
            for _ in range(pair_count):
                if self.position >= len(self.encoded):
                    raise DeterministicCborError("CBOR map key is truncated")
                if self.encoded[self.position] >> 5 != 3:
                    raise DeterministicCborError("CBOR map keys must be text strings")
                key_start = self.position
                key = self.read_value(depth + 1)
                if type(key) is not str:
                    raise AssertionError("the text-key precheck must produce text")
                key_encoding = self.encoded[key_start : self.position]
                if key in seen:
                    raise DeterministicCborError("CBOR map keys must be unique")
                if previous_key_encoding is not None and key_encoding <= previous_key_encoding:
                    raise DeterministicCborError(
                        "CBOR map keys are not in core deterministic order"
                    )
                seen.add(key)
                previous_key_encoding = key_encoding
                items.append((key, self.read_value(depth + 1)))
            return CborMap(tuple(items))
        if major == 6:
            raise DeterministicCborError("CBOR tags are outside this profile")

        if additional == 20:
            return False
        if additional == 21:
            return True
        if additional == 22:
            return None
        if 28 <= additional <= 30:
            raise DeterministicCborError("reserved CBOR simple-value encoding")
        if additional == 31:
            raise DeterministicCborError("a CBOR break code is not a value")
        raise DeterministicCborError("CBOR floats and other simple values are outside this profile")


def decode_deterministic_cbor(encoded: bytes) -> CborValue:
    """Decode exactly one bounded RFC 8949 core-deterministic value."""
    if type(encoded) is not bytes:
        raise TypeError("encoded must be exact bytes")
    if not 1 <= len(encoded) <= _MAX_ENCODED_BYTES:
        raise DeterministicCborError("encoded size is outside the supported range")

    parser = _CborParser(encoded)
    value = parser.read_value(depth=1)
    if parser.position != len(encoded):
        raise DeterministicCborError("trailing bytes follow the CBOR value")
    return value


class _CborEncoder:
    def __init__(self) -> None:
        self.output = bytearray()
        self.budget = _Budget()
        self.active_containers: set[int] = set()

    @staticmethod
    def _head(major: int, argument: int) -> bytes:
        prefix = major << 5
        if argument <= 23:
            return bytes((prefix | argument,))
        if argument <= 0xFF:
            return bytes((prefix | 24, argument))
        if argument <= 0xFFFF:
            return bytes((prefix | 25,)) + argument.to_bytes(2, "big")
        if argument <= 0xFFFF_FFFF:
            return bytes((prefix | 26,)) + argument.to_bytes(4, "big")
        return bytes((prefix | 27,)) + argument.to_bytes(8, "big")

    def _append(self, part: bytes) -> None:
        if len(self.output) + len(part) > _MAX_ENCODED_BYTES:
            raise DeterministicCborError("encoded CBOR exceeds the supported size")
        self.output.extend(part)

    @staticmethod
    def _utf8(text: str, *, role: str) -> bytes:
        if len(text) > _MAX_STRING_BYTES:
            raise DeterministicCborError(f"{role} exceeds the supported size")
        try:
            return text.encode("utf-8")
        except UnicodeEncodeError as error:
            raise DeterministicCborError(f"{role} is not valid Unicode scalar text") from error

    def write_value(self, value: object, depth: int) -> None:
        self.budget.claim_node(depth)

        if value is None:
            self._append(b"\xf6")
            return
        if type(value) is bool:
            self._append(b"\xf5" if value else b"\xf4")
            return
        if type(value) is int:
            if not _MIN_CBOR_INTEGER <= value <= _MAX_CBOR_INTEGER:
                raise DeterministicCborError("integer is outside the basic CBOR range")
            if value >= 0:
                self._append(self._head(0, value))
            else:
                self._append(self._head(1, -1 - value))
            return
        if type(value) is bytes:
            self.budget.claim_string(len(value))
            self._append(self._head(2, len(value)) + value)
            return
        if type(value) is str:
            payload = self._utf8(value, role="CBOR text")
            self.budget.claim_string(len(payload))
            self._append(self._head(3, len(payload)) + payload)
            return
        if type(value) is CborArray:
            if type(value.values) is not tuple:
                raise TypeError("CborArray.values must be an exact tuple")
            if len(value.values) > _MAX_CONTAINER_ITEMS:
                raise DeterministicCborError("CBOR array exceeds the item limit")
            if len(value.values) > _MAX_NODES - self.budget.nodes:
                raise DeterministicCborError("CBOR node count exceeds the supported limit")

            container_id = id(value)
            if container_id in self.active_containers:
                raise DeterministicCborError("active CBOR container cycle detected")
            self.active_containers.add(container_id)
            try:
                self._append(self._head(4, len(value.values)))
                for child in value.values:
                    self.write_value(child, depth + 1)
            finally:
                self.active_containers.remove(container_id)
            return
        if type(value) is CborMap:
            if type(value.items) is not tuple:
                raise TypeError("CborMap.items must be an exact tuple")
            if len(value.items) > _MAX_CONTAINER_ITEMS:
                raise DeterministicCborError("CBOR map exceeds the pair limit")
            if len(value.items) * 2 > _MAX_NODES - self.budget.nodes:
                raise DeterministicCborError("CBOR node count exceeds the supported limit")

            prepared: list[tuple[bytes, int, object]] = []
            seen: set[str] = set()
            prepared_key_bytes = 0
            for item in value.items:
                if type(item) is not tuple or len(item) != 2:
                    raise TypeError("map items must be exact key-value tuples")
                key, child = item
                if type(key) is not str:
                    raise TypeError("CBOR map keys must be exact strings")
                if key in seen:
                    raise DeterministicCborError("CBOR map keys must be unique")
                seen.add(key)
                payload = self._utf8(key, role="CBOR map key")
                if len(payload) > _MAX_STRING_BYTES:
                    raise DeterministicCborError("CBOR map key exceeds the supported size")
                prepared_key_bytes += len(payload)
                if self.budget.string_bytes + prepared_key_bytes > _MAX_CUMULATIVE_STRING_BYTES:
                    raise DeterministicCborError(
                        "cumulative CBOR string bytes exceed the supported limit"
                    )
                prepared.append((self._head(3, len(payload)) + payload, len(payload), child))
            prepared.sort(key=lambda item: item[0])

            container_id = id(value)
            if container_id in self.active_containers:
                raise DeterministicCborError("active CBOR container cycle detected")
            self.active_containers.add(container_id)
            try:
                self._append(self._head(5, len(prepared)))
                for encoded_key, payload_length, child in prepared:
                    self.budget.claim_node(depth + 1)
                    self.budget.claim_string(payload_length)
                    self._append(encoded_key)
                    self.write_value(child, depth + 1)
            finally:
                self.active_containers.remove(container_id)
            return

        raise TypeError("value is outside the closed CBOR model")


def encode_deterministic_cbor(value: CborValue) -> bytes:
    """Encode one value using RFC 8949 core deterministic rules."""
    encoder = _CborEncoder()
    encoder.write_value(value, depth=1)
    return bytes(encoder.output)
```

## Example

```python
official_vectors: tuple[tuple[CborValue, str], ...] = (
    (0, "00"),
    (23, "17"),
    (24, "1818"),
    (-24, "37"),
    (-25, "3818"),
    (b"", "40"),
    ("", "60"),
    (False, "f4"),
    (True, "f5"),
    (None, "f6"),
    (CborArray((1, 2, 3)), "83010203"),
    (
        CborMap((("a", 1), ("b", CborArray((2, 3))))),
        "a26161016162820203",
    ),
)

for value, expected_hex in official_vectors:
    encoded = encode_deterministic_cbor(value)
    assert encoded == bytes.fromhex(expected_hex)
    assert decode_deterministic_cbor(encoded) == value

assert encode_deterministic_cbor(_MAX_CBOR_INTEGER).hex() == "1bffffffffffffffff"
assert encode_deterministic_cbor(_MIN_CBOR_INTEGER).hex() == "3bffffffffffffffff"
assert encode_deterministic_cbor("x" * 23)[:1] == b"\x77"
assert encode_deterministic_cbor("x" * 24)[:2] == b"\x78\x18"

unordered = CborMap((("aa", 1), ("z", 0)))
canonical = CborMap((("z", 0), ("aa", 1)))
assert encode_deterministic_cbor(unordered).hex() == "a2617a0062616101"
assert decode_deterministic_cbor(encode_deterministic_cbor(unordered)) == canonical


def decoding_rejects(encoded: bytes) -> bool:
    try:
        decode_deterministic_cbor(encoded)
    except DeterministicCborError:
        return True
    return False


def encoding_rejects(value: object) -> bool:
    try:
        encode_deterministic_cbor(value)  # type: ignore[arg-type]
    except (DeterministicCborError, TypeError):
        return True
    return False


assert all(
    decoding_rejects(bytes.fromhex(encoded_hex))
    for encoded_hex in (
        "1817",  # overlong integer argument
        "9f01ff",  # indefinite array
        "61ff",  # invalid UTF-8
        "a262616101617a00",  # out-of-order encoded keys
        "a2616101616102",  # duplicate key
        "a10100",  # non-text map key
        "f93e00",  # floating-point value
        "c000",  # tag
        "0000",  # trailing value
    )
)

maximum_depth: CborValue = 0
for _ in range(_MAX_DEPTH - 1):
    maximum_depth = CborArray((maximum_depth,))
assert decode_deterministic_cbor(encode_deterministic_cbor(maximum_depth)) == maximum_depth
assert encoding_rejects(CborArray((maximum_depth,)))
assert encoding_rejects(b"x" * (_MAX_STRING_BYTES + 1))
assert encoding_rejects(CborArray((None,) * (_MAX_CONTAINER_ITEMS + 1)))

too_many_nodes = b"\x99\x07\xd0" + b"\x84\xf6\xf6\xf6\xf6" * 2_000
assert decoding_rejects(too_many_nodes)

cyclic = CborArray(())
object.__setattr__(cyclic, "values", (cyclic,))
assert encoding_rejects(cyclic)
```

## Trade-offs and Limitations

Decoding is linear in the encoded byte count. Encoding is linear except for
sorting each map's `k` encoded keys: that takes `O(k log k)` comparisons and up
to `O(B log k)` compared-byte work for `B` total encoded key bytes. The
complete input, returned tree, and encoded output remain in memory. The root is
depth one; nodes include map keys, and the cumulative string budget includes
byte strings and UTF-8 bytes for text values and map keys.

This is the RFC 8949 core deterministic ordering, not the alternate
length-first compatibility profile. The decoder rejects otherwise valid CBOR
outside the closed model, non-preferred encodings, indefinite items, duplicate
or misordered keys, concatenated values, and every configured resource excess.
The encoder accepts maps in any item order, but decoded `CborMap.items` are in
canonical wire order.

No Unicode normalization, tag interpretation, float normalization, schema
validation, checksum, signature, encryption, streaming, or incremental output
is provided. A protocol must define those concerns separately.

## Related Snippets

<!-- catalog:related:start -->
- [Encode and Decode Canonical Bencode Under Structural Limits](encode-and-decode-canonical-bencode-under-structural-limits.md)
- [Parse Bounded UTF-8 JSON Without Duplicate Object Names or Non-Finite Numbers](parse-bounded-utf-8-json-without-duplicate-object-names-or-non-finite-numbers.md)
- [Derive a Versioned Cache Key from Deterministically Encoded Bounded JSON](derive-a-versioned-cache-key-from-deterministically-encoded-bounded-json.md)
<!-- catalog:related:end -->
