---
title: "Encode and Decode One Canonical Size-Capped Netstring"
snippet_type: recipe
use_cases:
  - networking
  - serialization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - read-and-write-size-capped-varint-frames.md
  - ../testing-tooling/render-bounded-request-snapshots-into-a-length-framed-replay-payload.md
---

# Encode and Decode One Canonical Size-Capped Netstring

## Idea and Problem

Encode one bounded byte payload as its decimal length, a colon, the exact payload, and a trailing comma, then reject every non-canonical or incomplete representation during decoding.

The decoder accepts exactly one already materialized netstring. A seven-digit
length-field cap bounds integer parsing, while the payload cap bounds the input
and returned allocation. Canonical decimal spelling prevents alternate forms
such as `00:,` from representing the same empty payload.

## When to Use

Use this recipe when both endpoints explicitly agree on netstring framing and
one complete frame is already available in memory. It is useful for small
binary protocol fixtures, local IPC messages, or deterministic persisted
records that need a simple length-delimited representation.

Use a streaming state machine when bytes arrive incrementally or several
frames share one transport. Add a separate authenticated envelope when the
payload needs integrity or peer authentication; length framing provides
neither.

## Implementation

```python
MAX_NETSTRING_PAYLOAD_BYTES = 1_048_576
_MAX_LENGTH_DIGITS = 7
_MAX_ENCODED_BYTES = MAX_NETSTRING_PAYLOAD_BYTES + _MAX_LENGTH_DIGITS + 2


class NetstringError(ValueError):
    pass


class NetstringTooLarge(NetstringError):
    pass


def encode_netstring(payload: bytes) -> bytes:
    """Return the canonical netstring for one exact byte payload."""
    if type(payload) is not bytes:
        raise TypeError("payload must be exact bytes")
    if len(payload) > MAX_NETSTRING_PAYLOAD_BYTES:
        raise NetstringTooLarge("payload exceeds the supported byte limit")

    prefix = str(len(payload)).encode("ascii")
    return prefix + b":" + payload + b","


def decode_netstring(frame: bytes) -> bytes:
    """Decode exactly one complete canonical in-memory netstring."""
    if type(frame) is not bytes:
        raise TypeError("frame must be exact bytes")
    if len(frame) > _MAX_ENCODED_BYTES:
        raise NetstringTooLarge("encoded netstring exceeds the supported byte limit")

    colon_position = frame.find(b":", 0, _MAX_LENGTH_DIGITS + 1)
    if colon_position < 1:
        raise NetstringError("length must contain one to seven ASCII digits")

    raw_length = frame[:colon_position]
    if any(byte < ord("0") or byte > ord("9") for byte in raw_length):
        raise NetstringError("length must contain only ASCII digits")
    if len(raw_length) > 1 and raw_length[0] == ord("0"):
        raise NetstringError("length must use canonical decimal notation")

    payload_length = int(raw_length)
    if payload_length > MAX_NETSTRING_PAYLOAD_BYTES:
        raise NetstringTooLarge("declared payload exceeds the supported byte limit")

    expected_size = colon_position + 1 + payload_length + 1
    if len(frame) != expected_size:
        raise NetstringError("frame must contain exactly one complete netstring")
    if frame[-1] != ord(","):
        raise NetstringError("payload must end with a comma")

    return frame[colon_position + 1 : -1]
```

## Example

```python
empty = encode_netstring(b"")
binary_payload = b"\x00:,9"
binary = encode_netstring(binary_payload)

largest_payload = b"x" * MAX_NETSTRING_PAYLOAD_BYTES
largest = encode_netstring(largest_payload)

assert empty == b"0:,"
assert binary == b"4:\x00:,9,"
assert decode_netstring(empty) == b""
assert decode_netstring(binary) == binary_payload
assert largest.startswith(b"1048576:")
assert len(largest) == _MAX_ENCODED_BYTES
assert decode_netstring(largest) == largest_payload

malformed_frames = (
    b"",
    b":,",
    b"a:,",
    b"00:,",
    b"01:a,",
    b"12345678:,",
    b"1:a;",
    b"2:a,",
    b"1:a,tail",
    b"0:,0:,",
    b"1048577:,",
)
rejected = 0
for malformed in malformed_frames:
    try:
        decode_netstring(malformed)
    except NetstringError:
        rejected += 1
    else:
        raise AssertionError(f"accepted malformed netstring: {malformed!r}")
assert rejected == len(malformed_frames)
```

## Trade-offs and Limitations

Encoding and decoding take `O(n)` time and allocate a new `O(n)` bytes object.
The decoder bounds input before parsing and accepts only exact `bytes`; it does
not coerce text, mutable buffers, or integer-like values. The seven-digit
field limit is part of this closed profile, not a claim about every netstring
implementation.

The function handles one complete in-memory frame, not incremental reads,
multiple concatenated frames, or recovery after malformed input. It does not
provide a checksum, authentication, encryption, compression, schema, content
type, or text encoding. A caller must apply those policies separately and must
not treat successful framing as proof that payload contents are trustworthy.

## Related Snippets

<!-- catalog:related:start -->
- [Read and Write Size-Capped Varint Frames](read-and-write-size-capped-varint-frames.md)
- [Render Bounded Request Snapshots into a Length-Framed Replay Payload](../testing-tooling/render-bounded-request-snapshots-into-a-length-framed-replay-payload.md)
<!-- catalog:related:end -->
