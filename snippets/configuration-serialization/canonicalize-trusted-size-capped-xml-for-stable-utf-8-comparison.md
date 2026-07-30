---
title: "Canonicalize Trusted Size-Capped XML for Stable UTF-8 Comparison"
snippet_type: recipe
use_cases:
  - serialization
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - parse-a-bounded-xml-envelope-with-closed-variant-dispatch.md
  - create-reproducible-gzip-bytes-with-an-explicit-zero-modification-time.md
  - render-a-stable-unified-diff-for-nested-json-values.md
---

# Canonicalize Trusted Size-Capped XML for Stable UTF-8 Comparison

## Idea and Problem

Canonicalize one small trusted XML string into bounded UTF-8 bytes so equivalent attribute order and empty-element syntax can be compared consistently.

`xml.etree.ElementTree.canonicalize` implements C14N 2.0 and writes text, not
bytes. This wrapper fixes its options, rejects DTD-bearing input before parsing,
and supplies a text sink that encodes and counts every write before retaining
it. The result therefore has both a 64 KiB source limit and a 256 KiB canonical
UTF-8 limit.

## When to Use

Use this recipe when the complete XML text is already trusted, available in
memory, and needed for a stable local equality comparison or review fixture.
Both producers must agree on this exact canonicalization profile: comments are
removed, surrounding text is preserved, namespace prefixes are not rewritten,
and no attributes or tags are excluded.

Do not use this wrapper as a hostile-input defense. Use a hardened XML workflow
for untrusted documents, and use a schema-aware parser when application meaning
depends on a vocabulary rather than serialized form.

## Implementation

```python
import re
import xml.etree.ElementTree as ElementTree
from collections.abc import Callable

_MAX_XML_SOURCE_BYTES = 64 * 1_024
_MAX_CANONICAL_XML_BYTES = 256 * 1_024
_DOCTYPE_MARKER = re.compile(r"<!DOCTYPE", flags=re.ASCII | re.IGNORECASE)


class CanonicalXmlError(ValueError):
    pass


class _BoundedUtf8TextSink:
    __slots__ = ("_chunks", "_max_bytes", "_size")

    def __init__(self, *, max_bytes: int) -> None:
        self._chunks: list[bytes] = []
        self._max_bytes = max_bytes
        self._size = 0

    def write(self, text: str) -> None:
        if type(text) is not str:
            raise TypeError("canonical XML output must be text")
        encoded = text.encode("utf-8", errors="strict")
        if len(encoded) > self._max_bytes - self._size:
            raise CanonicalXmlError("canonical XML exceeds the output byte limit")
        self._chunks.append(encoded)
        self._size += len(encoded)

    def to_bytes(self) -> bytes:
        return b"".join(self._chunks)


def canonicalize_trusted_xml(xml_text: str) -> bytes:
    """Return bounded C14N 2.0 bytes for one trusted XML string."""
    if type(xml_text) is not str:
        raise TypeError("xml_text must be an exact string")
    try:
        source_bytes = xml_text.encode("utf-8", errors="strict")
    except UnicodeEncodeError as error:
        raise CanonicalXmlError("XML text is not strict UTF-8 encodable") from error
    if not 1 <= len(source_bytes) <= _MAX_XML_SOURCE_BYTES:
        raise CanonicalXmlError("XML source byte length is outside 1..65536")
    if _DOCTYPE_MARKER.search(xml_text) is not None:
        raise CanonicalXmlError("DTD declarations are not accepted")

    sink = _BoundedUtf8TextSink(max_bytes=_MAX_CANONICAL_XML_BYTES)
    try:
        ElementTree.canonicalize(
            xml_data=xml_text,
            out=sink,
            with_comments=False,
            strip_text=False,
            rewrite_prefixes=False,
        )
    except ElementTree.ParseError as error:
        raise CanonicalXmlError("XML text is malformed") from error
    return sink.to_bytes()
```

## Example

```python
def raises_canonical_xml_error(operation: Callable[[], object]) -> bool:
    try:
        operation()
    except CanonicalXmlError:
        return True
    return False


attribute_first = '<root z="2" a="1"><empty /></root>'
attribute_second = '<root a="1" z="2"><empty></empty></root>'
expected = b'<root a="1" z="2"><empty></empty></root>'

assert canonicalize_trusted_xml(attribute_first) == expected
assert canonicalize_trusted_xml(attribute_second) == expected
assert canonicalize_trusted_xml("<root><!-- removed --><x> a </x></root>") == (
    b"<root><x> a </x></root>"
)

assert raises_canonical_xml_error(lambda: canonicalize_trusted_xml("<root>"))
assert raises_canonical_xml_error(
    lambda: canonicalize_trusted_xml("<!DOCTYPE root><root />")
)
assert raises_canonical_xml_error(
    lambda: canonicalize_trusted_xml("<!doctype root><root />")
)
assert raises_canonical_xml_error(lambda: canonicalize_trusted_xml("\ud800"))

tiny_sink = _BoundedUtf8TextSink(max_bytes=4)
tiny_sink.write("éé")
assert tiny_sink.to_bytes() == b"\xc3\xa9\xc3\xa9"
assert raises_canonical_xml_error(lambda: tiny_sink.write("x"))
assert tiny_sink.to_bytes() == b"\xc3\xa9\xc3\xa9"
```

## Trade-offs and Limitations

Parsing and sink writes are linear in their bounded inputs, while the
canonicalizer may also sort names within an element; `O(n log n)` is a safe
worst-case time bound for `n` bounded input and output bytes. The sink retains
at most 256 KiB of encoded chunks plus parser and per-chunk overhead, then
allocates the final joined byte string. The lexical DTD check is deliberately
conservative and also rejects the marker inside comments or text.

Canonical byte equality is not schema-aware or application-level semantic
equality. This recipe does not validate a schema, authenticate content, create
or verify signatures or digests, read files, fetch network resources, or make
the standard-library XML parser safe for hostile documents. Different C14N
profiles are separate contracts and must not be mixed silently.

## Related Snippets

<!-- catalog:related:start -->
- [Parse a Bounded XML Envelope with Closed Variant Dispatch](parse-a-bounded-xml-envelope-with-closed-variant-dispatch.md)
- [Create Reproducible gzip Bytes with an Explicit Zero Modification Time](create-reproducible-gzip-bytes-with-an-explicit-zero-modification-time.md)
- [Render a Stable Unified Diff for Nested JSON Values](render-a-stable-unified-diff-for-nested-json-values.md)
<!-- catalog:related:end -->
