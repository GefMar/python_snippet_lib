---
title: "Assemble Fixed-Width Feature Slots from Bounded Named Sources"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - group-items-by-an-exact-compatibility-signature-and-report-unmatched-inputs.md
  - merge-bounded-row-batches-with-a-first-seen-schema-union.md
  - validate-parsed-csv-rows-with-bounded-structured-problems.md
---

# Assemble Fixed-Width Feature Slots from Bounded Named Sources

## Idea and Problem

Assemble one immutable numeric vector from named, fixed-width slots without letting arrival order define the output layout.

An ordered slot specification is the layout contract. Each slot is either
required or carries an explicit default block. Observed blocks override
defaults, while the result records whether every slot was observed or
defaulted. If any required slot is absent, the function returns only a
structured missing-slot result, never a misleading partial vector.

## When to Use

Use this algorithm after a bounded set of independent producers has already
created finite numeric blocks and one consumer requires a stable flat layout.
It fits in-memory scoring, ranking, or rule inputs whose slot names and widths
are versioned outside the function. The specification order, not observation
order, determines both the assembled values and audit entries.

Use a schema migration or a typed domain model when slot meaning changes over
time. Use a relational join when there can be multiple records per source.
Extraction, grouping, training, persistence, and I/O belong outside this
boundary.

## Implementation

```python
import math
import re
from dataclasses import dataclass
from typing import Literal


SlotPolicy = Literal["required", "default"]
AuditSource = Literal["observed", "defaulted"]

_MAX_SLOTS = 64
_MAX_SLOT_WIDTH = 256
_MAX_TOTAL_WIDTH = 4_096
_SLOT_NAME = re.compile(r"[a-z][a-z0-9_]{0,47}", re.ASCII)
_POLICIES = frozenset({"required", "default"})


@dataclass(frozen=True, slots=True)
class SlotSpec:
    name: str
    width: int
    policy: SlotPolicy
    default: tuple[float, ...] | None = None


@dataclass(frozen=True, slots=True)
class ObservedBlock:
    name: str
    values: tuple[float, ...]


@dataclass(frozen=True, slots=True)
class SlotAudit:
    name: str
    source: AuditSource


@dataclass(frozen=True, slots=True)
class FeatureAssembly:
    values: tuple[float, ...]
    audit: tuple[SlotAudit, ...]


@dataclass(frozen=True, slots=True)
class MissingRequiredSlots:
    names: tuple[str, ...]


AssemblyResult = FeatureAssembly | MissingRequiredSlots


def _validate_name(value: object) -> str:
    if type(value) is not str:
        raise TypeError("slot names must be exact strings")
    if _SLOT_NAME.fullmatch(value) is None:
        raise ValueError("slot names must be canonical ASCII identifiers")
    return value


def _validate_block(
    value: object,
    *,
    width: int,
    label: str,
) -> tuple[float, ...]:
    if type(value) is not tuple:
        raise TypeError(f"{label} must be an exact tuple")
    if len(value) != width:
        raise ValueError(f"{label} has the wrong width")
    for number in value:
        if type(number) is not float:
            raise TypeError(f"{label} must contain exact floats")
        if not math.isfinite(number):
            raise ValueError(f"{label} must contain only finite values")
    return value


def _prepare_specs(value: object) -> tuple[tuple[SlotSpec, ...], int]:
    if type(value) is not tuple:
        raise TypeError("specs must be an exact tuple")
    if not 1 <= len(value) <= _MAX_SLOTS:
        raise ValueError("slot count is outside the supported range")

    prepared = []
    known_names: set[str] = set()
    total_width = 0
    for raw_spec in value:
        if type(raw_spec) is not SlotSpec:
            raise TypeError("specs must contain exact SlotSpec values")
        name = _validate_name(raw_spec.name)
        if name in known_names:
            raise ValueError("slot names must be unique")
        known_names.add(name)

        if type(raw_spec.width) is not int:
            raise TypeError("slot widths must be exact integers")
        if not 1 <= raw_spec.width <= _MAX_SLOT_WIDTH:
            raise ValueError("slot width is outside the supported range")
        if type(raw_spec.policy) is not str:
            raise TypeError("slot policy must be an exact string")
        if raw_spec.policy not in _POLICIES:
            raise ValueError("slot policy must be required or default")

        if raw_spec.policy == "required":
            if raw_spec.default is not None:
                raise ValueError("a required slot cannot carry a default block")
            default = None
        else:
            if raw_spec.default is None:
                raise ValueError("a default slot must carry a default block")
            default = _validate_block(
                raw_spec.default,
                width=raw_spec.width,
                label=f"default block for {name!r}",
            )

        total_width += raw_spec.width
        if total_width > _MAX_TOTAL_WIDTH:
            raise ValueError("assembled width exceeds the supported limit")
        prepared.append(SlotSpec(name, raw_spec.width, raw_spec.policy, default))

    return tuple(prepared), total_width


def _prepare_observed(
    value: object,
    *,
    specs_by_name: dict[str, SlotSpec],
) -> dict[str, tuple[float, ...]]:
    if type(value) is not tuple:
        raise TypeError("observed blocks must be an exact tuple")
    if len(value) > _MAX_SLOTS:
        raise ValueError("observed block count exceeds the supported limit")

    prepared: dict[str, tuple[float, ...]] = {}
    for raw_block in value:
        if type(raw_block) is not ObservedBlock:
            raise TypeError("observed blocks must contain exact ObservedBlock values")
        name = _validate_name(raw_block.name)
        spec = specs_by_name.get(name)
        if spec is None:
            raise ValueError(f"observed block {name!r} has no slot specification")
        if name in prepared:
            raise ValueError("observed block names must be unique")
        prepared[name] = _validate_block(
            raw_block.values,
            width=spec.width,
            label=f"observed block for {name!r}",
        )
    return prepared


def assemble_feature_slots(
    specs: tuple[SlotSpec, ...],
    observed: tuple[ObservedBlock, ...],
) -> AssemblyResult:
    prepared_specs, expected_width = _prepare_specs(specs)
    specs_by_name = {spec.name: spec for spec in prepared_specs}
    observed_by_name = _prepare_observed(
        observed,
        specs_by_name=specs_by_name,
    )

    missing = tuple(
        spec.name
        for spec in prepared_specs
        if spec.policy == "required" and spec.name not in observed_by_name
    )
    if missing:
        return MissingRequiredSlots(missing)

    values: list[float] = []
    audit = []
    for spec in prepared_specs:
        block = observed_by_name.get(spec.name)
        if block is None:
            if spec.default is None:
                raise AssertionError("validated default slot has no default block")
            block = spec.default
            source: AuditSource = "defaulted"
        else:
            source = "observed"
        values.extend(block)
        audit.append(SlotAudit(spec.name, source))

    if len(values) != expected_width:
        raise AssertionError("assembled width differs from validated slot width")
    return FeatureAssembly(tuple(values), tuple(audit))
```

## Example

```python
specs = (
    SlotSpec("profile", 2, "required"),
    SlotSpec("context", 1, "default", (0.0,)),
    SlotSpec("quality", 2, "required"),
)
observed = (
    ObservedBlock("quality", (0.8, 0.2)),
    ObservedBlock("profile", (1.0, 3.0)),
)
inputs_before = (specs, observed)

assembled = assemble_feature_slots(specs, observed)
missing = assemble_feature_slots(specs, (observed[1],))

assert (
    assembled,
    missing,
    (specs, observed),
) == (
    FeatureAssembly(
        values=(1.0, 3.0, 0.0, 0.8, 0.2),
        audit=(
            SlotAudit("profile", "observed"),
            SlotAudit("context", "defaulted"),
            SlotAudit("quality", "observed"),
        ),
    ),
    MissingRequiredSlots(("quality",)),
    inputs_before,
)
```

## Trade-offs and Limitations

Validation and assembly are linear in the bounded slot count and total width.
The function keeps the complete specification, observations, and result in
memory. Exact tuples, exact floats, conservative ASCII names, and fixed limits
make the layout deterministic, but deliberately reject generators, integers,
decimal values, and unbounded feature sets.

Defaults keep an assembly complete while the audit preserves their use. They
can still hide an unhealthy producer if callers ignore that audit. A missing
result reports all required slot names in specification order and has no value
vector to consume accidentally. The function rejects unknown or duplicate
observations instead of choosing a winner, and it never mutates the caller's
frozen inputs.

## Related Snippets

<!-- catalog:related:start -->
- [Group Items by an Exact Compatibility Signature and Report Unmatched Inputs](group-items-by-an-exact-compatibility-signature-and-report-unmatched-inputs.md)
- [Merge Bounded Row Batches with a First-Seen Schema Union](merge-bounded-row-batches-with-a-first-seen-schema-union.md)
- [Validate Parsed CSV Rows with Bounded Structured Problems](validate-parsed-csv-rows-with-bounded-structured-problems.md)
<!-- catalog:related:end -->
