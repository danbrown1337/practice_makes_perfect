---
description: 'Python coding conventions and guidelines'
applyTo: '**/*.py'
---

# Python Coding Conventions

## Source of Truth

Tooling config is authoritative; this file explains intent, it does not
re-litigate style. All code must pass the project's configured tools.

**Currently configured (P2-16, Phase 7): pytest only** (`pytest.ini`;
target runtime Python 3.11 per `.python-version`). Ruff and mypy are NOT
yet configured in this repository — adopting them is part of the
deliberately-sequenced toolchain migration (CANONICAL_PATTERNS_SCOPING:
uv / pyproject / Ruff), not something to improvise per-change. Until
that migration lands, follow the conventions below by hand and do not
invent lint/type gates that don't exist.

Once the migration adds `pyproject.toml` / `ruff.toml` / `mypy.ini`,
those configs win over this file. Do not hand-format around the
formatter or argue line length; run the tools and conform to them.

## Core Guidelines

- Follow **PEP 8**, as enforced by Ruff. Don't restate or override it.
- Use **type hints** on every function signature (params and return).
- Write a **docstring** for every public module, class, and function
  (see Docstrings below). Private helpers get a docstring when non-obvious.
- Use **descriptive names**: `snake_case` for functions/variables,
  `PascalCase` for classes, `UPPER_CASE` for constants.
- Keep functions **small and single-purpose**. If you can't name it
  cleanly, it's doing too much — split it.
- Prefer **pure functions** (same input → same output, no hidden side
  effects); isolate I/O and state at the edges.
- Comment the **why, not the what**. The code says what; comments explain
  the non-obvious reason. For algorithms, briefly note the approach.

## Style

- Imports at the **top of the file**, grouped stdlib → third-party → local.
  No imports inside function bodies unless avoiding a circular import or
  gating a heavy/optional dependency.
- **f-strings** for formatting, not `%` or `.format()`.
- Use **`pathlib`** for filesystem paths, not `os.path` string joins.
- Iterate **directly** (`for item in items`), not by index.
- Comprehensions for simple transforms; fall back to a loop when the logic
  gets complex enough to hurt readability.
- **Guard clauses** over deep nesting — return early on bad input.

## Correctness & Safety

- **Never use a bare `except:`.** Catch specific exception types.
- **No mutable default arguments.** Use `None` as the default and create
  the object inside the function.
- **Fail loud, not silent.** Raise on invalid state rather than returning a
  wrong-but-plausible value. Never swallow errors to keep a path "working."
- Use **context managers** (`with`) for files, locks, and connections so
  cleanup happens even on exception.
- **Validate inputs** at function boundaries; raise `ValueError` /
  `TypeError` with a clear message on bad arguments.
- Keep **secrets out of code** — environment variables or a secrets
  manager, never hardcoded.

## Docstrings (Google style)

Use **Google-style** docstrings. Place the docstring immediately after the
`def` / `class` line. Do **not** repeat the type in `Args:` — the type hint
in the signature is the single source of truth.

Sections, when applicable: `Args:`, `Returns:`, `Raises:`. One-line
docstrings are fine for trivial functions.

## Testing

- Use **pytest**. Tests live in `tests/`, named `test_*.py`.
- Test **public behavior**, not internals.
- Cover **edge cases**: empty inputs, invalid types, boundary values, and
  the error paths (assert that the right exception is raised).
- Use `pytest.mark.parametrize` for tables of cases instead of copy-paste.
- State the **intent** of each test in its name or a short docstring.

## Anti-Patterns to Avoid

- Bare `except:` or `except Exception:` that hides the real error.
- Mutable default arguments (`def f(x=[])`).
- Returning a default/sentinel to mask a failure instead of raising.
- Catch-and-`pass`, or catch-and-log-then-continue on unrecoverable state.
- Deeply nested conditionals where guard clauses would flatten the logic.
- Restating types in the docstring when they're already in the signature.

## Example

```python
"""Geometry helpers."""

import math


def calculate_area(radius: float) -> float:
    """Calculate the area of a circle.

    Args:
        radius: Radius of the circle. Must be non-negative.

    Returns:
        The area of the circle, computed as pi * radius ** 2.

    Raises:
        ValueError: If radius is negative.
    """
    if radius < 0:
        raise ValueError(f"radius must be non-negative, got {radius}")
    return math.pi * radius ** 2
```
