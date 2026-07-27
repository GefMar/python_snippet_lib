---
title: "Audit Parsed Compose Services for Digest-Pinned Images"
snippet_type: recipe
use_cases:
  - configuration
  - security
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - authenticate-bounded-payloads-with-versioned-hmac-keys.md
  - ../storage-databases/fingerprint-a-bounded-flat-file-set-with-framed-sha-256.md
  - ../configuration-serialization/parse-a-bounded-component-options-expression.md
---

# Audit Parsed Compose Services for Digest-Pinned Images

## Idea and Problem

Audit already parsed Compose service records with a conservative rule that accepts each declared image only when it ends in one lowercase SHA-256 digest pin.

Mutable tags can resolve to different image content over time. This passive
check distinguishes a build-only service, which may omit `image`, from a
service that declares an image reference. An image remains subject to the pin
rule when `build` is also declared because Compose can still use that image
according to its pull policy.

## When to Use

Use this at a configuration boundary after a trusted parser has reduced each
service to its name, optional literal image text, and whether `build` is
declared. It fits offline reviews that intentionally require a narrow
`@sha256:` shape. Resolve or disable Compose interpolation before constructing
the records; this function rejects dollar signs instead of consulting the
environment.

## Implementation

```python
import re
from dataclasses import dataclass


_MAX_SERVICES = 256
_MAX_SERVICE_NAME = 128
_MAX_IMAGE_TEXT = 512
_SERVICE_NAME = re.compile(r"[A-Za-z0-9][A-Za-z0-9_.-]{0,127}", re.ASCII)
_PINNED_IMAGE = re.compile(r"[^@\s]+@sha256:[0-9a-f]{64}", re.ASCII)


@dataclass(frozen=True, slots=True)
class ComposeService:
    name: str
    image: str | None
    build_declared: bool


@dataclass(frozen=True, slots=True)
class ImagePinViolation:
    service_name: str
    code: str


def _validate_service(service: ComposeService) -> None:
    if type(service.name) is not str:
        raise TypeError("service name must be an exact str")
    if not 1 <= len(service.name) <= _MAX_SERVICE_NAME:
        raise ValueError("service name length is outside the supported range")
    if _SERVICE_NAME.fullmatch(service.name) is None:
        raise ValueError("service name contains unsupported characters")
    if type(service.build_declared) is not bool:
        raise TypeError("build_declared must be an exact bool")

    if service.image is None:
        return
    if type(service.image) is not str:
        raise TypeError("image must be an exact str or None")
    if not 1 <= len(service.image) <= _MAX_IMAGE_TEXT:
        raise ValueError("image length is outside the supported range")
    if (
        service.image != service.image.strip()
        or not service.image.isascii()
        or not service.image.isprintable()
        or any(character.isspace() for character in service.image)
    ):
        raise ValueError("image contains malformed text")
    if "$" in service.image:
        raise ValueError("image interpolation is not accepted")


def audit_compose_image_pins(
    services: tuple[ComposeService, ...],
) -> tuple[ImagePinViolation, ...]:
    if type(services) is not tuple:
        raise TypeError("services must be an exact tuple")
    if len(services) > _MAX_SERVICES:
        raise ValueError("service limit exceeded")

    seen_names: set[str] = set()
    for service in services:
        if type(service) is not ComposeService:
            raise TypeError("services must contain exact ComposeService records")
        _validate_service(service)
        if service.name in seen_names:
            raise ValueError("duplicate service name")
        seen_names.add(service.name)

    violations: list[ImagePinViolation] = []
    for service in services:
        if service.image is None:
            if not service.build_declared:
                violations.append(
                    ImagePinViolation(service.name, "missing-image-and-build")
                )
        elif _PINNED_IMAGE.fullmatch(service.image) is None:
            violations.append(
                ImagePinViolation(service.name, "image-not-digest-pinned")
            )

    return tuple(violations)
```

## Example

```python
digest = "a" * 64
services = (
    ComposeService(
        "gateway",
        f"registry.example/gateway@sha256:{digest}",
        False,
    ),
    ComposeService("asset-builder", None, True),
    ComposeService("worker", "registry.example/worker:latest", True),
    ComposeService("worker-copy", "registry.example/worker:latest", False),
)
snapshot = tuple((item.name, item.image, item.build_declared) for item in services)

violations = audit_compose_image_pins(services)

interpolation_rejected = False
try:
    audit_compose_image_pins(
        (ComposeService("database", "${DATABASE_IMAGE}", False),)
    )
except ValueError:
    interpolation_rejected = True

assert (
    violations
    == (
        ImagePinViolation("worker", "image-not-digest-pinned"),
        ImagePinViolation("worker-copy", "image-not-digest-pinned"),
    )
    and tuple((item.name, item.image, item.build_declared) for item in services)
    == snapshot
    and interpolation_rejected
)
```

## Trade-offs and Limitations

This is a shape audit, not full OCI image-reference validation. A matching
digest identifies content but does not authenticate its publisher or prove
that the content is safe. Build-only services pass without an image pin because
there is no declared image reference to inspect; build inputs and outputs need
a separate policy. At most one frozen violation is returned per bounded service.
The ASCII grammar also deliberately rejects interpolation and unusual references
rather than resolving them. The function does not read YAML, inspect Docker,
contact a registry, access the environment, or evaluate pull policies.

## Related Snippets

<!-- catalog:related:start -->
- [Authenticate Bounded Payloads with Versioned HMAC Keys](authenticate-bounded-payloads-with-versioned-hmac-keys.md)
- [Fingerprint a Bounded Flat File Set with Framed SHA-256](../storage-databases/fingerprint-a-bounded-flat-file-set-with-framed-sha-256.md)
- [Parse a Bounded Component Options Expression](../configuration-serialization/parse-a-bounded-component-options-expression.md)
<!-- catalog:related:end -->
