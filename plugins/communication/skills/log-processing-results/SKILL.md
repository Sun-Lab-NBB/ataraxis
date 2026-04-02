---
name: log-processing-results
description: >-
  Complete reference for log processing output data formats, feather file discovery, output verification,
  event distribution analysis, and interpretation guidance. Use when evaluating log processing results,
  when the user asks about extracted event data, timing statistics, or microcontroller data quality.
user-invocable: true
---

# Log processing results

Complete output data format documentation for the microcontroller data extraction pipeline. Covers feather
file discovery, schema reference, output verification, event analysis, and interpretation guidance.

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
- Processing status cross-referencing

**Does not cover:**
- Input archive format, source ID semantics, or DataLogger output (see `/log-input-format`)
- Processing workflow, batch operations, or status monitoring (see `/log-processing`)
- Extraction configuration management (see `/extraction-configuration`)
- Microcontroller hardware setup or discovery (see `/microcontroller-setup`)
- MCP server connectivity issues (see `/mcp-environment-setup`)

---

## Available tools

### Discovery tool

| Tool                                  | Purpose                                                               |
|---------------------------------------|-----------------------------------------------------------------------|
| `discover_microcontroller_data_tool`  | Discovers manifests, log archives, and processed feather outputs      |

**Parameters:**

| Parameter        | Type  | Default    | Description                                             |
|------------------|-------|------------|---------------------------------------------------------|
| `root_directory` | `str` | (required) | Absolute path to root directory to search for manifests |

### Verification tool

| Tool                            | Purpose                                                            |
|---------------------------------|--------------------------------------------------------------------|
| `verify_processing_output_tool` | Validates feather file schema correctness and output completeness  |

**Parameters:**

| Parameter          | Type  | Default    | Description                                                |
|--------------------|-------|------------|------------------------------------------------------------|
| `output_directory` | `str` | (required) | Absolute path to the output directory to verify            |

**Return structure:**
```text
verified:           Boolean indicating all files passed schema validation
files[]:            Per-file verification results:
  path:             Absolute path to the feather file
  valid:            Boolean schema correctness
  row_count:        Number of rows in the file
  columns:          List of column names found
  file_type:        "module" or "kernel" (inferred from filename)
total_files:        Number of feather files found
tracker:            ProcessingTracker status summary (if tracker exists)
```

### Event analysis tool

| Tool                          | Purpose                                                          |
|-------------------------------|------------------------------------------------------------------|
| `query_extracted_events_tool` | Reads feather files and computes event distribution and timing   |

**Parameters:**

| Parameter         | Type        | Default    | Description                                                  |
|-------------------|-------------|------------|--------------------------------------------------------------|
| `feather_files`   | `list[str]` | (required) | Absolute paths to feather files from verification output     |
| `max_sample_rows` | `int`       | `10`       | Number of sample rows to include per file                    |

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
  event_distribution:     Per-event-code frequency (event_code → count)
  command_distribution:   Per-command-code frequency (command_code → count)
  inter_event_timing:
    mean_us/ms:           Mean inter-event interval
    median_us/ms:         Median inter-event interval
    std_us/ms:            Standard deviation
    min_us/ms:            Minimum interval
    max_us/ms:            Maximum interval
  sample_rows[]:          First N rows (binary data omitted for readability)
total_files:              Number of files analyzed
```

---

## Recommended query order

1. **`discover_microcontroller_data_tool`** — Find all manifests and confirmed sources under the root.
2. **`verify_processing_output_tool`** — Verify feather files exist with correct schema in the output
   directory. Use the `files` list for downstream analysis.
3. **`query_extracted_events_tool`** — Compute event statistics for verified feather files.

---

## Output data reference

### Directory structure

The processing pipeline writes all output under a `microcontroller_data/` subdirectory within the output
directory provided by the user:

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

Rows are ordered chronologically. Each row corresponds to one extracted message matching the extraction
config's event codes.

### Naming conventions

**Module files:** `controller_{source_id}_module_{type}_{id}.feather`
- Example: `controller_101_module_1_1.feather` for controller 101, module type 1, instance 1

**Kernel files:** `controller_{source_id}_kernel.feather`
- Example: `controller_101_kernel.feather` for controller 101 kernel messages

Multiple feather files may be produced per source ID (one per module extraction target plus an optional
kernel file). This differs from the axvs pipeline which produces one file per source.

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

- **Command codes** (`command` column) indicate what operation the module or kernel was executing when
  the message was sent. These correspond to firmware-defined command IDs.
- **Event codes** (`event` column) identify the specific type of message. These are the same codes
  specified in the extraction config.

### Inter-event timing quality

- **Stable timing** (low `std_us` relative to `mean_us`) indicates regular communication cadence
- **High jitter** (large `std_us` or wide `min_us`–`max_us` spread) may indicate serial bandwidth
  contention, buffering, or competing processes
- **Very long gaps** (`max_us` >> `mean_us`) suggest intermittent communication interruptions

### Data payload reconstruction

For rows where `dtype` and `data` are non-null, the original data value can be reconstructed:

```python
import numpy as np

# From a feather row:
dtype_str = row["dtype"]  # e.g., "float32"
payload = row["data"]     # binary bytes

value = np.frombuffer(payload, dtype=np.dtype(dtype_str))
```

The `dtype` column contains the numpy dtype string that was used to serialize the data on the
microcontroller side. Common dtypes include `uint8`, `uint16`, `uint32`, `int32`, `float32`.

### Module vs kernel files

- **Module files** contain data from specific hardware modules (sensors, actuators, encoders). Each
  module has its own file with events specific to that module's firmware.
- **Kernel files** contain system-level messages from the microcontroller's kernel (status reports,
  error codes, keepalive signals). Kernel messages are shared across all modules on a controller.

---

## Discovery output reference

The `discover_microcontroller_data_tool` returns a flat list of confirmed source entries. Processing
status can be determined by checking whether processed output exists in the output directories:

```text
{
    "sources": [
        {
            "recording_root": "/path/to/session",
            "source_id": "101",
            "name": "teensy_main",
            "log_archive": "/path/to/101_log.npz",
            "log_directory": "/path/to/session_data_log",
            "modules": [
                {"module_type": 1, "module_id": 1, "name": "encoder"},
                {"module_type": 2, "module_id": 1, "name": "lick_sensor"}
            ]
        }
    ],
    "log_directories": ["/path/to/session_data_log"],
    "total_sources": 1,
    "total_log_directories": 1
}
```

To determine detailed job status, use `get_batch_status_overview_tool` from `/log-processing`.

---

## Related skills

| Skill                        | Relationship                                                        |
|------------------------------|---------------------------------------------------------------------|
| `/mcp-environment-setup`     | Prerequisite: MCP server connectivity for tool access               |
| `/microcontroller-setup`     | Upstream: MCP discovery tools that locate archives and recordings   |
| `/microcontroller-interface` | Upstream: code that produces the data analyzed here                 |
| `/extraction-configuration`  | Context: extraction config determines which events appear in output |
| `/log-input-format`          | Reference: input archive format and source ID semantics             |
| `/log-processing`            | Upstream: processing workflow that produces this output             |
| `/pipeline`                  | Context: results analysis is phase 6 of the end-to-end pipeline     |

---

## Verification checklist

```text
Log Processing Output Completeness:
- [ ] Output directory contains `microcontroller_data/` subdirectory
- [ ] Feather files verified via `verify_processing_output_tool` (schema correct)
- [ ] All expected module and kernel files present (cross-reference with extraction config)
- [ ] Processing tracker shows SUCCEEDED for all jobs
- [ ] Event analysis performed via `query_extracted_events_tool`
- [ ] Event distribution matches expected firmware event codes
- [ ] Inter-event timing is within acceptable range for the experiment
- [ ] Data payloads reconstructable (dtype + data columns present where expected)
```
