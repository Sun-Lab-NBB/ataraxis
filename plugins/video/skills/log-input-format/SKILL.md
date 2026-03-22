---
name: log-input-format
description: >-
  Documents the input data format required by the log processing pipeline: NPZ log archives produced by
  DataLogger and assemble_log_archives, source ID semantics, multi-logger recording structures, and
  archive internal layout. Use when the user asks about log archive format, source IDs, DataLogger output,
  or why processing fails due to missing or malformed archives.
user-invocable: true
---

# Log input format

Documents the input data format required by the camera timestamp extraction pipeline, including how log
archives are produced, their internal structure, and source ID semantics.

---

## Scope

**Covers:**
- NPZ archive format and naming conventions
- How DataLogger and assemble_log_archives produce archives
- Camera manifest system for archive identification
- Source ID semantics, assignment, and uniqueness constraints
- Multi-logger recording structures
- Archive internal message layout (onset, frame, data messages)
- Directory hierarchy from recording root to log archives
- Prerequisites for running the processing pipeline

**Does not cover:**
- Processing workflow or batch operations (see `/log-processing`)
- Output data formats or frame statistics (see `/log-processing-results`)
- Camera hardware setup or configuration (see `/camera-setup`)
- MCP server connectivity (see `/mcp-environment-setup`)

---

## Archive format

### Naming convention

Log archives follow the naming pattern `{source_id}_log.npz`:

```text
51_log.npz     # Source ID 51 (from VideoSystem system_id=51)
112_log.npz    # Source ID 112 (from VideoSystem system_id=112)
1_log.npz      # Source ID 1
```

The suffix `_log.npz` is defined as `LOG_ARCHIVE_SUFFIX` in `log_processing.py`. The source ID portion
is the integer form of the originating system's ID — leading zeros from the raw `.npy` filenames are
stripped during archive assembly.

### How archives are produced

Archives are created in two phases:

1. **Runtime logging** — A `DataLogger` instance receives `LogPackage` messages via its multiprocessing
   input queue. Each `VideoSystem` sends frame acquisition timestamps to the shared logger using its
   `system_id` as the `source_id`. The logger persists each message as an individual `.npy` file named
   `{source_id:03d}_{acquisition_time:020d}.npy` inside its output directory.

2. **Post-runtime assembly** — `assemble_log_archives()` from `ataraxis-data-structures` consolidates
   all `.npy` files in a log directory into one `.npz` archive per unique source ID. It groups files by
   source ID (extracted as `int(filename.split("_")[0])`), sorts by timestamp, and writes uncompressed
   `.npz` archives. The original `.npy` files are removed by default.

**You MUST ensure archives have been assembled before running the processing pipeline.** The pipeline
expects `.npz` files, not raw `.npy` files. If no `.npz` archives are found during discovery, instruct
the user to run `assemble_log_archives()` on the log directories first.

### Camera manifest

Each DataLogger output directory should contain a `camera_manifest.yaml` file that identifies which
archives were produced by ataraxis-video-system. The manifest is a YAML file with the following structure:

```yaml
sources:
- id: 51
  name: face_camera
- id: 52
  name: body_camera
```

**How manifests are produced:**

- **Automatic:** `VideoSystem.__init__()` writes a manifest entry to the DataLogger output directory using
  the `name` parameter. Each VideoSystem sharing a DataLogger appends its entry to the same manifest file.
- **MCP sessions:** `start_video_session` creates a VideoSystem with `name="live_camera"`, which writes
  a manifest automatically.
- **Manual:** Use `write_camera_manifest_tool` (see `/camera-setup`) to retroactively tag legacy log
  directories that predate the manifest system.

**Why manifests matter:** The `discover_recording_log_archives_tool` uses manifest-based routing to
identify axvs-produced log archives. Directories without a `camera_manifest.yaml` will not be discovered
by this tool. Manifests also associate source IDs with human-readable names and enable the discovery tool
to locate corresponding video files by camera name.

---

## Source ID semantics

### What source IDs represent

A source ID is a `np.uint8` value (0–255) that identifies the hardware system that produced log data.
In ataraxis-video-system, each `VideoSystem` instance has a `system_id` that becomes the `source_id`
in all log entries sent to the `DataLogger`.

```text
VideoSystem(system_id=51, data_logger=logger)
    → LogPackage(source_id=np.uint8(51), ...)
    → Raw .npy files: 051_00000000000000000000.npy, 051_00000000000001234567.npy, ...
    → Assembled archive: 51_log.npz
    → Processed output: camera_51_timestamps.feather
```

### Uniqueness constraints

Source IDs have **two different uniqueness scopes**:

| Scope                       | Constraint                                                              |
|-----------------------------|-------------------------------------------------------------------------|
| Within a single DataLogger  | Source IDs MUST be unique. Multiple VideoSystems sharing one DataLogger |
|                             | must have different system_ids.                                         |
| Across DataLogger instances | Source IDs MAY repeat. Two loggers in the same recording can each have  |
|                             | a source with the same ID without conflict, because each logger writes  |
|                             | to its own output directory.                                            |

This means source IDs are unique **per log directory**, not globally across a recording session.

### Common source ID assignments

Runtime VideoSystem instances (actual recording cameras) are advised to use IDs in the range 51-100.
The CLI (`system_id=111`) and MCP server (`system_id=112`) use fixed IDs for testing and exploration
sessions, not production recording.

---

## Recording directory structure

### Single-logger recording

A recording session with one DataLogger produces:

```text
recording_root/
├── video_051.mp4                        # Video output from VideoSystem (system_id=51)
├── video_052.mp4                        # Video output from VideoSystem (system_id=52)
└── session_data_log/                    # DataLogger output (instance_name="session")
    ├── camera_manifest.yaml             # Camera manifest (auto-written by VideoSystem.__init__)
    ├── 051_00000000000000000000.npy     # Raw logs (before assembly)
    ├── 051_00000000000001234567.npy
    ├── 052_00000000000000000000.npy
    ├── 052_00000000000001234567.npy
    ├── 51_log.npz                       # Assembled archive (after assembly)
    └── 52_log.npz
```

The DataLogger creates its output directory as `{instance_name}_data_log/` inside the provided
`output_directory`. All VideoSystem instances sharing this logger write to the same directory,
distinguished by their source IDs.

### Multi-logger recording

A recording session can use multiple DataLogger instances. Each creates its own output directory:

```text
recording_root/
├── behavior_data_log/                   # DataLogger instance_name="behavior"
│   ├── camera_manifest.yaml             # Manifest for behavior cameras
│   ├── 51_log.npz                       # Camera system_id=51
│   └── 52_log.npz                       # Camera system_id=52
└── physiology_data_log/                 # DataLogger instance_name="physiology"
    ├── 1_log.npz                        # Neural system_id=1
    └── 2_log.npz                        # Neural system_id=2
```

Each log directory is an **independent processing unit**. The discovery tool groups archives by their
parent directory (the DataLogger output directory), and each directory is prepared and processed
independently.

### Multiple recordings under one root

The discovery tool recursively searches a root directory and groups log directories by recording root:

```text
experiment_data/
├── session_2025_03_20/
│   └── main_data_log/
│       ├── 51_log.npz
│       └── 52_log.npz
└── session_2025_03_21/
    └── main_data_log/
        ├── 51_log.npz
        └── 52_log.npz
```

Source IDs `51` and `52` appear in both sessions. This is valid because each log directory is processed
independently — source ID uniqueness is only required within a single log directory.

---

## Archive internal structure

### Message format

Each entry in an `.npz` archive stores a serialized message as a byte array:

```text
[source_id: 1 byte][elapsed_us: 8 bytes (uint64)][payload: N bytes]
```

Archive keys follow the pattern `{source_id:03d}_{elapsed_us:020d}`, preserving the 3-digit zero-padded
source ID and 20-digit zero-padded timestamp from the original `.npy` filenames.

### Message types

| Type  | Identifier        | Payload                               | Purpose                           |
|-------|-------------------|---------------------------------------|-----------------------------------|
| Onset | `elapsed_us == 0` | 8 bytes: int64 UTC epoch microseconds | Absolute time reference           |
| Frame | `elapsed_us > 0`  | Empty (`payload.size == 0`)           | Frame acquisition event           |
| Data  | `elapsed_us > 0`  | Non-empty (`payload.size > 0`)        | Generic data event (filtered out) |

**Onset message:** The first message in every archive has `elapsed_us=0`. Its payload contains the UTC
epoch timestamp (microseconds since epoch) that serves as the absolute time reference. All other
timestamps in the archive are relative to this onset.

**Frame messages:** Each frame saved by the VideoSystem produces a message with `elapsed_us` set to the
microseconds elapsed since onset and an empty payload. The processing pipeline extracts only these
messages.

**Data messages:** Messages with non-empty payloads carry domain-specific data. The camera timestamp
extraction pipeline filters these out (`payload.size == 0` check).

### Timestamp resolution

The processing pipeline converts relative timestamps to absolute UTC microseconds:

```text
absolute_timestamp_us = onset_us + elapsed_us
```

The `LogArchiveReader` from `ataraxis-data-structures` handles onset discovery and timestamp conversion.
Archives with fewer than 2000 messages are processed sequentially; larger archives use parallel
processing via `ProcessPoolExecutor`.

---

## Processing prerequisites

Before running the log processing pipeline, verify these conditions:

1. **Camera manifest present** — Log directories should contain a `camera_manifest.yaml` file for
   `discover_recording_log_archives_tool` to locate them. If missing, use `write_camera_manifest_tool`
   to create one. Note: the processing pipeline itself does not require manifests — they are only
   needed for the manifest-based discovery tool.

2. **Archives assembled** — Log directories contain `.npz` files, not just raw `.npy` files. If only
   `.npy` files are present, `assemble_log_archives()` must be run first.

3. **Archive naming valid** — Files match the `{source_id}_log.npz` pattern. The preparation tool uses
   `LOG_ARCHIVE_SUFFIX` (`_log.npz`) to identify archives within each log directory.

4. **Onset message present** — Each archive must contain exactly one onset message (timestamp=0) with
   a valid UTC epoch payload. Archives missing the onset message cannot be processed.

5. **Frame messages present** — Archives must contain at least one frame message (empty payload) to
   produce output. Archives with only onset or data messages yield empty results.

---

## Related skills

| Skill                     | Relationship                                                           |
|---------------------------|------------------------------------------------------------------------|
| `/log-processing`         | Downstream: consumes archives in the format documented here            |
| `/log-processing-results` | Downstream: documents the output format produced from these archives   |
| `/camera-interface`       | Upstream: VideoSystem instances that produce the log data              |
| `/mcp-environment-setup`  | Prerequisite: MCP server connectivity for discovery and processing     |

---

## Verification checklist

```text
Log Input Format:
- [ ] Camera manifest (camera_manifest.yaml) present in log directories (for discovery tool)
- [ ] Log directories contain assembled .npz archives (not raw .npy files)
- [ ] Archive filenames match {source_id}_log.npz pattern
- [ ] Source IDs are unique within each log directory
- [ ] Each archive contains an onset message (timestamp=0 with UTC epoch payload)
- [ ] Each archive contains frame messages (empty payload entries)
- [ ] Directory structure matches expected DataLogger output layout
```
