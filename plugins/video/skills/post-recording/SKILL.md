---
name: post-recording
description: >-
  Guides post-recording verification and handoff: log archive assembly, video file validation, output
  completeness checks, and readiness assessment for downstream log processing. Use after stopping a video
  session to verify outputs before processing frame timestamps.
user-invocable: false
---

# Post-recording

Guides the steps between stopping a recording session and starting log processing.

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
- Log processing workflow, batch operations, or the discovery tool contract (see `/log-processing`)
- Input archive format details or source ID semantics (see `/log-input-format`)
- Output feather files or frame statistics analysis (see `/log-processing-results`)
- The `axvs run` session this skill may be recovering (see `/cli-reference`)
- MCP server connectivity (see `/video-mcp-environment-setup`)

**Handoff rules:** If the user asks about archive internal format or source IDs, invoke `/log-input-format`. If ready
for processing, invoke `/log-processing`. If asking about frame statistics after processing, invoke
`/log-processing-results`. If verification fails in a way that calls for re-recording, invoke `/camera-setup` for an MCP
session or `/camera-interface` for code. If MCP tools are unavailable, invoke `/video-mcp-environment-setup`.

---

## Available tools

### Session stop tool

| Tool                      | Purpose                                                                            |
|---------------------------|------------------------------------------------------------------------------------|
| `stop_video_session_tool` | Stops the active session, returns video path and log directory, assembles archives |

`/camera-setup` owns this tool's return structure. `video_file` and `log_directory` feed the two verification steps
below, and `archives_assembled` decides whether assembly already happened. When it is `true`, the raw `.npy` entries
were consolidated into `.npz` archives and removed.

A code-based session never calls this tool at all, so its archives always need the manual route.

### Archive assembly tool

| Tool                         | Purpose                                                             |
|------------------------------|---------------------------------------------------------------------|
| `assemble_log_archives_tool` | Manually assembles `.npy` files into `.npz` archives in a directory |

**Parameters:**

| Parameter          | Type   | Default    | Description                                                              |
|--------------------|--------|------------|--------------------------------------------------------------------------|
| `log_directory`    | `str`  | (required) | Absolute path to the DataLogger output directory containing `.npy` files |
| `remove_sources`   | `bool` | `true`     | Keyword-only. Remove `.npy` files after successful assembly              |
| `verify_integrity` | `bool` | `false`    | Keyword-only. Verify archive integrity against sources first             |

**Return structure:**

```text
{"status": "assembled", "directory": "/path/to/session_data_log", "archives": ["51_log.npz"],
 "source_ids": ["51"], "archive_count": 1}
```

Read `source_ids` and `archive_count` directly to confirm every expected source assembled. No follow-up discovery call
is needed for that check.

Assembly covers the `.npy` entries lying **directly inside** `log_directory` and never descends into subdirectories, so
the path must be the DataLogger output directory itself (`{instance_name}_data_log/`), never a recording root grouping
several of them. When the directory holds no `.npy` of its own but its subdirectories do, the tool returns an error
saying so instead of reporting a silently empty success. Other error dictionaries cover an absent path, a non-directory
path, and a failed assembly.

### Video validation tool

| Tool                       | Purpose                                                        |
|----------------------------|----------------------------------------------------------------|
| `validate_video_file_tool` | Inspects a video file for codec, resolution, frame count, etc. |

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

| Tool                        | Purpose                                                                          |
|-----------------------------|----------------------------------------------------------------------------------|
| `discover_camera_data_tool` | Verifies archives, video files, and manifests exist via manifest-based discovery |

`/log-processing` owns this tool's parameters and full return structure. Verification reads each source entry's
`log_archive` to confirm the archive landed, and `video_file` to pair it with the recording. Both are detail fields, so
every call this skill makes passes `include_items=True` and `detailed=True`. The tool requires a `camera_manifest.yaml`
in every DataLogger output directory, which `VideoSystem.__init__()` writes automatically.

**Caution:** `video_file` is resolved by a name-then-ID substring heuristic with path-proximity tie-breaking, not an
exact path. An absent `video_file` key means "not matched", not necessarily "not on disk", so confirm with
`validate_video_file_tool` using the path returned by the stop tool. Beware false matches when a camera name or the
zero-padded ID appears in an unrelated `.mp4` stem under the root. `timestamps_file` carries no such risk. It is
resolved by exact filename (`camera_{source_id}_timestamps.feather`) inside each discovered `camera_timestamps/`
directory, with proximity used only to choose between same-named outputs of different recordings.

---

## Post-recording workflow

You MUST follow these steps after every recording session.

1. **Stop the session**: Call `stop_video_session_tool`. Record the returned `video_file` path and `log_directory`
   path from the response.

2. **Verify video file**: Call `validate_video_file_tool` with the `video_file` path. Confirm:
   - The file exists and has non-zero `file_size_bytes`
   - `frame_count`, `duration_seconds`, and `bit_rate_bps` are each `null` when ffprobe does not report the
     corresponding field, so treat a `null` as "unknown" rather than zero. `file_size_bytes` is always
     populated, since it falls back to a filesystem stat
   - `codec`, `width`, `height`, and `frame_rate` match expected session parameters
   - A `null` `video_file` means the session had no video output directory / no saver configured (`output_directory` was
     `None` at construction). A non-null path whose validation returns `{"error": "No video stream found in file."}`,
     backed by a file of only a few hundred bytes on disk, is the signal that `start_frame_saving_tool` was never
     called. The encoder process starts with the session and always creates the `.mp4` container, and that container
     holds no encoded stream
   - An `{"error": "ffprobe failed: ..."}` response carries ffprobe's own diagnostic after the colon, so quote that
     text when reporting the failure rather than calling the file merely unreadable

3. **Verify archive assembly**: If `archives_assembled` is `true` in the stop response, call `discover_camera_data_tool`
   with the recording root, `include_items=True`, and `detailed=True` to confirm archives exist for all expected source
   IDs. Each entry of the returned `sources` list carries a `log_archive` path. If `archives_assembled` is `false`, call
   `assemble_log_archives_tool` with the `log_directory` path, then verify with the discovery tool.

4. **Confirm archive presence for cross-referencing**: `discover_camera_data_tool` confirms that each
   source's `log_archive` exists but does not compute an archive message count. The genuine frame-count
   vs. message-count cross-check (video `frame_count` from `validate_video_file_tool` against the archive
   message count) can only be performed after `/log-processing`, where the message count becomes available.
   The two counts should be approximately equal (within 1-2 frames due to pipeline buffering), and large discrepancies
   indicate data loss.

5. **Assess readiness**: Run through the Verification checklist below. When all conditions are met, invoke
   `/log-processing` to begin timestamp extraction.

---

## Manual archive assembly

Use `assemble_log_archives_tool` when:
- The `stop_video_session_tool` response shows `archives_assembled: false`
- Processing log directories from code-based sessions that called `logger.stop()` without assembly
- Recovering from partial session failures
- Recovering an `axvs run` session whose process was killed outright. An ordinary interrupt still assembles,
  because the CLI runs assembly in a `finally` block that executes on every exit path

**A directory holding both `.npy` entries and `.npz` archives is a half-assembled recording, and assembling it again is
unsafe.** Assembly overwrites an existing archive of the same source, and neither the tool nor the library function
beneath it guards against this. The destructive call therefore succeeds silently and reports `status: "assembled"`.
Check for both extensions before calling. If both are present, have the user back up the existing archives and remove
them from the log directory before retrying.

Assembly is directory-wide, not camera-wide. It groups by source ID alone and consolidates every source in the
directory, whatever library produced it, so **one** call covers a DataLogger shared with a sibling library. On a mixed
recording, confirm nobody has already assembled on the microcontroller side before calling it here. See `/pipeline` for
the shared-directory rules and `/communication:log-input-format` for the archives on the other side.

After calling the tool, verify the result with another `include_items=True`, `detailed=True` discovery call to confirm
all expected source IDs have corresponding `.npz` archives. For legacy sessions without manifests, use
`write_camera_manifest_tool` (see `/camera-setup`) to retroactively register camera sources before running discovery.

---

## Video quality assessment

### Video property checks

| Property    | Good                          | Concerning                         | Action                                          |
|-------------|-------------------------------|------------------------------------|-------------------------------------------------|
| Frame count | Within 1% of `fps * duration` | > 1% deficit                       | Check frame drops via `/log-processing-results` |
| File size   | Proportional to duration      | Zero, or under a few hundred bytes | Re-record, check encoder configuration          |
| Codec       | Matches configured encoder    | Unexpected codec                   | Verify encoder parameters in session start      |
| Resolution  | Matches camera configuration  | Different from expected            | Check `width`/`height` parameters               |
| FPS         | Matches configured frame rate | Significantly lower                | Check encoder throughput and speed preset       |

### Correlating video metadata with log data

- Video `frame_count` should approximate the number of frame messages in the log archive. The onset is a separate
  message type that no frame message count includes, so no adjustment is applied to either figure.
- Video `duration_seconds` should match `(last_timestamp - first_timestamp) / 1e6` from processed timestamps. See
  `/log-processing-results` for the extracted column semantics and the processing output layout.

---

## Handoff to log processing

Video files are named `{system_id:03d}.mp4` (zero-padded to 3 digits, e.g. `001.mp4`, `042.mp4`), whereas log archives
use the bare integer `{source_id}_log.npz` (`1_log.npz`, `42_log.npz`). The same source ID drives both, and only the
video name is padded.

---

## Troubleshooting

| Symptom                                         | Likely cause                             | Resolution                                         |
|-------------------------------------------------|------------------------------------------|----------------------------------------------------|
| "No video stream found in file." on a tiny file | Saving was never started                 | Verify `start_frame_saving_tool` was called        |
| No video file in video output directory         | Session ran with `output_directory=None` | Re-record with a video output directory configured |
| Video file is 0 bytes                           | FFMPEG encoding failed silently          | Check FFMPEG installation, re-record               |
| No `.npz` archives after stopping               | Auto-assembly failed or nothing logged   | Call `assemble_log_archives_tool` manually         |
| Both `.npy` and `.npz` in one directory         | Half-assembled recording                 | Back up the archives first, never re-assemble      |
| Assembly produces empty archives                | No frame messages were logged            | Verify `start_frame_saving_tool` was called        |
| Raw `.npy` files remain after assembly          | Assembly ran with `remove_sources=false` | Re-run with `remove_sources=true`                  |
| Frame count mismatch (video vs archive)         | Buffer flush timing or interruption      | 1-2 frames normal, large gaps indicate loss        |
| `validate_video_file_tool` returns error        | File corrupt or ffprobe unavailable      | Check FFMPEG installation, re-record if needed     |
| MCP tools unavailable                           | Server not running                       | Invoke `/video-mcp-environment-setup`              |

A frame deficit concentrated at the **end** of a recording is a distinct case. `stop()` waits up to 10 minutes (600
seconds) for the saver queue to drain, then abandons the daemon consumer process and loses the unencoded tail, with no
error surfaced by the stop tool. It means the encoder could not keep up at shutdown, so use a faster
`encoder_speed_preset` or hardware encoding next time.

---

## Related skills

| Skill                             | Relationship                                                |
|-----------------------------------|-------------------------------------------------------------|
| `/camera-setup`                   | Upstream: MCP session management, and owns the stop tool    |
| `/camera-interface`               | Upstream: VideoSystem code that produces recordings         |
| `/cli-reference`                  | Upstream: the `axvs run` session this skill recovers        |
| `/log-input-format`               | Reference: archive format and source ID semantics           |
| `/communication:log-input-format` | Reference: the sibling archives sharing the same directory  |
| `/log-processing`                 | Downstream: processes archives, and owns the discovery tool |
| `/log-processing-results`         | Downstream: analyzes processed frame statistics             |
| `/pipeline`                       | Context: end-to-end orchestration including this phase      |
| `/video-mcp-environment-setup`    | Prerequisite: MCP server connectivity for tool access       |

---

## Verification checklist

```text
Post-Recording Verification, tool-settled (call `discover_camera_data_tool` on the recording root, with
`include_items=True` and `detailed=True`):
- [ ] Log archives assembled (.npz files present in DataLogger output directory)
- [ ] All expected source IDs have corresponding archives (frame-count vs message-count cross-check deferred to
      /log-processing)

Post-Recording Verification, reader-judged:
- [ ] Video session stopped via stop_video_session_tool
- [ ] Video file validated via validate_video_file_tool (codec, resolution, frame count, FPS)
- [ ] No raw .npy files remain in log directory
- [ ] Archive naming matches {source_id}_log.npz pattern
- [ ] Handoff conditions met for /log-processing
```
