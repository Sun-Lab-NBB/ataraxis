---
name: post-recording
description: >-
  Guides post-recording verification and handoff: log archive assembly, video file validation, output
  completeness checks, and readiness assessment for downstream log processing. Use after stopping a video
  session to verify outputs before processing frame timestamps.
user-invocable: true
---

# Post-recording

Guides the steps between stopping a recording session and starting log processing. Covers archive
assembly, video validation, output completeness checks, and handoff conditions.

---

## Scope

**Covers:**
- Log archive assembly from raw `.npy` files to `.npz` archives
- Video file validation (existence, size, metadata inspection)
- Output directory completeness verification
- Video quality assessment guidance
- Correlating video frame counts with log archive message counts
- Handoff conditions to `/log-processing`
- Troubleshooting assembly failures, missing archives, and corrupt video

**Does not cover:**
- Camera discovery or interactive session management (see `/camera-setup`)
- Writing VideoSystem integration code (see `/camera-interface`)
- Log processing workflow or batch operations (see `/log-processing`)
- Input archive format details or source ID semantics (see `/log-input-format`)
- Output feather files or frame statistics analysis (see `/log-processing-results`)
- MCP server connectivity (see `/mcp-environment-setup`)

**Handoff rules:** If the user asks about archive internal format or source IDs, invoke `/log-input-format`.
If ready for processing, invoke `/log-processing`. If asking about frame statistics after processing, invoke
`/log-processing-results`.

---

## Available tools

### Session stop tool

| Tool                 | Purpose                                                                       |
|----------------------|-------------------------------------------------------------------------------|
| `stop_video_session` | Stops the active session; returns video path, log directory; auto-assembles   |

The enhanced `stop_video_session` returns a dictionary:

```text
{
    "status": "stopped",
    "video_file": "/path/to/112.mp4" or null,
    "log_directory": "/path/to/mcp_video_session_data_log",
    "archives_assembled": true or false,
    "source_ids": ["112"]
}
```

When `archives_assembled` is `true`, the raw `.npy` log files have been consolidated into `.npz` archives
and the source `.npy` files have been removed. When `false`, auto-assembly failed, and you should use the
manual assembly tool.

### Archive assembly tool

| Tool                         | Purpose                                                             |
|------------------------------|---------------------------------------------------------------------|
| `assemble_log_archives_tool` | Manually assembles `.npy` files into `.npz` archives in a directory |

**Parameters:**

| Parameter          | Type   | Default    | Description                                                      |
|--------------------|--------|------------|------------------------------------------------------------------|
| `log_directory`    | `str`  | (required) | Absolute path to DataLogger output directory containing `.npy`   |
| `remove_sources`   | `bool` | `true`     | Whether to remove `.npy` files after successful assembly         |
| `verify_integrity` | `bool` | `false`    | Whether to verify archive integrity against source files first   |

### Video validation tool

| Tool                       | Purpose                                                         |
|----------------------------|-----------------------------------------------------------------|
| `validate_video_file_tool` | Inspects a video file for codec, resolution, frame count, etc.  |

**Parameters:**

| Parameter    | Type  | Default    | Description                              |
|--------------|-------|------------|------------------------------------------|
| `video_file` | `str` | (required) | Absolute path to the video file to check |

**Return structure:**

```text
{
    "file": "/path/to/112.mp4",
    "valid": true,
    "codec": "h264",
    "codec_long_name": "H.264 / AVC / MPEG-4 AVC / MPEG-4 part 10",
    "width": 600,
    "height": 400,
    "frame_count": 9000,
    "duration_seconds": 300.0,
    "bit_rate_bps": 5000000,
    "file_size_bytes": 187500000,
    "pixel_format": "yuv420p",
    "frame_rate": "30/1"
}
```

### Archive verification tool

| Tool                                   | Purpose                                                                           |
|----------------------------------------|-----------------------------------------------------------------------------------|
| `discover_recording_log_archives_tool` | Verifies archives, video files, and manifests exist via manifest-based discovery  |

**Parameters:**

| Parameter        | Type  | Default    | Description                                             |
|------------------|-------|------------|---------------------------------------------------------|
| `root_directory` | `str` | (required) | Absolute path to root directory to search for manifests |

**Note:** This tool requires `camera_manifest.yaml` files to exist in DataLogger output directories.
These manifests are written automatically by `VideoSystem.__init__()`. For each manifest source, the
tool locates the corresponding log archive, video file, and processed feather output.

---

## Post-recording workflow

You MUST follow these steps after every recording session.

1. **Stop the session** — Call `stop_video_session`. Record the returned `video_file` path and `log_directory`
   path from the response.

2. **Verify video file** — Call `validate_video_file_tool` with the `video_file` path. Confirm:
   - The file exists and has non-zero `file_size_bytes`
   - `frame_count` is greater than 0
   - `codec`, `width`, `height`, and `frame_rate` match expected session parameters
   - If `video_file` is `null`, no frames were saved (verify that `start_frame_saving` was called)

3. **Verify archive assembly** — If `archives_assembled` is `true` in the stop response, call
   `discover_recording_log_archives_tool` with the recording root to confirm archives exist for all expected
   source IDs. The discovery tool uses manifest-based routing, so it requires a `camera_manifest.yaml` in
   the log directory (written automatically by `VideoSystem.__init__()`). Each source in the response
   includes a `log_archive` field (path or `null`). If `archives_assembled` is `false`, call
   `assemble_log_archives_tool` with the `log_directory` path, then verify with the discovery tool.

4. **Cross-reference frame counts** — Compare the video `frame_count` from `validate_video_file_tool` with the
   archive message count from the discovery tool. These should be approximately equal (within 1-2 frames due to
   pipeline buffering). Large discrepancies indicate data loss.

5. **Assess readiness** — Run through the handoff checklist below. When all conditions are met, invoke
   `/log-processing` to begin timestamp extraction.

---

## Manual archive assembly

Use `assemble_log_archives_tool` when:
- The `stop_video_session` response shows `archives_assembled: false`
- Processing log directories from code-based sessions that called `logger.stop()` without assembly
- Recovering from partial session failures
- Assembling archives from sessions run via the `axvs run` CLI that were interrupted before assembly

After calling the tool, verify the result with `discover_recording_log_archives_tool` to confirm all expected
source IDs have corresponding `.npz` archives.

**Note:** `discover_recording_log_archives_tool` requires a `camera_manifest.yaml` in the log directory.
For MCP and code-based sessions using the current library version, this manifest is written automatically.
For legacy sessions without manifests, use `write_camera_manifest_tool` (see `/camera-setup`) to
retroactively register camera sources before running discovery.

---

## Video quality assessment

### Video property checks

| Property    | Good                          | Concerning              | Action                                          |
|-------------|-------------------------------|-------------------------|-------------------------------------------------|
| Frame count | Within 1% of `fps * duration` | > 1% deficit            | Check frame drops via `/log-processing-results` |
| File size   | Proportional to duration      | Zero or very small      | Re-record; check encoder configuration          |
| Codec       | Matches configured encoder    | Unexpected codec        | Verify encoder parameters in session start      |
| Resolution  | Matches camera configuration  | Different from expected | Check `width`/`height` parameters               |
| FPS         | Matches configured frame rate | Significantly lower     | Check encoder throughput and speed preset       |

### Correlating video metadata with log data

- Video `frame_count` should approximate the number of frame messages in the log archive minus the onset
  message (i.e., `archive_frame_messages - 1`).
- Video `duration_seconds` should match `(last_timestamp - first_timestamp)` from processed timestamps.
- These cross-checks can only be fully validated after log processing completes via `/log-processing-results`.
  At this stage, use the archive message count from `discover_recording_log_archives_tool` for a rough
  comparison.
- Processed output (feather files and tracker) is written to a `camera_data/` subdirectory under the
  output directory, not directly into the log directory.

---

## Handoff to log processing

All the following conditions must be true before invoking `/log-processing`:

```text
Post-Recording Readiness:
- [ ] Video session stopped (no active session)
- [ ] Video file exists and has non-zero size (or output_directory was None)
- [ ] Archives assembled (.npz files present in DataLogger output directory)
- [ ] No raw .npy files remain in the log directory
- [ ] All expected source IDs have corresponding archives
- [ ] Archive naming matches {source_id}_log.npz pattern
```

---

## Troubleshooting

| Symptom                                  | Likely Cause                             | Resolution                                     |
|------------------------------------------|------------------------------------------|------------------------------------------------|
| No video file in output directory        | Saving was never started                 | Verify `start_frame_saving` was called         |
| Video file is 0 bytes                    | FFMPEG encoding failed silently          | Check FFMPEG installation; re-record           |
| No `.npz` archives after stopping        | Auto-assembly failed or nothing logged   | Call `assemble_log_archives_tool` manually     |
| Assembly produces empty archives         | No frame messages were logged            | Verify `start_frame_saving` was called         |
| Raw `.npy` files remain after assembly   | Assembly ran with `remove_sources=false` | Re-run with `remove_sources=true`              |
| Frame count mismatch (video vs archive)  | Buffer flush timing or interruption      | 1-2 frames normal; large gaps indicate loss    |
| `validate_video_file_tool` returns error | File corrupt or ffprobe unavailable      | Check FFMPEG installation; re-record if needed |
| MCP tools unavailable                    | Server not running                       | Invoke `/mcp-environment-setup`                |

---

## Related skills

| Skill                     | Relationship                                              |
|---------------------------|-----------------------------------------------------------|
| `/camera-setup`           | Upstream: MCP session management that produces recordings |
| `/camera-interface`       | Upstream: VideoSystem code that produces recordings       |
| `/log-input-format`       | Reference: archive format and source ID semantics         |
| `/log-processing`         | Downstream: processes archives into frame timestamps      |
| `/log-processing-results` | Downstream: analyzes processed frame statistics           |
| `/pipeline`               | Context: end-to-end orchestration including this phase    |
| `/mcp-environment-setup`  | Prerequisite: MCP server connectivity for tool access     |

---

## Verification checklist

```text
Post-Recording Verification:
- [ ] Video session stopped via stop_video_session
- [ ] Video file validated via validate_video_file_tool (codec, resolution, frame count, FPS)
- [ ] Log archives assembled (.npz files present in DataLogger output directory)
- [ ] All expected source IDs have corresponding archives
- [ ] No raw .npy files remain in log directory
- [ ] Frame count cross-referenced between video metadata and archive message count
- [ ] Handoff conditions met for /log-processing
```
