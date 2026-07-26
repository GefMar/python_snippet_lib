---
title: "Normalize Optional CSV Columns in a Single Pass"
snippet_type: recipe
use_cases:
  - data-transformation
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../configuration-serialization/prune-empty-values-from-json-like-data.md
---

# Normalize Optional CSV Columns in a Single Pass

## Idea and Problem

Normalize selected columns in already parsed CSV rows while handling optional columns, short rows, and duplicate normalized records explicitly.

An ordered unique header defines every output position. Named normalizers are
resolved once before iteration and either fail or are skipped when their column
is absent. Each row is padded to the header width, normalized in place, and
optionally deduplicated within this invocation without keeping mutable state
between runs.

## When to Use

Use this recipe after `csv.reader` or another dialect parser when several input
schemas share some normalizable columns. Choose strict missing-column behavior
for stable schemas and opt into ignoring missing columns for genuinely optional
fields. The normalized result must fit in memory. Normalizers should be
deterministic functions from one string to one string and should raise on
invalid values rather than silently inventing data.

## Implementation

```python
from collections.abc import Callable, Iterable, Mapping, Sequence
from dataclasses import dataclass


ValueNormalizer = Callable[[str], str]


@dataclass(frozen=True, slots=True)
class CsvNormalizationResult:
    header: tuple[str, ...]
    rows: tuple[tuple[str, ...], ...]
    padded_rows: int
    duplicate_rows: int


def normalize_csv_rows(
    header: Sequence[str],
    rows: Iterable[Sequence[str]],
    normalizers: Mapping[str, ValueNormalizer],
    *,
    ignore_missing_columns: bool = False,
    deduplicate: bool = True,
) -> CsvNormalizationResult:
    if isinstance(header, (str, bytes)):
        raise TypeError("header must be a sequence of column names")
    normalized_header = tuple(header)
    if not normalized_header:
        raise ValueError("header must contain at least one column")
    if any(not isinstance(name, str) or not name for name in normalized_header):
        raise ValueError("column names must be non-empty strings")
    if len(set(normalized_header)) != len(normalized_header):
        raise ValueError("column names must be unique")
    if not isinstance(ignore_missing_columns, bool) or not isinstance(deduplicate, bool):
        raise TypeError("normalization policy flags must be booleans")

    positions = {name: index for index, name in enumerate(normalized_header)}
    active: list[tuple[str, int, ValueNormalizer]] = []
    for name, normalizer in normalizers.items():
        if not isinstance(name, str) or not name:
            raise ValueError("normalizer names must be non-empty strings")
        if not callable(normalizer):
            raise TypeError(f"normalizer for {name!r} must be callable")
        if name not in positions:
            if ignore_missing_columns:
                continue
            raise KeyError(f"normalizer column is missing from header: {name!r}")
        active.append((name, positions[name], normalizer))

    output: list[tuple[str, ...]] = []
    seen: set[tuple[str, ...]] = set()
    padded_rows = 0
    duplicate_rows = 0

    for row_number, row in enumerate(rows, start=1):
        if isinstance(row, (str, bytes)):
            raise TypeError(f"row {row_number} must be a sequence of strings")
        values = list(row)
        if any(not isinstance(value, str) for value in values):
            raise TypeError(f"row {row_number} must contain only strings")
        if len(values) > len(normalized_header):
            raise ValueError(f"row {row_number} is wider than the header")
        if len(values) < len(normalized_header):
            values.extend([""] * (len(normalized_header) - len(values)))
            padded_rows += 1

        for name, position, normalizer in active:
            normalized = normalizer(values[position])
            if not isinstance(normalized, str):
                raise TypeError(f"normalizer for {name!r} must return text")
            values[position] = normalized

        normalized_row = tuple(values)
        if deduplicate and normalized_row in seen:
            duplicate_rows += 1
            continue
        if deduplicate:
            seen.add(normalized_row)
        output.append(normalized_row)

    return CsvNormalizationResult(
        header=normalized_header,
        rows=tuple(output),
        padded_rows=padded_rows,
        duplicate_rows=duplicate_rows,
    )
```

## Example

```python
from collections.abc import Iterator


def input_rows() -> Iterator[tuple[str, ...]]:
    yield (" Alice@Example.COM ", "MT")
    yield ("alice@example.com", "MT", "")
    yield (" Bob@Example.COM ", "GB", "active")


result = normalize_csv_rows(
    ("email", "country", "note"),
    input_rows(),
    {
        "email": lambda value: value.strip().lower(),
        "phone": lambda value: value.strip(),
    },
    ignore_missing_columns=True,
)

try:
    normalize_csv_rows(("email",), [("a", "extra")], {})
except ValueError:
    long_row_rejected = True
else:
    long_row_rejected = False

try:
    normalize_csv_rows(("email",), [], {"phone": str.strip})
except KeyError:
    missing_column_rejected = True
else:
    missing_column_rejected = False

assert (
    result,
    long_row_rejected,
    missing_column_rejected,
) == (
    CsvNormalizationResult(
        header=("email", "country", "note"),
        rows=(
            ("alice@example.com", "MT", ""),
            ("bob@example.com", "GB", "active"),
        ),
        padded_rows=1,
        duplicate_rows=1,
    ),
    True,
    True,
)
```

## Trade-offs and Limitations

The helper traverses the input once but materializes all returned rows, requiring
`O(n)` output memory. Deduplication adds an `O(u)` set for `u` unique normalized
rows and treats all columns as the record identity. Use an iterator-to-writer
pipeline when output must stream, and add a separately bounded or external
deduplication policy only when needed. CSV dialect detection, decoding, quoting,
schema inference, and domain validation remain separate. Padding can hide
truncated input unless its counter is monitored. Normalizer exceptions stop the
iteration after earlier input may already have caused external work in the
caller's source generator.

## Related Snippets

<!-- catalog:related:start -->
- [Prune Empty Values from JSON-Like Data](../configuration-serialization/prune-empty-values-from-json-like-data.md)
<!-- catalog:related:end -->
