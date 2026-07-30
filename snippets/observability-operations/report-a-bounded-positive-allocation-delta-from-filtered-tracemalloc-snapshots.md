---
title: "Report a Bounded Positive Allocation Delta from Filtered tracemalloc Snapshots"
snippet_type: recipe
use_cases:
  - observability
  - performance-optimization
  - testing
tested_python:
  - "3.14"
dependencies: []
related:
  - measure-and-freeze-elapsed-time-in-a-context.md
  - compute-a-process-cpu-rate-from-two-linux-procfs-samples.md
  - capture-a-bounded-pickle-friendly-exception-report.md
---

# Report a Bounded Positive Allocation Delta from Filtered tracemalloc Snapshots

## Idea and Problem

Compare two one-frame allocation snapshots, keep positive byte deltas only for explicitly labelled source lines, and return a caller-selected number of immutable records.

`tracemalloc.Filter` accepts a glob-style filename pattern, so an exact callback
filename must have `*`, `?`, and bracket syntax escaped before filtering. The
callback result is retained until the second snapshot so allocations reachable
from that result can remain visible. Sorting by measured size, block delta,
source line, and label makes ties deterministic without publishing a filename.

## When to Use

Use this diagnostic around one trusted, synchronous, argument-free Python
function when the interpreter is otherwise quiescent. Label from 1 through 16
exact source lines that represent allocation sites, and choose a `top_k` no
larger than the number of labels. The callback should return the objects whose
retained allocations matter; its result is released after the second snapshot.

Run the helper in a dedicated diagnostic process when unrelated threads,
callbacks, or existing tracing users may be active. `tracemalloc` state is
process-global, and this recipe requires tracing to be inactive on entry.

## Implementation

```python
import glob
import inspect
import tracemalloc
from dataclasses import dataclass
from types import FunctionType

_MAX_LABELLED_LINES = 16
_MAX_LABEL_CHARACTERS = 64
_MAX_SOURCE_LINE = (1 << 31) - 1


@dataclass(frozen=True, slots=True)
class LabelledSourceLine:
    label: str
    line_number: int


@dataclass(frozen=True, slots=True)
class PositiveAllocationDelta:
    label: str
    line_number: int
    size_delta_bytes: int
    block_count_delta: int


def _validate_labelled_source_lines(
    labelled_lines: tuple[LabelledSourceLine, ...],
) -> dict[int, str]:
    if type(labelled_lines) is not tuple:
        raise TypeError("labelled_lines must be an exact tuple")
    if not 1 <= len(labelled_lines) <= _MAX_LABELLED_LINES:
        raise ValueError("labelled_lines must contain 1 through 16 records")

    labels: set[str] = set()
    lines: dict[int, str] = {}
    for labelled_line in labelled_lines:
        if type(labelled_line) is not LabelledSourceLine:
            raise TypeError("labelled_lines must contain exact records")
        label = labelled_line.label
        line_number = labelled_line.line_number
        if (
            type(label) is not str
            or not 1 <= len(label) <= _MAX_LABEL_CHARACTERS
            or not label.isascii()
            or not label.isprintable()
            or label.strip() != label
        ):
            raise ValueError("labels must be bounded printable ASCII without padding")
        if type(line_number) is not int:
            raise TypeError("line numbers must be exact integers")
        if not 1 <= line_number <= _MAX_SOURCE_LINE:
            raise ValueError("a source line is outside the supported range")
        if label in labels or line_number in lines:
            raise ValueError("labels and source lines must both be unique")
        labels.add(label)
        lines[line_number] = label
    return lines


def report_positive_allocation_delta(
    callback: FunctionType,
    labelled_lines: tuple[LabelledSourceLine, ...],
    *,
    top_k: int,
) -> tuple[PositiveAllocationDelta, ...]:
    """Run one trusted callback and report selected positive line deltas."""
    if type(callback) is not FunctionType:
        raise TypeError("callback must be an exact Python function")
    if (
        inspect.iscoroutinefunction(callback)
        or inspect.isgeneratorfunction(callback)
        or inspect.isasyncgenfunction(callback)
    ):
        raise TypeError("callback must be a synchronous non-generator function")

    line_labels = _validate_labelled_source_lines(labelled_lines)
    if type(top_k) is not int:
        raise TypeError("top_k must be an exact integer")
    if not 1 <= top_k <= len(labelled_lines):
        raise ValueError("top_k must fit the labelled-line count")
    if tracemalloc.is_tracing():
        raise RuntimeError("tracemalloc must be inactive on entry")

    filename_pattern = glob.escape(callback.__code__.co_filename)
    filename_filter = (tracemalloc.Filter(True, filename_pattern),)
    _callback_result: object | None = None
    before: tracemalloc.Snapshot | None = None
    after: tracemalloc.Snapshot | None = None
    report: tuple[PositiveAllocationDelta, ...]

    tracemalloc.start(1)
    try:
        before = tracemalloc.take_snapshot()
        _callback_result = callback()
        after = tracemalloc.take_snapshot()

        filtered_before = before.filter_traces(filename_filter)
        filtered_after = after.filter_traces(filename_filter)
        positive_deltas: list[PositiveAllocationDelta] = []
        for statistic in filtered_after.compare_to(filtered_before, "lineno"):
            line_number = statistic.traceback[0].lineno
            label = line_labels.get(line_number)
            if label is None or statistic.size_diff <= 0:
                continue
            positive_deltas.append(
                PositiveAllocationDelta(
                    label=label,
                    line_number=line_number,
                    size_delta_bytes=statistic.size_diff,
                    block_count_delta=statistic.count_diff,
                ),
            )

        positive_deltas.sort(
            key=lambda delta: (
                -delta.size_delta_bytes,
                -delta.block_count_delta,
                delta.line_number,
                delta.label,
            ),
        )
        report = tuple(positive_deltas[:top_k])
    finally:
        _callback_result = None
        before = None
        after = None
        tracemalloc.stop()

    return report
```

## Example

```python


def allocate_three_buffers() -> tuple[bytearray, bytearray, bytearray]:
    largest = bytearray(32_768)
    middle = bytearray(16_384)
    smallest = bytearray(8_192)
    return largest, middle, smallest


first_line = allocate_three_buffers.__code__.co_firstlineno + 1
selected_lines = (
    LabelledSourceLine("largest-buffer", first_line),
    LabelledSourceLine("middle-buffer", first_line + 1),
    LabelledSourceLine("smallest-buffer", first_line + 2),
)

assert not tracemalloc.is_tracing()
report = report_positive_allocation_delta(
    allocate_three_buffers,
    selected_lines,
    top_k=2,
)


def fail_during_callback() -> None:
    raise RuntimeError("synthetic callback failure")


failure_line = fail_during_callback.__code__.co_firstlineno + 1
try:
    report_positive_allocation_delta(
        fail_during_callback,
        (LabelledSourceLine("failure", failure_line),),
        top_k=1,
    )
except RuntimeError as error:
    failure_propagated = str(error) == "synthetic callback failure"
else:
    failure_propagated = False

assert (
    len(report) == 2
    and all(delta.size_delta_bytes > 0 for delta in report)
    and tuple(delta.label for delta in report) == ("largest-buffer", "middle-buffer")
    and tuple(delta.line_number for delta in report) == (first_line, first_line + 1)
    and report[0].size_delta_bytes > report[1].size_delta_bytes
    and failure_propagated
    and not tracemalloc.is_tracing()
)
```

## Trade-offs and Limitations

Only the returned tuple is bounded. `tracemalloc.start(1)` observes Python
allocations process-wide, and both complete snapshots are created before the
filename and line filters are applied. Callback work, trace count, snapshot
memory, snapshot comparison, and elapsed time are not prebounded by `top_k` or
the 16 labels. The callback result also remains alive through the second full
snapshot and can itself be arbitrarily large.

The measurement is a diagnostic net delta, not an exact allocation ledger.
Allocator behavior and unrelated activity can change byte and block deltas;
native allocations not traced by Python are absent. A positive delta does not
prove a leak, and the helper neither enforces a memory limit nor reports freed
or unchanged labelled lines. One-frame tracing attributes each trace only to
its most recent Python source line.

Filtering uses the callback code object's escaped filename, while the returned
records intentionally expose only caller-supplied labels and line numbers.
Labels are not verified against source text, and different executions can
produce different rankings. Use one synchronous invocation in a quiescent
interpreter with no competing tracing owner. The `finally` block stops tracing
whether the callback, snapshot processing, or comparison succeeds or raises.

## Related Snippets

<!-- catalog:related:start -->
- [Measure and Freeze Elapsed Time in a Context](measure-and-freeze-elapsed-time-in-a-context.md)
- [Compute a Process CPU Rate from Two Linux procfs Samples](compute-a-process-cpu-rate-from-two-linux-procfs-samples.md)
- [Capture a Bounded Pickle-Friendly Exception Report](capture-a-bounded-pickle-friendly-exception-report.md)
<!-- catalog:related:end -->
