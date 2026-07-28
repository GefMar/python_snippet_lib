---
title: "Decode One Bounded Strict UTF-8 Stream Across Arbitrary Byte Chunks"
snippet_type: recipe
use_cases:
  - parsing
  - resource-management
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - parse-bounded-utf-8-json-without-duplicate-object-names-or-non-finite-numbers.md
  - ../data-processing/limit-text-lines-across-arbitrary-chunks.md
  - ../networking-protocols/parse-a-bounded-percent-encoded-query-string-as-strict-utf-8-with-explicit-duplicate-rules.md
---

# Decode One Bounded Strict UTF-8 Stream Across Arbitrary Byte Chunks

## Idea and Problem

Decode one bounded UTF-8 stream without treating byte-chunk boundaries as character boundaries.

An incremental decoder retains an incomplete multibyte sequence until later
chunks arrive. The complete tuple is validated before decoding begins, and a
final flush rejects a truncated sequence at the end instead of silently
discarding it.

## When to Use

Use this recipe when one finite byte stream has already been collected as
arbitrary chunks and the caller needs strict UTF-8 text only after the entire
stream succeeds. Empty input and empty chunks are valid, and a UTF-8 byte-order
mark is retained as the `\ufeff` character rather than removed.

Use a streaming text consumer when decoded prefixes must be processed before
the source ends. Define framing separately when several logical documents share
one byte stream, and apply normalization or application parsing only after this
decoding boundary.

## Implementation

```python
from codecs import getincrementaldecoder

_MAX_CHUNKS = 4_096
_MAX_CHUNK_BYTES = 65_536
_MAX_TOTAL_BYTES = 1_048_576


class StrictUtf8StreamError(ValueError):
    """Raised when the chunks are not one complete strict UTF-8 stream."""


def decode_strict_utf8_chunks(chunks: tuple[bytes, ...]) -> str:
    """Decode one fully validated bounded UTF-8 stream."""
    if type(chunks) is not tuple:
        raise TypeError("chunks must be an exact tuple")
    if len(chunks) > _MAX_CHUNKS:
        raise ValueError("chunk count exceeds the supported limit")

    total_bytes = 0
    for index, chunk in enumerate(chunks):
        if type(chunk) is not bytes:
            raise TypeError(f"chunks[{index}] must be exact bytes")
        if len(chunk) > _MAX_CHUNK_BYTES:
            raise ValueError(f"chunks[{index}] exceeds the per-chunk byte limit")
        if len(chunk) > _MAX_TOTAL_BYTES - total_bytes:
            raise ValueError("chunk bytes exceed the total supported limit")
        total_bytes += len(chunk)

    decoder = getincrementaldecoder("utf-8")(errors="strict")
    decoded_parts: list[str] = []
    try:
        for chunk in chunks:
            decoded = decoder.decode(chunk, final=False)
            if decoded:
                decoded_parts.append(decoded)
        final_text = decoder.decode(b"", final=True)
    except UnicodeDecodeError:
        raise StrictUtf8StreamError("chunks are not one complete strict UTF-8 stream") from None

    if final_text:
        decoded_parts.append(final_text)
    return "".join(decoded_parts)
```

## Example

```python
chunks = (b"\xef", b"\xbb\xbfprice: \xe2", b"\x82", b"\xac")
text = decode_strict_utf8_chunks(chunks)

try:
    decode_strict_utf8_chunks((b"incomplete: \xe2", b"\x82"))
except StrictUtf8StreamError:
    incomplete_rejected = True
else:
    incomplete_rejected = False

assert (text, decode_strict_utf8_chunks(()), incomplete_rejected) == (
    "\ufeffprice: \u20ac",
    "",
    True,
)
```

## Trade-offs and Limitations

Prevalidation takes `O(chunks)` time, and decoding takes `O(total_bytes)` time.
The decoded fragments and final joined string use bounded memory proportional
to the admitted stream. Input chunks already exist in memory, and joining can
temporarily retain both the fragments and the completed string.

The function raises `StrictUtf8StreamError` before returning any text when a
chunk is malformed, but it is a one-shot in-memory operation rather than a
transactional external write. It deliberately does not report byte offsets,
replace malformed data, remove a byte-order mark, normalize Unicode, count
grapheme clusters, frame documents, or stream partial text. Exact tuples and
exact `bytes` keep coercion and mutable-buffer behavior outside the contract.

## Related Snippets

<!-- catalog:related:start -->
- [Parse Bounded UTF-8 JSON Without Duplicate Object Names or Non-Finite Numbers](parse-bounded-utf-8-json-without-duplicate-object-names-or-non-finite-numbers.md)
- [Limit Text Lines Across Arbitrary Chunks](../data-processing/limit-text-lines-across-arbitrary-chunks.md)
- [Parse a Bounded Percent-Encoded Query String as Strict UTF-8 with Explicit Duplicate Rules](../networking-protocols/parse-a-bounded-percent-encoded-query-string-as-strict-utf-8-with-explicit-duplicate-rules.md)
<!-- catalog:related:end -->
