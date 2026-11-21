---
title: "Plan Unblocked Dependency Waves with graphlib TopologicalSorter"
snippet_type: recipe
use_cases:
  - automation
  - data-transformation
tested_python:
  - "3.14"
dependencies: []
related:
  - resolve-stable-ordering-constraints-with-topological-sort.md
  - ../testing-tooling/enumerate-every-topological-order-of-a-tiny-dag-for-schedule-tests.md
---

# Plan Unblocked Dependency Waves with graphlib TopologicalSorter

## Idea and Problem

Group dependency-driven tasks into waves and identify the tasks that remain blocked by cycles.

A wave contains every task whose prerequisites were placed in earlier
waves. Mark the entire wave done in `TopologicalSorter` to simulate planning
the next one. Even when `prepare()` reports a cycle, `get_ready()` can
still return work that is independent of the cycle.

## When to Use

Use this to preview stages in a small build or import plan. Map each task
name to a finite iterable of prerequisite names. A name that appears only
as a prerequisite becomes a task with no prerequisites of its own. Names
are sorted within each wave for predictable presentation.

## Implementation

```python
from collections.abc import Iterable, Mapping
from contextlib import suppress
from dataclasses import dataclass
from graphlib import CycleError, TopologicalSorter


@dataclass(frozen=True, slots=True)
class DependencyPlan:
    waves: tuple[tuple[str, ...], ...]
    blocked: tuple[str, ...]


def plan_dependency_waves(prerequisites: Mapping[str, Iterable[str]]) -> DependencyPlan:
    sorter: TopologicalSorter[str] = TopologicalSorter()
    remaining: set[str] = set()
    for task, required in prerequisites.items():
        if isinstance(required, (str, bytes)):
            raise TypeError("prerequisites must be an iterable of task names")
        names = tuple(required)
        sorter.add(task, *names)
        remaining.add(task)
        remaining.update(names)

    # Ready tasks are still available after prepare() reports a cycle.
    with suppress(CycleError):
        sorter.prepare()

    waves: list[tuple[str, ...]] = []
    while ready := sorter.get_ready():
        waves.append(tuple(sorted(ready)))
        remaining.difference_update(ready)
        sorter.done(*ready)

    return DependencyPlan(tuple(waves), tuple(sorted(remaining)))
```

## Example

```python
prerequisites = {
    "build": ("lint",),
    "publish": ("build", "docs"),
    "loop-a": ("loop-b",),
    "loop-b": ("loop-a",),
    "summary": ("docs", "loop-a"),
}

assert plan_dependency_waves(prerequisites) == DependencyPlan(
    waves=(("docs", "lint"), ("build",), ("publish",)),
    blocked=("loop-a", "loop-b", "summary"),
)
```

## Trade-offs and Limitations

Blocked names include cycle members and tasks that depend on them. For
example, `summary` is blocked because it needs `loop-a`. Use cycle analysis
when you need to identify the loops themselves.

The plan assumes every task in a wave finishes before the next wave starts.
It holds the dependency graph and all waves in memory. An executor needs to
account for worker capacity and actual task outcomes, calling `done()` only
for completed work under its own failure policy.

## Related Snippets

<!-- catalog:related:start -->
- [Resolve Stable Ordering Constraints with Topological Sort](resolve-stable-ordering-constraints-with-topological-sort.md)
- [Enumerate Every Topological Order of a Tiny DAG for Schedule Tests](../testing-tooling/enumerate-every-topological-order-of-a-tiny-dag-for-schedule-tests.md)
<!-- catalog:related:end -->
