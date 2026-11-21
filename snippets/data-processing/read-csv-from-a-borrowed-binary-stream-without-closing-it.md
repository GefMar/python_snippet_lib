---
title: "Read CSV from a Borrowed Binary Stream Without Closing It"
snippet_type: recipe
use_cases:
  - parsing
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - decode-strict-csv-records-across-arbitrary-text-chunks.md
  - try-an-optional-text-header-without-consuming-a-mismatch.md
---

# Read CSV from a Borrowed Binary Stream Without Closing It

## Idea and Problem

Decode and parse a CSV document from an open binary stream while leaving its caller responsible for closing it.

`csv.reader()` expects text. A `TextIOWrapper` can decode the bytes, but
closing the wrapper also closes its buffer. Read the rows inside the helper,
then detach the buffer in `finally` so the same caller-owned stream is still
available after a successful read or a decoding error.

## When to Use

Use this for a small UTF-8 CSV document in a healthy, readable buffered
stream that the caller already owns. The helper needs exclusive access while
it reads. Use `newline=""` so the CSV parser handles line endings, including
newlines inside quoted fields.

## Implementation

```python
import csv
from io import BufferedIOBase, TextIOWrapper


def read_borrowed_csv(source: BufferedIOBase) -> list[list[str]]:
    text = TextIOWrapper(source, encoding="utf-8", newline="")
    try:
        return list(csv.reader(text))
    finally:
        text.detach()
```

## Example

```python
from io import BytesIO


document = b'id,description\r\n1,"line one\r\nline two"\r\n'
with BytesIO(document) as source:
    rows = read_borrowed_csv(source)
    source.seek(0)
    original = source.read()

malformed = b"id,description\n1,\xff\n"
with BytesIO(malformed) as bad_source:
    try:
        read_borrowed_csv(bad_source)
    except UnicodeDecodeError:
        pass
    else:
        raise AssertionError("invalid UTF-8 must raise")
    bad_source.seek(0)
    recovered = bad_source.read()

assert (rows, original, recovered) == (
    [["id", "description"], ["1", "line one\r\nline two"]],
    document,
    malformed,
)
```

## Trade-offs and Limitations

The function materializes the CSV rows and expects an input small enough to
fit in memory. Detaching leaves the text wrapper unusable, so a returned CSV
iterator could not keep reading. Use a reader with a longer explicit scope
for large files.

The binary stream is not rewound. Decoding may read ahead, including before
a decoding failure. If the caller needs to reread the same bytes, it must
seek on a seekable source. The ownership pattern assumes normal stream
operation during detachment. An error raised by detachment itself propagates.

## Related Snippets

<!-- catalog:related:start -->
- [Decode Strict CSV Records Across Arbitrary Text Chunks](decode-strict-csv-records-across-arbitrary-text-chunks.md)
- [Try an Optional Text Header Without Consuming a Mismatch](try-an-optional-text-header-without-consuming-a-mismatch.md)
<!-- catalog:related:end -->
