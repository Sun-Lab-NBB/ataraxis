# Libraries and tools

Conventions for ataraxis libraries, Numba, Click CLI, testing, and linting in projects.

---

## Ataraxis library preferences

Projects use a suite of ataraxis libraries that provide standardized, high-performance utilities. **Prefer these
libraries** over standard library alternatives or reimplementation for their designated tasks when the project depends
on them. Projects that do not depend on `ataraxis-base-utilities` (such as `ataraxis-automation` itself) should use
standard Python patterns (`raise`, `click.echo()`) instead.

**You MUST invoke `/explore-dependencies` to obtain a current API snapshot of each ataraxis dependency before writing
code that uses ataraxis library features.** The table below is a brief domain summary. The canonical domain-to-library
mapping lives in `/explore-dependencies`, which is authoritative for the current API of each dependency.

### Domain-to-library summary

| Domain                      | Preferred library          | Key APIs                                                         |
|-----------------------------|----------------------------|------------------------------------------------------------------|
| Console output, errors      | `ataraxis-base-utilities`  | `Console`, `console.echo()`, `console.error()`                   |
| Progress tracking           | `ataraxis-base-utilities`  | `Console.track()`, `Console.progress()`, `ProgressBar`           |
| List/array/filesystem utils | `ataraxis-base-utilities`  | `ensure_list()`, `chunk_iterable()`, `ensure_directory_exists()` |
| Worker/job resolution       | `ataraxis-base-utilities`  | `resolve_worker_count()`                                         |
| Byte serialization          | `ataraxis-base-utilities`  | `convert_scalar_to_bytes()` etc.                                 |
| Precision timing and delays | `ataraxis-time`            | `PrecisionTimer`, `Timeout`                                      |
| Timestamps and conversion   | `ataraxis-time`            | `get_timestamp()`, `convert_time()`                              |
| YAML configuration          | `ataraxis-data-structures` | `YamlConfig`                                                     |
| Shared memory               | `ataraxis-data-structures` | `SharedMemoryArray`                                              |
| High-throughput logging     | `ataraxis-data-structures` | `DataLogger`, `LogPackage`                                       |
| Atomic and direct writing   | `ataraxis-data-structures` | `atomic_write()`, `direct_write()`                               |
| Directory transfer/removal  | `ataraxis-data-structures` | `transfer_directory()`, `delete_directory()`                     |
| Checksums                   | `ataraxis-data-structures` | `calculate_directory_checksum()`                                 |
| Data interpolation          | `ataraxis-data-structures` | `interpolate_data()`                                             |
| Worker thread limiting      | `ataraxis-data-structures` | `limit_worker_threads()`, `initialize_worker_threads()`          |
| Job state tracking          | `ataraxis-data-structures` | `ProcessingTracker`                                              |
| Log archive reading         | `ataraxis-data-structures` | `LogArchiveReader`, `assemble_log_archives()`                    |

### Console and error handling notes

In projects that depend on `ataraxis-base-utilities`, use `console.echo()` for all console output and `console.error()`
for error reporting. Use `console.echo(message=..., raw=True)` for pre-formatted content (tables, aligned output) that
should bypass line-width formatting.

`console.enable()` and `console.disable()` belong at entry points that own the runtime, such as a CLI command, an MCP
server, or the top-level `__init__.py` of a library that drives its own pipeline. Code reached as a worker under another
library's entry point leaves the console state to that caller. Neither placement is a style finding on its own. If
`console.echo()` is called in a library that enables the console nowhere, verify with the user whether this is
intentional.

---

## Numba functions

### Decorator patterns

```python
# Standard cached function
@numba.njit(cache=True)
def _compute_values(...) -> None:
    ...

# Parallelized function
@numba.njit(cache=True, parallel=True)
def _process_batch(...) -> None:
    for i in prange(data.shape[0]):  # Parallel outer loop
        for j in range(data.shape[1]):  # Sequential inner loop
            ...

# Inlined helper (for small, frequently-called functions)
@numba.njit(cache=True, inline="always")
def compute_coefficients(...) -> None:
    ...
```

### Guidelines

- Always use `cache=True` for disk caching (avoids recompilation)
- Use `parallel=True` with `prange` only when no race conditions exist
- Use `inline="always"` for small helper functions called in hot loops
- Don't use `nogil` unless explicitly using threading
- Use Python type hints (not Numba signature strings) for readability

---

## Click CLI conventions

CLIs use [Click](https://click.palletsprojects.com/) with consistent patterns across all projects.

### Group and command setup

```python
_CONTEXT_SETTINGS: dict[str, int] = {"max_content_width": 120}


@click.group("axvs", context_settings=_CONTEXT_SETTINGS)
def axvs_cli() -> None:
    """Manages video capture sessions and camera configurations."""


@axvs_cli.command("discover")
@click.option(
    "-i",
    "--interface",
    required=False,
    default="all",
    type=click.Choice(["all", "opencv", "harvester"], case_sensitive=False),
    help="The camera interface to use for discovery.",
)
def discover_cameras(interface: str) -> None:
    """Discovers all compatible cameras connected to the system."""
    ...
```

### Option naming

- **Short flags**: Single or double lowercase letters (`-i`, `-sp`, `-id`)
- **Long flags**: Lowercase with hyphens (`--input-path`, `--camera-index`, `--output-directory`)
- **Path options**: Use `click.Path()` with explicit validation:

```python
@click.option(
    "-o",
    "--output-directory",
    required=True,
    type=click.Path(exists=False, file_okay=False, dir_okay=True, writable=True, path_type=Path),
    help="The directory to save output files.",
)
```

### Output formatting

- Use `console.echo()` for standard CLI output when `ataraxis-base-utilities` is available, with
  `console.echo(message=..., raw=True)` for pre-formatted tables or aligned output
- Use `click.echo()` for projects that do not depend on `ataraxis-base-utilities`
- Use `console.error()` for error reporting when available, and standard `raise` otherwise

### Entry points

Define CLI entry points in `pyproject.toml`. See `/pyproject-style` for the section that declares them and for its
format.

---

## Test files

Test files follow simplified documentation conventions.

### Module docstrings

Test module docstrings use the "Contains tests for..." format:

```python
"""Contains tests for classes and methods provided by the saver.py module."""
```

### Test function docstrings

Test function docstrings use third-person imperative mood with "Verifies...":

```python
def test_video_saver_init_repr(tmp_path, has_ffmpeg):
    """Verifies the functioning of the VideoSaver __init__() and __repr__() methods."""
```

**Important**: Test function docstrings do not include Args, Returns, or Raises sections.

### Fixture docstrings

Pytest fixtures use third-person imperative mood docstrings describing what the fixture provides:

```python
@pytest.fixture(scope="session")
def has_nvidia():
    """Checks for NVIDIA GPU availability in the test environment."""
    ...
```

### Parallel execution groups

The suite runs under `pytest-xdist` with `-n logical --dist loadgroup`, so every test is free to land on any worker
process. A test that contends for a process-wide or on-disk resource MUST carry the `@pytest.mark.xdist_group` marker,
and all mutually contending tests MUST share one group name. `loadgroup` routes the tests sharing a group name to a
single worker, which serializes them against one another, and it serializes nothing else. A contending test left
unmarked therefore runs concurrently with its rival and fails intermittently.

Resources that contend include a named shared-memory buffer, a `DataLogger` output directory, a serial port, a spawned
process or worker pool, and any other fixed name or path the operating system allows one owner to hold at a time.

```python
@pytest.mark.xdist_group(name="group1")
def test_data_logger_initialization(tmp_path: Path) -> None:
    """Verifies the initialization of the DataLogger class with different parameters."""
    ...
```

Pass the group name through the `name` keyword, as with every other call. Group names span the whole run rather than the
declaring module, so tests in separate modules that contend for one resource carry the same name. `/tox-config` owns the
`-n logical --dist loadgroup` flags themselves.

---

## Linting and code quality

### Running checks

Run `tox -e lint` after making changes, and resolve or suppress every issue it reports. See `/tox-config` for what that
environment runs, and `/pyproject-style` for the files each checker covers and for the wider ignore list that test files
carry.

If `tox` is unavailable, the underlying tools can be run directly:
- `ruff check .` and `ruff format --check .` for style violations
- `mypy` for type violations

Prefer `tox -e lint` when possible, as it ensures consistent tool versions and configuration.

`ruff format` is applied to every changed file before the commit. During an audit use the read-only `ruff format --diff
.` form, because bare `tox -e lint` rewrites and auto-fixes the source under review.

### Resolution policy

Resolve every issue the linter reports. Suppress with a coded `# noqa` or `# type: ignore` only when the resolution adds
a branch, allocation, or indirection that the rule itself does not require, and state that reason in the suppression
comment.

### console.error and terminating statements

Ruff reasons about control flow syntactically and does not follow `NoReturn` through a call, so its treatment depends on
the enclosing return annotation. In a function annotated `-> None`, no `raise` or `return` follows the call. In a
function annotated with any other return type, ruff `RET503` requires a terminating statement on the path that ends in
`console.error`, and `RET503` is part of the shared corpus that `lint.select = ["ALL"]` enables. Keep an unreachable
`return` there, mark it with `# pragma: no cover` so it stays outside the measured corpus, and leave the reason in a
comment:

```python
# Tail of a guard clause inside a function annotated '-> np.uint64'.
console.error(message=message, error=ValueError)
# Satisfies ruff RET503. console.error() is NoReturn, so this line never executes.
return np.uint64(0)  # pragma: no cover
```

### Magic numbers (PLR2004)

For magic number warnings, prefer defining constants:

```python
_ADJUSTMENT_FACTOR: float = 1.5
"""Empirically determined scaling factor applied to the raw threshold."""


def calculate_threshold(self, value: float) -> float:
    """Calculates the adjusted threshold."""
    return value * _ADJUSTMENT_FACTOR
```

### Using noqa

When suppressing a warning, always include the specific error code:

```python
if mode == 0:  # noqa: PLR2004 - LICK_TRAINING mode value from VisualizerMode enum.
    ...
```

### Coverage exclusions

The test suite MUST cover 100% of the measured statements, so every statement the suite cannot reach has to be excluded
deliberately. Two mechanisms do this, and the size of the target decides which one applies.

A module that the suite never exercises as a whole is excluded through the coverage `omit` list, which
`/pyproject-style` defines together with the interface modules it covers. A module that appears in that list carries no
`# pragma: no cover` comments at all.

Everything else stays in the measured corpus, and `# pragma: no cover` marks the individual statements the suite cannot
reach. Use it for defensive or unreachable guard branches, for hardware-dependent code paths, and for the branch of an
OS-dependent path that the current platform never takes. Annotate the narrowest construct that covers the excluded code,
which is the `if` or `except` line for a whole block and the statement itself for a single line. A pragma that spans a
function the suite could exercise hides real gaps, so reach for the whole-module `omit` list rather than annotating a
long run of members one by one.

### IDE inspection directives

IDE-specific inspection-suppression comments are NOT used in this codebase and MUST be removed whenever encountered. The
canonical example is the PyCharm/JetBrains `# noinspection ...` directive (e.g., `# noinspection PyShadowingBuiltins`
placed above a `copyright` assignment in a Sphinx `conf.py`).

Ruff and mypy are the authoritative checkers, and only their suppressions bear weight and MUST be preserved:

- Ruff: `# noqa: CODE` with the specific error code (see [Using noqa](#using-noqa)).
- Mypy: `# type: ignore[code]` with the specific error code.

When a real violation cannot be resolved, suppress it with the appropriate ruff or mypy comment rather than with an IDE
directive. Files already excluded from both checkers (for example the Sphinx `conf.py`, excluded via ruff
`extend-exclude`, and anything under the mypy `docs/` exclusion) need no suppression at all, so they MUST NOT carry `#
noinspection` comments either.
