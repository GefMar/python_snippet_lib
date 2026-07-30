---
title: "Read One Bounded Async Byte Field with Either Line Feed or NUL Terminator"
snippet_type: recipe
use_cases:
  - networking
  - parsing
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - read-and-write-size-capped-varint-frames.md
  - parse-one-bounded-printable-ascii-http-1-field-section-with-exact-crlf-framing.md
  - release-a-pooled-response-connection-only-after-clean-eof.md
---

# Read One Bounded Async Byte Field with Either Line Feed or NUL Terminator

## Idea and Problem

Read one byte field that ends at either line feed or NUL while preserving which terminator matched and enforcing a 4,096-byte payload limit.

Python 3.13 or later lets `asyncio.StreamReader.readuntil()` accept a tuple of
separators. Both separators in this profile are one byte, so removing the last
byte yields the payload without an ambiguous longest-match rule. Empty payloads
are valid and return as `b""` with their terminator.

## When to Use

Use this at a protocol boundary whose peer has explicitly negotiated line-feed
or NUL framing and whose field is opaque bytes at this layer. Construct the
production reader through the normal stream API with the same limit, for
example `asyncio.open_connection(host, port, limit=MAX_FIELD_PAYLOAD)` or
`asyncio.start_server(handler, host, port, limit=MAX_FIELD_PAYLOAD)`.

The connection owner must treat `ByteFieldFramingError` as terminal: close the
writer and abandon the reader rather than attempting another frame. Decode or
validate the returned payload separately under the protocol's content rules.

## Implementation

```python
import asyncio

MAX_FIELD_PAYLOAD = 4_096
_FIELD_TERMINATORS = (b"\n", b"\0")


class ByteFieldFramingError(ValueError):
    """The next field is incomplete or outside the framing profile."""


async def read_bounded_byte_field(
    reader: asyncio.StreamReader,
) -> tuple[bytes, bytes]:
    if not isinstance(reader, asyncio.StreamReader):
        raise TypeError("reader must be an asyncio StreamReader")
    try:
        framed = await reader.readuntil(_FIELD_TERMINATORS)
    except (asyncio.IncompleteReadError, asyncio.LimitOverrunError) as error:
        raise ByteFieldFramingError("field is incomplete or exceeds the payload limit") from error

    payload, terminator = framed[:-1], framed[-1:]
    if len(payload) > MAX_FIELD_PAYLOAD:
        raise ByteFieldFramingError("field exceeds the payload limit")
    return payload, terminator


```

## Example

```python
async def _test_only_feed(
    reader: asyncio.StreamReader,
    *chunks: bytes,
) -> None:
    # Test-only fixture: production readers come from a connection or server.
    for chunk in chunks:
        reader.feed_data(chunk)
        await asyncio.sleep(0)
    reader.feed_eof()


async def _read_test_chunks(*chunks: bytes) -> tuple[bytes, bytes]:
    reader = asyncio.StreamReader(limit=MAX_FIELD_PAYLOAD)
    feeder = asyncio.create_task(_test_only_feed(reader, *chunks))
    try:
        return await read_bounded_byte_field(reader)
    finally:
        await feeder


async def main() -> tuple[bytes, bytes]:
    assert await _read_test_chunks(b"alpha\n") == (b"alpha", b"\n")
    assert await _read_test_chunks(b"be", b"ta", b"\0") == (
        b"beta",
        b"\0",
    )
    assert await _read_test_chunks(b"x" * MAX_FIELD_PAYLOAD, b"\n") == (
        b"x" * MAX_FIELD_PAYLOAD,
        b"\n",
    )

    consecutive = asyncio.StreamReader(limit=MAX_FIELD_PAYLOAD)
    await _test_only_feed(consecutive, b"first\n\0")
    assert await read_bounded_byte_field(consecutive) == (b"first", b"\n")
    assert await read_bounded_byte_field(consecutive) == (b"", b"\0")

    for malformed in (
        (b"incomplete",),
        (b"x" * (MAX_FIELD_PAYLOAD + 1), b"\n"),
    ):
        try:
            await _read_test_chunks(*malformed)
        except ByteFieldFramingError:
            pass
        else:
            raise AssertionError("malformed field was accepted")

    return await _read_test_chunks(b"\n")


assert asyncio.run(main()) == (b"", b"\n")
```

## Trade-offs and Limitations

This is a closed binary framing profile, not a text-line reader. Carriage return
before line feed remains part of the payload, and an embedded NUL always ends
the field. The limit bounds a successful payload; it is not a general memory,
rate, idle-time, or total-message budget. Apply an outer timeout and connection
budget when the peer can stall.

The helper cannot inspect or retrofit the reader's configured limit through a
public API. A production owner must create the reader with
`limit=MAX_FIELD_PAYLOAD`; a larger limit can retain more peer-controlled data
before this function rejects the returned payload.

`LimitOverrunError` leaves the over-limit data in the reader's buffer, while an
incomplete read has reached EOF. The wrapper deliberately maps both cases to
one terminal error, so retrying on the same reader is unsupported. Only one
coroutine may read a `StreamReader` at a time. Cancellation is not converted
into a framing error and must be handled by the connection owner's lifecycle
policy.

## Related Snippets

<!-- catalog:related:start -->
- [Read and Write Size-Capped Varint Frames](read-and-write-size-capped-varint-frames.md)
- [Parse One Bounded Printable-ASCII HTTP/1 Field Section with Exact CRLF Framing](parse-one-bounded-printable-ascii-http-1-field-section-with-exact-crlf-framing.md)
- [Release a Pooled Response Connection Only After Clean EOF](release-a-pooled-response-connection-only-after-clean-eof.md)
<!-- catalog:related:end -->
