---
title: "Capture Executed Lines for One Code Object with sys.monitoring"
snippet_type: testing-technique
use_cases:
  - observability
  - testing
tested_python:
  - "3.14"
dependencies: []
related:
  - audit-bounded-boolean-test-rows-for-unique-cause-mc-dc-coverage.md
  - ../observability-operations/report-a-bounded-positive-allocation-delta-from-filtered-tracemalloc-snapshots.md
---

# Capture Executed Lines for One Code Object with sys.monitoring

## Idea and Problem

Capture the source lines reached in one Python code object during a scoped diagnostic run.

Use a local `LINE` event mask to observe the code object without registering a
global event mask for this tool. Keep unique line numbers and release the tool
slot when the diagnostic completes or the surrounding code raises.

## When to Use

Use this in a controlled CPython test when line evidence from one known
function is enough to explain which source ran. Reserve an available tool ID
for your application, or use an isolated diagnostic process. Use a coverage
tool if you need reports across modules or evidence for branches that share
one source line.

## Implementation

```python
import sys
from collections.abc import Iterator
from contextlib import contextmanager
from types import CodeType


@contextmanager
def capture_executed_lines(code: CodeType, *, tool_id: int) -> Iterator[set[int]]:
    monitoring = sys.monitoring
    line_event = monitoring.events.LINE
    observed: set[int] = set()

    def record_line(_code: CodeType, line_number: int) -> None:
        observed.add(line_number)

    monitoring.use_tool_id(tool_id, "local-line-capture")
    try:
        monitoring.register_callback(tool_id, line_event, record_line)
        monitoring.set_local_events(tool_id, code, line_event)
        yield observed
    finally:
        monitoring.free_tool_id(tool_id)
```

## Example

```python
class StopDiagnostic(Exception):
    pass


def format_message(draft: bool) -> str:
    if draft:
        return "draft: report"
    return "report"


def other_message() -> str:
    return "unrelated"


tool_id = 3  # A slot used by this isolated example.
code = format_message.__code__
first_line = code.co_firstlineno
try:
    with capture_executed_lines(code, tool_id=tool_id) as lines:
        assert format_message(draft=True) == "draft: report"
        assert other_message() == "unrelated"
        raise StopDiagnostic
except StopDiagnostic:
    pass

assert sys.monitoring.get_tool(tool_id) is None
assert format_message(draft=False) == "report"
assert lines == {first_line + 1, first_line + 2}
```

## Trade-offs and Limitations

This cleanup requires Python 3.14 or later. On that version, `free_tool_id()`
also clears the registered callbacks and event masks. Reserving an occupied
ID raises before this scope acquires it. Other monitoring tools can still use
their own IDs.

A local mask applies to the code object throughout the interpreter. Recursive
calls, functions sharing that object and calls from other threads can add
observations. Use a controlled single-threaded run to attribute lines to one
scenario. Line events add overhead, and a set records neither order nor visit
counts. Line numbers refer to the source revision you ran.

## Related Snippets

<!-- catalog:related:start -->
- [Audit Bounded Boolean Test Rows for Unique-Cause MC/DC Coverage](audit-bounded-boolean-test-rows-for-unique-cause-mc-dc-coverage.md)
- [Report a Bounded Positive Allocation Delta from Filtered tracemalloc Snapshots](../observability-operations/report-a-bounded-positive-allocation-delta-from-filtered-tracemalloc-snapshots.md)
<!-- catalog:related:end -->
