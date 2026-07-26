---
title: "Check Whether a Generated File Is Older Than Its Inputs"
snippet_type: recipe
use_cases:
  - automation
  - caching
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - replace-a-file-atomically-with-a-sibling-temporary-file.md
  - store-bytes-by-their-content-digest.md
---

# Check Whether a Generated File Is Older Than Its Inputs

## Idea and Problem

Decide whether a generated file needs rebuilding by comparing its modification time with every declared input without changing any filesystem metadata.

A missing output is stale, while an existing output remains fresh when its
timestamp equals or exceeds all input timestamps. Materializing and validating
the complete input collection first prevents an absent output from hiding a
missing source or an unsupported file type.

## When to Use

Use this recipe for a local build or export whose inputs and output are regular
files on a trusted, stable filesystem. The inputs must form a finite, non-empty
iterable, and modification time must be an acceptable freshness signal. Prefer
content digests or a build system with dependency tracking when metadata can be
preserved, rounded, or changed independently of file content.

## Implementation

```python
import stat
from collections.abc import Iterable
from pathlib import Path


def _regular_file_mtime_ns(path: Path, *, role: str) -> int:
    status = path.stat(follow_symlinks=False)
    if not stat.S_ISREG(status.st_mode):
        raise ValueError(f"{role} must be a regular non-link file: {path}")
    return status.st_mtime_ns


def generated_file_is_stale(output: Path, inputs: Iterable[Path]) -> bool:
    if not isinstance(output, Path):
        raise TypeError("output must be a pathlib.Path")

    input_paths = tuple(inputs)
    if not input_paths:
        raise ValueError("inputs must contain at least one path")
    if any(not isinstance(path, Path) for path in input_paths):
        raise TypeError("inputs must contain only pathlib.Path values")

    input_mtimes = tuple(
        _regular_file_mtime_ns(path, role="input") for path in input_paths
    )
    try:
        output_mtime = _regular_file_mtime_ns(output, role="output")
    except FileNotFoundError:
        return True

    return any(input_mtime > output_mtime for input_mtime in input_mtimes)
```

## Example

```python
import os
from pathlib import Path
from tempfile import TemporaryDirectory


with TemporaryDirectory() as directory:
    root = Path(directory)
    first_input = root / "source-a.txt"
    second_input = root / "source-b.txt"
    output = root / "summary.txt"
    first_input.write_text("alpha", encoding="utf-8")
    second_input.write_text("beta", encoding="utf-8")
    output.write_text("alpha beta", encoding="utf-8")

    equal_time = 1_700_000_000_000_000_000
    for path in (first_input, second_input, output):
        os.utime(path, ns=(equal_time, equal_time))

    equal_timestamp_is_fresh = not generated_file_is_stale(
        output,
        (first_input, second_input),
    )

    newer_time = equal_time + 2_000_000_000
    os.utime(second_input, ns=(newer_time, newer_time))
    output_mtime_before = output.stat(follow_symlinks=False).st_mtime_ns
    newer_input_is_stale = generated_file_is_stale(
        output,
        (first_input, second_input),
    )
    output_mtime_after = output.stat(follow_symlinks=False).st_mtime_ns

    missing_output_is_stale = generated_file_is_stale(
        root / "missing.txt",
        (first_input,),
    )

    try:
        generated_file_is_stale(output, ())
    except ValueError:
        empty_inputs_rejected = True
    else:
        empty_inputs_rejected = False

    directory_input = root / "directory"
    directory_input.mkdir()
    try:
        generated_file_is_stale(output, (directory_input,))
    except ValueError:
        directory_input_rejected = True
    else:
        directory_input_rejected = False

    try:
        generated_file_is_stale(output, (root / "absent-source.txt",))
    except FileNotFoundError:
        missing_input_propagated = True
    else:
        missing_input_propagated = False

assert (
    equal_timestamp_is_fresh,
    newer_input_is_stale,
    missing_output_is_stale,
    output_mtime_before == output_mtime_after,
    empty_inputs_rejected,
    directory_input_rejected,
    missing_input_propagated,
) == (True, True, True, True, True, True, True)
```

## Trade-offs and Limitations

The function materializes all input paths and performs one metadata lookup per
file. Filesystem timestamp resolution varies, clocks can move, and copying or
restoring a file can preserve an old modification time despite changed
content. Conversely, touching an unchanged input can cause an unnecessary
rebuild. The checks and the later build are not atomic, so another process can
replace a path after validation. Symlinks and non-regular files are rejected,
but no lock or isolation boundary is provided. A content digest gives a stronger content
identity signal at the cost of reading every byte.

## Related Snippets

<!-- catalog:related:start -->
- [Replace a File Atomically with a Sibling Temporary File](replace-a-file-atomically-with-a-sibling-temporary-file.md)
- [Store Bytes by Their Content Digest](store-bytes-by-their-content-digest.md)
<!-- catalog:related:end -->
