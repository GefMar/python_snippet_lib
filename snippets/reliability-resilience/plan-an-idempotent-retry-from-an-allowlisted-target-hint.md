---
title: "Plan an Idempotent Retry from an Allowlisted Target Hint"
snippet_type: algorithm
use_cases:
  - concurrency-control
  - retry-recovery
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - plan-a-versioned-transition-for-the-current-workflow-attempt.md
  - retry-only-eligible-items-in-a-bounded-batch.md
  - plan-recovery-across-object-and-metadata-publication-states.md
---

# Plan an Idempotent Retry from an Allowlisted Target Hint

## Idea and Problem

Plan one retry of an idempotent operation without letting an untrusted target hint escape a closed allowlist.

The planner reduces one completed attempt and a coherent immutable pin snapshot
to `COMPLETE`, `RETRY`, or `FAIL` advice. It preserves the supplied
idempotency key exactly, uses only caller-supplied monotonic time, and describes
the conditional pin update that must precede a retry.

## When to Use

Use this algorithm when a completed attempt has already been classified as
successful, definitely retryable, terminal, or ambiguous, and retry permission
for the idempotent operation is an explicit boolean. The key must be the same
nonempty key used for the completed attempt, and both times must come from the
same monotonic clock epoch.

The current target must equal the pinned target when a pin exists, or the
initial target otherwise. No hint means stay on that current target. For a
definitely retryable failure, any present hint that is malformed or absent
from the table returns `FAIL` with `INVALID_TARGET_HINT`; a declared hint can
select only its validated table target. Hints do not override `SUCCESS`,
`TERMINAL_FAILURE`, or `AMBIGUOUS_OUTCOME` classification.

## Implementation

```python
import math
import re
from dataclasses import dataclass
from enum import StrEnum

_MAX_ATTEMPTS = 64
_MAX_HINTS = 32
_MAX_HINT_TARGETS = 16
_MAX_REVISION = 2**63 - 1
_TARGET_ID = re.compile(r"[A-Za-z0-9][A-Za-z0-9._:-]{0,63}", re.ASCII)
_HINT_TOKEN = re.compile(r"[A-Za-z0-9][A-Za-z0-9._:-]{0,31}", re.ASCII)
_IDEMPOTENCY_KEY = re.compile(
    r"[A-Za-z0-9][A-Za-z0-9._:-]{0,127}",
    re.ASCII,
)


class AttemptOutcome(StrEnum):
    SUCCESS = "success"
    DEFINITE_RETRYABLE_FAILURE = "definite-retryable-failure"
    TERMINAL_FAILURE = "terminal-failure"
    AMBIGUOUS_OUTCOME = "ambiguous-outcome"


class RetryAction(StrEnum):
    COMPLETE = "complete"
    RETRY = "retry"
    FAIL = "fail"


class RetryReason(StrEnum):
    SUCCEEDED = "succeeded"
    RETRY_APPROVED = "retry-approved"
    TERMINAL_FAILURE = "terminal-failure"
    AMBIGUOUS_OUTCOME = "ambiguous-outcome"
    INVALID_TARGET_HINT = "invalid-target-hint"
    RETRY_NOT_AUTHORIZED = "retry-not-authorized"
    ATTEMPT_BUDGET_EXHAUSTED = "attempt-budget-exhausted"
    DEADLINE_REACHED = "deadline-reached"
    PIN_REVISION_EXHAUSTED = "pin-revision-exhausted"


@dataclass(frozen=True, slots=True)
class AllowlistedTargetHint:
    token: str
    target_id: str


@dataclass(frozen=True, slots=True)
class TargetHintTable:
    entries: tuple[AllowlistedTargetHint, ...]


@dataclass(frozen=True, slots=True)
class PinnedTarget:
    target_id: str
    idempotency_key: str
    revision: int


@dataclass(frozen=True, slots=True)
class RetryAdvice:
    action: RetryAction
    reason: RetryReason
    target_id: str
    idempotency_key: str
    expected_pin_revision: int | None
    new_pin: PinnedTarget | None


def _validated_target_id(value: object, *, field: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if _TARGET_ID.fullmatch(value) is None:
        raise ValueError(f"{field} must be a 1 to 64 character ASCII target ID")
    return value


def _validated_key(value: object, *, field: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if _IDEMPOTENCY_KEY.fullmatch(value) is None:
        raise ValueError(f"{field} must be a nonempty 1 to 128 character ASCII token")
    return value


def _validated_revision(value: object, *, field: str) -> int:
    if type(value) is not int:
        raise TypeError(f"{field} must be an exact integer")
    if not 0 <= value <= _MAX_REVISION:
        raise ValueError(f"{field} is outside the supported revision range")
    return value


def _validated_attempt(value: object, *, field: str) -> int:
    if type(value) is not int:
        raise TypeError(f"{field} must be an exact integer")
    if not 1 <= value <= _MAX_ATTEMPTS:
        raise ValueError(f"{field} must be between 1 and {_MAX_ATTEMPTS}")
    return value


def _validated_time(value: object, *, field: str) -> float:
    if type(value) not in (int, float):
        raise TypeError(f"{field} must be an exact integer or float")
    try:
        converted = float(value)
    except OverflowError:
        raise ValueError(f"{field} must be representable as a finite float") from None
    if not math.isfinite(converted) or converted < 0.0:
        raise ValueError(f"{field} must be finite and non-negative")
    return converted


def _validated_pin(value: object, *, key: str) -> PinnedTarget | None:
    if value is None:
        return None
    if type(value) is not PinnedTarget:
        raise TypeError("pinned_target must be an exact PinnedTarget or None")
    pin = PinnedTarget(
        target_id=_validated_target_id(
            value.target_id,
            field="pinned_target.target_id",
        ),
        idempotency_key=_validated_key(
            value.idempotency_key,
            field="pinned_target.idempotency_key",
        ),
        revision=_validated_revision(
            value.revision,
            field="pinned_target.revision",
        ),
    )
    if pin.idempotency_key != key:
        raise ValueError("pinned_target changes the idempotency identity")
    return pin


def _validated_hint_table(value: object) -> dict[str, str]:
    if type(value) is not TargetHintTable:
        raise TypeError("hint_table must be an exact TargetHintTable")
    if type(value.entries) is not tuple:
        raise TypeError("hint_table.entries must be an exact tuple")
    if len(value.entries) > _MAX_HINTS:
        raise ValueError("hint_table exceeds the supported hint count")

    by_token: dict[str, str] = {}
    targets: set[str] = set()
    for index, raw_entry in enumerate(value.entries):
        field = f"hint_table.entries[{index}]"
        if type(raw_entry) is not AllowlistedTargetHint:
            raise TypeError(f"{field} must be an exact AllowlistedTargetHint")
        if type(raw_entry.token) is not str:
            raise TypeError(f"{field}.token must be an exact string")
        if _HINT_TOKEN.fullmatch(raw_entry.token) is None:
            raise ValueError(f"{field}.token must be a 1 to 32 character ASCII token")
        target_id = _validated_target_id(
            raw_entry.target_id,
            field=f"{field}.target_id",
        )
        if raw_entry.token in by_token:
            raise ValueError("hint_table tokens must be unique")
        by_token[raw_entry.token] = target_id
        targets.add(target_id)
    if len(targets) > _MAX_HINT_TARGETS:
        raise ValueError("hint_table exceeds the supported target count")
    return by_token


def _final_advice(
    action: RetryAction,
    reason: RetryReason,
    *,
    target_id: str,
    idempotency_key: str,
) -> RetryAdvice:
    return RetryAdvice(
        action=action,
        reason=reason,
        target_id=target_id,
        idempotency_key=idempotency_key,
        expected_pin_revision=None,
        new_pin=None,
    )


def plan_idempotent_retry(
    outcome: AttemptOutcome,
    *,
    current_target: str,
    initial_target: str,
    retry_authorized: bool,
    idempotency_key: str,
    attempt_number: int,
    max_attempts: int,
    observed_at: float,
    deadline: float,
    pinned_target: PinnedTarget | None,
    hint_token: object | None,
    hint_table: TargetHintTable,
) -> RetryAdvice:
    if type(outcome) is not AttemptOutcome:
        raise TypeError("outcome must be an exact AttemptOutcome")
    current = _validated_target_id(current_target, field="current_target")
    initial = _validated_target_id(initial_target, field="initial_target")
    if type(retry_authorized) is not bool:
        raise TypeError("retry_authorized must be an exact boolean")
    key = _validated_key(idempotency_key, field="idempotency_key")
    attempt = _validated_attempt(attempt_number, field="attempt_number")
    attempt_limit = _validated_attempt(max_attempts, field="max_attempts")
    if attempt > attempt_limit:
        raise ValueError("attempt_number cannot exceed max_attempts")
    observed = _validated_time(observed_at, field="observed_at")
    retry_deadline = _validated_time(deadline, field="deadline")
    pin = _validated_pin(pinned_target, key=key)
    targets_by_hint = _validated_hint_table(hint_table)

    governed_target = initial if pin is None else pin.target_id
    if current != governed_target:
        raise ValueError("current_target does not match the governing target state")

    hint_present = hint_token is not None
    if type(hint_token) is str and _HINT_TOKEN.fullmatch(hint_token) is not None:
        hinted_target = targets_by_hint.get(hint_token)
    else:
        hinted_target = None

    if outcome is AttemptOutcome.SUCCESS:
        return _final_advice(
            RetryAction.COMPLETE,
            RetryReason.SUCCEEDED,
            target_id=current,
            idempotency_key=key,
        )
    if outcome is AttemptOutcome.TERMINAL_FAILURE:
        return _final_advice(
            RetryAction.FAIL,
            RetryReason.TERMINAL_FAILURE,
            target_id=current,
            idempotency_key=key,
        )
    if outcome is AttemptOutcome.AMBIGUOUS_OUTCOME:
        return _final_advice(
            RetryAction.FAIL,
            RetryReason.AMBIGUOUS_OUTCOME,
            target_id=current,
            idempotency_key=key,
        )

    if hint_present and hinted_target is None:
        return _final_advice(
            RetryAction.FAIL,
            RetryReason.INVALID_TARGET_HINT,
            target_id=current,
            idempotency_key=key,
        )
    if not retry_authorized:
        return _final_advice(
            RetryAction.FAIL,
            RetryReason.RETRY_NOT_AUTHORIZED,
            target_id=current,
            idempotency_key=key,
        )
    if attempt >= attempt_limit:
        return _final_advice(
            RetryAction.FAIL,
            RetryReason.ATTEMPT_BUDGET_EXHAUSTED,
            target_id=current,
            idempotency_key=key,
        )
    if observed >= retry_deadline:
        return _final_advice(
            RetryAction.FAIL,
            RetryReason.DEADLINE_REACHED,
            target_id=current,
            idempotency_key=key,
        )
    if pin is not None and pin.revision == _MAX_REVISION:
        return _final_advice(
            RetryAction.FAIL,
            RetryReason.PIN_REVISION_EXHAUSTED,
            target_id=current,
            idempotency_key=key,
        )

    next_target = current if hinted_target is None else hinted_target
    next_revision = 0 if pin is None else pin.revision + 1
    return RetryAdvice(
        action=RetryAction.RETRY,
        reason=RetryReason.RETRY_APPROVED,
        target_id=next_target,
        idempotency_key=key,
        expected_pin_revision=None if pin is None else pin.revision,
        new_pin=PinnedTarget(next_target, key, next_revision),
    )
```

## Example

```python
hints = TargetHintTable(
    entries=(
        AllowlistedTargetHint("move-west", "worker-west"),
        AllowlistedTargetHint("stay-east", "worker-east"),
    )
)
pin = PinnedTarget("worker-east", "request-37", revision=4)
arguments = dict(
    current_target="worker-east",
    initial_target="worker-east",
    retry_authorized=True,
    idempotency_key="request-37",
    attempt_number=2,
    max_attempts=4,
    observed_at=20.0,
    deadline=30.0,
    pinned_target=pin,
    hint_table=hints,
)

retry = plan_idempotent_retry(
    AttemptOutcome.DEFINITE_RETRYABLE_FAILURE,
    hint_token="move-west",
    **arguments,
)
unknown = plan_idempotent_retry(
    AttemptOutcome.DEFINITE_RETRYABLE_FAILURE,
    hint_token="undeclared-target",
    **arguments,
)

assert (retry, unknown) == (
    RetryAdvice(
        RetryAction.RETRY,
        RetryReason.RETRY_APPROVED,
        "worker-west",
        "request-37",
        expected_pin_revision=4,
        new_pin=PinnedTarget("worker-west", "request-37", revision=5),
    ),
    RetryAdvice(
        RetryAction.FAIL,
        RetryReason.INVALID_TARGET_HINT,
        "worker-east",
        "request-37",
        expected_pin_revision=None,
        new_pin=None,
    ),
)
```

## Trade-offs and Limitations

The complete preflight validates every supplied state and policy field before
classifying advice: target IDs, key, table entries, entry and distinct-target
counts, attempts, pin revision, outcome, authorization, and finite non-negative
monotonic times. Inconsistent pin identity or target state raises instead of
being interpreted as a retry decision. The table accepts at most 32 unique
hint tokens naming at most 16 targets; IDs, hints, and the key are conservative
bounded ASCII tokens.

The function does not rewrite a URL, parse a header, consult proxy state, call
a transport, sleep, calculate backoff, or perform a network request. It also
cannot prove that the operation is idempotent, that the service honors the key,
or that any attempt was delivered. A stable idempotency key is not an
exactly-once guarantee, and an ambiguous completion always fails even when a
repeat might eventually be safe under a richer protocol.

Before executing `RETRY`, storage must atomically compare the pin against the
expected revision and replace it with `new_pin`; `None` means create only if no
pin exists. On a comparison miss, discard the advice and replan rather than
sending. Test all four outcomes, authorization denial, attempt and deadline
equality, absent and present pins, maximum revision, inconsistent identity,
duplicate and over-limit tables, no hint, malformed hints, unknown hints, and
every declared hint target.

## Related Snippets

<!-- catalog:related:start -->
- [Plan a Versioned Transition for the Current Workflow Attempt](plan-a-versioned-transition-for-the-current-workflow-attempt.md)
- [Retry Only Eligible Items in a Bounded Batch](retry-only-eligible-items-in-a-bounded-batch.md)
- [Plan Recovery Across Object and Metadata Publication States](plan-recovery-across-object-and-metadata-publication-states.md)
<!-- catalog:related:end -->
