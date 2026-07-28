---
title: "Extract Bounded Features with an Explicit Abstention Outcome"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - assemble-fixed-width-feature-slots-from-bounded-named-sources.md
  - project-bounded-records-into-multiple-closed-output-schemas.md
  - validate-parsed-csv-rows-with-bounded-structured-problems.md
---

# Extract Bounded Features with an Explicit Abstention Outcome

## Idea and Problem

Extract a complete, versioned feature tuple from one immutable ceramic-tile fact record, or return a value-free reason for expected abstention.

Structural validation covers the entire record before extraction policy runs.
Malformed facts therefore raise an exception even when the record is also
incomplete. A valid but incomplete, unrepresented, or out-of-range record
returns ordinary frozen abstention data, and no partial numeric tuple escapes.

## When to Use

Use this algorithm at a small in-memory boundary where one closed fact schema
feeds one fixed feature schema. It is useful when missing facts and categories
outside the represented vocabulary are expected data outcomes rather than
programming errors, but malformed containers, duplicate names, and invalid
scalars must remain distinguishable.

This is extraction, not fixed-width block assembly. It accounts for every
declared output directly from one validated fact record and never fills absent
facts with defaults. Assembly is the better boundary after independent
producers have already created complete numeric blocks.

## Implementation

```python
import math
import re
from dataclasses import dataclass
from typing import Literal

AbstentionReason = Literal[
    "missing_fact",
    "unrepresented_category",
    "outside_feature_range",
]

_FACT_SCHEMA_VERSION = "tile-facts/1"
_FEATURE_SCHEMA_VERSION = "tile-features/1"
_MAX_CATEGORY_FACTS = 4
_MAX_SCALAR_FACTS = 6
_MAX_TOTAL_FACTS = 8
_MAX_CATEGORY_BYTES = 24
_MAX_OUTPUT_FEATURES = 8
_MAX_VALID_SCALAR = 10_000.0
_CATEGORY_VALUE = re.compile(r"[a-z][a-z0-9-]{0,23}", re.ASCII)
_CATEGORY_NAMES = frozenset({"finish", "form"})
_SCALAR_NAMES = frozenset({"length_mm", "thickness_mm", "width_mm"})
_FINISH_VALUES = frozenset({"matte", "satin", "glazed"})
_FORM_VALUES = frozenset({"square", "rectangle"})
_REQUIRED_CATEGORIES = ("finish", "form")
_REQUIRED_SCALARS = ("length_mm", "width_mm", "thickness_mm")
_FEATURE_DECLARATION = (
    "length_fraction",
    "width_fraction",
    "thickness_fraction",
    "finish_matte",
    "finish_satin",
    "finish_glazed",
    "form_square",
    "form_rectangle",
)


@dataclass(frozen=True, slots=True)
class CategoryFact:
    name: str
    value: str


@dataclass(frozen=True, slots=True)
class ScalarFact:
    name: str
    value: float


@dataclass(frozen=True, slots=True)
class TileFactRecord:
    schema_version: str
    categories: tuple[CategoryFact, ...]
    scalars: tuple[ScalarFact, ...]


@dataclass(frozen=True, slots=True)
class ExtractedFeatures:
    schema_version: str
    values: tuple[float, ...]


@dataclass(frozen=True, slots=True)
class FeatureAbstention:
    reason: AbstentionReason


ExtractionResult = ExtractedFeatures | FeatureAbstention


def _validate_fact_name(value: object, *, allowed: frozenset[str], kind: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{kind} names must be exact strings")
    if value not in allowed:
        raise ValueError(f"{kind} name is not part of the fact schema")
    return value


def _copy_validated_record(
    value: object,
) -> tuple[dict[str, str], dict[str, float]]:
    if type(value) is not TileFactRecord:
        raise TypeError("record must be an exact TileFactRecord value")
    if type(value.schema_version) is not str:
        raise TypeError("fact schema version must be an exact string")
    if value.schema_version != _FACT_SCHEMA_VERSION:
        raise ValueError("fact schema version is not supported")
    if type(value.categories) is not tuple:
        raise TypeError("category facts must be an exact tuple")
    if type(value.scalars) is not tuple:
        raise TypeError("scalar facts must be an exact tuple")
    if len(value.categories) > _MAX_CATEGORY_FACTS:
        raise ValueError("category fact count exceeds the supported budget")
    if len(value.scalars) > _MAX_SCALAR_FACTS:
        raise ValueError("scalar fact count exceeds the supported budget")
    if len(value.categories) + len(value.scalars) > _MAX_TOTAL_FACTS:
        raise ValueError("total fact count exceeds the supported budget")

    categories: dict[str, str] = {}
    for fact in value.categories:
        if type(fact) is not CategoryFact:
            raise TypeError("category entries must be exact CategoryFact values")
        name = _validate_fact_name(
            fact.name,
            allowed=_CATEGORY_NAMES,
            kind="category fact",
        )
        if name in categories:
            raise ValueError("category fact names must be unique")
        if type(fact.value) is not str:
            raise TypeError("category values must be exact strings")
        if _CATEGORY_VALUE.fullmatch(fact.value) is None:
            raise ValueError("category value does not match the fact grammar")
        if len(fact.value.encode("ascii")) > _MAX_CATEGORY_BYTES:
            raise ValueError("category value exceeds the supported byte budget")
        categories[name] = fact.value

    scalars: dict[str, float] = {}
    for fact in value.scalars:
        if type(fact) is not ScalarFact:
            raise TypeError("scalar entries must be exact ScalarFact values")
        name = _validate_fact_name(
            fact.name,
            allowed=_SCALAR_NAMES,
            kind="scalar fact",
        )
        if name in scalars:
            raise ValueError("scalar fact names must be unique")
        if type(fact.value) is not float:
            raise TypeError("scalar values must be exact floats")
        if not math.isfinite(fact.value):
            raise ValueError("scalar values must be finite")
        if not 0.0 <= fact.value <= _MAX_VALID_SCALAR:
            raise ValueError("scalar value is outside the valid fact range")
        scalars[name] = fact.value

    return categories, scalars


def extract_tile_features(record: TileFactRecord) -> ExtractionResult:
    categories, scalars = _copy_validated_record(record)

    missing_category = any(name not in categories for name in _REQUIRED_CATEGORIES)
    missing_scalar = any(name not in scalars for name in _REQUIRED_SCALARS)
    if missing_category or missing_scalar:
        return FeatureAbstention("missing_fact")

    finish = categories.get("finish")
    form = categories.get("form")
    if finish not in _FINISH_VALUES or form not in _FORM_VALUES:
        return FeatureAbstention("unrepresented_category")

    length = scalars.get("length_mm")
    width = scalars.get("width_mm")
    thickness = scalars.get("thickness_mm")
    if length is None or width is None or thickness is None:
        raise AssertionError("required scalar disappeared after validation")
    if not (50.0 <= length <= 1_200.0 and 50.0 <= width <= 1_200.0 and 2.0 <= thickness <= 40.0):
        return FeatureAbstention("outside_feature_range")

    values = (
        length / 1_200.0,
        width / 1_200.0,
        thickness / 40.0,
        1.0 if finish == "matte" else 0.0,
        1.0 if finish == "satin" else 0.0,
        1.0 if finish == "glazed" else 0.0,
        1.0 if form == "square" else 0.0,
        1.0 if form == "rectangle" else 0.0,
    )
    if len(_FEATURE_DECLARATION) > _MAX_OUTPUT_FEATURES:
        raise AssertionError("feature declaration exceeds the output budget")
    if len(values) != len(_FEATURE_DECLARATION):
        raise AssertionError("feature values do not match their declaration")
    if any(not math.isfinite(feature) for feature in values):
        raise AssertionError("feature extraction produced a non-finite value")
    return ExtractedFeatures(_FEATURE_SCHEMA_VERSION, tuple(values))
```

## Example

```python
complete = TileFactRecord(
    schema_version="tile-facts/1",
    categories=(
        CategoryFact("finish", "satin"),
        CategoryFact("form", "rectangle"),
    ),
    scalars=(
        ScalarFact("width_mm", 300.0),
        ScalarFact("thickness_mm", 10.0),
        ScalarFact("length_mm", 600.0),
    ),
)
missing = TileFactRecord(
    schema_version="tile-facts/1",
    categories=(CategoryFact("finish", "matte"),),
    scalars=(),
)
unrepresented = TileFactRecord(
    schema_version="tile-facts/1",
    categories=(
        CategoryFact("finish", "textured"),
        CategoryFact("form", "square"),
    ),
    scalars=complete.scalars,
)

malformed_records = (
    TileFactRecord(
        schema_version="tile-facts/1",
        categories=(),
        scalars=(ScalarFact("length_mm", True),),
    ),
    TileFactRecord(
        schema_version="tile-facts/1",
        categories=(
            CategoryFact("finish", "matte"),
            CategoryFact("finish", "satin"),
        ),
        scalars=(),
    ),
)
malformed_errors = []
for malformed in malformed_records:
    try:
        extract_tile_features(malformed)
    except (TypeError, ValueError) as error:
        malformed_errors.append(type(error))

assert (
    extract_tile_features(complete),
    extract_tile_features(missing),
    extract_tile_features(unrepresented),
    tuple(malformed_errors),
) == (
    ExtractedFeatures(
        schema_version="tile-features/1",
        values=(0.5, 0.25, 0.25, 0.0, 1.0, 0.0, 0.0, 1.0),
    ),
    FeatureAbstention("missing_fact"),
    FeatureAbstention("unrepresented_category"),
    (TypeError, ValueError),
)
```

## Trade-offs and Limitations

Validation and extraction take `O(C + S + F)` time and memory for category
facts, scalar facts, and declared features; all three dimensions have fixed
budgets. Exact tuples, exact frozen records, and exact built-in strings and
floats conservatively reject mutable containers, subclasses, integer-to-float
coercion, and booleans masquerading as integers. Duplicate names and non-finite
numbers are malformed input, not abstentions, and returned data contains only
new immutable tuples and frozen values.

Abstention precedence is missing facts, then unrepresented categories, then
feature-range coverage. Each code intentionally omits fact names and values.
The schema and ranges are local and require a new version when their meaning or
declaration order changes. This extraction performs no model inference or
scoring, makes no security or approval decision, performs no external lookup,
and triggers no human-review side effect.

## Related Snippets

<!-- catalog:related:start -->
- [Assemble Fixed-Width Feature Slots from Bounded Named Sources](assemble-fixed-width-feature-slots-from-bounded-named-sources.md)
- [Project Bounded Records into Multiple Closed Output Schemas](project-bounded-records-into-multiple-closed-output-schemas.md)
- [Validate Parsed CSV Rows with Bounded Structured Problems](validate-parsed-csv-rows-with-bounded-structured-problems.md)
<!-- catalog:related:end -->
