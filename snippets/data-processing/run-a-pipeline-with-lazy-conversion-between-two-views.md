---
title: "Run a Pipeline with Lazy Conversion Between Two Views"
snippet_type: pattern
use_cases:
  - data-transformation
  - interoperability
tested_python:
  - "3.14"
dependencies: []
related:
  - ../python-language/dispatch-named-strategies-with-an-explicit-function-mapping.md
  - ../python-language/make-a-defensive-copy-at-a-mutable-input-boundary.md
  - ../security-privacy/separate-executable-and-redacted-views-of-a-command-argument-vector.md
---

# Run a Pipeline with Lazy Conversion Between Two Views

## Idea and Problem

Execute an ordered transformation pipeline while converting only when the next step requires the other of two representations.

Each step declares the view it accepts and returns. The runner retains one
authoritative value and one explicit view tag instead of caching two copies,
so a step result of `None` is still a valid current payload. Consecutive steps
on the same view require no conversion, and the requested output view causes
at most one final conversion.

## When to Use

Use this pattern for a short, trusted, in-memory pipeline whose value has two
well-defined representations and whose transformations naturally operate on
one representation or the other. Provide conversion callbacks with documented
round-trip, mutation, and failure behavior.

Prefer one canonical representation when conversion cost is negligible. Use
a graph planner when more than two views exist or conversions have different
costs. Add an explicit transaction boundary when partial callback side effects
cannot be tolerated.

## Implementation

```python
from collections.abc import Callable
from dataclasses import dataclass
from enum import Enum
from typing import Any, Generic, TypeVar


LeftT = TypeVar("LeftT")
RightT = TypeVar("RightT")
_MAX_PIPELINE_STEPS = 128


class View(Enum):
    LEFT = "left"
    RIGHT = "right"


@dataclass(frozen=True, slots=True)
class ViewValue(Generic[LeftT, RightT]):
    view: View
    value: LeftT | RightT

    def __post_init__(self) -> None:
        if type(self.view) is not View:
            raise TypeError("view must be a View")


@dataclass(frozen=True, slots=True)
class LeftStep(Generic[LeftT]):
    transform: Callable[[LeftT], LeftT]

    def __post_init__(self) -> None:
        if not callable(self.transform):
            raise TypeError("a left transform must be callable")


@dataclass(frozen=True, slots=True)
class RightStep(Generic[RightT]):
    transform: Callable[[RightT], RightT]

    def __post_init__(self) -> None:
        if not callable(self.transform):
            raise TypeError("a right transform must be callable")


def run_lazy_view_pipeline(
    initial: ViewValue[LeftT, RightT],
    *,
    steps: tuple[LeftStep[LeftT] | RightStep[RightT], ...],
    to_left: Callable[[RightT], LeftT],
    to_right: Callable[[LeftT], RightT],
    output_view: View,
) -> ViewValue[LeftT, RightT]:
    if type(initial) is not ViewValue:
        raise TypeError("initial must be a ViewValue")
    if type(output_view) is not View:
        raise TypeError("output_view must be a View")
    if type(steps) is not tuple:
        raise TypeError("steps must be an immutable tuple")
    if len(steps) > _MAX_PIPELINE_STEPS:
        raise ValueError("step count exceeds the supported limit")
    if not callable(to_left) or not callable(to_right):
        raise TypeError("both conversion callbacks must be callable")
    for step in steps:
        if type(step) not in (LeftStep, RightStep):
            raise TypeError("every step must be a LeftStep or RightStep")
        if not callable(step.transform):
            raise TypeError("every step transform must be callable")

    current_view = initial.view
    current_value: Any = initial.value
    for step in steps:
        target_view = View.LEFT if type(step) is LeftStep else View.RIGHT
        if current_view is not target_view:
            current_value = (
                to_left(current_value)
                if target_view is View.LEFT
                else to_right(current_value)
            )
            current_view = target_view
        current_value = step.transform(current_value)

    if current_view is not output_view:
        current_value = (
            to_left(current_value)
            if output_view is View.LEFT
            else to_right(current_value)
        )
        current_view = output_view
    return ViewValue(current_view, current_value)
```

## Example

```python
conversion_calls: list[str] = []


def to_right(value: str | None) -> tuple[str, ...] | None:
    conversion_calls.append("left-to-right")
    if value is None:
        return None
    return tuple(part.strip() for part in value.split("|"))


def to_left(value: tuple[str, ...] | None) -> str | None:
    conversion_calls.append("right-to-left")
    if value is None:
        return None
    return "|".join(value)


steps = (
    LeftStep(lambda value: value.strip() if value is not None else None),
    RightStep(lambda value: value + ("gamma",) if value is not None else None),
    RightStep(
        lambda value: tuple(part.upper() for part in value)
        if value is not None
        else None
    ),
    LeftStep(lambda value: value + "!" if value is not None else None),
)
result = run_lazy_view_pipeline(
    ViewValue(View.LEFT, " alpha | beta "),
    steps=steps,
    to_left=to_left,
    to_right=to_right,
    output_view=View.RIGHT,
)

none_result = run_lazy_view_pipeline(
    ViewValue(View.LEFT, None),
    steps=(),
    to_left=lambda value: None,
    to_right=lambda value: None,
    output_view=View.RIGHT,
)

executed: list[bool] = []
try:
    run_lazy_view_pipeline(
        ViewValue(View.LEFT, "unchanged"),
        steps=(LeftStep(lambda value: executed.append(True)), object()),
        to_left=to_left,
        to_right=to_right,
        output_view=View.LEFT,
    )
except TypeError:
    invalid_pipeline_rejected = True
else:
    invalid_pipeline_rejected = False

failure = RuntimeError("step failed")


def fail_unchanged(value: str) -> str:
    raise failure


try:
    run_lazy_view_pipeline(
        ViewValue(View.LEFT, "value"),
        steps=(LeftStep(fail_unchanged),),
        to_left=to_left,
        to_right=to_right,
        output_view=View.LEFT,
    )
except RuntimeError as error:
    same_failure = error is failure
else:
    same_failure = False

assert (
    result,
    conversion_calls,
    none_result,
    invalid_pipeline_rejected,
    executed,
    same_failure,
) == (
    ViewValue(View.RIGHT, ("ALPHA", "BETA", "GAMMA!")),
    ["left-to-right", "right-to-left", "left-to-right"],
    ViewValue(View.RIGHT, None),
    True,
    [],
    True,
)
```

## Trade-offs and Limitations

The runner uses `O(1)` orchestration state beyond the immutable step tuple, but
each callback may allocate an entire replacement representation. It validates
the complete bounded pipeline and both converters before invoking anything.
Callback exceptions propagate unchanged, after any earlier callbacks and their
side effects have already occurred.

Frozen wrappers do not make mutable payloads or callback-owned state
immutable. Conversions may be lossy, expensive, or stateful, and a later step
can mutate a value before failing. There is no rollback, caching, thread
safety, parallel execution, runtime payload-type enforcement, or proof that
the two converters are inverses.

## Related Snippets

<!-- catalog:related:start -->
- [Dispatch Named Strategies with an Explicit Function Mapping](../python-language/dispatch-named-strategies-with-an-explicit-function-mapping.md)
- [Make a Defensive Copy at a Mutable Input Boundary](../python-language/make-a-defensive-copy-at-a-mutable-input-boundary.md)
- [Separate Executable and Redacted Views of a Command Argument Vector](../security-privacy/separate-executable-and-redacted-views-of-a-command-argument-vector.md)
<!-- catalog:related:end -->
