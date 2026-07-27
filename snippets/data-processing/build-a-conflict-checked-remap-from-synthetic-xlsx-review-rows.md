---
title: "Build a Conflict-Checked Remap from Synthetic XLSX Review Rows"
snippet_type: integration
use_cases:
  - data-transformation
  - interoperability
  - validation
tested_python:
  - "3.14"
dependencies:
  - name: openpyxl
    version: "3.1.5"
related:
  - validate-parsed-csv-rows-with-bounded-structured-problems.md
  - select-one-record-per-key-with-an-explicit-ranking-rule.md
  - ../storage-databases/build-and-apply-a-deterministic-mapping-delta.md
---

# Build a Conflict-Checked Remap from Synthetic XLSX Review Rows

## Idea and Problem

Turn a bounded worksheet of marked item pairs into immutable keep, remap, and delete decisions while rejecting ambiguous spreadsheet state.

The fixed four-column schema carries two item identifiers and their integer
revisions. A caller-chosen solid ARGB fill on an identifier marks that item for
removal: one marked member remaps to the unmarked member, two unmarked members
are both kept, and two marked members are both deleted. Repeated rows may
confirm a decision, but they cannot change an item's revision, outcome, or
remap target.

## When to Use

Use this boundary when a caller already owns a normal in-memory `Worksheet`
whose exact header is `item_one_id`, `item_one_revision`, `item_two_id`, and
`item_two_revision`. Each of the 1-512 data rows must contain two distinct
conservative ASCII identifiers and exact non-negative integer revisions. The
identifier cells are the only decision cells; revision and header cells must
have no fill.

Supply the exact eight-digit uppercase ARGB value used by the review fixture.
Use another representation when colors are inherited, conditional formatting
controls appearance, or the worksheet shape is not fully governed by the
caller.

## Implementation

```python
import re
from dataclasses import dataclass

from openpyxl.cell.cell import Cell
from openpyxl.worksheet.worksheet import Worksheet


_MAX_DATA_ROWS = 512
_MAX_REVISION = 2_147_483_647
_HEADER = (
    "item_one_id",
    "item_one_revision",
    "item_two_id",
    "item_two_revision",
)
_ITEM_ID = re.compile(r"[A-Za-z0-9][A-Za-z0-9._-]{0,63}", re.ASCII)
_ARGB = re.compile(r"[0-9A-F]{8}", re.ASCII)


@dataclass(frozen=True, slots=True)
class KeepDecision:
    item_id: str
    revision: int


@dataclass(frozen=True, slots=True)
class RemapDecision:
    item_id: str
    revision: int
    target_id: str


@dataclass(frozen=True, slots=True)
class DeleteDecision:
    item_id: str
    revision: int


type ReviewDecision = KeepDecision | RemapDecision | DeleteDecision


def _marker_state(
    cell: Cell,
    *,
    removal_marker: str,
    marker_allowed: bool,
) -> bool:
    fill = cell.fill
    if fill.fill_type is None:
        return False
    if (
        marker_allowed
        and fill.fill_type == "solid"
        and fill.fgColor.type == "rgb"
        and fill.fgColor.rgb == removal_marker
        and fill.fgColor.tint == 0.0
    ):
        return True
    raise ValueError(f"cell {cell.coordinate} has an unsupported fill")


def _item_id(cell: Cell) -> str:
    value = cell.value
    if type(value) is not str or _ITEM_ID.fullmatch(value) is None:
        raise ValueError(f"cell {cell.coordinate} has an invalid item identifier")
    return value


def _revision(cell: Cell) -> int:
    value = cell.value
    if type(value) is not int or not 0 <= value <= _MAX_REVISION:
        raise ValueError(f"cell {cell.coordinate} has an invalid revision")
    return value


def build_review_decisions(
    worksheet: Worksheet,
    *,
    removal_marker: str,
) -> tuple[ReviewDecision, ...]:
    if not isinstance(worksheet, Worksheet):
        raise TypeError("worksheet must be an openpyxl Worksheet")
    if type(removal_marker) is not str:
        raise TypeError("removal_marker must be an exact string")
    if _ARGB.fullmatch(removal_marker) is None:
        raise ValueError("removal_marker must be uppercase eight-digit ARGB")
    if worksheet.merged_cells.ranges:
        raise ValueError("merged cells are not supported")
    if worksheet.max_column != len(_HEADER):
        raise ValueError("worksheet must have exactly four populated columns")
    if not 2 <= worksheet.max_row <= _MAX_DATA_ROWS + 1:
        raise ValueError("worksheet must contain between 1 and 512 data rows")

    rows = worksheet.iter_rows(
        min_row=1,
        max_row=worksheet.max_row,
        min_col=1,
        max_col=len(_HEADER),
    )
    header_cells = next(rows)
    for cell in header_cells:
        if cell.data_type == "f":
            raise ValueError(f"formula in cell {cell.coordinate} is not supported")
        _marker_state(
            cell,
            removal_marker=removal_marker,
            marker_allowed=False,
        )
    if tuple(cell.value for cell in header_cells) != _HEADER:
        raise ValueError("worksheet header does not match the fixed pair schema")

    revisions: dict[str, int] = {}
    decisions: dict[str, ReviewDecision] = {}

    for cells in rows:
        marked = []
        for column, cell in enumerate(cells):
            if cell.data_type == "f":
                raise ValueError(f"formula in cell {cell.coordinate} is not supported")
            marked.append(
                _marker_state(
                    cell,
                    removal_marker=removal_marker,
                    marker_allowed=column in (0, 2),
                )
            )

        first_id = _item_id(cells[0])
        first_revision = _revision(cells[1])
        second_id = _item_id(cells[2])
        second_revision = _revision(cells[3])
        if first_id == second_id:
            raise ValueError("the two items in a review row must be distinct")

        for item_id, revision in (
            (first_id, first_revision),
            (second_id, second_revision),
        ):
            previous_revision = revisions.get(item_id)
            if previous_revision is not None and previous_revision != revision:
                raise ValueError(f"item {item_id!r} has inconsistent revision metadata")
            revisions[item_id] = revision

        first_marked, second_marked = marked[0], marked[2]
        if first_marked and second_marked:
            row_decisions: tuple[ReviewDecision, ReviewDecision] = (
                DeleteDecision(first_id, first_revision),
                DeleteDecision(second_id, second_revision),
            )
        elif first_marked:
            row_decisions = (
                RemapDecision(first_id, first_revision, second_id),
                KeepDecision(second_id, second_revision),
            )
        elif second_marked:
            row_decisions = (
                KeepDecision(first_id, first_revision),
                RemapDecision(second_id, second_revision, first_id),
            )
        else:
            row_decisions = (
                KeepDecision(first_id, first_revision),
                KeepDecision(second_id, second_revision),
            )

        for decision in row_decisions:
            previous = decisions.get(decision.item_id)
            if previous is not None and previous != decision:
                raise ValueError(f"item {decision.item_id!r} has conflicting decisions")
            decisions[decision.item_id] = decision

    return tuple(decisions[item_id] for item_id in sorted(decisions))
```

## Example

```python
from openpyxl import Workbook
from openpyxl.styles import PatternFill


removal_marker = "FFF4B183"
workbook = Workbook()
worksheet = workbook.active
worksheet.append(
    ("item_one_id", "item_one_revision", "item_two_id", "item_two_revision")
)
worksheet.append(("item-a", 1, "item-b", 1))
worksheet.append(("item-c", 2, "item-d", 3))
worksheet.append(("item-e", 4, "item-f", 5))
worksheet.append(("item-g", 6, "item-h", 7))

marker_fill = PatternFill(fill_type="solid", fgColor=removal_marker)
worksheet["A3"].fill = marker_fill
worksheet["C4"].fill = marker_fill
worksheet["A5"].fill = marker_fill
worksheet["C5"].fill = marker_fill


def snapshot() -> tuple[tuple[object, ...], ...]:
    return tuple(
        (
            cell.coordinate,
            cell.value,
            cell.data_type,
            cell.fill.fill_type,
            cell.fill.fgColor.type if cell.fill.fill_type == "solid" else None,
            cell.fill.fgColor.rgb if cell.fill.fill_type == "solid" else None,
        )
        for row in worksheet.iter_rows(min_row=1, max_row=5, min_col=1, max_col=4)
        for cell in row
    )


before = snapshot()
decisions = build_review_decisions(worksheet, removal_marker=removal_marker)

assert decisions == (
    KeepDecision("item-a", 1),
    KeepDecision("item-b", 1),
    RemapDecision("item-c", 2, "item-d"),
    KeepDecision("item-d", 3),
    KeepDecision("item-e", 4),
    RemapDecision("item-f", 5, "item-e"),
    DeleteDecision("item-g", 6),
    DeleteDecision("item-h", 7),
)
assert snapshot() == before
assert workbook.sheetnames == ["Sheet"]
```

## Trade-offs and Limitations

The function scans a small rectangular worksheet and sorts the distinct item
identifiers, using memory proportional to their count. The exact header and
revision columns deliberately make repeated-item metadata checkable, but they
also reject otherwise usable sheets with renamed, reordered, sparse, or
decoratively filled cells. A repeated decision is idempotent; a remap chain is
not inferred because its intermediate item would have both keep and remap
outcomes and is therefore a conflict.

Only direct `PatternFill` state is interpreted. Theme, indexed, automatic,
gradient, patterned, tinted, conditional, and inherited colors are not
resolved. Formula cells and merged ranges are rejected rather than evaluated
or expanded. The caller-supplied ARGB marker is matched exactly, including its
alpha digits and letter case.

This is a read-only decision builder for a caller-owned in-memory worksheet.
It does not open or save files, preserve or execute macros, mutate workbook
cells, apply remaps or deletions, interact with a spreadsheet application, or
provide audit storage, persistence, or transactional publication.

## Related Snippets

<!-- catalog:related:start -->
- [Validate Parsed CSV Rows with Bounded Structured Problems](validate-parsed-csv-rows-with-bounded-structured-problems.md)
- [Select One Record per Key with an Explicit Ranking Rule](select-one-record-per-key-with-an-explicit-ranking-rule.md)
- [Build and Apply a Deterministic Mapping Delta](../storage-databases/build-and-apply-a-deterministic-mapping-delta.md)
<!-- catalog:related:end -->
