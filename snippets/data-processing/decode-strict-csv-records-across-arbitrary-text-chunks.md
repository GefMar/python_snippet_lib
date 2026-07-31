---
title: "Decode Strict CSV Records Across Arbitrary Text Chunks"
snippet_type: pattern
use_cases:
  - data-transformation
  - parsing
  - resource-management
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - decode-bounded-json-lines-across-arbitrary-text-chunks.md
  - limit-text-lines-across-arbitrary-chunks.md
  - validate-parsed-csv-rows-with-bounded-structured-problems.md
---

# Decode Strict CSV Records Across Arbitrary Text Chunks

## Idea and Problem

Decode a bounded CSV document without treating arbitrary producer chunks as CSV lines.

Python's `csv.reader` consumes an iterable of physical lines. Passing network,
decompression, or text-decoder chunks directly is incorrect because a chunk
may end in the middle of a field, doubled quote, or CRLF terminator. The
adapter below reconstructs bounded physical lines while preserving their exact
LF, CRLF, or lone-CR endings. The standard parser remains responsible for CSV
records, including quoted fields that span several physical lines.

The function freezes one explicit dialect: comma delimiters, double-quote
quoting, doubled embedded quotes, no escape character, no skipped initial
spaces, minimal quoting, and `strict=True`. Its result is materialized only
after the complete input succeeds, so no accepted prefix escapes when a later
chunk is invalid.

## When to Use

Use this pattern after bytes have already been decoded into trusted text
chunks and a bounded in-memory table is the desired result. It fits small
imports, protocol fixtures, and batch responses whose chunk boundaries must
not affect CSV semantics.

Use an incremental row consumer when retaining every row is unnecessary. Use
an incremental character decoder first when byte sequences can cross chunks.
Choose a dedicated CSV library or a schema layer when the dialect is
configurable, fields need typed conversion, or error locations and recovery
are part of the interface.

## Implementation

```python
import csv
from collections.abc import Iterable, Iterator

_MAX_CSV_CHUNKS = 65_536
_MAX_CSV_INPUT_CHARACTERS = 1_048_576
_MAX_CSV_PHYSICAL_LINE_CHARACTERS = 65_536
_MAX_CSV_RECORDS = 10_000
_MAX_CSV_FIELDS_PER_RECORD = 256
_MAX_CSV_CELLS = 100_000
_MAX_CSV_FIELD_CHARACTERS = 65_536


def _bounded_csv_physical_lines(chunks: Iterable[str]) -> Iterator[str]:
    chunk_count = 0
    input_characters = 0
    line_characters = 0
    line_parts: list[str] = []
    pending_carriage_return = False

    for chunk in chunks:
        chunk_count += 1
        if chunk_count > _MAX_CSV_CHUNKS:
            raise ValueError("CSV input contains too many chunks")
        if type(chunk) is not str:
            raise TypeError("every CSV chunk must be an exact string")

        input_characters += len(chunk)
        if input_characters > _MAX_CSV_INPUT_CHARACTERS:
            raise ValueError("CSV input exceeds the character limit")
        try:
            chunk.encode("utf-8", errors="strict")
        except UnicodeEncodeError:
            raise ValueError("CSV chunks must not contain surrogate code points") from None

        for character in chunk:
            if pending_carriage_return:
                if character == "\n":
                    line_parts.append(character)
                    line_characters += 1
                    if line_characters > _MAX_CSV_PHYSICAL_LINE_CHARACTERS:
                        raise ValueError("a CSV physical line exceeds the character limit")
                    yield "".join(line_parts)
                    line_parts.clear()
                    line_characters = 0
                    pending_carriage_return = False
                    continue

                yield "".join(line_parts)
                line_parts.clear()
                line_characters = 0
                pending_carriage_return = False

            line_parts.append(character)
            line_characters += 1
            if line_characters > _MAX_CSV_PHYSICAL_LINE_CHARACTERS:
                raise ValueError("a CSV physical line exceeds the character limit")

            if character == "\r":
                pending_carriage_return = True
            elif character == "\n":
                yield "".join(line_parts)
                line_parts.clear()
                line_characters = 0

    if line_parts:
        yield "".join(line_parts)


def decode_strict_csv_chunks(
    chunks: Iterable[str],
) -> tuple[tuple[str, ...], ...]:
    """Decode one bounded CSV document independently of chunk boundaries."""
    if csv.field_size_limit() < _MAX_CSV_FIELD_CHARACTERS:
        raise RuntimeError("the process CSV field-size limit is unexpectedly low")

    reader = csv.reader(
        _bounded_csv_physical_lines(chunks),
        delimiter=",",
        quotechar='"',
        doublequote=True,
        escapechar=None,
        skipinitialspace=False,
        quoting=csv.QUOTE_MINIMAL,
        strict=True,
    )
    records: list[tuple[str, ...]] = []
    cell_count = 0
    try:
        for row in reader:
            if len(records) >= _MAX_CSV_RECORDS:
                raise ValueError("CSV input contains too many records")
            if len(row) > _MAX_CSV_FIELDS_PER_RECORD:
                raise ValueError("a CSV record contains too many fields")
            cell_count += len(row)
            if cell_count > _MAX_CSV_CELLS:
                raise ValueError("CSV input contains too many cells")
            if any(len(field) > _MAX_CSV_FIELD_CHARACTERS for field in row):
                raise ValueError("a decoded CSV field exceeds the character limit")
            records.append(tuple(row))
    except csv.Error as error:
        raise ValueError(f"invalid CSV under the fixed dialect: {error}") from None

    return tuple(records)
```

## Example

```python
import csv as _csv_oracle
from io import StringIO

document = (
    'name,note\r\nalpha,"first\r\nsecond"\r\nbeta,"a doubled ""quote"""\r\ngamma,a"literal quote\r'
)
expected = (
    ("name", "note"),
    ("alpha", "first\r\nsecond"),
    ("beta", 'a doubled "quote"'),
    ("gamma", 'a"literal quote'),
)

# A CRLF, quoted field, and doubled quote all cross producer boundaries.
chunks = (
    "name,note\r",
    '\nalpha,"first\r',
    '\nsecond"\r\nbeta,"a doubled "',
    '"quote"""\r\ngamma,a"literal quote\r',
    "",
)
assert decode_strict_csv_chunks(chunks) == expected

# Every single cut and one-character chunking has the joined-file result.
oracle = tuple(
    tuple(row)
    for row in _csv_oracle.reader(
        StringIO(document, newline=""),
        delimiter=",",
        quotechar='"',
        doublequote=True,
        escapechar=None,
        skipinitialspace=False,
        quoting=_csv_oracle.QUOTE_MINIMAL,
        strict=True,
    )
)
assert oracle == expected
for cut in range(len(document) + 1):
    assert decode_strict_csv_chunks((document[:cut], document[cut:])) == oracle
assert decode_strict_csv_chunks(tuple(document)) == oracle


def every_contiguous_chunking(text: str) -> Iterable[tuple[str, ...]]:
    if not text:
        yield ()
        return
    for boundary_mask in range(1 << (len(text) - 1)):
        chunks_under_test: list[str] = []
        start = 0
        for boundary in range(len(text) - 1):
            if boundary_mask & (1 << boundary):
                chunks_under_test.append(text[start : boundary + 1])
                start = boundary + 1
        chunks_under_test.append(text[start:])
        yield tuple(chunks_under_test)


short_document = '"a\r\n""b",c'
short_oracle = (('a\r\n"b', "c"),)
for short_chunks in every_contiguous_chunking(short_document):
    assert decode_strict_csv_chunks(short_chunks) == short_oracle

assert decode_strict_csv_chunks(()) == ()
assert decode_strict_csv_chunks(("\n\r\n\r",)) == ((), (), ())
assert decode_strict_csv_chunks((",\n",)) == (("", ""),)


def rejects(chunks: Iterable[str]) -> bool:
    try:
        decode_strict_csv_chunks(chunks)
    except (TypeError, ValueError, RuntimeError):
        return True
    return False


assert rejects(('"unterminated',))
assert rejects(("\ud800",))
assert rejects((b"not text",))  # type: ignore[arg-type]
assert rejects(("x" * (_MAX_CSV_PHYSICAL_LINE_CHARACTERS + 1),))
```

## Trade-offs and Limitations

The adapter retains one physical line at a time, but `csv.reader` may retain a
quoted record across several lines and this function deliberately materializes
all decoded rows. The aggregate input, line, row, column, cell, and field caps
bound that work. Chunk lengths are measured in Unicode code points, not bytes;
the input has already crossed the encoding boundary.

`strict=True` is the standard library's documented strict mode, not a new CSV
grammar. In particular, the Python parser accepts a quote inside an unquoted
field as literal text. The implementation neither mutates the process-global
`csv.field_size_limit` nor controls another thread that changes it; an
externally lowered limit is rejected at entry, while a concurrent later change
can still affect parsing.

The function does not decode bytes, strip a BOM, sniff dialects, infer types,
validate a schema, neutralize spreadsheet formulas, or recover after an invalid
record. Preserving text is not the same as declaring it safe for a spreadsheet
or command interpreter.

## Related Snippets

<!-- catalog:related:start -->
- [Decode Bounded JSON Lines Across Arbitrary Text Chunks](decode-bounded-json-lines-across-arbitrary-text-chunks.md)
- [Limit Text Lines Across Arbitrary Chunks](limit-text-lines-across-arbitrary-chunks.md)
- [Validate Parsed CSV Rows with Bounded Structured Problems](validate-parsed-csv-rows-with-bounded-structured-problems.md)
<!-- catalog:related:end -->
