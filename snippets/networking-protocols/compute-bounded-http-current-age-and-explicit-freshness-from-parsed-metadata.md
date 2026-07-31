---
title: "Compute Bounded HTTP Current Age and Explicit Freshness from Parsed Metadata"
snippet_type: recipe
use_cases:
  - interoperability
  - networking
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - parse-a-bounded-response-cache-control-field-under-a-closed-profile.md
  - apply-an-http-patch-only-when-a-strong-etag-still-matches.md
  - build-a-canonical-http-origin-key.md
---

# Compute Bounded HTTP Current Age and Explicit Freshness from Parsed Metadata

## Idea and Problem

Compute the HTTP current-age equation and one explicit freshness lifetime from already parsed response metadata while preserving every intermediate value for inspection.

The age calculation adds response delay to a received `Age` value, compares
that corrected value with apparent age, and then adds resident time. Freshness
lifetime follows a closed priority order: `s-maxage` for a shared cache, then
`max-age`, then `Expires - Date`. An absent explicit lifetime produces an
indeterminate freshness result instead of a guessed heuristic.

## When to Use

Use this recipe after a strict HTTP parser has converted the relevant dates and
delta-seconds into bounded integer seconds. It is useful in a small cache
adapter, a conformance fixture, or a diagnostic path that needs to show exactly
how a response age and explicit lifetime were derived.

Keep cacheability, request directives, validators, warning generation, stale
serving, clock acquisition, and field parsing outside this function. Those
decisions need more message context than this numerical kernel accepts.

## Implementation

```python
from dataclasses import dataclass
from enum import StrEnum

_MAX_TIMESTAMP = (1 << 63) - 1
_MAX_DELTA_SECONDS = (1 << 31) - 1


class FreshnessSource(StrEnum):
    S_MAXAGE = "s-maxage"
    MAX_AGE = "max-age"
    EXPIRES = "expires"
    NONE = "none"


@dataclass(frozen=True, slots=True)
class HttpAgeAndFreshness:
    apparent_age: int
    corrected_age_value: int
    response_delay: int
    corrected_initial_age: int
    resident_time: int
    current_age: int
    freshness_lifetime: int | None
    freshness_source: FreshnessSource
    is_fresh: bool | None


def _require_bounded_integer(
    name: str,
    value: object,
    maximum: int,
) -> int:
    if type(value) is not int:
        raise TypeError(f"{name} must be an exact integer")
    if not 0 <= value <= maximum:
        raise ValueError(f"{name} is outside the supported range")
    return value


def _require_optional_delta(name: str, value: object) -> int | None:
    if value is None:
        return None
    return _require_bounded_integer(name, value, _MAX_DELTA_SECONDS)


def compute_http_age_and_freshness(
    *,
    request_time: int,
    response_time: int,
    now: int,
    date_value: int,
    age_value: int,
    expires_value: int | None = None,
    max_age: int | None = None,
    s_maxage: int | None = None,
    shared: bool,
) -> HttpAgeAndFreshness:
    """Return current age and explicit freshness under a bounded profile."""
    request_time = _require_bounded_integer(
        "request_time",
        request_time,
        _MAX_TIMESTAMP,
    )
    response_time = _require_bounded_integer(
        "response_time",
        response_time,
        _MAX_TIMESTAMP,
    )
    now = _require_bounded_integer("now", now, _MAX_TIMESTAMP)
    date_value = _require_bounded_integer(
        "date_value",
        date_value,
        _MAX_TIMESTAMP,
    )
    age_value = _require_bounded_integer(
        "age_value",
        age_value,
        _MAX_DELTA_SECONDS,
    )
    if expires_value is not None:
        expires_value = _require_bounded_integer(
            "expires_value",
            expires_value,
            _MAX_TIMESTAMP,
        )
    max_age = _require_optional_delta("max_age", max_age)
    s_maxage = _require_optional_delta("s_maxage", s_maxage)
    if type(shared) is not bool:
        raise TypeError("shared must be an exact boolean")
    if not request_time <= response_time <= now:
        raise ValueError("times must satisfy request_time <= response_time <= now")

    response_delay = response_time - request_time
    apparent_age = max(0, response_time - date_value)
    corrected_age_value = age_value + response_delay
    corrected_initial_age = max(apparent_age, corrected_age_value)
    resident_time = now - response_time
    current_age = corrected_initial_age + resident_time

    if shared and s_maxage is not None:
        freshness_lifetime = s_maxage
        freshness_source = FreshnessSource.S_MAXAGE
    elif max_age is not None:
        freshness_lifetime = max_age
        freshness_source = FreshnessSource.MAX_AGE
    elif expires_value is not None:
        freshness_lifetime = expires_value - date_value
        freshness_source = FreshnessSource.EXPIRES
    else:
        freshness_lifetime = None
        freshness_source = FreshnessSource.NONE

    is_fresh = None if freshness_lifetime is None else freshness_lifetime > current_age
    return HttpAgeAndFreshness(
        apparent_age=apparent_age,
        corrected_age_value=corrected_age_value,
        response_delay=response_delay,
        corrected_initial_age=corrected_initial_age,
        resident_time=resident_time,
        current_age=current_age,
        freshness_lifetime=freshness_lifetime,
        freshness_source=freshness_source,
        is_fresh=is_fresh,
    )
```

## Example

```python
shared_result = compute_http_age_and_freshness(
    request_time=1_000,
    response_time=1_002,
    now=1_010,
    date_value=990,
    age_value=3,
    expires_value=1_040,
    max_age=30,
    s_maxage=5,
    shared=True,
)
private_result = compute_http_age_and_freshness(
    request_time=1_000,
    response_time=1_002,
    now=1_010,
    date_value=990,
    age_value=3,
    expires_value=1_040,
    max_age=30,
    s_maxage=5,
    shared=False,
)
expired_result = compute_http_age_and_freshness(
    request_time=100,
    response_time=100,
    now=100,
    date_value=200,
    age_value=0,
    expires_value=199,
    shared=False,
)
shifted_result = compute_http_age_and_freshness(
    request_time=2_000,
    response_time=2_002,
    now=2_010,
    date_value=1_990,
    age_value=3,
    expires_value=2_040,
    max_age=30,
    s_maxage=5,
    shared=True,
)

assert (shared_result.current_age, shared_result.freshness_source, shared_result.is_fresh) == (
    20,
    FreshnessSource.S_MAXAGE,
    False,
)
assert (private_result.freshness_lifetime, private_result.is_fresh) == (30, True)
assert (
    expired_result.freshness_lifetime,
    expired_result.freshness_source,
    expired_result.is_fresh,
) == (-1, FreshnessSource.EXPIRES, False)
assert shifted_result == shared_result

maximum_result = compute_http_age_and_freshness(
    request_time=_MAX_TIMESTAMP,
    response_time=_MAX_TIMESTAMP,
    now=_MAX_TIMESTAMP,
    date_value=0,
    age_value=_MAX_DELTA_SECONDS,
    max_age=_MAX_DELTA_SECONDS,
    shared=False,
)
assert maximum_result.current_age == _MAX_TIMESTAMP
assert maximum_result.is_fresh is False

equality_result = compute_http_age_and_freshness(
    request_time=0,
    response_time=0,
    now=0,
    date_value=0,
    age_value=0,
    max_age=0,
    shared=False,
)
unknown_result = compute_http_age_and_freshness(
    request_time=0,
    response_time=0,
    now=0,
    date_value=0,
    age_value=0,
    shared=False,
)
assert equality_result.is_fresh is False
assert (
    unknown_result.freshness_lifetime,
    unknown_result.freshness_source,
    unknown_result.is_fresh,
) == (None, FreshnessSource.NONE, None)


def rejects_http_snapshot(**changes: object) -> bool:
    arguments: dict[str, object] = {
        "request_time": 1,
        "response_time": 2,
        "now": 3,
        "date_value": 1,
        "age_value": 0,
        "shared": False,
    }
    arguments.update(changes)
    try:
        compute_http_age_and_freshness(**arguments)  # type: ignore[arg-type]
    except (TypeError, ValueError):
        return True
    return False


assert all(
    (
        rejects_http_snapshot(request_time=True),
        rejects_http_snapshot(age_value=-1),
        rejects_http_snapshot(max_age=_MAX_DELTA_SECONDS + 1),
        rejects_http_snapshot(shared=0),
        rejects_http_snapshot(request_time=3),
    )
)

source_rank = {
    FreshnessSource.S_MAXAGE: 1,
    FreshnessSource.MAX_AGE: 2,
    FreshnessSource.EXPIRES: 3,
    FreshnessSource.NONE: 4,
}
sources_seen: set[FreshnessSource] = set()
checked_cases = 0
signature = 0

for request_time in range(3):
    for response_time in range(request_time, 4):
        for now in range(response_time, 5):
            for date_value in range(5):
                for age_value in range(4):
                    for max_age in (None, 0, 3):
                        for s_maxage in (None, 0, 4):
                            for expires_value in (None, 0, 3, 6):
                                for shared in (False, True):
                                    result = compute_http_age_and_freshness(
                                        request_time=request_time,
                                        response_time=response_time,
                                        now=now,
                                        date_value=date_value,
                                        age_value=age_value,
                                        expires_value=expires_value,
                                        max_age=max_age,
                                        s_maxage=s_maxage,
                                        shared=shared,
                                    )

                                    expected_apparent = max(
                                        0,
                                        response_time - date_value,
                                    )
                                    expected_delay = response_time - request_time
                                    expected_corrected = age_value + expected_delay
                                    expected_initial = max(
                                        expected_apparent,
                                        expected_corrected,
                                    )
                                    expected_resident = now - response_time
                                    expected_current = max(
                                        0,
                                        response_time - date_value,
                                        age_value + response_time - request_time,
                                    ) + (now - response_time)

                                    if shared and s_maxage is not None:
                                        expected_lifetime = s_maxage
                                        expected_source = FreshnessSource.S_MAXAGE
                                    elif max_age is not None:
                                        expected_lifetime = max_age
                                        expected_source = FreshnessSource.MAX_AGE
                                    elif expires_value is not None:
                                        expected_lifetime = expires_value - date_value
                                        expected_source = FreshnessSource.EXPIRES
                                    else:
                                        expected_lifetime = None
                                        expected_source = FreshnessSource.NONE
                                    expected_fresh = (
                                        None
                                        if expected_lifetime is None
                                        else expected_lifetime > expected_current
                                    )

                                    assert result == HttpAgeAndFreshness(
                                        apparent_age=expected_apparent,
                                        corrected_age_value=expected_corrected,
                                        response_delay=expected_delay,
                                        corrected_initial_age=expected_initial,
                                        resident_time=expected_resident,
                                        current_age=expected_current,
                                        freshness_lifetime=expected_lifetime,
                                        freshness_source=expected_source,
                                        is_fresh=expected_fresh,
                                    )
                                    sources_seen.add(result.freshness_source)
                                    signature += (
                                        (result.current_age + 1)
                                        * source_rank[result.freshness_source]
                                        + abs(result.freshness_lifetime or 0)
                                        + int(result.is_fresh is True)
                                    )
                                    checked_cases += 1

assert checked_cases == 40_320
assert sources_seen == set(FreshnessSource)
assert signature > checked_cases
```

## Trade-offs and Limitations

The function uses `O(1)` time and `O(1)` auxiliary space. All timestamps must be
nonnegative integer seconds in one coordinate system, and the request,
response, and evaluation times must be ordered. Python integers preserve exact
intermediate sums beyond the input bounds.

This deliberately narrow profile requires an explicit parsed `Date` value and
does not substitute a response timestamp when it is absent. An `Expires` value
before `Date` produces a negative lifetime, which remains visible and is stale;
it is not clamped to zero. `s-maxage` is considered only for a shared cache, and
freshness is strict: a response whose current age equals its lifetime is not
fresh.

The result is evidence, not a complete cache decision. It does not parse HTTP
fields, infer heuristic freshness, decide whether a response may be stored,
honor request controls, perform validation, or authorize serving stale content.

## Related Snippets

<!-- catalog:related:start -->
- [Parse a Bounded Response Cache-Control Field Under a Closed Profile](parse-a-bounded-response-cache-control-field-under-a-closed-profile.md)
- [Apply an HTTP Patch Only When a Strong ETag Still Matches](apply-an-http-patch-only-when-a-strong-etag-still-matches.md)
- [Build a Canonical HTTP Origin Key](build-a-canonical-http-origin-key.md)
<!-- catalog:related:end -->
