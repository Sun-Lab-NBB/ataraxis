# Libraries and tools

Conventions for ataraxis libraries, Numba, Click CLI, testing, and linting in Sun Lab projects.

---

## Ataraxis library preferences

Sun Lab projects use a suite of ataraxis libraries that provide standardized, high-performance utilities. **Prefer
these libraries** over standard library alternatives or reimplementation for their designated tasks when the project
depends on them. Projects that do not depend on `ataraxis-base-utilities` (such as `ataraxis-automation` itself) should
use standard Python patterns (`raise`, `print()`, `click.echo()`) instead.

### Console output (ataraxis-base-utilities)

In projects that depend on `ataraxis-base-utilities`, use `console.echo()` for console output instead of `print()`:

```python
from ataraxis_base_utilities import console

# Good - use console.echo() for all output
console.echo(message="Processing frame 1 of 100...")
console.echo(message="Analysis complete.", level="SUCCESS")
console.echo(message="Potential memory issue detected.", level="WARNING")

# Avoid - do not use print()
print("Processing frame 1 of 100...")  # Wrong - use console.echo()
```

**Log levels**: `DEBUG`, `INFO` (default), `SUCCESS`, `WARNING`, `ERROR`, `CRITICAL`

The global `console` instance is pre-configured and shared across Sun Lab projects that depend on
`ataraxis-base-utilities`. Whether to call `console.enable()` in the top-level `__init__.py` is a per-library choice.
If `console.echo()` is called in a library that does not have `console.enable()` anywhere, verify with the user
whether this is intentional.

**Exception**: Use `print()` or `click.echo()` when output requires specific formatting that would be disrupted by
console's line-width formatting, such as tables created with `tabulate` or manually aligned ASCII tables:

```python
from tabulate import tabulate

# Good - print() for pre-formatted tabulate output
table = tabulate(data, headers=["Port", "Device", "Status"], tablefmt="grid")
print("Device Information:")
print(table)

# Good - click.echo() for manually aligned CLI tables
click.echo("Precision | Duration | Mean Time")
click.echo("----------+----------+----------")
for row in results:
    click.echo(f"{row.precision:9} | {row.duration:8} | {row.mean:9.3f}")
```

When using this exception, add a comment explaining why standard console output is not used.

### List conversion (ataraxis-base-utilities)

Use `ensure_list()` to normalize inputs to list form:

```python
from ataraxis_base_utilities import ensure_list

# Good - handles scalars, numpy arrays, and iterables
items = ensure_list(input_item=user_input)

# Avoid - manual type checking
if isinstance(user_input, list):
    items = user_input
elif isinstance(user_input, np.ndarray):
    items = user_input.tolist()
else:
    items = [user_input]
```

### Iterable chunking (ataraxis-base-utilities)

Use `chunk_iterable()` for batching operations:

```python
from ataraxis_base_utilities import chunk_iterable

# Good - preserves numpy array types
for batch in chunk_iterable(iterable=large_array, chunk_size=100):
    process_batch(batch=batch)

# Avoid - manual slicing logic
for i in range(0, len(large_array), 100):
    batch = large_array[i:i + 100]
```

### Worker resolution (ataraxis-base-utilities)

Use `resolve_worker_count()` and `resolve_parallel_job_capacity()` for CPU core allocation:

```python
from ataraxis_base_utilities import resolve_worker_count, resolve_parallel_job_capacity

# Good - auto-detect available cores with system reserve
workers = resolve_worker_count()

# Good - cap at a caller-specified maximum
workers = resolve_worker_count(requested_workers=8)

# Good - determine parallel job slots
jobs = resolve_parallel_job_capacity(workers_per_job=4)

# Avoid - manual os.cpu_count() arithmetic
import os
workers = max(1, os.cpu_count() - 4)  # Wrong - fragile, no None guard
```

`resolve_worker_count()` auto-detects cores, subtracts a reserve (default 2), and clamps to at least 1. A positive
`requested_workers` caps the result against the available budget. `resolve_parallel_job_capacity()` divides available
cores by the per-job allocation.

### Byte serialization (ataraxis-base-utilities)

Use `convert_scalar_to_bytes()`, `convert_bytes_to_scalar()`, `convert_array_to_bytes()`, and
`convert_bytes_to_array()` for numpy byte serialization:

```python
import numpy as np
from ataraxis_base_utilities import (
    convert_scalar_to_bytes,
    convert_bytes_to_scalar,
    convert_array_to_bytes,
    convert_bytes_to_array,
)

# Good - serialize a scalar to bytes
data = convert_scalar_to_bytes(value=42, dtype=np.dtype("<i4"))

# Good - deserialize bytes back to a scalar
value = convert_bytes_to_scalar(data=data, dtype=np.dtype("<i4"))

# Good - serialize a 1D array to bytes
raw = convert_array_to_bytes(array=np.array([1, 2, 3], dtype=np.int32))

# Good - deserialize bytes back to a typed array
array = convert_bytes_to_array(data=raw, dtype=np.dtype(np.int32))

# Avoid - manual numpy byte manipulation
raw_bytes = np.array([42], dtype="<i4").view(np.uint8)  # Wrong - use convert_scalar_to_bytes()
value = np.frombuffer(raw_bytes, dtype="<i4")[0]         # Wrong - use convert_bytes_to_scalar()
```

All serialization functions accept `np.dtype` objects (not strings) for the `dtype` parameter. Scalar serialization
uses an internal LRU cache for repeated value+dtype pairs. The `convert_bytes_to_scalar()` return type includes
`int | float | bool` depending on the target dtype.

### Timing and delays (ataraxis-time)

Use `PrecisionTimer` for all timing operations:

```python
from ataraxis_time import PrecisionTimer, TimerPrecisions

# Good - high-precision interval timing
timer = PrecisionTimer(precision=TimerPrecisions.MICROSECOND)
timer.reset()
# ... operation ...
elapsed_us = timer.elapsed

# Good - non-blocking delay (releases GIL for other threads)
timer.delay(delay=5000, allow_sleep=True, block=False)  # 5ms delay

# Good - human-readable elapsed time for logging or CLI output
timer.reset()
# ... long operation ...
console.echo(message=f"Completed in {timer.format_elapsed()}.")  # e.g. "2 h 30 m"

# Good - lap tracking for multi-phase benchmarking
timer.reset()
# ... phase 1 ...
timer.lap()
# ... phase 2 ...
timer.lap()
all_laps = timer.laps  # tuple of lap durations in precision units

# Avoid - time.sleep() for precision timing
import time
time.sleep(0.005)  # Wrong for microsecond precision work

# Avoid - manual elapsed time formatting
elapsed = time.time() - start_time  # Wrong - use timer.format_elapsed()
print(f"Took {elapsed:.2f}s")
```

**Precision options**: `NANOSECOND`, `MICROSECOND` (default), `MILLISECOND`, `SECOND`

### Timeout guards (ataraxis-time)

Use `Timeout` for timeout-guarded loops instead of manual timer comparisons:

```python
from ataraxis_time import Timeout, TimerPrecisions

# Good - timeout guard for waiting on a condition
timeout = Timeout(duration=5_000_000, precision=TimerPrecisions.MICROSECOND)  # 5 second timeout
while not timeout.expired:
    if check_condition():
        break
    remaining = timeout.remaining  # time left before expiry

# Good - activity-based timeout (watchdog pattern)
timeout = Timeout(duration=1000, precision=TimerPrecisions.MILLISECOND)
while not timeout.expired:
    if received_heartbeat():
        timeout.kick()  # resets countdown without changing duration

# Good - reset with new duration
timeout.reset(duration=2000)  # restart with a different duration

# Avoid - manual timeout logic
start = time.time()  # Wrong
while time.time() - start < 5.0:
    if check_condition():
        break
```

### Fixed-interval polling (ataraxis-time)

Use `PrecisionTimer.poll()` for fixed-interval loops:

```python
from ataraxis_time import PrecisionTimer, TimerPrecisions

# Good - fixed-interval polling with precise timing
timer = PrecisionTimer(precision=TimerPrecisions.MILLISECOND)
for cycle in timer.poll(interval=100):  # 100 ms polling interval
    data = read_sensor()
    if data.is_complete or cycle >= 1000:
        break

# Avoid - manual polling loop with sleep
while True:  # Wrong
    data = read_sensor()
    if data.is_complete:
        break
    time.sleep(0.1)
```

### Timestamps (ataraxis-time)

Use `get_timestamp()` for generating timestamps:

```python
from ataraxis_time import get_timestamp, TimestampFormats, TimestampPrecisions

# Good - string format for filenames
timestamp = get_timestamp(output_format=TimestampFormats.STRING)
output_path = data_directory / f"session_{timestamp}.npy"

# Good - integer format for calculations (microseconds since epoch)
timestamp_us = get_timestamp(output_format=TimestampFormats.INTEGER)

# Good - bytes format for binary serialization
timestamp_bytes = get_timestamp(output_format=TimestampFormats.BYTES)

# Good - control timestamp precision (e.g. day-level for daily logs)
day_stamp = get_timestamp(
    output_format=TimestampFormats.STRING,
    precision=TimestampPrecisions.DAY,
)  # "2026-02-16"

# Avoid - datetime manipulation
from datetime import datetime
timestamp = datetime.now().strftime("%Y-%m-%d-%H-%M-%S")  # Wrong
```

**Precision options**: `YEAR`, `MONTH`, `DAY`, `HOUR`, `MINUTE`, `SECOND`, `MICROSECOND` (default)

### Timestamp conversion and parsing (ataraxis-time)

Use `convert_timestamp()` to convert between timestamp formats and `parse_timestamp()` to parse arbitrary datetime
strings:

```python
from ataraxis_time import convert_timestamp, parse_timestamp, TimestampFormats, TimestampPrecisions

# Good - decode a byte-serialized timestamp to a string
string_timestamp = convert_timestamp(
    timestamp=byte_timestamp,
    output_format=TimestampFormats.STRING,
)

# Good - parse an external datetime string into a standardized timestamp
timestamp_us = parse_timestamp(
    date_string="2026-02-16 14:30:00",
    format_string="%Y-%m-%d %H:%M:%S",
    output_format=TimestampFormats.INTEGER,
)

# Avoid - manual datetime parsing and formatting
dt = datetime.strptime(date_str, "%Y-%m-%d")  # Wrong
microseconds = int(dt.timestamp() * 1_000_000)
```

### Time unit conversion (ataraxis-time)

Use `convert_time()` for converting between time units:

```python
from ataraxis_time import convert_time, TimeUnits

# Good - explicit unit conversion
duration_seconds = convert_time(
    time=elapsed_microseconds,
    from_units=TimeUnits.MICROSECOND,
    to_units=TimeUnits.SECOND,
)

# Avoid - manual conversion with magic numbers
duration_seconds = elapsed_microseconds / 1_000_000  # Wrong
```

**Supported units**: `NANOSECOND`, `MICROSECOND`, `MILLISECOND`, `SECOND`, `MINUTE`, `HOUR`, `DAY`

### Rate and interval conversion (ataraxis-time)

Use `rate_to_interval()` and `interval_to_rate()` for frequency-to-interval conversions:

```python
from ataraxis_time import rate_to_interval, interval_to_rate, TimeUnits

# Good - convert sampling rate to polling interval
interval_us = rate_to_interval(rate=30.0, to_units=TimeUnits.MICROSECOND)  # 30 Hz → microseconds

# Good - convert interval back to frequency
frequency_hz = interval_to_rate(interval=33333.333, from_units=TimeUnits.MICROSECOND)  # → ~30 Hz

# Avoid - manual Hz conversion with magic numbers
interval_us = 1_000_000 / 30  # Wrong
```

### Timedelta interop (ataraxis-time)

Use `to_timedelta()` and `from_timedelta()` for `datetime.timedelta` conversions:

```python
from ataraxis_time import to_timedelta, from_timedelta, TimeUnits

# Good - convert timer units to timedelta for standard library interop
delta = to_timedelta(time=5_000_000, from_units=TimeUnits.MICROSECOND)  # 5 second timedelta

# Good - convert timedelta back to timer units
microseconds = from_timedelta(timedelta_value=delta, to_units=TimeUnits.MICROSECOND)

# Avoid - manual timedelta arithmetic
delta = datetime.timedelta(seconds=elapsed_us / 1_000_000)  # Wrong
```

### YAML configuration (ataraxis-data-structures)

Use `YamlConfig` as a base class for configuration dataclasses:

```python
from dataclasses import dataclass
from pathlib import Path
from ataraxis_data_structures import YamlConfig

# Good - subclass YamlConfig for YAML serialization
@dataclass
class ExperimentConfig(YamlConfig):
    """Defines experiment configuration parameters."""

    animal_id: str
    """The unique identifier for the animal."""
    session_duration: float
    """The duration in seconds."""

# Saving and loading
config = ExperimentConfig(animal_id="M001", session_duration=3600.0)
config.to_yaml(file_path=Path("config.yaml"))
loaded_config = ExperimentConfig.from_yaml(file_path=Path("config.yaml"))

# Avoid - manual YAML handling
import yaml
with open("config.yaml", "w") as file:
    yaml.dump(config.__dict__, file)  # Wrong
```

### Shared memory (ataraxis-data-structures)

Use `SharedMemoryArray` for inter-process data sharing:

```python
from ataraxis_data_structures import SharedMemoryArray
import numpy as np

# Good - create shared array in main process
prototype = np.zeros((100, 100), dtype=np.float32)
shared_array = SharedMemoryArray.create_array(name="frame_buffer", prototype=prototype)

# In child process - connect before use
shared_array.connect()
with shared_array.array() as arr:  # Thread-safe access
    arr[:] = new_data
shared_array.disconnect()

# Cleanup in main process
shared_array.destroy()

# Avoid - multiprocessing.Array or manual shared memory
from multiprocessing import Array
shared = Array('f', 10000)  # Wrong for complex array operations
```

### Data logging (ataraxis-data-structures)

Use `DataLogger` and `LogPackage` for high-throughput logging:

```python
from pathlib import Path
from ataraxis_data_structures import DataLogger, LogPackage
import numpy as np

# Good - dedicated logger process for parallel I/O
logger = DataLogger(
    output_directory=Path("/data/experiment"),
    instance_name="neural_data",
    thread_count=5,
)
logger.start()

# Package and submit data
package = LogPackage(
    source_id=np.uint8(1),
    acquisition_time=np.uint64(elapsed_us),
    serialized_data=data_array.tobytes(),
)
logger.input_queue.put(package)

# Cleanup
logger.stop()

# Avoid - direct file writes in acquisition loop
np.save(f"frame_{i}.npy", data)  # Wrong - blocks acquisition
```

### Quick reference table

| Task                    | Use This                         | Not This                                                        |
|-------------------------|----------------------------------|-----------------------------------------------------------------|
| Console output          | `console.echo()`                 | `print()` (when base-utilities unavailable or formatted tables) |
| Error handling          | `console.error()`                | `raise` (when base-utilities unavailable)                       |
| Convert to list         | `ensure_list()`                  | Manual type checking                                            |
| Batch iteration         | `chunk_iterable()`               | Manual slicing                                                  |
| Worker count            | `resolve_worker_count()`         | `os.cpu_count()` arithmetic                                     |
| Parallel job capacity   | `resolve_parallel_job_capacity()`| Manual core division                                            |
| Scalar to/from bytes    | `convert_scalar_to_bytes()` etc. | `np.array([v]).view(np.uint8)` / `np.frombuffer()`              |
| Array to/from bytes     | `convert_array_to_bytes()` etc.  | `np.frombuffer()` / `.tobytes()`                                |
| Precision timing        | `PrecisionTimer`                 | `time.time()`, `time.perf_counter()`                            |
| Delays                  | `PrecisionTimer.delay()`         | `time.sleep()`                                                  |
| Elapsed formatting      | `PrecisionTimer.format_elapsed()`| Manual `f"{elapsed:.2f}s"` formatting                           |
| Lap tracking            | `PrecisionTimer.lap()` / `.laps` | Manual elapsed snapshots in a list                              |
| Fixed-interval polling  | `PrecisionTimer.poll()`          | `while True: time.sleep(interval)`                              |
| Timeout guards          | `Timeout`                        | Manual `time.time() - start < limit` checks                     |
| Timestamps              | `get_timestamp()`                | `datetime.now().strftime()`                                     |
| Timestamp conversion    | `convert_timestamp()`            | Manual `datetime.fromtimestamp()` formatting                    |
| Timestamp parsing       | `parse_timestamp()`              | Manual `datetime.strptime()` + epoch math                       |
| Time unit conversion    | `convert_time()`                 | Manual division/multiplication                                  |
| Rate/interval conversion| `rate_to_interval()` etc.        | Manual `1_000_000 / hz` arithmetic                              |
| Timedelta interop       | `to_timedelta()` etc.            | Manual `timedelta(seconds=us / 1e6)`                            |
| YAML serialization      | `YamlConfig` subclass            | `yaml.dump()`/`yaml.load()`                                     |
| Inter-process arrays    | `SharedMemoryArray`              | `multiprocessing.Array`                                         |
| High-throughput logging | `DataLogger` + `LogPackage`      | Direct file writes                                              |

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

Sun Lab CLIs use [Click](https://click.palletsprojects.com/) with consistent patterns across all projects.

### Group and command setup

```python
CONTEXT_SETTINGS: dict[str, int] = {"max_content_width": 120}


@click.group("axvs", context_settings=CONTEXT_SETTINGS)
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

- Use `console.echo()` for standard CLI output when `ataraxis-base-utilities` is available
- Use `print()` or `click.echo()` for projects that do not depend on `ataraxis-base-utilities`, or
  for pre-formatted tables (tabulate, manually aligned ASCII)
- Use `console.error()` for error reporting when available; otherwise use standard `raise`

### Entry points

Define CLI entry points in `pyproject.toml`:

```toml
[project.scripts]
axvs = "ataraxis_video_system.cli:axvs_cli"
```

---

## Test files

Test files follow simplified documentation conventions.

### Module docstrings

Test module docstrings use the "Contains tests for..." format:

```python
"""Contains tests for classes and methods provided by the saver.py module."""
```

### Test function docstrings

Test function docstrings use imperative mood with "Verifies...":

```python
def test_video_saver_init_repr(tmp_path, has_ffmpeg):
    """Verifies the functioning of the VideoSaver __init__() and __repr__() methods."""
```

**Important**: Test function docstrings do not include Args, Returns, or Raises sections.

### Fixture docstrings

Pytest fixtures use imperative mood docstrings describing what the fixture provides:

```python
@pytest.fixture(scope="session")
def has_nvidia():
    """Checks for NVIDIA GPU availability in the test environment."""
    ...
```

---

## Linting and code quality

### Running checks

Run `tox -e lint` after making changes. This runs both **ruff** (style and formatting) and **mypy** (strict type
checking) in a single command. All issues must be resolved or suppressed with specific ignore comments.

If `tox` is unavailable, the underlying tools can be run directly:
- `ruff check .` and `ruff format --check .` for style violations
- `mypy` for type violations

Prefer `tox -e lint` when possible, as it ensures consistent tool versions and configuration.

### Resolution policy

Prefer resolving issues unless the resolution would:
- Make the code unnecessarily complex
- Hurt performance by adding redundant checks
- Harm codebase readability instead of helping it

### Magic numbers (PLR2004)

For magic number warnings, prefer defining constants:

```python
def calculate_threshold(self, value: float) -> float:
    """Calculates the adjusted threshold."""
    adjustment_factor = 1.5  # Empirically determined scaling factor.
    return value * adjustment_factor
```

### Using noqa

When suppressing a warning, always include the specific error code:

```python
if mode == 3:  # noqa: PLR2004 - LICK_TRAINING mode value from VisualizerMode enum.
    ...
```
