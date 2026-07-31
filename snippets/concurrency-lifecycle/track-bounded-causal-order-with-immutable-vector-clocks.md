---
title: "Track Bounded Causal Order with Immutable Vector Clocks"
snippet_type: algorithm
use_cases:
  - concurrency-control
  - data-transformation
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - report-equal-share-lease-ownership-excess-from-a-bounded-queue-snapshot.md
  - plan-bounded-worker-replacements-from-generation-and-restart-state.md
  - ../algorithms-data-structures/compute-the-transitive-reduction-of-a-bounded-directed-acyclic-graph.md
---

# Track Bounded Causal Order with Immutable Vector Clocks

## Idea and Problem

Represent causal knowledge for a fixed participant set as one immutable counter per participant.

A local event increments its participant's component. Receiving a clock first
merges componentwise maxima and then ticks the receiving participant. Two
clocks are causally ordered exactly when every component of one is no greater
than the corresponding component of the other. If each clock is greater in a
different component, their events are concurrent and the comparison preserves
that partial order instead of inventing a winner.

## When to Use

Use vector clocks when up to 64 known participants exchange event metadata and
the application needs to distinguish causal precedence from concurrency. The
participant tuple is part of every clock's identity, so independently created
clocks remain compatible only when they use the same canonical tuple. Serialize
each participant's local event stream, attach the current clock to messages,
and on receipt merge the attached clock before recording the receiving event
with `tick()`.

Use a scalar sequence or a consensus-backed log when all events need one total
order. Use a membership-aware causal-clock design when participants can join,
leave, or be renamed while clocks are live. The caller remains responsible for
transport, durable storage, and authenticating the participant that claims a
tick.

## Implementation

```python
import re
from dataclasses import dataclass
from enum import StrEnum

_MAX_PARTICIPANTS = 64
_MAX_PARTICIPANT_ID_LENGTH = 32
_MAX_COUNTER = (1 << 63) - 1
_PARTICIPANT_ID = re.compile(r"[a-z][a-z0-9_-]*", re.ASCII)


class CausalOrder(StrEnum):
    BEFORE = "before"
    EQUAL = "equal"
    AFTER = "after"
    CONCURRENT = "concurrent"


def _validate_participants(participants: object) -> tuple[str, ...]:
    if type(participants) is not tuple:
        raise TypeError("participants must be an exact tuple")
    if not 1 <= len(participants) <= _MAX_PARTICIPANTS:
        raise ValueError("participant count is outside 1..64")

    previous: str | None = None
    for participant in participants:
        if type(participant) is not str:
            raise TypeError("participant IDs must be exact strings")
        if not 1 <= len(participant) <= _MAX_PARTICIPANT_ID_LENGTH:
            raise ValueError("participant ID length is outside 1..32")
        if _PARTICIPANT_ID.fullmatch(participant) is None:
            raise ValueError("participant IDs must be canonical ASCII identifiers")
        if previous is not None and participant <= previous:
            raise ValueError("participant IDs must be unique and strictly sorted")
        previous = participant
    return participants


def _validate_counters(counters: object, participant_count: int) -> tuple[int, ...]:
    if type(counters) is not tuple:
        raise TypeError("counters must be an exact tuple")
    if len(counters) != participant_count:
        raise ValueError("one counter is required for every participant")
    for counter in counters:
        if type(counter) is not int:
            raise TypeError("counters must be exact integers")
        if not 0 <= counter <= _MAX_COUNTER:
            raise ValueError("counter is outside 0..2^63-1")
    return counters


@dataclass(frozen=True, slots=True)
class VectorClock:
    participants: tuple[str, ...]
    counters: tuple[int, ...]

    def __post_init__(self) -> None:
        checked_participants = _validate_participants(self.participants)
        _validate_counters(self.counters, len(checked_participants))

    @classmethod
    def zero(cls, participants: tuple[str, ...]) -> "VectorClock":
        checked_participants = _validate_participants(participants)
        return cls(checked_participants, (0,) * len(checked_participants))

    def tick(self, participant: str) -> "VectorClock":
        if type(participant) is not str:
            raise TypeError("participant must be an exact string")
        try:
            index = self.participants.index(participant)
        except ValueError:
            raise ValueError("cannot tick an unknown participant") from None

        current = self.counters[index]
        if current == _MAX_COUNTER:
            raise OverflowError("participant counter cannot be incremented")
        updated = (*self.counters[:index], current + 1, *self.counters[index + 1 :])
        return VectorClock(self.participants, updated)

    def _require_compatible(self, other: object) -> "VectorClock":
        if type(other) is not VectorClock:
            raise TypeError("other must be an exact VectorClock")
        if self.participants != other.participants:
            raise ValueError("vector clocks have different participant tuples")
        return other

    def merge(self, other: "VectorClock") -> "VectorClock":
        checked_other = self._require_compatible(other)
        counters = tuple(
            max(left, right)
            for left, right in zip(self.counters, checked_other.counters, strict=True)
        )
        return VectorClock(self.participants, counters)

    def compare(self, other: "VectorClock") -> CausalOrder:
        checked_other = self._require_compatible(other)
        saw_less = False
        saw_greater = False
        for left, right in zip(self.counters, checked_other.counters, strict=True):
            saw_less |= left < right
            saw_greater |= left > right
            if saw_less and saw_greater:
                return CausalOrder.CONCURRENT
        if saw_less:
            return CausalOrder.BEFORE
        if saw_greater:
            return CausalOrder.AFTER
        return CausalOrder.EQUAL
```

## Example

```python
from dataclasses import FrozenInstanceError
from itertools import product


def componentwise_order(left: tuple[int, ...], right: tuple[int, ...]) -> CausalOrder:
    less_or_equal = all(a <= b for a, b in zip(left, right, strict=True))
    greater_or_equal = all(a >= b for a, b in zip(left, right, strict=True))
    if less_or_equal and greater_or_equal:
        return CausalOrder.EQUAL
    if less_or_equal:
        return CausalOrder.BEFORE
    if greater_or_equal:
        return CausalOrder.AFTER
    return CausalOrder.CONCURRENT


participant_pool = ("alpha", "beta", "gamma")
checked_pairs = 0
checked_triples = 0
checked_ticks = 0

for participant_count in range(1, 4):
    participants = participant_pool[:participant_count]
    clocks = tuple(
        VectorClock(participants, counters)
        for counters in product(range(3), repeat=participant_count)
    )

    for left, right in product(clocks, repeat=2):
        merged = left.merge(right)
        assert left.compare(right) is componentwise_order(left.counters, right.counters)
        assert merged == right.merge(left)
        assert merged.counters == tuple(
            max(a, b) for a, b in zip(left.counters, right.counters, strict=True)
        )
        assert merged.compare(left) in (CausalOrder.EQUAL, CausalOrder.AFTER)
        assert merged.compare(right) in (CausalOrder.EQUAL, CausalOrder.AFTER)
        checked_pairs += 1

    for first, second, third in product(clocks, repeat=3):
        assert first.merge(second).merge(third) == first.merge(second.merge(third))
        checked_triples += 1

    for clock in clocks:
        assert clock.merge(clock) == clock
        for index, participant in enumerate(participants):
            ticked = clock.tick(participant)
            expected = list(clock.counters)
            expected[index] += 1
            assert ticked.counters == tuple(expected)
            assert ticked.compare(clock) is CausalOrder.AFTER
            checked_ticks += 1

maximum_participants = tuple(f"node-{index:02d}" for index in range(64))
zero = VectorClock.zero(maximum_participants)
maximum = VectorClock(
    maximum_participants,
    (_MAX_COUNTER,) + (0,) * (len(maximum_participants) - 1),
)

rejected_operations = (
    (lambda: maximum.tick("node-00"), OverflowError),
    (lambda: zero.tick("unknown"), ValueError),
    (lambda: zero.merge(VectorClock.zero(("other",))), ValueError),
    (lambda: zero.compare(VectorClock.zero(("other",))), ValueError),
    (lambda: zero.merge(object()), TypeError),
)
for operation, expected_error in rejected_operations:
    try:
        operation()
    except expected_error:
        pass
    else:
        raise AssertionError(f"expected {expected_error.__name__}")

invalid_clocks = (
    (lambda: VectorClock.zero(()), ValueError),
    (lambda: VectorClock.zero(("beta", "alpha")), ValueError),
    (lambda: VectorClock.zero(("alpha", "alpha")), ValueError),
    (lambda: VectorClock.zero(("UPPER",)), ValueError),
    (lambda: VectorClock.zero(("a" * 33,)), ValueError),
    (lambda: VectorClock.zero(tuple(f"p{index:02d}" for index in range(65))), ValueError),
    (lambda: VectorClock(("alpha",), ()), ValueError),
    (lambda: VectorClock(("alpha",), (True,)), TypeError),
    (lambda: VectorClock(("alpha",), (_MAX_COUNTER + 1,)), ValueError),
)
for constructor, expected_error in invalid_clocks:
    try:
        constructor()
    except expected_error:
        pass
    else:
        raise AssertionError(f"expected {expected_error.__name__}")

try:
    zero.counters = ()
except FrozenInstanceError:
    frozen = True
else:
    frozen = False

assert (
    checked_pairs,
    checked_triples,
    checked_ticks,
    zero.tick("node-63").counters[-1],
    VectorClock.zero(("a" * 32,)).counters,
    frozen,
) == (819, 20_439, 102, 1, (0,), True)
```

## Trade-offs and Limitations

Construction, ticking, merging, and comparison each take `O(p)` time for `p`
participants. A tick or merge allocates `O(p)` immutable tuple state, while
comparison keeps only bounded scalar state. The 64-participant limit bounds
both work and representation size, but each attached clock still grows
linearly with the closed participant set.

Vector-clock ordering is only as trustworthy as the event and message
discipline around it. Concurrent ticks for the same participant must be
serialized by the caller, received clocks must be merged before the receive
event is ticked, and counters cannot wrap. Componentwise merge is
commutative, associative, and idempotent, but it does not resolve concurrent
application values, establish physical-time order, authenticate senders, or
provide durable delivery. This profile also excludes dynamic membership,
dotted vectors, pruning, persistence, and transport encoding.

## Related Snippets

<!-- catalog:related:start -->
- [Report Equal-Share Lease Ownership Excess from a Bounded Queue Snapshot](report-equal-share-lease-ownership-excess-from-a-bounded-queue-snapshot.md)
- [Plan Bounded Worker Replacements from Generation and Restart State](plan-bounded-worker-replacements-from-generation-and-restart-state.md)
- [Compute the Transitive Reduction of a Bounded Directed Acyclic Graph](../algorithms-data-structures/compute-the-transitive-reduction-of-a-bounded-directed-acyclic-graph.md)
<!-- catalog:related:end -->
