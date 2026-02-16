---
name: python-style
description: >-
  Applies Sun Lab Python coding conventions when writing, reviewing, or refactoring code. Covers
  .py files, docstrings, type annotations, naming, formatting, error handling, imports, file
  ordering, and ataraxis library preferences. Use when writing new Python code, modifying existing
  code, reviewing pull requests, or when the user asks about Python coding standards.
user-invocable: true
---

# Python code style guide

Applies Sun Lab Python coding conventions.

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
- README file conventions (invoke `/readme-style`)
- Commit message conventions (invoke `/commit`)
- Skill file and CLAUDE.md conventions (invoke `/skill-design`)
- Codebase exploration workflows (invoke `/explore-codebase`)

---

## Workflow

You MUST follow these steps when this skill is invoked.

### Step 1: Read this skill

Read this entire file. The core conventions below apply to ALL Python code.

### Step 2: Load relevant reference files

Based on the task, load the appropriate reference files:

| Task                                     | Reference to load                                             |
|------------------------------------------|---------------------------------------------------------------|
| Writing or modifying docstrings/types    | [docstrings-and-types.md](references/docstrings-and-types.md) |
| Writing classes, dataclasses, or enums   | [class-patterns.md](references/class-patterns.md)             |
| Using ataraxis libs, Numba, Click, tests | [libraries-and-tools.md](references/libraries-and-tools.md)   |
| Reviewing code before submission         | [anti-patterns.md](references/anti-patterns.md)               |

Load multiple references when the task spans multiple domains.

### Step 3: Apply conventions

Write or modify Python code following all conventions from this file and the loaded references.

### Step 4: Verify compliance

Complete the verification checklist at the end of this file. Every item must pass before
submitting work. For anti-pattern examples, load
[anti-patterns.md](references/anti-patterns.md).

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

### Constants

Module-level constants with type annotations and descriptive names:

```python
# Minimum number of samples required for statistical validity.
_MINIMUM_SAMPLE_COUNT: int = 100
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

Exception: Single positional arguments for obvious cases like `range(4)`, `len(array)`.

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
message format:

```python
raise ValueError(message)
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

---

## Imports

- All imports must be at the **top of the file**. Deferred or inline imports are not allowed.
- Import sorting and grouping is enforced by **ruff**. Do not manually reorder imports.

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

---

## \_\_init\_\_.py conventions

Package `__init__.py` files define the public API:

```python
"""Provides assets for processing and analyzing neural imaging data.

See the `documentation <https://project-api-docs.netlify.app/>`_ for the description of
available assets. See the `source code repository <https://github.com/Sun-Lab-NBB/project-name>`_
for more details.

Authors: Author Name (Handle)
"""

from .module_one import ClassOne, function_one
from .module_two import ClassTwo, ClassThree

# Optional, per-library choice:
# from ataraxis_base_utilities import console
#
# if not console.enabled:
#     console.enable()

__all__ = [
    "ClassOne",
    "ClassThree",
    "ClassTwo",
    "function_one",
]
```

### Rules

- **Module docstring**: The first line MUST be the bare project description — the same sentence
  used in all other canonical description locations (`pyproject.toml`, `welcome.rst`, `README.md`)
  with no language or project name prefix. Include documentation link, source repository link,
  and authors. Email addresses in the `Authors:` line are optional and omitted by default
- **Console initialization**: Enabling the global `console` in the top-level `__init__.py` is a
  per-library choice, not a strict requirement. If `console.echo()` is called elsewhere in the
  library without `console.enable()` present, verify with the user whether this is intentional
- **Explicit `__all__`**: Every `__init__.py` must declare `__all__` with all public API members
- **Alphabetical sorting**: Sort `__all__` entries alphabetically
- **No logic**: `__init__.py` files contain only imports, optional `console.enable()`, and `__all__`

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
used across all Sun Lab projects (C#, C++), where the public API is presented first and
implementation details follow. Readers see the interface before the helpers that support it.

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

| Skill               | Relationship                                                       |
|---------------------|--------------------------------------------------------------------|
| `/readme-style`     | Provides README conventions; invoke for README tasks               |
| `/commit`           | Provides commit message conventions; invoke for commit tasks       |
| `/skill-design`     | Provides skill file conventions; invoke for skill authoring tasks  |
| `/explore-codebase` | Provides project context that informs style-compliant code changes |

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
- [ ] Google-style docstrings on all public and private members
- [ ] Docstring section order: Summary -> Extended Description -> Notes -> Args -> Returns -> Raises
- [ ] No Examples sections or in-code examples in docstrings
- [ ] Imperative mood in summaries ("Processes..." not "This method processes...")
- [ ] Prose used instead of bullet lists in docstrings
- [ ] All parameters and returns have type annotations
- [ ] NumPy arrays specify dtype explicitly (NDArray[np.float32])
- [ ] Type aliases use PEP 695 `type` statement syntax
- [ ] Full words used (no abbreviations like `pos`, `idx`, `val`)
- [ ] Private members use `_underscore` prefix
- [ ] Keyword arguments used for function calls
- [ ] Error handling uses console.error() when ataraxis-base-utilities is available (else raise)
- [ ] Double quotes used for all strings (enforced by ruff)
- [ ] F-strings used exclusively (no % formatting or .format())
- [ ] Lines under 120 characters
- [ ] All imports at top of file (no deferred or inline imports)
- [ ] Import sorting delegated to ruff (do not manually reorder)
- [ ] Local imports use direct name imports (no module imports)
- [ ] Cross-package imports go through package __init__.py (not submodules)
- [ ] __init__.py files have __all__ (alphabetically sorted); console.enable() is a per-library choice
- [ ] If console.echo() is used without console.enable() anywhere, verify intent with user
- [ ] Public definitions above private definitions in file
- [ ] Enums and dataclasses above worker functions and classes
- [ ] Definitions ordered by call hierarchy or grouped by purpose
- [ ] Inline comments use third person imperative
- [ ] No heavy section separator blocks (# ====== or # ------)
- [ ] Numba functions use cache=True
- [ ] Decorator stacking order: @staticmethod/@classmethod, @njit, custom, @property
- [ ] Dataclasses use frozen=True for immutable configs (omit for mutable state)
- [ ] Enum members have inline docstrings; StrEnum for strings, IntEnum for codes
- [ ] __repr__ uses ClassName(key=value) format; no __str__
- [ ] Boolean checks use truthiness (not == True); None checks use `is None`
- [ ] Guard clauses / early returns preferred over deep nesting
- [ ] I/O operations separated from processing logic
- [ ] Context managers used for resource management
- [ ] Pathlib used for path manipulation (not string concatenation)
- [ ] Two blank lines between top-level definitions
- [ ] Trailing commas in multi-line structures

Ataraxis Library Preferences (when ataraxis-base-utilities is a dependency):
- [ ] Console output uses console.echo() instead of print() (exception: tabulate/formatted tables)
- [ ] Error handling uses console.error() instead of raise
- [ ] List conversion uses ensure_list() instead of manual type checks
- [ ] Batch iteration uses chunk_iterable() instead of manual slicing
- [ ] Worker count uses resolve_worker_count() instead of os.cpu_count() arithmetic
- [ ] Parallel job capacity uses resolve_parallel_job_capacity() instead of manual core division
- [ ] Scalar byte serialization uses convert_scalar_to_bytes()/convert_bytes_to_scalar()
- [ ] Array byte serialization uses convert_array_to_bytes()/convert_bytes_to_array()
- [ ] Precision timing uses PrecisionTimer instead of time.time()/perf_counter()
- [ ] Delays use PrecisionTimer.delay() instead of time.sleep()
- [ ] Timestamps use get_timestamp() instead of datetime.strftime()
- [ ] Time unit conversion uses convert_time() instead of manual math
- [ ] YAML-serializable configs subclass YamlConfig
- [ ] Inter-process arrays use SharedMemoryArray instead of multiprocessing.Array
- [ ] High-throughput logging uses DataLogger/LogPackage instead of direct writes
```
