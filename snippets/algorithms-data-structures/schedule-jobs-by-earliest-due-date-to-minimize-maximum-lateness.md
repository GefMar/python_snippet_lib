---
title: "Schedule Jobs by Earliest Due Date to Minimize Maximum Lateness"
snippet_type: algorithm
use_cases:
  - automation
  - data-transformation
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - select-a-maximum-cardinality-set-of-non-overlapping-half-open-integer-intervals.md
  - generate-smooth-weighted-round-robin-slots-for-fixed-consumers.md
---

# Schedule Jobs by Earliest Due Date to Minimize Maximum Lateness

## Idea and Problem

Order jobs on one worker to minimize the largest signed difference between a completion time and its due date.

Sort jobs by due date and run them consecutively. If a job with a later due
date precedes an adjacent job with an earlier one, swapping the two cannot
increase their maximum lateness. The earlier-due job finishes no later than
before. The other job inherits that old finish and has the later due date.
Removing these inversions shows why due-date order achieves the optimum.

## When to Use

Use this when every job is ready at time zero, the worker runs one job at a
time and every job must run to completion. Durations are non-negative integers
and due dates use the same integer clock. Equal due dates retain input order.
The result includes the chosen order and its maximum signed lateness.

## Implementation

```python
from collections.abc import Iterable
from dataclasses import dataclass


@dataclass(frozen=True, slots=True)
class Job:
    name: str
    duration: int
    due: int

    def __post_init__(self) -> None:
        if type(self.name) is not str:
            raise TypeError("job name must be a string")
        if type(self.duration) is not int or type(self.due) is not int:
            raise TypeError("duration and due date must be integers")
        if self.duration < 0:
            raise ValueError("duration must be non-negative")


def schedule_by_due_date(jobs: Iterable[Job]) -> tuple[tuple[Job, ...], int | None]:
    pending = tuple(jobs)
    if any(type(job) is not Job for job in pending):
        raise TypeError("jobs must contain Job records")
    order = tuple(sorted(pending, key=lambda job: job.due))

    finish = 0
    maximum: int | None = None
    for job in order:
        finish += job.duration
        lateness = finish - job.due
        maximum = lateness if maximum is None else max(maximum, lateness)
    return order, maximum
```

## Example

```python
calibrate = Job("calibrate", 3, 8)
count = Job("count", 2, 3)
pack = Job("pack", 2, 3)
check = Job("check", 2, 5)
send = Job("send", 1, 10)

assert (
    schedule_by_due_date((calibrate, count, pack)),
    schedule_by_due_date((send, check)),
    schedule_by_due_date(()),
) == (((count, pack, calibrate), 1), ((check, send), -3), ((), None))
```

## Trade-offs and Limitations

Sorting dominates the work and requires holding the job list. The function
computes signed lateness, so the maximum is negative when all jobs finish
early. Tardiness clips each lateness at zero. An empty schedule returns
`None` for the maximum because it has no completion times.

Release times, dependencies, setup costs and multiple workers require
additional scheduling rules. This objective also differs from minimizing
the number of late jobs or their total lateness penalties.

## Related Snippets

<!-- catalog:related:start -->
- [Select a Maximum-Cardinality Set of Non-Overlapping Half-Open Integer Intervals](select-a-maximum-cardinality-set-of-non-overlapping-half-open-integer-intervals.md)
- [Generate Smooth Weighted Round-Robin Slots for Fixed Consumers](generate-smooth-weighted-round-robin-slots-for-fixed-consumers.md)
<!-- catalog:related:end -->
