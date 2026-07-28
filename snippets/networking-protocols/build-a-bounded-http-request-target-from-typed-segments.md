---
title: "Build a Bounded HTTP Request Target from Typed Segments"
snippet_type: recipe
use_cases:
  - networking
  - serialization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - build-a-canonical-http-origin-key.md
  - collect-matching-cursor-pages-with-an-explicit-page-budget.md
  - ../configuration-serialization/parse-a-bounded-component-options-expression.md
---

# Build a Bounded HTTP Request Target from Typed Segments

## Idea and Problem

Build one deterministic HTTP request target from explicitly typed path segments and one closed cursor-pagination and ordering specification.

The segment types select separate grammars instead of relying on positions in
an untyped list. Text is encoded as a path component, integers receive one
canonical decimal spelling, and query pairs are emitted in declaration order.
Every input and output budget is checked before the immutable plan is returned.

## When to Use

Use this recipe when application code already owns a small route vocabulary
but must safely combine caller-supplied segment values with fixed pagination
and ordering fields. It fits boundaries where the result will later be passed
to a reviewed HTTP client as its path and query target.

Use a URI-template or framework router when route expansion must follow that
tool's semantics. This helper deliberately builds only a request target; an
origin, HTTP method, headers, body, and transport configuration belong to other
boundaries.

## Implementation

```python
import re
from dataclasses import dataclass
from typing import Literal
from urllib.parse import quote, urlencode

SortField = Literal["changed", "label", "sequence"]
SortDirection = Literal["asc", "desc"]

_MAX_PATH_SEGMENTS = 6
_MAX_RAW_SEGMENT_BYTES = 48
_MAX_ENCODED_SEGMENT_BYTES = 144
_MAX_QUERY_PAIRS = 4
_MAX_TARGET_BYTES = 512
_MIN_NUMBER = 0
_MAX_NUMBER = 9_999_999
_MAX_PAGE_SIZE = 250
_SLUG = re.compile(r"[a-z][a-z0-9-]{0,31}", re.ASCII)
_LABEL = re.compile(
    r"[A-Za-z0-9](?:[A-Za-z0-9._~ -]{0,46}[A-Za-z0-9])?",
    re.ASCII,
)
_CURSOR = re.compile(r"[A-Za-z0-9_-]{8,40}", re.ASCII)
_SORT_FIELDS = frozenset({"changed", "label", "sequence"})
_SORT_DIRECTIONS = frozenset({"asc", "desc"})


@dataclass(frozen=True, slots=True)
class SlugSegment:
    value: str


@dataclass(frozen=True, slots=True)
class LabelSegment:
    value: str


@dataclass(frozen=True, slots=True)
class NumberSegment:
    value: int


PathSegment = SlugSegment | LabelSegment | NumberSegment


@dataclass(frozen=True, slots=True)
class PageOrder:
    after: str | None
    take: int
    sort_field: SortField
    direction: SortDirection


@dataclass(frozen=True, slots=True)
class RequestTargetPlan:
    path: str
    query_pairs: tuple[tuple[str, str], ...]
    query: str
    target: str


def _encode_text_segment(
    value: object,
    *,
    grammar: re.Pattern[str],
    kind: str,
) -> str:
    if type(value) is not str:
        raise TypeError(f"{kind} values must be exact strings")
    raw = value.encode("utf-8")
    if not 1 <= len(raw) <= _MAX_RAW_SEGMENT_BYTES:
        raise ValueError(f"{kind} byte length is outside the supported range")
    if value in {".", ".."}:
        raise ValueError("dot path segments are not supported")
    if "/" in value or "\\" in value:
        raise ValueError("path segment values must not contain separators")
    if "%" in value:
        raise ValueError("pre-escaped path segment values are not supported")
    if grammar.fullmatch(value) is None:
        raise ValueError(f"{kind} value does not match its grammar")

    encoded = quote(value, safe="-._~", encoding="utf-8", errors="strict")
    if len(encoded.encode("ascii")) > _MAX_ENCODED_SEGMENT_BYTES:
        raise ValueError(f"encoded {kind} exceeds the supported byte length")
    return encoded


def _encode_segment(segment: object) -> str:
    if type(segment) is SlugSegment:
        return _encode_text_segment(
            segment.value,
            grammar=_SLUG,
            kind="slug segment",
        )
    if type(segment) is LabelSegment:
        return _encode_text_segment(
            segment.value,
            grammar=_LABEL,
            kind="label segment",
        )
    if type(segment) is NumberSegment:
        if type(segment.value) is not int:
            raise TypeError("number segment values must be exact integers")
        if not _MIN_NUMBER <= segment.value <= _MAX_NUMBER:
            raise ValueError("number segment value is outside the supported range")
        encoded = quote(str(segment.value), safe="", encoding="ascii", errors="strict")
        if len(encoded.encode("ascii")) > _MAX_ENCODED_SEGMENT_BYTES:
            raise ValueError("encoded number segment exceeds the supported byte length")
        return encoded
    raise TypeError("path entries must be supported exact segment values")


def _prepare_page_order(value: object) -> tuple[tuple[str, str], ...]:
    if type(value) is not PageOrder:
        raise TypeError("page order must be an exact PageOrder value")
    if type(value.take) is not int:
        raise TypeError("page size must be an exact integer")
    if not 1 <= value.take <= _MAX_PAGE_SIZE:
        raise ValueError("page size is outside the supported range")
    if type(value.sort_field) is not str:
        raise TypeError("sort field must be an exact string")
    if value.sort_field not in _SORT_FIELDS:
        raise ValueError("sort field is not supported")
    if type(value.direction) is not str:
        raise TypeError("sort direction must be an exact string")
    if value.direction not in _SORT_DIRECTIONS:
        raise ValueError("sort direction is not supported")

    pairs = [("take", str(value.take))]
    if value.after is not None:
        if type(value.after) is not str:
            raise TypeError("cursor must be an exact string or None")
        if _CURSOR.fullmatch(value.after) is None:
            raise ValueError("cursor does not match the supported grammar")
        if len(value.after.encode("ascii")) > 40:
            raise ValueError("cursor exceeds the supported byte length")
        pairs.append(("after", value.after))
    pairs.extend(
        (
            ("order", value.sort_field),
            ("direction", value.direction),
        )
    )
    frozen_pairs = tuple(pairs)
    if not 3 <= len(frozen_pairs) <= _MAX_QUERY_PAIRS:
        raise ValueError("query pair count is outside the supported range")
    return frozen_pairs


def build_request_target(
    segments: tuple[PathSegment, ...],
    page_order: PageOrder,
) -> RequestTargetPlan:
    if type(segments) is not tuple:
        raise TypeError("path segments must be an exact tuple")
    if not 1 <= len(segments) <= _MAX_PATH_SEGMENTS:
        raise ValueError("path segment count is outside the supported range")

    encoded_segments = tuple(_encode_segment(segment) for segment in segments)
    path = "/" + "/".join(encoded_segments)
    query_pairs = _prepare_page_order(page_order)
    query = urlencode(
        query_pairs,
        doseq=False,
        safe="",
        encoding="utf-8",
        errors="strict",
        quote_via=quote,
    )
    target = f"{path}?{query}"
    if len(target.encode("ascii")) > _MAX_TARGET_BYTES:
        raise ValueError("request target exceeds the supported byte length")

    return RequestTargetPlan(path, query_pairs, query, target)
```

## Example

```python
segments = (
    SlugSegment("collections"),
    LabelSegment("July set"),
    NumberSegment(73),
)
page_order = PageOrder(
    after="AF3_z9Kq",
    take=40,
    sort_field="changed",
    direction="desc",
)
inputs_before = (segments, page_order)

plan = build_request_target(segments, page_order)

rejected = []
for bad_segments in (
    (SlugSegment("collections"), LabelSegment("../private")),
    (SlugSegment("collections"), NumberSegment(True)),
):
    try:
        build_request_target(bad_segments, page_order)
    except (TypeError, ValueError) as error:
        rejected.append(type(error))

assert (plan, tuple(rejected), (segments, page_order)) == (
    RequestTargetPlan(
        path="/collections/July%20set/73",
        query_pairs=(
            ("take", "40"),
            ("after", "AF3_z9Kq"),
            ("order", "changed"),
            ("direction", "desc"),
        ),
        query="take=40&after=AF3_z9Kq&order=changed&direction=desc",
        target=("/collections/July%20set/73?take=40&after=AF3_z9Kq&order=changed&direction=desc"),
    ),
    (ValueError, TypeError),
    inputs_before,
)
```

## Trade-offs and Limitations

Validation and encoding take time and memory linear in the bounded segment and
target byte counts. Exact tuples, exact frozen segment classes, exact scalar
built-ins, and newly allocated strings and tuples prevent mutable containers or
subclasses from changing the returned plan. The ASCII grammars are intentionally
narrow; labels permit internal spaces and dots but reject dot-only components,
separators, pre-escaped text, Unicode, and empty values.

The query vocabulary, pair order, limits, cursor syntax, and sort fields are a
local contract rather than a universal pagination convention. This function
does not perform a request and makes no claim about headers, bodies,
authentication, retries, responses, TLS, framework routing, or any other
network behavior. An HTTP client still owns those concerns.

## Related Snippets

<!-- catalog:related:start -->
- [Build a Canonical HTTP Origin Key](build-a-canonical-http-origin-key.md)
- [Collect Matching Cursor Pages with an Explicit Page Budget](collect-matching-cursor-pages-with-an-explicit-page-budget.md)
- [Parse a Bounded Component Options Expression](../configuration-serialization/parse-a-bounded-component-options-expression.md)
<!-- catalog:related:end -->
