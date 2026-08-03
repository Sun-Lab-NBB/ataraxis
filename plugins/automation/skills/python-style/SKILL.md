---
name: python-style
description: >-
  Applies Python coding conventions when writing, reviewing, or refactoring code. Covers
  .py files, docstrings, type annotations, naming, formatting, error handling, imports, file
  ordering, and ataraxis library preferences. Use when writing new Python code, modifying existing
  code, reviewing pull requests, or when the user asks about Python coding standards.
user-invocable: false
---

# Python code style guide

Applies Python coding conventions.

You MUST read this skill and load the relevant reference files before writing or modifying Python
code. You MUST verify your changes against the checklist before submitting.

---

## Scope

**Covers:**
- Python code style (docstrings, type annotations, naming, formatting, error handling)
- Ataraxis library usage preferences (console, timing, data structures)
- File organization, import conventions, and module structure
- Click CLI conventions
- Enum, dataclass, and class design patterns
- Test file conventions

**Does not cover:**
- README file conventions (see `/readme-style`)
- Commit message conventions (see `/commit`)
- Skill file and CLAUDE.md conventions (see `/skill-design`)
- Codebase exploration workflows (see `/explore-codebase`)

---

## Workflow

You MUST follow these steps when this skill is invoked.

### Step 1: Read this skill

Read this entire file. The core conventions below apply to ALL Python code.

### Step 2: Load relevant reference files

Based on the task, load the appropriate reference files:

| Task                                                  | Reference to load                                             |
|-------------------------------------------------------|---------------------------------------------------------------|
| Writing or modifying any docstring, comment, or type  | [docstrings-and-types.md](references/docstrings-and-types.md) |
| Writing classes, dataclasses, enums, or `__init__.py` | [class-patterns.md](references/class-patterns.md)             |
| Using ataraxis libs, Numba, Click, tests              | [libraries-and-tools.md](references/libraries-and-tools.md)   |
| Using ataraxis library features                       | Invoke `/explore-dependencies` first, then load above         |
| Reviewing code before submission                      | [anti-patterns.md](references/anti-patterns.md)               |

Load multiple references when the task spans multiple domains.

### Step 3: Apply conventions

Write or modify Python code following all conventions from this file and the loaded references.

### Step 4: Verify compliance

Complete the verification checklist at the end of this file. Every item must pass before
submitting work. For anti-pattern examples, load
[anti-patterns.md](references/anti-patterns.md).

---

## Cross-language consistency

Projects span Python, C++, and C#. These conventions maximize visual and structural
consistency across languages while respecting each language's idiomatic standards.

**Shared across all languages:**
- 120 character line limit
- 4-space indentation (no tabs)
- Comprehensive documentation on ALL public and private members
- Third-person imperative mood for documentation ("Provides...", "Determines whether...")
- A leading underscore marks a symbol private to the module or class that defines it (`_snake_case`
  for any private Python symbol, `_snake_case` for C++ data members, `_camelCase` for C# fields)
- Full words in identifiers (no abbreviations)
- Guard clauses preferred over deep nesting
- Prose over bullet lists in documentation
- No example/code blocks in documentation (they go stale)
- I/O operations separated from processing logic
- Only full stops and commas separate clauses in documentation prose (no semicolons, no em-dashes)
- State what the code does now, not what it avoids doing or formerly did (positive description)

**Python-specific divergences from C++:**
- Functions and methods use snake_case (not PascalCase as in C++)
- Constants use `_UPPER_SNAKE_CASE` for private / `UPPER_SNAKE_CASE` for public (not `kPascalCase` as in C++)
- Enum values use `UPPER_SNAKE_CASE` (not `kPascalCase` as in C++)
- Documentation uses Google-style docstrings (not Doxygen `@tags`)
- Error handling uses `console.error()` or `raise` (not status codes)
- Brace style is not applicable (Python uses indentation)

**Python-specific divergences from C#:**
- Functions and methods use snake_case (not PascalCase as in C#)
- Constants use `_UPPER_SNAKE_CASE` for private / `UPPER_SNAKE_CASE` for public (not PascalCase as in C#)
- Enum values use `UPPER_SNAKE_CASE` (not PascalCase as in C#)
- Documentation uses Google-style docstrings (not XML `<summary>` tags)
- Private members use `_snake_case` (not `_camelCase` as in C#)
- Public fields use snake_case (not camelCase as in C#)

---

## Naming conventions

### Variables

Use **full words**, not abbreviations:

| Avoid             | Prefer                              |
|-------------------|-------------------------------------|
| `t`, `t_sq`       | `interpolation_factor`, `t_squared` |
| `coeff`, `coeffs` | `coefficient`, `coefficients`       |
| `pos`, `idx`      | `position`, `index`                 |
| `img`, `val`      | `image`, `value`                    |

### Functions

- Use descriptive verb phrases: `compute_coefficients`, `extract_features`
- Private functions start with underscore: `_process_batch`, `_validate_input`
- Avoid generic names like `process`, `handle`, `do_something`

### Visibility prefixes and the module boundary

A leading underscore marks a symbol private to the **module** that defines it. This covers functions,
classes, constants, class attributes, and methods. Any symbol referenced from another module MUST
carry a public name, so a helper that acquires a caller in a second module is renamed rather than
imported with its underscore intact:

```python
# Good - the helper is referenced from another module, so it carries a public name
from .archive import resolve_archive_path

# Bad - reaching into another module for a symbol that module marked private
from .archive import _resolve_archive_path
```

When splitting or refactoring a module, every symbol that now crosses a module boundary is promoted
to a public name as part of the split. A symbol that stays behind keeps its underscore.

Tests are the sole exception. Test modules may access private members of the code under test, and
ruff ignores `SLF001` under `tests/`.

### Constants

Module-level constants with type annotations, descriptive names, and inline docstrings. Constants
intended for export (listed in `__all__`) use bare `UPPER_SNAKE_CASE`; constants internal to a
module use `_UPPER_SNAKE_CASE`:

```python
_MINIMUM_SAMPLE_COUNT: int = 100
"""Minimum number of samples required for statistical validity."""

MAXIMUM_QUANTIZATION_VALUE: int = 51
"""Maximum quantization value accepted by the encoder."""
```

---

## Function calls

**Always use keyword arguments** for clarity:

```python
# Good
np.zeros((4,), dtype=np.float32)
compute_coefficients(interpolation_factor=t, output=result)

# Avoid
np.zeros((4,), np.float32)
compute_coefficients(t, result)
```

Exceptions:
- Single positional arguments for obvious cases like `range(4)`, `len(array)`.
- Numba `jitclass` method calls, which do not support keyword arguments. Use positional
  arguments for these calls and add a brief inline comment if the call is not self-explanatory.
  Note: standard `@njit` / `@jit` functions do support keyword arguments and are not exempt
  from this rule.

On the signature side, make boolean flag parameters keyword-only by placing them after a `*,`
separator, so callers must pass them by name:

```python
def transfer_directory(source: Path, destination: Path, *, verify_integrity: bool = False,
                       remove_source: bool = False) -> None: ...
```

---

## Error handling

In projects that depend on `ataraxis-base-utilities`, use `console.error` for error reporting:

```python
from ataraxis_base_utilities import console

def process_data(self, data: NDArray[np.float32], threshold: float) -> None:
    if not (0 < threshold <= 1):
        message = (
            f"Unable to process data with the given threshold. The threshold must be in range "
            f"(0, 1], but got {threshold}."
        )
        console.error(message=message, error=ValueError)
```

In projects that do not depend on `ataraxis-base-utilities`, use standard `raise` with the same
message format.

`console.error` is typed `NoReturn` and always raises the supplied exception, so treat it as a
terminating call in guard clauses. Mypy understands the `NoReturn` contract, so it never asks for a
`raise` or a `return` after the call.

Ruff reasons about control flow syntactically and does not follow `NoReturn` through a call, so its
treatment depends on the enclosing return annotation. In a function annotated `-> None`, no `raise`
or `return` follows the call. In a function annotated with any other return type, ruff `RET503`
requires a terminating statement on the path that ends in `console.error`, and `RET503` is part of
the shared corpus that `lint.select = ["ALL"]` enables. Keep an unreachable `return` there, mark it
with `# pragma: no cover` so it stays outside the measured corpus, and leave the reason in a
comment:

```python
# Tail of a guard clause inside a function annotated '-> np.uint64'.
console.error(message=message, error=ValueError)
# Satisfies ruff RET503. console.error() is NoReturn, so this line never executes.
return np.uint64(0)  # pragma: no cover
```

### Error message format

- Start with context: "Unable to [action] using [input]."
- Explain the constraint: "The [parameter] must be [constraint]"
- Show actual value: "but got {value}."
- Use f-strings for interpolation
- Always assign the message to a `message` variable before passing to `console.error()` or `raise`

---

## Comments

### Inline comments

- Use third person imperative ("Configures..." not "This section configures...")
- Place above the code, not at end of line (unless very short)
- Use comments to explain non-obvious logic or provide context

```python
# The constant 2.046392675 is the theoretical injectivity bound for 2D cubic B-splines.
limit = (1.0 / 2.046392675) * self._grid_sampling * factor
```

### What to avoid

- Don't reiterate the obvious (e.g., `# Set x to 5` before `x = 5`)
- Don't add docstrings/comments to code you didn't write or modify
- Don't add type annotations as comments (use actual type hints)
- Don't use heavy section separator blocks (e.g., `# ======` or `# ------`)
- Don't use IDE-specific suppression comments (e.g., PyCharm `# noinspection ...`). Remove any you encounter — only
  ruff (`# noqa: CODE`) and mypy (`# type: ignore[code]`) suppressions are authoritative and must be preserved

---

## Imports

- All imports must be at the **top of the file**. Deferred or inline imports are not allowed.
- Import sorting and grouping is enforced by **ruff**. Do not manually reorder imports.

### Process-wide configuration above the imports

A setting that must be applied before the import it governs runs sits above the imports in the
top-level `__init__.py`, and every import below it carries `# noqa: E402`. Such a setting takes
effect only if it precedes the import, so moving it down changes behavior rather than formatting.
Configuration carrying no such ordering requirement stays below the imports. See
[class-patterns.md](references/class-patterns.md) for the qualifying settings and an example.

### Local import rules

All local (within-library) imports must directly import the required names:

```python
# Good - import specific names
from .spline_grid import SplineGrid
from .deformation import Deformation, zoom, diffuse

# Bad - importing the module itself
from . import spline_grid
```

### Cross-package vs within-package imports

**Cross-package imports** must go through the package's `__init__.py`:

```python
# Good - imports from the package's __init__.py
from ..configuration import RuntimeContext, SingleDayConfiguration

# Bad - imports directly from a submodule in another package
from ..configuration.single_day import RuntimeContext, SingleDayConfiguration
```

**Within-package imports** use direct module imports:

```python
from .spline_grid import SplineGrid
```

The rule binds the exporting side as well. Any symbol consumed outside the (sub)package that defines
it MUST be re-exported from that package's `__init__.py`, added to both the import list and `__all__`,
and imported through the package namespace rather than through the submodule that declares it. This
holds for internal implementation symbols and not only for the curated public API, so a subpackage
`__init__.py` may export a broader set than the distribution's top-level `__init__.py`. Exporting the
symbol and reaching past the export are two halves of one rule, and a cross-package consumer is
evidence that the export is missing.

Tests are the sole exception. Test modules may import directly from any submodule of any package.

---

## \_\_init\_\_.py conventions

A package's `__init__.py` defines the set of symbols that package offers to the rest of the
distribution. Every symbol consumed outside the package appears there twice, once in the import list
and once in `__all__`, whether the symbol belongs to the curated public API or exists only to serve a
sibling package. A subpackage that is missing an entry for a symbol another package already imports is
non-compliant, and the fix is the added export rather than a deeper import at the call site. The
distribution's top-level `__init__.py` stays narrower and re-exports the curated public API alone.

See [class-patterns.md](references/class-patterns.md) for top-level library and subpackage
`__init__.py` docstring, `__all__`, and console initialization conventions.

---

## File-level ordering

All definitions within a file follow this vertical ordering from top to bottom:

1. **Module docstring**
2. **Imports**
3. **Constants** (module-level `_UPPER_SNAKE_CASE` values)
4. **Enumerations and dataclasses** (type definitions that other code depends on)
5. **Public functions and classes** (no prefix)
6. **Private functions and classes** (`_` prefixed)

### Visibility ordering

Public definitions appear **above** private definitions. This matches the C-family convention
used across all projects (C#, C++), where the public API is presented first and
implementation details follow. Readers see the interface before the helpers that support it.

This rule governs two levels. At module level, public functions and classes precede private ones.
Inside a class body, public methods and properties precede private methods and properties, so a
private helper sits below every public member of its class rather than beside the member that calls
it. Dunder methods are exempt and keep their conventional position at the top of the class body,
directly after the class docstring. See [class-patterns.md](references/class-patterns.md) for the
full class member order.

### Call-hierarchy ordering

Within each visibility group, definitions should **loosely follow the order in which they are
called** during the library's runtime. When there is no clear call hierarchy, group definitions
**by purpose**.

### Enumerations and dataclasses first

Enumerations and dataclasses define the types that worker functions and classes operate on. They
must appear **above** the functions and classes that use them.

**Exception — dataclass-only modules**: In files whose primary product is the dataclasses
themselves, the order is: enumerations first, then public helper functions, then private helper
functions, then dataclasses at the bottom.

### Stub files

`.pyi` stub files and the `py.typed` marker are GENERATED, never hand-authored. `tox -e stubs`
produces them and they ship with releases; never create or hand-edit a stub — change typing by
editing the `.py` source and regenerating. If stale stubs are cluttering a dev session, purge them
with `tox -e lint` (`automation-cli purge-stubs`).

---

## Boolean expressions

Use truthiness checks instead of explicit comparisons to `True` or `False`:

```python
# Good - truthiness
if not self._is_enabled:
    return
if items:
    process(items=items)
if not file_list:
    console.error(message="No files found.", error=FileNotFoundError)

# Avoid - explicit boolean comparison
if self._is_enabled == True:   # Wrong
if self._is_enabled is True:   # Wrong
if len(items) > 0:             # Wrong - use truthiness instead
```

**Exception**: Always use `is None` / `is not None` for None checks, never truthiness:

```python
# Good - explicit None check
if self._data is not None:
    process(data=self._data)
```

---

## Guard clauses

Prefer early returns (guard clauses) over deeply nested conditionals:

```python
# Good - guard clauses reduce nesting
def process_session(self, data: NDArray[np.float32], threshold: float) -> NDArray[np.float32]:
    """Processes session data with the given threshold."""
    if not self._is_enabled:
        return data

    if data.size == 0:
        message = "Unable to process session data. The data array is empty."
        console.error(message=message, error=ValueError)

    # Main logic at minimal indentation level.
    filtered = data[data > threshold]
    return filtered
```

---

## Blank lines

- **Two blank lines** between top-level definitions (classes, functions)
- **One blank line** between method definitions within a class
- **No blank line** after a `def` line before the docstring
- **One blank line** after import blocks before code

---

## Line length and formatting

- Maximum line length: **120 characters**
- Break long function calls across multiple lines with trailing commas
- Use parentheses for multi-line strings in error messages

### String formatting

- **F-strings only**: Always use f-strings for string interpolation. No `%` formatting or
  `.format()`.
- **F-string consistency**: When any line requires interpolation, use the `f` prefix on **all**
  lines of the multi-line string.
- **Double quotes**: All strings must use double quotes (enforced by ruff). Single quotes are
  only acceptable inside f-string expressions.

### Trailing commas

- Always use trailing commas when the closing bracket is on a separate line
- Do not use trailing commas when everything is on one line

### Pathlib

Use `pathlib.Path` for all path manipulation instead of string operations:

```python
config_path = Path(base_directory) / "config" / "settings.yaml"
```

---

## Related skills

| Skill                   | Relationship                                                                  |
|-------------------------|-------------------------------------------------------------------------------|
| `/explore-dependencies` | Provides live ataraxis dependency API snapshots; invoke before using features |
| `/cpp-style`            | Provides C++ conventions; Python conventions parallel these                   |
| `/csharp-style`         | Provides C# conventions; Python conventions parallel these                    |
| `/tox-config`           | Provides tox environment conventions, including the parallel test-run flags   |
| `/pyproject-style`      | Provides pyproject.toml conventions, including the shared test-corpus ignores |
| `/audit-project`        | Audits the code just written, before it is committed                          |
| `/readme-style`         | Provides README conventions; invoke for README tasks                          |
| `/commit`               | Provides commit message conventions; invoke for commit tasks                  |
| `/skill-design`         | Provides skill file conventions; invoke for skill authoring tasks             |
| `/explore-codebase`     | Provides project context that informs style-compliant code changes            |

---

## Proactive behavior

When reviewing or modifying code, proactively check for style violations and fix them. When
writing new code, apply all conventions from this skill and its references without being asked.
If you notice existing code near your changes that violates conventions, mention it to the user
but do not fix it unless asked.

---

## Verification checklist

**You MUST verify your edits against this checklist before submitting any changes to Python
files.**

```text
Python Style Compliance:
(Scope: these items govern library code under src/. Test files under tests/ are linted too, but they
are held to this checklist as relaxed by the shared test corpus and by the project's own
tests/**/*.py per-file-ignores key: they may assert, access private members, import directly from
submodules, inline expected values, leave fixture arguments unreferenced, and omit docstrings and
annotations. Items naming a library-code construct, such as interface modules, console.enable(), or
__init__.py exports, do not apply to test files. Every remaining item applies to test files as
written. See /pyproject-style.)

Judgment items. No tool inspects these, so this checklist is their only enforcement. Walk every one
against the code you wrote.
- [ ] Docstring section order: Summary -> Extended Description -> Notes -> Args -> Returns -> Raises
- [ ] No Examples sections or in-code examples in docstrings
- [ ] Imperative mood in summaries ("Processes..." not "This method processes...")
- [ ] Prose used instead of bullet lists in docstrings
- [ ] No Sphinx specifiers (:class:, :func:, :meth:, :data:) outside MCP tool docstrings
- [ ] Sentences in comments and docstrings stay under 40 words
- [ ] Every member defaults to the summary line alone, with longer blocks earned by a nameable non-obvious property
- [ ] Each retained sentence survives the cover test (unable to be reconstructed from name, signature, and body)
- [ ] Documentation records current behavior only, never the edit that produced it
- [ ] Edits leave documentation no longer than it started unless the new behavior is harder to derive
- [ ] Documentation carries only facts the reader cannot infer from the code (no padding or trivia)
- [ ] Docstrings describe what the code does, with project usage and call-site context left out
- [ ] Peer-format expectations documented only when counter-intuitive enough to mislead the reader
- [ ] Docstring length proportional to conceptual difficulty, not to how long the function is
- [ ] Docstrings do not restate information already conveyed by the type signature
- [ ] Docstrings accurately describe the function's observable behavior
- [ ] Comments and docstrings free of typos and grammar errors
- [ ] Inline comments explain why, not what (no narrate-the-code comments)
- [ ] No stale references in comments (closed issues, removed code, outdated TODOs)
- [ ] Prose separators are full stops and commas only, no semicolons or em-dashes (colons and code syntax exempt)
- [ ] Documentation states what the code does, not what it is not or used to be (contrast only when load-bearing)
- [ ] Module docstring description is at most 5 sentences, with detail relocated into the members it documents
- [ ] NumPy arrays specify dtype explicitly (NDArray[np.float32])
- [ ] Full words used (no abbreviations like `pos`, `idx`, `val`)
- [ ] Private members use `_underscore` prefix
- [ ] Every symbol referenced from another module carries a public name, with the underscore
      reserved for symbols used only inside the module that defines them
- [ ] Every symbol consumed outside its defining (sub)package is re-exported from that package's
      __init__.py, in both the import block and __all__, and imported through the package namespace
- [ ] Keyword arguments used for function calls (except Numba `jitclass` method calls)
- [ ] Error handling uses console.error() when ataraxis-base-utilities is available (else raise)
- [ ] Local imports use direct name imports (no module imports)
- [ ] Cross-package imports go through package __init__.py (not submodules)
- [ ] Any pre-import configuration block is limited to settings that must precede the import they govern, states
      that requirement in a comment, and carries # noqa: E402 on the imports below it
- [ ] __init__.py files have __all__ (alphabetically sorted)
- [ ] Top-level library __init__.py has extended docstring (description, docs link, repo link, authors)
- [ ] Subpackage __init__.py files have single-line docstrings only (no links or authors)
- [ ] console.enable() / console.disable() confined to entry points that own the runtime, absent from ordinary
      worker paths reached by a downstream import (never reported as a finding on its own)
- [ ] Public definitions above private definitions, both at module level and inside each class body
- [ ] Dunder methods kept at the top of the class body, directly after the class docstring
- [ ] Enums and dataclasses above worker functions and classes
- [ ] Definitions ordered by call hierarchy or grouped by purpose
- [ ] Inline comments use third person imperative
- [ ] No heavy section separator blocks (# ====== or # ------)
- [ ] No IDE-specific suppression comments (PyCharm # noinspection etc.); only ruff # noqa / mypy # type: ignore kept
- [ ] Interface modules (cli.py, MCP tool modules, __main__.py) excluded via the pyproject omit list
- [ ] pragma: no cover used only for unreachable guards and platform-specific branches inside measured modules
- [ ] Each pragma: no cover annotates the narrowest construct that covers the excluded code
- [ ] Tests contending for a process-wide or on-disk resource carry @pytest.mark.xdist_group, and all
      mutually contending tests share one group name (flags owned by /tox-config)
- [ ] Numba functions use cache=True
- [ ] Decorator stacking order: @staticmethod/@classmethod, @njit, custom, @property
- [ ] Dataclasses use frozen=True for immutable configs (omit for mutable state)
- [ ] Dataclasses use slots=True by default (omit for YamlConfig subclasses or classes needing __dict__)
- [ ] Enum members have inline docstrings; StrEnum for strings, IntEnum for codes
- [ ] __repr__ uses ClassName(key=value) format; no __str__
- [ ] Boolean checks use truthiness (not == True); None checks use `is None`
- [ ] Guard clauses / early returns preferred over deep nesting
- [ ] I/O operations separated from processing logic
- [ ] Context managers used for resource management
- [ ] Pathlib used for path manipulation (not string concatenation)

Tooling-enforced items. Ruff resolves or reports each of these, so run `tox -e lint` rather than
hand-checking them. They stay listed for reviews performed without the linter.
- [ ] Google-style docstrings on all public and private members
- [ ] All parameters and returns have type annotations
- [ ] Type aliases use PEP 695 `type` statement syntax
- [ ] Double quotes used for all strings (enforced by ruff)
- [ ] F-strings used exclusively (no % formatting or .format())
- [ ] Lines under 120 characters
- [ ] All imports at top of file (no deferred or inline imports)
- [ ] Import sorting delegated to ruff (do not manually reorder)
- [ ] Two blank lines between top-level definitions
- [ ] Trailing commas in multi-line structures

Ataraxis Library Preferences (when ataraxis libraries are dependencies):
- [ ] Invoked /explore-dependencies to obtain current API snapshot for each ataraxis dependency
- [ ] Used ataraxis library features instead of standard library equivalents where available
- [ ] Console output uses console.echo() instead of print(); raw=True for pre-formatted content
- [ ] Error handling uses console.error() instead of raise (when ataraxis-base-utilities available)
- [ ] console.enable() / console.disable() placed at runtime-owning entry points rather than on ordinary worker paths
```
