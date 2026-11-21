---
title: "Locate the Innermost Failing Expression as UTF-8 Byte Columns"
snippet_type: testing-technique
use_cases:
  - observability
  - testing
tested_python:
  - "3.14"
dependencies: []
related:
  - ../observability-operations/capture-a-bounded-pickle-friendly-exception-report.md
  - ../observability-operations/project-a-bounded-asyncio-call-graph-without-retaining-live-frames.md
---

# Locate the Innermost Failing Expression as UTF-8 Byte Columns

## Idea and Problem

Retrieve the source coordinates for the innermost runtime failure in a Python traceback.

Take the last frame summary with `extract_tb(..., limit=-1)`. Its columns
are UTF-8 byte offsets in CPython, so slice the encoded source line to
recover a single-line expression. Copy the coordinates when the diagnostic
only needs to keep the location.

## When to Use

Use this for exceptions from trusted code on CPython with debug ranges
available. Match the coordinates with the same unmodified source that
produced the code. Preserve the line's indentation while obtaining its
bytes, since the offsets include that indentation.

## Implementation

```python
from dataclasses import dataclass
from traceback import extract_tb
from types import TracebackType


@dataclass(frozen=True, slots=True)
class ExpressionSpan:
    first_line: int
    last_line: int
    first_column: int
    end_column: int


def innermost_expression(traceback: TracebackType | None) -> ExpressionSpan | None:
    if traceback is None:
        return None

    (frame,) = extract_tb(traceback, limit=-1)
    if (
        frame.lineno is None
        or frame.end_lineno is None
        or frame.colno is None
        or frame.end_colno is None
    ):
        return None
    return ExpressionSpan(frame.lineno, frame.end_lineno, frame.colno, frame.end_colno)
```

## Example

```python
from inspect import getsourcelines


def missing_entry() -> int:
    return len("café") + {"present": 1}["missing"]


try:
    missing_entry()
except KeyError as error:
    span = innermost_expression(error.__traceback__)
else:
    raise AssertionError("expected the missing-entry error")

assert span is not None and span.first_line == span.last_line
source, start_line = getsourcelines(missing_entry)
line = source[span.first_line - start_line]
expression = '{"present": 1}["missing"]'
selected = line.encode("utf-8")[span.first_column : span.end_column]

assert (selected, span.first_column > line.index(expression), innermost_expression(None)) == (
    expression.encode("utf-8"),
    True,
    None,
)
```

## Trade-offs and Limitations

Line numbers start at one. Columns start at zero and the end column is
exclusive. For a multiline expression, the columns apply to its first and
last lines respectively. A multiline renderer needs all intervening lines.

Missing traceback or debug ranges yield `None`, which a diagnostic can
handle with a line-only fallback. Source can be unavailable or changed by
the time the error is reported. The returned span describes the innermost
frame of the supplied traceback, while causes and grouped exceptions have
their own tracebacks.

## Related Snippets

<!-- catalog:related:start -->
- [Capture a Bounded Pickle-Friendly Exception Report](../observability-operations/capture-a-bounded-pickle-friendly-exception-report.md)
- [Project a Bounded asyncio Call Graph Without Retaining Live Frames](../observability-operations/project-a-bounded-asyncio-call-graph-without-retaining-live-frames.md)
<!-- catalog:related:end -->
