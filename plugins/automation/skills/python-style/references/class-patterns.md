# Class and data patterns

Conventions for Python classes, dataclasses, enums, decorator ordering, and structural patterns in projects.

---

## Class design

### Constructor parameters

Use explicit parameters instead of tuples/dicts:

```python
# Good
def __init__(self, field_height: int, field_width: int, sampling: float) -> None:
    self._field_shape: tuple[int, int] = (field_height, field_width)

# Avoid
def __init__(self, field_shape: tuple[int, int], sampling: float) -> None:
    self._field_shape = field_shape
```

### Properties vs methods

- Use `@property` for simple attribute access that may involve computation
- Use a method when the call takes parameters, mutates state, or performs I/O

### Method types

- **Instance methods** (no decorator): Use when the method accesses instance attributes (`self`)
- **`@staticmethod`**: Use when the method doesn't access instance or class attributes
- **`@classmethod`**: Use when the method needs access to class attributes but not instance attributes

### Visibility (public vs private)

- **Private** (`_` prefix): Use for anything internal to the class/module
- **Public** (no prefix): Use only for methods intended to be used from other modules

A leading underscore marks a symbol private to the **module** that defines it. This covers functions, classes,
constants, class attributes, and methods.

Every symbol sits in exactly one of three tiers, and the tier is decided by the widest boundary the symbol's consumers
actually cross rather than by the visibility its author intended:

| Widest consumer                    | Name          | Listed in the package `__init__.py` |
|------------------------------------|---------------|-------------------------------------|
| The defining module alone          | `_underscore` | No                                  |
| Another module in the same package | public        | No                                  |
| Another package                    | public        | Yes                                 |

The name a symbol carries MATCHES its tier in both directions. Any symbol referenced from another module MUST carry a
public name, so a helper that acquires a caller in a second module is renamed rather than imported with its underscore
intact. Any symbol referenced only inside the module that defines it MUST carry the underscore, so a public name is
earned by a real cross-module consumer rather than granted by default:

```python
# Good - the helper is referenced from another module, so it carries a public name
from .archive import resolve_archive_path

# Bad - reaching into another module for a symbol that module marked private
from .archive import _resolve_archive_path
```

When splitting or refactoring a module, every symbol that now crosses a module boundary is promoted to a public name as
part of the split. A symbol that stays behind keeps its underscore, and a symbol whose last external caller went away is
demoted back.

Tests are the sole exception. Test modules may access private members of the code under test, and ruff ignores `SLF001`
under `tests/`. A test is therefore not a cross-module consumer for the purpose of the tiers above, so a symbol
referenced only by its defining module and by that module's tests keeps its underscore.

### Member ordering

Members within a class body follow this vertical ordering from top to bottom:

1. **Class-level fields** (dataclass fields, class constants)
2. **Dunder methods** (`__post_init__`, `__init__`, `__repr__`, `__del__`, `__getitem__`, and the rest)
3. **Public methods and properties** (no prefix)
4. **Private methods and properties** (`_` prefixed)

This mirrors the module-level visibility rule described in the `/python-style` File-level ordering section, for the same
reason: the reader meets the interface before the helpers that support it. A private helper therefore sits below every
public member of its class, even when only one public member calls it. Within the public group and within the private
group, order members by call hierarchy or group them by purpose.

Dunder methods are the one exception to visibility ordering. Their leading underscores mark a language protocol rather
than private visibility, so they stay at the top of the class body where readers expect to find construction and
representation.

```python
class ArchiveReader:
    """Reads messages from a serialized log archive."""

    def __init__(self, archive_path: Path) -> None: ...

    def __repr__(self) -> str: ...

    @property
    def message_count(self) -> int: ...

    def read_all_messages(self) -> tuple[NDArray[np.uint64], list[bytes]]: ...

    def _load_message_keys(self) -> list[str]: ...
```

### \_\_repr\_\_ conventions

Implement `__repr__` on classes to display the class name and key attributes. Do not implement `__str__` separately,
because `__repr__` serves both purposes.

```python
def __repr__(self) -> str:
    """Returns a string representation of the DataProcessor instance."""
    return (
        f"DataProcessor(data_path={self._data_path}, sampling_rate={self._sampling_rate}, "
        f"enabled={self._enabled})"
    )
```

Rules:
- Format: `ClassName(key_attr=value, key_attr=value)`
- Include the attributes that let a reader tell one instance from another, which are usually the constructor arguments,
  and leave out derived caches and buffers
- Use the actual class name, not a generic string
- Docstring uses third-person imperative mood: "Returns a string representation of the {ClassName} instance."

---

## Dataclass conventions

Use dataclasses for grouping related data.

```python
from dataclasses import dataclass, field

# Immutable configuration - use frozen=True and slots=True
@dataclass(frozen=True, slots=True)
class ExperimentConfig:
    """Defines configuration parameters for an experiment session."""

    animal_id: str
    """The unique identifier for the animal."""
    session_duration: float
    """The duration of the session in seconds."""
    trial_count: int = 10
    """The number of trials to run. Defaults to 10."""


# Mutable state tracker - omit frozen=True, still use slots=True
@dataclass(slots=True)
class ProcessingState:
    """Tracks the runtime state of a processing pipeline."""

    status: int = 0
    """The current processing status code."""
    completed_jobs: list[str] = field(default_factory=list)
    """The list of completed job identifiers."""

    def mark_complete(self, job_id: str) -> None:
        """Marks the specified job as complete."""
        self.completed_jobs.append(job_id)
```

### Guidelines

- Use `frozen=True` for configuration objects that should not be modified after creation
- Omit `frozen=True` for dataclasses that require mutation (state trackers, caches, builders)
- Use `slots=True` by default on all dataclasses. Slotted dataclasses use less memory, have faster attribute access, and
  prevent accidental attribute creation. Omit `slots=True` only when the dataclass subclasses a base that relies on
  `__dict__` for serialization or deserialization (e.g., `YamlConfig`), or when the class otherwise requires dynamic
  attribute assignment
- Use `field(default_factory=...)` for mutable default values (lists, dicts, sets)
- Use `field(repr=False)` for internal fields that should not appear in string representation
- Document each field with inline docstrings using triple-quoted strings
- Use `field(init=False)` for computed fields that are set in `__post_init__`

### \_\_post\_init\_\_ patterns

Use `__post_init__` for validation, type conversion, and computed field initialization:

```python
# YamlConfig subclasses must NOT use slots=True (YamlConfig relies on __dict__ for serialization)
@dataclass
class SessionConfig(YamlConfig):
    """Defines session configuration parameters."""

    session_type: str | SessionTypes
    """The session type identifier."""
    base_path: Path
    """The root path for session data."""
    data_path: Path = field(init=False)
    """The resolved path to session data files. Computed from base_path."""

    def __post_init__(self) -> None:
        """Validates configuration and computes derived fields."""
        # Converts string values loaded from YAML to proper enum types.
        if isinstance(self.session_type, str):
            self.session_type = SessionTypes(self.session_type)

        # Computes derived paths from base path.
        self.data_path = self.base_path / "data"
```

Common `__post_init__` uses:
- Converting string values from YAML deserialization to enum types
- Computing derived fields from other field values
- Validating field constraints with `console.error()`

---

## Enum conventions

### Base class selection

| Base Class | Use Case                                           | Example              |
|------------|----------------------------------------------------|----------------------|
| `StrEnum`  | Human-readable identifiers, string-based matching  | Session types, modes |
| `IntEnum`  | Fixed numeric codes, protocol values, status codes | Log IDs, MQTT codes  |

```python
from enum import IntEnum, StrEnum


class SessionTypes(StrEnum):
    """Defines the supported session types for experiment sessions."""

    LICK_TRAINING = "lick training"
    """Teaches animals to use the water delivery port."""
    RUN_TRAINING = "run training"
    """Trains animals to navigate the virtual corridor."""
    EXPERIMENT = "experiment"
    """Runs the full experiment with all stimuli active."""


class CameraLogIds(IntEnum):
    """Defines the log source identifiers for camera subsystems."""

    FRAME_DATA = 1
    """Identifies frame data packets in the binary log."""
    TIMESTAMP = 2
    """Identifies timestamp packets in the binary log."""
```

### Rules

- **Inline docstrings**: Document every enum member with a triple-quoted string on the line below
- **Class docstring**: Third-person imperative mood ("Defines the..."), do not use Args or Attributes sections
- **Value types**: Use string values for `StrEnum`, integer values for `IntEnum`
- **Naming**: PascalCase for the enum class name, UPPER_SNAKE_CASE for member names
- **Custom methods**: Add utility methods for type conversion when needed:

```python
class SerialProtocols(IntEnum):
    """Defines supported serial communication protocols."""

    COBS = 1
    """Consistent Overhead Byte Stuffing encoding."""

    def as_uint8(self) -> np.uint8:
        """Returns the protocol code as a numpy uint8 value."""
        return np.uint8(self.value)
```

---

## Decorator ordering

When stacking multiple decorators on a single method, use the following order (outermost to innermost):

```python
# 1. @staticmethod or @classmethod (outermost)
# 2. @numba.njit or other compilation decorators
# 3. Custom decorators
# 4. @property or other descriptors (innermost)

@staticmethod
@numba.njit(cache=True)  # type: ignore[untyped-decorator]
def _compute_values(
    target_buffer: NDArray[np.uint8],
    scalar_object: int,
    start_index: int,
) -> int:
    """Computes values for the target buffer."""
    ...
```

For Click CLI commands, stack decorators bottom-up (options closest to `def`, group/command on top):

```python
@cli_group.command("process")
@click.option("-i", "--input-path", required=True, type=click.Path(exists=True, path_type=Path))
@click.option("-o", "--output-path", required=True, type=click.Path(path_type=Path))
def process_data(input_path: Path, output_path: Path) -> None:
    """Processes the input data and saves the results."""
    ...
```

Add `# type: ignore[untyped-decorator]` comments for numba decorators that mypy cannot type-check.

---

## I/O separation

Separate I/O operations from processing logic. This makes code easier to test, maintain, and reuse.

```python
# Good - I/O separated from logic
def load_session_data(file_path: Path) -> NDArray[np.float32]:
    """Loads session data from file."""
    return np.load(file_path)

def analyze_session(data: NDArray[np.float32]) -> dict[str, float]:
    """Analyzes session data and returns statistics."""
    return {"mean": float(np.mean(data)), "std": float(np.std(data))}

# Avoid - I/O mixed with logic
def load_and_analyze(file_path: Path) -> dict[str, float]:
    """Loads and analyzes session data."""
    data = np.load(file_path)  # I/O operation
    return {"mean": float(np.mean(data))}  # Processing logic
```

### Guidelines

- I/O functions should only perform I/O (take filepath, return data)
- Processing functions should take standard data types and return standard data types
- This separation enables easier unit testing without file system dependencies

---

## Context managers

Use context managers (`with` statements) for resource management.

```python
# Good - use context managers for file handling
with open(file_path, "r") as file:
    data = file.read()

# Good - multiple context managers (Python 3.10+)
with (
    open(input_path, "r") as input_file,
    open(output_path, "w") as output_file,
):
    output_file.write(input_file.read())

# Good - context manager for temporary directory
with tempfile.TemporaryDirectory() as temp_dir:
    temp_path = Path(temp_dir) / "output.txt"
    process_data(output_path=temp_path)
```

### Guidelines

- Always use context managers for files, locks, database connections, and temporary resources
- Use parentheses for multiple context managers on separate lines
- Prefer context managers over try/finally for resource cleanup

---

## Comprehensions

Always prefer comprehensions over explicit loops when building a new collection. Comprehensions are algorithmically more
optimal because CPython executes them in a dedicated C-level loop, avoiding repeated `list.append()` method lookups and
call overhead.

```python
# Good - simple comprehension on one line
squares = [x ** 2 for x in range(10)]
valid_items = {key: value for key, value in data.items() if value > 0}

# Good - multi-line comprehension for longer expressions
filtered_data = [
    process_value(value)
    for value in raw_data
    if value > threshold
]

# Good - nested comprehension split across lines for readability
result = [
    [x * y for x in row if x > 0]
    for row in matrix
    if sum(row) > threshold
]
```

### Guidelines

- Always use comprehensions for building lists, dicts, and sets from iteration
- Split a comprehension across multiple lines when the one-line form exceeds the 120 character limit or carries more
  than one `for` or `if` clause
- Use generator expressions (`()`) instead of list comprehensions when the result is only iterated once (e.g., passed
  directly to `sum()`, `any()`, `all()`)
- Use explicit loops only when the loop body has **side effects** (I/O, mutation, logging) that do not produce a
  collection

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
- Numba `jitclass` method calls, which do not support keyword arguments. Use positional arguments for these calls and
  add a brief inline comment if the call is not self-explanatory. Note: standard `@njit` / `@jit` functions do support
  keyword arguments and are not exempt from this rule.

On the signature side, make boolean flag parameters keyword-only by placing them after a `*,` separator, so callers must
pass them by name:

```python
def transfer_directory(source: Path, destination: Path, *, verify_integrity: bool = False,
                       remove_source: bool = False) -> None: ...
```

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

- **F-strings only**: Always use f-strings for string interpolation. No `%` formatting or `.format()`.
- **F-string consistency**: When any line requires interpolation, use the `f` prefix on **all** lines of the multi-line
  string.
- **Double quotes**: All strings must use double quotes (enforced by ruff). Single quotes are only acceptable inside
  f-string expressions.

### Trailing commas

- Always use trailing commas when the closing bracket is on a separate line
- Do not use trailing commas when everything is on one line

### Pathlib

Use `pathlib.Path` for all path manipulation instead of string operations:

```python
config_path = Path(base_directory) / "config" / "settings.yaml"
```

---

## \_\_init\_\_.py conventions

There are two types of `__init__.py` files with different docstring requirements.

### Top-level library \_\_init\_\_.py

The top-level `__init__.py` (e.g., `src/library_name/__init__.py`) uses an extended docstring with documentation links
and authors:

```python
"""Provides assets for processing and analyzing neural imaging data.

See the `documentation <https://project-api-docs.netlify.app/>`_ for the description of
available assets. See the `source code repository <https://github.com/Sun-Lab-NBB/project-name>`_
for more details.

Authors: Author Name (Handle)
"""

from .module_one import ClassOne, function_one
from .module_two import ClassTwo, ClassThree

# console.enable() belongs here when this library owns the runtime it participates in. A library
# that runs only as a worker under another library's entry point leaves the call to that caller.

__all__ = [
    "ClassOne",
    "ClassThree",
    "ClassTwo",
    "function_one",
]
```

### Subpackage \_\_init\_\_.py

Subpackage `__init__.py` files (e.g., `src/library_name/subpackage/__init__.py`) use a single-line docstring only:

```python
"""Provides configuration and runtime data classes for the processing pipeline."""

from .config import Config, Settings
from .data import DataStore

__all__ = [
    "Config",
    "DataStore",
    "Settings",
]
```

### Rules

- **Top-level docstring**: The first line MUST be the bare project description, the same sentence used in all other
  canonical description locations (`pyproject.toml`, `welcome.rst`, `README.md`) with no language or project name
  prefix. Include documentation link, source repository link, and authors. Email addresses in the `Authors:` line are
  optional and omitted by default
- **Subpackage docstring**: Use a single-line docstring describing what the subpackage provides. Do NOT include
  documentation links, source repository links, or authors, which belong only in the top-level library `__init__.py`
- **Console initialization**: the test for `console.enable()` and `console.disable()` is whether the code that calls it
  owns the runtime at that moment. An entry point that the user invokes directly, such as a CLI command, an MCP server,
  or the `__init__.py` of a library that drives its own pipeline, calls it and is correct to. Code that runs as a worker
  under another library's entry point leaves the console state to that caller, so the call does not appear on ordinary
  library paths reached by a downstream import. Both placements are legitimate, and neither is reported as a style
  finding unless the user names it as one in that specific case
- **Explicit `__all__`**: Every `__init__.py` must declare `__all__` with all public API members
- **Export set**: The import list and `__all__` hold exactly the symbols other packages import, which covers internal
  implementation symbols and lets a subpackage `__init__.py` list more than the top-level library `__init__.py`. The
  `/python-style` Cross-package vs within-package imports section states the importing half of the rule, and the two
  paragraphs below state the exporting half

Any symbol consumed outside the (sub)package that defines it MUST be re-exported from that package's `__init__.py`,
added to both the import list and `__all__`, and imported through the package namespace rather than through the
submodule that declares it. This holds for internal implementation symbols and not only for the curated public API, so a
subpackage `__init__.py` may export a broader set than the distribution's top-level `__init__.py`. Exporting the symbol
and reaching past the export are two halves of one rule, and a cross-package consumer is evidence that the export is
missing.

The same test that requires an export also BOUNDS it. A symbol that no package outside the defining one consumes does
NOT appear in that package's `__init__.py`, in either the import list or `__all__`. The absence of a cross-package
consumer is evidence that the export is unwarranted, and the fix is the removed entry rather than a caller invented to
justify it.
- **Manual check**: `per-file-ignores` waives `F401` for `**/__init__.py`, so the export list is checked by reading it
  against the set of packages that import from it
- **Alphabetical sorting**: Sort `__all__` entries alphabetically
- **One-time configuration logic**: `__init__.py` files may contain logic that benefits from being executed exactly once
  on import (e.g., setting the multiprocessing start method, configuring environment variables for platform
  compatibility). See the placement rule below. Beyond that, `__init__.py` files should contain only imports and
  `__all__`

### Process-wide configuration above the imports

A setting that must run before the import it governs sits above the imports, and every import below it carries `# noqa:
E402`. The qualifying settings are the multiprocessing start method, the Numba threading layer, an environment variable
that a dependency reads at import time, and the level of a third-party logger that emits during its own import.

```python
# Configures the numba threading layer before any numba function compiles. macOS uses OpenMP because
# tbb4py publishes no Apple Silicon wheel, and every other platform uses TBB.
import sys

from numba import config

config.THREADING_LAYER = "omp" if sys.platform == "darwin" else "tbb"

from .pipelines import run_pipeline  # noqa: E402
```

The block states in a comment what it configures and why the position is required, because the position is the only
thing keeping the setting effective. A reader who cannot see that reason moves the block down during an unrelated
cleanup and silently disables the setting.
