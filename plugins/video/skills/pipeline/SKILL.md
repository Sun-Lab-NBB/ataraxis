---
name: pipeline
description: >-
  Orchestrates the end-to-end ataraxis-video-system recording and analysis pipeline. Covers canonical phase
  ordering with handoff conditions, multi-camera planning with system ID allocation and DataLogger topology,
  and decision trees for interface, encoding, and processing configuration. Use when planning a full recording
  workflow, setting up multi-camera rigs, or deciding between MCP and code.
user-invocable: false
---

# Pipeline

Orchestrates the end-to-end camera recording and data analysis pipeline.

---

## Scope

**Covers:**
- Canonical pipeline phase ordering with handoff conditions
- System ID allocation, which this skill owns for the whole plugin
- Multi-camera planning: DataLogger topology and the coordinated lifecycle
- Multi-camera log processing and cross-camera frame statistics comparison
- Cross-system recordings that share one DataLogger with a sibling ataraxis library
- MCP vs code decision guidance
- Quick-start references for common scenarios

**Does not cover:**
- Detailed tool usage for any individual phase (see phase-specific skills)
- Encoding parameter selection (see `/camera-interface`)
- Core and memory budget planning for a processing batch (see `/log-processing`)
- The `axvs` command surface (see `/cli-reference`)
- MCP server connectivity (see `/video-mcp-environment-setup`)

**Handoff rules:** This skill dispatches to phase-specific skills at each stage. You MUST invoke the relevant skill
for detailed tool usage, parameter reference, and troubleshooting.

---

## Pipeline phases

```text
Phase 1  Environment setup     →  /video-mcp-environment-setup
Phase 2  Camera discovery      →  /camera-setup
Phase 3  Recording session     →  /camera-setup (MCP) or /camera-interface (code)
Phase 4  Post-recording        →  /post-recording
Phase 5  Log processing        →  /log-processing
Phase 6  Results analysis      →  /log-processing-results
```

Three skills sit outside the phase order and serve every phase. `/pipeline` plans the run, `/log-input-format` documents
the archives phases 3 through 5 hand along, and `/cli-reference` answers questions about the `axvs` commands a user runs
by hand.

### Who owns what

Every one of the MCP server's 27 tools has exactly one skill that documents its parameters and return structure. Other
skills name a tool only to say which of its fields they read. When two skills seem to disagree, the owner below is
authoritative.

| Skill                          | Owns                                                                             |
|--------------------------------|----------------------------------------------------------------------------------|
| `/video-mcp-environment-setup` | Server reachability, the install, and the `axvs --help` exemption                |
| `/camera-setup`                | The runtime, CTI, camera discovery, session, GenICam, and manifest tools         |
| `/camera-interface`            | The VideoSystem API, the MCP-to-code mapping, and every encoding recommendation  |
| `/post-recording`              | `assemble_log_archives_tool` and `validate_video_file_tool`                      |
| `/log-processing`              | `discover_camera_data_tool`, the batch tools, and job resource sizing            |
| `/log-processing-results`      | `analyze_camera_frame_statistics_tool` and the feather output schema             |
| `/log-input-format`            | The `.npz` archive layout, the manifest contract, and processing prerequisites   |
| `/cli-reference`               | Every `axvs` command and option, and the MCP-down fallback                       |
| `/pipeline`                    | Phase ordering, system ID allocation, DataLogger topology, cross-camera analysis |

### Phase 1: Environment setup

- **Skill:** `/video-mcp-environment-setup`
- **Actions:** Verify MCP server connectivity, check `axvs` command availability, verify Python version
- **Handoff condition:** MCP tools accessible, and `check_runtime_requirements_tool` returns OK for all needed
  components
- **Skip condition:** MCP already verified in this session

### Phase 2: Camera discovery and configuration

- **Skill:** `/camera-setup`
- **Actions:** `check_runtime_requirements_tool`, `list_cameras_tool`, configure CTI if Harvesters, inspect and
  configure GenICam nodes
- **Handoff condition:** Camera(s) discoverable, encoding requirements met (FFMPEG OK, GPU if needed)
- **Decision point:** Single camera vs multi-camera (see multi-camera planning below)

### Phase 3: Recording session

- **Skill:** `/camera-setup` (MCP) or `/camera-interface` (code)
- **MCP path:** `start_video_session_tool` → `start_frame_saving_tool` →
  `stop_frame_saving_tool` → `stop_video_session_tool`
- **Code path:** DataLogger init/start → VideoSystem init/start → `start_frame_saving` →
  `stop_frame_saving` → `stop` → logger stop → `assemble_log_archives`
- **Handoff condition:** Session stopped, video file(s) exist

### Phase 4: Post-recording verification

- **Skill:** `/post-recording`
- **Actions:** Validate video file, verify archives assembled, cross-reference frame counts
- **Handoff condition:** All archives present, video validated, readiness checklist passed

### Phase 5: Log processing

- **Skill:** `/log-processing`
- **Actions:** Discover archives, prepare batch, execute jobs, monitor progress
- **Handoff condition:** All jobs SUCCEEDED in ProcessingTracker

### Phase 6: Results analysis

- **Skill:** `/log-processing-results`
- **Actions:** Discover feather files, analyze frame statistics per camera, interpret results

---

## Decision trees

### Interface selection

```text
Does the camera support GenTL (GenICam Transport Layer)?
  YES → Is the host an Intel Mac, or any Mac on Python 3.14?
    YES → OpenCV (the GenICam runtime is never installed there)
    NO  → Harvesters (preferred interface, provides GenICam node control)
  NO  → Is the camera a USB webcam or consumer device?
    YES → OpenCV
    NO  → Is this a test or development scenario without hardware?
      YES → Mock
      NO  → Check camera vendor documentation for GenTL support
```

VideoSystem raises `NotImplementedError` for the Harvesters interface on the Macs that install no runtime, so the
platform gate comes before the camera's own capability. `/camera-setup` owns the runtime detail behind it.

### MCP vs code

| Scenario                                             | Recommendation                            |
|------------------------------------------------------|-------------------------------------------|
| Verify the host, discover cameras, configure GenICam | MCP via `/camera-setup`                   |
| Single camera, interactive testing or exploration    | MCP via `/camera-setup`                   |
| Single camera, production with custom encoding       | Code via `/camera-interface`              |
| Multi-camera simultaneous recording                  | Code via `/camera-interface`              |
| Verify outputs and assemble archives                 | MCP via `/post-recording`                 |
| Log processing (any scenario)                        | MCP via `/log-processing`                 |
| Results analysis (any scenario)                      | MCP via `/log-processing-results`         |
| Answer a question about an `axvs` command            | Reference only via `/cli-reference`       |
| Work an operation while the MCP server is down       | Hand the user a command, `/cli-reference` |

MCP supports only one active video session at a time. Multi-camera recording requires Python code.

There is no third path. Orchestration happens through the MCP tools or through the `axvs` CLI a user runs by hand. The
library exports its orchestration symbols, but they are not an agent-facing surface, so never drive a batch from Python.
You MUST never invoke `axvs` either, with `--help` the sole exemption `/video-mcp-environment-setup` owns.

### Encoding selection

`/camera-interface` owns the use-case table, the CPU and GPU trade-off, the cross-encoder quantization equivalence, and
the FFMPEG error catalog, and it is the only place those figures are authoritative. `/camera-setup` covers what is
specific to the MCP session defaults.

The one planning fact this skill contributes is that a multi-camera rig encodes every channel concurrently, so the
per-camera preset that works alone may not survive the rig. Plan for GPU encoding and a faster preset than a
single-camera setup would need, then confirm the choice against `/camera-interface`.

---

## Multi-camera planning

### System ID allocation

A camera's `system_id` IS its source ID at the DataLogger level. VideoSystem registers it as the `source_id`, and it
names the camera's `{system_id}_log.npz` archive (see `/log-input-format`). This skill uses "source ID" for the shared
DataLogger namespace and `system_id` for the VideoSystem constructor.

| Range   | Assignment                         | Notes                                                |
|---------|------------------------------------|------------------------------------------------------|
| 51-100  | Camera VideoSystem instances       | Plugin convention, not a library-enforced range      |
| 101-150 | MicroControllerInterface instances | The sibling communication plugin's convention. Avoid |
| 111     | CLI (`axvs run`)                   | Fixed in the library. Interactive testing only       |
| 112     | MCP server sessions                | Fixed in the library. Agent-driven testing only      |

The library constrains one thing and requires one more. A `system_id` must fit `np.uint8` (0-255, enforced with an
`OverflowError`), and every source sharing one DataLogger must carry a unique one. That second rule is a requirement the
library does not check, since a duplicate ID silently replaces the earlier manifest entry rather than raising. 111 and
112 are the only values the library itself reserves, and the axvs README's own quickstart uses 101 for a camera.

The 51-100 band is this plugin's allocation convention for keeping camera code clear of the reserved pair and of the
101-150 band `/communication:pipeline` advises for microcontrollers. You MUST confirm the rig's existing allocation with
the user rather than assuming it follows the convention. Within the band, allocate sequentially from 51 (51, 52, 53 for
a 3-camera rig).

Note that 111 falls inside the communication plugin's advised band, so a rig that runs `axvs run` against the same
DataLogger a controller 111 writes to collides. This is only a concern for interactive testing, since production camera
code never uses 111.

### DataLogger topology

A single shared DataLogger is the preferred topology for all use cases:

```text
DataLogger(instance_name="session")
  ├── VideoSystem(system_id=51, name="face_camera")    → 51_log.npz + camera_manifest.yaml
  ├── VideoSystem(system_id=52, name="body_camera")    → 52_log.npz
  └── VideoSystem(system_id=53, name="arena_camera")   → 53_log.npz
```

All cameras share one log directory, all timestamps are correlated, one `assemble_log_archives` call consolidates
everything, and one processing batch covers all source IDs. Each VideoSystem writes an entry to `camera_manifest.yaml`
during initialization, enabling manifest-based discovery downstream. The manifest write is idempotent per source ID,
because re-constructing a VideoSystem against an already-used output directory replaces that source's entry rather than
appending a duplicate. The read-replace-write sequence runs under a lock file beside the manifest and aborts if the lock
cannot be taken within 10 seconds, so the concurrent registrations of several VideoSystems sharing one DataLogger are
safe.

Use multiple DataLoggers only where the user reports a single logger's buffering backing up during a run, which is rare.
Each DataLogger then creates a separate output directory that must be assembled and processed independently, and
cross-camera timestamp comparison requires merging data from separate directories.

### Coordinated lifecycle

```text
Startup (in order):
  1. DataLogger(s) → __init__() → start()
  2. VideoSystem(s) → __init__() → start()
  3. All VideoSystems → start_frame_saving()

Shutdown (reverse order):
  4. All VideoSystems → stop_frame_saving()
  5. VideoSystem(s) → stop()
  6. DataLogger(s) → stop()
  7. assemble_log_archives() for each DataLogger output directory
```

- DataLogger must be started BEFORE any VideoSystem that references it
- VideoSystem must be stopped BEFORE its DataLogger
- Assembly must run AFTER `DataLogger.stop()`

### Multi-camera code structure

```python
from pathlib import Path

import numpy as np
from ataraxis_data_structures import DataLogger, assemble_log_archives

from ataraxis_video_system import CameraInterfaces, VideoSystem

# The guard is mandatory. The library sets the 'spawn' start method at import, and every spawned child
# re-imports this module, so unguarded module-level construction re-runs in each child.
if __name__ == "__main__":
    session_directory = Path("/path/to/session")

    # Starts the shared DataLogger first.
    logger = DataLogger(output_directory=session_directory, instance_name="session")
    logger.start()

    # Initializes and starts each camera with a unique system ID and descriptive name.
    cameras: list[VideoSystem] = []
    camera_configs = [(51, 0, "face_camera"), (52, 1, "body_camera"), (53, 2, "arena_camera")]
    for camera_id, camera_index, camera_name in camera_configs:
        camera = VideoSystem(
            system_id=np.uint8(camera_id),
            data_logger=logger,
            name=camera_name,
            output_directory=session_directory,
            camera_interface=CameraInterfaces.HARVESTERS,
            camera_index=camera_index,
        )
        camera.start()
        cameras.append(camera)

    # Starts frame saving on all cameras.
    for camera in cameras:
        camera.start_frame_saving()

    # ... recording ...

    # Shuts down in reverse order.
    for camera in cameras:
        camera.stop_frame_saving()
    for camera in cameras:
        camera.stop()
    logger.stop()

    # Assembles archives after the DataLogger has fully stopped.
    assemble_log_archives(log_directory=logger.output_directory, remove_sources=True)
```

---

## Cross-system recordings

A DataLogger is not a video-system object. It comes from ataraxis-data-structures, and any ataraxis library that logs
can share one. The common case is a rig where cameras and microcontrollers record the same session, which is what puts
every source's timestamps on one clock.

```text
DataLogger(instance_name="session")
  ├── VideoSystem(system_id=51)                  → 51_log.npz  + camera_manifest.yaml
  ├── VideoSystem(system_id=52)                  → 52_log.npz
  └── MicroControllerInterface(controller_id=101) → 101_log.npz + microcontroller_manifest.yaml
```

Four rules govern the shared directory:

- **One source ID namespace.** Uniqueness spans every source on the logger, not just the cameras. Allocate the
  camera IDs against the controller IDs already in use, and invoke `/communication:pipeline` for that side.
- **Two manifests, one directory.** Each library writes its own manifest naming only its own sources, and each
  library's discovery reads only its own. A camera source is invisible to the communication tooling, and the
  reverse holds, which is what keeps the two processing batches independent.
- **One assembly call.** `assemble_log_archives()` groups by source ID and consolidates every source in the directory
  at once, whatever library produced it.
- **Two processing batches.** `/log-processing` here and `/communication:log-processing` there both read the same
  directory and each prepares only its own sources. The two trackers have different filenames, so the batches do
  not contend.

You MUST shut the stack down in strict reverse order. Every VideoSystem and every MicroControllerInterface must stop
before the shared DataLogger does.

---

## Multi-camera log processing

All cameras sharing a DataLogger write to the same log directory and the same `camera_manifest.yaml`. This simplifies
batch processing:

1. `discover_camera_data_tool` finds the manifest and names all confirmed sources (e.g., 51, 52, 53) in the
   `breakdown` a bare call reports. Listing each source's own row, carrying its camera name, log archive, video
   file, and feather output, takes `include_items=True` and `detailed=True` (see `/log-processing`)
2. `prepare_log_processing_batch_tool` creates one job per source ID (see `/log-processing` for its full
   signature)
3. Process all source IDs in a single batch for efficiency
4. Output: one feather file per camera under a `camera_timestamps/` subdirectory
   (`camera_timestamps/camera_51_timestamps.feather`, `camera_timestamps/camera_52_timestamps.feather`, etc.)

For multi-DataLogger setups, pass each DataLogger output directory as its own entry in the `log_directories` list. One
batch call can carry several, and each is prepared independently, so a separate batch per directory is not required.
Passing a parent directory that spans several DataLogger outputs is rejected rather than merged. Preparation fails that
entry and returns it under `failed_directories`, paired with the error explaining which rule it broke. The
`axvs process` CLI reports the equivalent ValueError through its console, either "Each DataLogger output directory must
be prepared and processed on its own invocation" or a manifest-count error when the tree holds several
`camera_manifest.yaml` files.

---

## Cross-camera frame statistics comparison

After processing, use `analyze_camera_frame_statistics_tool` with all camera feather files (pass the `timestamps_file`
paths from a `discover_camera_data_tool` call carrying `include_items=True` and `detailed=True`) and compare:

- **Estimated FPS**: All cameras should match the configured rate. A camera with lower FPS than others
  indicates an interface or encoding bottleneck on that specific channel.
- **Timing jitter (std_us)**: Identifies which camera has the worst jitter. High jitter on one camera
  with low jitter on others points to a per-camera issue (cable, hub port, GenICam config).
- **Drop rate**: Compare `drop_rate_percent` across cameras to identify bandwidth bottlenecks. If all
  cameras drop simultaneously, the issue is system-wide (disk I/O, CPU, GPU saturation).
- **Correlated drops**: Check if drops occur at the same `frame_index` ranges across cameras. Correlated
  drops indicate system-level events, and uncorrelated drops indicate per-camera issues.
- **Start synchronization**: Compare `first_timestamp_us` across cameras. The delta between the earliest
  and latest first timestamps measures acquisition start synchronization quality.

---

## Quick-start scenarios

### Single USB camera, first test

1. `/video-mcp-environment-setup`: verify MCP connectivity (if first session)
2. `/camera-setup`: `list_cameras_tool` → `start_video_session_tool` → test → `stop_video_session_tool`
3. `/post-recording`: verify video and archives
4. Done (skip processing for quick test)

### Single Harvesters camera, production recording

1. `/camera-setup`: configure GenICam nodes, test with MCP session
2. `/camera-interface`: write VideoSystem code with production encoding parameters
3. `/post-recording`: verify video and archives
4. `/log-processing`: extract timestamps
5. `/log-processing-results`: analyze frame quality

### Multi-camera rig, behavioral experiment

1. `/camera-setup`: discover all cameras, configure GenICam nodes individually
2. `/pipeline`: plan system IDs and DataLogger topology
3. `/camera-interface`: write multi-camera code following the coordinated lifecycle pattern
4. `/post-recording`: verify all videos and archives
5. `/log-processing`: batch process all source IDs together
6. `/log-processing-results`: cross-camera comparison

### Cameras and microcontrollers in one session

1. `/communication:pipeline`: plan the controller IDs and confirm the shared DataLogger topology
2. `/pipeline`: allocate camera system IDs against the controller IDs already taken
3. `/camera-interface` and `/communication:microcontroller-interface`: write both sides against one logger
4. `/post-recording`: assemble once, then verify the camera outputs
5. `/log-processing` and `/communication:log-processing`: one batch per library over the same directory
6. `/log-processing-results`: frame statistics for the camera sources

---

## Related skills

| Skill                          | Relationship                                                       |
|--------------------------------|--------------------------------------------------------------------|
| `/video-mcp-environment-setup` | Phase 1: environment verification                                  |
| `/camera-setup`                | Phase 2-3: MCP-based discovery, testing, and recording             |
| `/camera-interface`            | Phase 3: code-based integration, and owns encoding selection       |
| `/post-recording`              | Phase 4: output verification and archive assembly                  |
| `/cli-reference`               | Reference: the `axvs` commands behind the interactive testing path |
| `/log-input-format`            | Reference: archive format for troubleshooting                      |
| `/log-processing`              | Phase 5: timestamp extraction, and owns batch resource planning    |
| `/log-processing-results`      | Phase 6: frame statistics and quality analysis                     |
| `/communication:pipeline`      | Peer: the microcontroller side of a shared-DataLogger recording    |

---

## Verification checklist

```text
Pipeline Orchestration, tool-settled (call `check_runtime_requirements_tool`, `list_cameras_tool`,
`get_batch_status_overview_tool`, and `analyze_camera_frame_statistics_tool`):
- [ ] Environment verified (MCP server connected, FFMPEG/GPU/CTI checked)
- [ ] Camera(s) discovered and configuration validated
- [ ] Log processing completed (all source IDs processed)
- [ ] Frame statistics analyzed for all cameras

Pipeline Orchestration, reader-judged:
- [ ] Interface decision made (MCP vs code, single vs multi-camera)
- [ ] System IDs allocated (unique per DataLogger, 51-100 by plugin convention, or the rig's existing
      allocation confirmed with the user)
- [ ] Source IDs checked against every non-camera source on the same DataLogger (if cross-system)
- [ ] DataLogger topology decided (single vs multiple)
- [ ] Encoding parameters selected for use case
- [ ] Recording session completed (all cameras started and stopped in order)
- [ ] Post-recording verification passed (video + archives)
- [ ] Cross-camera comparison performed (if multi-camera)
```
