---
title: "Find Newly Pending Tasks at an asyncio Test Boundary"
snippet_type: testing-technique
use_cases:
  - concurrency-control
  - lifecycle-management
  - testing
tested_python:
  - "3.14"
dependencies: []
related:
  - ../observability-operations/project-a-bounded-asyncio-call-graph-without-retaining-live-frames.md
  - ../concurrency-lifecycle/drain-bounded-asyncio-queue-work-with-graceful-shutdown.md
---

# Find Newly Pending Tasks at an asyncio Test Boundary

## Idea and Problem

Report tasks that appeared during an async test operation and are still unfinished at its checkpoint.

Take an `asyncio.all_tasks()` snapshot just before the operation and another
after it. Subtract the earlier task objects to ignore fixtures that were
already running. Use names for reporting only, since unrelated tasks can have
the same name. Check before `asyncio.run()` shuts down the event loop and
cancels remaining tasks.

## When to Use

Use this in a controlled event loop when the operation is expected to finish
its own background work before returning. Take both snapshots from the same
loop. Give test-created tasks descriptive names and choose a checkpoint where
the operation's documented cleanup has had a chance to finish.

## Implementation

```python
import asyncio


def newly_pending_names(before: set[asyncio.Task[object]]) -> tuple[str, ...]:
    newly_pending = asyncio.all_tasks() - before
    return tuple(sorted(task.get_name() for task in newly_pending))
```

## Example

```python
async def inspect_operation() -> tuple[tuple[str, ...], tuple[str, ...]]:
    release = asyncio.Event()
    baseline_worker = asyncio.create_task(release.wait(), name="fixture worker")
    added_worker = None
    try:
        before = asyncio.all_tasks()
        completed = asyncio.create_task(asyncio.sleep(0), name="one-off step")
        await completed
        added_worker = asyncio.create_task(release.wait(), name="fixture worker")
        at_checkpoint = newly_pending_names(before)
    finally:
        release.set()
        await baseline_worker
        if added_worker is not None:
            await added_worker

    return at_checkpoint, newly_pending_names(before)


assert asyncio.run(inspect_operation()) == (("fixture worker",), ())
```

## Trade-offs and Limitations

The snapshots see only unfinished tasks in the running loop. They cannot
show tasks that started and finished between snapshots, tasks in another
loop, or work scheduled as a callback rather than a task. A pending task at
the checkpoint is an observation, not proof it would stay pending forever.

An unrelated task started in the same interval would also appear. Keep the
test isolated and retain the baseline only until comparison. The example
releases and awaits the workers it created. The diagnostic itself does not
cancel or join tasks, so ownership stays with the code that started them.

## Related Snippets

<!-- catalog:related:start -->
- [Project a Bounded asyncio Call Graph Without Retaining Live Frames](../observability-operations/project-a-bounded-asyncio-call-graph-without-retaining-live-frames.md)
- [Drain Bounded asyncio Queue Work with Graceful Shutdown](../concurrency-lifecycle/drain-bounded-asyncio-queue-work-with-graceful-shutdown.md)
<!-- catalog:related:end -->
