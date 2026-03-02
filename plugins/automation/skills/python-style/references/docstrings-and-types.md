# Docstrings and type annotations

Detailed conventions for Python docstrings and type annotations in Sun Lab projects.

---

## Docstrings

Use **Google-style docstrings** with sections in this order:
**Summary -> Extended Description -> Notes -> Args -> Returns -> Raises**

```python
def function_name(param1: int, param2: str = "default") -> bool:
    """Brief one-line summary of what the function does.

    Extended description goes here if needed. This is optional and should only be used when the
    function's behavior is too complex to be fully explained by the summary line. Most functions
    should not need this section.

    Notes:
        Additional context, background, or implementation details. Use this for explaining algorithms,
        referencing papers, or clarifying non-obvious behavior. Multi-sentence explanations go here.

    Args:
        param1: Description without repeating the type (types are in signature).
        param2: Description of parameter with default value behavior if relevant.

    Returns:
        Description of return value. For simple returns, one line is sufficient. For complex returns
        (tuples, dicts), describe each element in prose form.

    Raises:
        ValueError: When this error occurs and why.
        TypeError: When this error occurs and why.
    """
```

**Section order**: Summary line first, then extended description (if needed), then Notes, Args, Returns, and Raises.
Do not include Examples sections or in-code examples in docstrings.

### Rules

- **Punctuation**: Always use proper punctuation in all documentation.
- **Imperative mood**: Use verbs like "Computes...", "Defines...", "Configures..." for ALL members.
- **Boolean descriptions**: Use "Determines whether..." for boolean parameters. For properties that return a boolean,
  use "Returns..." (the property docstring convention supersedes the boolean convention).
- **Parameters**: Start descriptions with uppercase. Don't repeat type info.
- **Returns**: Describe what is returned, not the type.
- **Prose over lists**: Always use prose instead of bullet lists or dashes in docstrings.

### Class docstrings with attributes

For classes, include an Attributes section listing all instance attributes:

```python
class DataProcessor:
    """Processes experimental data for analysis.

    Args:
        data_path: Path to the input data file.
        sampling_rate: The sampling rate in Hz.
        enable_filtering: Determines whether to apply bandpass filtering.

    Attributes:
        _data_path: Cached path to input data.
        _sampling_rate: Cached sampling rate parameter.
        _enable_filtering: Cached filtering flag.
        _processed_data: Dictionary storing processed results.
    """
```

### Enum and dataclass attributes

For enums and dataclasses, document each attribute inline using triple-quoted strings:

```python
class VisualizerMode(IntEnum):
    """Defines the display modes for the BehaviorVisualizer."""

    LICK_TRAINING = 0
    """Displays only lick sensor and valve plots."""
    RUN_TRAINING = 1
    """Displays lick, valve, and running speed plots."""
    EXPERIMENT = 2
    """Displays all plots including the trial performance panel."""


@dataclass
class SessionConfig:
    """Defines the configuration parameters for an experiment session."""

    animal_id: str
    """The unique identifier for the animal."""
    session_duration: float
    """The duration of the session in seconds."""
```

### Property docstrings

Property docstrings should ideally be a single sentence, even if it spans multiple lines. Do not
split a property summary into a one-line summary plus an extended description paragraph — keep it
as one continuous sentence that wraps naturally at the line-length limit.

```python
@property
def field_shape(self) -> tuple[int, int]:
    """Returns the shape of the data field as (height, width)."""
    return self._field_shape

@property
def rigid_y_offsets(self) -> NDArray[np.int32]:
    """Returns the vertical translation offsets from rigid registration, one value per frame or a zero array
    when the underlying data is absent.
    """
    ...
```

### Module docstrings

Follow the same imperative mood pattern as other docstrings:

```python
"""Provides assets for processing and analyzing neural imaging data."""
```

### CLI command docstrings

CLI commands use a specialized format because Click parses these into help messages. Do not use standard docstring
sections (Notes, Args, Returns, Raises) as they will appear verbatim in the CLI help output.

```python
@click.command()
def process_data(input_path: Path, output_path: Path) -> None:
    """Processes raw experimental data and saves the results.

    This command reads data from the input path, applies standard preprocessing
    steps, and writes the processed output to the specified location.
    """
```

### MCP server tool docstrings

MCP tools serve dual purposes: documenting for developers and providing instructions to AI agents.

```python
@mcp.tool()
def start_video_session(
    output_directory: str,
    frame_rate: int = 30,
) -> str:
    """Starts a video capture session with the specified parameters.

    Creates a VideoSystem instance and begins acquiring frames from the camera.

    Important:
        The AI agent calling this tool MUST ask the user to provide the output_directory path
        before calling this tool. Do not assume or guess the output directory.

    Args:
        output_directory: The path to the directory where video files will be saved.
        frame_rate: The target frame rate in frames per second. Defaults to 30.
    """
```

### MCP server response formatting

MCP tool responses should be concise and information-dense.

```python
# Good - concise, information-dense
return f"Session started: {interface} #{camera_index} {width}x{height}@{frame_rate}fps -> {output_directory}"

# Avoid - verbose multi-line formatting
return (
    f"Video Session Started\n"
    f"- Interface: {interface}\n"
    f"- Camera: {camera_index}\n"
)
```

**Formatting conventions**:

- **Concise output**: Keep responses to a single line when possible
- **Key-value pairs**: Use `Key: value` format with `|` separators for multiple items
- **Errors**: Start with "Error:" followed by a brief description

---

## Type annotations

### General rules

- All function parameters and return types must have annotations
- Use `-> None` for functions that don't return a value
- Use `| None` for optional types (not `Optional[T]`)
- Use lowercase `tuple`, `list`, `dict` (not `Tuple`, `List`, `Dict`)
- Avoid `any` type; use explicit union types instead

### NumPy arrays

```python
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from numpy.typing import NDArray

def process(data: NDArray[np.float32]) -> NDArray[np.float32]:
    ...
```

- Always specify dtype explicitly: `NDArray[np.float32]`, `NDArray[np.uint16]`, `NDArray[np.bool_]`
- Never use unparameterized `NDArray`
- Use `TYPE_CHECKING` block for `NDArray` to avoid runtime import overhead

### Class attributes

```python
def __init__(self, height: int, width: int) -> None:
    self._field_shape: tuple[int, int] = (height, width)
```

### Type aliases

Use PEP 695 `type` statement syntax (Python 3.12+) for type aliases:

```python
# Good - PEP 695 type statement
type CRCType = np.uint8 | np.uint16 | np.uint32

type PrototypeType = (
    np.bool_
    | np.uint8
    | np.int8
    | np.uint16
    | np.int16
    | np.float32
)

# Avoid - old-style TypeAlias
from typing import TypeAlias
CRCType: TypeAlias = np.uint8 | np.uint16 | np.uint32
```

Use `Literal` types for constrained string or value parameters:

```python
from typing import Literal

def load_data(file_path: Path, *, memory_map: bool = False) -> NDArray[np.float32]:
    """Loads data from file with optional memory mapping."""
    mmap_mode: Literal["r"] | None = "r" if memory_map else None
    return np.load(file_path, mmap_mode=mmap_mode)
```
