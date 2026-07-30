---
title: "Classify One HTTP/1 Response Body Framing from Validated Metadata"
snippet_type: recipe
use_cases:
  - networking
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - encode-a-bounded-http-1-1-chunked-body-for-protocol-tests.md
  - extract-bounded-singleton-http-headers-with-explicit-rules.md
  - release-a-pooled-response-connection-only-after-clean-eof.md
---

# Classify One HTTP/1 Response Body Framing from Validated Metadata

## Idea and Problem

Choose exactly one HTTP/1 response-body framing mode from bounded, already normalized metadata before a connection reader consumes any body bytes.

The classifier makes precedence explicit: contradictory framing is invalid;
method and status semantics can suppress a body; a successful CONNECT changes
the connection into a tunnel; and only then do transfer coding, content length,
and close-delimited framing decide how a body ends.

## When to Use

Use this recipe at the boundary between a strict HTTP/1 header parser and a
bounded response-body reader. It is useful when repeated Content-Length values
have already been decoded as integers, Transfer-Encoding has already been
split into lowercase coding tokens, and the caller needs a small closed result
instead of scattered framing conditions.

Use a maintained HTTP implementation for general clients, servers, proxies,
or security gateways. This classifier intentionally does not parse raw fields,
apply message-routing rules, decode transfer codings, handle upgrades, or
manage connection reuse.

## Implementation

```python
import re
from dataclasses import dataclass
from enum import StrEnum

_MAX_INT64 = 2**63 - 1
_METHOD = re.compile(
    r"[!#$%&'*+\-.^_`|~0-9A-Z]{1,32}",
    re.ASCII,
).fullmatch
_TRANSFER_CODING = re.compile(
    r"[!#$%&'*+\-.^_`|~0-9a-z]{1,32}",
    re.ASCII,
).fullmatch


class ResponseBodyMode(StrEnum):
    INVALID = "invalid"
    NO_BODY = "no-body"
    TUNNEL = "tunnel"
    CHUNKED = "chunked"
    FIXED = "fixed"
    UNTIL_EOF = "until-eof"


@dataclass(frozen=True, slots=True)
class ResponseBodyFraming:
    mode: ResponseBodyMode
    content_length: int | None = None


def classify_response_body_framing(
    method: str,
    status_code: int,
    transfer_codings: tuple[str, ...],
    content_lengths: tuple[int, ...],
) -> ResponseBodyFraming:
    if type(method) is not str:
        raise TypeError("method must be an exact string")
    if _METHOD(method) is None:
        raise ValueError("method must be a canonical uppercase token of 1..32 bytes")
    if type(status_code) is not int:
        raise TypeError("status_code must be an exact integer")
    if not 100 <= status_code <= 599:
        raise ValueError("status_code is outside 100..599")

    if type(transfer_codings) is not tuple:
        raise TypeError("transfer_codings must be an exact tuple")
    if len(transfer_codings) > 8:
        raise ValueError("transfer_codings contains more than 8 items")
    for index, coding in enumerate(transfer_codings):
        if type(coding) is not str:
            raise TypeError(f"transfer_codings[{index}] must be an exact string")
        if _TRANSFER_CODING(coding) is None:
            raise ValueError(
                f"transfer_codings[{index}] must be a lowercase token of 1..32 bytes"
            )

    if type(content_lengths) is not tuple:
        raise TypeError("content_lengths must be an exact tuple")
    if len(content_lengths) > 8:
        raise ValueError("content_lengths contains more than 8 items")
    for index, length in enumerate(content_lengths):
        if type(length) is not int:
            raise TypeError(f"content_lengths[{index}] must be an exact integer")
        if not 0 <= length <= _MAX_INT64:
            raise ValueError(f"content_lengths[{index}] is outside 0..2**63-1")

    if transfer_codings and content_lengths:
        return ResponseBodyFraming(ResponseBodyMode.INVALID)
    if content_lengths and len(set(content_lengths)) != 1:
        return ResponseBodyFraming(ResponseBodyMode.INVALID)

    chunked_count = transfer_codings.count("chunked")
    if chunked_count > 1 or (
        chunked_count == 1 and transfer_codings[-1] != "chunked"
    ):
        return ResponseBodyFraming(ResponseBodyMode.INVALID)

    if (
        method == "HEAD"
        or 100 <= status_code < 200
        or status_code in {204, 304}
    ):
        return ResponseBodyFraming(ResponseBodyMode.NO_BODY)
    if method == "CONNECT" and 200 <= status_code < 300:
        return ResponseBodyFraming(ResponseBodyMode.TUNNEL)
    if transfer_codings:
        mode = (
            ResponseBodyMode.CHUNKED
            if transfer_codings[-1] == "chunked"
            else ResponseBodyMode.UNTIL_EOF
        )
        return ResponseBodyFraming(mode)
    if content_lengths:
        return ResponseBodyFraming(
            ResponseBodyMode.FIXED,
            content_lengths[0],
        )
    return ResponseBodyFraming(ResponseBodyMode.UNTIL_EOF)
```

## Example

```python
semantic_classes = (
    ("HEAD", 200, ResponseBodyMode.NO_BODY),
    ("GET", 100, ResponseBodyMode.NO_BODY),
    ("GET", 199, ResponseBodyMode.NO_BODY),
    ("GET", 204, ResponseBodyMode.NO_BODY),
    ("GET", 304, ResponseBodyMode.NO_BODY),
    ("CONNECT", 204, ResponseBodyMode.NO_BODY),
    ("CONNECT", 200, ResponseBodyMode.TUNNEL),
    ("CONNECT", 205, ResponseBodyMode.TUNNEL),
    ("CONNECT", 299, ResponseBodyMode.TUNNEL),
    ("GET", 200, None),
    ("GET", 205, None),
    ("CONNECT", 300, None),
    ("GET", 599, None),
)
framing_classes = (
    ((), (), ResponseBodyMode.UNTIL_EOF, None),
    (("chunked",), (), ResponseBodyMode.CHUNKED, None),
    (("gzip", "chunked"), (), ResponseBodyMode.CHUNKED, None),
    (("gzip",), (), ResponseBodyMode.UNTIL_EOF, None),
    ((), (0,), ResponseBodyMode.FIXED, 0),
    ((), (7, 7), ResponseBodyMode.FIXED, 7),
    (("gzip",), (7,), ResponseBodyMode.INVALID, None),
    ((), (7, 8), ResponseBodyMode.INVALID, None),
    (("chunked", "gzip"), (), ResponseBodyMode.INVALID, None),
    (("chunked", "chunked"), (), ResponseBodyMode.INVALID, None),
)

checked_cases = 0
for method, status, semantic_override in semantic_classes:
    for codings, lengths, framing_mode, framing_length in framing_classes:
        if framing_mode is ResponseBodyMode.INVALID:
            expected_mode = ResponseBodyMode.INVALID
        elif semantic_override is not None:
            expected_mode = semantic_override
        else:
            expected_mode = framing_mode
        expected_length = (
            framing_length
            if expected_mode is ResponseBodyMode.FIXED
            else None
        )
        observed = classify_response_body_framing(
            method,
            status,
            codings,
            lengths,
        )
        assert (observed.mode, observed.content_length) == (
            expected_mode,
            expected_length,
        )
        checked_cases += 1

maximum_codings = classify_response_body_framing(
    "X" * 32,
    599,
    ("a" * 32,) * 8,
    (),
)
maximum_lengths = classify_response_body_framing(
    "GET",
    200,
    (),
    (_MAX_INT64,) * 8,
)

invalid_inputs = (
    ("get", 200, (), ()),
    ("X" * 33, 200, (), ()),
    ("GET", 99, (), ()),
    ("GET", 600, (), ()),
    ("GET", True, (), ()),
    ("GET", 200, ("Chunked",), ()),
    ("GET", 200, ("",), ()),
    ("GET", 200, ("a" * 33,), ()),
    ("GET", 200, ("gzip",) * 9, ()),
    ("GET", 200, (), (True,)),
    ("GET", 200, (), (-1,)),
    ("GET", 200, (), (_MAX_INT64 + 1,)),
    ("GET", 200, (), (0,) * 9),
)
rejected = 0
for arguments in invalid_inputs:
    try:
        classify_response_body_framing(*arguments)
    except (TypeError, ValueError):
        rejected += 1

assert (
    checked_cases == len(semantic_classes) * len(framing_classes)
    and maximum_codings
    == ResponseBodyFraming(ResponseBodyMode.UNTIL_EOF)
    and maximum_lengths
    == ResponseBodyFraming(ResponseBodyMode.FIXED, _MAX_INT64)
    and rejected == len(invalid_inputs)
)
```

## Trade-offs and Limitations

Validation and classification take `O(T + C)` time for `T` transfer codings
and `C` Content-Length occurrences, with `O(C)` temporary space for the
duplicate-length check. Both collections are capped at eight items.

Returning `INVALID` separates contradictory but well-formed metadata from
malformed Python input, which raises immediately. Equal repeated lengths are
accepted under this closed profile; deployments with a stricter policy can
reject every repetition before calling the classifier. A `NO_BODY` or `TUNNEL`
result deliberately discards any otherwise valid Content-Length value because
it must not drive response-body reads in those modes.

The result says how framing ends, not whether the message is safe to reuse.
The caller still needs bounded decoders, clean-EOF accounting, timeout policy,
and explicit handling for connection closure, upgrades, and protocol errors.

## Related Snippets

<!-- catalog:related:start -->
- [Encode a Bounded HTTP/1.1 Chunked Body for Protocol Tests](encode-a-bounded-http-1-1-chunked-body-for-protocol-tests.md)
- [Extract Bounded Singleton HTTP Headers with Explicit Rules](extract-bounded-singleton-http-headers-with-explicit-rules.md)
- [Release a Pooled Response Connection Only After Clean EOF](release-a-pooled-response-connection-only-after-clean-eof.md)
<!-- catalog:related:end -->
