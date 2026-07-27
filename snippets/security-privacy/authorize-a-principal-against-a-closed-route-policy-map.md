---
title: "Authorize a Principal Against a Closed Route Policy Map"
snippet_type: recipe
use_cases:
  - security
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../python-language/build-a-read-only-mapping-with-canonical-text-keys.md
  - ../configuration-serialization/match-a-client-against-a-bounded-platform-availability-rule.md
---

# Authorize a Principal Against a Closed Route Policy Map

## Idea and Problem

Freeze explicit principal and route permissions once, then make every authorization decision fail closed over those validated sets.

Both sides of the decision must be named in the policy. A request is allowed
only when the known principal's grants intersect the known route's required
permissions; there is no implicit route, default permission, or wildcard.

## When to Use

Use this recipe after authentication has produced one stable principal name and
a small application has a closed set of named routes. It fits any-of ACLs where
one matching permission is sufficient. Use a policy engine when decisions need
resource attributes, deny rules, inheritance, or all-of expressions, and keep
credential verification outside this lookup.

## Implementation

```python
import re
from collections.abc import Collection, Mapping
from dataclasses import dataclass
from types import MappingProxyType


_NAME = re.compile(r"[a-z][a-z0-9_.-]{0,63}").fullmatch
_MAX_ENTRIES = 64
_MAX_PERMISSIONS = 16


@dataclass(frozen=True, slots=True)
class ClosedRoutePolicy:
    principal_grants: Mapping[str, frozenset[str]]
    route_requirements: Mapping[str, frozenset[str]]


def _valid_name(value: object) -> bool:
    return type(value) is str and _NAME(value) is not None


def _freeze_permissions(
    source: Mapping[str, Collection[str]],
    *,
    kind: str,
) -> Mapping[str, frozenset[str]]:
    if not isinstance(source, Mapping):
        raise TypeError(f"{kind} policy must be a mapping")
    if not 1 <= len(source) <= _MAX_ENTRIES:
        raise ValueError(f"{kind} policy size is outside the supported range")

    frozen: dict[str, frozenset[str]] = {}
    for name, supplied in source.items():
        if not _valid_name(name):
            raise ValueError(f"{kind} names must use the supported syntax")
        if isinstance(supplied, (str, bytes)) or not isinstance(supplied, Collection):
            raise TypeError("permissions must be finite collections")

        permissions = tuple(supplied)
        if not 1 <= len(permissions) <= _MAX_PERMISSIONS:
            raise ValueError("permission count is outside the supported range")
        if any(not _valid_name(permission) for permission in permissions):
            raise ValueError("permissions must use the supported syntax")
        if len(set(permissions)) != len(permissions):
            raise ValueError("permissions must not contain duplicates")
        frozen[name] = frozenset(permissions)

    return MappingProxyType(frozen)


def build_closed_route_policy(
    principal_grants: Mapping[str, Collection[str]],
    route_requirements: Mapping[str, Collection[str]],
) -> ClosedRoutePolicy:
    principals = _freeze_permissions(principal_grants, kind="principal")
    routes = _freeze_permissions(route_requirements, kind="route")
    return ClosedRoutePolicy(principals, routes)


def is_route_allowed(
    policy: ClosedRoutePolicy,
    *,
    principal: object,
    route: object,
) -> bool:
    if not isinstance(policy, ClosedRoutePolicy):
        raise TypeError("policy must be a ClosedRoutePolicy")
    if not _valid_name(principal) or not _valid_name(route):
        return False

    grants = policy.principal_grants.get(principal)
    requirements = policy.route_requirements.get(route)
    return (
        grants is not None
        and requirements is not None
        and not grants.isdisjoint(requirements)
    )
```

## Example

```python
principal_source = {"reader": {"report.read"}}
route_source = {"report.view": {"report.read", "report.audit"}}
policy = build_closed_route_policy(principal_source, route_source)

principal_source["reader"].clear()
route_source["report.view"].clear()

allowed = is_route_allowed(policy, principal="reader", route="report.view")
unknown_denied = not is_route_allowed(
    policy,
    principal="reader",
    route="report.remove",
)

assert allowed and unknown_denied
```

## Trade-offs and Limitations

The policy uses fixed any-of semantics and materializes every permission set.
It cannot express an explicit deny, require several permissions together, or
consider resource attributes. Freezing protects the decision from later
mutation of the supplied containers, but policy replacement and revocation
remain the caller's responsibility. The function assumes that authentication
already established the principal name; it does not parse credentials, verify
signatures, normalize URLs, conceal timing differences, or produce an audit
record.

## Related Snippets

<!-- catalog:related:start -->
- [Build a Read-Only Mapping with Canonical Text Keys](../python-language/build-a-read-only-mapping-with-canonical-text-keys.md)
- [Match a Client Against a Bounded Platform Availability Rule](../configuration-serialization/match-a-client-against-a-bounded-platform-availability-rule.md)
<!-- catalog:related:end -->
