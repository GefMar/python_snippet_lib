---
title: "Index Bounded Python Scope Bindings with symtable without Execution"
snippet_type: testing-technique
use_cases:
  - parsing
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - extract-a-bounded-static-python-annotation-index-without-evaluation.md
  - compare-bounded-python-expressions-structurally-without-execution.md
  - ../data-processing/count-static-imports-across-bounded-python-notebook-cells.md
---

# Index Bounded Python Scope Bindings with symtable without Execution

## Idea and Problem

Inspect how the Python compiler classifies bounded source bindings and closure relationships without executing the source.

Syntax alone does not decide whether a name is local, free, nonlocal, global,
imported, or a type parameter. The `symtable` module exposes the compiler's
scope analysis directly. Child-index paths identify even repeated or synthetic
scope names, while fixed scope, depth, symbol, and retained-name budgets bound
the result after compilation.

## When to Use

Use this index in tests, documentation tools, or static audits that need the
compiler's binding decisions for one self-contained source string. It is
particularly useful for checking closure capture, explicit `global` and
`nonlocal` declarations, inlined-comprehension bindings, and generic syntax.

This is not a type checker, control-flow analyzer, linter, or import resolver.
It does not report every AST occurrence or source span. Use an AST-oriented
tool when spelling and positions matter, and a full static-analysis framework
when cross-module inference or control-flow-sensitive facts are required.

## Implementation

```python
import symtable
from dataclasses import dataclass
from enum import StrEnum

_MAX_SOURCE_CHARACTERS = 65_536
_MAX_SOURCE_BYTES = 131_072
_MAX_SCOPES = 512
_MAX_SCOPE_DEPTH = 64
_MAX_SYMBOLS = 4_096
_MAX_IDENTIFIER_BYTES = 262_144


class BindingFlag(StrEnum):
    PARAMETER = "parameter"
    IMPORTED = "imported"
    ASSIGNED = "assigned"
    ANNOTATED = "annotated"
    REFERENCED = "referenced"
    LOCAL = "local"
    GLOBAL = "global"
    DECLARED_GLOBAL = "declared-global"
    NONLOCAL = "nonlocal"
    FREE = "free"
    NAMESPACE = "namespace"
    TYPE_PARAMETER = "type-parameter"
    COMPREHENSION_ITERATOR = "comprehension-iterator"
    COMPREHENSION_CELL = "comprehension-cell"
    FREE_CLASS = "free-class"


@dataclass(frozen=True, slots=True)
class SymbolBinding:
    name: str
    flags: tuple[BindingFlag, ...]


@dataclass(frozen=True, slots=True)
class ScopeBindings:
    child_path: tuple[int, ...]
    scope_type: symtable.SymbolTableType
    name: str
    line: int
    nested: bool
    optimized: bool
    symbols: tuple[SymbolBinding, ...]


_FLAG_TESTS = (
    (BindingFlag.PARAMETER, symtable.Symbol.is_parameter),
    (BindingFlag.IMPORTED, symtable.Symbol.is_imported),
    (BindingFlag.ASSIGNED, symtable.Symbol.is_assigned),
    (BindingFlag.ANNOTATED, symtable.Symbol.is_annotated),
    (BindingFlag.REFERENCED, symtable.Symbol.is_referenced),
    (BindingFlag.LOCAL, symtable.Symbol.is_local),
    (BindingFlag.GLOBAL, symtable.Symbol.is_global),
    (BindingFlag.DECLARED_GLOBAL, symtable.Symbol.is_declared_global),
    (BindingFlag.NONLOCAL, symtable.Symbol.is_nonlocal),
    (BindingFlag.FREE, symtable.Symbol.is_free),
    (BindingFlag.NAMESPACE, symtable.Symbol.is_namespace),
    (BindingFlag.TYPE_PARAMETER, symtable.Symbol.is_type_parameter),
    (BindingFlag.COMPREHENSION_ITERATOR, symtable.Symbol.is_comp_iter),
    (BindingFlag.COMPREHENSION_CELL, symtable.Symbol.is_comp_cell),
    (BindingFlag.FREE_CLASS, symtable.Symbol.is_free_class),
)


def _identifier_size(name: str) -> int:
    try:
        return len(name.encode("utf-8"))
    except UnicodeEncodeError as error:
        raise ValueError("compiler identifier contains a Unicode surrogate") from error


def index_scope_bindings(source: str) -> tuple[ScopeBindings, ...]:
    if type(source) is not str:
        raise TypeError("source must be an exact string")
    if len(source) > _MAX_SOURCE_CHARACTERS:
        raise ValueError("source exceeds the supported character count")
    try:
        encoded_source = source.encode("utf-8")
    except UnicodeEncodeError as error:
        raise ValueError("source must not contain Unicode surrogates") from error
    if len(encoded_source) > _MAX_SOURCE_BYTES:
        raise ValueError("source exceeds the supported UTF-8 byte count")

    try:
        root = symtable.symtable(source, "<snippet>", "exec")
    except (MemoryError, RecursionError) as error:
        raise ValueError("source exceeds the compiler's supported depth") from error

    scopes: list[ScopeBindings] = []
    symbol_count = 0
    identifier_bytes = 0
    pending = [(root, (), 0)]
    while pending:
        table, child_path, depth = pending.pop()
        if len(scopes) >= _MAX_SCOPES:
            raise ValueError("scope table count exceeds the supported limit")
        if depth > _MAX_SCOPE_DEPTH:
            raise ValueError("scope table depth exceeds the supported limit")

        identifier_bytes += _identifier_size(table.get_name())
        if identifier_bytes > _MAX_IDENTIFIER_BYTES:
            raise ValueError("retained identifiers exceed the supported byte count")

        table_symbols = table.get_symbols()
        symbol_count += len(table_symbols)
        if symbol_count > _MAX_SYMBOLS:
            raise ValueError("symbol count exceeds the supported limit")

        bindings: list[SymbolBinding] = []
        for symbol in sorted(table_symbols, key=symtable.Symbol.get_name):
            name = symbol.get_name()
            identifier_bytes += _identifier_size(name)
            if identifier_bytes > _MAX_IDENTIFIER_BYTES:
                raise ValueError(
                    "retained identifiers exceed the supported byte count"
                )
            flags = tuple(
                flag for flag, predicate in _FLAG_TESTS if predicate(symbol)
            )
            bindings.append(SymbolBinding(name=name, flags=flags))

        scopes.append(
            ScopeBindings(
                child_path=child_path,
                scope_type=table.get_type(),
                name=table.get_name(),
                line=table.get_lineno(),
                nested=table.is_nested(),
                optimized=table.is_optimized(),
                symbols=tuple(bindings),
            )
        )

        children = table.get_children()
        pending.extend(
            (children[index], (*child_path, index), depth + 1)
            for index in range(len(children) - 1, -1, -1)
        )
    return tuple(scopes)
```

## Example

```python
source = """\
import math as maths
shared = 3

def outer[T](step):
    total = 0
    def inner(value):
        nonlocal total
        global shared
        total += step
        return shared + total + value
    return inner

squares = [item * item for item in range(3)]
raise RuntimeError("this source must never execute")
side_effect()
"""

scopes = index_scope_bindings(source)


def scope_named(name: str, scope_type: symtable.SymbolTableType) -> ScopeBindings:
    return next(
        scope
        for scope in scopes
        if scope.name == name and scope.scope_type is scope_type
    )


def symbol_named(scope: ScopeBindings, name: str) -> SymbolBinding:
    return next(symbol for symbol in scope.symbols if symbol.name == name)


module = scopes[0]
outer = scope_named("outer", symtable.SymbolTableType.FUNCTION)
inner = scope_named("inner", symtable.SymbolTableType.FUNCTION)
type_parameters = scope_named("outer", symtable.SymbolTableType.TYPE_PARAMETERS)

assert BindingFlag.IMPORTED in symbol_named(module, "maths").flags
assert BindingFlag.COMPREHENSION_ITERATOR in symbol_named(module, "item").flags
assert BindingFlag.TYPE_PARAMETER in symbol_named(type_parameters, "T").flags
assert BindingFlag.PARAMETER in symbol_named(outer, "step").flags
assert BindingFlag.FREE in symbol_named(inner, "step").flags
assert BindingFlag.NONLOCAL in symbol_named(inner, "total").flags
assert BindingFlag.DECLARED_GLOBAL in symbol_named(inner, "shared").flags

scope_limit = "class Marker:\n    pass\n" + "".join(
    f"def f{index}():\n    pass\n" for index in range(255)
)
assert len(index_scope_bindings(scope_limit)) == 512

too_many_scopes = "".join(
    f"def f{index}():\n    pass\n" for index in range(256)
)
try:
    index_scope_bindings(too_many_scopes)
except ValueError:
    pass
else:
    raise AssertionError("513 compiler scopes must be rejected")

symbol_limit = "\n".join(f"value_{index} = 0" for index in range(4_096))
assert len(index_scope_bindings(symbol_limit)[0].symbols) == 4_096

too_many_symbols = symbol_limit + "\none_more = 0"
try:
    index_scope_bindings(too_many_symbols)
except ValueError:
    pass
else:
    raise AssertionError("4,097 compiler symbols must be rejected")


def nested_classes(depth: int) -> str:
    declarations = [
        "    " * level + f"class Scope{level}:"
        for level in range(depth)
    ]
    return "\n".join((*declarations, "    " * depth + "pass"))


assert max(len(scope.child_path) for scope in index_scope_bindings(
    nested_classes(64)
)) == 64
try:
    index_scope_bindings(nested_classes(65))
except ValueError:
    pass
else:
    raise AssertionError("scope depth 65 must be rejected")

assert scopes[0].child_path == ()
```

## Trade-offs and Limitations

After the compiler returns, let `T` be the number of tables, `D` their maximum
depth, `I` retained identifier bytes, and `s_t` the symbol count in table `t`.
Path construction, name retention, and per-table sorting use
`O(T * D + I + sum(s_t log(s_t + 1)))` wrapper work. The immutable output
retains the bounded paths, names, flags, tables, and symbols.

Compilation happens before the table and symbol budgets can be checked. The
source byte cap bounds admitted input, but compiler-front-end cost is
implementation-dependent and the post-build limits are not a hostile-input
sandbox. Compiler recursion and memory-depth failures are normalized, while a
normal `SyntaxError` preserves its source diagnostics.

The module table is depth zero. Compiler-created annotation, type-parameter,
and other synthetic tables and symbols count like user-visible ones. Child
indexes depend on the compiler's table order for the tested Python version;
they are stable identities within one index, not a cross-version serialization
format. The helper never imports, compiles to bytecode, or executes the source.

## Related Snippets

<!-- catalog:related:start -->
- [Extract a Bounded Static Python Annotation Index without Evaluation](extract-a-bounded-static-python-annotation-index-without-evaluation.md)
- [Compare Bounded Python Expressions Structurally without Execution](compare-bounded-python-expressions-structurally-without-execution.md)
- [Count Static Imports Across Bounded Python Notebook Cells](../data-processing/count-static-imports-across-bounded-python-notebook-cells.md)
<!-- catalog:related:end -->
