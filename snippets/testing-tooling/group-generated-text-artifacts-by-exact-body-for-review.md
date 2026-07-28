---
title: "Group Generated Text Artifacts by Exact Body for Review"
snippet_type: "testing-technique"
use_cases:
  - "automation"
  - "testing"
  - "validation"
tested_python:
  - "3.14"
dependencies: []
related:
  - "compare-a-bounded-text-capture-against-a-golden-fixture.md"
  - "../data-processing/group-items-by-an-exact-compatibility-signature-and-report-unmatched-inputs.md"
  - "../storage-databases/store-bytes-by-their-content-digest.md"
---

# Group Generated Text Artifacts by Exact Body for Review

## Idea and Problem

Group already-rendered text artifacts by complete normalized UTF-8 bytes so reviewers inspect each distinct body once without losing its intended output names.

Use a digest only as domain-separated report metadata. Full bytes remain the equality key, so a digest collision cannot merge two different bodies.

## When to Use

Use this after deterministic rendering has produced a bounded in-memory tuple of logical POSIX-style names and text bodies. It is useful for compact test reports, preview tools, and generation-plan reviews where byte-for-byte equality is the intended definition of sameness.

Do not use it as a filesystem writer, a rendering pipeline, or a substitute for content-aware comparison.

## Implementation

```python
import hashlib
import re
from dataclasses import dataclass
from pathlib import PurePosixPath

_NAME_PART_PATTERN = re.compile(r"[A-Za-z0-9][A-Za-z0-9._-]{0,63}", re.ASCII)
_REPORT_DOMAIN = b"generated-text-review:v1\x00"
_MAX_ARTIFACTS = 1_024
_MAX_NAME_BYTES = 160
_MAX_NAME_PARTS = 8
_MAX_BODY_BYTES = 1_048_576
_MAX_TOTAL_BYTES = 8_388_608


@dataclass(frozen=True, slots=True)
class RenderedTextArtifact:
    output_name: str
    text: str


@dataclass(frozen=True, slots=True)
class ExactBodyGroup:
    normalized_text: str
    output_names: tuple[str, ...]
    utf8_bytes: int
    report_id: str


def _safe_relative_name(value: object) -> str:
    if type(value) is not str:
        raise TypeError("output names must be strings")
    try:
        encoded = value.encode("utf-8", errors="strict")
    except UnicodeEncodeError:
        raise ValueError("output names must contain valid UTF-8 text") from None
    if not 1 <= len(encoded) <= _MAX_NAME_BYTES:
        raise ValueError(f"output names must occupy 1 to {_MAX_NAME_BYTES} UTF-8 bytes")

    path = PurePosixPath(value)
    if path.is_absolute() or path.as_posix() != value:
        raise ValueError(f"output name must be a normalized relative path: {value!r}")
    if not 1 <= len(path.parts) <= _MAX_NAME_PARTS:
        raise ValueError(f"output names may contain at most {_MAX_NAME_PARTS} parts")
    if any(
        part in {".", ".."} or _NAME_PART_PATTERN.fullmatch(part) is None for part in path.parts
    ):
        raise ValueError(f"output name contains an unsafe path part: {value!r}")
    return value


def _normalized_body(value: object) -> tuple[str, bytes]:
    if type(value) is not str:
        raise TypeError("artifact text must be a string")
    normalized = value if value.endswith("\n") else value + "\n"
    if len(normalized) > _MAX_BODY_BYTES:
        raise ValueError("an artifact body exceeds the per-body budget")
    try:
        encoded = normalized.encode("utf-8", errors="strict")
    except UnicodeEncodeError:
        raise ValueError("artifact text must contain valid UTF-8 text") from None
    if len(encoded) > _MAX_BODY_BYTES:
        raise ValueError("an artifact body exceeds the per-body UTF-8 budget")
    return normalized, encoded


def _body_report_id(body: bytes) -> str:
    digest = hashlib.sha256()
    digest.update(_REPORT_DOMAIN)
    digest.update(len(body).to_bytes(8, byteorder="big"))
    digest.update(body)
    return f"sha256:{digest.hexdigest()}"


def group_text_artifacts_for_review(
    artifacts: tuple[RenderedTextArtifact, ...],
) -> tuple[ExactBodyGroup, ...]:
    if type(artifacts) is not tuple or not 1 <= len(artifacts) <= _MAX_ARTIFACTS:
        raise ValueError(f"artifacts must be a tuple containing 1 to {_MAX_ARTIFACTS} items")

    checked: list[tuple[str, str, bytes]] = []
    names: set[str] = set()
    total_bytes = 0
    for artifact in artifacts:
        if type(artifact) is not RenderedTextArtifact:
            raise TypeError("every item must be a RenderedTextArtifact")
        name = _safe_relative_name(artifact.output_name)
        if name in names:
            raise ValueError(f"duplicate output name: {name!r}")
        normalized, body = _normalized_body(artifact.text)
        total_bytes += len(body)
        if total_bytes > _MAX_TOTAL_BYTES:
            raise ValueError("artifact bodies exceed the aggregate UTF-8 budget")
        names.add(name)
        checked.append((name, normalized, body))

    names_by_body: dict[bytes, list[str]] = {}
    text_by_body: dict[bytes, str] = {}
    for name, normalized, body in checked:
        names_by_body.setdefault(body, []).append(name)
        text_by_body.setdefault(body, normalized)

    groups = (
        ExactBodyGroup(
            normalized_text=text_by_body[body],
            output_names=tuple(sorted(group_names)),
            utf8_bytes=len(body),
            report_id=_body_report_id(body),
        )
        for body, group_names in names_by_body.items()
    )
    return tuple(sorted(groups, key=lambda group: group.output_names))
```

The normalization rule appends one LF only when the input does not already end in LF. It preserves every existing character, including extra trailing blank lines and a CR before an existing final LF.

## Example

```python
artifacts = (
    RenderedTextArtifact("alpha/report.txt", "mode=fast"),
    RenderedTextArtifact("beta/report.txt", "mode=fast\n"),
    RenderedTextArtifact("gamma/report.txt", "mode=slow\n"),
)

groups = group_text_artifacts_for_review(artifacts)

assert [(group.output_names, group.normalized_text) for group in groups] == [
    (("alpha/report.txt", "beta/report.txt"), "mode=fast\n"),
    (("gamma/report.txt",), "mode=slow\n"),
]
assert [group.utf8_bytes for group in groups] == [
    len(b"mode=fast\n"),
    len(b"mode=slow\n"),
]
assert all(group.report_id.startswith("sha256:") for group in groups)
assert all(len(group.report_id) == 71 for group in groups)
```

## Trade-offs and Limitations

- The function materializes every normalized body before grouping; aggregate accounting includes each intended output even when bodies are equal.
- Only a missing final LF is normalized. Other line endings, trailing blank lines, Unicode normalization forms, and whitespace remain byte-significant.
- SHA-256 identifiers label reports; they neither decide equality nor authenticate content.
- Names are deliberately restricted to a conservative portable subset and are logical names only. No path is opened or written.
- Reports contain full generated text and may expose secrets supplied by the caller.

## Related Snippets

<!-- catalog:related:start -->
- [Compare a Bounded Text Capture Against a Golden Fixture](compare-a-bounded-text-capture-against-a-golden-fixture.md)
- [Group Items by an Exact Compatibility Signature and Report Unmatched Inputs](../data-processing/group-items-by-an-exact-compatibility-signature-and-report-unmatched-inputs.md)
- [Store Bytes by Their Content Digest](../storage-databases/store-bytes-by-their-content-digest.md)
<!-- catalog:related:end -->
