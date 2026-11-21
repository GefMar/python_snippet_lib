---
title: "Profile One Synchronous Workload and Read Call Counts"
snippet_type: testing-technique
use_cases:
  - observability
  - performance-optimization
  - testing
tested_python:
  - "3.14"
dependencies: []
related:
  - capture-executed-lines-for-one-code-object-with-sys-monitoring.md
  - ../observability-operations/measure-wall-and-thread-cpu-time-for-one-synchronous-operation.md
---

# Profile One Synchronous Workload and Read Call Counts

## Idea and Problem

Measure how often a function ran during one diagnostic workload without adding counters to the function.

Run the workload through `cProfile.Profile.runcall()`, then read call counts
from the structured `pstats` report. A recursive function can have more total
calls than primitive calls, which are entries not induced by recursion.

## When to Use

Use this for a small, synchronous, single-threaded CPython diagnostic with
no other profiler active. The helper below looks up plain Python functions
whose code names are unique in the report. It checks the retained row's file
and line before accepting its counts.

## Implementation

```python
import cProfile
from collections.abc import Callable
from pstats import Stats, StatsProfile
from types import FunctionType


def profile_workload[**P, Result](
    workload: Callable[P, Result], /, *args: P.args, **kwargs: P.kwargs
) -> tuple[Result, StatsProfile]:
    profiler = cProfile.Profile()
    result = profiler.runcall(workload, *args, **kwargs)
    return result, Stats(profiler).get_stats_profile()


def call_counts(report: StatsProfile, function: FunctionType) -> tuple[int, int]:
    code = function.__code__
    row = report.func_profiles.get(code.co_name)
    if row is None or (row.file_name, row.line_number) != (
        code.co_filename,
        code.co_firstlineno,
    ):
        raise LookupError(f"no matching profile row for {code.co_name!r}")

    total, _, primitive = row.ncalls.partition("/")
    return int(total), int(primitive or total)
```

## Example

```python
type Node = tuple[str, tuple[Node, ...]]


def count_nodes(node: Node) -> int:
    return 1 + sum(count_nodes(child) for child in node[1])


def count_forest(roots: tuple[Node, ...]) -> int:
    return sum(count_nodes(root) for root in roots)


forest = (
    ("oak", (("east", ()), ("west", ()))),
    ("pine", (("north", ()),)),
)
result, report = profile_workload(count_forest, forest)

assert (result, call_counts(report, count_nodes), call_counts(report, count_forest)) == (
    5,
    (5, 2),
    (1, 1),
)
```

## Trade-offs and Limitations

The structured report is keyed by function name, so a same-named row can
overwrite another. The file and line check rejects a mismatched row, but it
cannot recover omitted rows. For such workloads, inspect the full
`pstats.Stats` report with file names and line numbers intact.

Profiling changes timings, and the counts depend on the input and execution
path. Use the results to investigate call patterns, not as a benchmark.

## Related Snippets

<!-- catalog:related:start -->
- [Capture Executed Lines for One Code Object with sys.monitoring](capture-executed-lines-for-one-code-object-with-sys-monitoring.md)
- [Measure Wall and Thread CPU Time for One Synchronous Operation](../observability-operations/measure-wall-and-thread-cpu-time-for-one-synchronous-operation.md)
<!-- catalog:related:end -->
