---
title: "Project a Bounded asyncio Call Graph Without Retaining Live Frames"
snippet_type: recipe
use_cases:
  - concurrency-control
  - observability
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - capture-a-bounded-pickle-friendly-exception-report.md
  - ../concurrency-lifecycle/gather-async-results-with-bounded-concurrency.md
  - ../concurrency-lifecycle/race-a-preferred-async-read-against-bounded-alternatives.md
---

# Project a Bounded asyncio Call Graph Without Retaining Live Frames

## Idea and Problem

Project one owned asyncio task's live call graph into bounded immutable diagnostic records that retain no frames, locals, filenames, or object identities.

Python 3.14 can capture the stack of a task and the tasks or futures awaiting
it. Returning that graph directly would keep live frames and their local state
reachable. This projection assigns local task/future labels and copies only
bounded module and function names, line numbers, and local node-index edges,
then releases every captured graph reference before returning.

## When to Use

Use this helper from the running event loop while diagnosing one unfinished
task in a small, caller-owned task family. Choose a frame limit from 0 through
8 and a projected node limit from 1 through 32. A zero frame limit records only
task labels and `awaited_by` relationships.

Do not aim it at an unknown or hostile task topology. The standard-library
capture builds its graph before this helper can enforce the projected node
limit, so the limit constrains only the returned records, not capture time or
memory.

## Implementation

```python
import asyncio
from dataclasses import dataclass

_MAX_FRAME_LIMIT = 8
_MAX_NODE_LIMIT = 32
_MAX_MODULE_CHARACTERS = 128
_MAX_QUALIFIED_NAME_CHARACTERS = 256
_TRUNCATION_MARKER = "..."


class AsyncCallGraphProjectionError(ValueError):
    pass


class AsyncCallGraphNodeLimitError(RuntimeError):
    pass


@dataclass(frozen=True, slots=True)
class AsyncCallFrame:
    module: str
    qualified_name: str
    line_number: int


@dataclass(frozen=True, slots=True)
class AsyncCallNode:
    label: str
    frames: tuple[AsyncCallFrame, ...]


@dataclass(frozen=True, slots=True)
class AwaitedByEdge:
    awaited_index: int
    waiter_index: int


@dataclass(frozen=True, slots=True)
class AsyncCallGraphProjection:
    nodes: tuple[AsyncCallNode, ...]
    awaited_by: tuple[AwaitedByEdge, ...]


def _bounded_call_graph_integer(
    value: int,
    *,
    name: str,
    minimum: int,
    maximum: int,
) -> int:
    if type(value) is not int:
        raise TypeError(f"{name} must be an exact integer")
    if not minimum <= value <= maximum:
        raise AsyncCallGraphProjectionError(f"{name} is outside the supported range")
    return value


def _truncate_call_graph_text(value: str, maximum: int) -> str:
    if len(value) <= maximum:
        return value
    retained = maximum - len(_TRUNCATION_MARKER)
    return value[:retained] + _TRUNCATION_MARKER


def _project_call_graph_frame(frame: object) -> AsyncCallFrame | None:
    if frame is None:
        return None
    code = frame.f_code
    module = frame.f_globals.get("__name__", "<unknown>")
    if type(module) is not str:
        module = "<unknown>"
    return AsyncCallFrame(
        module=_truncate_call_graph_text(module, _MAX_MODULE_CHARACTERS),
        qualified_name=_truncate_call_graph_text(
            code.co_qualname,
            _MAX_QUALIFIED_NAME_CHARACTERS,
        ),
        line_number=frame.f_lineno,
    )


def _project_call_graph_label(
    future: asyncio.Future[object],
    node_index: int,
) -> str:
    kind = "task" if isinstance(future, asyncio.Task) else "future"
    return f"{kind}:{node_index}"


def project_asyncio_call_graph(
    task: asyncio.Task[object],
    *,
    frame_limit: int = 4,
    max_nodes: int = 16,
) -> AsyncCallGraphProjection:
    frame_limit = _bounded_call_graph_integer(
        frame_limit,
        name="frame_limit",
        minimum=0,
        maximum=_MAX_FRAME_LIMIT,
    )
    max_nodes = _bounded_call_graph_integer(
        max_nodes,
        name="max_nodes",
        minimum=1,
        maximum=_MAX_NODE_LIMIT,
    )
    if not isinstance(task, asyncio.Task):
        raise TypeError("task must be an asyncio Task")
    loop = asyncio.get_running_loop()
    if task.get_loop() is not loop:
        raise AsyncCallGraphProjectionError("task must belong to the current event loop")
    if task.done():
        raise AsyncCallGraphProjectionError("task must be unfinished")

    graph = asyncio.capture_call_graph(task, limit=frame_limit)
    if graph is None:
        raise AsyncCallGraphProjectionError("the task call graph is unavailable")

    pending = [(graph, None)]
    node_indexes: dict[asyncio.Future[object], int] = {}
    nodes: list[AsyncCallNode] = []
    edges: list[AwaitedByEdge] = []
    edge_pairs: set[tuple[int, int]] = set()
    current_graph = None
    future = None
    try:
        while pending:
            current_graph, awaited_index = pending.pop()
            future = current_graph.future
            node_index = node_indexes.get(future)
            if node_index is None:
                if len(nodes) >= max_nodes:
                    raise AsyncCallGraphNodeLimitError("projected node limit exceeded")
                node_index = len(nodes)
                node_indexes[future] = node_index
                projected_frames = tuple(
                    projected
                    for entry in current_graph.call_stack
                    if (projected := _project_call_graph_frame(entry.frame)) is not None
                )
                nodes.append(
                    AsyncCallNode(
                        label=_project_call_graph_label(future, node_index),
                        frames=projected_frames,
                    ),
                )
                pending.extend(
                    (waiter, node_index) for waiter in reversed(current_graph.awaited_by)
                )

            if awaited_index is not None:
                edge_pair = (awaited_index, node_index)
                if edge_pair not in edge_pairs:
                    edge_pairs.add(edge_pair)
                    edges.append(AwaitedByEdge(*edge_pair))

        projection = AsyncCallGraphProjection(tuple(nodes), tuple(edges))
    finally:
        pending.clear()
        node_indexes.clear()
        edge_pairs.clear()
        graph = None
        current_graph = None
        future = None
        task = None

    return projection
```

## Example

```python
async def run_call_graph_example() -> AsyncCallGraphProjection:
    leaf_started = asyncio.Event()
    waiter_started = asyncio.Event()
    release_leaf = asyncio.Event()

    async def leaf() -> str:
        leaf_started.set()
        await release_leaf.wait()
        return "done"

    async def waiter(leaf_task: asyncio.Task[str]) -> str:
        waiter_started.set()
        return await leaf_task

    leaf_task = asyncio.create_task(leaf(), name="leaf-worker")
    await leaf_started.wait()
    waiter_task = asyncio.create_task(waiter(leaf_task), name="waiter-worker")
    await waiter_started.wait()
    try:
        projection = project_asyncio_call_graph(
            leaf_task,
            frame_limit=4,
            max_nodes=4,
        )
    finally:
        release_leaf.set()
        await waiter_task
    return projection


projection = asyncio.run(run_call_graph_example())
labels = tuple(node.label for node in projection.nodes)
qualified_names = {
    frame.qualified_name
    for node in projection.nodes
    for frame in node.frames
}

assert (
    labels,
    projection.awaited_by,
    any(name.endswith(".leaf") for name in qualified_names),
    any(name.endswith(".waiter") for name in qualified_names),
) == (
    ("task:0", "task:1"),
    (AwaitedByEdge(awaited_index=0, waiter_index=1),),
    True,
    True,
)
```

## Trade-offs and Limitations

The frame limit applies separately to each call stack; it does not limit the
number of futures that `capture_call_graph()` visits. The node limit is checked
only while projecting an already captured graph. Capture can therefore consume
more time and memory, or fail recursively, before `AsyncCallGraphNodeLimitError`
can be raised. Restrict the helper to a known bounded task family.

The snapshot is neither atomic nor consistent with later task state. It does
not cancel, pause, secure, or otherwise control any task, and it offers no
hostile-process guarantee. Module, qualified function and line information can
still reveal sensitive application structure; redact or omit them before wider
publication. Local task/future labels disclose only the node kind and index,
not arbitrary `Task.get_name()` content. Truncation bounds stored symbol text
but can make two symbols indistinguishable. Node indexes and edge order are
local to one capture and must not be treated as persistent identities.

## Related Snippets

<!-- catalog:related:start -->
- [Capture a Bounded Pickle-Friendly Exception Report](capture-a-bounded-pickle-friendly-exception-report.md)
- [Gather Async Results with Bounded Concurrency](../concurrency-lifecycle/gather-async-results-with-bounded-concurrency.md)
- [Race a Preferred Async Read Against Bounded Alternatives](../concurrency-lifecycle/race-a-preferred-async-read-against-bounded-alternatives.md)
<!-- catalog:related:end -->
