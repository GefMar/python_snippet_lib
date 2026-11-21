---
title: "Report Duplicate Entry Point Names Without Loading Plugins"
snippet_type: testing-technique
use_cases:
  - interoperability
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../python-language/load-bounded-trusted-extension-factories-by-one-entry-point.md
  - ../python-language/dispatch-named-strategies-with-an-explicit-function-mapping.md
---

# Report Duplicate Entry Point Names Without Loading Plugins

## Idea and Problem

Identify colliding plugin names within one entry point group and report the distribution and target behind each claim.

Two installed distributions can advertise the same name in a group. Looking
up just one match can hide a collision. Group the metadata by exact name
before choosing any plugin. A stable report makes the result independent of
distribution discovery order.

## When to Use

Use this for a startup check or a test of an installed plugin environment when
the host expects a unique provider for every name in one group. Supply
`importlib.metadata.distributions()` for the current environment or restrict
the metadata search to known directories. Use valid distribution metadata.

## Implementation

```python
from collections import defaultdict
from collections.abc import Iterable
from importlib.metadata import Distribution


type Provider = tuple[str, str, str]  # Distribution name, version, and target.


def report_entry_point_collisions(
    packages: Iterable[Distribution], *, group: str
) -> dict[str, tuple[Provider, ...]]:
    providers_by_name: dict[str, list[Provider]] = defaultdict(list)
    for package in packages:
        for entry_point in package.entry_points.select(group=group):
            providers_by_name[entry_point.name].append(
                (package.name, package.version, entry_point.value)
            )

    return {
        name: tuple(sorted(providers))
        for name, providers in sorted(providers_by_name.items())
        if len(providers) > 1
    }
```

## Example

```python
from importlib.metadata import distributions
from pathlib import Path
from tempfile import TemporaryDirectory


def write_metadata(root: Path, name: str, version: str, entry_points: str) -> None:
    info = root / f"{name.replace('-', '_')}-{version}.dist-info"
    info.mkdir()
    (info / "METADATA").write_text(
        f"Metadata-Version: 2.1\nName: {name}\nVersion: {version}\n",
        encoding="utf-8",
    )
    (info / "entry_points.txt").write_text(entry_points, encoding="utf-8")


group = "document.renderers"
with TemporaryDirectory() as temporary:
    root = Path(temporary)
    write_metadata(
        root,
        "beta-renderer",
        "2.0",
        "[document.renderers]\npreview = beta_demo:render\nPreview = beta_demo:case\n",
    )
    write_metadata(
        root,
        "alpha-renderer",
        "1.0",
        "[document.renderers]\npreview = alpha_demo:render\ntext = alpha_demo:plain\n"
        "[document.transforms]\nPreview = alpha_demo:transform\n",
    )
    packages = tuple(distributions(path=[str(root)]))
    first = report_entry_point_collisions(packages, group=group)
    second = report_entry_point_collisions(reversed(packages), group=group)

expected = {
    "preview": (
        ("alpha-renderer", "1.0", "alpha_demo:render"),
        ("beta-renderer", "2.0", "beta_demo:render"),
    ),
}
assert first == second == expected
```

## Trade-offs and Limitations

The report compares names and groups case-sensitively. A host that normalizes
names needs its own collision policy. Every discovered claim participates,
even when two claims have the same target text. The report leaves provider
selection to the host.

This reads metadata without calling `EntryPoint.load()`. The fixture has no
importable plugin modules. A report cannot establish that a target imports,
has the required interface, or can be trusted. Gathering claims takes memory
proportional to the number of entry points in the group. Prefer an isolated
environment for reproducible startup checks.

## Related Snippets

<!-- catalog:related:start -->
- [Load Bounded Trusted Extension Factories by One Entry Point](../python-language/load-bounded-trusted-extension-factories-by-one-entry-point.md)
- [Dispatch Named Strategies with an Explicit Function Mapping](../python-language/dispatch-named-strategies-with-an-explicit-function-mapping.md)
<!-- catalog:related:end -->
