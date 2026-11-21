---
title: "Build Configuration Help from Dataclass Field Docs"
snippet_type: recipe
use_cases:
  - automation
  - configuration
  - testing
tested_python:
  - "3.14"
dependencies: []
related:
  - ../storage-databases/project-a-dataclass-into-a-validated-insert-row.md
  - ../python-language/normalize-raw-docstring-indentation-with-inspect-cleandoc.md
---

# Build Configuration Help from Dataclass Field Docs

## Idea and Problem

Collect ordered help text for documented configuration fields without constructing a configuration instance.

Python 3.14 lets you put a documentation string on `dataclasses.field()`.
Read it from each `Field` returned by `fields()` and use the same field
order in the help text. Filtering for initializer fields with a description
keeps derived and undocumented entries out of this view.

## When to Use

Use this when a dataclass models settings that your code documents in its
field declarations. Pass the dataclass type directly before any real
configuration is available. The result can feed a help screen or a test of
documented options.

## Implementation

```python
from dataclasses import fields


def configuration_help(config_type: type) -> tuple[tuple[str, str], ...]:
    if not isinstance(config_type, type):
        raise TypeError("expected a dataclass type")
    return tuple(
        (option.name, option.doc) for option in fields(config_type) if option.init and option.doc
    )
```

## Example

```python
from dataclasses import dataclass, field


events: list[str] = []


def default_columns() -> list[str]:
    events.append("factory")
    return ["date", "count"]


@dataclass
class ExportOptions:
    target: str = field(doc="Name of the output file.")
    row_limit: int = field(default=50, doc="Maximum number of rows.")
    dry_run: bool = False
    columns: list[str] = field(default_factory=default_columns, doc="Columns to include.")
    resolved_label: str = field(init=False, doc="Label computed by the loader.")

    def __post_init__(self) -> None:
        events.append("instance")
        self.resolved_label = self.target


assert (configuration_help(ExportOptions), events) == (
    (
        ("target", "Name of the output file."),
        ("row_limit", "Maximum number of rows."),
        ("columns", "Columns to include."),
    ),
    [],
)
```

## Trade-offs and Limitations

This shows the selected field descriptions. It does not look up default
values, call factories, validate any settings, or infer command-line flag
names. Init-only variables are outside `fields()`, and this helper also
skips fields that are excluded from the initializer.

Treat the descriptions as authored text. A caller still needs to format them
for its output medium and review them when the settings change.

## Related Snippets

<!-- catalog:related:start -->
- [Project a Dataclass into a Validated Insert Row](../storage-databases/project-a-dataclass-into-a-validated-insert-row.md)
- [Normalize Raw Docstring Indentation with inspect.cleandoc](../python-language/normalize-raw-docstring-indentation-with-inspect-cleandoc.md)
<!-- catalog:related:end -->
