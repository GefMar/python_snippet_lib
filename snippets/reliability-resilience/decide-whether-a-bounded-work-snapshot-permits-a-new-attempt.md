---
title: "Decide Whether a Bounded Work Snapshot Permits a New Attempt"
snippet_type: algorithm
use_cases:
  - lifecycle-management
  - retry-recovery
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - model-independent-blocking-reasons-as-an-immutable-set.md
  - ../observability-operations/resolve-the-latest-status-with-an-explicit-mapping.md
  - ../concurrency-lifecycle/select-a-reusable-lease-for-an-explicit-remaining-horizon.md
---

# Decide Whether a Bounded Work Snapshot Permits a New Attempt

## Idea and Problem

Classify whether one scope has blockers in a complete bounded snapshot of external work attempts.

The caller declares disjoint blocking and terminal status vocabularies. A
declared blocking status blocks only its matching scope, a terminal status does
not block, and any unmapped status in that scope fails closed as an explicit
unknown-status blocker. The result preserves snapshot order and exposes the
reason for every blocker.

## When to Use

Use this algorithm after obtaining one coherent, finite snapshot from an
external work system and before considering a new attempt for one scope. Keep
raw status strings at this boundary, and revise the two declared sets when the
external lifecycle changes.

Treat the result as advice about that snapshot. An executor must atomically
recheck authoritative state and claim permission before creating or starting
work; a read followed by this decision does not provide mutual exclusion.

## Implementation

```python
import re
from dataclasses import dataclass
from enum import StrEnum

_MAX_ATTEMPTS = 256
_MAX_STATUSES_PER_CLASS = 32
_MAX_STATUS_VOCABULARY = 64
_MAX_BLOCKERS = 64
_MAX_ATTEMPT_ID_BYTES = 96
_MAX_SCOPE_BYTES = 64
_MAX_STATUS_BYTES = 48
_MAX_TOTAL_TEXT_BYTES = 64 * 1024
_TOKEN = re.compile(r"[A-Za-z0-9][A-Za-z0-9._:-]*", re.ASCII)


class BlockerKind(StrEnum):
    DECLARED_BLOCKING = "declared-blocking"
    UNKNOWN_STATUS = "unknown-status"


@dataclass(frozen=True, slots=True)
class WorkAttemptSnapshot:
    attempt_id: str
    scope_key: str
    raw_status: str


@dataclass(frozen=True, slots=True)
class AttemptBlocker:
    attempt_id: str
    raw_status: str
    kind: BlockerKind


@dataclass(frozen=True, slots=True)
class NewAttemptDecision:
    scope_key: str
    permits_new_attempt: bool
    blockers: tuple[AttemptBlocker, ...]


def _validated_token(
    value: object,
    *,
    field: str,
    byte_limit: int,
) -> tuple[str, int]:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if not 1 <= len(value) <= byte_limit or _TOKEN.fullmatch(value) is None:
        raise ValueError(f"{field} must be a bounded conservative ASCII token")
    return value, len(value)


def _validated_status_set(value: object, *, field: str) -> frozenset[str]:
    if type(value) is not frozenset:
        raise TypeError(f"{field} must be an exact frozenset")
    if len(value) > _MAX_STATUSES_PER_CLASS:
        raise ValueError(f"{field} exceeds the supported status count")

    checked: set[str] = set()
    for raw_status in value:
        status, _ = _validated_token(
            raw_status,
            field=f"a status in {field}",
            byte_limit=_MAX_STATUS_BYTES,
        )
        checked.add(status)
    return frozenset(checked)


def _validated_snapshot(
    value: object,
) -> tuple[tuple[WorkAttemptSnapshot, ...], int]:
    if type(value) is not tuple:
        raise TypeError("attempts must be an exact tuple")
    if len(value) > _MAX_ATTEMPTS:
        raise ValueError("attempts exceeds the supported record count")

    checked: list[WorkAttemptSnapshot] = []
    seen_ids: set[str] = set()
    total_text_bytes = 0
    for index, raw_attempt in enumerate(value):
        field = f"attempts[{index}]"
        if type(raw_attempt) is not WorkAttemptSnapshot:
            raise TypeError(f"{field} must be an exact WorkAttemptSnapshot")
        attempt_id, id_bytes = _validated_token(
            raw_attempt.attempt_id,
            field=f"{field}.attempt_id",
            byte_limit=_MAX_ATTEMPT_ID_BYTES,
        )
        scope_key, scope_bytes = _validated_token(
            raw_attempt.scope_key,
            field=f"{field}.scope_key",
            byte_limit=_MAX_SCOPE_BYTES,
        )
        raw_status, status_bytes = _validated_token(
            raw_attempt.raw_status,
            field=f"{field}.raw_status",
            byte_limit=_MAX_STATUS_BYTES,
        )
        if attempt_id in seen_ids:
            raise ValueError("attempt IDs must be unique across the snapshot")
        seen_ids.add(attempt_id)
        total_text_bytes += id_bytes + scope_bytes + status_bytes
        if total_text_bytes > _MAX_TOTAL_TEXT_BYTES:
            raise ValueError("snapshot text exceeds the aggregate byte limit")
        checked.append(WorkAttemptSnapshot(attempt_id, scope_key, raw_status))
    return tuple(checked), total_text_bytes


def decide_new_attempt_permission(
    attempts: tuple[WorkAttemptSnapshot, ...],
    *,
    requested_scope: str,
    blocking_statuses: frozenset[str],
    terminal_statuses: frozenset[str],
) -> NewAttemptDecision:
    blocking = _validated_status_set(
        blocking_statuses,
        field="blocking_statuses",
    )
    terminal = _validated_status_set(
        terminal_statuses,
        field="terminal_statuses",
    )
    if blocking & terminal:
        raise ValueError("blocking_statuses and terminal_statuses must be disjoint")
    if len(blocking | terminal) > _MAX_STATUS_VOCABULARY:
        raise ValueError("declared status vocabulary exceeds the supported count")

    scope_key, scope_bytes = _validated_token(
        requested_scope,
        field="requested_scope",
        byte_limit=_MAX_SCOPE_BYTES,
    )
    snapshot, snapshot_bytes = _validated_snapshot(attempts)
    vocabulary_bytes = sum(len(status) for status in blocking | terminal)
    if snapshot_bytes + scope_bytes + vocabulary_bytes > _MAX_TOTAL_TEXT_BYTES:
        raise ValueError("all supplied text exceeds the aggregate byte limit")

    blockers: list[AttemptBlocker] = []
    for attempt in snapshot:
        if attempt.scope_key != scope_key or attempt.raw_status in terminal:
            continue
        if attempt.raw_status in blocking:
            kind = BlockerKind.DECLARED_BLOCKING
        else:
            kind = BlockerKind.UNKNOWN_STATUS
        blockers.append(AttemptBlocker(attempt.attempt_id, attempt.raw_status, kind))
        if len(blockers) > _MAX_BLOCKERS:
            raise ValueError("blocker output exceeds the supported record count")

    frozen_blockers = tuple(blockers)
    return NewAttemptDecision(
        scope_key=scope_key,
        permits_new_attempt=not frozen_blockers,
        blockers=frozen_blockers,
    )
```

## Example

```python
attempts = (
    WorkAttemptSnapshot("attempt-finished", "daily-index", "done"),
    WorkAttemptSnapshot("attempt-active", "daily-index", "running"),
    WorkAttemptSnapshot("attempt-unrelated", "monthly-export", "running"),
    WorkAttemptSnapshot("attempt-new-state", "daily-index", "awaiting-review"),
)
decision = decide_new_attempt_permission(
    attempts,
    requested_scope="daily-index",
    blocking_statuses=frozenset({"queued", "running"}),
    terminal_statuses=frozenset({"cancelled", "done", "failed"}),
)

assert decision == NewAttemptDecision(
    scope_key="daily-index",
    permits_new_attempt=False,
    blockers=(
        AttemptBlocker(
            "attempt-active",
            "running",
            BlockerKind.DECLARED_BLOCKING,
        ),
        AttemptBlocker(
            "attempt-new-state",
            "awaiting-review",
            BlockerKind.UNKNOWN_STATUS,
        ),
    ),
)
```

## Trade-offs and Limitations

Validation and classification are linear in at most 256 attempts and 64
declared statuses. IDs, scope keys, statuses, aggregate text, and the 64-record
blocker output are independently bounded. The returned frozen tuple keeps the
validated input order; no sorting or external status precedence is implied.

An unknown status fails closed only for the requested scope. This prevents
silent admission after status-vocabulary drift while allowing unrelated work
to remain unrelated. The helper performs no I/O, polling, work creation, task
execution, locking, or state mutation. It neither acquires ownership nor
guarantees mutual exclusion; an executor needs an atomic authoritative
recheck-and-claim operation.

## Related Snippets

<!-- catalog:related:start -->
- [Model Independent Blocking Reasons as an Immutable Set](model-independent-blocking-reasons-as-an-immutable-set.md)
- [Resolve the Latest Status with an Explicit Mapping](../observability-operations/resolve-the-latest-status-with-an-explicit-mapping.md)
- [Select a Reusable Lease for an Explicit Remaining Horizon](../concurrency-lifecycle/select-a-reusable-lease-for-an-explicit-remaining-horizon.md)
<!-- catalog:related:end -->
