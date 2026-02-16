# Class and data patterns

Conventions for Python classes, dataclasses, enums, decorator ordering, and structural patterns in Sun Lab projects.

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
- Use methods for operations that clearly "do something" or take parameters

### Method types

- **Instance methods** (no decorator): Use when the method accesses instance attributes (`self`)
- **`@staticmethod`**: Use when the method doesn't access instance or class attributes
- **`@classmethod`**: Use when the method needs access to class attributes but not instance attributes

### Visibility (public vs private)

- **Private** (`_` prefix): Use for anything internal to the class/module
- **Public** (no prefix): Use only for methods intended to be used from other modules

### \_\_repr\_\_ conventions

Implement `__repr__` on classes to display the class name and key attributes. Do not implement `__str__` separately
— `__repr__` serves both purposes.

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
- Include only the most important attributes, not every internal field
- Use the actual class name, not a generic string
- Docstring uses imperative mood: "Returns a string representation of the {ClassName} instance."

---

## Dataclass conventions

Use dataclasses for grouping related data.

```python
from dataclasses import dataclass, field

# Immutable configuration - use frozen=True
@dataclass(frozen=True)
class ExperimentConfig:
    """Defines configuration parameters for an experiment session."""

    animal_id: str
    """The unique identifier for the animal."""
    session_duration: float
    """The duration of the session in seconds."""
    trial_count: int = 10
    """The number of trials to run. Defaults to 10."""


# Mutable state tracker - omit frozen=True
@dataclass
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
- Use `field(default_factory=...)` for mutable default values (lists, dicts, sets)
- Use `field(repr=False)` for internal fields that should not appear in string representation
- Document each field with inline docstrings using triple-quoted strings
- Use `field(init=False)` for computed fields that are set in `__post_init__`

### \_\_post\_init\_\_ patterns

Use `__post_init__` for validation, type conversion, and computed field initialization:

```python
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

| Base Class | Use Case                                           | Example                |
|------------|----------------------------------------------------|------------------------|
| `StrEnum`  | Human-readable identifiers, string-based matching  | Session types, modes   |
| `IntEnum`  | Fixed numeric codes, protocol values, status codes | Log IDs, MQTT codes    |

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
- **Class docstring**: Imperative mood ("Defines the..."), do not use Args or Attributes sections
- **Value types**: Use string values for `StrEnum`, integer values for `IntEnum`
- **Naming**: UPPER_SNAKE_CASE for member names
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
@numba.njit(nogil=True, cache=True)  # type: ignore[untyped-decorator]
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

Always prefer comprehensions over explicit loops when building a new collection. Comprehensions
are algorithmically more optimal because CPython executes them in a dedicated C-level loop,
avoiding repeated `list.append()` method lookups and call overhead.

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
- Split complex comprehensions across multiple lines for readability
- Use generator expressions (`()`) instead of list comprehensions when the result is only
  iterated once (e.g., passed directly to `sum()`, `any()`, `all()`)
- Use explicit loops only when the loop body has **side effects** (I/O, mutation, logging) that
  do not produce a collection
