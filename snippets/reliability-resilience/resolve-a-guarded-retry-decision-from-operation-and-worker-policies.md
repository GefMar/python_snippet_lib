---
title: "Resolve a Guarded Retry Decision from Operation and Worker Policies"
snippet_type: algorithm
use_cases:
  - retry-recovery
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - plan-an-idempotent-retry-from-an-allowlisted-target-hint.md
  - decide-whether-a-bounded-work-snapshot-permits-a-new-attempt.md
  - retry-only-eligible-items-in-a-bounded-batch.md
---

# Resolve a Guarded Retry Decision from Operation and Worker Policies

## Idea and Problem

Resolve retry advice from a closed failure vocabulary while keeping operation policy, worker fallback, ambiguity, and idempotency as separate gates.

An operation table chooses `RETRY`, `STOP`, or `ABSTAIN` for a known failure.
Only an explicit `ABSTAIN` consults the worker table; an absent operation rule
does not silently inherit a worker default. Ambiguous outcomes return `REPLAN`
before either policy is read, and an otherwise selected retry is downgraded to
`STOP` unless idempotency permission is exactly true.

## When to Use

Use this algorithm after an adapter has classified one completed outcome into
the closed `FailureKind` enum. Keep exception inspection and protocol-specific
classification at that adapter boundary rather than building a registry of
exception classes into retry policy.

Each policy is a finite immutable tuple of exact rules. Missing or unrecognized
failure input, missing rules, and an abstaining worker all fail closed. Attempt
counts and deadline budgets are the caller's responsibility: satisfying this
policy says only that retry is semantically eligible, not that any budget still
permits another attempt.

## Implementation

```python
from dataclasses import dataclass
from enum import StrEnum

_MAX_RULES_PER_POLICY = 5


class FailureKind(StrEnum):
    OVERLOADED = "overloaded"
    TEMPORARY_UNAVAILABLE = "temporary-unavailable"
    DEFINITE_TIMEOUT = "definite-timeout"
    REJECTED = "rejected"
    INVALID_RESPONSE = "invalid-response"
    AMBIGUOUS_OUTCOME = "ambiguous-outcome"


class RuleDisposition(StrEnum):
    RETRY = "retry"
    STOP = "stop"
    ABSTAIN = "abstain"


class RetryAdvice(StrEnum):
    RETRY = "retry"
    STOP = "stop"
    REPLAN = "replan"


class PolicyLayer(StrEnum):
    OPERATION = "operation"
    WORKER = "worker"


class RetryReason(StrEnum):
    MISSING_FAILURE_KIND = "missing-failure-kind"
    UNKNOWN_FAILURE_KIND = "unknown-failure-kind"
    AMBIGUOUS_OUTCOME = "ambiguous-outcome"
    OPERATION_RULE_MISSING = "operation-rule-missing"
    OPERATION_STOP = "operation-stop"
    WORKER_RULE_MISSING = "worker-rule-missing"
    WORKER_STOP = "worker-stop"
    WORKER_ABSTAINED = "worker-abstained"
    IDEMPOTENCY_PERMISSION_REQUIRED = "idempotency-permission-required"
    OPERATION_RETRY = "operation-retry"
    WORKER_RETRY = "worker-retry"


@dataclass(frozen=True, slots=True)
class RetryRule:
    failure: FailureKind
    disposition: RuleDisposition


@dataclass(frozen=True, slots=True)
class RetryPolicy:
    rules: tuple[RetryRule, ...]


@dataclass(frozen=True, slots=True)
class GuardedRetryDecision:
    advice: RetryAdvice
    reason: RetryReason
    failure: FailureKind | None
    selected_by: PolicyLayer | None


def _policy_rules(
    value: object,
    *,
    field: str,
) -> dict[FailureKind, RuleDisposition]:
    if type(value) is not RetryPolicy:
        raise TypeError(f"{field} must be an exact RetryPolicy")
    if type(value.rules) is not tuple:
        raise TypeError(f"{field}.rules must be an exact tuple")
    if len(value.rules) > _MAX_RULES_PER_POLICY:
        raise ValueError(f"{field}.rules exceeds the supported rule count")

    resolved: dict[FailureKind, RuleDisposition] = {}
    for index, rule in enumerate(value.rules):
        item = f"{field}.rules[{index}]"
        if type(rule) is not RetryRule:
            raise TypeError(f"{item} must be an exact RetryRule")
        if type(rule.failure) is not FailureKind:
            raise TypeError(f"{item}.failure must be an exact FailureKind")
        if type(rule.disposition) is not RuleDisposition:
            raise TypeError(f"{item}.disposition must be an exact RuleDisposition")
        if rule.failure is FailureKind.AMBIGUOUS_OUTCOME:
            raise ValueError("ambiguous outcomes cannot have policy rules")
        if rule.failure in resolved:
            raise ValueError(f"{field}.rules contains a duplicate failure")
        resolved[rule.failure] = rule.disposition
    return resolved


def resolve_guarded_retry(
    failure: object,
    *,
    operation_policy: RetryPolicy,
    worker_policy: RetryPolicy,
    idempotency_permitted: bool,
) -> GuardedRetryDecision:
    if type(idempotency_permitted) is not bool:
        raise TypeError("idempotency_permitted must be an exact boolean")

    def decision(
        advice: RetryAdvice,
        reason: RetryReason,
        known_failure: FailureKind | None = None,
        selected_by: PolicyLayer | None = None,
    ) -> GuardedRetryDecision:
        return GuardedRetryDecision(
            advice,
            reason,
            known_failure,
            selected_by,
        )

    if failure is None:
        return decision(RetryAdvice.STOP, RetryReason.MISSING_FAILURE_KIND)
    if type(failure) is not FailureKind:
        return decision(RetryAdvice.STOP, RetryReason.UNKNOWN_FAILURE_KIND)
    if failure is FailureKind.AMBIGUOUS_OUTCOME:
        return decision(
            RetryAdvice.REPLAN,
            RetryReason.AMBIGUOUS_OUTCOME,
            failure,
        )

    operation_rules = _policy_rules(
        operation_policy,
        field="operation_policy",
    )
    operation_choice = operation_rules.get(failure)
    if operation_choice is None:
        return decision(
            RetryAdvice.STOP,
            RetryReason.OPERATION_RULE_MISSING,
            failure,
        )
    if operation_choice is RuleDisposition.STOP:
        return decision(
            RetryAdvice.STOP,
            RetryReason.OPERATION_STOP,
            failure,
            PolicyLayer.OPERATION,
        )

    selected_by = PolicyLayer.OPERATION
    if operation_choice is RuleDisposition.ABSTAIN:
        worker_rules = _policy_rules(worker_policy, field="worker_policy")
        worker_choice = worker_rules.get(failure)
        if worker_choice is None:
            return decision(
                RetryAdvice.STOP,
                RetryReason.WORKER_RULE_MISSING,
                failure,
            )
        if worker_choice is RuleDisposition.STOP:
            return decision(
                RetryAdvice.STOP,
                RetryReason.WORKER_STOP,
                failure,
                PolicyLayer.WORKER,
            )
        if worker_choice is RuleDisposition.ABSTAIN:
            return decision(
                RetryAdvice.STOP,
                RetryReason.WORKER_ABSTAINED,
                failure,
                PolicyLayer.WORKER,
            )
        selected_by = PolicyLayer.WORKER

    if not idempotency_permitted:
        return decision(
            RetryAdvice.STOP,
            RetryReason.IDEMPOTENCY_PERMISSION_REQUIRED,
            failure,
            selected_by,
        )
    retry_reason = (
        RetryReason.OPERATION_RETRY
        if selected_by is PolicyLayer.OPERATION
        else RetryReason.WORKER_RETRY
    )
    return decision(RetryAdvice.RETRY, retry_reason, failure, selected_by)
```

## Example

```python
operation = RetryPolicy(
    (
        RetryRule(FailureKind.OVERLOADED, RuleDisposition.RETRY),
        RetryRule(
            FailureKind.TEMPORARY_UNAVAILABLE,
            RuleDisposition.ABSTAIN,
        ),
        RetryRule(FailureKind.REJECTED, RuleDisposition.STOP),
    )
)
worker = RetryPolicy(
    (
        RetryRule(
            FailureKind.TEMPORARY_UNAVAILABLE,
            RuleDisposition.RETRY,
        ),
        RetryRule(FailureKind.DEFINITE_TIMEOUT, RuleDisposition.RETRY),
    )
)

operation_retry = resolve_guarded_retry(
    FailureKind.OVERLOADED,
    operation_policy=operation,
    worker_policy=worker,
    idempotency_permitted=True,
)
worker_retry_blocked = resolve_guarded_retry(
    FailureKind.TEMPORARY_UNAVAILABLE,
    operation_policy=operation,
    worker_policy=worker,
    idempotency_permitted=False,
)
missing_operation_rule = resolve_guarded_retry(
    FailureKind.DEFINITE_TIMEOUT,
    operation_policy=operation,
    worker_policy=worker,
    idempotency_permitted=True,
)
ambiguous = resolve_guarded_retry(
    FailureKind.AMBIGUOUS_OUTCOME,
    operation_policy=operation,
    worker_policy=worker,
    idempotency_permitted=True,
)

assert (
    operation_retry,
    worker_retry_blocked,
    missing_operation_rule.reason,
    ambiguous.advice,
) == (
    GuardedRetryDecision(
        RetryAdvice.RETRY,
        RetryReason.OPERATION_RETRY,
        FailureKind.OVERLOADED,
        PolicyLayer.OPERATION,
    ),
    GuardedRetryDecision(
        RetryAdvice.STOP,
        RetryReason.IDEMPOTENCY_PERMISSION_REQUIRED,
        FailureKind.TEMPORARY_UNAVAILABLE,
        PolicyLayer.WORKER,
    ),
    RetryReason.OPERATION_RULE_MISSING,
    RetryAdvice.REPLAN,
)
```

## Trade-offs and Limitations

Each consulted policy is limited to five exact tuple entries. Rules must use the
closed enum values, duplicate failure rules are rejected, and unreachable
rules for ambiguous outcomes are invalid. Exact type checks prevent strings
from impersonating `StrEnum` members and integers from impersonating the
idempotency boolean. Advice and reasons are enum-backed values, so identical
validated inputs produce the same result without incorporating arbitrary
exception text.

Unknown and missing classifications stop before policy, and ambiguous outcomes
replan before policy. A missing operation rule stops without consulting the
worker; the worker is consulted only for operation `ABSTAIN`, and its missing
or abstaining result stops. This conservative resolver contains no exception-
class registry, callback, attempt loop, execution, sleep, clock access,
persistence, or ownership mechanism. The caller must enforce attempt and
deadline budgets and must perform any retry or replanning itself.

## Related Snippets

<!-- catalog:related:start -->
- [Plan an Idempotent Retry from an Allowlisted Target Hint](plan-an-idempotent-retry-from-an-allowlisted-target-hint.md)
- [Decide Whether a Bounded Work Snapshot Permits a New Attempt](decide-whether-a-bounded-work-snapshot-permits-a-new-attempt.md)
- [Retry Only Eligible Items in a Bounded Batch](retry-only-eligible-items-in-a-bounded-batch.md)
<!-- catalog:related:end -->
