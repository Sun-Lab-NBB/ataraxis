---
name: log-input-format
description: >-
  Documents the input data format required by the log processing pipeline: NPZ log archives produced by
  DataLogger, source ID semantics, microcontroller manifest system, archive internal message layout, and
  communication protocol. Use when the user asks about microcontroller (serial) log archive format, source
  IDs, the manifest system, DataLogger output, or why microcontroller log processing fails due to missing or
  malformed archives.
user-invocable: false
---

# Log input format

Documents the input data format required by the microcontroller data extraction pipeline, including how log archives are
produced, their internal structure, manifest system, and source ID semantics.

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
- MCP server connectivity (see `/communication-mcp-environment-setup`)

---

## Archive format

### Naming convention

Log archives follow the naming pattern `{source_id}_log.npz`:

```text
101_log.npz    # Source ID 101 (from MicroControllerInterface controller_id=101)
102_log.npz    # Source ID 102 (from MicroControllerInterface controller_id=102)
51_log.npz     # Source ID 51 (from VideoSystem system_id=51, if sharing the logger)
```

The suffix `_log.npz` is defined as `LOG_ARCHIVE_SUFFIX` in `ataraxis-data-structures` (`serialized_data_logger.py`),
and imported by ataraxis-communication-interface's `orchestration/discovery.py` and `interfaces/discovery_tools.py`. The
source ID portion is the integer form of the originating system's controller ID.

### How archives are produced

Archives are created in two phases:

1. **Runtime logging**: A `DataLogger` instance receives `LogPackage` messages via its multiprocessing input queue. Each
   `MicroControllerInterface` sends all communication messages to the shared logger using its `controller_id` as the
   `source_id`. The logger persists each message as an individual `.npy` file named
   `{source_id:03d}_{acquisition_time:020d}.npy` inside its output directory.

2. **Post-runtime assembly**: `assemble_log_archives()` from `ataraxis-data-structures` consolidates all `.npy` files in
   a log directory into one `.npz` archive per unique source ID. It groups files by source ID, sorts by timestamp, and
   writes uncompressed `.npz` archives. The original `.npy` files are removed by default.

**You MUST ensure archives have been assembled before running the processing pipeline.** The pipeline expects `.npz`
files, not raw `.npy` files. If no `.npz` archives are found during discovery, instruct the user to run
`assemble_log_archives_tool` (see `/microcontroller-setup`) on the log directories first.

### Microcontroller manifest

Each DataLogger output directory should contain a `microcontroller_manifest.yaml` file that identifies which archives
were produced by ataraxis-communication-interface controllers. The manifest structure:

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

- **Automatic:** `MicroControllerInterface.__init__()` writes a manifest entry to the DataLogger output directory using
  the `controller_id`, `name`, and module list. Each MicroControllerInterface sharing a DataLogger appends its entry to
  the same manifest file.
- **Manual:** Use `write_microcontroller_manifest_tool` (see `/microcontroller-setup`) to retroactively tag legacy log
  directories that predate the manifest system.

**Why manifests matter:** The `discover_microcontroller_data_tool` uses manifest-based routing to identify
controller-produced log archives. Directories without a `microcontroller_manifest.yaml` will not be discovered.
Manifests also associate controller IDs with human-readable names and enumerate the hardware modules managed by each
controller.

**Manifest gates, config selects:** The `ExtractionConfig` bounds the work. A controller it does not declare is never
processed, whatever the manifest registers and whatever `.npz` files sit on disk. Within that bound the caller's
`source_ids` argument selects, and omitting it requests every configured controller. The manifest only gates: a
requested controller it does not register yields no job, and neither the manifest nor the archives on disk can add a
controller the config omits.

**The failure model is not uniform:** what happens to a requested controller that clears neither gate depends on the
entry point, the local library path fails the whole call, the MCP path records the controller and prepares the rest.
`/log-processing` is authoritative for that lenient-sourcing model, for the per-reason remedies, and for the
`skipped_sources` key that reports them. Read it before diagnosing a batch that prepared fewer jobs than requested.

**Key difference from ataraxis-video-system manifests:** ataraxis-communication-interface manifests include a `modules`
list per controller, providing full hardware module metadata (type, id, name). ataraxis-video-system camera manifests
only have source ID and camera name.

---

## Source ID semantics

### What source IDs represent

A source ID is a `np.uint8` value (1-255) that identifies the hardware system that produced log data. In
ataraxis-communication-interface, each `MicroControllerInterface` instance has a `controller_id` that becomes the
`source_id` in all log entries sent to the `DataLogger`.

```text
MicroControllerInterface(controller_id=np.uint8(101), data_logger=logger)
    → LogPackage(source_id=np.uint8(101), ...)
    → Raw .npy files: 101_00000000000000000000.npy, 101_00000000000001234567.npy, ...
    → Assembled archive: 101_log.npz
    → Processed output: controller_101_module_1_1.feather, controller_101_kernel.feather
```

### Integer form versus string form

One source ID has two written forms, and the boundary between them is fixed. The manifest and the extraction
configuration store the ID as an **integer**. Every surface that names a job, an archive, a tracker entry, or an output
file uses its **string** form. The conversion is plain `str(id)` and `int(source_id)` with no padding, so controller 51
is `"51"` and never `"051"`.

| Surface                                                            | Form                             |
|--------------------------------------------------------------------|----------------------------------|
| `microcontroller_manifest.yaml` `id:` field                        | int                              |
| `write_microcontroller_manifest_tool(controller_id=)`              | int                              |
| `read_microcontroller_manifest_tool` -> `controllers[].id`         | int                              |
| `ExtractionConfig` `controller_id:`, `read_extraction_config_tool` | int                              |
| `write_extraction_config_tool(controllers=[...])`                  | either, the tool applies `int()` |
| `discover_microcontroller_data_tool` -> `sources[].source_id`      | str                              |
| `discover_microcontroller_data_tool` -> `breakdown.source_id` keys | str                              |
| `assemble_log_archives_tool` -> `source_ids`                       | str                              |
| `prepare_log_processing_batch_tool(source_ids=)`                   | str                              |
| `reset_log_processing_jobs_tool(source_ids=)`                      | str                              |
| every `source_id` in a prepare, execute, status, or timing return  | str                              |
| `{source_id}_log.npz`, `controller_{source_id}_*.feather`          | str, plain decimal, no padding   |
| `.npy` filenames and archive keys, `{source_id:03d}_...`           | zero-padded to three digits      |

**You MUST copy a `source_id` string verbatim** from a discovery, prepare, or status return into the next tool call.
Both consumers compare exactly. Job preparation tests each requested ID against the string forms of the configured
controller IDs, and `reset_log_processing_jobs_tool` tests them against the tracker's job specifiers. A zero-padded,
whitespace-bearing, or integer value matches nothing.

**A mismatch fails quietly.** A padded ID such as `"051"` raises nothing on the MCP path. The batch simply prepares no
job for it and reports it under `skipped_sources` as absent from the extraction configuration, which blames the config
rather than the malformed ID. Re-read the ID from `discover_microcontroller_data_tool` before editing a config in
response to that message.

### Uniqueness constraints

Source IDs have **two different uniqueness scopes**:

| Scope                       | Constraint                                                                                                                                                                 |
|-----------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Within a single DataLogger  | Source IDs MUST be unique. Multiple controllers sharing one DataLogger must have different controller_ids.                                                                 |
| Across DataLogger instances | Source IDs MAY repeat. Two loggers in the same recording can each have a source with the same ID without conflict, because each logger writes to its own output directory. |

This means source IDs are unique **per log directory**, not globally across a recording session.

### Common source ID assignments

Runtime MicroControllerInterface instances are advised to use IDs in the range **101-150**. This convention is advised
but not enforced. Any `np.uint8` value (1-255) is valid as long as source IDs are unique across **all** sources within
each DataLogger instance, including sources from other libraries (e.g., ataraxis-video-system). The 101-150 range avoids
collisions with other libraries' advised ranges.

---

## Recording directory structure

### Single-logger recording

A recording session with one DataLogger produces:

```text
recording_root/
└── session_data_log/                         # DataLogger output (instance_name="session")
    ├── microcontroller_manifest.yaml         # Controller manifest (auto-written on init)
    ├── 101_00000000000000000000.npy          # Raw logs (before assembly)
    ├── 101_00000000000001234567.npy
    ├── 102_00000000000000000000.npy
    ├── 101_log.npz                           # Assembled archive (after assembly)
    └── 102_log.npz
```

The DataLogger creates its output directory as `{instance_name}_data_log/` inside the provided `output_directory`. All
MicroControllerInterface instances sharing this logger write to the same directory, distinguished by their source IDs.

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

Each log directory is an **independent processing unit**. The discovery tool groups archives by their parent directory,
and each directory is prepared and processed independently.

### Mixed ataraxis-video-system and ataraxis-communication-interface recording

When microcontrollers and cameras share a DataLogger, the log directory contains both types of manifests and archives.
The ataraxis-communication-interface processing pipeline only processes archives referenced in the
`microcontroller_manifest.yaml`, and the ataraxis-video-system pipeline only processes archives referenced in
`camera_manifest.yaml`.

Both archive kinds carry the same `{source_id}_log.npz` name, so the manifest is the only thing that tells them apart.
An unregistered archive is invisible to a pipeline rather than mis-parsed. `/video:log-input-format` documents the
camera archives and their payload layout, and `/video:pipeline` owns the shared source ID namespace.

Assembly is directory-wide, not per-library. The source ID grouping above ignores which library wrote an entry, so
**one** call covers both libraries. Running it a second time after either side has already assembled is the destructive
case `/microcontroller-setup` documents.

---

## Archive internal structure

### Message format

Each entry in an `.npz` archive stores a serialized message as a byte array. The shared DataLogger container layout is:

```text
[source_id: 1 byte][elapsed_us: 8 bytes (uint64)][payload: N bytes]
```

Everything after the source ID and timestamp is the opaque payload, whose structure is domain-specific and must be
parsed by the consumer. For ataraxis-communication-interface data/state messages, the first payload byte is a `protocol`
byte that identifies the message type:

```text
[source_id: 1 byte][elapsed_us: 8 bytes (uint64)][protocol: 1 byte][payload: N bytes]
```

The onset message is the exception: its payload contains only the 8-byte timestamp and no protocol byte.

Archive keys follow the pattern `{source_id:03d}_{elapsed_us:020d}`, preserving the 3-digit zero-padded source ID and
20-digit zero-padded timestamp from the original `.npy` filenames.

### Protocol codes

`SerialProtocols` (`communication/protocols.py`) defines thirteen codes, and the firmware's `kProtocols` enumeration
mirrors them value for value. See `/microcontroller:firmware-module` for the firmware side.

| Code | Name                      | Direction | Role                                                 |
|------|---------------------------|-----------|------------------------------------------------------|
| 0    | UNDEFINED                 | N/A       | Initializer only, never appears on the wire          |
| 1    | REPEATED_MODULE_COMMAND   | PC -> MC  | Recurrently executed module command                  |
| 2    | ONE_OFF_MODULE_COMMAND    | PC -> MC  | Single-shot module command                           |
| 3    | DEQUEUE_MODULE_COMMAND    | PC -> MC  | Clears a module's queued commands                    |
| 4    | KERNEL_COMMAND            | PC -> MC  | Kernel command, always single-shot                   |
| 5    | MODULE_PARAMETERS         | PC -> MC  | Module parameter object                              |
| 6    | MODULE_DATA               | MC -> PC  | **Extracted**, module event plus a typed data object |
| 7    | KERNEL_DATA               | MC -> PC  | **Extracted**, kernel event plus a typed data object |
| 8    | MODULE_STATE              | MC -> PC  | **Extracted**, module event code alone               |
| 9    | KERNEL_STATE              | MC -> PC  | **Extracted**, kernel event code alone               |
| 10   | RECEPTION_CODE            | MC -> PC  | Acknowledges a received command or parameter message |
| 11   | CONTROLLER_IDENTIFICATION | MC -> PC  | Reports the controller's own ID                      |
| 12   | MODULE_IDENTIFICATION     | MC -> PC  | Reports one module's combined type-and-id code       |

**Both directions are logged.** `SerialCommunication` logs every message it sends as well as every payload it receives,
so an archive holds outgoing commands and parameters (1-5) and inbound service messages (10-12) alongside the data and
state messages the pipeline extracts. Only protocols 6-9 are routed to an accumulator, and every other code is skipped
without error. An archive's message count therefore exceeds its extracted row count by design, before the per-module
event-code filter narrows the extracted set further. Do not read that gap as data loss. See `/log-processing-results`.

### Message types

| Type  | Identifier        | Payload                                                                                              | Purpose                     |
|-------|-------------------|------------------------------------------------------------------------------------------------------|-----------------------------|
| Onset | `elapsed_us == 0` | 8 bytes: uint64 UTC epoch microseconds                                                               | Absolute time reference     |
| Data  | `elapsed_us > 0`  | Protocol byte + module type and ID (module messages) + command + event + prototype code + typed data | Module/kernel data message  |
| State | `elapsed_us > 0`  | Protocol byte + module type and ID (module messages) + command + event                               | Module/kernel state message |

**Onset message:** The first message in every archive has `elapsed_us=0`. Its payload contains the UTC epoch timestamp
(microseconds since epoch) that serves as the absolute time reference. All other timestamps in the archive are relative
to this onset.

**Data/State messages:** Each communication event produces a message with `elapsed_us` set to the microseconds elapsed
since onset. The processing pipeline extracts messages matching the extraction config's event codes, computes absolute
timestamps, and writes them to feather files.

### Data payload structure

After the leading protocol byte, the remaining bytes follow protocol-specific layouts.

**MODULE_DATA** (protocol 6):

```text
[module_type: 1 byte][module_id: 1 byte][command: 1 byte][event: 1 byte][prototype_code: 1 byte][data: N bytes]
```

**MODULE_STATE** (protocol 8):

```text
[module_type: 1 byte][module_id: 1 byte][command: 1 byte][event: 1 byte]
```

**KERNEL_DATA** (protocol 7):

```text
[command: 1 byte][event: 1 byte][prototype_code: 1 byte][data: N bytes]
```

**KERNEL_STATE** (protocol 9):

```text
[command: 1 byte][event: 1 byte]
```

**Service messages** (protocols 10, 11, 12) share one layout, a bare service code with no command or event byte:

```text
[service_code: 1, 2, or 4 bytes]
```

The firmware's packed `ServiceMessage` struct admits a `uint8`, `uint16`, or `uint32` code. In practice RECEPTION_CODE
and CONTROLLER_IDENTIFICATION send one byte, and MODULE_IDENTIFICATION sends the module's combined type-and-id value as
a `uint16`. These messages carry no event code, so no extraction config can select them.

- **module_type**: Module family code of the sending module (module messages only)
- **module_id**: Instance ID of the sending module (module messages only)
- **command**: The command code the module/kernel was executing
- **event**: The event code identifying the message type. The processing pipeline filters on the event code alone (the
  command is recorded but not used to select messages), and firmware guarantees event codes are unique within each
  module/kernel. See `/extraction-configuration`.
- **prototype_code**: Identifies the numpy dtype and size of the data bytes (auto-resolved at compile time by the
  firmware library. Data messages only)
- **data**: The serialized data value (data messages only)

### Timestamp resolution

The processing pipeline converts relative timestamps to absolute UTC microseconds:

```text
absolute_timestamp_us = onset_us + elapsed_us
```

---

## Processing prerequisites

Before running the log processing pipeline, verify these conditions:

1. **Microcontroller manifest present**: Log directories contain a `microcontroller_manifest.yaml` file.

2. **Archives assembled**: Log directories contain `.npz` files, not just raw `.npy` files.

3. **Archive naming valid**: Files match the `{source_id}_log.npz` pattern.

   - **One archive per source ID**: Exactly one `{source_id}_log.npz` may exist anywhere under the search root.
     `resolve_jobs()` indexes the archive names with `index_marker_files()`, and a source ID resolving to zero files or
     to several is left unresolved, duplicates never resolve first-wins. The unresolved source then either fails the
     call or is reported as a skip, on the entry-point split described under "Manifest gates, config selects" above and
     owned by `/log-processing`.
   - **Single parent directory per invocation**: All source IDs processed in one invocation must resolve to the same
     parent directory. If the resolved archives span more than one directory, job preparation raises a `ValueError`
     ("The resolved log archives sit in N different directories ... Each DataLogger output directory must be prepared
     and processed on its own invocation"). Unlike the two gates above, this check is not subject to the
     lenient-versus-strict split and raises on both entry points. Point the pipeline at each logger directory
     separately.
   - **One manifest per invocation**: A search root holding more than one `microcontroller_manifest.yaml` spans several
     recordings and raises a `ValueError` before any archive is indexed, on both entry points.

4. **Onset message present**: Each archive contains exactly one onset message (elapsed_us=0) with a valid UTC epoch
   payload.

5. **Extraction config valid**: A validated `ExtractionConfig` YAML file must exist with event codes matching the
   firmware's data/state message events. See `/extraction-configuration`.

---

## Related skills

| Skill                                  | Relationship                                                         |
|----------------------------------------|----------------------------------------------------------------------|
| `/microcontroller-setup`               | Upstream: MCP tools that assemble and discover archives              |
| `/microcontroller-interface`           | Upstream: MicroControllerInterface instances that produce log data   |
| `/extraction-configuration`            | Context: extraction config determines which messages are extracted   |
| `/log-processing`                      | Downstream: consumes archives in the format documented here          |
| `/log-processing-results`              | Downstream: documents the output format produced from these archives |
| `/cli-reference`                       | Reference: the `axci process` path that reads these archives         |
| `/pipeline`                            | Context: reference skill for the end-to-end pipeline phases          |
| `/communication-mcp-environment-setup` | Prerequisite: MCP server connectivity for discovery and processing   |
| `/microcontroller:firmware-module`     | Context: firmware side of the protocol codes and message layouts     |
| `/video:log-input-format`              | Peer: the camera archives that may share the same log directory      |

---

## Verification checklist

```text
Log Input Format:
- [ ] Microcontroller manifest (microcontroller_manifest.yaml) present in log directories
- [ ] Log directories contain assembled .npz archives (not raw .npy files)
- [ ] Archive filenames match {source_id}_log.npz pattern
- [ ] Source IDs are unique within each log directory
- [ ] Source IDs passed to processing tools are unpadded strings copied verbatim from a discovery return
- [ ] Each archive contains an onset message (elapsed_us=0 with UTC epoch payload)
- [ ] Extraction config has event codes matching firmware message events
- [ ] Directory structure matches expected DataLogger output layout
```
