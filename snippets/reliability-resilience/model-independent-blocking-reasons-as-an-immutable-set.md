---
title: "Model Independent Blocking Reasons as an Immutable Set"
snippet_type: pattern
use_cases:
  - configuration
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - hold-a-switch-active-through-a-monotonic-cooldown.md
  - ../observability-operations/resolve-the-latest-status-with-an-explicit-mapping.md
---

# Model Independent Blocking Reasons as an Immutable Set

## Idea and Problem

Represent independent reasons that prevent an operation as immutable identifiers and derive the combined blocked state from whether that set is empty.

Each policy adds or removes only its own stable reason ID. No separate boolean
can drift out of sync, and idempotent updates make repeated observations safe.
Canonical sorted serialization keeps the value deterministic at persistence or
message boundaries.

## When to Use

Use this value model when several independent checks may block the same local
operation and clearing one check must not erase the others. Define reason IDs
as stable machine-facing protocol values and keep human messages elsewhere.
Wrap persistence in compare-and-swap or a transaction when multiple writers
can update the stored state concurrently.

## Implementation

```python
import re
from collections.abc import Iterable
from dataclasses import dataclass


_REASON_ID = re.compile(r"[a-z][a-z0-9_.-]{0,63}", re.ASCII)


def _validate_reason_id(reason: str) -> str:
    if not isinstance(reason, str):
        raise TypeError("reason must be text")
    if _REASON_ID.fullmatch(reason) is None:
        raise ValueError("reason is not a stable reason identifier")
    return reason


@dataclass(frozen=True, slots=True, init=False)
class BlockingState:
    reasons: frozenset[str]

    def __init__(self, reasons: Iterable[str] = ()) -> None:
        if isinstance(reasons, (str, bytes)):
            raise TypeError("reasons must be an iterable of reason IDs")
        collected: set[str] = set()
        for reason in reasons:
            value = _validate_reason_id(reason)
            if value in collected:
                raise ValueError("reasons contain a duplicate ID")
            collected.add(value)
        object.__setattr__(self, "reasons", frozenset(collected))

    @property
    def blocked(self) -> bool:
        return bool(self.reasons)

    def add(self, reason: str) -> "BlockingState":
        value = _validate_reason_id(reason)
        if value in self.reasons:
            return self
        return BlockingState((*self.reasons, value))

    def remove(self, reason: str) -> "BlockingState":
        value = _validate_reason_id(reason)
        if value not in self.reasons:
            return self
        return BlockingState(
            existing for existing in self.reasons if existing != value
        )

    def to_ids(self) -> tuple[str, ...]:
        return tuple(sorted(self.reasons))
```

## Example

```python
initial = BlockingState()
with_window = initial.add("maintenance_window")
with_both = with_window.add("dependency.unavailable")
one_cleared = with_both.remove("maintenance_window")
all_cleared = one_cleared.remove("dependency.unavailable")

try:
    BlockingState(["duplicate", "duplicate"])
except ValueError:
    duplicate_rejected = True
else:
    duplicate_rejected = False

assert (
    initial.blocked,
    with_both.blocked,
    with_both.to_ids(),
    one_cleared.to_ids(),
    all_cleared.blocked,
    with_both.add("maintenance_window") is with_both,
    with_both.remove("not_present") is with_both,
    initial.to_ids(),
    duplicate_rejected,
) == (
    False,
    True,
    ("dependency.unavailable", "maintenance_window"),
    ("dependency.unavailable",),
    False,
    True,
    True,
    (),
    True,
)
```

## Trade-offs and Limitations

This is an immutable state value, not a thread or process synchronization lock,
an authorization decision, or a policy evaluator. Persistence, atomic updates,
expiry, reason precedence, auditing, and user-facing explanations remain the
caller's responsibility. Returning the same instance for an idempotent update
is safe because the value cannot change, but callers should compare values
rather than rely on identity. Adding the same persisted reason twice is
rejected so malformed serialized state is not silently normalized.

## Related Snippets

<!-- catalog:related:start -->
- [Hold a Switch Active Through a Monotonic Cooldown](hold-a-switch-active-through-a-monotonic-cooldown.md)
- [Resolve the Latest Status with an Explicit Mapping](../observability-operations/resolve-the-latest-status-with-an-explicit-mapping.md)
<!-- catalog:related:end -->
