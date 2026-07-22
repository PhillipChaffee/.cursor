---
description: "Where code lives: placement, moves, and growth limits."
alwaysApply: true
---

# Code Organization & Placement

Where code lives is a first-class concern. Apply these rules when writing or moving code in any repository, in any language.

## Placement decision

For every new function, class, dataclass, enum, constant, or module, ask: "Where would a reader look for this?" Put it there — not in the file you happen to be editing.

- Entry points (e.g. `main.py`, `app.py`, `index.ts`, route/handler files) stay thin: wiring and orchestration only. Domain logic, helpers, and data shapes go in the module that owns the domain.
- Data shapes (dataclasses, enums, DTOs, TypedDicts, interfaces) belong in the package's existing models/types module when one exists. Do not define them inline next to their first consumer.
- Generic helpers belong in the package's existing utils/helpers module when one exists. Check for that module — and whether the helper already exists in it — before writing a new one.
- Before writing any new helper, search the package and shared libraries for an existing implementation.

## Moving code

- When a better home exists for something being touched, propose the move — do not silently keep appending to the wrong file.
- When moving a symbol: move its tests, update all imports, and update test patch/mock targets (e.g. `patch("old.module.symbol")`) in the same change.
- Renames and pure moves are their own commits, separate from behavior changes.

## Growth limits

- Do not let grab-bag files (`utils.py`, `helpers.py`, `common.py`, `misc.py`) accumulate unrelated functions — split by domain once a file loses a coherent theme.
- Do not grow entry-point files: if a change adds significant non-wiring logic to an entry point, extract a module first.
- Tests mirror source layout: tests for `package/module.py` live in `package/tests/test_module.py` (or the repo's established mirror convention).
