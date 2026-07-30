---
title: "Render One Bounded Set-Cookie Field from Explicit Attributes"
snippet_type: recipe
use_cases:
  - interoperability
  - networking
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - parse-a-bounded-cookie-request-field-without-collapsing-duplicate-names.md
  - extract-bounded-singleton-http-headers-with-explicit-rules.md
  - parse-a-bounded-response-cache-control-field-under-a-closed-profile.md
---

# Render One Bounded Set-Cookie Field from Explicit Attributes

## Idea and Problem

Render one strict bounded Set-Cookie field value from typed attributes without quoting, encoding, or inventing browser policy.

The cookie pair uses a closed ASCII name and cookie-octet grammar. Supported
attributes appear in one fixed order, and cross-field validation enforces the
security invariants attached to `__Secure-`, `__Host-`, and `SameSite=None`.
The result is only the field value, so it cannot accidentally add a header name
or line framing supplied by the HTTP layer.

## When to Use

Use this recipe at a small server response boundary when cookie values are
already encoded into the permitted octets and the application explicitly owns
Max-Age, Path, Secure, HttpOnly, and SameSite choices. An empty cookie value is
valid, and omission remains distinct from a present attribute.

Use a framework or reviewed cookie policy layer when Domain, Expires,
Partitioned, public-suffix decisions, browser compatibility, deletion, signing,
or value encoding is required. Do not feed arbitrary Unicode or user text into
this renderer; encode application data before this boundary under a separate
size and security contract.

## Implementation

```python
import re
from dataclasses import dataclass
from enum import StrEnum

_MAX_INT64 = 2**63 - 1
_MAX_NAME_CHARACTERS = 64
_MAX_VALUE_CHARACTERS = 1_024
_MAX_PATH_CHARACTERS = 256
_MAX_OUTPUT_CHARACTERS = 4_096
_TOKEN = re.compile(
    rf"[!#$%&'*+\-.^_`|~0-9A-Za-z]{{1,{_MAX_NAME_CHARACTERS}}}",
    re.ASCII,
).fullmatch
_COOKIE_VALUE = re.compile(
    rf"[\x21\x23-\x2B\x2D-\x3A\x3C-\x5B\x5D-\x7E]"
    rf"{{0,{_MAX_VALUE_CHARACTERS}}}",
    re.ASCII,
).fullmatch


class SameSite(StrEnum):
    STRICT = "Strict"
    LAX = "Lax"
    NONE = "None"


@dataclass(frozen=True, slots=True)
class SetCookieSpec:
    name: str
    value: str
    max_age: int | None = None
    path: str | None = None
    secure: bool = False
    http_only: bool = False
    same_site: SameSite | None = None


def render_set_cookie(specification: SetCookieSpec) -> str:
    if type(specification) is not SetCookieSpec:
        raise TypeError("specification must be an exact SetCookieSpec")
    if type(specification.name) is not str:
        raise TypeError("specification.name must be an exact string")
    if _TOKEN(specification.name) is None:
        raise ValueError("specification.name is outside the closed token grammar")
    if type(specification.value) is not str:
        raise TypeError("specification.value must be an exact string")
    if _COOKIE_VALUE(specification.value) is None:
        raise ValueError("specification.value is outside the cookie-octet grammar")

    if specification.max_age is not None:
        if type(specification.max_age) is not int:
            raise TypeError("specification.max_age must be exactly int or None")
        if not 1 <= specification.max_age <= _MAX_INT64:
            raise ValueError("specification.max_age is outside 1..2**63-1")
    if specification.path is not None:
        if type(specification.path) is not str:
            raise TypeError("specification.path must be exactly str or None")
        if not 1 <= len(specification.path) <= _MAX_PATH_CHARACTERS:
            raise ValueError("specification.path is outside the supported length")
        if not specification.path.startswith("/"):
            raise ValueError("specification.path must begin with /")
        if any(
            not 0x21 <= ord(character) <= 0x7E or character == ";"
            for character in specification.path
        ):
            raise ValueError("specification.path is outside the closed ASCII grammar")
    if type(specification.secure) is not bool:
        raise TypeError("specification.secure must be an exact boolean")
    if type(specification.http_only) is not bool:
        raise TypeError("specification.http_only must be an exact boolean")
    if (
        specification.same_site is not None
        and type(specification.same_site) is not SameSite
    ):
        raise TypeError("specification.same_site must be exactly SameSite or None")

    if specification.name.startswith("__Secure-") and not specification.secure:
        raise ValueError("a __Secure- cookie requires Secure")
    if specification.name.startswith("__Host-") and (
        not specification.secure or specification.path != "/"
    ):
        raise ValueError("a __Host- cookie requires Secure and Path=/")
    if specification.same_site is SameSite.NONE and not specification.secure:
        raise ValueError("SameSite=None requires Secure")

    parts = [f"{specification.name}={specification.value}"]
    if specification.max_age is not None:
        parts.append(f"Max-Age={specification.max_age}")
    if specification.path is not None:
        parts.append(f"Path={specification.path}")
    if specification.secure:
        parts.append("Secure")
    if specification.http_only:
        parts.append("HttpOnly")
    if specification.same_site is not None:
        parts.append(f"SameSite={specification.same_site.value}")

    rendered = "; ".join(parts)
    if len(rendered) > _MAX_OUTPUT_CHARACTERS:
        raise ValueError("rendered field exceeds the output length limit")
    return rendered
```

## Example

```python
rendered = render_set_cookie(
    SetCookieSpec(
        "__Host-session",
        "abc-123",
        max_age=3_600,
        path="/",
        secure=True,
        http_only=True,
        same_site=SameSite.LAX,
    )
)

def check_attribute_product() -> int:
    from itertools import product

    checked = 0
    for max_age, path, secure, http_only, same_site in product(
        (None, 1, _MAX_INT64),
        (None, "/", "/scope"),
        (False, True),
        (False, True),
        (None, SameSite.STRICT, SameSite.LAX, SameSite.NONE),
    ):
        specification = SetCookieSpec(
            "session",
            "",
            max_age=max_age,
            path=path,
            secure=secure,
            http_only=http_only,
            same_site=same_site,
        )
        if same_site is SameSite.NONE and not secure:
            try:
                render_set_cookie(specification)
            except ValueError:
                pass
            else:
                raise AssertionError("SameSite=None without Secure was accepted")
        else:
            result = render_set_cookie(specification)
            parts = result.split("; ")
            expected_names = ["session="]
            if max_age is not None:
                expected_names.append(f"Max-Age={max_age}")
            if path is not None:
                expected_names.append(f"Path={path}")
            if secure:
                expected_names.append("Secure")
            if http_only:
                expected_names.append("HttpOnly")
            if same_site is not None:
                expected_names.append(f"SameSite={same_site.value}")
            assert parts == expected_names
        checked += 1
    return checked


checked = check_attribute_product()

invalid_specs = (
    SetCookieSpec("bad name", "value"),
    SetCookieSpec("name", 'bad"value'),
    SetCookieSpec("name", "bad;value"),
    SetCookieSpec("name", "value", max_age=0),
    SetCookieSpec("name", "value", path="relative"),
    SetCookieSpec("name", "value", path="/bad;path"),
    SetCookieSpec("__Secure-name", "value"),
    SetCookieSpec("__Host-name", "value", path="/", secure=False),
    SetCookieSpec("__Host-name", "value", path="/scope", secure=True),
)
rejected = 0
for invalid in invalid_specs:
    try:
        render_set_cookie(invalid)
    except ValueError:
        rejected += 1

maximum = render_set_cookie(
    SetCookieSpec(
        "n" * _MAX_NAME_CHARACTERS,
        "v" * _MAX_VALUE_CHARACTERS,
        max_age=_MAX_INT64,
        path="/" + "p" * (_MAX_PATH_CHARACTERS - 1),
        secure=True,
        http_only=True,
        same_site=SameSite.NONE,
    )
)

assert (
    rendered
    == "__Host-session=abc-123; Max-Age=3600; Path=/; Secure; "
    "HttpOnly; SameSite=Lax"
    and checked == 144
    and rejected == len(invalid_specs)
    and len(maximum) < _MAX_OUTPUT_CHARACTERS
)
```

## Trade-offs and Limitations

Validation and rendering take `O(N)` time and memory for the bounded output
length `N`. Refusing quoting and percent encoding makes the transformation
transparent but requires callers to choose an application encoding before this
boundary. Attribute order is deterministic without carrying HTTP semantics.

The profile deliberately omits Domain and Expires, so every rendered cookie is
host-only and Max-Age cannot express deletion through zero. Prefix and
SameSite checks cover only the explicit cross-field invariants above; they do
not prove that a browser will accept, retain, send, or protect the cookie in a
particular deployment.

## Related Snippets

<!-- catalog:related:start -->
- [Parse a Bounded Cookie Request Field Without Collapsing Duplicate Names](parse-a-bounded-cookie-request-field-without-collapsing-duplicate-names.md)
- [Extract Bounded Singleton HTTP Headers with Explicit Rules](extract-bounded-singleton-http-headers-with-explicit-rules.md)
- [Parse a Bounded Response Cache-Control Field Under a Closed Profile](parse-a-bounded-response-cache-control-field-under-a-closed-profile.md)
<!-- catalog:related:end -->
