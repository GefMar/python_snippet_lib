---
title: "Route Metric Label Values Through a Stable Overflow Bucket Under a Cardinality Cap"
snippet_type: pattern
use_cases:
  - observability
  - resource-management
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - group-metric-samples-by-their-exact-label-key-shape.md
  - collect-validated-metric-specifications-from-explicit-classes.md
  - ../reliability-resilience/plan-one-discrete-token-bucket-admission-from-an-explicit-tick-snapshot.md
---

# Route Metric Label Values Through a Stable Overflow Bucket Under a Cardinality Cap

## Idea and Problem

Route the first bounded set of distinct metric label values exactly and collapse later unseen values into one typed overflow outcome.

An immutable state records only admitted values in arrival order. Repeated
admitted values keep their exact route, while an overflow decision carries no
rejected value and leaves the state unchanged. Separate exact and overflow
types prevent a real label value from being mistaken for an in-band marker.

## When to Use

Use this pattern for one process-local metric label dimension whose observations
pass through one serialized owner. Choose the capacity before the state
lifetime begins, make each returned state the current snapshot before routing
the next observation, and map the typed overflow outcome to a backend
representation at a separate integration boundary.

Prefer a fixed label vocabulary when possible. Use a telemetry backend's own
global controls when cardinality must be coordinated across workers, hosts, or
restarts.

## Implementation

```python
from dataclasses import dataclass

_MAX_ADMITTED_VALUES = 256
_MAX_LABEL_VALUE_BYTES = 256


def _validated_label_value(value: object) -> str:
    if type(value) is not str:
        raise TypeError("label value must be an exact string")
    if len(value) > _MAX_LABEL_VALUE_BYTES:
        raise ValueError("label value exceeds the supported byte size")
    if any(not character.isprintable() for character in value):
        raise ValueError("label value must contain printable text only")
    if len(value.encode("utf-8")) > _MAX_LABEL_VALUE_BYTES:
        raise ValueError("label value exceeds the supported byte size")
    return value


@dataclass(frozen=True, slots=True)
class LabelCardinalityState:
    capacity: int
    admitted_values: tuple[str, ...] = ()

    def __post_init__(self) -> None:
        if type(self.capacity) is not int:
            raise TypeError("capacity must be an exact integer")
        if not 0 <= self.capacity <= _MAX_ADMITTED_VALUES:
            raise ValueError("capacity is outside the supported range")
        if type(self.admitted_values) is not tuple:
            raise TypeError("admitted_values must be an exact tuple")
        if len(self.admitted_values) > self.capacity:
            raise ValueError("admitted values exceed capacity")

        seen: set[str] = set()
        for value in self.admitted_values:
            validated = _validated_label_value(value)
            if validated in seen:
                raise ValueError("admitted values must be unique")
            seen.add(validated)


@dataclass(frozen=True, slots=True)
class ExactLabelRoute:
    value: str

    def __post_init__(self) -> None:
        _validated_label_value(self.value)


@dataclass(frozen=True, slots=True)
class OverflowLabelRoute:
    pass


@dataclass(frozen=True, slots=True)
class LabelRoutingDecision:
    state: LabelCardinalityState
    route: ExactLabelRoute | OverflowLabelRoute

    def __post_init__(self) -> None:
        if type(self.state) is not LabelCardinalityState:
            raise TypeError("state must be an exact LabelCardinalityState")
        if type(self.route) not in (ExactLabelRoute, OverflowLabelRoute):
            raise TypeError("route has an unsupported type")


def route_metric_label_value(
    state: LabelCardinalityState,
    value: str,
) -> LabelRoutingDecision:
    if type(state) is not LabelCardinalityState:
        raise TypeError("state must be an exact LabelCardinalityState")
    label_value = _validated_label_value(value)

    if label_value in state.admitted_values:
        return LabelRoutingDecision(state, ExactLabelRoute(label_value))
    if len(state.admitted_values) < state.capacity:
        successor = LabelCardinalityState(
            capacity=state.capacity,
            admitted_values=(*state.admitted_values, label_value),
        )
        return LabelRoutingDecision(successor, ExactLabelRoute(label_value))
    return LabelRoutingDecision(state, OverflowLabelRoute())
```

## Example

```python
initial = LabelCardinalityState(capacity=2)
first = route_metric_label_value(initial, "api")
marker_text = route_metric_label_value(first.state, "<overflow>")
overflow = route_metric_label_value(marker_text.state, "worker")
repeated = route_metric_label_value(overflow.state, "api")

assert (
    marker_text.state.admitted_values,
    marker_text.route,
    overflow.route,
    overflow.state is marker_text.state,
    repeated.state is marker_text.state,
    "worker" in overflow.state.admitted_values,
) == (
    ("api", "<overflow>"),
    ExactLabelRoute("<overflow>"),
    OverflowLabelRoute(),
    True,
    True,
    False,
)
```

## Trade-offs and Limitations

Stability lasts only along one serialized chain of successor states. Admission
is intentionally arrival-order dependent, so early arbitrary values can fill
the capacity and deny exact routes to later useful values. There is no
eviction, frequency policy, normalization, expiry, or recovery of identities
collapsed into overflow. A fresh state after restart can admit a different
set, and independent processes can diverge.

Immutability does not provide concurrency control. Two callers reducing the
same snapshot can each plan a different admission, so a shared owner must
serialize transitions or conditionally store one successor before emitting an
exact route. Validation and membership checks take linear time in at most 256
retained values, and each new admission copies that bounded tuple. The cap
controls one label dimension only; combinations with other labels can still
create many metric series. A backend adapter must preserve the typed distinction
or reserve its own overflow encoding rather than reintroducing a string
collision.

## Related Snippets

<!-- catalog:related:start -->
- [Group Metric Samples by Their Exact Label-Key Shape](group-metric-samples-by-their-exact-label-key-shape.md)
- [Collect Validated Metric Specifications from Explicit Classes](collect-validated-metric-specifications-from-explicit-classes.md)
- [Plan One Discrete Token-Bucket Admission from an Explicit Tick Snapshot](../reliability-resilience/plan-one-discrete-token-bucket-admission-from-an-explicit-tick-snapshot.md)
<!-- catalog:related:end -->
