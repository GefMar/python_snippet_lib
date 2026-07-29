---
title: "Recover a Verified Prefix from a Bounded CRC-Framed Byte Log"
snippet_type: recipe
use_cases:
  - parsing
  - persistence
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../configuration-serialization/append-a-fixed-width-crc-check-to-a-human-readable-identifier.md
  - ../networking-protocols/read-and-write-size-capped-varint-frames.md
  - validate-a-bounded-stage-verify-pointer-switch-log.md
---

# Recover a Verified Prefix from a Bounded CRC-Framed Byte Log

## Idea and Problem

Recover every complete, checksum-verified record before a torn final frame in a bounded immutable byte log.

Each private frame starts with the exact magic and version `b"LOG1"`, followed
by a four-byte payload length, its four-byte ones-complement, the payload, and a
four-byte CRC-32. The redundant length rejects a damaged full header before its
declared body size is trusted. A partial header is accepted as a torn tail only
while every available byte can still begin a valid bounded header.

## When to Use

Use this format for a small append-only byte log whose producer and recovery
reader share this exact private convention. Recovery is appropriate for an
immutable snapshot known to consist of a valid frame prefix followed by, at
most, one interrupted append.

Treat the returned tail as entirely unverified: it begins at the first
incomplete frame, not merely at the first missing byte. If corruption can occur
in the middle, records need authentication, or a reader must search for later
frames after damage, use a format and recovery protocol designed for those
requirements.

## Implementation

```python
import zlib
from dataclasses import dataclass

_MAGIC = b"LOG1"
_UINT32_MASK = 0xFFFF_FFFF
_LENGTH_BYTES = 4
_HEADER_BYTES = len(_MAGIC) + 2 * _LENGTH_BYTES
_CHECKSUM_BYTES = 4
_FRAME_OVERHEAD = _HEADER_BYTES + _CHECKSUM_BYTES
_MAX_PAYLOAD_BYTES = 65_536
_MAX_LOG_BYTES = 4 * 1024 * 1024
_MAX_FRAMES = 4_096


class CrcLogError(ValueError):
    pass


class CrcLogCorruptionError(CrcLogError):
    def __init__(self, frame_offset: int, reason: str) -> None:
        self.frame_offset = frame_offset
        self.reason = reason
        super().__init__(f"corrupt frame at byte offset {frame_offset}: {reason}")


class CrcLogFrameLimitError(CrcLogError):
    def __init__(self, frame_offset: int) -> None:
        self.frame_offset = frame_offset
        super().__init__(
            f"unparsed bytes remain after {_MAX_FRAMES} frames at byte offset {frame_offset}"
        )


@dataclass(frozen=True, slots=True)
class RecoveredCrcLog:
    payloads: tuple[bytes, ...]
    verified_prefix_length: int
    discarded_tail: bytes


def _uint32be(value: int) -> bytes:
    return value.to_bytes(_LENGTH_BYTES, "big")


def _frame_header(payload_length: int) -> bytes:
    return _MAGIC + _uint32be(payload_length) + _uint32be(payload_length ^ _UINT32_MASK)


def _encode_frame(payload: bytes) -> bytes:
    header = _frame_header(len(payload))
    checksum = zlib.crc32(header + payload) & _UINT32_MASK
    return header + payload + _uint32be(checksum)


def encode_crc_log(payloads: tuple[bytes, ...]) -> bytes:
    """Encode a bounded tuple using the exact private frame convention."""
    if type(payloads) is not tuple:
        raise TypeError("payloads must be an exact tuple")
    if len(payloads) > _MAX_FRAMES:
        raise ValueError("payload count exceeds the supported frame limit")

    encoded_size = 0
    for index, payload in enumerate(payloads):
        if type(payload) is not bytes:
            raise TypeError(f"payloads[{index}] must be exact bytes")
        if len(payload) > _MAX_PAYLOAD_BYTES:
            raise ValueError(f"payloads[{index}] exceeds the supported byte limit")
        encoded_size += _FRAME_OVERHEAD + len(payload)
        if encoded_size > _MAX_LOG_BYTES:
            raise ValueError("encoded log would exceed the supported byte limit")

    return b"".join(_encode_frame(payload) for payload in payloads)


def _partial_header_is_possible(partial: bytes) -> bool:
    magic_bytes = min(len(partial), len(_MAGIC))
    if partial[:magic_bytes] != _MAGIC[:magic_bytes]:
        return False
    if len(partial) <= len(_MAGIC):
        return True

    length_stop = min(len(partial), len(_MAGIC) + _LENGTH_BYTES)
    length_prefix = partial[len(_MAGIC) : length_stop]
    if len(length_prefix) < _LENGTH_BYTES:
        smallest_length = int.from_bytes(
            length_prefix + b"\x00" * (_LENGTH_BYTES - len(length_prefix)),
            "big",
        )
        return smallest_length <= _MAX_PAYLOAD_BYTES

    payload_length = int.from_bytes(length_prefix, "big")
    if payload_length > _MAX_PAYLOAD_BYTES:
        return False
    complement_prefix = partial[len(_MAGIC) + _LENGTH_BYTES :]
    expected_complement = _uint32be(payload_length ^ _UINT32_MASK)
    return complement_prefix == expected_complement[: len(complement_prefix)]


def _validated_payload_length(header: bytes, frame_offset: int) -> int:
    if header[: len(_MAGIC)] != _MAGIC:
        raise CrcLogCorruptionError(frame_offset, "invalid magic or version")

    length_start = len(_MAGIC)
    length_stop = length_start + _LENGTH_BYTES
    payload_length = int.from_bytes(header[length_start:length_stop], "big")
    complemented_length = int.from_bytes(header[length_stop:_HEADER_BYTES], "big")
    if complemented_length != payload_length ^ _UINT32_MASK:
        raise CrcLogCorruptionError(frame_offset, "invalid complemented length")
    if payload_length > _MAX_PAYLOAD_BYTES:
        raise CrcLogCorruptionError(frame_offset, "payload length exceeds the frame limit")
    return payload_length


def recover_crc_log(log_bytes: bytes) -> RecoveredCrcLog:
    """Return complete verified payloads and any first unverified tail."""
    if type(log_bytes) is not bytes:
        raise TypeError("log_bytes must be exact bytes")
    if len(log_bytes) > _MAX_LOG_BYTES:
        raise ValueError("log_bytes exceeds the supported byte limit")

    payloads: list[bytes] = []
    frame_offset = 0
    while frame_offset < len(log_bytes):
        if len(payloads) == _MAX_FRAMES:
            raise CrcLogFrameLimitError(frame_offset)

        remaining = len(log_bytes) - frame_offset
        if remaining < _HEADER_BYTES:
            tail = log_bytes[frame_offset:]
            if not _partial_header_is_possible(tail):
                raise CrcLogCorruptionError(frame_offset, "invalid partial header")
            return RecoveredCrcLog(tuple(payloads), frame_offset, tail)

        header = log_bytes[frame_offset : frame_offset + _HEADER_BYTES]
        payload_length = _validated_payload_length(header, frame_offset)
        payload_start = frame_offset + _HEADER_BYTES
        payload_stop = payload_start + payload_length
        frame_stop = payload_stop + _CHECKSUM_BYTES
        if frame_stop > len(log_bytes):
            return RecoveredCrcLog(
                tuple(payloads),
                frame_offset,
                log_bytes[frame_offset:],
            )

        payload = log_bytes[payload_start:payload_stop]
        expected_checksum = int.from_bytes(log_bytes[payload_stop:frame_stop], "big")
        actual_checksum = zlib.crc32(header + payload) & _UINT32_MASK
        if expected_checksum != actual_checksum:
            raise CrcLogCorruptionError(frame_offset, "CRC-32 mismatch")

        payloads.append(payload)
        frame_offset = frame_stop

    return RecoveredCrcLog(tuple(payloads), frame_offset, b"")
```

## Example

```python
payloads = (b"start", b"", b"finish")
encoded = encode_crc_log(payloads)
verified_prefix = encode_crc_log(payloads[:2])

complete = recover_crc_log(encoded)
torn = recover_crc_log(encoded[:-2])

tampered = bytearray(encoded)
tampered[_HEADER_BYTES] ^= 1
try:
    recover_crc_log(bytes(tampered))
except CrcLogCorruptionError as error:
    corruption_offset = error.frame_offset
else:
    raise AssertionError("the tampered frame was not rejected")

assert (
    complete,
    torn.payloads,
    torn.verified_prefix_length,
    torn.discarded_tail,
    corruption_offset,
) == (
    RecoveredCrcLog(payloads, len(encoded), b""),
    payloads[:2],
    len(verified_prefix),
    encoded[len(verified_prefix) : -2],
    0,
)
```

## Trade-offs and Limitations

Encoding and recovery take `O(B)` time for `B` admitted bytes. Recovery copies
verified payloads and an unverified tail into an `O(B)` frozen result; its
payload list has at most 4,096 entries. A complete header is validated before
body availability is considered, and any byte after 4,096 verified frames
raises a distinct limit error.

An incomplete body or checksum is indistinguishable from a coordinated change
that leaves a valid bounded header but makes the declared frame longer than the
available suffix. The recovery guarantee therefore requires a valid prefix plus
one suffix tear; it is not a general corruption classifier. There is no
resynchronization, file mutation, truncation, concurrent-writer coordination,
or crash-atomic filesystem claim.

The length complement catches simple header damage, and CRC-32 detects many
accidental changes, including any single-bit change in a complete bounded
frame. Neither mechanism provides authenticity, collision resistance, or
protection from deliberate modification.

## Related Snippets

<!-- catalog:related:start -->
- [Append a Fixed-Width CRC Check to a Human-Readable Identifier](../configuration-serialization/append-a-fixed-width-crc-check-to-a-human-readable-identifier.md)
- [Read and Write Size-Capped Varint Frames](../networking-protocols/read-and-write-size-capped-varint-frames.md)
- [Validate a Bounded Stage-Verify-Pointer-Switch Log](validate-a-bounded-stage-verify-pointer-switch-log.md)
<!-- catalog:related:end -->
