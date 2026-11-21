---
title: "Select a CSV Delimiter from an Exact Header"
snippet_type: recipe
use_cases:
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - reject-ambiguous-dictreader-headers-and-ragged-csv-rows.md
  - validate-parsed-csv-rows-with-bounded-structured-problems.md
---

# Select a CSV Delimiter from an Exact Header

## Idea and Problem

Select a delimiter from known alternatives using an exact expected header rather than a punctuation heuristic.

Parse the first logical record under each alternative. Accept a delimiter only when it is the sole option that produces all expected field names in order.

## When to Use

Use this for bounded CSV samples from a known export family where the delimiter varies but the header schema does not. Supply the complete first logical record, including any quoted line breaks.

## Implementation

```python
from csv import Error, reader
from io import StringIO


def select_delimiter(sample: str, header: tuple[str, ...]) -> str:
    if not header or len(sample) > 8192:
        raise ValueError("expected a nonempty header and a bounded sample")

    matches = []
    for delimiter in (",", ";", "\t"):
        with StringIO(sample, newline="") as stream:
            try:
                first = next(reader(stream, delimiter=delimiter, strict=True), None)
            except Error:
                continue
        if first is not None and tuple(first) == header:
            matches.append(delimiter)

    if len(matches) != 1:
        raise ValueError("expected exactly one delimiter to match the header")
    return matches[0]
```

## Example

```python
selected = select_delimiter('"id;raw";value\r\n1;2\r\n', ("id;raw", "value"))
tabbed = select_delimiter("id\tvalue\n1\t2\n", ("id", "value"))
try:
    select_delimiter("id\n1\n", ("id",))
except ValueError:
    rejected_ambiguity = True
else:
    rejected_ambiguity = False

assert (selected, tabbed, rejected_ambiguity) == (";", "\t", True)
```

## Trade-offs and Limitations

This finds a delimiter only within the closed candidate list, using the standard quoting rules. It does not validate the data records, infer encodings, or choose a delimiter for a genuinely ambiguous single-column header. Apply the selected delimiter and validate the full file separately.

## Related Snippets

<!-- catalog:related:start -->
- [Reject Ambiguous DictReader Headers and Ragged CSV Rows](reject-ambiguous-dictreader-headers-and-ragged-csv-rows.md)
- [Validate Parsed CSV Rows with Bounded Structured Problems](validate-parsed-csv-rows-with-bounded-structured-problems.md)
<!-- catalog:related:end -->
