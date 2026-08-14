---
name: python-style
description: >-
  Applies Python coding conventions when writing, reviewing, or refactoring code. Covers .py files, docstrings, type
  annotations, naming, formatting, error handling, imports, file ordering, and ataraxis library preferences. Use when
  writing new Python code, modifying existing code, reviewing pull requests, or when the user asks about Python coding
  standards.
user-invocable: false
---

# Python code style guide

Applies Python coding conventions.

You MUST read this skill and load the relevant reference files before writing or modifying Python code. You MUST verify
your changes against the checklist before submitting.

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

| Task                                                                                             | Reference to load                                             |
|--------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| Writing or modifying any docstring, comment, or type                                             | [docstrings-and-types.md](references/docstrings-and-types.md) |
| Writing classes, dataclasses, enums, `__init__.py`, function calls, guard clauses, or formatting | [class-patterns.md](references/class-patterns.md)             |
| Using ataraxis libs, Numba, Click, tests                                                         | [libraries-and-tools.md](references/libraries-and-tools.md)   |
| Using ataraxis library features                                                                  | Invoke `/explore-dependencies` first, then load above         |
| Reviewing code before submission                                                                 | [anti-patterns.md](references/anti-patterns.md)               |

Load multiple references when the task spans multiple domains.

### Step 3: Apply conventions

Write or modify Python code following all conventions from this file and the loaded references.

### Step 4: Verify compliance

Complete the verification checklist at the end of this file. Every item must pass before submitting work. For
anti-pattern examples, load [anti-patterns.md](references/anti-patterns.md).

---

## Cross-language consistency

Projects span Python, C++, and C#. These conventions maximize visual and structural consistency across languages while
respecting each language's idiomatic standards.

**Shared across all languages:**
- 120 character line limit, with wrapped prose filled to that limit rather than broken at a narrower width
- 4-space indentation (no tabs)
- Comprehensive documentation on ALL public and private members
- Third-person imperative mood for documentation ("Provides...", "Determines whether...")
- A leading underscore marks a symbol private to the module or class that defines it (`_snake_case` for any private
  Python symbol, `_snake_case` for C++ data members, `_camelCase` for C# fields)
- Full words in identifiers (no abbreviations)
- Guard clauses preferred over deep nesting
- Prose over bullet lists in documentation
- No example/code blocks in documentation (they go stale)
- I/O operations separated from processing logic
- Only full stops and commas separate clauses in documentation prose (no semicolons, no em-dashes)
- State what the code does and what is currently true, not what it is not or used to be (contrast only when
  load-bearing)

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

- Use snake_case for every function and method, both public and private
- Use descriptive verb phrases: `compute_coefficients`, `extract_features`
- Private functions start with underscore: `_process_batch`, `_validate_input`
- Avoid generic names like `process`, `handle`, `do_something`

### Visibility prefixes and the module boundary

A leading underscore marks a symbol private to the **module** that defines it, covering functions, classes, constants,
class attributes, and methods. The name a symbol carries MATCHES the widest boundary its consumers actually cross, so
any symbol referenced from another module MUST carry a public name and any symbol referenced only inside its defining
module MUST carry the underscore.

See [class-patterns.md](references/class-patterns.md) for the three visibility tiers, the promotion and demotion that
follow a module split, and the test exception.

### Constants

Module-level constants with type annotations, descriptive names, and inline docstrings. Constants intended for export
(listed in `__all__`) use bare `UPPER_SNAKE_CASE`, and constants internal to a module use `_UPPER_SNAKE_CASE`:

```python
_MINIMUM_SAMPLE_COUNT: int = 100
"""Minimum number of samples required for statistical validity."""

MAXIMUM_QUANTIZATION_VALUE: int = 51
"""Maximum quantization value accepted by the encoder."""
```

---

## Function calls

See [class-patterns.md](references/class-patterns.md) for keyword-argument conventions, their exceptions, and the
keyword-only placement of boolean flag parameters.

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

In projects that do not depend on `ataraxis-base-utilities`, use standard `raise` with the same message format.

`console.error` is typed `NoReturn` and always raises the supplied exception, so treat it as a terminating call in guard
clauses. Mypy understands the `NoReturn` contract, so it never asks for a `raise` or a `return` after the call. Ruff
instead reasons syntactically, so a function annotated with a return type other than `None` needs an unreachable
`return` after the call to satisfy ruff `RET503`. See [libraries-and-tools.md](references/libraries-and-tools.md) for
that pattern and the `# pragma: no cover` annotation it carries.

### Error message format

- Start with context: "Unable to [action] using [input]."
- Explain the constraint: "The [parameter] must be [constraint]"
- Show actual value: "but got {value}."
- Use f-strings for interpolation
- Always assign the message to a `message` variable before passing to `console.error()` or `raise`

---

## Comments

See [docstrings-and-types.md](references/docstrings-and-types.md) for inline comment conventions and what to avoid.

---

## Imports

- All imports must be at the **top of the file**. Deferred or inline imports are not allowed.
- Import sorting and grouping is enforced by **ruff**. Do not manually reorder imports.

### Process-wide configuration above the imports

A setting that must be applied before the import it governs runs sits above the imports in the top-level `__init__.py`,
and every import below it carries `# noqa: E402`. Such a setting takes effect only if it precedes the import, so moving
it down changes behavior rather than formatting. Configuration carrying no such ordering requirement stays below the
imports. See [class-patterns.md](references/class-patterns.md) for the qualifying settings and an example.

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

The rule binds the exporting side as well. Any symbol consumed outside the (sub)package that defines it MUST be
re-exported from that package's `__init__.py`, and a symbol that no outside package consumes MUST NOT appear there. See
[class-patterns.md](references/class-patterns.md) for both halves of that rule.

Tests are the sole exception. Test modules may import directly from any submodule of any package.

---

## \_\_init\_\_.py conventions

See [class-patterns.md](references/class-patterns.md) for top-level library and subpackage `__init__.py` docstring,
`__all__`, and console initialization conventions.

---

## Unused assets

An asset with no consumer is REMOVED rather than kept. This covers functions, classes, methods, properties, constants,
enum members, dataclass fields, type aliases, parameters, and whole modules.

Ruff reports only the cases a single file reveals, which are unused imports outside `__init__.py` (`F401`), unused local
variables (`F841`), and unused arguments (`ARG`). It carries no rule for an unused module-level definition, so a
function, class, or constant nobody calls clears every gate the project runs and is found by reading alone.

Three things count as a consumer, and nothing else does:

- A reference from library code under `src/`
- An entry in the distribution's top-level `__init__.py` `__all__`, which places the symbol in the curated public API
  and hands it to downstream code this repository cannot see
- A registration the interpreter resolves at runtime rather than by name, which covers `pyproject.toml` entry points,
  Click commands, MCP tool registrations, plugin registries, and `getattr` dispatch

A reference from `tests/` alone is NOT a consumer. A helper that only its own tests exercise is dead library code with a
live test, and the pair is removed together.

Removing a symbol from the curated public API is a breaking change, so it waits for a release permitted to break the API
rather than landing as a cleanup.

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

Public definitions appear **above** private definitions, matching the C-family convention used across all projects (C#,
C++) that presents the interface before the helpers supporting it.

This rule governs two levels. At module level, public functions and classes precede private ones. Inside a class body,
public methods and properties precede private methods and properties, so a private helper sits below every public member
of its class rather than beside the member that calls it. Dunder methods are exempt and keep their conventional position
at the top of the class body, directly after the class docstring. See [class-patterns.md](references/class-patterns.md)
for the full class member order.

### Call-hierarchy ordering

Within each visibility group, definitions should **loosely follow the order in which they are called** during the
library's runtime. When there is no clear call hierarchy, group definitions **by purpose**.

### Enumerations and dataclasses first

Enumerations and dataclasses define the types that worker functions and classes operate on. They must appear **above**
the functions and classes that use them.

**Exception, dataclass-only modules**: In files whose primary product is the dataclasses themselves, the order is:
enumerations first, then public helper functions, then private helper functions, then dataclasses at the bottom.

### Stub files

`.pyi` stub files and the `py.typed` marker are GENERATED, never hand-authored, and they ship with releases. Change
typing by editing the `.py` source and regenerating, rather than by creating or hand-editing a stub. See `/tox-config`
for the environments that generate and purge stubs.

---

## Boolean expressions and guard clauses

See [class-patterns.md](references/class-patterns.md) for truthiness checks, the `is None` exception, and early-return
conventions.

---

## Blank lines, line length, and formatting

### Wrap width

Break a comment or docstring line only where it would otherwise pass 120 characters, and fill each line to that limit
before breaking. Prose wrapped at a narrower width reads as a rigid block, re-wraps badly at any other viewport, and
advertises a limit the project does not set. The test is mechanical: a wrapped line that ends before column 100 while
its next word would still fit under 120 is re-flowed. A line ending early because the sentence or the paragraph ends, or
because it holds a table row, a list item, or a code span, is already correct.

See [class-patterns.md](references/class-patterns.md) for blank-line placement, the 120 character limit, string
formatting, trailing commas, and pathlib usage.

---

## Related skills

| Skill                   | Relationship                                                                           |
|-------------------------|----------------------------------------------------------------------------------------|
| `/explore-dependencies` | Provides live ataraxis dependency API snapshots, invoked before using library features |
| `/cpp-style`            | Provides C++ conventions that Python conventions parallel                              |
| `/csharp-style`         | Provides C# conventions that Python conventions parallel                               |
| `/tox-config`           | Provides tox environment conventions, including the parallel test-run flags            |
| `/pyproject-style`      | Provides pyproject.toml conventions, including the shared test-corpus ignores          |
| `/audit-project`        | Audits the code just written, before it is committed                                   |
| `/readme-style`         | Provides README conventions, invoked for README tasks                                  |
| `/commit`               | Provides commit message conventions, invoked for commit tasks                          |
| `/skill-design`         | Provides skill file conventions, invoked for skill authoring tasks                     |
| `/explore-codebase`     | Provides project context that informs style-compliant code changes                     |

---

## Proactive behavior

When reviewing or modifying code, proactively check for style violations and fix them. When writing new code, apply all
conventions from this skill and its references without being asked. If you notice existing code near your changes that
violates conventions, mention it to the user but do not fix it unless asked.

---

## Verification checklist

**You MUST verify your edits against this checklist before submitting any changes to Python files.**

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
- [ ] Class docstrings carry an Attributes section listing every instance attribute (dataclasses and enums
      document their fields inline instead)
- [ ] No Examples sections or in-code examples in docstrings
- [ ] Third-person imperative mood in summaries ("Processes..." not "This method processes...")
- [ ] Boolean parameters and attributes documented with "Determines whether..." (boolean properties use "Returns...")
- [ ] Prose used instead of bullet lists in docstrings
- [ ] No Sphinx specifiers (:class:, :func:, :meth:, :data:) outside MCP tool docstrings
- [ ] Sentences in comments and docstrings stay under 40 words
- [ ] Every member defaults to the summary line alone, with longer blocks earned by a nameable non-obvious property
- [ ] @property docstrings are a single sentence (no summary plus extended-description split)
- [ ] Test functions carry the summary line only, with no Args, Returns, or Raises sections
- [ ] Click command docstrings carry the summary and prose only, with no Notes, Args, Returns, or Raises sections
- [ ] MCP tool docstrings carry a Returns section describing the response structure, naming the keys in prose
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
- [ ] Prose separators are full stops and commas only, no semicolons or em-dashes (colons, hyphen bullets, and code
      syntax exempt)
- [ ] Documentation states what the code does, not what it is not or used to be (contrast only when load-bearing)
- [ ] Module docstring description is at most 5 sentences, with detail relocated into the members it documents
- [ ] NumPy arrays specify dtype explicitly (NDArray[np.float32])
- [ ] Full words used (no abbreviations like `pos`, `idx`, `val`)
- [ ] Private members use `_underscore` prefix
- [ ] Constants use UPPER_SNAKE_CASE (bare when exported in __all__, _UPPER_SNAKE_CASE when module-internal)
- [ ] Enum classes use PascalCase; enum members use UPPER_SNAKE_CASE
- [ ] Every symbol referenced from another module carries a public name, and every symbol referenced
      only inside its defining module carries the underscore (tests are not cross-module consumers)
- [ ] Every symbol consumed outside its defining (sub)package is re-exported from that package's
      __init__.py, in both the import block and __all__, and imported through the package namespace
- [ ] No symbol appears in a package's __init__.py import block or __all__ unless a package outside
      that one imports it, with the top-level __init__.py carrying the curated public API alone
- [ ] Every asset has a consumer, so functions, classes, methods, constants, enum members, fields,
      type aliases, and whole modules that nothing under src/ references are removed rather than kept
- [ ] A symbol exercised only by its own tests is removed together with those tests
- [ ] Keyword arguments used for function calls (except Numba `jitclass` method calls)
- [ ] Boolean flag parameters declared keyword-only behind a `*,` separator in the signature
- [ ] Error handling uses console.error() when ataraxis-base-utilities is available (else raise)
- [ ] Error messages assigned to a `message` variable before passing to console.error() or raise
- [ ] Error messages state the failed action, the violated constraint, and the actual value received
- [ ] Invoked /explore-dependencies for a current API snapshot of each ataraxis dependency in use
- [ ] Ataraxis library features used in place of standard library equivalents wherever the project
      depends on the library that provides them
- [ ] Console output uses console.echo() instead of print() when ataraxis-base-utilities is
      available, with raw=True for pre-formatted content
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
- [ ] No hand-authored .pyi stubs or py.typed markers, with typing changed in the .py source and regenerated
- [ ] Inline comments use third-person imperative mood
- [ ] No heavy section separator blocks (# ====== or # ------)
- [ ] No IDE-specific suppression comments (PyCharm # noinspection etc.); only ruff # noqa / mypy # type: ignore kept
- [ ] Interface modules (cli.py, MCP tool modules, __main__.py) excluded via the pyproject omit list
- [ ] pragma: no cover used only for unreachable guards, hardware paths, and platform branches in measured modules
- [ ] Each pragma: no cover annotates the narrowest construct that covers the excluded code
- [ ] Tests contending for a process-wide or on-disk resource carry @pytest.mark.xdist_group, and all
      mutually contending tests share one group name (flags owned by /tox-config)
- [ ] Numba functions use cache=True
- [ ] @staticmethod used when a method touches neither self nor cls, @classmethod when it touches cls alone
- [ ] Decorator stacking order: @staticmethod/@classmethod, @njit, custom, @property
- [ ] Click options use lowercase short and hyphenated long flags, with click.Path() carrying explicit validation,
      and command decorators stacked bottom-up with options closest to `def`
- [ ] Dataclasses use frozen=True for immutable configs (omit for mutable state)
- [ ] Dataclasses use slots=True by default (omit when subclassing a non-slotted base such as YamlConfig)
- [ ] Enum members have inline docstrings; StrEnum for strings, IntEnum for codes
- [ ] __repr__ uses ClassName(key=value) format; no __str__
- [ ] Boolean checks use truthiness (not == True); None checks use `is None`
- [ ] Guard clauses / early returns preferred over deep nesting
- [ ] Comprehensions used to build new collections, with generator expressions where the result is iterated once
      and explicit loops kept for side-effecting bodies
- [ ] I/O operations separated from processing logic
- [ ] Context managers used for resource management
- [ ] Pathlib used for path manipulation (not string concatenation)

Tooling-enforced items. Ruff resolves or reports each of these, so run `tox -e lint` rather than
hand-checking them. They stay listed for reviews performed without the linter.
- [ ] Google-style docstrings on all public and private members
- [ ] All parameters and returns have type annotations
- [ ] Type aliases use PEP 695 `type` statement syntax
- [ ] Functions and methods use snake_case (both public and private, the private ones underscore-prefixed)
- [ ] Double quotes used for all strings (enforced by ruff)
- [ ] F-strings used exclusively (no % formatting or .format())
- [ ] Lines under 120 characters, with wrapped prose filled to that limit rather than broken at a narrower width
- [ ] 4-space indentation, no tabs
- [ ] All imports at top of file (no deferred or inline imports)
- [ ] Import sorting delegated to ruff (do not manually reorder)
- [ ] Two blank lines between top-level definitions
- [ ] Trailing commas in multi-line structures
- [ ] ruff format applied before commit (tox -e lint)
```
