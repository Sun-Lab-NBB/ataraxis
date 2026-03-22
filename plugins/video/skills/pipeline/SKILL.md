---
name: pipeline
description: >-
  End-to-end orchestration guide for the ataraxis-video-system recording and analysis pipeline. Covers
  canonical phase ordering with handoff conditions, multi-camera planning with system ID allocation and
  DataLogger topology, and decision trees for interface, encoding, and processing configuration. Use when
  planning a full recording workflow, setting up multi-camera rigs, or deciding between MCP and code.
user-invocable: true
---

# Pipeline

End-to-end orchestration reference for camera recording and data analysis. Covers single-camera and
multi-camera setups, phase ordering, handoff conditions, and decision guidance.

---

## Scope

**Covers:**
- Canonical pipeline phase ordering with handoff conditions
- Decision trees for interface, encoding, and processing configuration
- Multi-camera planning: system ID allocation, DataLogger topology, coordinated lifecycle
- Multi-camera log processing and cross-camera frame statistics comparison
- MCP vs code decision guidance
- Quick-start references for common scenarios

**Does not cover:**
- Detailed tool usage for any individual phase (see phase-specific skills)
- MCP server connectivity (see `/mcp-environment-setup`)

**Handoff rules:** This skill dispatches to phase-specific skills at each stage. Always invoke the relevant
skill for detailed tool usage, parameter reference, and troubleshooting.

---

## Pipeline phases

```text
Environment    Camera         Recording    Post-         Log            Results
Setup       →  Discovery   →            →  Recording  →  Processing  →  Analysis
    |              |              |            |              |              |
/mcp-env-    /camera-setup  /camera-setup /post-       /log-          /log-processing
 setup                      or /camera-    recording    processing     -results
                             interface
```

### Phase 1: Environment setup

- **Skill:** `/mcp-environment-setup`
- **Actions:** Verify MCP server connectivity, check `axvs` command availability, verify Python version
- **Handoff condition:** MCP tools accessible; `check_runtime_requirements` returns OK for all needed
  components
- **Skip condition:** MCP already verified in this session

### Phase 2: Camera discovery and configuration

- **Skill:** `/camera-setup`
- **Actions:** `check_runtime_requirements`, `list_cameras`, configure CTI if Harvesters, inspect and
  configure GenICam nodes
- **Handoff condition:** Camera(s) discoverable, encoding requirements met (FFMPEG OK, GPU if needed)
- **Decision point:** Single camera vs multi-camera (see multi-camera planning below)

### Phase 3: Recording session

- **Skill:** `/camera-setup` (MCP) or `/camera-interface` (code)
- **MCP path:** `start_video_session` → `start_frame_saving` → `stop_frame_saving` → `stop_video_session`
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
Does the camera support GenTL (GeniCam Transport Layer)?
  YES → Harvesters (preferred interface; provides GenICam node control)
  NO  → Is the camera a USB webcam or consumer device?
    YES → OpenCV
    NO  → Is this a test or development scenario without hardware?
      YES → Mock
      NO  → Check camera vendor documentation for GenTL support
```

### MCP vs code

| Scenario                                          | Recommendation                    |
|---------------------------------------------------|-----------------------------------|
| Single camera, interactive testing or exploration | MCP via `/camera-setup`           |
| Single camera, production with custom encoding    | Code via `/camera-interface`      |
| Multi-camera simultaneous recording               | Code via `/camera-interface`      |
| Log processing (any scenario)                     | MCP via `/log-processing`         |
| Results analysis (any scenario)                   | MCP via `/log-processing-results` |

MCP supports only one active video session at a time. Multi-camera recording requires Python code.

### Encoding selection

| Use Case                        | Encoder | Preset      | Pixel Format | QP    | GPU |
|---------------------------------|---------|-------------|--------------|-------|-----|
| Interactive testing             | H264    | FAST (3)    | YUV420       | 15    | -1  |
| Scientific imaging (high-speed) | H265    | SLOWEST (7) | YUV444       | 0-5   | 0   |
| Behavioral video (color)        | H265    | SLOW (5)    | YUV420       | 15-20 | 0   |
| Archival (storage-sensitive)    | H265    | SLOWER (6)  | YUV420       | 20-25 | 0   |
| Multi-camera rig (bandwidth)    | H265    | FAST (3)    | YUV420       | 15    | 0   |

These are healthy starting points. Actual parameters must be fine-tuned by the end user for their specific
camera, scene content, and throughput requirements.

See `/camera-interface` for detailed encoding guidance and FFMPEG error interpretation. See `/camera-setup`
for MCP encoding parameter reference.

---

## Multi-camera planning

### System ID allocation

| Range  | Assignment                   | Notes                                                        |
|--------|------------------------------|--------------------------------------------------------------|
| 51-100 | Camera VideoSystem instances | One unique ID per camera; advised range for all camera code  |
| 111    | CLI (`axvs run`)             | Fixed; interactive testing only                              |
| 112    | MCP server sessions          | Fixed; agent-driven testing only                             |

All other IDs are used by other production assets in the broader system. Camera code should stay within
the 51-100 band. Allocate camera IDs sequentially starting at 51 (e.g., 51, 52, 53 for a 3-camera rig).

### DataLogger topology

A single shared DataLogger is the preferred topology for all use cases:

```text
DataLogger(instance_name="session")
  ├── VideoSystem(system_id=51) → 051_log.npz
  ├── VideoSystem(system_id=52) → 052_log.npz
  └── VideoSystem(system_id=53) → 053_log.npz
```

All cameras share one log directory, all timestamps are correlated, one `assemble_log_archives` call
consolidates everything, and one processing batch covers all source IDs.

Multiple DataLoggers should only be used if a single logger cannot handle the load, leading to excessive
buffering. This is extremely rare in practice. When it does occur, each DataLogger creates a separate
output directory that must be assembled and processed independently, and cross-camera timestamp comparison
requires merging data from separate directories.

### Coordinated lifecycle

The ordering of initialization and shutdown is critical for multi-camera setups:

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

session_directory = Path("/path/to/session")

# Starts the shared DataLogger first.
logger = DataLogger(output_directory=session_directory, instance_name="session")
logger.start()

# Initializes and starts each camera with a unique system ID.
cameras: list[VideoSystem] = []
for camera_id, camera_index in [(51, 0), (52, 1), (53, 2)]:
    camera = VideoSystem(
        system_id=np.uint8(camera_id),
        data_logger=logger,
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

## Multi-camera log processing

All cameras sharing a DataLogger write to the same log directory. This simplifies batch processing:

1. `discover_recording_log_archives_tool` finds all source IDs (e.g., 051, 052, 053) in one directory
2. `prepare_log_processing_batch_tool` creates one job per source ID
3. Process all source IDs in a single batch for efficiency
4. Output: one feather file per camera (`camera_051_timestamps.feather`, `camera_052_timestamps.feather`,
   etc.)

For multi-DataLogger setups, process each DataLogger output directory as a separate batch.

---

## Cross-camera frame statistics comparison

After processing, use `analyze_camera_frame_statistics_tool` on each camera's feather file and compare:

- **Estimated FPS** — All cameras should match the configured rate. A camera with lower FPS than others
  indicates an interface or encoding bottleneck on that specific channel.
- **Timing jitter (std_us)** — Identifies which camera has the worst jitter. High jitter on one camera
  with low jitter on others points to a per-camera issue (cable, hub port, GenICam config).
- **Drop rate** — Compare `drop_rate_percent` across cameras to identify bandwidth bottlenecks. If all
  cameras drop simultaneously, the issue is system-wide (disk I/O, CPU, GPU saturation).
- **Correlated drops** — Check if drops occur at the same `frame_index` ranges across cameras. Correlated
  drops indicate system-level events; uncorrelated drops indicate per-camera issues.
- **Start synchronization** — Compare `first_timestamp_us` across cameras. The delta between the earliest
  and latest first timestamps measures acquisition start synchronization quality.

---

## Quick-start scenarios

### Single USB camera, first test

1. `/mcp-environment-setup` — verify MCP connectivity (if first session)
2. `/camera-setup` — `list_cameras` → `start_video_session` → test → `stop_video_session`
3. `/post-recording` — verify video and archives
4. Done (skip processing for quick test)

### Single Harvesters camera, production recording

1. `/camera-setup` — configure GenICam nodes, test with MCP session
2. `/camera-interface` — write VideoSystem code with production encoding parameters
3. `/post-recording` — verify video and archives
4. `/log-processing` — extract timestamps
5. `/log-processing-results` — analyze frame quality

### Multi-camera rig, behavioral experiment

1. `/camera-setup` — discover all cameras, configure GenICam nodes individually
2. `/pipeline` — plan system IDs and DataLogger topology
3. `/camera-interface` — write multi-camera code following the coordinated lifecycle pattern
4. `/post-recording` — verify all videos and archives
5. `/log-processing` — batch process all source IDs together
6. `/log-processing-results` — cross-camera comparison

---

## Related skills

| Skill                     | Role                                                   |
|---------------------------|--------------------------------------------------------|
| `/mcp-environment-setup`  | Phase 1: environment verification                      |
| `/camera-setup`           | Phase 2-3: MCP-based discovery, testing, and recording |
| `/camera-interface`       | Phase 3: code-based VideoSystem integration            |
| `/post-recording`         | Phase 4: output verification and archive assembly      |
| `/log-input-format`       | Reference: archive format for troubleshooting          |
| `/log-processing`         | Phase 5: timestamp extraction                          |
| `/log-processing-results` | Phase 6: frame statistics and quality analysis         |

---

## Verification checklist

```text
Pipeline Orchestration:
- [ ] Environment verified (MCP server connected, FFMPEG/GPU/CTI checked)
- [ ] Camera(s) discovered and configuration validated
- [ ] Interface decision made (MCP vs code, single vs multi-camera)
- [ ] System IDs allocated (unique per camera, 51-100 range)
- [ ] DataLogger topology decided (single vs multiple)
- [ ] Encoding parameters selected for use case
- [ ] Recording session completed (all cameras started and stopped in order)
- [ ] Post-recording verification passed (video + archives)
- [ ] Log processing completed (all source IDs processed)
- [ ] Frame statistics analyzed for all cameras
- [ ] Cross-camera comparison performed (if multi-camera)
```
