---
title: "Register and Unregister a Bounded Single-Byte Charmap Codec"
snippet_type: integration
use_cases:
  - interoperability
  - serialization
tested_python:
  - "3.14"
dependencies: []
related:
  - ../algorithms-data-structures/build-a-canonical-unicode-caseless-comparison-key.md
  - parse-explicit-decimal-and-binary-byte-sizes.md
---

# Register and Unregister a Bounded Single-Byte Charmap Codec

## Idea and Problem

Temporarily expose a complete reversible single-byte character map through Python's standard codec lookup and text I/O interfaces.

The context manager validates one canonical codec name and an exact table of
256 unique Unicode code points. It constructs stateless, incremental, and
stream components, registers one exact search function, and unregisters that
same function during teardown so the registry cache is cleared as well.

## When to Use

Use this integration at a controlled application composition point when a
documented legacy byte encoding has a fixed one-to-one mapping and existing
code expects a normal Python encoding name. Register it once during
single-threaded setup and remove it during matching teardown. Prefer an
existing standard codec whenever one accurately describes the format.

## Implementation

```python
import codecs
import re
from collections.abc import Iterator
from contextlib import contextmanager


_CODEC_NAME = re.compile(r"[a-z][a-z0-9_]{0,31}\Z")


def _build_charmap_codec(
    name: str,
    decoding_table: str,
) -> codecs.CodecInfo:
    encoding_table = codecs.charmap_build(decoding_table)

    class Codec(codecs.Codec):
        def encode(self, input: str, errors: str = "strict"):
            return codecs.charmap_encode(input, errors, encoding_table)

        def decode(self, input: bytes, errors: str = "strict"):
            return codecs.charmap_decode(input, errors, decoding_table)

    class IncrementalEncoder(codecs.IncrementalEncoder):
        def encode(self, input: str, final: bool = False) -> bytes:
            return codecs.charmap_encode(input, self.errors, encoding_table)[0]

    class IncrementalDecoder(codecs.IncrementalDecoder):
        def decode(self, input: bytes, final: bool = False) -> str:
            return codecs.charmap_decode(input, self.errors, decoding_table)[0]

    class StreamWriter(Codec, codecs.StreamWriter):
        pass

    class StreamReader(Codec, codecs.StreamReader):
        pass

    codec = Codec()
    return codecs.CodecInfo(
        name=name,
        encode=codec.encode,
        decode=codec.decode,
        incrementalencoder=IncrementalEncoder,
        incrementaldecoder=IncrementalDecoder,
        streamwriter=StreamWriter,
        streamreader=StreamReader,
    )


@contextmanager
def registered_single_byte_charmap(
    name: str,
    decoding_table: str,
) -> Iterator[codecs.CodecInfo]:
    if not isinstance(name, str) or _CODEC_NAME.fullmatch(name) is None:
        raise ValueError("name must be a bounded lowercase codec identifier")
    if not isinstance(decoding_table, str) or len(decoding_table) != 256:
        raise ValueError("decoding_table must contain exactly 256 code points")
    if "\ufffe" in decoding_table:
        raise ValueError("U+FFFE is the charmap undefined-byte sentinel")
    if len(set(decoding_table)) != 256:
        raise ValueError("every decoded code point must be unique")

    try:
        codecs.lookup(name)
    except LookupError:
        pass
    else:
        raise ValueError(f"codec name is already registered: {name!r}")

    codec_info = _build_charmap_codec(name, decoding_table)

    def search_codec(requested_name: str) -> codecs.CodecInfo | None:
        return codec_info if requested_name == name else None

    codecs.register(search_codec)
    try:
        if codecs.lookup(name).name != name:
            raise RuntimeError("the requested codec resolved to another registration")
        yield codec_info
    finally:
        codecs.unregister(search_codec)
```

## Example

```python
from pathlib import Path
from tempfile import TemporaryDirectory


decoding_table = "".join(chr(value) for value in range(128)) + "".join(
    chr(0xE000 + value) for value in range(128)
)
raw = bytes(range(256))

with registered_single_byte_charmap(
    "example_reversible_charmap",
    decoding_table,
):
    decoded = raw.decode("example-reversible-charmap")
    round_trip = decoded.encode("example_reversible_charmap")

    incremental = codecs.getincrementaldecoder(
        "example_reversible_charmap",
    )()
    split_decode = incremental.decode(raw[:127], final=False)
    split_decode += incremental.decode(raw[127:], final=True)

    with TemporaryDirectory() as temporary_directory:
        path = Path(temporary_directory) / "encoded.bin"
        path.write_bytes(raw)
        with path.open(
            encoding="example_reversible_charmap",
            newline="",
        ) as stream:
            text_io_decode = stream.read()

    replaced = "not mapped: ☃".encode(
        "example_reversible_charmap",
        errors="replace",
    )

try:
    codecs.lookup("example_reversible_charmap")
except LookupError:
    unregistered = True
else:
    unregistered = False

try:
    with registered_single_byte_charmap(
        "example_invalid_charmap",
        decoding_table[:-1] + "\ufffe",
    ):
        pass
except ValueError:
    undefined_sentinel_rejected = True
else:
    undefined_sentinel_rejected = False

assert (
    len(decoded),
    round_trip,
    split_decode == decoded,
    text_io_decode == decoded,
    replaced.endswith(b"?"),
    unregistered,
    undefined_sentinel_rejected,
) == (256, raw, True, True, True, True, True)
```

## Trade-offs and Limitations

Codec registration is process-global, and lookup results are cached. This
context therefore assumes controlled single-threaded registration and teardown;
concurrent users of the same name can observe removal. The table is deliberately
fully reversible and cannot represent any character outside its 256 code
points under `strict` errors. Other standard error handlers are delegated to
the codec machinery. Stateful or multibyte formats need a dedicated state
machine and should not be forced into this charmap pattern.

## Related Snippets

<!-- catalog:related:start -->
- [Build a Canonical Unicode Caseless Comparison Key](../algorithms-data-structures/build-a-canonical-unicode-caseless-comparison-key.md)
- [Parse Explicit Decimal and Binary Byte Sizes](parse-explicit-decimal-and-binary-byte-sizes.md)
<!-- catalog:related:end -->
