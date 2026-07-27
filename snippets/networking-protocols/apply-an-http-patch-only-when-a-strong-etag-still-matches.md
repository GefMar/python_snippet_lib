---
title: "Apply an HTTP Patch Only When a Strong ETag Still Matches"
snippet_type: integration
use_cases:
  - networking
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - resume-a-bounded-http-byte-stream-with-validated-range-responses.md
  - ../storage-databases/compare-and-swap-a-versioned-sqlite-setting-with-one-conditional-update.md
  - ../testing-tooling/verify-ordered-http-client-expectations-with-bounded-mismatch-reports.md
---

# Apply an HTTP Patch Only When a Strong ETag Still Matches

## Idea and Problem

Read one bounded resource and apply at most one patch guarded by its strong ETag so a stale client cannot silently overwrite a newer representation.

The HTTP adapter owns transport details, while the workflow owns intent: one
read, an immutable flat snapshot for a side-effect-free mutation callback, and
one conditional patch carrying the exact opaque validator in `If-Match`. A
precondition failure becomes an explicit conflict outcome and is never retried
without a new user decision.

## When to Use

Use this integration boundary when an HTTP API publishes strong entity tags and
enforces `If-Match` atomically for state-changing requests. The resource and
intended changes must fit a small flat JSON object, and the caller must be able
to treat a conflict as a normal outcome.

Use a server-defined revision field when that is the documented concurrency
contract. Use an application transaction when several resources must change
atomically. Do not emulate optimistic concurrency with a read followed by an
unconditional patch.

## Implementation

```python
import json
import math
import re
from collections.abc import Mapping
from dataclasses import dataclass
from enum import StrEnum
from types import MappingProxyType
from typing import Protocol, TypeAlias


JsonScalar: TypeAlias = str | int | float | bool | None
_MAX_FIELDS = 64
_MAX_KEY_CHARACTERS = 64
_MAX_TEXT_CHARACTERS = 2_048
_MAX_BODY_BYTES = 32 * 1024
_MAX_ETAG_CHARACTERS = 256
_PATH = re.compile(r"/[A-Za-z0-9._~!$&'()*+,;=:@%/-]{0,511}", re.ASCII)
_KEY = re.compile(r"[A-Za-z][A-Za-z0-9._-]{0,63}", re.ASCII)
_STRONG_ETAG = re.compile(r'"[\x21\x23-\x7e]*"', re.ASCII)


@dataclass(frozen=True, slots=True)
class ResourceSnapshot:
    etag: str
    fields: Mapping[str, JsonScalar]


@dataclass(frozen=True, slots=True)
class PatchReply:
    status: int


class ConditionalPatchAdapter(Protocol):
    def read(self, path: str) -> ResourceSnapshot:
        ...

    def patch(
        self,
        path: str,
        changes: Mapping[str, JsonScalar],
        *,
        if_match: str,
    ) -> PatchReply:
        ...


class PatchMutation(Protocol):
    def __call__(
        self,
        current: Mapping[str, JsonScalar],
        /,
    ) -> Mapping[str, JsonScalar]:
        ...


class PatchState(StrEnum):
    UPDATED = "updated"
    CONFLICT = "conflict"


@dataclass(frozen=True, slots=True)
class ConditionalPatchResult:
    state: PatchState
    status: int
    matched_etag: str


class UnexpectedPatchStatus(RuntimeError):
    pass


def _validated_etag(value: object) -> str:
    if (
        not isinstance(value, str)
        or len(value) > _MAX_ETAG_CHARACTERS
        or _STRONG_ETAG.fullmatch(value) is None
    ):
        raise ValueError("a bounded strong ETag is required")
    return value


def _validated_fields(value: object, *, name: str) -> dict[str, JsonScalar]:
    if not isinstance(value, Mapping):
        raise TypeError(f"{name} must be a mapping")
    if not 1 <= len(value) <= _MAX_FIELDS:
        raise ValueError(f"{name} field count is outside the supported range")

    result: dict[str, JsonScalar] = {}
    for key, item in value.items():
        if not isinstance(key, str) or _KEY.fullmatch(key) is None:
            raise ValueError(f"{name} contains a non-canonical field name")
        if isinstance(item, str):
            if len(item) > _MAX_TEXT_CHARACTERS:
                raise ValueError(f"{name} contains oversized text")
        elif isinstance(item, bool) or item is None:
            pass
        elif isinstance(item, int):
            if not -(2**53) <= item <= 2**53:
                raise ValueError(f"{name} contains an out-of-range integer")
        elif isinstance(item, float):
            if not math.isfinite(item):
                raise ValueError(f"{name} contains a non-finite number")
        else:
            raise TypeError(f"{name} values must be flat JSON scalars")
        result[key] = item

    encoded = json.dumps(
        result,
        ensure_ascii=False,
        allow_nan=False,
        sort_keys=True,
        separators=(",", ":"),
    ).encode("utf-8")
    if len(encoded) > _MAX_BODY_BYTES:
        raise ValueError(f"{name} exceeds the encoded byte limit")
    return result


def apply_conditional_patch(
    adapter: ConditionalPatchAdapter,
    path: str,
    mutate: PatchMutation,
) -> ConditionalPatchResult:
    if not isinstance(path, str) or _PATH.fullmatch(path) is None:
        raise ValueError("path is outside the conservative absolute-path format")
    if not callable(mutate):
        raise TypeError("mutate must be callable")

    snapshot = adapter.read(path)
    if not isinstance(snapshot, ResourceSnapshot):
        raise TypeError("read must return a ResourceSnapshot")
    etag = _validated_etag(snapshot.etag)
    current = _validated_fields(snapshot.fields, name="resource")

    proposed = mutate(MappingProxyType(current))
    changes = _validated_fields(proposed, name="changes")
    reply = adapter.patch(path, changes, if_match=etag)
    if not isinstance(reply, PatchReply):
        raise TypeError("patch must return a PatchReply")
    if isinstance(reply.status, bool) or not isinstance(reply.status, int):
        raise TypeError("patch status must be an integer")
    if reply.status == 412:
        state = PatchState.CONFLICT
    elif reply.status in (200, 204):
        state = PatchState.UPDATED
    else:
        raise UnexpectedPatchStatus(f"unexpected PATCH status {reply.status}")
    return ConditionalPatchResult(
        state=state,
        status=reply.status,
        matched_etag=etag,
    )
```

## Example

```python
class MemoryAdapter:
    def __init__(self, *, conflict: bool = False) -> None:
        self.conflict = conflict
        self.reads = 0
        self.patches: list[tuple[str, dict[str, JsonScalar], str]] = []

    def read(self, path: str) -> ResourceSnapshot:
        self.reads += 1
        return ResourceSnapshot('"revision-7"', {"label": "draft", "priority": 2})

    def patch(
        self,
        path: str,
        changes: Mapping[str, JsonScalar],
        *,
        if_match: str,
    ) -> PatchReply:
        self.patches.append((path, dict(changes), if_match))
        return PatchReply(412 if self.conflict else 204)


updated_adapter = MemoryAdapter()
updated = apply_conditional_patch(
    updated_adapter,
    "/records/sample",
    lambda current: {"label": "ready", "priority": current["priority"]},
)
conflict = apply_conditional_patch(
    MemoryAdapter(conflict=True),
    "/records/sample",
    lambda _current: {"label": "ready"},
)

assert (
    updated.state,
    updated_adapter.reads,
    updated_adapter.patches,
    conflict.state,
) == (
    PatchState.UPDATED,
    1,
    [("/records/sample", {"label": "ready", "priority": 2}, '"revision-7"')],
    PatchState.CONFLICT,
)
```

## Trade-offs and Limitations

Correctness depends on the server applying the strong validator atomically.
The adapter must preserve the ETag exactly and map the server's precondition
failure to status 412. This narrow contract accepts only 200 or 204 as a
completed update; asynchronous or multi-status responses require another
outcome model. Some APIs use another documented conflict status or a body
revision instead; follow that contract rather than assuming HTTP details.

The mutation callback can still perform external side effects even though its
input mapping is read-only, so callers must keep it pure. A conflict does not
prove which field changed or whether the same intent is still valid. This
helper deliberately performs no retry, merge, cache lookup, authentication,
deadline, or transport-error translation, and its flat JSON boundary is not a
general JSON Patch or Merge Patch implementation.

## Related Snippets

<!-- catalog:related:start -->
- [Resume a Bounded HTTP Byte Stream with Validated Range Responses](resume-a-bounded-http-byte-stream-with-validated-range-responses.md)
- [Compare and Swap a Versioned SQLite Setting with One Conditional Update](../storage-databases/compare-and-swap-a-versioned-sqlite-setting-with-one-conditional-update.md)
- [Verify Ordered HTTP Client Expectations with Bounded Mismatch Reports](../testing-tooling/verify-ordered-http-client-expectations-with-bounded-mismatch-reports.md)
<!-- catalog:related:end -->
