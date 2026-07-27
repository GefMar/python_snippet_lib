---
title: "Bisect a Bracketed Monotone Async Metric Within an Evaluation Budget"
snippet_type: algorithm
use_cases:
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - match-strict-mutual-nearest-neighbors-with-a-comparison-budget.md
  - ../reliability-resilience/wait-for-a-predicate-until-a-monotonic-deadline.md
---

# Bisect a Bracketed Monotone Async Metric Within an Evaluation Budget

## Idea and Problem

Locate a zero of a non-decreasing metric when every probe is asynchronous and the caller must cap the total number of evaluations.

The two endpoints are evaluated first to prove that they bracket zero. Each
later probe halves the remaining interval, while the result records the best
observed residual, the final bracket, the exact call count, and why evaluation
stopped. Exhausting the budget is reported rather than disguised as
convergence.

## When to Use

Use this algorithm when a remote or otherwise asynchronous measurement is
deterministic enough to be treated as a finite, non-decreasing function of one
finite numeric input. The caller must already know a bracket whose lower
residual is non-positive and whose upper residual is non-negative. Subtract a
target value inside the evaluator so that the desired result is zero.

## Implementation

```python
import math
from collections.abc import Awaitable, Callable
from dataclasses import dataclass
from typing import Literal


StopReason = Literal["residual", "interval", "budget", "float-stagnation"]


@dataclass(frozen=True, slots=True)
class AsyncBisectionResult:
    best_probe: float
    best_residual: float
    lower: float
    upper: float
    evaluations: int
    converged: bool
    stop_reason: StopReason


async def bisect_monotone_async(
    evaluate: Callable[[float], Awaitable[float]],
    *,
    lower: float,
    upper: float,
    position_tolerance: float,
    residual_tolerance: float = 0.0,
    max_evaluations: int = 64,
) -> AsyncBisectionResult:
    numeric_inputs = (
        ("lower", lower),
        ("upper", upper),
        ("position_tolerance", position_tolerance),
        ("residual_tolerance", residual_tolerance),
    )
    normalized: dict[str, float] = {}
    for name, value in numeric_inputs:
        if not isinstance(value, (int, float)) or isinstance(value, bool):
            raise TypeError(f"{name} must be a real number")
        try:
            normalized[name] = float(value)
        except OverflowError as error:
            raise ValueError(f"{name} must be finite") from error
        if not math.isfinite(normalized[name]):
            raise ValueError(f"{name} must be finite")
    lower = normalized["lower"]
    upper = normalized["upper"]
    position_tolerance = normalized["position_tolerance"]
    residual_tolerance = normalized["residual_tolerance"]
    if lower >= upper:
        raise ValueError("lower must be less than upper")
    if not math.isfinite(upper - lower):
        raise ValueError("the bracket span must be finite")
    if position_tolerance <= 0.0:
        raise ValueError("position_tolerance must be positive")
    if residual_tolerance < 0.0:
        raise ValueError("residual_tolerance must be non-negative")
    if isinstance(max_evaluations, bool) or not isinstance(max_evaluations, int):
        raise TypeError("max_evaluations must be an integer")
    if not 2 <= max_evaluations <= 256:
        raise ValueError("max_evaluations must be between 2 and 256")

    async def checked_evaluation(probe: float) -> float:
        residual = await evaluate(probe)
        if not isinstance(residual, (int, float)) or isinstance(residual, bool):
            raise TypeError("the evaluator must return a real number")
        try:
            residual = float(residual)
        except OverflowError as error:
            raise ValueError("the evaluator returned a non-finite residual") from error
        if not math.isfinite(residual):
            raise ValueError("the evaluator returned a non-finite residual")
        return residual

    lower_residual = await checked_evaluation(lower)
    upper_residual = await checked_evaluation(upper)
    evaluations = 2
    if lower_residual > 0.0 or upper_residual < 0.0:
        raise ValueError("endpoint residuals do not bracket zero")

    if abs(lower_residual) <= abs(upper_residual):
        best_probe, best_residual = lower, lower_residual
    else:
        best_probe, best_residual = upper, upper_residual

    def result(reason: StopReason) -> AsyncBisectionResult:
        return AsyncBisectionResult(
            best_probe=best_probe,
            best_residual=best_residual,
            lower=lower,
            upper=upper,
            evaluations=evaluations,
            converged=reason in ("residual", "interval"),
            stop_reason=reason,
        )

    if abs(best_residual) <= residual_tolerance:
        return result("residual")
    if upper - lower <= position_tolerance:
        return result("interval")

    while evaluations < max_evaluations:
        width = upper - lower
        if width <= position_tolerance:
            return result("interval")

        midpoint = lower + width / 2.0
        if not math.isfinite(midpoint):
            midpoint = lower / 2.0 + upper / 2.0
        if midpoint <= lower or midpoint >= upper:
            return result("float-stagnation")

        midpoint_residual = await checked_evaluation(midpoint)
        evaluations += 1
        if abs(midpoint_residual) < abs(best_residual):
            best_probe, best_residual = midpoint, midpoint_residual
        if abs(midpoint_residual) <= residual_tolerance:
            return result("residual")

        if midpoint_residual < 0.0:
            lower, lower_residual = midpoint, midpoint_residual
        else:
            upper, upper_residual = midpoint, midpoint_residual
        if upper - lower <= position_tolerance:
            return result("interval")

    return result("budget")
```

## Example

```python
import asyncio


async def run_example() -> tuple[float, str, int, bool, int, str, bool]:
    async def shifted(value: float) -> float:
        await asyncio.sleep(0)
        return value - 0.375

    converged = await bisect_monotone_async(
        shifted,
        lower=0.0,
        upper=1.0,
        position_tolerance=1e-12,
        residual_tolerance=0.0,
        max_evaluations=8,
    )

    budget_limited = await bisect_monotone_async(
        lambda value: shifted(value + 0.075),
        lower=0.0,
        upper=1.0,
        position_tolerance=1e-12,
        max_evaluations=2,
    )
    return (
        converged.best_probe,
        converged.stop_reason,
        converged.evaluations,
        converged.converged,
        budget_limited.evaluations,
        budget_limited.stop_reason,
        budget_limited.converged,
    )


assert asyncio.run(run_example()) == (
    0.375,
    "residual",
    5,
    True,
    2,
    "budget",
    False,
)
```

## Trade-offs and Limitations

The method assumes monotonic, repeatable observations; noise, caching drift,
rate limits, or a moving target can invalidate its bracket and stopping
meaning. Evaluations are deliberately sequential, so the algorithm does not
hide latency through speculation. It provides no retries, averaging, timeout,
or side-effect rollback, and evaluator exceptions or cancellation propagate.
The memory cost is `O(1)`, while the number of calls is bounded explicitly.
Use a noise-aware optimizer or a service-specific search policy when probes
are stochastic or expensive side effects cannot safely be repeated.

## Related Snippets

<!-- catalog:related:start -->
- [Match Strict Mutual Nearest Neighbors with a Comparison Budget](match-strict-mutual-nearest-neighbors-with-a-comparison-budget.md)
- [Wait for a Predicate Until a Monotonic Deadline](../reliability-resilience/wait-for-a-predicate-until-a-monotonic-deadline.md)
<!-- catalog:related:end -->
