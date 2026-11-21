---
title: "Report Independent Table Cases with unittest subTest"
snippet_type: testing-technique
use_cases:
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - generate-a-deterministic-pairwise-covering-matrix-from-closed-string-parameters.md
  - clean-up-unittest-fixtures-when-setup-fails-with-entercontext.md
---

# Report Independent Table Cases with unittest subTest

## Idea and Problem

Run a table of small checks in one unittest method and report each failing row under its own label.

An assertion that raises in a plain loop exits the test method. Put each row
inside `self.subTest(case=label)` so the test runner records its failure with
the label and can continue to later rows. Use a stable descriptive name so
the report points directly to the input that needs attention.

## When to Use

Use this for independent, cheap cases that share the same setup and teardown
and differ only in inputs and expected results. The example uses an incomplete
label normalizer to demonstrate that two failing cases are reported while
the other rows still run.

## Implementation

```python
from unittest import TestCase


def normalize_label(text: str) -> str:
    return text.strip()


class LabelCases(TestCase):
    def test_label_normalization(self) -> None:
        self.visited: list[str] = []
        cases = (
            ("uppercase", "READY", "ready"),
            ("surrounding spaces", " ready ", "ready"),
            ("word separator", "two words", "two-words"),
            ("unchanged", "done", "done"),
        )

        for label, source, expected in cases:
            self.visited.append(label)
            with self.subTest(case=label):
                self.assertEqual(normalize_label(source), expected)
```

## Example

```python
from io import StringIO
from unittest import TextTestRunner


case = LabelCases("test_label_normalization")
with StringIO() as report:
    result = TextTestRunner(stream=report, failfast=False).run(case)
    messages = report.getvalue()

assert case.visited == ["uppercase", "surrounding spaces", "word separator", "unchanged"]
assert (result.testsRun, len(result.failures), len(result.errors)) == (1, 2, 0)
assert "uppercase" in messages and "word separator" in messages
```

## Trade-offs and Limitations

The runner counts one test method and reports the failed subtests within it.
Setup and teardown still run once for the method. Mutable fixture state can
carry over between rows unless each row resets it. Use separate test methods
when cases need independent fixtures or independent selection by the runner.

An exception outside the subtest block can still stop the loop. A runner
configured with `failfast=True` can stop at the first failed subtest. Keep
labels small and descriptive because they can be displayed in test reports.

## Related Snippets

<!-- catalog:related:start -->
- [Generate a Deterministic Pairwise-Covering Matrix from Closed String Parameters](generate-a-deterministic-pairwise-covering-matrix-from-closed-string-parameters.md)
- [Clean Up unittest Fixtures When setUp Fails with enterContext](clean-up-unittest-fixtures-when-setup-fails-with-entercontext.md)
<!-- catalog:related:end -->
