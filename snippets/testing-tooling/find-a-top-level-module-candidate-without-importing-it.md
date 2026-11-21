---
title: "Find a Top-Level Module Candidate Without Importing It"
snippet_type: testing-technique
use_cases:
  - interoperability
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - report-duplicate-entry-point-names-without-loading-plugins.md
  - ../python-language/load-bounded-trusted-extension-factories-by-one-entry-point.md
---

# Find a Top-Level Module Candidate Without Importing It

## Idea and Problem

Identify the candidate that Python's path-based finder selects for a top-level module name in an explicit directory order.

Two import roots can contain modules with the same name. Calling
`PathFinder.find_spec()` returns the selected module specification without
asking its loader to execute the module. Inspect the specification to see
where lookup would go and to distinguish a namespace package from a missing
result.

## When to Use

Use this in a controlled test or a packaging diagnostic when you have
trusted search roots and want to check which one takes precedence. Pass
absolute directories in the order you want to test. Supply one top-level
identifier, such as `toolkit`, without a package prefix.

## Implementation

```python
from importlib.machinery import ModuleSpec, PathFinder
from pathlib import Path


def find_path_candidate(name: str, roots: tuple[Path, ...]) -> ModuleSpec | None:
    if not name.isidentifier():
        raise ValueError("name must be a top-level module identifier")
    return PathFinder.find_spec(name, [str(root) for root in roots])
```

## Example

```python
import sys
from tempfile import TemporaryDirectory


module_name = "fixture_adapter"
namespace_name = "fixture_assets"
missing_name = "missing_fixture_candidate"

with TemporaryDirectory() as temporary:
    directory = Path(temporary)
    first, second = directory / "first", directory / "second"
    roots = (first, second)
    for root in roots:
        root.mkdir()
        (root / f"{module_name}.py").write_text(
            'raise AssertionError("fixture module must not execute")\n', encoding="utf-8"
        )
        (root / namespace_name).mkdir()

    cache = sys.path_importer_cache
    keys = tuple(map(str, roots))
    saved = {key: cache[key] for key in keys if key in cache}
    try:
        preferred = find_path_candidate(module_name, roots)
        swapped = find_path_candidate(module_name, (second, first))
        namespace = find_path_candidate(namespace_name, roots)
        missing = find_path_candidate(missing_name, roots)

        assert preferred is not None and preferred.origin == str(first / f"{module_name}.py")
        assert swapped is not None and swapped.origin == str(second / f"{module_name}.py")
        assert namespace is not None and namespace.origin is None
        assert tuple(namespace.submodule_search_locations or ()) == (
            str(first / namespace_name),
            str(second / namespace_name),
        )
    finally:
        for key in keys:
            cache.pop(key, None)
        cache.update(saved)

assert {key: cache[key] for key in keys if key in cache} == saved
assert missing is None and all(name not in sys.modules for name in (module_name, namespace_name))
```

## Trade-offs and Limitations

This asks only the path-based finder. It does not consult loaded modules in
`sys.modules` or dispatch to other meta-path finders, so it cannot predict
every result of an actual `import`. In the ordinary filesystem case, a
namespace package has search locations and no file origin.

The finder can invoke registered path hooks and use or populate
`sys.path_importer_cache`. Use trusted interpreter hooks and trusted paths.
The test restores cache entries for the two temporary directories it owns.
When checking files created during a live process, account for cached
directory listings before interpreting a missing result.

## Related Snippets

<!-- catalog:related:start -->
- [Report Duplicate Entry Point Names Without Loading Plugins](report-duplicate-entry-point-names-without-loading-plugins.md)
- [Load Bounded Trusted Extension Factories by One Entry Point](../python-language/load-bounded-trusted-extension-factories-by-one-entry-point.md)
<!-- catalog:related:end -->
