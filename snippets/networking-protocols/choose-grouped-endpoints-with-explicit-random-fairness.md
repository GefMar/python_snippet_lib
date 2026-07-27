---
title: "Choose Grouped Endpoints with Explicit Random Fairness"
snippet_type: algorithm
use_cases:
  - networking
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../algorithms-data-structures/map-keys-with-an-immutable-consistent-hash-ring.md
  - ../machine-learning-statistics/sample-weighted-negative-items-outside-explicit-user-histories.md
  - ../reliability-resilience/assign-stable-schedule-slots-with-a-digest.md
---

# Choose Grouped Endpoints with Explicit Random Fairness

## Idea and Problem

Choose one endpoint from a frozen grouped snapshot while making endpoint-uniform and nonempty-group-uniform randomness separate explicit policies.

Flattening every endpoint gives each endpoint the same probability. Choosing a
group first gives each nonempty group the same probability, so endpoints in a
small group receive more probability than endpoints in a large group. Naming
the policy prevents that weighting decision from being hidden in iteration
code.

## When to Use

Use this algorithm after another component has produced a bounded, immutable
snapshot whose endpoint identifiers are already validated and equally healthy.
It fits replayable planning or tests that need either equal treatment of every
endpoint or equal treatment of every nonempty group.

Use consistent hashing when choices must remain sticky across membership
changes, weighted sampling when endpoints have explicit capacities, and a
real load balancer when health, latency, retries, or live utilization affect
the decision.

## Implementation

```python
from dataclasses import dataclass
from random import Random


_MAX_GROUPS = 256
_MAX_ENDPOINTS = 10_000
_MAX_IDENTIFIER_BYTES = 256
_MAX_SEED = (1 << 63) - 1
_POLICIES = frozenset({"endpoint-uniform", "group-uniform"})


def _endpoint_identifier(value: object, *, field: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if not value or len(value) > _MAX_IDENTIFIER_BYTES:
        raise ValueError(f"{field} length is outside the supported range")
    if (
        not value.isascii()
        or not value.isprintable()
        or value != value.strip()
        or any(character.isspace() for character in value)
    ):
        raise ValueError(f"{field} must be trimmed printable ASCII without spaces")
    return value


@dataclass(frozen=True, slots=True)
class EndpointGroup:
    group_id: str
    endpoint_ids: tuple[str, ...]


@dataclass(frozen=True, slots=True)
class EndpointChoice:
    group_id: str
    endpoint_id: str
    policy: str


def choose_grouped_endpoint(
    groups: tuple[EndpointGroup, ...],
    *,
    policy: str,
    seed: int,
) -> EndpointChoice:
    if type(groups) is not tuple:
        raise TypeError("groups must be an exact tuple")
    if not 1 <= len(groups) <= _MAX_GROUPS:
        raise ValueError("group count is outside the supported range")
    if type(policy) is not str:
        raise TypeError("policy must be an exact string")
    if policy not in _POLICIES:
        raise ValueError("policy is not supported")
    if type(seed) is not int:
        raise TypeError("seed must be an exact integer")
    if not 0 <= seed <= _MAX_SEED:
        raise ValueError("seed is outside the supported range")

    seen_groups: set[str] = set()
    seen_endpoints: set[str] = set()
    nonempty_groups: list[tuple[str, tuple[str, ...]]] = []
    flattened: list[tuple[str, str]] = []
    endpoint_count = 0

    for group in groups:
        if type(group) is not EndpointGroup:
            raise TypeError("groups must contain exact EndpointGroup values")
        group_id = _endpoint_identifier(group.group_id, field="group_id")
        if group_id in seen_groups:
            raise ValueError("group identifiers must be unique")
        seen_groups.add(group_id)

        if type(group.endpoint_ids) is not tuple:
            raise TypeError("endpoint_ids must be an exact tuple")
        endpoint_count += len(group.endpoint_ids)
        if endpoint_count > _MAX_ENDPOINTS:
            raise ValueError("endpoint count exceeds the supported limit")

        validated_endpoints: list[str] = []
        for raw_endpoint_id in group.endpoint_ids:
            endpoint_id = _endpoint_identifier(
                raw_endpoint_id,
                field="endpoint_id",
            )
            if endpoint_id in seen_endpoints:
                raise ValueError("endpoint identifiers must be globally unique")
            seen_endpoints.add(endpoint_id)
            validated_endpoints.append(endpoint_id)
            flattened.append((group_id, endpoint_id))

        if validated_endpoints:
            nonempty_groups.append((group_id, tuple(validated_endpoints)))

    if not flattened:
        raise ValueError("at least one endpoint is required")

    rng = Random(seed)
    if policy == "endpoint-uniform":
        group_id, endpoint_id = flattened[rng.randrange(len(flattened))]
    else:
        group_id, endpoint_ids = nonempty_groups[
            rng.randrange(len(nonempty_groups))
        ]
        endpoint_id = endpoint_ids[rng.randrange(len(endpoint_ids))]

    return EndpointChoice(group_id, endpoint_id, policy)
```

## Example

```python
groups = (
    EndpointGroup("central", ("api-a", "api-b", "api-c")),
    EndpointGroup("edge", ("api-d",)),
    EndpointGroup("spare", ()),
)
snapshot = tuple((group.group_id, group.endpoint_ids) for group in groups)

per_endpoint = choose_grouped_endpoint(
    groups,
    policy="endpoint-uniform",
    seed=1,
)
per_group = choose_grouped_endpoint(
    groups,
    policy="group-uniform",
    seed=1,
)
replayed = choose_grouped_endpoint(
    groups,
    policy="endpoint-uniform",
    seed=1,
)

try:
    choose_grouped_endpoint(
        (
            EndpointGroup("first", ("same",)),
            EndpointGroup("second", ("same",)),
        ),
        policy="endpoint-uniform",
        seed=1,
    )
except ValueError:
    duplicate_rejected = True
else:
    duplicate_rejected = False

assert (
    per_endpoint == EndpointChoice("central", "api-b", "endpoint-uniform")
    and per_group == EndpointChoice("central", "api-c", "group-uniform")
    and replayed == per_endpoint
    and tuple((group.group_id, group.endpoint_ids) for group in groups) == snapshot
    and duplicate_rejected
)
```

## Trade-offs and Limitations

Validation and endpoint-uniform materialization cost `O(groups + endpoints)`
time and `O(endpoints)` additional memory. Global endpoint uniqueness prevents
accidental weighting through duplicate identifiers, while empty groups are
ignored by both policies. Group-uniform selection intentionally gives an
endpoint in a smaller group more probability than one in a larger group.

The local seed isolates this function from module-level random state and makes
replay possible on the tested Python runtime. Exact draws are not a
cross-version serialization contract, and this generator is not suitable for
security decisions. The function treats identifiers as opaque text: it does
not discover services, parse addresses, check health, perform DNS, retry, open
connections, or observe live load.

## Related Snippets

<!-- catalog:related:start -->
- [Map Keys with an Immutable Consistent Hash Ring](../algorithms-data-structures/map-keys-with-an-immutable-consistent-hash-ring.md)
- [Sample Weighted Negative Items Outside Explicit User Histories](../machine-learning-statistics/sample-weighted-negative-items-outside-explicit-user-histories.md)
- [Assign Stable Schedule Slots with a Digest](../reliability-resilience/assign-stable-schedule-slots-with-a-digest.md)
<!-- catalog:related:end -->
