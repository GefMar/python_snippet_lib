---
title: "Calculate Total Slack in an Unlimited-Capacity Task DAG"
snippet_type: algorithm
use_cases:
  - automation
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - find-a-deterministic-critical-path-in-a-bounded-vertex-weighted-dag.md
  - plan-unblocked-dependency-waves-with-graphlib-topologicalsorter.md
---

# Calculate Total Slack in an Unlimited-Capacity Task DAG

## Idea and Problem

Find how much each task can slip from its earliest finish without delaying the earliest possible completion of the whole project.

In topological order, compute the earliest finish of each task from its
duration and the latest of its predecessors' earliest finishes. The maximum
earliest finish is the project makespan. In reverse order, start with that
deadline for every task and constrain each predecessor to finish by its
successor's latest start. The difference between latest and earliest finish
is the task's total slack.

## When to Use

Use this for a finite dependency graph with non-negative integer durations.
Every prerequisite must finish before its dependent task can start, and any
number of ready tasks can run in parallel. All terminal tasks share the same
deadline, including those in disconnected branches. Names without a
predecessor entry can start at time zero.

## Implementation

```python
from collections.abc import Iterable, Mapping
from graphlib import TopologicalSorter


def project_total_slack(
    durations: Mapping[str, int], predecessors: Mapping[str, Iterable[str]]
) -> tuple[int, dict[str, int]]:
    """Return the whole-project makespan and total slack by task name."""
    effort = dict(durations)
    if any(type(name) is not str or type(duration) is not int for name, duration in effort.items()):
        raise TypeError("task names must be strings and durations must be integers")
    if any(duration < 0 for duration in effort.values()):
        raise ValueError("durations must be non-negative")
    if any(name not in effort for name in predecessors):
        raise ValueError("every dependent task needs a duration")

    graph = {name: tuple(predecessors.get(name, ())) for name in effort}
    if any(type(parent) is not str for parents in graph.values() for parent in parents):
        raise TypeError("predecessor names must be strings")
    if any(parent not in effort for parents in graph.values() for parent in parents):
        raise ValueError("every predecessor needs a duration")

    order = tuple(TopologicalSorter(graph).static_order())
    earliest_finish: dict[str, int] = {}
    for name in order:
        earliest_start = max((earliest_finish[parent] for parent in graph[name]), default=0)
        earliest_finish[name] = earliest_start + effort[name]
    makespan = max(earliest_finish.values(), default=0)

    latest_finish = dict.fromkeys(effort, makespan)
    for name in reversed(order):
        latest_start = latest_finish[name] - effort[name]
        for parent in graph[name]:
            latest_finish[parent] = min(latest_finish[parent], latest_start)

    return makespan, {name: latest_finish[name] - earliest_finish[name] for name in effort}
```

## Example

```python
durations = {"prepare": 2, "build": 5, "style": 3, "publish": 1, "labels": 4, "audit": 0}
predecessors = {
    "build": ("prepare",),
    "style": ("prepare",),
    "publish": ("build", "style"),
    "audit": ("publish",),
}

assert (project_total_slack(durations, predecessors), project_total_slack({}, {})) == (
    (8, {"prepare": 0, "build": 0, "style": 2, "publish": 0, "labels": 4, "audit": 0}),
    (0, {}),
)
```

## Trade-offs and Limitations

The forward and backward passes each visit every task and edge once. The
function stores the dependency graph and the two timing tables. A cycle
raises `graphlib.CycleError` during topological sorting.

Slack is available when considering one task's delay within a feasible
schedule. Several tasks can share the same slack, so an independent delay
of each task by its full slack may exceed the deadline. Limited workers,
release dates and per-task deadlines require a different scheduling model.

## Related Snippets

<!-- catalog:related:start -->
- [Find a Deterministic Critical Path in a Bounded Vertex-Weighted DAG](find-a-deterministic-critical-path-in-a-bounded-vertex-weighted-dag.md)
- [Plan Unblocked Dependency Waves with graphlib TopologicalSorter](plan-unblocked-dependency-waves-with-graphlib-topologicalsorter.md)
<!-- catalog:related:end -->
