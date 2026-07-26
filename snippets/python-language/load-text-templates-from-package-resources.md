---
title: "Load Text Templates from Package Resources"
snippet_type: recipe
use_cases:
  - resource-management
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../configuration-serialization/get-nested-values-with-a-validated-dot-path.md
  - read-fixed-size-blocks-with-iter-sentinel.md
---

# Load Text Templates from Package Resources

## Idea and Problem

Load a text template from ordered package-resource roots without assuming that packaged data has a persistent filesystem path.

`importlib.resources` exposes files from normal packages, wheels, and compatible
loaders through a traversable interface. Strict POSIX resource names keep
lookups below the configured roots, while ordered roots provide a small and
explicit fallback policy.

## When to Use

Use this recipe when templates ship as package data and runtime source
directories may be absent. The packages and resource roots must be trusted;
only the requested relative template name is treated as input. Use a framework
loader when template inheritance, application discovery, caching, live
overrides, or rendering behavior belongs to that framework.

## Implementation

```python
from dataclasses import dataclass
from importlib import resources
from types import ModuleType


class TemplateResourceError(Exception):
    pass


class InvalidTemplateName(TemplateResourceError):
    pass


class TemplateNotFound(TemplateResourceError):
    pass


def _relative_resource_parts(name: str, *, allow_empty: bool) -> tuple[str, ...]:
    if not isinstance(name, str):
        raise TypeError("resource name must be text")
    if name == "" and allow_empty:
        return ()
    if not name or "\0" in name or "\\" in name:
        raise InvalidTemplateName(name)

    parts = tuple(name.split("/"))
    if any(part in {"", ".", ".."} or ":" in part for part in parts):
        raise InvalidTemplateName(name)
    return parts


@dataclass(frozen=True, slots=True)
class PackageResourceRoot:
    anchor: str | ModuleType
    directory: str = ""

    def __post_init__(self) -> None:
        _relative_resource_parts(self.directory, allow_empty=True)


def load_text_template(
    template_name: str,
    roots: tuple[PackageResourceRoot, ...],
    *,
    encoding: str = "utf-8",
) -> str:
    name_parts = _relative_resource_parts(template_name, allow_empty=False)
    if not isinstance(encoding, str) or not encoding:
        raise TypeError("encoding must be non-empty text")

    for root in roots:
        candidate = resources.files(root.anchor)
        root_parts = _relative_resource_parts(root.directory, allow_empty=True)
        for part in (*root_parts, *name_parts):
            candidate = candidate.joinpath(part)
        if candidate.is_file():
            return candidate.read_text(encoding=encoding)

    raise TemplateNotFound(template_name)
```

## Example

```python
import importlib
import sys
from pathlib import Path
from tempfile import TemporaryDirectory


with TemporaryDirectory() as temporary_directory:
    package = Path(temporary_directory) / "example_assets"
    (package / "overrides" / "emails").mkdir(parents=True)
    (package / "defaults" / "emails").mkdir(parents=True)
    (package / "__init__.py").write_text("", encoding="utf-8")
    (package / "overrides" / "emails" / "welcome.txt").write_text(
        "override",
        encoding="utf-8",
    )
    (package / "defaults" / "emails" / "welcome.txt").write_text(
        "default",
        encoding="utf-8",
    )
    (package / "defaults" / "emails" / "fallback.txt").write_text(
        "fallback",
        encoding="utf-8",
    )
    (package / "defaults" / "emails" / "broken.txt").write_bytes(b"\xff")

    sys.path.insert(0, temporary_directory)
    importlib.invalidate_caches()
    try:
        roots = (
            PackageResourceRoot("example_assets", "overrides"),
            PackageResourceRoot("example_assets", "defaults"),
        )
        preferred = load_text_template("emails/welcome.txt", roots)
        fallback = load_text_template("emails/fallback.txt", roots)

        rejected_names = []
        for name in (
            "",
            "/absolute.txt",
            "../outside.txt",
            "a//b",
            "a\\b",
            "C:outside.txt",
            "C:/outside.txt",
        ):
            try:
                load_text_template(name, roots)
            except InvalidTemplateName:
                rejected_names.append(name)

        try:
            load_text_template("emails/missing.txt", roots)
        except TemplateNotFound:
            missing_rejected = True
        else:
            missing_rejected = False

        try:
            load_text_template("emails/broken.txt", roots)
        except UnicodeDecodeError:
            decoding_error_propagated = True
        else:
            decoding_error_propagated = False
    finally:
        sys.path.remove(temporary_directory)
        sys.modules.pop("example_assets", None)

assert (
    preferred,
    fallback,
    tuple(rejected_names),
    missing_rejected,
    decoding_error_propagated,
) == (
    "override",
    "fallback",
    (
        "",
        "/absolute.txt",
        "../outside.txt",
        "a//b",
        "a\\b",
        "C:outside.txt",
        "C:/outside.txt",
    ),
    True,
    True,
)
```

## Trade-offs and Limitations

Package resources are read-only and may be backed by an archive rather than a
real path. This function reads the whole file, performs no caching or size
limit, and propagates package-import, loader, and decoding failures. Its
portable-name subset rejects backslashes and colon-bearing components, so it
cannot address every filename that is legal on a particular platform. It does
not render templates or discover application-specific overrides. Name
validation protects only the lookup suffix; configured package anchors and
root directories remain trusted code configuration.

## Related Snippets

<!-- catalog:related:start -->
- [Get Nested Values with a Validated Dot Path](../configuration-serialization/get-nested-values-with-a-validated-dot-path.md)
- [Read Fixed-Size Blocks with iter() and a Sentinel](read-fixed-size-blocks-with-iter-sentinel.md)
<!-- catalog:related:end -->
