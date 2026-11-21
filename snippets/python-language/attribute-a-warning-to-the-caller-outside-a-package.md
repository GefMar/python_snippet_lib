---
title: "Attribute a Warning to the Caller Outside a Package"
snippet_type: recipe
use_cases:
  - observability
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - keep-a-compatibility-wrapper-with-warnings-deprecated.md
---

# Attribute a Warning to the Caller Outside a Package

## Idea and Problem

Point a warning at the code calling a package even when internal wrapper depth varies between entry points.

A fixed `stacklevel` can point to an internal wrapper on one path and to the
external caller on another. `warnings.warn()` can skip frames by filename
prefix. Include the directory separator in the package prefix so neighboring
directories with similar names still count as external.

## When to Use

Use this for a warning emitted through several layers of a library, such as
when callers choose a legacy output format. Pass the trusted package root in
the same absolute filename spelling as the running code. For a single stable
wrapper depth, a fixed `stacklevel` may be sufficient.

## Implementation

```python
import os
import warnings


def warn_legacy_export(package_directory: str) -> None:
    package_prefix = os.path.join(package_directory, "")
    warnings.warn(
        "Use the current table export format.",
        DeprecationWarning,
        stacklevel=2,
        skip_file_prefixes=(package_prefix,),
    )
```

## Example

```python
from pathlib import Path


package_directory = Path.cwd() / "export_tools"
caller_file = package_directory.with_name("export_tools_cli") / "report.py"
namespace = {
    "warn_legacy_export": warn_legacy_export,
    "package_directory": str(package_directory),
}
wrappers = """\
def export_direct():
    warn_legacy_export(package_directory)

def export_indirect():
    export_direct()
"""

# Synthetic filenames exercise the stack lookup without creating files.
exec(compile(wrappers, str(package_directory / "api.py"), "exec"), namespace)
calls = "export_direct()\nexport_indirect()\n"
with warnings.catch_warnings(record=True) as recorded:
    warnings.simplefilter("always", DeprecationWarning)
    exec(compile(calls, str(caller_file), "exec"), namespace)

assert [(item.filename, item.lineno, item.category) for item in recorded] == [
    (str(caller_file), 1, DeprecationWarning),
    (str(caller_file), 2, DeprecationWarning),
]
```

## Trade-offs and Limitations

`skip_file_prefixes` is available from Python 3.12. It matches the filenames
recorded in stack frames as strings. It does not resolve symlinks or infer
package ownership, so configure matching prefixes explicitly if a loader uses
different filename spellings. A wrapper outside those prefixes counts as an
external caller even if another application frame appears higher on the stack.

This changes attribution and therefore can change which warning filters match.
Normal warning filters still apply. The example uses a scoped `always` filter
so both calls are observable.

## Related Snippets

<!-- catalog:related:start -->
- [Keep a Compatibility Wrapper with warnings.deprecated](keep-a-compatibility-wrapper-with-warnings-deprecated.md)
<!-- catalog:related:end -->
