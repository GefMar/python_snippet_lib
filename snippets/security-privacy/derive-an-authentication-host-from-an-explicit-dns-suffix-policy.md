---
title: "Derive an Authentication Host from an Explicit DNS Suffix Policy"
snippet_type: recipe
use_cases:
  - security
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../networking-protocols/build-a-canonical-http-origin-key.md
---

# Derive an Authentication Host from an Explicit DNS Suffix Policy

## Idea and Problem

Derive a companion authentication hostname by replacing one validated source suffix with the longest supported suffix of a validated target hostname.

The policy is immutable and explicit: supported suffixes are data supplied by
the caller, and a subset can disable authentication-host derivation. Malformed
input and a source outside the policy are configuration errors, while an
unsupported or disabled target produces a distinct immutable refusal result.

## When to Use

Use this recipe when one service prefix, such as `login`, is reused across a
small, reviewed set of regional DNS suffixes. Both inputs must already be
lowercase ASCII DNS hostnames without a scheme, port, user information,
brackets, or trailing root dot. Every supported match must leave at least one
label before the suffix.

Do not use this helper to parse URLs, discover public suffixes, decide cookie
scope, authorize requests, or prove that two hostnames have the same owner.

## Implementation

```python
import re
from dataclasses import dataclass
from ipaddress import ip_address


_MAX_HOST_LENGTH = 253
_MAX_SUFFIXES = 64
_DNS_LABEL = re.compile(
    r"[a-z0-9](?:[a-z0-9-]{0,61}[a-z0-9])?",
    re.ASCII,
)
_NUMERIC_LABEL = re.compile(r"[0-9]+", re.ASCII)


def _dns_hostname(value: object, *, field: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be exact text")
    if not 1 <= len(value) <= _MAX_HOST_LENGTH:
        raise ValueError(f"{field} length is outside the supported range")
    if not value.isascii() or value != value.lower():
        raise ValueError(f"{field} must be lowercase ASCII")
    if value.endswith("."):
        raise ValueError(f"{field} must not contain a trailing root dot")
    if any(marker in value for marker in (":", "/", "@", "[", "]")):
        raise ValueError(f"{field} must be a DNS hostname, not an authority or URL")

    labels = value.split(".")
    if any(not label for label in labels):
        raise ValueError(f"{field} must not contain empty labels")
    if any(len(label) > 63 or _DNS_LABEL.fullmatch(label) is None for label in labels):
        raise ValueError(f"{field} contains an invalid DNS label")

    try:
        ip_address(value)
    except ValueError:
        pass
    else:
        raise ValueError(f"{field} must not be an IP address")
    if all(_NUMERIC_LABEL.fullmatch(label) is not None for label in labels):
        raise ValueError(f"{field} must not be an ambiguous numeric address")
    return value


@dataclass(frozen=True, slots=True)
class DnsSuffixPolicy:
    supported_suffixes: tuple[str, ...]
    disabled_suffixes: frozenset[str] = frozenset()

    def __post_init__(self) -> None:
        if type(self.supported_suffixes) is not tuple:
            raise TypeError("supported_suffixes must be an exact tuple")
        if not 1 <= len(self.supported_suffixes) <= _MAX_SUFFIXES:
            raise ValueError("supported suffix count is outside the supported range")

        seen: set[str] = set()
        for raw_suffix in self.supported_suffixes:
            suffix = _dns_hostname(raw_suffix, field="supported suffix")
            if suffix in seen:
                raise ValueError("supported suffixes must be unique")
            seen.add(suffix)

        if type(self.disabled_suffixes) is not frozenset:
            raise TypeError("disabled_suffixes must be an exact frozenset")
        for suffix in self.disabled_suffixes:
            _dns_hostname(suffix, field="disabled suffix")
        if not self.disabled_suffixes.issubset(seen):
            raise ValueError("disabled suffixes must be supported suffixes")


@dataclass(frozen=True, slots=True)
class DerivedAuthenticationHost:
    host: str


@dataclass(frozen=True, slots=True)
class UnsupportedAuthenticationTarget:
    target_host: str


@dataclass(frozen=True, slots=True)
class DisabledAuthenticationTarget:
    target_host: str
    suffix: str


AuthenticationHostDecision = (
    DerivedAuthenticationHost
    | UnsupportedAuthenticationTarget
    | DisabledAuthenticationTarget
)


def _longest_matching_suffix(
    host: str,
    supported_suffixes: tuple[str, ...],
) -> str | None:
    matches = [
        suffix
        for suffix in supported_suffixes
        if host == suffix or host.endswith("." + suffix)
    ]
    if not matches:
        return None
    return max(matches, key=lambda suffix: suffix.count("."))


def _prefix_before_suffix(host: str, suffix: str, *, field: str) -> str:
    if host == suffix:
        raise ValueError(f"{field} must leave a non-empty prefix before its suffix")
    return host[: -(len(suffix) + 1)]


def derive_authentication_host(
    source_host: str,
    target_host: str,
    policy: DnsSuffixPolicy,
) -> AuthenticationHostDecision:
    if type(policy) is not DnsSuffixPolicy:
        raise TypeError("policy must be a DnsSuffixPolicy")
    source = _dns_hostname(source_host, field="source_host")
    target = _dns_hostname(target_host, field="target_host")

    source_suffix = _longest_matching_suffix(source, policy.supported_suffixes)
    if source_suffix is None:
        raise ValueError("source_host does not match a supported suffix")
    source_prefix = _prefix_before_suffix(
        source,
        source_suffix,
        field="source_host",
    )

    target_suffix = _longest_matching_suffix(target, policy.supported_suffixes)
    if target_suffix is None:
        return UnsupportedAuthenticationTarget(target)
    _prefix_before_suffix(target, target_suffix, field="target_host")
    if target_suffix in policy.disabled_suffixes:
        return DisabledAuthenticationTarget(target, target_suffix)

    derived = _dns_hostname(
        f"{source_prefix}.{target_suffix}",
        field="derived authentication host",
    )
    return DerivedAuthenticationHost(derived)
```

## Example

```python
policy = DnsSuffixPolicy(
    supported_suffixes=("example", "region.example", "disabled.example"),
    disabled_suffixes=frozenset({"disabled.example"}),
)

derived = derive_authentication_host(
    "login.example",
    "api.region.example",
    policy,
)
disabled = derive_authentication_host(
    "login.example",
    "api.disabled.example",
    policy,
)
unsupported = derive_authentication_host(
    "login.example",
    "api.other",
    policy,
)

invalid_hosts = (
    "Service.example",
    "https://service.example",
    "service.example:443",
    "reader@service.example",
    "[2001:db8::1]",
    "service.example.",
    "service..example",
    "192.0.2.1",
)
rejected = []
for host in invalid_hosts:
    try:
        derive_authentication_host(host, "api.region.example", policy)
    except ValueError:
        rejected.append(host)

try:
    derive_authentication_host("login.other", "api.region.example", policy)
except ValueError:
    source_mismatch_rejected = True
else:
    source_mismatch_rejected = False

try:
    derive_authentication_host("login.example", "region.example", policy)
except ValueError:
    empty_target_prefix_rejected = True
else:
    empty_target_prefix_rejected = False

assert (
    derived,
    disabled,
    unsupported,
    tuple(rejected),
    source_mismatch_rejected,
    empty_target_prefix_rejected,
) == (
    DerivedAuthenticationHost("login.region.example"),
    DisabledAuthenticationTarget("api.disabled.example", "disabled.example"),
    UnsupportedAuthenticationTarget("api.other"),
    invalid_hosts,
    True,
    True,
)
```

## Trade-offs and Limitations

Validation and matching cost `O(s * l)` for at most 64 suffixes and the bounded
hostname labels. Overlapping suffixes are deliberate: the longest matching
label sequence wins. A policy update can therefore change a decision even
when both hostnames remain unchanged.

The accepted names are narrower than general DNS input. Uppercase text,
Unicode and IDNA names, address literals, trailing dots, and URL syntax are
rejected rather than normalized. The configured suffixes are not a Public
Suffix List, and the result makes no claim about DNS resolution, cookies,
authentication, authorization, TLS identity, or service ownership.

## Related Snippets

<!-- catalog:related:start -->
- [Build a Canonical HTTP Origin Key](../networking-protocols/build-a-canonical-http-origin-key.md)
<!-- catalog:related:end -->
