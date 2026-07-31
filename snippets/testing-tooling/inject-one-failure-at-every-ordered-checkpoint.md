---
title: "Inject One Failure at Every Ordered Checkpoint"
snippet_type: pattern
use_cases:
  - lifecycle-management
  - resource-management
  - testing
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - assert-an-exact-iterator-pull-budget-with-a-fail-closed-probe.md
  - find-a-shortest-invariant-violation-in-a-bounded-deterministic-state-model.md
  - shrink-a-bounded-failing-sequence-to-a-one-deletion-minimal-subsequence.md
---

# Inject One Failure at Every Ordered Checkpoint

## Idea and Problem

Exercise one deterministic scenario repeatedly so a dedicated exception escapes from each declared checkpoint in turn.

Exception-safety bugs often hide between acquiring a resource, changing state,
and publishing a result. A small failpoint probe makes those boundaries
explicit. One run has no injected failure; every other fresh run selects one
checkpoint and raises there after recording the exact visited prefix.

The probe also validates its own test protocol. Checkpoints must occur exactly
once in declaration order, and `finish` must receive the exception that escaped
the scenario. Comparing that object by identity detects code that swallowed or
replaced the injected failure instead of allowing a false-positive test.

## When to Use

Use this pattern around a small deterministic unit whose exception guarantees
depend on a known sequence of meaningful boundaries: before and after resource
acquisition, before publication, or after a reversible mutation. Create a new
scenario state and a new probe for the successful run and for every target.

Use broader model-based or property-based testing when the operation order is
not fixed. Use dedicated concurrency tooling for schedules and races, and
process-level fault injection for crashes, signals, or abrupt termination.
Only checkpoints that the test author deliberately places can be exercised.

## Implementation

```python
class InjectedCheckpointFailure(RuntimeError):
    """The one failure object deliberately raised by a probe run."""

    def __init__(self, label: str) -> None:
        super().__init__(f"injected failure at checkpoint {label!r}")
        self.label = label


class FailpointProtocolError(AssertionError):
    """Raised when the scenario does not obey the declared probe protocol."""


class FailpointProbe:
    """Validate one ordered run and optionally fail at one checkpoint."""

    _MAX_LABELS = 32
    _MAX_LABEL_LENGTH = 32

    def __init__(
        self,
        labels: tuple[str, ...],
        target: str | None,
    ) -> None:
        if type(labels) is not tuple:
            raise TypeError("labels must be an exact tuple")
        if not 1 <= len(labels) <= self._MAX_LABELS:
            raise ValueError("labels must contain 1..32 entries")
        for index, label in enumerate(labels):
            if type(label) is not str:
                raise TypeError(f"labels[{index}] must be an exact string")
            if not 1 <= len(label) <= self._MAX_LABEL_LENGTH:
                raise ValueError(f"labels[{index}] length is outside 1..32")
            if any(not 32 <= ord(character) <= 126 for character in label):
                raise ValueError(f"labels[{index}] must contain printable ASCII")
        if len(set(labels)) != len(labels):
            raise ValueError("checkpoint labels must be unique")
        if target is not None and type(target) is not str:
            raise TypeError("target must be an exact string or None")
        if target is not None and target not in labels:
            raise ValueError("target must name a declared checkpoint")

        self._labels = labels
        self._target = target
        self._visited: list[str] = []
        self._tripped = False
        self._finished = False
        self._failure = InjectedCheckpointFailure(target) if target is not None else None

    @property
    def visited(self) -> tuple[str, ...]:
        """Return an immutable snapshot of checkpoints reached so far."""
        return tuple(self._visited)

    def checkpoint(self, label: str) -> None:
        """Record the next declared label and fail if it is the target."""
        if type(label) is not str:
            raise TypeError("label must be an exact string")
        if self._finished:
            raise FailpointProtocolError("probe has already finished")
        if self._tripped:
            raise FailpointProtocolError("no checkpoint is legal after failure")

        position = len(self._visited)
        if position == len(self._labels):
            raise FailpointProtocolError("scenario reached an extra checkpoint")
        expected = self._labels[position]
        if label != expected:
            raise FailpointProtocolError(f"expected checkpoint {expected!r}, received {label!r}")

        self._visited.append(label)
        if label == self._target:
            self._tripped = True
            assert self._failure is not None
            raise self._failure

    def finish(self, escaped: BaseException | None) -> tuple[str, ...]:
        """Validate the complete trace and the exact escaped exception."""
        if self._finished:
            raise FailpointProtocolError("finish may be called only once")
        self._finished = True
        visited = tuple(self._visited)

        if self._target is None:
            if escaped is not None:
                raise FailpointProtocolError("successful run leaked an exception")
            if visited != self._labels:
                raise FailpointProtocolError("successful run missed a checkpoint")
            return visited

        target_position = self._labels.index(self._target)
        expected_prefix = self._labels[: target_position + 1]
        if visited != expected_prefix:
            raise FailpointProtocolError("injected run has the wrong trace prefix")
        if escaped is not self._failure:
            raise FailpointProtocolError("the exact injected exception did not escape the scenario")
        return visited
```

## Example

```python
from dataclasses import dataclass, field

CHECKPOINTS = ("before acquire", "after acquire", "after mutation")


@dataclass
class ScenarioState:
    acquired: bool = False
    released: bool = False
    values: list[str] = field(default_factory=list)


def exercise(probe: FailpointProbe, state: ScenarioState) -> None:
    probe.checkpoint("before acquire")
    state.acquired = True
    try:
        probe.checkpoint("after acquire")
        state.values.append("changed")
        probe.checkpoint("after mutation")
    finally:
        state.released = True


for selected_target in (None, *CHECKPOINTS):
    current_probe = FailpointProbe(CHECKPOINTS, selected_target)
    current_state = ScenarioState()
    escaped_exception = None
    try:
        exercise(current_probe, current_state)
    except Exception as error:
        escaped_exception = error

    trace = current_probe.finish(escaped_exception)
    if selected_target is None:
        assert trace == CHECKPOINTS
        assert current_state.values == ["changed"]
    else:
        target_index = CHECKPOINTS.index(selected_target)
        assert trace == CHECKPOINTS[: target_index + 1]
        assert isinstance(escaped_exception, InjectedCheckpointFailure)
    assert current_state.released is current_state.acquired


def protocol_is_rejected(action: object) -> bool:
    try:
        action()  # type: ignore[operator]
    except (TypeError, ValueError, FailpointProtocolError):
        return True
    return False


swallowed = FailpointProbe(("only",), "only")
try:
    swallowed.checkpoint("only")
except InjectedCheckpointFailure:
    pass
assert protocol_is_rejected(lambda: swallowed.finish(None))

substituted = FailpointProbe(("only",), "only")
try:
    substituted.checkpoint("only")
except InjectedCheckpointFailure:
    pass
assert protocol_is_rejected(lambda: substituted.finish(RuntimeError("other")))

missing = FailpointProbe(("first", "second"), None)
missing.checkpoint("first")
assert protocol_is_rejected(lambda: missing.finish(None))

out_of_order = FailpointProbe(("first", "second"), None)
assert protocol_is_rejected(lambda: out_of_order.checkpoint("second"))

unknown_checkpoint = FailpointProbe(("first",), None)
assert protocol_is_rejected(lambda: unknown_checkpoint.checkpoint("unknown"))

repeated = FailpointProbe(("first", "second"), None)
repeated.checkpoint("first")
assert protocol_is_rejected(lambda: repeated.checkpoint("first"))

after_trip = FailpointProbe(("first", "second"), "first")
try:
    after_trip.checkpoint("first")
except InjectedCheckpointFailure:
    pass
assert protocol_is_rejected(lambda: after_trip.checkpoint("second"))

finished = FailpointProbe(("only",), None)
finished.checkpoint("only")
assert finished.finish(None) == ("only",)
assert protocol_is_rejected(lambda: finished.finish(None))
assert protocol_is_rejected(lambda: finished.checkpoint("only"))

assert protocol_is_rejected(lambda: FailpointProbe(("same", "same"), None))
assert protocol_is_rejected(lambda: FailpointProbe(("ok",), "unknown"))
assert protocol_is_rejected(lambda: FailpointProbe(("ok",), False))
assert protocol_is_rejected(lambda: FailpointProbe(("bad\n",), None))
```

## Trade-offs and Limitations

Each run uses one probe, at most 32 labels, and `O(number of visited labels)`
storage. A complete campaign executes the scenario once successfully and once
per label, so its cost is linear in the number of checkpoints times the cost
of a fresh scenario. Labels describe protocol boundaries; they are not code
coverage measurements.

The probe is deliberately synchronous, single-use, and not thread-safe. It
does not inject allocation failures, I/O errors with rich payloads, process
crashes, signals, timing changes, or concurrent schedules. It cannot prove
cleanup for effects that the scenario does not expose as assertions, and a
fresh isolated state is essential because earlier runs may have applied a
prefix of their mutations.

If cleanup raises another exception while an injected failure is active,
identity validation rejects the run. That is useful evidence that the original
failure was replaced, but the surrounding test must still decide which
exception guarantee and cleanup policy the operation should provide.

## Related Snippets

<!-- catalog:related:start -->
- [Assert an Exact Iterator Pull Budget with a Fail-Closed Probe](assert-an-exact-iterator-pull-budget-with-a-fail-closed-probe.md)
- [Find a Shortest Invariant Violation in a Bounded Deterministic State Model](find-a-shortest-invariant-violation-in-a-bounded-deterministic-state-model.md)
- [Shrink a Bounded Failing Sequence to a One-Deletion-Minimal Subsequence](shrink-a-bounded-failing-sequence-to-a-one-deletion-minimal-subsequence.md)
<!-- catalog:related:end -->
