---
title: "Render Bounded Request Snapshots into a Length-Framed Replay Payload"
snippet_type: recipe
use_cases:
  - networking
  - serialization
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../networking-protocols/build-a-bounded-http-request-target-from-typed-segments.md
  - ../networking-protocols/read-and-write-size-capped-varint-frames.md
  - verify-ordered-http-client-expectations-with-bounded-mismatch-reports.md
---

# Render Bounded Request Snapshots into a Length-Framed Replay Payload

## Idea and Problem

Render a finite immutable tuple of already-sanitized request snapshots into deterministic length-framed bytes without parsing logs or performing replay.

The byte grammar is `payload = *frame`, `frame = length LF body`,
`length = NZDIGIT *DIGIT`, `body = "target:" target LF *header-line LF`, and
`header-line = "header:" lower-name ":" value LF`. Here `DIGIT` is `30-39`,
`NZDIGIT` is `31-39`, `LF` is `0A`, `SP` is `20`, and `VCHAR` is `21-7E` in
hexadecimal. Bodies are nonempty, so zero is never a length. `lower-name` is
`[a-z][a-z0-9-]{0,63}`, and `value` is empty or VCHAR runs separated by one
SP. `target` is `/` plus zero or more unreserved characters (`A-Z`, `a-z`,
`0-9`, `-._~`), sub-delimiters (`!$&'()*+,;=`), `:`, `@`, `/`, or `%` followed
by two uppercase hexadecimal digits, then an optional `?` and the same set plus
`?`. Header lines use unique normalized names in ascending byte order. Thus the
decimal prefix is canonical and counts every body byte, including its final
empty line.

## When to Use

Use this renderer when a test or offline review tool already owns a small,
privacy-reviewed tuple of request targets and explicitly approved headers. The
caller must supply a closed lowercase safe-header allowlist for the particular
fixture. Request targets must use origin-form ASCII with uppercase percent
escapes, and header values must use visible ASCII separated by single spaces.

Raw operational records, packet captures, credentials, cookies, and arbitrary request
objects are unsafe inputs. Sanitize and minimize them before constructing these
models, under a policy outside this helper. Use a protocol-specific fixture or
controlled test server when methods, bodies, redirects, connection behavior,
or exact wire semantics matter.

## Implementation

```python
import re
from dataclasses import dataclass

_MAX_RECORDS = 64
_MAX_ALLOWLIST_NAMES = 32
_MAX_HEADERS_PER_RECORD = 16
_MAX_TARGET_BYTES = 1_024
_MAX_HEADER_NAME_BYTES = 64
_MAX_HEADER_VALUE_BYTES = 1_024
_MAX_RECORD_BODY_BYTES = 32 * 1_024
_MAX_TOTAL_OUTPUT_BYTES = 256 * 1_024

_TARGET = re.compile(
    r"/(?:[A-Za-z0-9._~!$&'()*+,;=:@/-]|%[0-9A-F]{2})*"
    r"(?:\?(?:[A-Za-z0-9._~!$&'()*+,;=:@/?-]|%[0-9A-F]{2})*)?",
    re.ASCII,
)
_INPUT_HEADER_NAME = re.compile(r"[A-Za-z][A-Za-z0-9-]{0,63}", re.ASCII)
_CANONICAL_HEADER_NAME = re.compile(r"[a-z][a-z0-9-]{0,63}", re.ASCII)
_HEADER_VALUE = re.compile(r"(?:[!-~]+(?: [!-~]+)*)?", re.ASCII)
_FORBIDDEN_HEADER_NAMES = frozenset(
    {
        "authorization",
        "connection",
        "content-length",
        "cookie",
        "forwarded",
        "host",
        "proxy-authenticate",
        "proxy-authorization",
        "set-cookie",
        "te",
        "trailer",
        "transfer-encoding",
        "upgrade",
        "www-authenticate",
        "x-api-key",
        "x-forwarded-for",
    }
)


class ReplayPayloadError(Exception):
    pass


class ReplayPayloadTypeError(TypeError, ReplayPayloadError):
    pass


class ReplayPayloadValueError(ValueError, ReplayPayloadError):
    pass


@dataclass(frozen=True, slots=True)
class SafeHeader:
    name: str
    value: str


@dataclass(frozen=True, slots=True)
class RequestSnapshot:
    target: str
    headers: tuple[SafeHeader, ...] = ()


def _checked_allowlist(value: object) -> frozenset[str]:
    if type(value) is not frozenset:
        raise ReplayPayloadTypeError("safe_header_allowlist must be an exact frozenset")
    if len(value) > _MAX_ALLOWLIST_NAMES:
        raise ReplayPayloadValueError("safe_header_allowlist exceeds its name limit")
    if any(type(name) is not str for name in value):
        raise ReplayPayloadTypeError("safe_header_allowlist must contain exact strings")

    for name in sorted(value):
        if (
            not name.isascii()
            or len(name) > _MAX_HEADER_NAME_BYTES
            or _CANONICAL_HEADER_NAME.fullmatch(name) is None
        ):
            raise ReplayPayloadValueError(
                "safe_header_allowlist names must be canonical lowercase ASCII"
            )
        if name in _FORBIDDEN_HEADER_NAMES:
            raise ReplayPayloadValueError(
                "safe_header_allowlist contains a sensitive or framing header"
            )
    return value


def _prepare_record_body(
    snapshot: object,
    *,
    safe_header_allowlist: frozenset[str],
) -> bytes:
    if type(snapshot) is not RequestSnapshot:
        raise ReplayPayloadTypeError("snapshots must contain exact RequestSnapshot values")
    if type(snapshot.target) is not str:
        raise ReplayPayloadTypeError("request targets must be exact strings")
    if not snapshot.target.isascii():
        raise ReplayPayloadValueError("a request target is outside the canonical grammar")
    target = snapshot.target.encode("ascii")
    if not 1 <= len(target) <= _MAX_TARGET_BYTES:
        raise ReplayPayloadValueError("a request target exceeds its byte limit")
    if _TARGET.fullmatch(snapshot.target) is None:
        raise ReplayPayloadValueError("a request target is outside the canonical grammar")

    if type(snapshot.headers) is not tuple:
        raise ReplayPayloadTypeError("snapshot headers must be an exact tuple")
    if len(snapshot.headers) > _MAX_HEADERS_PER_RECORD:
        raise ReplayPayloadValueError("a snapshot exceeds its header count limit")

    prepared_headers: list[tuple[bytes, bytes]] = []
    normalized_names: set[str] = set()
    for header in snapshot.headers:
        if type(header) is not SafeHeader:
            raise ReplayPayloadTypeError("headers must contain exact SafeHeader values")
        if type(header.name) is not str or type(header.value) is not str:
            raise ReplayPayloadTypeError("header names and values must be exact strings")
        if not header.name.isascii():
            raise ReplayPayloadValueError("a header name is outside the ASCII grammar")
        input_name_bytes = header.name.encode("ascii")
        if len(input_name_bytes) > _MAX_HEADER_NAME_BYTES:
            raise ReplayPayloadValueError("a header name exceeds its byte limit")
        if _INPUT_HEADER_NAME.fullmatch(header.name) is None:
            raise ReplayPayloadValueError("a header name is outside the ASCII grammar")

        normalized_name = header.name.lower()
        if normalized_name in normalized_names:
            raise ReplayPayloadValueError("a snapshot contains a normalized duplicate header")
        if normalized_name in _FORBIDDEN_HEADER_NAMES:
            raise ReplayPayloadValueError("a snapshot contains an unsafe header name")
        if normalized_name not in safe_header_allowlist:
            raise ReplayPayloadValueError("a snapshot contains an unallowed header name")

        if not header.value.isascii():
            raise ReplayPayloadValueError("a header value is outside the canonical ASCII grammar")
        value_bytes = header.value.encode("ascii")
        if len(value_bytes) > _MAX_HEADER_VALUE_BYTES:
            raise ReplayPayloadValueError("a header value exceeds its byte limit")
        if _HEADER_VALUE.fullmatch(header.value) is None:
            raise ReplayPayloadValueError("a header value is outside the canonical ASCII grammar")
        name_bytes = normalized_name.encode("ascii")

        normalized_names.add(normalized_name)
        prepared_headers.append((name_bytes, value_bytes))

    body_parts = [b"target:" + target + b"\n"]
    body_parts.extend(
        b"header:" + name + b":" + value + b"\n" for name, value in sorted(prepared_headers)
    )
    body_parts.append(b"\n")
    body = b"".join(body_parts)
    if len(body) > _MAX_RECORD_BODY_BYTES:
        raise ReplayPayloadValueError("a record body exceeds its byte limit")
    return body


def render_replay_payload(
    snapshots: tuple[RequestSnapshot, ...],
    *,
    safe_header_allowlist: frozenset[str],
) -> bytes:
    if type(snapshots) is not tuple:
        raise ReplayPayloadTypeError("snapshots must be an exact tuple")
    if len(snapshots) > _MAX_RECORDS:
        raise ReplayPayloadValueError("snapshot count exceeds the record limit")
    checked_allowlist = _checked_allowlist(safe_header_allowlist)

    prepared_frames: list[tuple[bytes, bytes]] = []
    total_bytes = 0
    for snapshot in snapshots:
        body = _prepare_record_body(
            snapshot,
            safe_header_allowlist=checked_allowlist,
        )
        prefix = str(len(body)).encode("ascii") + b"\n"
        total_bytes += len(prefix) + len(body)
        if total_bytes > _MAX_TOTAL_OUTPUT_BYTES:
            raise ReplayPayloadValueError("payload exceeds the total byte limit")
        prepared_frames.append((prefix, body))

    return b"".join(prefix + body for prefix, body in prepared_frames)
```

## Example

```python
snapshots = (
    RequestSnapshot(
        "/v1/items?limit=2",
        (
            SafeHeader("X-Trace-Label", "fixture-7"),
            SafeHeader("Accept", "application/json"),
        ),
    ),
    RequestSnapshot("/health"),
)
payload = render_replay_payload(
    snapshots,
    safe_header_allowlist=frozenset({"accept", "x-trace-label"}),
)

first_body = (
    b"target:/v1/items?limit=2\nheader:accept:application/json\nheader:x-trace-label:fixture-7\n\n"
)
second_body = b"target:/health\n\n"
assert (len(first_body), len(second_body), payload) == (
    88,
    16,
    b"88\n" + first_body + b"16\n" + second_body,
)

try:
    render_replay_payload(
        (
            RequestSnapshot(
                "/v1/items",
                (SafeHeader("X-Mode", "one"), SafeHeader("x-mode", "two")),
            ),
        ),
        safe_header_allowlist=frozenset({"x-mode"}),
    )
except ReplayPayloadValueError:
    normalized_duplicate_rejected = True
else:
    normalized_duplicate_rejected = False

assert normalized_duplicate_rejected
```

## Trade-offs and Limitations

Complete preflight builds every bounded body and checks the final framed size
before the function returns any bytes. This costs memory proportional to the
output limit, but prevents a late invalid record from producing a partial
payload. Test empty input, exact limits, one-byte-over-limit fields, mixed-case
duplicates, forbidden and merely unallowed names, lowercase percent escapes,
non-ASCII, CR/LF, controls, and the total-size boundary.

The renderer deliberately does not parse raw logs, remove secrets, write a
file, open a connection, call an HTTP client, or execute a replay. The framing
is a fixture format, not HTTP wire syntax. Its bytes do not prove replay
fidelity, authorization safety, request validity for a particular server, or
security of any downstream executor. A consumer must independently parse with
the same caps and revalidate policy before doing anything with a record.

## Related Snippets

<!-- catalog:related:start -->
- [Build a Bounded HTTP Request Target from Typed Segments](../networking-protocols/build-a-bounded-http-request-target-from-typed-segments.md)
- [Read and Write Size-Capped Varint Frames](../networking-protocols/read-and-write-size-capped-varint-frames.md)
- [Verify Ordered HTTP Client Expectations with Bounded Mismatch Reports](verify-ordered-http-client-expectations-with-bounded-mismatch-reports.md)
<!-- catalog:related:end -->
