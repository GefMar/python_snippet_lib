---
title: "Build a Collision-Audited Artifact Copy Plan"
snippet_type: algorithm
use_cases:
  - automation
  - resource-management
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - group-generated-text-artifacts-by-exact-body-for-review.md
  - ../storage-databases/plan-a-bounded-named-posix-layout-under-explicit-roots.md
  - ../storage-databases/build-a-bounded-file-manifest-with-internal-symlink-aliases.md
---

# Build a Collision-Audited Artifact Copy Plan

## Idea and Problem

Turn ordered successful task results and already-resolved artifact candidates into one deterministic, immutable copy manifest.

`ALL_MATCHES` requires 1 to 1,024 candidates and preserves every candidate's
filename. `EXACT_ONE_RENAMED` requires exactly one candidate and assigns the
rule's explicit filename. The planner validates the complete batch, including
source duplication and portable destination collisions, before returning any
`CopyIntent`.

## When to Use

Use this algorithm after a build adapter has finished all relevant tasks and
resolved each rule to a finite tuple of artifact candidates. Task order, rule
order, and candidate order are explicit and validated; the planner performs no
discovery. It is useful when a test or packaging layer needs to review one
bounded copy manifest before a separate executor acts.

Task results are ordered lexicographically by task ID. Rules follow task order
and then rule ID; candidates follow rule order and then source ID. These are
canonical input requirements rather than sorting performed by the planner.

All paths in the records are normalized relative POSIX-style logical
identifiers, not filesystem capabilities or containment proofs. Before using a
plan, an executor must resolve each logical path beneath separately authorized
roots, revalidate containment and source identity, and refuse unintended
overwrite.

## Implementation

```python
import re
from dataclasses import dataclass
from enum import StrEnum
from pathlib import PurePosixPath

_MAX_TASKS = 128
_MAX_RULES = 512
_MAX_CANDIDATES = 4_096
_MAX_PER_RULE = 1_024
_MAX_PATH_PARTS = 12
_MAX_PATH_BYTES = 768
_MAX_ARTIFACT_BYTES = 2**31 - 1
_MAX_TOTAL_BYTES = 2**34
_ID = re.compile(r"[a-z0-9][a-z0-9._:-]{0,63}", re.ASCII).fullmatch
_COMPONENT = re.compile(r"[A-Za-z0-9][A-Za-z0-9._-]{0,95}", re.ASCII).fullmatch


class RuleKind(StrEnum):
    ALL_MATCHES = "ALL_MATCHES"
    EXACT_ONE_RENAMED = "EXACT_ONE_RENAMED"


@dataclass(frozen=True, slots=True)
class TaskResult:
    task_id: str
    completed: bool
    successful: bool


@dataclass(frozen=True, slots=True)
class ArtifactRule:
    task_id: str
    rule_id: str
    kind: RuleKind
    destination_directory: str
    renamed_filename: str | None = None


@dataclass(frozen=True, slots=True)
class ArtifactCandidate:
    task_id: str
    rule_id: str
    source_id: str
    source_path: str
    byte_size: int


@dataclass(frozen=True, slots=True)
class CopyIntent:
    task_id: str
    rule_id: str
    source_id: str
    source_path: str
    destination_path: str
    byte_size: int


def _checked_id(value: object, *, field: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if _ID(value) is None:
        raise ValueError(f"{field} must be a conservative 1-to-64-byte ID")
    return value


def _checked_component(value: object, *, field: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if _COMPONENT(value) is None:
        raise ValueError(f"{field} is not a safe bounded filename component")
    return value


def _checked_path(value: object, *, field: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if not 1 <= len(value) <= _MAX_PATH_BYTES:
        raise ValueError(f"{field} is empty or exceeds its byte bound")
    if not value.isascii():
        raise ValueError(f"{field} must contain conservative ASCII components")
    path = PurePosixPath(value)
    if path.is_absolute() or path.as_posix() != value:
        raise ValueError(f"{field} must be a normalized relative POSIX identifier")
    if not 1 <= len(path.parts) <= _MAX_PATH_PARTS:
        raise ValueError(f"{field} exceeds its component-count bound")
    for index, part in enumerate(path.parts):
        _checked_component(part, field=f"{field}.parts[{index}]")
    return value


def _checked_size(value: object, *, field: str) -> int:
    if type(value) is not int:
        raise TypeError(f"{field} must be an exact integer")
    if not 0 <= value <= _MAX_ARTIFACT_BYTES:
        raise ValueError(f"{field} is outside the supported byte range")
    return value


def _strictly_ordered(values: tuple[object, ...], *, field: str) -> None:
    if values != tuple(sorted(values)) or len(set(values)) != len(values):
        raise ValueError(f"{field} must be unique and in strict deterministic order")


def _checked_tasks(value: object) -> tuple[TaskResult, ...]:
    if type(value) is not tuple:
        raise TypeError("task_results must be an exact tuple")
    if not 1 <= len(value) <= _MAX_TASKS:
        raise ValueError(f"task_results must contain 1 to {_MAX_TASKS} records")
    checked: list[TaskResult] = []
    for index, item in enumerate(value):
        field = f"task_results[{index}]"
        if type(item) is not TaskResult:
            raise TypeError(f"{field} must be an exact TaskResult")
        task_id = _checked_id(item.task_id, field=f"{field}.task_id")
        if type(item.completed) is not bool or type(item.successful) is not bool:
            raise TypeError(f"{field} status fields must be exact booleans")
        checked.append(TaskResult(task_id, item.completed, item.successful))
    frozen = tuple(checked)
    _strictly_ordered(tuple(item.task_id for item in frozen), field="task_results")
    if any(not item.completed or not item.successful for item in frozen):
        raise ValueError("every task must be completed and successful")
    return frozen


def _checked_rules(
    value: object,
    *,
    task_order: dict[str, int],
) -> tuple[ArtifactRule, ...]:
    if type(value) is not tuple:
        raise TypeError("rules must be an exact tuple")
    if not 1 <= len(value) <= _MAX_RULES:
        raise ValueError(f"rules must contain 1 to {_MAX_RULES} records")
    checked: list[ArtifactRule] = []
    for index, item in enumerate(value):
        field = f"rules[{index}]"
        if type(item) is not ArtifactRule:
            raise TypeError(f"{field} must be an exact ArtifactRule")
        task_id = _checked_id(item.task_id, field=f"{field}.task_id")
        rule_id = _checked_id(item.rule_id, field=f"{field}.rule_id")
        if task_id not in task_order:
            raise ValueError(f"{field} refers to an unknown task")
        if type(item.kind) is not RuleKind:
            raise TypeError(f"{field}.kind must be an exact RuleKind")
        destination = _checked_path(
            item.destination_directory,
            field=f"{field}.destination_directory",
        )
        if item.kind is RuleKind.ALL_MATCHES:
            if item.renamed_filename is not None:
                raise ValueError("ALL_MATCHES rules cannot rename candidates")
            renamed = None
        else:
            renamed = _checked_component(
                item.renamed_filename,
                field=f"{field}.renamed_filename",
            )
        checked.append(ArtifactRule(task_id, rule_id, item.kind, destination, renamed))

    frozen = tuple(checked)
    rule_ids = tuple(item.rule_id for item in frozen)
    if len(set(rule_ids)) != len(rule_ids):
        raise ValueError("rule IDs must be globally unique")
    order_keys = tuple((task_order[item.task_id], item.rule_id) for item in frozen)
    _strictly_ordered(order_keys, field="rules")
    return frozen


def _checked_candidates(
    value: object,
    *,
    rules_by_id: dict[str, ArtifactRule],
    rule_order: dict[str, int],
) -> tuple[ArtifactCandidate, ...]:
    if type(value) is not tuple:
        raise TypeError("candidates must be an exact tuple")
    if not 1 <= len(value) <= _MAX_CANDIDATES:
        raise ValueError(f"candidates must contain 1 to {_MAX_CANDIDATES} records")

    checked: list[ArtifactCandidate] = []
    source_ids: set[str] = set()
    source_paths: set[str] = set()
    total_bytes = 0
    for index, item in enumerate(value):
        field = f"candidates[{index}]"
        if type(item) is not ArtifactCandidate:
            raise TypeError(f"{field} must be an exact ArtifactCandidate")
        task_id = _checked_id(item.task_id, field=f"{field}.task_id")
        rule_id = _checked_id(item.rule_id, field=f"{field}.rule_id")
        source_id = _checked_id(item.source_id, field=f"{field}.source_id")
        source_path = _checked_path(item.source_path, field=f"{field}.source_path")
        byte_size = _checked_size(item.byte_size, field=f"{field}.byte_size")

        rule = rules_by_id.get(rule_id)
        if rule is None or rule.task_id != task_id:
            raise ValueError(f"{field} does not match one declared task rule")
        if source_id in source_ids:
            raise ValueError("source IDs must be globally unique")
        if source_path in source_paths:
            raise ValueError("one logical source appears more than once")
        source_ids.add(source_id)
        source_paths.add(source_path)
        total_bytes += byte_size
        if total_bytes > _MAX_TOTAL_BYTES:
            raise ValueError("candidate bytes exceed the aggregate byte budget")
        checked.append(ArtifactCandidate(task_id, rule_id, source_id, source_path, byte_size))

    frozen = tuple(checked)
    order_keys = tuple((rule_order[item.rule_id], item.source_id) for item in frozen)
    _strictly_ordered(order_keys, field="candidates")

    counts = {rule_id: 0 for rule_id in rules_by_id}
    for item in frozen:
        counts[item.rule_id] += 1
    for rule_id, rule in rules_by_id.items():
        count = counts[rule_id]
        if rule.kind is RuleKind.EXACT_ONE_RENAMED and count != 1:
            raise ValueError(f"rule {rule_id!r} must resolve to exactly one candidate")
        if rule.kind is RuleKind.ALL_MATCHES and not 1 <= count <= _MAX_PER_RULE:
            raise ValueError(f"rule {rule_id!r} has an invalid match count")
    return frozen


def build_artifact_copy_plan(
    task_results: tuple[TaskResult, ...],
    rules: tuple[ArtifactRule, ...],
    candidates: tuple[ArtifactCandidate, ...],
) -> tuple[CopyIntent, ...]:
    """Preflight a complete batch before returning immutable copy intents."""
    checked_tasks = _checked_tasks(task_results)
    task_order = {item.task_id: index for index, item in enumerate(checked_tasks)}
    checked_rules = _checked_rules(rules, task_order=task_order)
    rules_by_id = {item.rule_id: item for item in checked_rules}
    rule_order = {item.rule_id: index for index, item in enumerate(checked_rules)}
    checked_candidates = _checked_candidates(
        candidates,
        rules_by_id=rules_by_id,
        rule_order=rule_order,
    )

    intents: list[CopyIntent] = []
    destination_keys: set[str] = set()
    for candidate in checked_candidates:
        rule = rules_by_id[candidate.rule_id]
        source_name = PurePosixPath(candidate.source_path).name
        filename = source_name if rule.kind is RuleKind.ALL_MATCHES else rule.renamed_filename
        assert filename is not None
        destination = _checked_path(
            f"{rule.destination_directory}/{filename}",
            field="planned destination",
        )
        collision_key = destination.casefold()
        if collision_key in destination_keys:
            raise ValueError(f"destination collision: {destination!r}")
        destination_keys.add(collision_key)
        intents.append(
            CopyIntent(
                candidate.task_id,
                candidate.rule_id,
                candidate.source_id,
                candidate.source_path,
                destination,
                candidate.byte_size,
            )
        )
    return tuple(intents)
```

## Example

```python
tasks = (
    TaskResult("compile-docs", completed=True, successful=True),
    TaskResult("package-tool", completed=True, successful=True),
)
rules = (
    ArtifactRule(
        "compile-docs",
        "docs",
        RuleKind.ALL_MATCHES,
        "bundle/docs",
    ),
    ArtifactRule(
        "package-tool",
        "program",
        RuleKind.EXACT_ONE_RENAMED,
        "bundle/bin",
        "runner",
    ),
)
candidates = (
    ArtifactCandidate("compile-docs", "docs", "doc-a", "outputs/a.html", 120),
    ArtifactCandidate("compile-docs", "docs", "doc-b", "outputs/b.html", 180),
    ArtifactCandidate("package-tool", "program", "binary", "outputs/tool", 512),
)

plan = build_artifact_copy_plan(tasks, rules, candidates)

assert plan == (
    CopyIntent("compile-docs", "docs", "doc-a", "outputs/a.html", "bundle/docs/a.html", 120),
    CopyIntent("compile-docs", "docs", "doc-b", "outputs/b.html", "bundle/docs/b.html", 180),
    CopyIntent("package-tool", "program", "binary", "outputs/tool", "bundle/bin/runner", 512),
)
assert sum(intent.byte_size for intent in plan) == 812
```

## Trade-offs and Limitations

Validation is linear apart from bounded sorting comparisons used to confirm
caller order. The manifest contains at most 4,096 intents across 512 rules and
128 tasks, with at most 1,024 candidates per `ALL_MATCHES` rule. IDs,
components, paths, counts, statuses, individual sizes, and an aggregate
16-GiB byte budget are bounded. A zero-byte artifact is valid.

Every structural or semantic failure raises `TypeError` or `ValueError` and
returns no tuple. This all-or-nothing behavior prevents consumers from mistaking
a prefix for a complete manifest, but one bad rule prevents planning unrelated
copies. ASCII case-folded destination auditing is intentionally conservative
for portability and can reject names that a case-sensitive target could keep
distinct. Source duplication is exact-path based and does not resolve aliases.

The planner invokes no build tool, subprocess, glob, filesystem API, copy, or
overwrite operation. It neither proves that sources exist nor detects existing
destinations, symlinks, mounts, permission changes, or races. Source and
destination paths remain logical identifiers; the execution layer must map
them beneath authorized roots, revalidate containment and metadata immediately
before each copy, define overwrite policy, and report partial execution.

## Related Snippets

<!-- catalog:related:start -->
- [Group Generated Text Artifacts by Exact Body for Review](group-generated-text-artifacts-by-exact-body-for-review.md)
- [Plan a Bounded Named POSIX Layout Under Explicit Roots](../storage-databases/plan-a-bounded-named-posix-layout-under-explicit-roots.md)
- [Build a Bounded File Manifest with Internal Symlink Aliases](../storage-databases/build-a-bounded-file-manifest-with-internal-symlink-aliases.md)
<!-- catalog:related:end -->
