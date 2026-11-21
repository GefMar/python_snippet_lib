---
title: "Round Trip Mixed UTF-8 Bytes with surrogateescape"
snippet_type: idiom
use_cases:
  - data-transformation
  - interoperability
tested_python:
  - "3.14"
dependencies: []
related:
  - decode-one-bounded-strict-utf-8-stream-across-arbitrary-byte-chunks.md
  - ../testing-tooling/render-a-unified-diff-for-byte-lines-with-unknown-encoding.md
---

# Round Trip Mixed UTF-8 Bytes with surrogateescape

## Idea and Problem

Edit known Unicode text in a byte record while round-tripping undecodable UTF-8 bytes that were not changed.

Use the same surrogate escape error handler for decoding and re-encoding. It gives undecodable bytes distinct temporary code points rather than merging them into a replacement character.

## When to Use

Use this for a local repair or diagnostic workflow that normally receives UTF-8 but must preserve other bytes byte-for-byte. Only hand the intermediate string to code that can preserve its surrogate code points.

## Implementation

```python
payload = b"mode=old;\xc3\xa9tiquette=\xff\x80"
text = payload.decode("utf-8", "surrogateescape")
edited = text.replace("mode=old", "mode=new", 1)
restored = edited.encode("utf-8", "surrogateescape")
```

## Example

```python
broken = b"\xe2\x82"
round_trip = broken.decode("utf-8", "surrogateescape").encode("utf-8", "surrogateescape")
assert (restored, round_trip, text[-2:]) == (
    b"mode=new;\xc3\xa9tiquette=\xff\x80",
    broken,
    "\udcff\udc80",
)
```

## Trade-offs and Limitations

The intermediate string is not necessarily valid interchange Unicode and can fail strict UTF-8 encoding. Text transformations that drop or modify the surrogate code points also lose their source bytes. This is not encoding detection or validation. For text that must meet a UTF-8 contract, reject invalid input rather than carrying it forward.

## Related Snippets

<!-- catalog:related:start -->
- [Decode One Bounded Strict UTF-8 Stream Across Arbitrary Byte Chunks](decode-one-bounded-strict-utf-8-stream-across-arbitrary-byte-chunks.md)
- [Render a Unified Diff for Byte Lines with Unknown Encoding](../testing-tooling/render-a-unified-diff-for-byte-lines-with-unknown-encoding.md)
<!-- catalog:related:end -->
