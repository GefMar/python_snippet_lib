---
title: "Round-Trip a Bounded Nullable String Table with CSV Quote-Not-Null"
snippet_type: recipe
use_cases:
  - interoperability
  - parsing
  - serialization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../data-processing/read-bounded-csv-text-into-pandas-under-an-explicit-schema.md
  - ../data-processing/normalize-optional-csv-columns-in-a-single-pass.md
  - ../data-processing/transpose-bounded-ragged-exact-string-rows-with-none-for-missing-cells.md
---

# Round-Trip a Bounded Nullable String Table with CSV Quote-Not-Null

## Idea and Problem

Encode and decode a bounded string table while preserving the difference between None and an empty string in canonical CSV text.

With `csv.QUOTE_NOTNULL`, a writer leaves `None` as an unquoted empty field and
quotes every other value. A reader using the same mode maps the unquoted empty
field back to `None`, while `""` remains an empty string. Re-encoding the parsed
table and comparing it byte for byte closes the profile around one delimiter,
escaping rule, quoting policy, and line ending.

## When to Use

Use this codec when two Python 3.14 endpoints exchange a small rectangular
table containing only exact strings and `None`, and the wire format must retain
nullable-versus-empty semantics without a separate sentinel string. The
producer and consumer must agree on this exact canonical profile.

This is not a general CSV importer. It does not discover encodings or dialects,
infer schema types, convert numbers, or accept alternate null spellings. CSV
quoting also does not neutralize spreadsheet formulas; apply a separate export
policy before data is opened in spreadsheet software.

## Implementation

```python
import csv
from io import StringIO

_MAX_ROWS = 256
_MAX_COLUMNS = 64
_MAX_CELLS = 16_384
_MAX_CELL_BYTES = 4_096
_MAX_PAYLOAD_BYTES = 512 * 1_024
_MAX_CSV_BYTES = 1_024 * 1_024


class NullableCsvError(ValueError):
    pass


def _validate_nullable_table(
    table: tuple[tuple[str | None, ...], ...],
) -> None:
    if type(table) is not tuple:
        raise TypeError("table must be an exact tuple")
    if len(table) > _MAX_ROWS:
        raise NullableCsvError("table exceeds the supported row count")
    if not table:
        return

    width: int | None = None
    cell_count = 0
    payload_bytes = 0
    for row in table:
        if type(row) is not tuple:
            raise TypeError("table rows must be exact tuples")
        if width is None:
            width = len(row)
            if not 1 <= width <= _MAX_COLUMNS:
                raise NullableCsvError("table width is outside the supported range")
        elif len(row) != width:
            raise NullableCsvError("table rows must have one fixed width")
        if width == 1 and row[0] is None:
            raise NullableCsvError(
                "a one-column row cannot represent None canonically"
            )

        cell_count += len(row)
        if cell_count > _MAX_CELLS:
            raise NullableCsvError("table exceeds the supported cell count")
        for cell in row:
            if cell is None:
                continue
            if type(cell) is not str:
                raise TypeError("table cells must be exact strings or None")
            if len(cell) > _MAX_CELL_BYTES:
                raise NullableCsvError("table cell exceeds the supported byte count")
            try:
                encoded_cell = cell.encode("utf-8")
            except UnicodeEncodeError as error:
                raise NullableCsvError(
                    "table cells must not contain Unicode surrogates"
                ) from error
            if len(encoded_cell) > _MAX_CELL_BYTES:
                raise NullableCsvError("table cell exceeds the supported byte count")
            payload_bytes += len(encoded_cell)
            if payload_bytes > _MAX_PAYLOAD_BYTES:
                raise NullableCsvError("table exceeds the supported payload size")


def encode_nullable_string_table(
    table: tuple[tuple[str | None, ...], ...],
) -> str:
    _validate_nullable_table(table)
    buffer = StringIO(newline="")
    try:
        writer = csv.writer(
            buffer,
            delimiter=",",
            quotechar='"',
            doublequote=True,
            escapechar=None,
            lineterminator="\r\n",
            quoting=csv.QUOTE_NOTNULL,
            skipinitialspace=False,
            strict=True,
        )
        writer.writerows(table)
    except csv.Error as error:
        raise NullableCsvError("table cannot be encoded as canonical CSV") from error

    encoded = buffer.getvalue()
    if len(encoded.encode("utf-8")) > _MAX_CSV_BYTES:
        raise NullableCsvError("encoded CSV exceeds the supported byte count")
    return encoded


def decode_nullable_string_table(
    text: str,
) -> tuple[tuple[str | None, ...], ...]:
    if type(text) is not str:
        raise TypeError("text must be an exact string")
    if len(text) > _MAX_CSV_BYTES:
        raise NullableCsvError("CSV text exceeds the supported byte count")
    try:
        encoded_text = text.encode("utf-8")
    except UnicodeEncodeError as error:
        raise NullableCsvError("CSV text must not contain Unicode surrogates") from error
    if len(encoded_text) > _MAX_CSV_BYTES:
        raise NullableCsvError("CSV text exceeds the supported byte count")
    if csv.field_size_limit() < _MAX_CELL_BYTES:
        raise RuntimeError("the process CSV field-size limit is too small")

    try:
        reader = csv.reader(
            StringIO(text, newline=""),
            delimiter=",",
            quotechar='"',
            doublequote=True,
            escapechar=None,
            quoting=csv.QUOTE_NOTNULL,
            skipinitialspace=False,
            strict=True,
        )
        table = tuple(tuple(row) for row in reader)
    except csv.Error as error:
        raise NullableCsvError("CSV text is malformed") from error

    _validate_nullable_table(table)
    if encode_nullable_string_table(table) != text:
        raise NullableCsvError("CSV text is not in canonical form")
    return table
```

## Example

```python
table = (
    (None, "", "plain"),
    ("comma,value", 'say "hi"', "line\r\nbreak"),
)

encoded = encode_nullable_string_table(table)

assert encoded == (
    ',"","plain"\r\n'
    '"comma,value","say ""hi""","line\r\nbreak"\r\n'
)
assert decode_nullable_string_table(encoded) == table

boundary_cell = '"' * 4_096
assert decode_nullable_string_table(
    encode_nullable_string_table(((boundary_cell,),))
) == ((boundary_cell,),)

try:
    decode_nullable_string_table("plain\r\n")
except NullableCsvError:
    pass
else:
    raise AssertionError("an unquoted nonempty field is not canonical")

for invalid_table in (
    ((None,),),
    (("one",), ("two", "extra")),
    (("x" * 4_097,),),
):
    try:
        encode_nullable_string_table(invalid_table)
    except NullableCsvError:
        pass
    else:
        raise AssertionError("invalid table must not be encoded")

assert decode_nullable_string_table("") == ()
```

## Trade-offs and Limitations

For `B` CSV bytes and `C` cells, encoding or decoding uses `O(B + C)` time
and state. Decoding retains both the input string and immutable result, then
performs a second linear encoding pass to enforce canonical representation.

Python's CSV parser consults the process-global `csv.field_size_limit`. The
decoder requires that value to be at least 4,096 on entry and assumes no other
thread changes it during decoding; it does not mutate the setting itself. A
lower limit produces `RuntimeError`, while a concurrent change can still make
the parser fail with `NullableCsvError`.

The empty table encodes as empty text. Non-empty tables must be rectangular,
and a one-column row cannot contain `None`: the standard writer rejects its
ambiguous empty-record representation. Embedded commas, quotes, CRLF, and
empty strings are safe only because all non-null fields are quoted. The codec
returns text rather than performing file I/O, does not select an external
character encoding, and offers no formula injection defense or domain-specific
schema validation.

## Related Snippets

<!-- catalog:related:start -->
- [Read Bounded CSV Text into pandas Under an Explicit Schema](../data-processing/read-bounded-csv-text-into-pandas-under-an-explicit-schema.md)
- [Normalize Optional CSV Columns in a Single Pass](../data-processing/normalize-optional-csv-columns-in-a-single-pass.md)
- [Transpose Bounded Ragged Exact-String Rows with None for Missing Cells](../data-processing/transpose-bounded-ragged-exact-string-rows-with-none-for-missing-cells.md)
<!-- catalog:related:end -->
