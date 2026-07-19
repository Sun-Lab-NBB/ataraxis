# Docstrings and type annotations

Detailed conventions for Python docstrings and type annotations in projects.

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

### Documentation quality

Beyond structural rules, every comment and docstring must meet quality criteria that govern
information density, readability, and accuracy.

**Necessary minimalism**: Documentation exists to convey information the reader cannot infer
from the code itself. Each docstring and comment must be as short as possible while still
conveying every necessary fact. Do not pad with restatements, motivational prose, or
implementation trivia. If a function's behavior is fully evident from a single summary line and
the type signature, no extended description is needed.

**Sentence length**: Sentences over 40 words are difficult for humans to parse and must be
broken into smaller sentences at natural clause boundaries. Long sentences in docstrings,
comments, and inline annotations are a strong signal of over-explanation.

**Typo-free and grammatical**: Every comment, docstring, and inline annotation must be free of
typos and grammatical errors.

**Length proportionality**: Docstring length must be proportional to function complexity. A
3-line helper with self-evident behavior does not need a multi-paragraph docstring listing
trivia. A 200-line orchestration method warrants more documentation than a property accessor.

**No type-signature restating**: Docstrings must not restate information already conveyed by
the type signature or the parameter names. Replace "Takes an integer count and returns a
boolean indicating success" with "Returns True when the operation succeeds." The signature
already conveys the types.

**No narrate-the-code comments**: Inline comments must explain non-obvious context, intent, or
constraints — not narrate what the code already says. Replace `# increment counter` above
`counter += 1` with either no comment, or a comment that explains why the increment matters at
that point.

**No stale references**: Comments must not reference closed issue numbers, removed code,
deprecated versions, or outdated TODOs. When the code referenced by a comment is removed, the
comment must be removed or rewritten.

**Implementation accuracy**: Docstring claims must accurately describe the function's
observable behavior, signature, parameter semantics, and return value. A docstring that says
"returns the absolute path" for a function that returns a relative path is a defect, even when
the code itself is correct.

**Separator punctuation**: Within docstring and comment prose, only the full stop and the comma
separate clauses. Do not use a semicolon, and do not use an em-dash as a separator whether it is
typed `--`, `—`, or `–`. A colon is allowed where it is lexically appropriate, such as a docstring
section header or introducing an explanation or list. A single hyphen in a compound word, a list
marker, or a numeric range is not an em-dash and is fine. This rule governs prose only. Code stays
exempt, so a `;` in a PEP 508 dependency marker, a `--flag` in a CLI reference, or a `--` argument
separator is left as written.

**Positive description**: State what the code does and what is currently true. Do not define
behavior by contrast with what it does not do ("does X, not Y", "works by X rather than Y"), and
do not frame it against former behavior ("previously", "used to", "no longer"). The one exception
is a contrast that is load-bearing because it corrects a counter-intuitive but likely assumption,
and it must carry its reason. For example, "Iterates over columns rather than rows, because the
columnar store keeps each column contiguous in memory." Without that reason, drop the contrast and
keep only the positive statement.

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

The module docstring description is a lean, cohesive chunk of at most 5 sentences. It states what
the module provides and, where relevant, why it lives where it does. Detailed material, such as
methodology, caveats, interpretation guidance, and rationale, belongs in the docstrings of the
functions, classes, enums, and constants the module defines. Keep the module docstring itself to
that lean description. A multi-paragraph module docstring or a module-level `Notes:` section is a
violation, so relocate each detail to the member it concerns.

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
MCP tool docstrings MUST include a `Returns` section describing the response structure so that
developers reviewing the code can understand the tool's output contract at a glance. As with all
Returns sections, describe the semantic content without restating the return type — the type
annotation already conveys that. For tools returning structured responses, name the keys in
prose and describe what each conveys.

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

    Returns:
        A summary of the session parameters including the interface, camera index, resolution, frame rate,
        and output directory.
    """
```

### Sphinx cross-reference specifiers

Sphinx specifiers (`:class:`, `:func:`, `:meth:`, `:data:`, `:attr:`) are **allowed only inside
MCP tool docstrings**, where AI agents consume them as structured cross-references. All other
documentation — module docstrings, class docstrings, non-MCP function and method docstrings,
constant and attribute docstrings, and inline comments — must use plain prose. Refer to classes,
functions, and methods by name in double backticks without a specifier prefix.

```python
# Good — MCP tool docstring (specifiers allowed)
@mcp.tool()
def prepare_batch_tool(session_paths: list[str]) -> dict[str, Any]:
    """Prepares a batch using :func:`discover_jobs` and initializes a :class:`ProcessingTracker`."""

# Good — non-MCP docstring (prose with backticks)
class ActiveJob:
    """Tracks a pending job currently executing as a ``Future`` on the shared process pool."""

# Bad — non-MCP docstring using specifiers
class ActiveJob:
    """Tracks a pending job currently executing as a :class:`Future` on the shared process pool."""
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
from __future__ import annotations

from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from numpy.typing import NDArray

def process(data: NDArray[np.float32]) -> NDArray[np.float32]:
    ...
```

- Always specify dtype explicitly: `NDArray[np.float32]`, `NDArray[np.uint16]`, `NDArray[np.bool_]`
- Never use unparameterized `NDArray`
- Use `TYPE_CHECKING` block for `NDArray` to avoid runtime import overhead
- Add `from __future__ import annotations` at the top of the file so that all annotations are
  evaluated lazily as strings. This is preferred over bare `TYPE_CHECKING` guards because it
  avoids `NameError` at runtime when an annotation references a name that is only available
  during type checking

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
