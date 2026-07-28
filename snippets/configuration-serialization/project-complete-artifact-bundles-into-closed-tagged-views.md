---
title: "Project Complete Artifact Bundles into Closed Tagged Views"
snippet_type: recipe
use_cases:
  - configuration
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - build-a-deterministic-size-capped-ustar-archive-from-bytes.md
  - ../data-processing/project-bounded-records-into-multiple-closed-output-schemas.md
  - ../testing-tooling/group-generated-text-artifacts-by-exact-body-for-review.md
---

# Project Complete Artifact Bundles into Closed Tagged Views

## Idea and Problem

Validate a bounded tuple of complete immutable artifact bundles, then project their passive byte payloads into frozen views for every member of a closed tag enum.

A complete preflight rejects malformed identities, ownership mismatches,
duplicate tags, missing companions, budget overruns, and global identity or
projection-key collisions before any view objects are constructed.

## When to Use

Use this recipe when a trusted caller has already assembled a small in-memory
collection of companion artifacts and downstream code needs deterministic
tag-specific views. Bundle order is retained, and the filtering pass preserves
the declaration order of artifacts within each bundle.

The payload model is exact `bytes`, not a recursively frozen object graph.
Text identities use a deliberately narrow ASCII grammar so joining them into a
projection key is unambiguous.

## Implementation

```python
import re
from dataclasses import dataclass
from enum import StrEnum

_IDENTITY = re.compile(r"[a-z][a-z0-9_-]{0,47}", re.ASCII).fullmatch
_MAX_BUNDLES = 64
_MAX_ARTIFACTS = 192
_MAX_IDENTITY_TEXT_BYTES = 8_192
_MAX_PAYLOAD_BYTES = 1_048_576
_MAX_TOTAL_PAYLOAD_BYTES = 16_777_216


class ArtifactTag(StrEnum):
    MANIFEST = "manifest"
    CONTENT = "content"
    NOTICE = "notice"


@dataclass(frozen=True, slots=True)
class Artifact:
    bundle_id: str
    artifact_id: str
    tag: ArtifactTag
    payload: bytes


@dataclass(frozen=True, slots=True)
class ArtifactBundle:
    bundle_id: str
    artifacts: tuple[Artifact, ...]


@dataclass(frozen=True, slots=True)
class ProjectedArtifact:
    bundle_id: str
    artifact_id: str
    projection_key: str
    payload: bytes


@dataclass(frozen=True, slots=True)
class TaggedView:
    tag: ArtifactTag
    artifacts: tuple[ProjectedArtifact, ...]


def _identity(value: object, *, label: str) -> str:
    if type(value) is not str or _IDENTITY(value) is None:
        raise ValueError(f"{label} must use conservative ASCII syntax")
    return value


def project_artifact_bundles(
    bundles: tuple[ArtifactBundle, ...],
) -> tuple[TaggedView, ...]:
    if type(bundles) is not tuple:
        raise TypeError("bundles must be a tuple")
    if not 1 <= len(bundles) <= _MAX_BUNDLES:
        raise ValueError("bundle count is outside the supported range")

    required_tags = tuple(ArtifactTag)
    required_tag_set = frozenset(required_tags)
    checked: list[tuple[str, tuple[Artifact, ...]]] = []
    bundle_ids: set[str] = set()
    artifact_ids: set[str] = set()
    projection_keys: set[str] = set()
    identity_text_bytes = 0
    total_payload_bytes = 0
    artifact_count = 0

    for bundle in bundles:
        if type(bundle) is not ArtifactBundle:
            raise TypeError("bundles must contain exact ArtifactBundle values")
        bundle_id = _identity(bundle.bundle_id, label="bundle ID")
        if bundle_id in bundle_ids:
            raise ValueError("bundle IDs must be globally unique")
        if type(bundle.artifacts) is not tuple:
            raise TypeError("bundle artifacts must be a tuple")
        if len(bundle.artifacts) != len(required_tags):
            raise ValueError("every bundle must contain the exact required tag count")

        identity_text_bytes += len(bundle_id.encode("ascii"))
        seen_tags: set[ArtifactTag] = set()
        checked_artifacts: list[Artifact] = []
        for artifact in bundle.artifacts:
            if type(artifact) is not Artifact:
                raise TypeError("bundles must contain exact Artifact values")
            owner_id = _identity(artifact.bundle_id, label="artifact bundle ID")
            artifact_id = _identity(artifact.artifact_id, label="artifact ID")
            if owner_id != bundle_id:
                raise ValueError("artifact membership must match its containing bundle")
            if artifact_id in artifact_ids:
                raise ValueError("artifact IDs must be globally unique")
            if type(artifact.tag) is not ArtifactTag:
                raise TypeError("artifact tags must be exact ArtifactTag values")
            if artifact.tag in seen_tags:
                raise ValueError("a bundle cannot repeat an artifact tag")
            if type(artifact.payload) is not bytes:
                raise TypeError("artifact payloads must be exact immutable bytes")
            if len(artifact.payload) > _MAX_PAYLOAD_BYTES:
                raise ValueError("an artifact exceeds the payload byte budget")

            projection_key = f"{bundle_id}/{artifact.tag.value}/{artifact_id}"
            if projection_key in projection_keys:
                raise ValueError("artifact projection keys must be globally unique")

            artifact_count += 1
            if artifact_count > _MAX_ARTIFACTS:
                raise ValueError("artifact count exceeds the supported limit")
            identity_text_bytes += len(owner_id.encode("ascii"))
            identity_text_bytes += len(artifact_id.encode("ascii"))
            if identity_text_bytes > _MAX_IDENTITY_TEXT_BYTES:
                raise ValueError("artifact identities exceed the text byte budget")
            total_payload_bytes += len(artifact.payload)
            if total_payload_bytes > _MAX_TOTAL_PAYLOAD_BYTES:
                raise ValueError("artifacts exceed the aggregate payload byte budget")

            artifact_ids.add(artifact_id)
            projection_keys.add(projection_key)
            seen_tags.add(artifact.tag)
            checked_artifacts.append(artifact)

        if frozenset(seen_tags) != required_tag_set:
            raise ValueError("every bundle must contain each required tag exactly once")
        bundle_ids.add(bundle_id)
        checked.append((bundle_id, tuple(checked_artifacts)))

    return tuple(
        TaggedView(
            tag=tag,
            artifacts=tuple(
                ProjectedArtifact(
                    bundle_id=bundle_id,
                    artifact_id=artifact.artifact_id,
                    projection_key=(f"{bundle_id}/{artifact.tag.value}/{artifact.artifact_id}"),
                    payload=artifact.payload,
                )
                for bundle_id, artifacts in checked
                for artifact in artifacts
                if artifact.tag is tag
            ),
        )
        for tag in required_tags
    )
```

The returned tuple follows enum declaration order. Within each `TaggedView`,
the comprehension traverses bundles first and each bundle's tuple second, so
both ordering guarantees are explicit even though tag completeness permits one
artifact per tag in each bundle.

## Example

```python
bundles = (
    ArtifactBundle(
        bundle_id="aurora",
        artifacts=(
            Artifact("aurora", "aurora_body", ArtifactTag.CONTENT, b"alpha\n"),
            Artifact("aurora", "aurora_note", ArtifactTag.NOTICE, b"internal\n"),
            Artifact("aurora", "aurora_map", ArtifactTag.MANIFEST, b"v=1\n"),
        ),
    ),
    ArtifactBundle(
        bundle_id="boreal",
        artifacts=(
            Artifact("boreal", "boreal_map", ArtifactTag.MANIFEST, b"v=2\n"),
            Artifact("boreal", "boreal_body", ArtifactTag.CONTENT, b"beta\n"),
            Artifact("boreal", "boreal_note", ArtifactTag.NOTICE, b"public\n"),
        ),
    ),
)

views = project_artifact_bundles(bundles)

assert [(view.tag, [artifact.artifact_id for artifact in view.artifacts]) for view in views] == [
    (ArtifactTag.MANIFEST, ["aurora_map", "boreal_map"]),
    (ArtifactTag.CONTENT, ["aurora_body", "boreal_body"]),
    (ArtifactTag.NOTICE, ["aurora_note", "boreal_note"]),
]
```

## Trade-offs and Limitations

- Complete bundles are intentionally rigid: adding a tag changes the required companion set for every caller.
- IDs are globally unique even though projection keys also contain bundle IDs; this stronger rule keeps identity comparisons independent of a view.
- Payload and text accounting measures supplied bytes, not object overhead, and the function fully materializes all projected views.
- Artifacts are passive values. The recipe has no mutable or global registry, recursion, callbacks, rendering, import or plugin discovery, dispatch, push or deployment behavior, filesystem access, or network access.

## Related Snippets

<!-- catalog:related:start -->
- [Build a Deterministic Size-Capped USTAR Archive from Bytes](build-a-deterministic-size-capped-ustar-archive-from-bytes.md)
- [Project Bounded Records into Multiple Closed Output Schemas](../data-processing/project-bounded-records-into-multiple-closed-output-schemas.md)
- [Group Generated Text Artifacts by Exact Body for Review](../testing-tooling/group-generated-text-artifacts-by-exact-body-for-review.md)
<!-- catalog:related:end -->
