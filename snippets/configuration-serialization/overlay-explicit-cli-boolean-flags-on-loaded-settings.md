---
title: "Overlay Explicit CLI Boolean Flags on Loaded Settings"
snippet_type: recipe
use_cases:
  - configuration
  - parsing
tested_python:
  - "3.14"
dependencies: []
related:
  - parse-a-bounded-explicit-cli-argument-tuple-without-reading-sys-argv-or-exiting.md
  - resolve-bounded-configuration-layers-with-winner-provenance.md
---

# Overlay Explicit CLI Boolean Flags on Loaded Settings

## Idea and Problem

Override loaded boolean settings only when the caller supplies an explicit positive or negative CLI flag.

Each boolean has three CLI cases: absent, explicitly enabled, or explicitly
disabled. `BooleanOptionalAction` creates the two flag spellings, and
`default=SUPPRESS` leaves an omitted flag out of the parsed namespace. Apply
only the fields present in that namespace to preserve loaded values.

## When to Use

Use this when a command loads and validates its settings before applying CLI
overrides. Pass an argument sequence without the executable name. The parser
handles ordinary CLI help and errors, and the overlay can be tested with
explicit inputs independently of the command's entry point.

## Implementation

```python
from argparse import SUPPRESS, ArgumentParser, BooleanOptionalAction
from collections.abc import Sequence
from dataclasses import dataclass, replace


@dataclass(frozen=True, slots=True)
class ExportSettings:
    compress: bool
    include_header: bool
    delimiter: str


def apply_export_flags(loaded: ExportSettings, argv: Sequence[str]) -> ExportSettings:
    parser = ArgumentParser(prog="export-rows", allow_abbrev=False)
    parser.add_argument("--compress", action=BooleanOptionalAction, default=SUPPRESS)
    parser.add_argument("--include-header", action=BooleanOptionalAction, default=SUPPRESS)
    overrides = vars(parser.parse_args(tuple(argv)))
    return replace(loaded, **overrides)
```

## Example

```python
loaded = ExportSettings(compress=False, include_header=True, delimiter=";")
mixed_flags = ["--compress", "--no-include-header"]

assert (
    apply_export_flags(loaded, ()),
    apply_export_flags(loaded, ("--compress",)),
    apply_export_flags(loaded, ("--no-include-header",)),
    apply_export_flags(loaded, mixed_flags),
    loaded,
    mixed_flags,
) == (
    ExportSettings(compress=False, include_header=True, delimiter=";"),
    ExportSettings(compress=True, include_header=True, delimiter=";"),
    ExportSettings(compress=False, include_header=False, delimiter=";"),
    ExportSettings(compress=True, include_header=False, delimiter=";"),
    ExportSettings(compress=False, include_header=True, delimiter=";"),
    ["--compress", "--no-include-header"],
)
```

## Trade-offs and Limitations

If the same option appears multiple times, the last spelling wins. Use an
explicit conflict check if opposing flags should be an error. The parser can
print help or usage and raise `SystemExit`, which fits a command-line entry
point. An embedded parser may need a different error contract.

Only declared override flags belong in this namespace. Keep unrelated command
arguments out of the fields passed to `replace`. If settings have cross-field
constraints, validate the merged settings before using them.

## Related Snippets

<!-- catalog:related:start -->
- [Parse a Bounded Explicit CLI Argument Tuple Without Reading sys.argv or Exiting](parse-a-bounded-explicit-cli-argument-tuple-without-reading-sys-argv-or-exiting.md)
- [Resolve Bounded Configuration Layers with Winner Provenance](resolve-bounded-configuration-layers-with-winner-provenance.md)
<!-- catalog:related:end -->
