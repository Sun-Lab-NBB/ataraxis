---
name: log-input-format
description: >-
  Documents the input data format required by the log processing pipeline: NPZ log archives produced by
  DataLogger, source ID semantics, microcontroller manifest system, archive internal message layout, and
  communication protocol. Use when the user asks about log archive format, source IDs, DataLogger output,
  or why processing fails due to missing or malformed archives.
user-invocable: true
---

# Log input format

Documents the input data format required by the microcontroller data extraction pipeline, including how
log archives are produced, their internal structure, manifest system, and source ID semantics.

---

## Scope

**Covers:**
- NPZ archive format and naming conventions
- How DataLogger and assemble_log_archives produce archives
- Microcontroller manifest system for archive identification
- Source ID semantics, assignment, and uniqueness constraints
- Multi-controller and multi-logger recording structures
- Archive internal message layout (protocol codes, message types)
- Directory hierarchy from recording root to log archives
- Prerequisites for running the processing pipeline

**Does not cover:**
- Processing workflow or batch operations (see `/log-processing`)
- Output data formats or event analysis (see `/log-processing-results`)
- Extraction configuration management (see `/extraction-configuration`)
- Microcontroller hardware setup or discovery (see `/microcontroller-setup`)
- MCP server connectivity (see `/mcp-environment-setup`)

---

## Archive format

### Naming convention

Log archives follow the naming pattern `{source_id}_log.npz`:

```text
101_log.npz    # Source ID 101 (from MicroControllerInterface controller_id=101)
102_log.npz    # Source ID 102 (from MicroControllerInterface controller_id=102)
51_log.npz     # Source ID 51 (from VideoSystem system_id=51, if sharing the logger)
```

The suffix `_log.npz` is defined as `LOG_ARCHIVE_SUFFIX` in `log_processing.py`. The source ID portion
is the integer form of the originating system's controller ID.

### How archives are produced

Archives are created in two phases:

1. **Runtime logging** — A `DataLogger` instance receives `LogPackage` messages via its multiprocessing
   input queue. Each `MicroControllerInterface` sends all communication messages to the shared logger
   using its `controller_id` as the `source_id`. The logger persists each message as an individual
   `.npy` file named `{source_id:03d}_{acquisition_time:020d}.npy` inside its output directory.

2. **Post-runtime assembly** — `assemble_log_archives()` from `ataraxis-data-structures` consolidates
   all `.npy` files in a log directory into one `.npz` archive per unique source ID. It groups files by
   source ID, sorts by timestamp, and writes uncompressed `.npz` archives. The original `.npy` files
   are removed by default.

**You MUST ensure archives have been assembled before running the processing pipeline.** The pipeline
expects `.npz` files, not raw `.npy` files. If no `.npz` archives are found during discovery, instruct
the user to run `assemble_log_archives_tool` (see `/microcontroller-setup`) on the log directories first.

### Microcontroller manifest

Each DataLogger output directory should contain a `microcontroller_manifest.yaml` file that identifies
which archives were produced by ataraxis-communication-interface controllers. The manifest structure:

```yaml
controllers:
- id: 101
  name: teensy_main
  modules:
  - module_type: 1
    module_id: 1
    name: encoder
  - module_type: 2
    module_id: 1
    name: lick_sensor
- id: 102
  name: teensy_aux
  modules:
  - module_type: 3
    module_id: 1
    name: valve
```

**How manifests are produced:**

- **Automatic:** `MicroControllerInterface.__init__()` writes a manifest entry to the DataLogger output
  directory using the `controller_id`, `name`, and module list. Each MicroControllerInterface sharing a
  DataLogger appends its entry to the same manifest file.
- **Manual:** Use `write_microcontroller_manifest_tool` (see `/microcontroller-setup`) to retroactively
  tag legacy log directories that predate the manifest system.

**Why manifests matter:** The `discover_microcontroller_data_tool` uses manifest-based routing to identify
controller-produced log archives. Directories without a `microcontroller_manifest.yaml` will not be
discovered. Manifests also associate controller IDs with human-readable names and enumerate the hardware
modules managed by each controller.

**Key difference from axvs manifests:** AXCI manifests include a `modules` list per controller, providing
full hardware module metadata (type, id, name). AXVS camera manifests only have source ID and camera name.

---

## Source ID semantics

### What source IDs represent

A source ID is a `np.uint8` value (1-255) that identifies the hardware system that produced log data.
In ataraxis-communication-interface, each `MicroControllerInterface` instance has a `controller_id` that
becomes the `source_id` in all log entries sent to the `DataLogger`.

```text
MicroControllerInterface(controller_id=np.uint8(101), data_logger=logger)
    → LogPackage(source_id=np.uint8(101), ...)
    → Raw .npy files: 101_00000000000000000000.npy, 101_00000000000001234567.npy, ...
    → Assembled archive: 101_log.npz
    → Processed output: controller_101_module_1_1.feather, controller_101_kernel.feather
```

### Uniqueness constraints

Source IDs have **two different uniqueness scopes**:

| Scope                       | Constraint                                                                                                                                                                 |
|-----------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Within a single DataLogger  | Source IDs MUST be unique. Multiple controllers sharing one DataLogger must have different controller_ids.                                                                 |
| Across DataLogger instances | Source IDs MAY repeat. Two loggers in the same recording can each have a source with the same ID without conflict, because each logger writes to its own output directory. |

This means source IDs are unique **per log directory**, not globally across a recording session.

### Common source ID assignments

Runtime MicroControllerInterface instances are advised to use IDs in the range **101-150**. This
convention is advised but not enforced. Any `np.uint8` value (1-255) is valid as long as source IDs
are unique across **all** sources within each DataLogger instance, including sources from other
libraries (e.g., ataraxis-video-system). The 101-150 range avoids collisions with other libraries'
advised ranges.

---

## Recording directory structure

### Single-logger recording

A recording session with one DataLogger produces:

```text
recording_root/
└── session_data_log/                         # DataLogger output (instance_name="session")
    ├── microcontroller_manifest.yaml         # Controller manifest (auto-written by MCI.__init__)
    ├── 101_00000000000000000000.npy          # Raw logs (before assembly)
    ├── 101_00000000000001234567.npy
    ├── 102_00000000000000000000.npy
    ├── 101_log.npz                           # Assembled archive (after assembly)
    └── 102_log.npz
```

The DataLogger creates its output directory as `{instance_name}_data_log/` inside the provided
`output_directory`. All MicroControllerInterface instances sharing this logger write to the same
directory, distinguished by their source IDs.

### Multi-logger recording

A recording session can use multiple DataLogger instances. Each creates its own output directory:

```text
recording_root/
├── behavior_data_log/                        # DataLogger instance_name="behavior"
│   ├── microcontroller_manifest.yaml         # Manifest for behavior controllers
│   ├── 101_log.npz                           # Controller 101
│   └── 102_log.npz                           # Controller 102
└── imaging_data_log/                         # DataLogger instance_name="imaging"
    ├── camera_manifest.yaml                  # Camera manifest (from axvs)
    ├── 51_log.npz                            # Camera system_id=51
    └── 52_log.npz                            # Camera system_id=52
```

Each log directory is an **independent processing unit**. The discovery tool groups archives by their
parent directory, and each directory is prepared and processed independently.

### Mixed axvs and axci recording

When microcontrollers and cameras share a DataLogger, the log directory contains both types of manifests
and archives. The AXCI processing pipeline only processes archives referenced in the
`microcontroller_manifest.yaml`, and the AXVS pipeline only processes archives referenced in
`camera_manifest.yaml`.

---

## Archive internal structure

### Message format

Each entry in an `.npz` archive stores a serialized message as a byte array:

```text
[source_id: 1 byte][elapsed_us: 8 bytes (uint64)][protocol: 1 byte][payload: N bytes]
```

Archive keys follow the pattern `{source_id:03d}_{elapsed_us:020d}`, preserving the 3-digit zero-padded
source ID and 20-digit zero-padded timestamp from the original `.npy` filenames.

### Protocol codes

| Protocol Code | Name           | Description                                     |
|---------------|----------------|-------------------------------------------------|
| 6             | MODULE_DATA    | Data message from a hardware module             |
| 7             | KERNEL_DATA    | Data message from the kernel                    |
| 8             | MODULE_STATE   | State/status message from a hardware module     |
| 9             | KERNEL_STATE   | State/status message from the kernel            |

### Message types

| Type  | Identifier        | Payload                                      | Purpose                     |
|-------|-------------------|----------------------------------------------|-----------------------------|
| Onset | `elapsed_us == 0` | 8 bytes: int64 UTC epoch microseconds        | Absolute time reference     |
| Data  | `elapsed_us > 0`  | Protocol byte + command + event + typed data | Module/kernel data message  |
| State | `elapsed_us > 0`  | Protocol byte + command + event              | Module/kernel state message |

**Onset message:** The first message in every archive has `elapsed_us=0`. Its payload contains the UTC
epoch timestamp (microseconds since epoch) that serves as the absolute time reference. All other
timestamps in the archive are relative to this onset.

**Data/State messages:** Each communication event produces a message with `elapsed_us` set to the
microseconds elapsed since onset. The processing pipeline extracts messages matching the extraction
config's event codes, computes absolute timestamps, and writes them to feather files.

### Data payload structure

For MODULE_DATA and KERNEL_DATA messages, the payload contains:

```text
[command: 1 byte (uint8)][event: 1 byte (uint8)][prototype_code: 1 byte][data: N bytes]
```

- **command** — The command code the module/kernel was executing
- **event** — The event code identifying the message type
- **prototype_code** — Identifies the numpy dtype and size of the data bytes (auto-resolved at
  compile time by the firmware library)
- **data** — The serialized data value

For MODULE_STATE and KERNEL_STATE messages (no data):

```text
[command: 1 byte (uint8)][event: 1 byte (uint8)]
```

### Timestamp resolution

The processing pipeline converts relative timestamps to absolute UTC microseconds:

```text
absolute_timestamp_us = onset_us + elapsed_us
```

---

## Processing prerequisites

Before running the log processing pipeline, verify these conditions:

1. **Microcontroller manifest present** — Log directories should contain a
   `microcontroller_manifest.yaml` file for `discover_microcontroller_data_tool` to locate them. If
   missing, use `write_microcontroller_manifest_tool` to create one.

2. **Archives assembled** — Log directories contain `.npz` files, not just raw `.npy` files. If only
   `.npy` files are present, `assemble_log_archives_tool` must be run first.

3. **Archive naming valid** — Files match the `{source_id}_log.npz` pattern.

4. **Onset message present** — Each archive must contain exactly one onset message (elapsed_us=0)
   with a valid UTC epoch payload. Archives missing the onset message cannot be processed.

5. **Extraction config valid** — A validated `ExtractionConfig` YAML file must exist with event codes
   matching the firmware's data/state message events. See `/extraction-configuration`.

---

## Related skills

| Skill                        | Relationship                                                         |
|------------------------------|----------------------------------------------------------------------|
| `/microcontroller-setup`     | Upstream: MCP tools that assemble and discover archives              |
| `/microcontroller-interface` | Upstream: MicroControllerInterface instances that produce log data   |
| `/extraction-configuration`  | Context: extraction config determines which messages are extracted   |
| `/log-processing`            | Downstream: consumes archives in the format documented here          |
| `/log-processing-results`    | Downstream: documents the output format produced from these archives |
| `/pipeline`                  | Context: reference skill for the end-to-end pipeline phases          |
| `/mcp-environment-setup`     | Prerequisite: MCP server connectivity for discovery and processing   |

---

## Verification checklist

```text
Log Input Format:
- [ ] Microcontroller manifest (microcontroller_manifest.yaml) present in log directories
- [ ] Log directories contain assembled .npz archives (not raw .npy files)
- [ ] Archive filenames match {source_id}_log.npz pattern
- [ ] Source IDs are unique within each log directory
- [ ] Each archive contains an onset message (elapsed_us=0 with UTC epoch payload)
- [ ] Extraction config has event codes matching firmware message events
- [ ] Directory structure matches expected DataLogger output layout
```
