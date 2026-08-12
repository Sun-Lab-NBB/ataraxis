---
name: log-processing-results
description: >-
  Documents log processing output data formats, feather file discovery, output verification, event
  distribution analysis, and interpretation guidance. Use when evaluating log processing results, when
  the user asks about extracted event data, timing statistics, or microcontroller data quality.
user-invocable: false
---

# Log processing results

Documents the output data formats of the microcontroller data extraction pipeline, covering feather file discovery,
schema reference, output verification, event analysis, and interpretation guidance.

---

## Scope

**Covers:**
- Output data format: feather file schema, column names, and dtypes
- Directory structure and naming conventions
- Recording discovery via MCP tool
- Output verification via MCP tool (schema correctness, completeness)
- Event distribution and inter-event timing analysis via MCP tool
- Event and command code interpretation
- Data payload reconstruction guidance
- Locating and reading output files from analysis Python via the library's exported primitives
- Processing status cross-referencing

**Does not cover:**
- Input archive format, source ID semantics, or DataLogger output (see `/log-input-format`)
- Processing workflow, batch operations, or status monitoring (see `/log-processing`)
- Extraction configuration management (see `/extraction-configuration`)
- Microcontroller hardware setup or discovery (see `/microcontroller-setup`)
- Firmware code tables and the emission rules behind them (see `/microcontroller:firmware-module`)
- MCP server connectivity issues (see `/communication-mcp-environment-setup`)

---

## Available tools

### Discovery tool

| Tool                                 | Purpose                                                               |
|--------------------------------------|-----------------------------------------------------------------------|
| `discover_microcontroller_data_tool` | Discovers microcontroller manifests and confirmed log-archive sources |

**Parameters:**

| Parameter        | Type               | Default    | Description                                             |
|------------------|--------------------|------------|---------------------------------------------------------|
| `root_directory` | `str`              | (required) | Absolute path to root directory to search for manifests |
| `source_ids`     | `list[str] / None` | `None`     | Restricts the listing to these controller IDs           |
| `name`           | `str / None`       | `None`     | Restricts the listing to one controller name            |
| `include_items`  | `bool`             | `False`    | Lists sources when no filter is named                   |
| `detailed`       | `bool`             | `False`    | Adds `log_archive` and `modules` to each listed source  |

It also accepts `limit` and `start_row`, which page a long listing. `/log-processing` owns the full staging contract.

### Verification tool

| Tool                            | Purpose                                                           |
|---------------------------------|-------------------------------------------------------------------|
| `verify_processing_output_tool` | Validates feather file schema correctness and output completeness |

**Parameters:**

| Parameter          | Type  | Default    | Description                                     |
|--------------------|-------|------------|-------------------------------------------------|
| `output_directory` | `str` | (required) | Absolute path to the output directory to verify |

**Return structure:**
```text
verified:           Boolean indicating at least one feather file was found AND all found files passed
                    schema validation (an output directory holding zero feather files reports false)
files[]:            Per-file verification results:
  file:             Absolute path to the feather file
  filename:         Base filename of the feather file
  valid:            Boolean schema correctness
  row_count:        Number of rows in the file (absent when the file could not be read)
  columns:          List of column names found (absent when the file could not be read)
  type:             "module", "kernel", or "unknown" (inferred from filename)
  error:            Read failure message, present instead of row_count/columns
  missing_columns:  Expected columns absent from the file (only when valid is false)
  extra_columns:    Unexpected columns present in the file (only when valid is false)
total_files:        Number of feather files found
output_directory:   Absolute path to the output (parent) directory that was passed in (contains the microcontroller_data/ subdirectory)
data_path:          Absolute path to the microcontroller_data/ directory
tracker:            ProcessingTracker status summary. {} when the directory holds no tracker file, and
                    {error: "Unable to read tracker file."} when a tracker exists but cannot be parsed
  jobs[]:           One entry per job the tracker registers:
    job_id:         Hexadecimal job identifier
    source_id:      The job's tracker specifier, falling back to job_id[:8] when the specifier is empty
    status:         "SCHEDULED", "RUNNING", "SUCCEEDED", or "FAILED"
    error_message:  Present only when the tracker recorded one for that job
  summary:          Job counts: total, succeeded, failed, running, scheduled
```

A `source_id` that reads as an eight-character hex string is that fallback, not a controller identifier: it means the
tracker entry carries no specifier, so pair it with `job_id` rather than matching it against a manifest source.

**Error returns.** Three checks return a single top-level `{error: ...}` and inspect nothing further, so no `files`
list, `tracker` block, or `verified` flag is produced:

| Condition                                 | Returned error                                           |
|-------------------------------------------|----------------------------------------------------------|
| The path does not exist                   | `Directory does not exist: {path}`                       |
| The path resolves to a file               | `Path is not a directory: {path}`                        |
| The path holds no `microcontroller_data/` | `No 'microcontroller_data' subdirectory found under ...` |

The third message ends with "Processing may not have been run yet", which is only one of its three causes. Route it:

1. Processing genuinely has not run for this output directory: confirm the batch state through `/log-processing`.
2. The `microcontroller_data/` directory itself was passed instead of its parent. The tool always appends the
   subdirectory name, so the nested lookup fails. Pass the output directory the batch was prepared with.
3. `clean_log_processing_output_tool` (see `/log-processing`) deleted the subdirectory, taking the feather files and the
   tracker with it. Re-prepare and re-execute rather than treating this as data loss.

### Event analysis tool

| Tool                          | Purpose                                                        |
|-------------------------------|----------------------------------------------------------------|
| `query_extracted_events_tool` | Reads feather files and computes event distribution and timing |

**Parameters:**

| Parameter         | Type        | Default    | Description                                              |
|-------------------|-------------|------------|----------------------------------------------------------|
| `feather_files`   | `list[str]` | (required) | Absolute paths to feather files from verification output |
| `max_sample_rows` | `int`       | `10`       | Sample rows to include per file, must not be negative    |

**Return structure:**
```text
results[]:                Per-file analysis results:
  file:                   Absolute path to the feather file
  summary:
    total_rows:           Total number of extracted messages
    first_timestamp_us:   First message timestamp (microseconds since UTC epoch)
    last_timestamp_us:    Last message timestamp
    duration_us:          Total duration in microseconds
    duration_seconds:     Total duration in seconds
  event_distribution[]:   Per-event-code frequency list:
    event_code:           Event code (integer)
    count:                Number of occurrences
  command_distribution[]: Per-command-code frequency list:
    command_code:         Command code (integer)
    count:                Number of occurrences
  inter_event_timing:
    mean_us:              Mean inter-event interval (microseconds)
    mean_ms:              Mean inter-event interval (milliseconds)
    median_us:            Median inter-event interval (microseconds)
    median_ms:            Median inter-event interval (milliseconds)
    std_us:               Standard deviation (microseconds only)
    min_us:               Minimum interval (microseconds only)
    max_us:               Maximum interval (microseconds only)
  sample_rows[]:          First N rows (binary data replaced with has_data boolean)
  error:                  Read failure message, replacing the statistics keys for that file
total_files:              Number of files analyzed
```

A file that cannot be read yields `{file, error}` instead of the statistics keys. A negative `max_sample_rows` returns a
single top-level `{error: ...}` dictionary and analyzes nothing.

`inter_event_timing` is `{}` when `total_rows < 2`. For an empty file, `summary` is just `{total_rows: 0}` (no
timestamps/duration), with empty `event_distribution`, `command_distribution`, and `sample_rows`.

---

## Recommended query order

1. **`discover_microcontroller_data_tool`**: Find all manifests and confirmed sources under the root.
2. **`verify_processing_output_tool`**: Verify feather files exist with correct schema in the output directory. Use the
   `files` list for downstream analysis.
3. **`query_extracted_events_tool`**: Compute event statistics for verified feather files.

---

## Output data reference

### Directory structure

The processing pipeline writes all output under a `microcontroller_data/` subdirectory within the output directory
provided by the user:

```text
{output_directory}/
└── microcontroller_data/
    ├── microcontroller_processing_tracker.yaml
    ├── controller_{source_id}_module_{type}_{id}.feather
    ├── controller_{source_id}_module_{type}_{id}.feather
    ├── controller_{source_id}_kernel.feather
    └── ...
```

### Feather file schema

Each feather file is a Polars DataFrame serialized as Feather IPC format with five columns:

| Column         | Dtype    | Description                                                        |
|----------------|----------|--------------------------------------------------------------------|
| `timestamp_us` | `UInt64` | Message timestamp in microseconds since UTC epoch                  |
| `command`      | `UInt8`  | Command code the module/kernel was executing when message was sent |
| `event`        | `UInt8`  | Event code identifying the message type                            |
| `dtype`        | `String` | Numpy dtype string for the data payload (or null if no data)       |
| `data`         | `Binary` | Serialized binary data payload (or null if no data)                |

Rows are ordered chronologically. Each row corresponds to one extracted message matching the extraction config's event
codes.

**Note:** Each feather file is published through a temporary file and a rename, so a reader never observes a partially
written one. A job killed mid-write leaves the previously written file intact rather than a truncated file. Treat a
feather file that fails to decode as a foreign file or a damaged filesystem, not as partial output, and do not add a
"re-run because the write may have been cut short" step to a recovery path.

### Naming conventions

Filling in the two patterns from the directory tree above gives `controller_101_module_1_1.feather` for controller 101,
module type 1, instance 1, and `controller_101_kernel.feather` for that controller's kernel messages.

Multiple feather files may be produced per source ID (one per module extraction target plus an optional kernel file).

A configured module or kernel produces a feather file **only if at least one message matched its event codes**. A fully
empty archive produces no files at all. All of these cases still report `SUCCEEDED`. A missing expected file therefore
means the configured event codes never fired, not data loss or a processing failure. Cross-check it against the event
distribution (`query_extracted_events_tool`) or the input archive before treating it as a problem.

### Locating output files from analysis Python

Do not hand-write the patterns above in analysis code. The library exports the resolvers it writes the files with, so a
path built through them cannot drift from a rename in a future release:

| Name                  | Returns                                                                          |
|-----------------------|----------------------------------------------------------------------------------|
| `OutputLayout`        | `StrEnum` of the directory, tracker, prefix, infix, and suffix names             |
| `resolve_module_path` | One module's file path, from the data directory, source id, type, and id         |
| `resolve_kernel_path` | One source's kernel file path, from the data directory and source id             |
| `find_module_paths`   | Every module file in a data directory, sorted by path                            |
| `find_kernel_paths`   | Every kernel file in a data directory, sorted by path                            |
| `parse_module_path`   | `(source_id, module_type, module_id)` read back out of a filename, as three ints |
| `parse_kernel_path`   | The source id read back out of a kernel filename, as one int                     |

The six functions are top-level exports of `ataraxis_communication_interface`. `OutputLayout` is exported from
`ataraxis_communication_interface.orchestration` only, so importing it from the top level raises `ImportError`. Every
resolver takes the `microcontroller_data/` directory itself, which is the `data_path` value
`verify_processing_output_tool` returns, not the parent output directory.

**Note:** A hand-rolled `controller_*.feather` glob is wrong, because it matches the module and the kernel files
together. The two finders separate them by construction, `find_module_paths` globbing `controller_*_module_*.feather`
and `find_kernel_paths` globbing `controller_*_kernel.feather`, and both return an empty list rather than raising when
the directory does not exist. Pair each finder with its parser to recover a file's identity from the filename instead
of re-reading the manifest. Both parsers raise `ValueError` naming the offending filename when a name does not follow
the convention, where a hand-written `split("_")` would silently mis-index.

### ProcessingTracker file

The `microcontroller_processing_tracker.yaml` file tracks job lifecycle per output directory. Each job has:
- **job_id:** Unique hexadecimal identifier
- **job_name:** Always `microcontroller_data_extraction`
- **specifier:** The source ID string
- **status:** `SCHEDULED`, `RUNNING`, `SUCCEEDED`, or `FAILED`
- **started_at / completed_at:** Microsecond UTC timestamps (when available)
- **error_message:** Error details (when status is `FAILED`)

---

## Interpretation guidance

### Command vs event semantics

- **Command codes** (`command` column) indicate what operation the module or kernel was executing when the message was
  sent. These correspond to firmware-defined command IDs.
- **Event codes** (`event` column) identify the specific type of message. These are the same codes specified in the
  extraction config.

**Decoding the columns.** The library mirrors the firmware code tables as importable `IntEnum` classes, so decode
against the enum rather than against a number written into analysis code. All three are top-level exports of
`ataraxis_communication_interface`:

| Enum                 | Decodes                                                                           |
|----------------------|-----------------------------------------------------------------------------------|
| `KernelCommandCodes` | The `command` column of a kernel feather                                          |
| `KernelStatusCodes`  | The `event` column of a kernel feather                                            |
| `ModuleStatusCodes`  | The `event` column of a module feather, for the service codes the base class owns |

A module's own `command` column and its custom `event` codes are defined by that module's firmware, not by any of these
enums, so decode them against the module's own interface. `MINIMUM_CUSTOM_STATUS_CODE` and `MAXIMUM_CUSTOM_STATUS_CODE`
(also top-level exports) bound the custom range and tell the two apart.

**A kernel `command` value is not always a PC-sent command.** A `SETUP_COMPLETE` row carries `command` 2
(`RESET_CONTROLLER`) and a reception-phase fault carries `command` 1 (`RECEIVE_DATA`), neither of which the PC
addressed.

**Counting event code 2 undercounts recurrent commands.** An event-2 count in a module feather counts command
retirements rather than repetitions, so count an event the command body itself emits instead.
`/microcontroller:firmware-module` owns the firmware code tables and the emission rules behind both readings.

### Inter-event timing quality

- **Stable timing** (low `std_us` relative to `mean_us`) indicates regular communication cadence
- **High jitter** (large `std_us` or wide `min_us` to `max_us` spread) may indicate serial bandwidth contention,
  buffering, or competing processes
- **Very long gaps** (`max_us` >> `mean_us`) suggest intermittent communication interruptions

### Message loss is not measurable post-hoc

Per-message loss **cannot** be computed from extracted archives. The serial wire format carries no sequence or counter
field, so there is no expected-vs-received count, and extraction keeps only the configured event codes, so a sparse
`event_distribution` reflects the config, not dropped data. Do **not** report inter-event gaps (a large `max_us`) as
"lost messages": a long gap is a timing observation, not a quantified loss.

The only after-the-fact signal of a communication interruption is the Kernel keepalive-timeout status event
(`kKeepAliveTimeout`, kernel status code 10), emitted when the Kernel stops receiving keepalive messages from the PC. It
appears in the kernel feather **only if that kernel event code was included in the extraction config**. If a run needs
guaranteed-delivery accounting, that guarantee comes from the transport layer's runtime CRC/COBS verification during
acquisition, not from post-hoc log analysis.

### Data payload reconstruction

Read the payloads through the library's exported primitives rather than looping `np.frombuffer` over rows:

```python
import numpy as np
import polars as pl
from ataraxis_communication_interface import (
    ExtractedDataColumns,  # StrEnum of the five column names; indexes a frame and names a series
    partition_events,      # One pass over a frame -> {event_code: sub-frame}
    get_event_timestamps,  # (partition, event_code) -> timestamps, for a state-only event
    get_event_data,        # (partition, event_code, values_dtype) -> (timestamps, decoded values)
)

frame = pl.read_ipc(source="controller_101_module_1_1.feather")
all_timestamps = frame[ExtractedDataColumns.TIMESTAMP]  # Index by the enum, never by a literal column name
partition = partition_events(module_dataframe=frame)

state_times = get_event_timestamps(partition=partition, event_code=52)
data_times, values = get_event_data(partition=partition, event_code=51, values_dtype=np.float32)
```

Both readers return empty arrays when the partition holds no such event code, so an absent code needs no guard.
`get_event_data` concatenates the whole event stream into one buffer read instead of one read per message, and it
squeezes a scalar prototype's trailing axis. A scalar event therefore yields a 1-D array with one value per timestamp,
while an array prototype yields one row per timestamp.

Prefer these over a hand-rolled loop because the hand path silently produces wrong values where `get_event_data` raises
a `ValueError` that names the event code. It raises on four conditions:

| Condition                                     | What the hand path does instead              |
|-----------------------------------------------|----------------------------------------------|
| State-only event code, so no message has data | Yields nothing, or crashes on a null payload |
| One null payload inside a decodable stream    | Silently drops that message from the series  |
| Mixed dtypes under one event code             | Decodes some values under the wrong dtype    |
| Value count not a whole multiple of messages  | Mis-pairs values with timestamps             |

A mixed-dtype event code marks a table this library did not write, or a firmware revision that reused the code. Use
`get_event_timestamps` for the state-only case the first row names.

**Note:** `build_message_dataframe` is exported from `ataraxis_communication_interface.microcontroller` but not from the
top level, so importing it alongside the four names above raises `ImportError`. It is a writer, not a reader. Nothing in
the analysis path needs it.

The `dtype` column contains the numpy dtype string that was used to serialize the data on the microcontroller side.
Common dtypes include `uint8`, `uint16`, `uint32`, `int32`, `float32`.

Rows from `*_STATE` events carry null `dtype`/`data` by design, a state code reported via the `event` column with no
payload, while a `*_DATA` event normally carries a numpy dtype string and serialized bytes. Null in a data row is not
corruption of the row, but it does mark a message whose prototype code this library does not recognize.

### Decoding fault payloads

The runtime interface turns a fault message into a readable description as it arrives, but nothing writes that
description into a feather. Post-hoc, a fault row carries only its two raw payload values, so decode them here. A
`uint8` payload holding exactly two values means different things per event code, and reading the wrong pair inverts the
diagnosis:

| Feather and `event`                                      | The two `uint8` values mean                 |
|----------------------------------------------------------|---------------------------------------------|
| Kernel `RECEPTION_ERROR` (3) or `TRANSMISSION_ERROR` (4) | Communication status, then transport status |
| Kernel `MODULE_SETUP_ERROR` (2)                          | Module type, then module id                 |
| Kernel `MODULE_PARAMETERS_ERROR` (7)                     | Module type, then module id                 |
| Kernel `TARGET_MODULE_NOT_FOUND` (9)                     | Module type, then module id                 |
| Module `TRANSMISSION_ERROR` (1)                          | Communication status, then transport status |

Two kernel events carry a single value instead, and the second one changes the `dtype`. `INVALID_MESSAGE_PROTOCOL` (5)
carries the rejected protocol code as one `uint8`. `KEEPALIVE_TIMEOUT` (10) carries the timeout window in milliseconds
as one `uint32`, the value the firmware derives, which is twice the keepalive interval it was configured with, so do not
read it back as the configured interval.

`CommunicationStatusCodes` and `TransportStatusCodes` are top-level exports of `ataraxis_communication_interface`. Their
value ranges do not overlap, which is what makes the byte order recoverable if a caller loses it. The transport codes
come from the microcontroller-side transport library and carry meanings distinct from the PC-side `TransportLayerStatus`
codes, so never decode one against the other.

Read a pair with `get_event_data(partition=partition, event_code=..., values_dtype=np.uint8)`, which returns one
two-column row per fault message.

### Module vs kernel files

- **Module files** contain data from specific hardware modules (sensors, actuators, encoders). Each module has its own
  file with events specific to that module's firmware.
- **Kernel files** contain system-level messages from the microcontroller's kernel (status reports, error codes,
  keepalive signals). Kernel messages are shared across all modules on a controller.

---

## Discovery output reference

A bare `discover_microcontroller_data_tool` call reports the counts and a `breakdown` naming the controller IDs and the
controller names the scan found, and it lists no sources. Processing status can be determined by checking whether
processed output exists in the output directories:

```text
{
    "log_directories": ["/path/to/session_data_log"],
    "total_sources": 1,
    "total_log_directories": 1,
    "breakdown": {"source_id": {"101": 1}, "name": {"teensy_main": 1}},
    "sources": [                                      | requested by include_items=True
        {
            "recording_root": "/path/to/session",
            "source_id": "101",
            "name": "teensy_main",
            "log_directory": "/path/to/session_data_log",
            "log_archive": "/path/to/101_log.npz",    | requested by detailed=True
            "modules": [                              | requested by detailed=True
                {"module_type": 1, "module_id": 1, "name": "encoder"},
                {"module_type": 2, "module_id": 1, "name": "lick_sensor"}
            ]
        }
    ],
    "rows": 1, "matched_rows": 1, "start_row": 0, "next_start_row": null
}
```

To determine detailed job status, use `get_batch_status_overview_tool` from `/log-processing`.

---

## Related skills

| Skill                                  | Relationship                                                        |
|----------------------------------------|---------------------------------------------------------------------|
| `/communication-mcp-environment-setup` | Prerequisite: MCP server connectivity for tool access               |
| `/microcontroller-setup`               | Upstream: MCP discovery tools that locate archives and recordings   |
| `/microcontroller-interface`           | Upstream: code that produces the data analyzed here                 |
| `/extraction-configuration`            | Context: extraction config determines which events appear in output |
| `/log-input-format`                    | Reference: input archive format and source ID semantics             |
| `/log-processing`                      | Upstream: processing workflow that produces this output             |
| `/microcontroller:firmware-module`     | Reference: firmware code tables behind the extracted codes          |
| `/cli-reference`                       | Reference: the `axci process` path that produces the same output    |
| `/pipeline`                            | Context: results analysis is phase 6 of the end-to-end pipeline     |

---

## Verification checklist

```text
Log Processing Output Completeness, tool-settled (run `verify_processing_output_tool`, `query_extracted_events_tool`):
- [ ] Output directory contains `microcontroller_data/` subdirectory
- [ ] Every feather file reports a correct schema
- [ ] Processing tracker shows SUCCEEDED for all jobs
- [ ] Event distribution and inter-event timing computed for every verified feather file

Log Processing Output Completeness, reader-judged:
- [ ] All expected module and kernel files present, cross-referenced with the extraction config
- [ ] Event distribution matches expected firmware event codes
- [ ] Inter-event timing is within acceptable range for the experiment
- [ ] Data payloads reconstructable (dtype + data columns present where expected)
```
