---
title: "Decode Bounded Python Source Bytes with Python Encoding Detection"
snippet_type: testing-technique
use_cases:
  - parsing
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - extract-a-bounded-static-python-annotation-index-without-evaluation.md
  - index-bounded-python-scope-bindings-with-symtable-without-execution.md
  - ../configuration-serialization/register-and-unregister-a-bounded-single-byte-charmap-codec.md
---

# Decode Bounded Python Source Bytes with Python Encoding Detection

## Idea and Problem

Decode bounded Python source bytes under Python's own BOM and encoding-cookie rules without losing the prefix consumed during detection.

`tokenize.detect_encoding` asks a byte-oriented `readline` for at most two
physical lines and returns both the selected codec and any lines it consumed.
Replaying those lines before the unread remainder preserves the complete source
stream while respecting Python-specific encoding declarations.

## When to Use

Use this boundary before static source inspection, test-fixture analysis, or a
tool that receives Python files as bytes and must honor their declared source
encoding. The explicit total and detection-line limits keep malformed inputs
from turning the small prefix check into an unbounded read.

Do not use this as generic encoding detection. It implements Python source
rules only, and decoding text does not prove that the result is valid Python or
safe to compile, import, or execute.

## Implementation

```python
import io
import tokenize
from dataclasses import dataclass

_MAX_SOURCE_BYTES = 1 << 20
_MAX_DETECTION_LINE_BYTES = 4096


class PythonSourceDecodeError(ValueError):
    pass


@dataclass(frozen=True, slots=True)
class DecodedPythonSource:
    encoding: str
    text: str
    consumed_lines: tuple[bytes, ...]


def decode_bounded_python_source(source: bytes) -> DecodedPythonSource:
    if type(source) is not bytes:
        raise TypeError("source must be exact bytes")
    if len(source) > _MAX_SOURCE_BYTES:
        raise ValueError("source exceeds the supported byte length")

    reader = io.BytesIO(source)
    raw_consumed: list[bytes] = []

    def bounded_readline() -> bytes:
        line = reader.readline(_MAX_DETECTION_LINE_BYTES + 1)
        if len(line) > _MAX_DETECTION_LINE_BYTES:
            raise ValueError(
                "an encoding-detection line exceeds the supported length"
            )
        if line:
            raw_consumed.append(line)
        return line

    try:
        encoding, _ = tokenize.detect_encoding(bounded_readline)
        replayed = b"".join(raw_consumed) + reader.read()
        text = replayed.decode(encoding)
    except (LookupError, SyntaxError, UnicodeError):
        raise PythonSourceDecodeError(
            "Python source encoding is invalid"
        ) from None

    return DecodedPythonSource(
        encoding=encoding,
        text=text,
        consumed_lines=tuple(raw_consumed),
    )
```

## Example

```python
plain = decode_bounded_python_source(b"answer = 42\n")
latin_1 = decode_bounded_python_source(
    b'# coding: latin-1\nlabel = "caf\xe9"\n'
)
with_bom = decode_bounded_python_source(
    b"\xef\xbb\xbf# coding: utf-8\nvalue = 1\n"
)
empty = decode_bounded_python_source(b"")

try:
    decode_bounded_python_source(
        b"\xef\xbb\xbf# coding: latin-1\nvalue = 1\n"
    )
except PythonSourceDecodeError as error:
    conflict_message = str(error)
else:
    conflict_message = "not rejected"

assert (
    plain.encoding,
    plain.text,
    latin_1.encoding,
    latin_1.text,
    with_bom.encoding,
    with_bom.text.startswith("\ufeff"),
    with_bom.consumed_lines[0].startswith(b"\xef\xbb\xbf"),
    empty,
    conflict_message,
) == (
    "utf-8",
    "answer = 42\n",
    "iso-8859-1",
    '# coding: latin-1\nlabel = "café"\n',
    "utf-8-sig",
    False,
    True,
    DecodedPythonSource("utf-8", "", ()),
    "Python source encoding is invalid",
)
```

## Trade-offs and Limitations

Detection reads at most two bounded physical lines, and decoding is linear in
the source size with another complete text allocation. The input is capped at
one MiB, while either line consulted during detection is capped at 4096 bytes.
The returned `consumed_lines` are the exact non-empty source lines supplied to
`tokenize.detect_encoding`, before that API can remove a UTF-8 BOM from its own
returned prefix. Replaying the captured raw lines preserves the original bytes;
decoding them with the selected `utf-8-sig` codec removes the signature only
from the resulting text.

Cookie/BOM disagreement, unknown codecs, and invalid encoded bytes share one
value-free domain error so untrusted source fragments are not copied into logs.
Type, total-size, and detection-line-limit failures remain distinct programmer
or admission errors. The function does not normalize newlines, validate Python
syntax, tokenize the body, inspect an AST, follow imports, or execute code.

## Related Snippets

<!-- catalog:related:start -->
- [Extract a Bounded Static Python Annotation Index without Evaluation](extract-a-bounded-static-python-annotation-index-without-evaluation.md)
- [Index Bounded Python Scope Bindings with symtable without Execution](index-bounded-python-scope-bindings-with-symtable-without-execution.md)
- [Register and Unregister a Bounded Single-Byte Charmap Codec](../configuration-serialization/register-and-unregister-a-bounded-single-byte-charmap-codec.md)
<!-- catalog:related:end -->
