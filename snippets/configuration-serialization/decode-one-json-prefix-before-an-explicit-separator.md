---
title: "Decode One JSON Prefix Before an Explicit Separator"
snippet_type: recipe
use_cases:
  - parsing
  - interoperability
tested_python:
  - "3.14"
dependencies: []
related:
  - parse-bounded-utf-8-json-without-duplicate-object-names-or-non-finite-numbers.md
  - ../data-processing/decode-bounded-json-lines-across-arbitrary-text-chunks.md
---

# Decode One JSON Prefix Before an Explicit Separator

## Idea and Problem

Decode a leading JSON value and locate its following separator without mistaking punctuation inside a string for framing.

Use the end offset returned by the standard decoder, consume only JSON whitespace after the value, and require the separator at that exact position. The remainder stays unparsed.

## When to Use

Use this for a trusted, in-memory record format that defines one JSON prefix followed by a semicolon and raw text. Use a standard framing format instead when you control the protocol.

## Implementation

```python
from decimal import Decimal
from json import JSONDecoder


JSON_SPACE = " \t\r\n"


def reject_constant(token: str) -> None:
    raise ValueError(f"not a JSON number: {token}")


def split_json_prefix(text: str, max_chars: int = 4096) -> tuple[object, str]:
    if len(text) > max_chars:
        raise ValueError("record exceeds the character limit")

    source = text.lstrip(JSON_SPACE)
    decoder = JSONDecoder(parse_float=Decimal, parse_constant=reject_constant)
    value, end = decoder.raw_decode(source)
    suffix = source[end:].lstrip(JSON_SPACE)
    if not suffix.startswith(";"):
        raise ValueError("missing separator after JSON value")
    return value, suffix[1:]
```

## Example

```python
parsed = split_json_prefix('  {"label": "a;b", "ratio": 0.10} \t; raw; tail')
errors = []
for text in ("12x; raw", "NaN; raw", "12 raw"):
    try:
        split_json_prefix(text)
    except ValueError:
        errors.append(text)

assert (parsed, errors, split_json_prefix("false;")) == (
    ({"label": "a;b", "ratio": Decimal("0.10")}, " raw; tail"),
    ["12x; raw", "NaN; raw", "12 raw"],
    (False, ""),
)
```

## Trade-offs and Limitations

This is not a general stream splitter or strict policy for untrusted JSON. The size limit applies to the whole string, not bytes already read. Duplicate object names follow the decoder's default last-value behavior. The returned suffix is arbitrary text and needs its own validation if used elsewhere.

## Related Snippets

<!-- catalog:related:start -->
- [Parse Bounded UTF-8 JSON Without Duplicate Object Names or Non-Finite Numbers](parse-bounded-utf-8-json-without-duplicate-object-names-or-non-finite-numbers.md)
- [Decode Bounded JSON Lines Across Arbitrary Text Chunks](../data-processing/decode-bounded-json-lines-across-arbitrary-text-chunks.md)
<!-- catalog:related:end -->
