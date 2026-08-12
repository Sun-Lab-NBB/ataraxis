---
name: log-input-format
description: >-
  Documents the input data format required by the log processing pipeline: NPZ log archives produced by
  DataLogger and assemble_log_archives, source ID semantics, multi-logger recording structures, and
  archive internal layout. Use when the user asks about camera (video-system) log archive format, source IDs,
  DataLogger output, or why video log processing fails due to missing or malformed archives.
user-invocable: false
---

# Log input format

Documents the input data format required by the log processing pipeline, including how log archives are produced, their
internal structure, and source ID semantics.

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
- Running assembly, or recovering a session that never assembled (see `/post-recording`)
- The system ID allocation convention (see `/pipeline`)
- Archives written to the same directory by a sibling library (see `/communication:log-input-format`)
- MCP server connectivity (see `/video-mcp-environment-setup`)

**Handoff rules:** If a prerequisite below is unmet, invoke the skill that repairs it. Missing `.npz` archives and raw
`.npy` entries go to `/post-recording`, a missing manifest goes to `/camera-setup` for `write_camera_manifest_tool`, and
a source ID collision goes to `/pipeline`. Once every prerequisite holds, invoke `/log-processing`. If the archives in
question are `{controller_id}_log.npz` entries rather than camera archives, invoke `/communication:log-input-format`.

---

## Archive format

### Naming convention

Log archives follow the naming pattern `{source_id}_log.npz`:

```text
51_log.npz     # Source ID 51 (from VideoSystem system_id=51)
112_log.npz    # Source ID 112 (from VideoSystem system_id=112)
1_log.npz      # Source ID 1
```

The suffix `_log.npz` is the `LOG_ARCHIVE_SUFFIX` constant that ataraxis-data-structures defines and
ataraxis-video-system imports. The source ID portion is the integer form of the originating system's ID, so leading
zeros from the raw `.npy` filenames are stripped during archive assembly.

### How archives are produced

1. **Runtime logging**: A `DataLogger` instance receives `LogPackage` messages via its multiprocessing
   input queue. Each `VideoSystem` sends frame acquisition timestamps to the shared logger using its
   `system_id` as the `source_id`. The logger persists each message as an individual `.npy` file named
   `{source_id:03d}_{acquisition_time:020d}.npy` inside its output directory.

2. **Post-runtime assembly**: `assemble_log_archives()` from `ataraxis-data-structures` consolidates
   all `.npy` files in a log directory into one `.npz` archive per unique source ID. It groups files by
   source ID (extracted as `int(filename.split("_")[0])`), sorts by timestamp, and writes uncompressed
   `.npz` archives. The original `.npy` files are removed by default.

**You MUST ensure archives have been assembled before running the processing pipeline.** The pipeline expects `.npz`
files, not raw `.npy` files. If no `.npz` archives are found during discovery, instruct the user to run
`assemble_log_archives()` on the log directories first.

### Camera manifest

Each DataLogger output directory should contain a `camera_manifest.yaml` file that identifies which archives were
produced by ataraxis-video-system. The manifest is a YAML file with the following structure:

```yaml
sources:
- id: 51
  name: face_camera
- id: 52
  name: body_camera
```

**How manifests are produced:**

- **Automatic:** `VideoSystem.__init__()` writes a manifest entry to the DataLogger output directory using
  the `name` parameter. Each VideoSystem sharing a DataLogger registers into the same manifest file. The write
  is idempotent per source ID: an entry already registered under that ID is replaced rather than duplicated,
  and the read-replace-write sequence is held under a file lock so concurrent registrations are safe.
- **MCP sessions:** `start_video_session_tool` creates a VideoSystem with `name="live_camera"`, which writes
  a manifest automatically.
- **Manual:** Use `write_camera_manifest_tool` (see `/camera-setup`) to retroactively tag legacy log
  directories that predate the manifest system.

**Why manifests matter:** The manifest is a hard gate for both discovery and processing. `discover_camera_data_tool`
uses manifest-based routing to identify axvs-produced log archives, so directories without a `camera_manifest.yaml` will
not be discovered by this tool. Beyond discovery, processing requires the manifest as well. A tree holding none resolves
no job, and a manifest with no source entries raises `ValueError`. The source IDs to process are resolved from the
manifest when none are requested, and any requested source ID the manifest does not register is rejected. Manifests also
associate source IDs with human-readable names and enable the discovery tool to locate corresponding video files by
camera name.

**One manifest per tree:** exactly one `camera_manifest.yaml` may sit under the directory being processed. A tree
holding several spans several recordings or several DataLogger instances and raises `ValueError` rather than resolving
against the first match. Pass each DataLogger output directory individually.

---

## Source ID semantics

### What source IDs represent

A source ID is a `np.uint8` value (0-255) that identifies the hardware system that produced log data. In
ataraxis-video-system, each `VideoSystem` instance has a `system_id` that becomes the `source_id` in all log entries
sent to the `DataLogger`.

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

### Common source ID assignments

The CLI (`system_id=111`) and MCP server (`system_id=112`) use fixed IDs for testing and exploration sessions, not
production recording. These two are the only values the library reserves. Runtime VideoSystem instances (actual
recording cameras) are advised to use IDs in the range 51-100, which is this plugin's allocation convention rather than
a library-enforced range (see `/pipeline`).

---

## Recording directory structure

### Single-logger recording

```text
recording_root/
├── 051.mp4                             # Video output from VideoSystem (system_id=51)
├── 052.mp4                             # Video output from VideoSystem (system_id=52)
└── session_data_log/                    # DataLogger output (instance_name="session")
    ├── camera_manifest.yaml             # Camera manifest (auto-written by VideoSystem.__init__)
    ├── 051_00000000000000000000.npy     # Raw logs (before assembly)
    ├── 051_00000000000001234567.npy
    ├── 052_00000000000000000000.npy
    ├── 052_00000000000001234567.npy
    ├── 51_log.npz                       # Assembled archive (after assembly)
    └── 52_log.npz
```

The DataLogger creates its output directory as `{instance_name}_data_log/` inside the provided `output_directory`. All
VideoSystem instances sharing this logger write to the same directory, distinguished by their source IDs.

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

### Directories shared with a sibling library

A DataLogger is an ataraxis-data-structures object, so a directory may hold archives from several libraries at once. A
camera archive and a microcontroller archive are indistinguishable by filename, since both follow `{source_id}_log.npz`
. What separates them is the manifest:

```text
session_data_log/
├── camera_manifest.yaml                 # Registers 51 and 52
├── microcontroller_manifest.yaml        # Registers 101
├── 51_log.npz                           # Camera
├── 52_log.npz                           # Camera
└── 101_log.npz                          # Microcontroller
```

Discovery here reads `camera_manifest.yaml` alone and never resolves 101, so an unregistered archive is invisible rather
than mis-parsed. The payload layouts differ as well, and `/communication:log-input-format` documents the microcontroller
side. `/pipeline` owns the shared-namespace rules that keep the source IDs from colliding.

Each log directory is an **independent processing unit**. The discovery tool groups archives by their parent directory
(the DataLogger output directory), and each directory is prepared and processed independently.

Source-ID-to-archive resolution requires exactly one matching `{source_id}_log.npz` under the recursively searched tree,
raising `FileNotFoundError` when the archive is absent or resolves to more than one file. All resolved archives must
share a single parent directory, and otherwise the resolution raises `ValueError` reading "Each DataLogger output
directory must be prepared and processed on its own invocation". Processing therefore runs once per DataLogger output
directory, not once per recording root, and a duplicate source ID across nested subdirectories is a fatal error, not a
combined-processing path.

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

Source IDs `51` and `52` appear in both sessions. This is valid because each log directory is processed independently.
Source ID uniqueness is only required within a single log directory.

---

## Archive internal structure

### Message format

Each entry in an `.npz` archive stores a serialized message as a byte array:

```text
[source_id: 1 byte][elapsed_us: 8 bytes (uint64)][payload: N bytes]
```

Archive keys follow the pattern `{source_id:03d}_{elapsed_us:020d}`, preserving the 3-digit zero-padded source ID and
20-digit zero-padded timestamp from the original `.npy` filenames.

### Message types

| Type  | Identifier        | Payload                                | Purpose                           |
|-------|-------------------|----------------------------------------|-----------------------------------|
| Onset | `elapsed_us == 0` | 8 bytes: uint64 UTC epoch microseconds | Absolute time reference           |
| Frame | `elapsed_us > 0`  | Empty (`payload.size == 0`)            | Frame acquisition event           |
| Data  | `elapsed_us > 0`  | Non-empty (`payload.size > 0`)         | Generic data event (filtered out) |

**Onset message:** The first message in every archive has `elapsed_us == 0`. Its payload contains the UTC epoch
timestamp (microseconds since epoch) that serves as the absolute time reference. All other timestamps in the archive
are relative to this onset.

**Frame messages:** Each frame saved by the VideoSystem produces a message with `elapsed_us` set to the microseconds
elapsed since onset and an empty payload. The processing pipeline extracts only these messages.

### Timestamp resolution

The processing pipeline converts relative timestamps to absolute UTC microseconds:

```text
absolute_timestamp_us = onset_us + elapsed_us
```

The `LogArchiveReader` from `ataraxis-data-structures` handles onset discovery and timestamp conversion. An archive
holding fewer messages than the reader's own parallel processing threshold is always read sequentially. A larger
archive opens a `ProcessPoolExecutor` only when its job was dispatched at more than one worker, which the extraction
stage decides from a separate and higher threshold that `/log-processing` documents.

---

## Processing prerequisites

1. **Camera manifest present**: exactly one `camera_manifest.yaml` must sit under the tree being processed, as "Why
   manifests matter" and "One manifest per tree" above spell out. Job IDs are a deterministic function of `(job_name,
   source_id)`, so an "invalid job_id" error likewise means the source ID is not registered in the manifest. If missing,
   use `write_camera_manifest_tool` to create one.

2. **Archives assembled**: log directories hold assembled `.npz` files rather than raw `.npy` files, as "How archives
   are produced" above spells out.

3. **Archive naming valid**: files match the `{source_id}_log.npz` pattern, as "Naming convention" above spells out.

4. **Onset message present**: Each archive must contain exactly one onset message (`elapsed_us == 0`) with a valid UTC
   epoch payload. Archives missing the onset message cannot be processed.

5. **Frame messages present**: Archives must contain at least one frame message (empty payload) to
   produce output. Archives with only onset or data messages yield empty results.

---

## Related skills

| Skill                             | Relationship                                                         |
|-----------------------------------|----------------------------------------------------------------------|
| `/camera-setup`                   | Upstream: MCP sessions, and owns `write_camera_manifest_tool`        |
| `/camera-interface`               | Upstream: VideoSystem instances that produce the log data            |
| `/post-recording`                 | Upstream: validates and assembles archives in this format            |
| `/cli-reference`                  | Upstream: `axvs run` assembles the same archives on every exit path  |
| `/log-processing`                 | Downstream: consumes archives in the format documented here          |
| `/log-processing-results`         | Downstream: documents the output format produced from these archives |
| `/pipeline`                       | Owns source ID allocation and the shared-DataLogger namespace        |
| `/video-mcp-environment-setup`    | Prerequisite: MCP server connectivity for discovery and processing   |
| `/communication:log-input-format` | Peer: the microcontroller archives that may share the directory      |

---

## Verification checklist

```text
Log Input Format, tool-settled (run `discover_camera_data_tool` on the recording root under `include_items=True` and
`detailed=True`, then `prepare_log_processing_batch_tool`):
- [ ] Camera manifest (camera_manifest.yaml) present in log directories (for discovery tool)
- [ ] Log directories contain assembled .npz archives (not raw .npy files)
- [ ] Archive filenames match {source_id}_log.npz pattern

Log Input Format, reader-judged:
- [ ] Source IDs are unique within each log directory
- [ ] Each archive contains an onset message (`elapsed_us == 0` with UTC epoch payload)
- [ ] Each archive contains frame messages (empty payload entries)
- [ ] Directory structure matches expected DataLogger output layout
```
