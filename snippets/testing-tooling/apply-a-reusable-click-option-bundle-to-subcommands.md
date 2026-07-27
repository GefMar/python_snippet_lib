---
title: "Apply a Reusable Click Option Bundle to Subcommands"
snippet_type: integration
use_cases:
  - automation
  - configuration
  - testing
tested_python:
  - "3.14"
dependencies:
  - name: click
    version: "8.4.2"
related:
  - ../python-language/collect-decorated-methods-in-class-definition-order.md
  - ../python-language/dispatch-named-strategies-with-an-explicit-function-mapping.md
  - ../configuration-serialization/reject-unknown-options-with-conservative-typo-suggestions.md
---

# Apply a Reusable Click Option Bundle to Subcommands

## Idea and Problem

Apply the same Click option declarations to several subcommands while creating independent option objects for every command.

A plain decorator function can call Click's public `option` decorators each
time it is applied. This keeps the shared declarations in one place without
subclassing `Group`, mutating a registered command's parameter list, or
attaching one mutable `Option` instance to several commands.

## When to Use

Use this integration when a small, fixed set of options has identical names,
types, defaults, and help text on several subcommands. Apply the bundle
explicitly to every participating callback so readers can see which commands
accept it. Keep workflow-specific options next to their own command.

Use a real group option instead when one value belongs to the group invocation
and should be entered before the subcommand name. Avoid a shared bundle when
commands need subtly different defaults, validation, or environment-variable
behavior; separate declarations are clearer in that case.

## Implementation

```python
from collections.abc import Callable
from typing import Any, TypeVar, cast

import click


Callback = TypeVar("Callback", bound=Callable[..., Any])
_OUTPUT_FORMATS = ("text", "json")
_COLOR_MODES = ("auto", "always", "never")


def reusable_output_options(function: Callback) -> Callback:
    if isinstance(function, click.Command):
        raise TypeError(
            "reusable_output_options must run before command registration"
        )
    decorated = click.option(
        "--color",
        type=click.Choice(_COLOR_MODES, case_sensitive=True),
        default="auto",
        show_default=True,
        help="Control terminal color output.",
    )(function)
    decorated = click.option(
        "--output-format",
        type=click.Choice(_OUTPUT_FORMATS, case_sensitive=True),
        default="text",
        show_default=True,
        help="Select the output representation.",
    )(decorated)
    return cast(Callback, decorated)
```

## Example

```python
from click.testing import CliRunner


@click.group()
def cli() -> None:
    pass


@cli.command("inspect")
@reusable_output_options
@click.option(
    "--limit",
    type=click.IntRange(1, 5),
    default=2,
    show_default=True,
)
def inspect_command(output_format: str, color: str, limit: int) -> None:
    click.echo(f"inspect:{output_format}:{color}:{limit}")


@cli.command("export")
@reusable_output_options
@click.option("--compact", is_flag=True)
def export_command(output_format: str, color: str, compact: bool) -> None:
    click.echo(f"export:{output_format}:{color}:{compact}")


runner = CliRunner()
defaults = runner.invoke(cli, ("inspect",))
custom = runner.invoke(
    cli,
    (
        "export",
        "--output-format",
        "json",
        "--color",
        "never",
        "--compact",
    ),
)
invalid = runner.invoke(
    cli,
    ("inspect", "--output-format", "yaml"),
)
inspect_help = runner.invoke(cli, ("inspect", "--help"))
export_help = runner.invoke(cli, ("export", "--help"))

inspect_output_option = next(
    parameter
    for parameter in cli.commands["inspect"].params
    if parameter.name == "output_format"
)
export_output_option = next(
    parameter
    for parameter in cli.commands["export"].params
    if parameter.name == "output_format"
)
inspect_parameter_names = tuple(
    parameter.name for parameter in cli.commands["inspect"].params
)
export_parameter_names = tuple(
    parameter.name for parameter in cli.commands["export"].params
)


@click.command("already-registered")
def already_registered() -> None:
    pass


try:
    reusable_output_options(already_registered)
except TypeError:
    late_application_rejected = True
else:
    late_application_rejected = False

assert (
    defaults.exit_code,
    defaults.output.strip(),
    custom.exit_code,
    custom.output.strip(),
    invalid.exit_code != 0,
    "Invalid value for '--output-format'" in invalid.output,
    "--limit" in inspect_help.output,
    "--limit" not in export_help.output,
    "--output-format" in inspect_help.output,
    "--output-format" in export_help.output,
    inspect_output_option is not export_output_option,
    inspect_parameter_names,
    export_parameter_names,
    inspect_help.output.index("--output-format")
    < inspect_help.output.index("--color")
    < inspect_help.output.index("--limit"),
    late_application_rejected,
) == (
    0,
    "inspect:text:auto:2",
    0,
    "export:json:never:True",
    True,
    True,
    True,
    True,
    True,
    True,
    True,
    ("output_format", "color", "limit"),
    ("output_format", "color", "compact"),
    True,
    True,
)
```

## Trade-offs and Limitations

The bundle is deliberately explicit: every participating command needs one
decorator line, and changing the bundle changes all of those commands on their
next import. The helper applies decorators in reverse construction order so
Click presents `--output-format` before `--color`. Place the bundle beneath the
command decorator in source, as in the example, so it receives the callback
before registration. In exchange, Click constructs fresh parameters through
its public decorator API, and command registration retains its normal
lifecycle.

This pattern does not make the options valid before the subcommand name,
propagate values through `Context.obj`, discover commands dynamically, or
resolve collisions with command-local parameters. Apply a bundle at most once
per callback, keep its declarations static, and test every command's help and
parsing behavior when the shared interface changes.

## Related Snippets

<!-- catalog:related:start -->
- [Collect Decorated Methods in Class Definition Order](../python-language/collect-decorated-methods-in-class-definition-order.md)
- [Dispatch Named Strategies with an Explicit Function Mapping](../python-language/dispatch-named-strategies-with-an-explicit-function-mapping.md)
- [Reject Unknown Options with Conservative Typo Suggestions](../configuration-serialization/reject-unknown-options-with-conservative-typo-suggestions.md)
<!-- catalog:related:end -->
