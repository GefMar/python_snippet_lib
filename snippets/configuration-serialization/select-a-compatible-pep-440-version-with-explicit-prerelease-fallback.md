---
title: "Select a Compatible PEP 440 Version with Explicit Prerelease Fallback"
snippet_type: recipe
use_cases:
  - configuration
  - validation
tested_python:
  - "3.14"
dependencies:
  - name: packaging
    version: '26.2'
related:
  - parse-and-compare-bounded-semantic-versions-2-0-0.md
  - ../algorithms-data-structures/sort-dotted-release-labels-with-an-explicit-last-marker.md
---

# Select a Compatible PEP 440 Version with Explicit Prerelease Fallback

## Idea and Problem

Choose a compatible Python distribution version from a known candidate set while making prerelease fallback explicit.

Parse version labels with `packaging`, and apply the requested specifier to
each version. Prefer the newest matching non-prerelease. Consider the newest
matching prerelease only if the caller enables fallback and no ordinary
release matches.

## When to Use

Use this when selecting one version from a finite, prefiltered list, such as
a local release manifest. The caller supplies a PEP 440 specifier and decides
whether prereleases are eligible as a fallback. This is useful when an
ordinary release should win even if a newer prerelease is available.

## Implementation

```python
from collections.abc import Iterable

from packaging.specifiers import SpecifierSet
from packaging.version import Version


def choose_version(
    labels: Iterable[str],
    requirement: str,
    *,
    allow_prerelease_fallback: bool = False,
) -> Version | None:
    specifier = SpecifierSet(requirement)
    eligible = (
        version
        for version in map(Version, labels)
        if specifier.contains(version, prereleases=allow_prerelease_fallback)
    )
    return max(
        eligible,
        key=lambda version: (not version.is_prerelease, version),
        default=None,
    )
```

## Example

```python
ordinary = ("2.0rc1", "1.9", "1.10", "3.0")
previews = ("2.0rc1", "2.1a1", "2.0rc2")

assert (
    choose_version(ordinary, ">=1,<3", allow_prerelease_fallback=True),
    choose_version(previews, ">=2.0rc1,<2.1"),
    choose_version(previews, ">=2.0rc1,<2.1", allow_prerelease_fallback=True),
) == (Version("1.10"), None, Version("2.0rc2"))
```

## Trade-offs and Limitations

Invalid labels or specifiers produce errors. Enabling prereleases does not
override the specifier's comparison rules. For example, an exclusive upper
bound such as `<2.1` still excludes prereleases of that boundary release in
this example.

This scans each candidate and returns a parsed version, without preserving
its original spelling or choosing among artifacts with equivalent versions.
It does not check wheel tags, yanked status or dependencies of candidates.
Use a package resolver when those constraints matter.

## Related Snippets

<!-- catalog:related:start -->
- [Parse and Compare Bounded Semantic Versions 2.0.0](parse-and-compare-bounded-semantic-versions-2-0-0.md)
- [Sort Dotted Release Labels with an Explicit Last Marker](../algorithms-data-structures/sort-dotted-release-labels-with-an-explicit-last-marker.md)
<!-- catalog:related:end -->
