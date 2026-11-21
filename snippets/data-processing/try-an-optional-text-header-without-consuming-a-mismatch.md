---
title: "Try an Optional Text Header Without Consuming a Mismatch"
snippet_type: recipe
use_cases:
  - data-transformation
  - parsing
tested_python:
  - "3.14"
dependencies: []
related:
  - limit-text-lines-across-arbitrary-chunks.md
  - ../storage-databases/spool-a-bounded-byte-iterable-into-a-scoped-seekable-temporary-file.md
---

# Try an Optional Text Header Without Consuming a Mismatch

## Idea and Problem

Consume an optional marker line from a seekable text stream while preserving the first data line when the marker is absent.

Save the position returned by `tell()` before probing the marker. On a
mismatch, give that position back to `seek()`. A text position is an opaque
cookie representing decoder state. The number of decoded characters is
insufficient to reconstruct it.

## When to Use

Use this at a known line boundary in a seekable text stream configured with
universal newline translation. A file importer can accept an optional
format marker before passing the same stream to its row parser. The marker
must match the entire logical line, including its terminating newline.

## Implementation

```python
from typing import TextIO


def consume_optional_header(stream: TextIO, marker: str) -> bool:
    if not marker or "\r" in marker or "\n" in marker:
        raise ValueError("marker must be one nonempty line without a terminator")

    expected = marker + "\n"
    position = stream.tell()
    if stream.readline(len(expected)) == expected:
        return True

    stream.seek(position)
    return False
```

## Example

```python
from io import BytesIO, TextIOWrapper

payload = "résumé\r\n# format:2\r\n# format:20\r\ncafé\r\n".encode()
with TextIOWrapper(BytesIO(payload), encoding="utf-8", newline=None) as stream:
    first = stream.readline()
    recognized = consume_optional_header(stream, "# format:2")
    mistaken = consume_optional_header(stream, "# format:2")
    remaining = stream.read()

assert (first, recognized, mistaken, remaining) == (
    "résumé\n",
    True,
    False,
    "# format:20\ncafé\n",
)
```

## Trade-offs and Limitations

The probe reads at most the marker length plus one decoded character.
It does not limit work performed by the underlying decoder or prevent it from
reading ahead. Reusing the cookie requires the same stream and decoder
configuration. Keep the stream under one owner's control and use explicit
`readline()` calls before the probe. On CPython, iterating a text file can
disable `tell()` until iteration ends or the stream is repositioned.

Operations may raise I/O or decoding errors, which propagate to the caller.
For non-seekable input, store the probed fragment and hand it to the parser
together with the rest of the stream.

## Related Snippets

<!-- catalog:related:start -->
- [Limit Text Lines Across Arbitrary Chunks](limit-text-lines-across-arbitrary-chunks.md)
- [Spool a Bounded Byte Iterable into a Scoped Seekable Temporary File](../storage-databases/spool-a-bounded-byte-iterable-into-a-scoped-seekable-temporary-file.md)
<!-- catalog:related:end -->
