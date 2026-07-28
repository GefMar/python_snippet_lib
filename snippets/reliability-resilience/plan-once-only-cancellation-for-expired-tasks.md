---
title: "Plan Once-only Cancellation for Expired Tasks"
snippet_type: pattern
use_cases:
  - lifecycle-management
  - concurrency-control
  - retry-recovery
tested_python:
  - "3.14"
dependencies: []
related:
  - plan-a-versioned-transition-for-the-current-workflow-attempt.md
  - retry-only-eligible-items-in-a-bounded-batch.md
  - ../storage-databases/compare-and-swap-a-versioned-sqlite-setting-with-one-conditional-update.md
---

# Plan Once-only Cancellation for Expired Tasks

## Idea and Problem

Claim each expired task atomically before sending one idempotent cancellation request.

A soft lifetime can exempt selected tasks, but the hard lifetime cannot. A stable
key derived from the logical task identity makes an ambiguous remote result safe
to retry from durable `cancel_pending` state without repeating a logical request.

## When to Use

Use this pattern for a bounded maintenance pass over task snapshots when several
workers may discover the same expiry and the remote cancellation endpoint accepts
an idempotency key. Each task is handled independently, so a claim or remote error
does not prevent later snapshots from being considered.

The claim adapter must compare the supplied version and persist the reason, key,
and `cancel_pending` state atomically before reporting a win. The remote adapter
must give the key durable idempotency semantics. Those are external guarantees;
this in-memory coordinator cannot create either one.

## Implementation

```python
import re
from dataclasses import dataclass
from datetime import UTC, datetime, timedelta
from enum import StrEnum
from hashlib import sha256
from typing import Protocol

_MAX_TASKS = 128
_MAX_DURATION_SECONDS = 31_536_000
_TASK_ID = re.compile(r"[a-z0-9](?:[a-z0-9._-]{0,62}[a-z0-9])?", re.ASCII)
_VERSION = re.compile(r"[A-Za-z0-9][A-Za-z0-9._:-]{0,63}", re.ASCII)


class ExpiryReason(StrEnum):
    SOFT = "soft_expired"
    HARD = "hard_expired"


class ClaimResult(StrEnum):
    WON = "won"
    LOST = "lost"


class CancellationRequestResult(StrEnum):
    REQUESTED = "requested"
    RETRYABLE_FAILURE = "retryable_failure"


class CancellationOutcomeKind(StrEnum):
    NOT_EXPIRED = "not_expired"
    SOFT_EXEMPT = "soft_exempt"
    CLAIM_LOST = "claim_lost"
    CLAIM_UNCERTAIN = "claim_uncertain"
    CANCEL_PENDING = "cancel_pending"
    CANCEL_REQUESTED = "cancel_requested"


@dataclass(frozen=True, slots=True)
class TaskSnapshot:
    task_id: str
    version: str
    expiry_started_at: datetime
    soft_exempt: bool
    cancel_pending_reason: ExpiryReason | None = None


@dataclass(frozen=True, slots=True)
class CancellationOutcome:
    task_id: str
    kind: CancellationOutcomeKind
    reason: ExpiryReason | None
    idempotency_key: str | None


class ClaimCancellation(Protocol):
    def __call__(
        self,
        *,
        task_id: str,
        expected_version: str,
        reason: ExpiryReason,
        idempotency_key: str,
    ) -> ClaimResult: ...


class RequestCancellation(Protocol):
    def __call__(
        self,
        *,
        task_id: str,
        reason: ExpiryReason,
        idempotency_key: str,
    ) -> CancellationRequestResult: ...


def _validated_utc(value: object, *, field: str) -> datetime:
    if type(value) is not datetime:
        raise TypeError(f"{field} must be an exact datetime")
    if value.utcoffset() != timedelta(0):
        raise ValueError(f"{field} must be an aware UTC datetime")
    return value.astimezone(UTC)


def _validated_task(value: object, *, index: int) -> TaskSnapshot:
    field = f"tasks[{index}]"
    if type(value) is not TaskSnapshot:
        raise TypeError(f"{field} must be an exact TaskSnapshot")
    if type(value.task_id) is not str or _TASK_ID.fullmatch(value.task_id) is None:
        raise ValueError(f"{field}.task_id must be a canonical 1-to-64-character ID")
    if type(value.version) is not str or _VERSION.fullmatch(value.version) is None:
        raise ValueError(f"{field}.version must be a bounded version token")
    if type(value.soft_exempt) is not bool:
        raise TypeError(f"{field}.soft_exempt must be a boolean")
    if (
        value.cancel_pending_reason is not None
        and type(value.cancel_pending_reason) is not ExpiryReason
    ):
        raise TypeError(f"{field}.cancel_pending_reason must be an ExpiryReason or None")
    return TaskSnapshot(
        task_id=value.task_id,
        version=value.version,
        expiry_started_at=_validated_utc(
            value.expiry_started_at,
            field=f"{field}.expiry_started_at",
        ),
        soft_exempt=value.soft_exempt,
        cancel_pending_reason=value.cancel_pending_reason,
    )


def _validated_duration(value: object, *, field: str) -> int:
    if type(value) is not int:
        raise TypeError(f"{field} must be an integer number of seconds")
    if not 1 <= value <= _MAX_DURATION_SECONDS:
        raise ValueError(f"{field} must be between 1 and 31,536,000 seconds")
    return value


def _idempotency_key(task: TaskSnapshot) -> str:
    parts = (
        b"expired-task-cancellation-v1",
        task.task_id.encode("ascii"),
        task.expiry_started_at.isoformat(timespec="microseconds").encode("ascii"),
    )
    digest = sha256()
    for part in parts:
        digest.update(len(part).to_bytes(2, "big"))
        digest.update(part)
    return f"expired-task-v1-{digest.hexdigest()}"


def cancel_expired_tasks_once(
    tasks: tuple[TaskSnapshot, ...],
    *,
    now: datetime,
    soft_after_seconds: int,
    hard_after_seconds: int,
    claim_cancellation: ClaimCancellation,
    request_cancellation: RequestCancellation,
) -> tuple[CancellationOutcome, ...]:
    if type(tasks) is not tuple:
        raise TypeError("tasks must be an exact tuple")
    if not 1 <= len(tasks) <= _MAX_TASKS:
        raise ValueError("tasks must contain between 1 and 128 snapshots")
    if not callable(claim_cancellation) or not callable(request_cancellation):
        raise TypeError("claim_cancellation and request_cancellation must be callable")

    observed_at = _validated_utc(now, field="now")
    soft_seconds = _validated_duration(
        soft_after_seconds,
        field="soft_after_seconds",
    )
    hard_seconds = _validated_duration(
        hard_after_seconds,
        field="hard_after_seconds",
    )
    if hard_seconds < soft_seconds:
        raise ValueError("hard_after_seconds must be at least soft_after_seconds")

    ordered = tuple(_validated_task(value, index=index) for index, value in enumerate(tasks))
    if len({task.task_id for task in ordered}) != len(ordered):
        raise ValueError("task IDs must be unique")

    soft_age = timedelta(seconds=soft_seconds)
    hard_age = timedelta(seconds=hard_seconds)
    outcomes: list[CancellationOutcome] = []

    for task in ordered:
        age = observed_at - task.expiry_started_at
        if task.cancel_pending_reason is not None:
            reason = task.cancel_pending_reason
        elif age >= hard_age:
            reason = ExpiryReason.HARD
        elif age >= soft_age:
            if task.soft_exempt:
                outcomes.append(
                    CancellationOutcome(
                        task.task_id,
                        CancellationOutcomeKind.SOFT_EXEMPT,
                        ExpiryReason.SOFT,
                        None,
                    )
                )
                continue
            reason = ExpiryReason.SOFT
        else:
            outcomes.append(
                CancellationOutcome(
                    task.task_id,
                    CancellationOutcomeKind.NOT_EXPIRED,
                    None,
                    None,
                )
            )
            continue

        idempotency_key = _idempotency_key(task)
        try:
            claim_result = claim_cancellation(
                task_id=task.task_id,
                expected_version=task.version,
                reason=reason,
                idempotency_key=idempotency_key,
            )
        except Exception:
            kind = CancellationOutcomeKind.CLAIM_UNCERTAIN
        else:
            if type(claim_result) is not ClaimResult:
                kind = CancellationOutcomeKind.CLAIM_UNCERTAIN
            elif claim_result is ClaimResult.LOST:
                kind = CancellationOutcomeKind.CLAIM_LOST
            else:
                try:
                    request_result = request_cancellation(
                        task_id=task.task_id,
                        reason=reason,
                        idempotency_key=idempotency_key,
                    )
                except Exception:
                    kind = CancellationOutcomeKind.CANCEL_PENDING
                else:
                    if request_result is CancellationRequestResult.REQUESTED:
                        kind = CancellationOutcomeKind.CANCEL_REQUESTED
                    else:
                        kind = CancellationOutcomeKind.CANCEL_PENDING

        outcomes.append(
            CancellationOutcome(
                task.task_id,
                kind,
                reason,
                idempotency_key,
            )
        )

    return tuple(outcomes)
```

## Example

```python
now = datetime(2031, 4, 5, 12, tzinfo=UTC)
tasks = (
    TaskSnapshot("maple-exempt", "rev-1", now - timedelta(seconds=10), True),
    TaskSnapshot("birch-hard", "rev-1", now - timedelta(seconds=20), True),
    TaskSnapshot("cedar-lost", "rev-1", now - timedelta(seconds=10), False),
    TaskSnapshot("dune-retry", "rev-1", now - timedelta(seconds=10), False),
    TaskSnapshot("elm-later", "rev-1", now - timedelta(seconds=10), False),
    TaskSnapshot(
        "frost-before",
        "rev-1",
        now - timedelta(seconds=10) + timedelta(microseconds=1),
        False,
    ),
)
versions = {task.task_id: task.version for task in tasks}
pending: dict[str, tuple[ExpiryReason, str]] = {}
claim_numbers: dict[str, int] = {}
claim_calls: list[tuple[str, str, ExpiryReason, str]] = []
remote_calls: list[tuple[str, ExpiryReason, str]] = []
remote_attempts: dict[str, int] = {}


def claim_cancellation(
    *,
    task_id: str,
    expected_version: str,
    reason: ExpiryReason,
    idempotency_key: str,
) -> ClaimResult:
    claim_calls.append((task_id, expected_version, reason, idempotency_key))
    if task_id == "cedar-lost" or versions[task_id] != expected_version:
        return ClaimResult.LOST
    claim_numbers[task_id] = claim_numbers.get(task_id, 0) + 1
    versions[task_id] = f"claimed-{claim_numbers[task_id]}"
    pending[task_id] = (reason, idempotency_key)
    return ClaimResult.WON


def request_cancellation(
    *,
    task_id: str,
    reason: ExpiryReason,
    idempotency_key: str,
) -> CancellationRequestResult:
    remote_calls.append((task_id, reason, idempotency_key))
    remote_attempts[task_id] = remote_attempts.get(task_id, 0) + 1
    if task_id == "dune-retry" and remote_attempts[task_id] == 1:
        raise TimeoutError
    return CancellationRequestResult.REQUESTED


first = cancel_expired_tasks_once(
    tasks,
    now=now,
    soft_after_seconds=10,
    hard_after_seconds=20,
    claim_cancellation=claim_cancellation,
    request_cancellation=request_cancellation,
)

assert tuple(outcome.kind for outcome in first) == (
    CancellationOutcomeKind.SOFT_EXEMPT,
    CancellationOutcomeKind.CANCEL_REQUESTED,
    CancellationOutcomeKind.CLAIM_LOST,
    CancellationOutcomeKind.CANCEL_PENDING,
    CancellationOutcomeKind.CANCEL_REQUESTED,
    CancellationOutcomeKind.NOT_EXPIRED,
)
assert first[0].reason is ExpiryReason.SOFT  # Exact soft boundary, but exempt.
assert first[1].reason is ExpiryReason.HARD  # Exact hard boundary overrides exemption.
assert [task_id for task_id, _, _ in remote_calls] == [
    "birch-hard",
    "dune-retry",
    "elm-later",
]

retry_task = TaskSnapshot(
    task_id="dune-retry",
    version=versions["dune-retry"],
    expiry_started_at=tasks[3].expiry_started_at,
    soft_exempt=False,
    cancel_pending_reason=pending["dune-retry"][0],
)
second = cancel_expired_tasks_once(
    (retry_task,),
    now=now,
    soft_after_seconds=10,
    hard_after_seconds=20,
    claim_cancellation=claim_cancellation,
    request_cancellation=request_cancellation,
)

retry_keys = [key for task_id, _, key in remote_calls if task_id == "dune-retry"]
assert second[0].kind is CancellationOutcomeKind.CANCEL_REQUESTED
assert retry_keys == [first[3].idempotency_key, first[3].idempotency_key]
assert claim_calls[-1][3] == first[3].idempotency_key
```

## Trade-offs and Limitations

The pass accepts one exact tuple containing 1 to 128 frozen snapshots, so a
custom iterable cannot understate the callback budget. IDs are unique lowercase
canonical tokens of at most 64 characters; versions are opaque conservative
tokens of at most 64 characters. All timestamps must be aware UTC values. Both
durations are integral seconds from 1 through 31,536,000, with the hard duration
not shorter than the soft duration. Equality counts as expired: hard is checked
before soft for newly expired tasks, and a soft exemption never suppresses hard
expiry. A snapshot already in `cancel_pending` preserves its claimed reason so
an idempotent retry does not change the remote request payload after crossing
the hard boundary.

There is at most one claim and one remote call per snapshot. Claim exceptions or
unknown results become `claim_uncertain`; remote exceptions, unknown results, and
reported failures become `cancel_pending`. Processing then continues in input
order. Validation copies every snapshot before the first callback, and neither
the caller's tuple nor its snapshots are mutated. Outcomes contain only
bounded identifiers, keys, reasons, and closed kinds; they never contain exception
text, tracebacks, or callback values. Control-flow exceptions such as cancellation
and keyboard interruption are not swallowed.

The deterministic key uses the task ID and expiry-start timestamp, not the
changing version or expiry reason. Those identity fields must remain stable across
snapshots of the same logical task. A pending soft cancellation can therefore be
retried with the same key even if it has since reached hard expiry. A
`cancel_requested` outcome means only that the remote accepted or deduplicated the
request; it does not confirm that the task stopped. Keep `cancel_pending` durable
until a separate observation or reconciliation step establishes the terminal
state.

## Related Snippets

<!-- catalog:related:start -->
- [Plan a Versioned Transition for the Current Workflow Attempt](plan-a-versioned-transition-for-the-current-workflow-attempt.md)
- [Retry Only Eligible Items in a Bounded Batch](retry-only-eligible-items-in-a-bounded-batch.md)
- [Compare and Swap a Versioned SQLite Setting with One Conditional Update](../storage-databases/compare-and-swap-a-versioned-sqlite-setting-with-one-conditional-update.md)
<!-- catalog:related:end -->
