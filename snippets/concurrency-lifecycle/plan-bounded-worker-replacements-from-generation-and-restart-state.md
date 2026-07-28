---
title: "Plan Bounded Worker Replacements from Generation and Restart State"
snippet_type: algorithm
use_cases:
  - concurrency-control
  - lifecycle-management
  - retry-recovery
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - plan-a-keyed-worker-set-reload-from-a-coalesced-sighup-request.md
  - plan-priority-batches-with-an-age-gated-tail.md
  - ../observability-operations/classify-required-health-stamps-by-freshness.md
---

# Plan Bounded Worker Replacements from Generation and Restart State

## Idea and Problem

Reconcile bounded immutable desired and observed worker snapshots into exactly one deterministic advisory outcome per target.

This is more than a worker-name configuration reload. A retained name is safe
to retain only when its observed generation matches the desired generation and
its explicit health is healthy. Generation drift or unhealthy state requests
replacement, while a rolling restart budget and cooldown can instead produce
`EXHAUSTED` or `DEFER_COOLDOWN`. New desired names produce `START`, and names no
longer desired produce `RETIRE`.

## When to Use

Use this planner when one coordinator can take complete finite snapshots,
supply a monotonic `observed_at`, and give the complete snapshot a stable
revision that changes with authoritative desired or observed state. Each restart
timestamp represents an already-recorded replacement attempt, histories are
strictly increasing, and the restart window includes its lower boundary. Known
failed targets must remain in the observed snapshot so their history and budget
cannot disappear merely because no healthy instance is available.

The executor must atomically revalidate the authoritative desired revision,
observed generation, and health before acting. In the same transaction or
critical section it must record the action token as claimed or completed, so a
retry cannot execute one plan twice. Use a durable supervisor or transactional
coordination design when that atomic boundary does not exist.

## Implementation

```python
import math
import re
from dataclasses import dataclass
from enum import StrEnum

_MAX_TARGETS = 64
_MAX_HISTORY_PER_TARGET = 128
_MAX_TOTAL_HISTORY_ENTRIES = 2_048
_MAX_GENERATION = (1 << 63) - 1
_MAX_RESTARTS_IN_WINDOW = 128
_MAX_WINDOW_SECONDS = 31 * 24 * 60 * 60
_MAX_COOLDOWN_SECONDS = 7 * 24 * 60 * 60
_TARGET_ID = re.compile(r"[a-z][a-z0-9]*(?:-[a-z0-9]+)*", re.ASCII)
_REVISION = re.compile(r"[A-Za-z0-9][A-Za-z0-9._-]{0,63}", re.ASCII)


class WorkerHealth(StrEnum):
    HEALTHY = "healthy"
    UNHEALTHY = "unhealthy"


class WorkerOutcome(StrEnum):
    RETAIN = "retain"
    START = "start"
    REPLACE = "replace"
    DEFER_COOLDOWN = "defer-cooldown"
    EXHAUSTED = "exhausted"
    RETIRE = "retire"


@dataclass(frozen=True, slots=True)
class RestartPolicy:
    window_seconds: float
    max_restarts_in_window: int
    cooldown_seconds: float


@dataclass(frozen=True, slots=True)
class DesiredWorker:
    target_id: str
    generation: int
    restart_policy: RestartPolicy


@dataclass(frozen=True, slots=True)
class ObservedWorker:
    target_id: str
    generation: int
    health: WorkerHealth
    restart_times: tuple[float, ...] = ()


@dataclass(frozen=True, slots=True)
class WorkerAction:
    target_id: str
    outcome: WorkerOutcome
    desired_generation: int | None
    observed_generation: int | None
    restarts_in_window: int
    action_token: str


@dataclass(frozen=True, slots=True)
class WorkerReplacementPlan:
    observed_at: float
    revision: str
    actions: tuple[WorkerAction, ...]


def _identifier(value: object) -> str:
    if type(value) is not str:
        raise TypeError("target_id must be an exact string")
    if len(value) > 64 or _TARGET_ID.fullmatch(value) is None:
        raise ValueError("target_id must be a bounded canonical ASCII identifier")
    return value


def _generation(value: object) -> int:
    if type(value) is not int:
        raise TypeError("generation must be an exact integer")
    if not 0 <= value <= _MAX_GENERATION:
        raise ValueError("generation is outside the supported range")
    return value


def _finite_nonnegative(value: object, *, field: str) -> float:
    if type(value) not in (int, float):
        raise TypeError(f"{field} must be an exact int or float")
    try:
        converted = float(value)
    except OverflowError:
        raise ValueError(f"{field} cannot be represented as a float") from None
    if not math.isfinite(converted) or converted < 0:
        raise ValueError(f"{field} must be finite and nonnegative")
    return converted


def _checked_policy(value: object) -> RestartPolicy:
    if type(value) is not RestartPolicy:
        raise TypeError("restart_policy must be an exact RestartPolicy")
    window = _finite_nonnegative(value.window_seconds, field="window_seconds")
    if not 0 < window <= _MAX_WINDOW_SECONDS:
        raise ValueError("window_seconds is outside the supported range")
    if type(value.max_restarts_in_window) is not int:
        raise TypeError("max_restarts_in_window must be an exact integer")
    if not 0 <= value.max_restarts_in_window <= _MAX_RESTARTS_IN_WINDOW:
        raise ValueError("max_restarts_in_window is outside the supported range")
    cooldown = _finite_nonnegative(value.cooldown_seconds, field="cooldown_seconds")
    if cooldown > _MAX_COOLDOWN_SECONDS:
        raise ValueError("cooldown_seconds exceeds the supported range")
    return RestartPolicy(window, value.max_restarts_in_window, cooldown)


def _checked_desired(
    desired: object,
) -> dict[str, DesiredWorker]:
    if type(desired) is not tuple:
        raise TypeError("desired must be an exact tuple")
    if len(desired) > _MAX_TARGETS:
        raise ValueError("desired target count exceeds the supported limit")

    checked: dict[str, DesiredWorker] = {}
    for worker in desired:
        if type(worker) is not DesiredWorker:
            raise TypeError("desired must contain exact DesiredWorker values")
        target_id = _identifier(worker.target_id)
        if target_id in checked:
            raise ValueError("desired target IDs must be unique")
        checked[target_id] = DesiredWorker(
            target_id,
            _generation(worker.generation),
            _checked_policy(worker.restart_policy),
        )
    return checked


def _checked_observed(
    observed: object,
    *,
    observed_at: float,
) -> dict[str, ObservedWorker]:
    if type(observed) is not tuple:
        raise TypeError("observed must be an exact tuple")
    if len(observed) > _MAX_TARGETS:
        raise ValueError("observed target count exceeds the supported limit")

    checked: dict[str, ObservedWorker] = {}
    total_history_entries = 0
    for worker in observed:
        if type(worker) is not ObservedWorker:
            raise TypeError("observed must contain exact ObservedWorker values")
        target_id = _identifier(worker.target_id)
        if target_id in checked:
            raise ValueError("observed target IDs must be unique")
        generation = _generation(worker.generation)
        if type(worker.health) is not WorkerHealth:
            raise TypeError("health must be an exact WorkerHealth value")
        if type(worker.restart_times) is not tuple:
            raise TypeError("restart_times must be an exact tuple")
        if len(worker.restart_times) > _MAX_HISTORY_PER_TARGET:
            raise ValueError("a restart history exceeds its entry limit")
        total_history_entries += len(worker.restart_times)
        if total_history_entries > _MAX_TOTAL_HISTORY_ENTRIES:
            raise ValueError("total restart history exceeds its entry limit")

        times: list[float] = []
        previous: float | None = None
        for raw_time in worker.restart_times:
            restart_time = _finite_nonnegative(raw_time, field="restart_time")
            if restart_time > observed_at:
                raise ValueError("restart times must not be later than observed_at")
            if previous is not None and restart_time <= previous:
                raise ValueError("restart times must be strictly increasing")
            times.append(restart_time)
            previous = restart_time
        checked[target_id] = ObservedWorker(
            target_id,
            generation,
            worker.health,
            tuple(times),
        )
    return checked


def _active_restarts(
    worker: ObservedWorker,
    policy: RestartPolicy,
    observed_at: float,
) -> int:
    lower_bound = observed_at - policy.window_seconds
    return sum(restart_time >= lower_bound for restart_time in worker.restart_times)


def _replacement_outcome(
    worker: ObservedWorker,
    policy: RestartPolicy,
    *,
    observed_at: float,
    active_restarts: int,
) -> WorkerOutcome:
    if active_restarts >= policy.max_restarts_in_window:
        return WorkerOutcome.EXHAUSTED
    if worker.restart_times and observed_at - worker.restart_times[-1] < policy.cooldown_seconds:
        return WorkerOutcome.DEFER_COOLDOWN
    return WorkerOutcome.REPLACE


def _action_token(
    revision: str,
    target_id: str,
    outcome: WorkerOutcome,
    desired_generation: int | None,
    observed_generation: int | None,
) -> str:
    desired = "-" if desired_generation is None else str(desired_generation)
    observed = "-" if observed_generation is None else str(observed_generation)
    return f"{revision}:{target_id}:{outcome.value}:{desired}:{observed}"


def plan_worker_replacements(
    desired: tuple[DesiredWorker, ...],
    observed: tuple[ObservedWorker, ...],
    *,
    observed_at: float,
    revision: str,
) -> WorkerReplacementPlan:
    checked_at = _finite_nonnegative(observed_at, field="observed_at")
    if type(revision) is not str:
        raise TypeError("revision must be an exact string")
    if _REVISION.fullmatch(revision) is None:
        raise ValueError("revision must be a bounded canonical ASCII token")

    desired_by_id = _checked_desired(desired)
    observed_by_id = _checked_observed(observed, observed_at=checked_at)

    actions: list[WorkerAction] = []
    for target_id in sorted(desired_by_id.keys() | observed_by_id.keys()):
        wanted = desired_by_id.get(target_id)
        current = observed_by_id.get(target_id)
        desired_generation = None if wanted is None else wanted.generation
        observed_generation = None if current is None else current.generation

        if wanted is None:
            outcome = WorkerOutcome.RETIRE
            active_restarts = 0
        elif current is None:
            outcome = WorkerOutcome.START
            active_restarts = 0
        else:
            active_restarts = _active_restarts(
                current,
                wanted.restart_policy,
                checked_at,
            )
            if current.generation == wanted.generation and current.health is WorkerHealth.HEALTHY:
                outcome = WorkerOutcome.RETAIN
            else:
                outcome = _replacement_outcome(
                    current,
                    wanted.restart_policy,
                    observed_at=checked_at,
                    active_restarts=active_restarts,
                )

        actions.append(
            WorkerAction(
                target_id,
                outcome,
                desired_generation,
                observed_generation,
                active_restarts,
                _action_token(
                    revision,
                    target_id,
                    outcome,
                    desired_generation,
                    observed_generation,
                ),
            )
        )

    return WorkerReplacementPlan(checked_at, revision, tuple(actions))
```

## Example

```python
policy = RestartPolicy(
    window_seconds=60.0,
    max_restarts_in_window=2,
    cooldown_seconds=10.0,
)
desired = (
    DesiredWorker("gamma-replace", 2, policy),
    DesiredWorker("alpha-retain", 2, policy),
    DesiredWorker("epsilon-exhausted", 1, policy),
    DesiredWorker("delta-defer", 1, policy),
    DesiredWorker("beta-start", 1, policy),
)
observed = (
    ObservedWorker("zeta-retire", 3, WorkerHealth.HEALTHY),
    ObservedWorker("gamma-replace", 1, WorkerHealth.HEALTHY, (70.0,)),
    ObservedWorker("alpha-retain", 2, WorkerHealth.HEALTHY),
    ObservedWorker("delta-defer", 1, WorkerHealth.UNHEALTHY, (95.0,)),
    ObservedWorker(
        "epsilon-exhausted",
        1,
        WorkerHealth.UNHEALTHY,
        (50.0, 90.0),
    ),
)

plan = plan_worker_replacements(
    desired,
    observed,
    observed_at=100.0,
    revision="rev-8",
)
assert tuple(action.outcome for action in plan.actions) == (
    WorkerOutcome.RETAIN,
    WorkerOutcome.START,
    WorkerOutcome.DEFER_COOLDOWN,
    WorkerOutcome.EXHAUSTED,
    WorkerOutcome.REPLACE,
    WorkerOutcome.RETIRE,
)
assert plan.actions[1].action_token == "rev-8:beta-start:start:1:-"
assert len(plan.actions) == len({worker.target_id for worker in desired + observed})
```

## Trade-offs and Limitations

Validation is a complete preflight: no action is constructed until both input
tuples, all unique IDs, generations, enum values, policies, finite times, and
strict histories have passed. Planning then sorts the union of IDs, so input
permutations produce the same action order. Test all six outcomes, duplicate
IDs, zero budget, exact window and cooldown boundaries, future or repeated
times, `NaN` and infinity, empty snapshots, and mismatched generations in both
directions.

The rolling budget counts timestamps in the closed interval
`[observed_at - window_seconds, observed_at]`; exhaustion takes precedence over
cooldown, and equality at the cooldown boundary permits replacement. These are
policy choices, not universal supervision semantics. Bounded full histories
also cost more memory than a validated rolling counter.

This function reads no clock and creates no thread, process, callback, log,
lease, ownership claim, spawn, join, or other live side effect. It cannot stop
two executors from racing, guarantee that a worker is replaced, or make a
stale observation current. The stable token helps only when an executor
atomically revalidates generation and revision and records that token before
performing the separately owned lifecycle operation.

## Related Snippets

<!-- catalog:related:start -->
- [Plan a Keyed Worker-Set Reload from a Coalesced SIGHUP Request](plan-a-keyed-worker-set-reload-from-a-coalesced-sighup-request.md)
- [Plan Priority Batches with an Age-Gated Tail](plan-priority-batches-with-an-age-gated-tail.md)
- [Classify Required Health Stamps by Freshness](../observability-operations/classify-required-health-stamps-by-freshness.md)
<!-- catalog:related:end -->
