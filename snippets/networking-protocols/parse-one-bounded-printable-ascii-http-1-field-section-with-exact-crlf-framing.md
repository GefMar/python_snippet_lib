---
title: "Parse One Bounded Printable-ASCII HTTP/1 Field Section with Exact CRLF Framing"
snippet_type: recipe
use_cases:
  - networking
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - extract-bounded-singleton-http-headers-with-explicit-rules.md
  - encode-a-bounded-http-1-1-chunked-body-for-protocol-tests.md
  - parse-a-bounded-ascii-media-type-value.md
---

# Parse One Bounded Printable-ASCII HTTP/1 Field Section with Exact CRLF Framing

## Idea and Problem

Parse the bytes after an HTTP/1 start-line into an immutable field tuple while accepting only one small, exact wire-format subset.

The input contains only the field section, including its terminal empty CRLF
line. Scanning literal CRLF pairs keeps bare carriage returns, bare line feeds,
obsolete folding, and trailing bytes visible as errors. Each accepted name is
an ASCII token, and each value contains printable ASCII only.

## When to Use

Use this recipe for bounded protocol fixtures or a narrow adapter that has
already isolated the bytes between a start-line and any later message data. An
empty field section is exactly `b"\r\n"`; one field needs its own CRLF followed
by that terminal empty line.

Choose total-byte, field-count, and line-content limits from the surrounding
protocol boundary. The parser preserves original name case, order, and
duplicates so a later field-specific layer can decide whether a field is a
singleton, list, or invalid repetition.

## Implementation

```python
_MAX_SUPPORTED_TOTAL_BYTES = 64 * 1024
_MAX_SUPPORTED_FIELDS = 256
_MAX_SUPPORTED_LINE_CONTENT_BYTES = 8 * 1024
_HTTP_TOKEN_BYTES = frozenset(
    b"!#$%&'*+-.^_`|~0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"
)


class Http1FieldSectionError(ValueError):
    pass


def _validated_limit(
    value: object,
    *,
    name: str,
    minimum: int,
    maximum: int,
) -> int:
    if type(value) is not int:
        raise TypeError(f"{name} must be an exact integer")
    if not minimum <= value <= maximum:
        raise ValueError(f"{name} is outside the supported range")
    return value


def parse_http1_field_section(
    section: bytes,
    *,
    max_total_bytes: int = 16 * 1024,
    max_fields: int = 64,
    max_line_content_bytes: int = 4 * 1024,
) -> tuple[tuple[str, str], ...]:
    total_limit = _validated_limit(
        max_total_bytes,
        name="max_total_bytes",
        minimum=2,
        maximum=_MAX_SUPPORTED_TOTAL_BYTES,
    )
    field_limit = _validated_limit(
        max_fields,
        name="max_fields",
        minimum=0,
        maximum=_MAX_SUPPORTED_FIELDS,
    )
    line_limit = _validated_limit(
        max_line_content_bytes,
        name="max_line_content_bytes",
        minimum=2,
        maximum=_MAX_SUPPORTED_LINE_CONTENT_BYTES,
    )
    if type(section) is not bytes:
        raise TypeError("section must be exact immutable bytes")
    if len(section) > total_limit:
        raise Http1FieldSectionError("field section exceeds max_total_bytes")

    fields: list[tuple[str, str]] = []
    position = 0
    while True:
        delimiter = section.find(b"\r\n", position)
        if delimiter < 0:
            raise Http1FieldSectionError("field section is missing an exact CRLF delimiter")

        line = section[position:delimiter]
        if len(line) > line_limit:
            raise Http1FieldSectionError("field line content exceeds max_line_content_bytes")
        position = delimiter + 2
        if not line:
            if position != len(section):
                raise Http1FieldSectionError("terminal empty line must end the field section")
            return tuple(fields)

        if len(fields) >= field_limit:
            raise Http1FieldSectionError("field count exceeds max_fields")
        if any(octet < 0x20 or octet > 0x7E for octet in line):
            raise Http1FieldSectionError("field lines must contain only printable ASCII")

        colon = line.find(b":")
        if colon <= 0:
            raise Http1FieldSectionError("field line must contain a nonempty name")
        name_bytes = line[:colon]
        if any(octet not in _HTTP_TOKEN_BYTES for octet in name_bytes):
            raise Http1FieldSectionError("field name must be an ASCII HTTP token")

        value_bytes = line[colon + 1 :].strip(b" ")
        fields.append(
            (
                name_bytes.decode("ascii"),
                value_bytes.decode("ascii"),
            )
        )
```

## Example

```python
wire = (
    b"Host: example.test\r\nX-Trace:   alpha  beta   \r\nx-trace: duplicate\r\nEmpty:    \r\n\r\n"
)
longest_line = max(len(line) for line in wire.split(b"\r\n") if line)
parsed = parse_http1_field_section(
    wire,
    max_total_bytes=len(wire),
    max_fields=4,
    max_line_content_bytes=longest_line,
)
empty = parse_http1_field_section(b"\r\n", max_total_bytes=2, max_fields=0)


def is_rejected(candidate: object, **limits: object) -> bool:
    try:
        parse_http1_field_section(candidate, **limits)
    except (TypeError, ValueError):
        return True
    return False


invalid_sections = (
    b"",
    b"Host: value\r\n",
    b"\r\ntrailing",
    b"Host: value\n\n",
    b"Host: value\r\r\n\r\n",
    b"X-Test: one\r\n folded: two\r\n\r\n",
    b"Host : value\r\n\r\n",
    b": value\r\n\r\n",
    b"Bad\tName: value\r\n\r\n",
    b"X-Test: a\tb\r\n\r\n",
    b"X-Test: a\x00b\r\n\r\n",
    b"X-Test: \x7f\r\n\r\n",
    b"X-Test: \x80\r\n\r\n",
    bytearray(b"\r\n"),
)
invalid_limit_cases = (
    {"max_total_bytes": True},
    {"max_total_bytes": 1},
    {"max_total_bytes": _MAX_SUPPORTED_TOTAL_BYTES + 1},
    {"max_fields": 1.0},
    {"max_fields": -1},
    {"max_fields": _MAX_SUPPORTED_FIELDS + 1},
    {"max_line_content_bytes": False},
    {"max_line_content_bytes": 1},
    {"max_line_content_bytes": _MAX_SUPPORTED_LINE_CONTENT_BYTES + 1},
)
limit_overflows = (
    is_rejected(wire, max_total_bytes=len(wire) - 1),
    is_rejected(b"A: 1\r\nB: 2\r\n\r\n", max_fields=1),
    is_rejected(b"Long: x\r\n\r\n", max_line_content_bytes=6),
)

assert (
    parsed,
    empty,
    sum(is_rejected(candidate) for candidate in invalid_sections),
    sum(is_rejected(b"\r\n", **limits) for limits in invalid_limit_cases),
    limit_overflows,
) == (
    (
        ("Host", "example.test"),
        ("X-Trace", "alpha  beta"),
        ("x-trace", "duplicate"),
        ("Empty", ""),
    ),
    (),
    14,
    9,
    (True, True, True),
)
```

## Trade-offs and Limitations

The scan is linear in the bounded input and allocates one bytes slice and two
strings per accepted field before freezing the result. The total limit checks
bytes that the caller has already materialized; it is not a streaming read
budget. Line length excludes its CRLF delimiter, and equality with every limit
is accepted.

This is a deliberately closed subset of HTTP/1 field syntax. It rejects HTAB,
obsolete text, controls, DEL, non-ASCII bytes, obsolete line folding, and
tolerant bare-LF handling. It strips only leading and trailing SP from values,
preserves interior spaces, and performs no lowercasing, combination, duplicate
rejection, or field-specific interpretation.

The input has no start-line, body, trailer section, or bytes after the terminal
empty line. This parser therefore makes no complete-message framing,
intermediary-consistency, or request-smuggling guarantee. Use a maintained HTTP
implementation at a real network boundary.

## Related Snippets

<!-- catalog:related:start -->
- [Extract Bounded Singleton HTTP Headers with Explicit Rules](extract-bounded-singleton-http-headers-with-explicit-rules.md)
- [Encode a Bounded HTTP/1.1 Chunked Body for Protocol Tests](encode-a-bounded-http-1-1-chunked-body-for-protocol-tests.md)
- [Parse a Bounded ASCII Media Type Value](parse-a-bounded-ascii-media-type-value.md)
<!-- catalog:related:end -->
