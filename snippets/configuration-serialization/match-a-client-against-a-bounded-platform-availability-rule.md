---
title: "Match a Client Against a Bounded Platform Availability Rule"
snippet_type: recipe
use_cases:
  - configuration
  - interoperability
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - evaluate-a-bounded-boolean-tag-expression-with-an-ast-allowlist.md
  - ../algorithms-data-structures/sort-dotted-release-labels-with-an-explicit-last-marker.md
---

# Match a Client Against a Bounded Platform Availability Rule

## Idea and Problem

Evaluate one immutable client description against bounded data-only platform, version, authentication, and tag constraints while returning ordered blocking reasons.

Empty platform, manufacturer, or model sets mean unrestricted. A non-empty set
requires an exact canonical match, so a missing constrained client field fails.
The application-version interval is `[minimum, maximum)`: the lower bound is
inclusive and the upper bound is exclusive.

## When to Use

Use this recipe for local feature or content targeting whose complete rule is
small, trusted, and available as structured data. Ordered reason codes are
useful when callers need a stable explanation without logging raw client
values.

Use a reviewed policy engine when rules require expressions, delegation, or
independent administration. Client-reported platform fields and tags are
targeting hints, not proof of identity or authorization.

## Implementation

```python
import re
from dataclasses import dataclass


_TOKEN = re.compile(r"[a-z][a-z0-9-]{0,63}", re.ASCII)
_MAX_SELECTORS = 32
_MAX_VERSION_COMPONENTS = 8
_MAX_VERSION_DIGITS = 6
_MAX_VERSION_TEXT_LENGTH = (
    _MAX_VERSION_COMPONENTS * _MAX_VERSION_DIGITS
    + _MAX_VERSION_COMPONENTS
    - 1
)
_MAX_VERSION_VALUE = 999_999


def _canonical_token(value: object, *, name: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{name} must be exact text")
    if _TOKEN.fullmatch(value) is None:
        raise ValueError(f"{name} must be a canonical lowercase identifier")
    return value


def _token_set(value: object, *, name: str) -> frozenset[str]:
    if type(value) is not frozenset:
        raise TypeError(f"{name} must be a frozenset")
    if len(value) > _MAX_SELECTORS:
        raise ValueError(f"{name} has too many values")
    for item in value:
        _canonical_token(item, name=name)
    return value


@dataclass(frozen=True, order=True, slots=True)
class NumericVersion:
    parts: tuple[int, ...]

    def __post_init__(self) -> None:
        if type(self.parts) is not tuple:
            raise TypeError("version parts must be a tuple")
        if not 1 <= len(self.parts) <= _MAX_VERSION_COMPONENTS:
            raise ValueError("version component count is outside the limit")
        for part in self.parts:
            if type(part) is not int:
                raise TypeError("version components must be exact integers")
            if not 0 <= part <= _MAX_VERSION_VALUE:
                raise ValueError("a version component is outside the limit")

        normalized = list(self.parts)
        while len(normalized) > 1 and normalized[-1] == 0:
            normalized.pop()
        object.__setattr__(self, "parts", tuple(normalized))


def parse_numeric_version(text: str) -> NumericVersion:
    if type(text) is not str:
        raise TypeError("version must be exact text")
    if not 1 <= len(text) <= _MAX_VERSION_TEXT_LENGTH or not text.isascii():
        raise ValueError("version must be bounded non-empty ASCII text")
    components = text.split(".")
    if not 1 <= len(components) <= _MAX_VERSION_COMPONENTS:
        raise ValueError("version component count is outside the limit")

    values = []
    for component in components:
        if (
            not component
            or not component.isascii()
            or not component.isdecimal()
            or len(component) > _MAX_VERSION_DIGITS
            or (len(component) > 1 and component.startswith("0"))
        ):
            raise ValueError("version components must be canonical ASCII decimals")
        value = int(component)
        if value > _MAX_VERSION_VALUE:
            raise ValueError("a version component is outside the limit")
        values.append(value)
    return NumericVersion(tuple(values))


@dataclass(frozen=True, slots=True)
class ClientContext:
    platform: str
    app_version: NumericVersion | None = None
    manufacturer: str | None = None
    model: str | None = None
    tags: frozenset[str] = frozenset()
    authenticated: bool = False

    def __post_init__(self) -> None:
        _canonical_token(self.platform, name="platform")
        if self.app_version is not None and type(self.app_version) is not NumericVersion:
            raise TypeError("app_version must be a NumericVersion or None")
        for name, value in (
            ("manufacturer", self.manufacturer),
            ("model", self.model),
        ):
            if value is not None:
                _canonical_token(value, name=name)
        _token_set(self.tags, name="tags")
        if type(self.authenticated) is not bool:
            raise TypeError("authenticated must be a bool")


@dataclass(frozen=True, slots=True)
class AvailabilityRule:
    enabled: bool = True
    requires_authentication: bool = False
    platforms: frozenset[str] = frozenset()
    manufacturers: frozenset[str] = frozenset()
    models: frozenset[str] = frozenset()
    required_tags: frozenset[str] = frozenset()
    forbidden_tags: frozenset[str] = frozenset()
    minimum_version: NumericVersion | None = None
    maximum_version: NumericVersion | None = None

    def __post_init__(self) -> None:
        if type(self.enabled) is not bool:
            raise TypeError("enabled must be a bool")
        if type(self.requires_authentication) is not bool:
            raise TypeError("requires_authentication must be a bool")
        for name, value in (
            ("platforms", self.platforms),
            ("manufacturers", self.manufacturers),
            ("models", self.models),
            ("required_tags", self.required_tags),
            ("forbidden_tags", self.forbidden_tags),
        ):
            _token_set(value, name=name)
        if self.required_tags & self.forbidden_tags:
            raise ValueError("required and forbidden tags must not overlap")
        for name, value in (
            ("minimum_version", self.minimum_version),
            ("maximum_version", self.maximum_version),
        ):
            if value is not None and type(value) is not NumericVersion:
                raise TypeError(f"{name} must be a NumericVersion or None")
        if (
            self.minimum_version is not None
            and self.maximum_version is not None
            and self.minimum_version >= self.maximum_version
        ):
            raise ValueError("minimum_version must be below maximum_version")


@dataclass(frozen=True, slots=True)
class AvailabilityDecision:
    available: bool
    reasons: tuple[str, ...]


def match_client_availability(
    client: ClientContext,
    rule: AvailabilityRule,
) -> AvailabilityDecision:
    if type(client) is not ClientContext or type(rule) is not AvailabilityRule:
        raise TypeError("client and rule must use the declared immutable types")

    reasons = []
    if not rule.enabled:
        reasons.append("disabled")
    if rule.requires_authentication and not client.authenticated:
        reasons.append("authentication_required")
    if rule.platforms and client.platform not in rule.platforms:
        reasons.append("platform_not_allowed")
    if rule.manufacturers and client.manufacturer not in rule.manufacturers:
        reasons.append("manufacturer_not_allowed")
    if rule.models and client.model not in rule.models:
        reasons.append("model_not_allowed")

    if rule.minimum_version is not None or rule.maximum_version is not None:
        if client.app_version is None:
            reasons.append("version_missing")
        else:
            if (
                rule.minimum_version is not None
                and client.app_version < rule.minimum_version
            ):
                reasons.append("version_below_minimum")
            if (
                rule.maximum_version is not None
                and client.app_version >= rule.maximum_version
            ):
                reasons.append("version_at_or_above_maximum")

    if not rule.required_tags.issubset(client.tags):
        reasons.append("required_tags_missing")
    if rule.forbidden_tags & client.tags:
        reasons.append("forbidden_tag_present")
    return AvailabilityDecision(not reasons, tuple(reasons))
```

## Example

```python
rule = AvailabilityRule(
    requires_authentication=True,
    platforms=frozenset({"tablet-os"}),
    manufacturers=frozenset({"vendor-a"}),
    required_tags=frozenset({"preview"}),
    forbidden_tags=frozenset({"suspended"}),
    minimum_version=parse_numeric_version("2.9"),
    maximum_version=parse_numeric_version("3.0"),
)
accepted = match_client_availability(
    ClientContext(
        platform="tablet-os",
        manufacturer="vendor-a",
        app_version=parse_numeric_version("2.10"),
        tags=frozenset({"preview"}),
        authenticated=True,
    ),
    rule,
)
blocked = match_client_availability(
    ClientContext(
        platform="tablet-os",
        manufacturer="vendor-a",
        app_version=parse_numeric_version("3"),
        tags=frozenset({"preview", "suspended"}),
        authenticated=True,
    ),
    rule,
)

try:
    parse_numeric_version("2.bad..1")
except ValueError:
    malformed_rejected = True
else:
    malformed_rejected = False

assert (
    accepted,
    blocked.reasons,
    parse_numeric_version("2") == parse_numeric_version("2.0.0"),
    malformed_rejected,
) == (
    AvailabilityDecision(True, ()),
    ("version_at_or_above_maximum", "forbidden_tag_present"),
    True,
    True,
)
```

## Trade-offs and Limitations

Rule and client construction validate bounded sets and version components once;
evaluation then uses bounded set membership and tuple comparison. Ordered
reason codes intentionally reveal only which constraint failed, not client
values. Multiple reasons may be returned even after an earlier veto.

Numeric dotted tuples are not SemVer, PEP 440, or a platform-specific version
scheme: prereleases, epochs, metadata, and wildcards are rejected. Selectors
are exact and case-sensitive. The helper performs no expression evaluation,
rollout assignment, clock checks, remote flag delivery, authorization, or
proof that client-supplied attributes are truthful.

## Related Snippets

<!-- catalog:related:start -->
- [Evaluate a Bounded Boolean Tag Expression with an AST Allowlist](evaluate-a-bounded-boolean-tag-expression-with-an-ast-allowlist.md)
- [Sort Dotted Release Labels with an Explicit Last Marker](../algorithms-data-structures/sort-dotted-release-labels-with-an-explicit-last-marker.md)
<!-- catalog:related:end -->
