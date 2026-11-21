---
title: "Replace Literal Strings in One Leftmost-Longest Pass"
snippet_type: recipe
use_cases:
  - data-transformation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../algorithms-data-structures/find-all-exact-multi-pattern-text-matches-with-aho-corasick.md
  - ../testing-tooling/replace-selected-python-name-tokens-without-touching-strings-or-comments.md
---

# Replace Literal Strings in One Leftmost-Longest Pass

## Idea and Problem

Replace a set of literal substrings in one scan without feeding replacement text back into the matcher.

Escape the input keys as regex literals and order the alternatives longest first. The regex starts at the leftmost available match, and ordering resolves ties at that position. A function returns each replacement verbatim, including backslashes.

## When to Use

Use this for small, trusted substitution tables and fully available text. Choose this policy when an overlapping later match must not undo an earlier one, and a replacement that resembles another key must remain untouched.

## Implementation

```python
import re
from collections.abc import Mapping


def replace_literals(text: str, rules: Mapping[str, str]) -> str:
    replacements = dict(rules)
    if not replacements:
        return text
    if "" in replacements:
        raise ValueError("an empty search key has no span")

    keys = sorted(replacements, key=len, reverse=True)
    pattern = re.compile("|".join(map(re.escape, keys)))
    return pattern.sub(lambda match: replacements[match[0]], text)
```

## Example

```python
rules = {"a": "short", "ab": "a", ".": r"\1", "bc": "late", "!": ""}
rewritten = replace_literals("abc. a!", rules)

try:
    replace_literals("unchanged", {"": "-"})
except ValueError:
    rejected_empty_key = True
else:
    rejected_empty_key = False

assert (
    rewritten,
    rejected_empty_key,
    replace_literals("no rules", {}),
) == (r"ac\1 short", True, "no rules")
```

## Trade-offs and Limitations

This matches substrings exactly, not words or Unicode-equivalent forms. It does not apply replacements repeatedly, and a match consumes its complete original span. Large rule sets can make alternative matching expensive; reuse a compiled pattern for repeated workloads, or consider a multi-pattern matcher. The input and result are held in memory.

## Related Snippets

<!-- catalog:related:start -->
- [Find All Exact Multi-Pattern Text Matches with Aho-Corasick](../algorithms-data-structures/find-all-exact-multi-pattern-text-matches-with-aho-corasick.md)
- [Replace Selected Python NAME Tokens without Touching Strings or Comments](../testing-tooling/replace-selected-python-name-tokens-without-touching-strings-or-comments.md)
<!-- catalog:related:end -->
