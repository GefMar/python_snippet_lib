---
title: "Redact Userinfo and Selected Query Values in a Bounded HTTP URL"
snippet_type: recipe
use_cases:
  - observability
  - security
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../networking-protocols/parse-a-bounded-percent-encoded-query-string-as-strict-utf-8-with-explicit-duplicate-rules.md
  - separate-executable-and-redacted-views-of-a-command-argument-vector.md
  - create-and-verify-a-short-lived-hmac-download-url.md
---

# Redact Userinfo and Selected Query Values in a Bounded HTTP URL

## Idea and Problem

Produce bounded diagnostic text for a strict HTTP(S) URL without retaining authority credentials, a fragment, or values of explicitly sensitive query fields.

The boundary validates a closed visible-ASCII URI profile before separating
components. Query names are percent-decoded as strict UTF-8 only for policy
comparison; their original spelling, order, and duplicates stay intact in the
output. A literal plus remains a plus rather than silently acquiring HTML-form
space semantics.

## When to Use

Use this recipe immediately before bounded diagnostic logging when callers
already know which query names are sensitive. It handles absolute `http` and
`https` text, bracketed IPv6 authorities, optional numeric ports, duplicate
query fields, and percent-encoded UTF-8 names.

The returned string is a diagnostic view, not a URL to authorize, compare,
store as identity, or send in a request. The policy cannot discover secrets in
paths, unlisted names, nested values, or application-specific encodings. Keep
the original URL in the trusted request path and use a full URL policy layer
for fetch safety.

## Implementation

```python
import ipaddress
import re
from dataclasses import dataclass
from urllib.parse import unquote_to_bytes, urlsplit

_MAX_URL_CHARACTERS = 8_192
_MAX_AUTHORITY_CHARACTERS = 1_024
_MAX_QUERY_FIELDS = 128
_MAX_SENSITIVE_NAMES = 64
_MAX_SENSITIVE_NAME_BYTES = 128
_MAX_POLICY_BYTES = 4_096
_REDACTION = "REDACTED"

_PCT = r"%[0-9A-Fa-f]{2}"
_UNRESERVED = r"A-Za-z0-9._~\-"
_SUB_DELIMITERS = r"!$&'()*+,;="
_USERINFO = re.compile(rf"(?:[{_UNRESERVED}{_SUB_DELIMITERS}:]|{_PCT})*").fullmatch
_REG_NAME = re.compile(rf"(?:[{_UNRESERVED}{_SUB_DELIMITERS}]|{_PCT})+").fullmatch
_PATH = re.compile(rf"(?:[{_UNRESERVED}{_SUB_DELIMITERS}:@/]|{_PCT})*").fullmatch
_QUERY_OR_FRAGMENT = re.compile(
    rf"(?:[{_UNRESERVED}{_SUB_DELIMITERS}:@/?]|{_PCT})*"
).fullmatch


@dataclass(frozen=True, slots=True)
class RedactedUrlDiagnostic:
    text: str
    userinfo_removed: bool
    redacted_fields: int


def _validated_policy(value: object) -> frozenset[str]:
    if type(value) is not tuple:
        raise TypeError("sensitive_names must be an exact tuple")
    if len(value) > _MAX_SENSITIVE_NAMES:
        raise ValueError("sensitive_names exceeds the supported count")

    names: set[str] = set()
    aggregate_bytes = 0
    for index, name in enumerate(value):
        if type(name) is not str:
            raise TypeError(f"sensitive_names[{index}] must be an exact string")
        try:
            size = len(name.encode("utf-8"))
        except UnicodeEncodeError:
            raise ValueError("a sensitive name must be UTF-8 encodable") from None
        if not 1 <= size <= _MAX_SENSITIVE_NAME_BYTES:
            raise ValueError("a sensitive name is outside its UTF-8 byte limit")
        aggregate_bytes += size
        if aggregate_bytes > _MAX_POLICY_BYTES:
            raise ValueError("sensitive names exceed the aggregate byte limit")
        if name in names:
            raise ValueError("sensitive names must be unique")
        names.add(name)
    return frozenset(names)


def _split_authority(authority: str) -> tuple[str, bool]:
    if not 1 <= len(authority) <= _MAX_AUTHORITY_CHARACTERS:
        raise ValueError("authority is outside the supported length")
    if authority.count("@") > 1:
        raise ValueError("authority contains more than one raw @")

    userinfo, separator, host_port = authority.rpartition("@")
    if separator and _USERINFO(userinfo) is None:
        raise ValueError("userinfo is outside the closed URI grammar")
    if not separator:
        host_port = authority

    port_text: str | None = None
    if host_port.startswith("["):
        closing = host_port.find("]")
        if closing < 0:
            raise ValueError("bracketed host is missing its closing bracket")
        host_text = host_port[1:closing]
        suffix = host_port[closing + 1 :]
        if "%" in host_text:
            raise ValueError("IPv6 zone identifiers are outside the closed profile")
        try:
            ipaddress.IPv6Address(host_text)
        except ValueError:
            raise ValueError("bracketed host must be an IPv6 address") from None
        if suffix:
            if not suffix.startswith(":"):
                raise ValueError("unexpected text follows the bracketed host")
            port_text = suffix[1:]
    else:
        if host_port.count(":") > 1:
            raise ValueError("an IPv6 host must be enclosed in brackets")
        host_text, colon, candidate_port = host_port.rpartition(":")
        if colon:
            port_text = candidate_port
        else:
            host_text = host_port
        if _REG_NAME(host_text) is None:
            raise ValueError("host is outside the closed URI grammar")
        if host_text.replace(".", "").isdigit() and "." in host_text:
            try:
                ipaddress.IPv4Address(host_text)
            except ValueError:
                raise ValueError("numeric dotted host must be a valid IPv4 address") from None

    if port_text is not None:
        if not port_text.isascii() or not port_text.isdecimal():
            raise ValueError("a present port must contain decimal digits")
        if not 1 <= int(port_text) <= 65_535:
            raise ValueError("port is outside 1..65535")
    return host_port, bool(separator)


def redact_http_url(
    url: str,
    *,
    sensitive_names: tuple[str, ...],
) -> RedactedUrlDiagnostic:
    if type(url) is not str:
        raise TypeError("url must be an exact string")
    if not 1 <= len(url) <= _MAX_URL_CHARACTERS:
        raise ValueError("url is outside the supported length")
    if any(not 0x21 <= ord(character) <= 0x7E for character in url):
        raise ValueError("url must contain visible ASCII only")
    if "\\" in url:
        raise ValueError("backslashes are outside the closed URL profile")
    if not (url.startswith("http://") or url.startswith("https://")):
        raise ValueError("url must begin with lowercase http:// or https://")
    policy = _validated_policy(sensitive_names)

    without_fragment, _fragment_separator, fragment = url.partition("#")
    if "#" in fragment or _QUERY_OR_FRAGMENT(fragment) is None:
        raise ValueError("fragment is outside the closed URI grammar")

    scheme, remainder = without_fragment.split("://", 1)
    authority_end = len(remainder)
    for delimiter in ("/", "?"):
        position = remainder.find(delimiter)
        if position >= 0:
            authority_end = min(authority_end, position)
    authority = remainder[:authority_end]
    tail = remainder[authority_end:]
    host_port, userinfo_removed = _split_authority(authority)

    parsed = urlsplit(without_fragment)
    if parsed.scheme != scheme or parsed.netloc != authority or parsed.hostname is None:
        raise ValueError("URL parser disagrees with the validated authority")
    try:
        _parsed_port = parsed.port
    except ValueError:
        raise ValueError("authority contains an invalid port") from None

    if "?" in tail:
        path, query = tail.split("?", 1)
        query_separator = "?"
    else:
        path = tail
        query = ""
        query_separator = ""
    if _PATH(path) is None or _QUERY_OR_FRAGMENT(query) is None:
        raise ValueError("path or query is outside the closed URI grammar")

    redacted_fields = 0
    output_fields: list[str] = []
    if query:
        raw_fields = query.split("&")
        if len(raw_fields) > _MAX_QUERY_FIELDS:
            raise ValueError("query exceeds the supported field count")
        for raw_field in raw_fields:
            if not raw_field or "=" not in raw_field:
                raise ValueError("each query field needs a non-empty name and =")
            raw_name, raw_value = raw_field.split("=", 1)
            if not raw_name:
                raise ValueError("each query field needs a non-empty name")
            try:
                decoded_name = unquote_to_bytes(raw_name).decode("utf-8", "strict")
            except UnicodeDecodeError:
                raise ValueError("a query name is not strict percent-encoded UTF-8") from None
            if decoded_name in policy:
                raw_value = _REDACTION
                redacted_fields += 1
            output_fields.append(f"{raw_name}={raw_value}")

    diagnostic = (
        f"{scheme}://{host_port}{path}{query_separator}{'&'.join(output_fields)}"
    )
    if len(diagnostic) > _MAX_URL_CHARACTERS:
        raise ValueError("redacted diagnostic exceeds the output length limit")
    return RedactedUrlDiagnostic(diagnostic, userinfo_removed, redacted_fields)
```

## Example

```python
credential_url = (
    "https://"
    + "sample-user:sample-pass@"
    + "[2001:db8::1]:8443/a%2Fb"
    + "?token=sample&public=a%3Db&t%6fken=x#private"
)
observed = redact_http_url(
    credential_url,
    sensitive_names=("token",),
)
plus_is_literal = redact_http_url(
    "http://example.test/?a+b=visible&a%2Bb=hidden",
    sensitive_names=("a+b",),
)

invalid_urls = (
    "HTTP://example.test/?token=sample",
    "https://" + "first@second@" + "example.test/?token=sample",
    "https://example.test:70000/?token=sample",
    "https://example.test/path\\segment?token=sample",
    "https://example.test/?public=%ZZ",
    "https://example.test/?missing-delimiter",
    "https:" + "//[" + "2001:db8::1%25eth0" + "]/?public=sample",
    "https:" + "//[" + "2001:db8::1%ZZ" + "]/?public=sample",
)
rejected = 0
for invalid in invalid_urls:
    try:
        redact_http_url(invalid, sensitive_names=("token",))
    except ValueError:
        rejected += 1

assert (
    observed
    == RedactedUrlDiagnostic(
        "https://[2001:db8::1]:8443/a%2Fb"
        "?token=REDACTED&public=a%3Db&t%6fken=REDACTED",
        True,
        2,
    )
    and plus_is_literal.text
    == "http://example.test/?a+b=REDACTED&a%2Bb=REDACTED"
    and plus_is_literal.redacted_fields == 2
    and rejected == len(invalid_urls)
)
```

## Trade-offs and Limitations

Validation and redaction take `O(U + P)` time and memory for URL length `U`
and policy text `P`. The function intentionally validates before calling
`urlsplit`, because that standard-library parser separates components but does
not establish that arbitrary input is safe or valid. Reconstructing only the
closed profile also avoids parser normalization hiding malformed input.

The diagnostic preserves non-sensitive raw query spelling, which can still
contain identifying information. It removes userinfo rather than replacing it,
drops every fragment, and redacts only exact decoded names. It has no HTML-form
decoding, relative references, Unicode hosts, IPv6 zone identifiers, empty
query fields, secret discovery, request execution, or canonicalization policy.

## Related Snippets

<!-- catalog:related:start -->
- [Parse a Bounded Percent-Encoded Query String as Strict UTF-8 with Explicit Duplicate Rules](../networking-protocols/parse-a-bounded-percent-encoded-query-string-as-strict-utf-8-with-explicit-duplicate-rules.md)
- [Separate Executable and Redacted Views of a Command Argument Vector](separate-executable-and-redacted-views-of-a-command-argument-vector.md)
- [Create and Verify a Short-Lived HMAC Download URL](create-and-verify-a-short-lived-hmac-download-url.md)
<!-- catalog:related:end -->
