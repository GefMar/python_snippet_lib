---
title: "Reject Ambiguous DictReader Headers and Ragged CSV Rows"
snippet_type: recipe
use_cases:
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - validate-parsed-csv-rows-with-bounded-structured-problems.md
  - normalize-optional-csv-columns-in-a-single-pass.md
---

# Reject Ambiguous DictReader Headers and Ragged CSV Rows

## Idea and Problem

Read CSV into named rows while rejecting duplicate headers, missing fields, and surplus fields.

`DictReader` maps cells to the header names, so repeated names can overwrite
earlier cells. Check the header before consuming data records. With the
standard Excel dialect, a surplus list appears under the `None` key and a
missing cell is filled with `None`. A present empty cell remains an empty
string.

## When to Use

Use this for a standard comma-separated document with a required header and
an exact field count on each data record. Supply an iterable of text lines or
open a text file with `newline=""` so the CSV reader can process quoted line
breaks. Header names are compared literally.

## Implementation

```python
import csv
from collections.abc import Iterable, Iterator


def read_named_csv_rows(lines: Iterable[str]) -> Iterator[dict[str, str]]:
    reader = csv.DictReader(lines, dialect=csv.excel, restkey=None, restval=None, strict=True)
    header = reader.fieldnames
    if not header or any(not name for name in header) or len(set(header)) != len(header):
        raise ValueError("expected nonempty and unique CSV header names")

    for record_number, row in enumerate(reader, start=1):
        if None in row:
            raise ValueError(f"data record {record_number} has extra fields")
        if any(value is None for value in row.values()):
            raise ValueError(f"data record {record_number} is missing fields")
        yield row
```

## Example

```python
from io import StringIO

with StringIO('key,label\r\n7,\r\n\r\n8,"two\r\nlines"\r\n', newline="") as text:
    observed = list(read_named_csv_rows(text))

failures = []
for payload in ("key,key\n7,8\n", "key,label\n7\n", "key,label\n7,a,b\n"):
    with StringIO(payload, newline="") as text:
        try:
            list(read_named_csv_rows(text))
        except ValueError as error:
            failures.append(str(error))

assert (observed, failures) == (
    [{"key": "7", "label": ""}, {"key": "8", "label": "two\r\nlines"}],
    [
        "expected nonempty and unique CSV header names",
        "data record 1 is missing fields",
        "data record 1 has extra fields",
    ],
)
```

## Trade-offs and Limitations

`DictReader` skips blank data lines. The error number counts data records
yielded by the parser, so a quoted multiline field still belongs to one
record. An empty first line fails the header check. The shape rules apply to
the selected dialect, where parsed cell values are strings. Dialects that
decode some cells as `None` need a separate missing-value marker.

The reader yields valid records before reaching a later error. For an
all-or-nothing import, validate the full batch or write inside a transaction
that can roll back. CSV syntax errors detected by the parser propagate as
`csv.Error`; the row shape checks raise `ValueError`.

## Related Snippets

<!-- catalog:related:start -->
- [Validate Parsed CSV Rows with Bounded Structured Problems](validate-parsed-csv-rows-with-bounded-structured-problems.md)
- [Normalize Optional CSV Columns in a Single Pass](normalize-optional-csv-columns-in-a-single-pass.md)
<!-- catalog:related:end -->
