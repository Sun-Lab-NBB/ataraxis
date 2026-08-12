---
name: log-processing-results
description: >-
  Documents log processing output data formats, feather file discovery, frame statistics analysis, and
  interpretation guidance. Use when evaluating log processing results, when the user asks about frame timing
  data, frame drops, or camera acquisition quality.
user-invocable: false
---

# Log processing results

Documents the camera timestamp extraction output data format, covering feather file discovery, schema reference, frame
statistics analysis, and interpretation guidance for discussing results with users.

---

## Scope

**Covers:**
- Output data format: feather file schema, column names, and dtypes
- Directory structure and naming conventions
- Feather file discovery via MCP tool
- Frame statistics analysis via MCP tool
- Inter-frame timing interpretation
- Frame drop detection methodology and interpretation
- Processing status cross-referencing
- Output completeness verification

**Does not cover:**
- Input archive format, source ID semantics, or DataLogger output (see `/log-input-format`)
- Processing workflow, batch operations, or status monitoring (see `/log-processing`)
- The `discover_camera_data_tool` parameters and full return structure (see `/log-processing`)
- Camera hardware setup or configuration (see `/camera-setup`)
- Encoding parameter selection, which the remedies below name (see `/camera-interface`)
- Cross-camera comparison of these statistics on a multi-camera rig (see `/pipeline`)
- MCP server connectivity issues (see `/video-mcp-environment-setup`)

**Handoff rules:** If a source carries no `timestamps_file` or a statistic looks wrong because the job never succeeded,
invoke `/log-processing` to inspect the tracker and retry. If the remedy is an encoding or acquisition parameter change,
invoke `/camera-interface` for the code path or `/camera-setup` for the MCP path. If several cameras must be compared
against one another, invoke `/pipeline`. If MCP tools are unavailable, invoke `/video-mcp-environment-setup`.

---

## Available tools

### Discovery tool

| Tool                        | Purpose                                                                            |
|-----------------------------|------------------------------------------------------------------------------------|
| `discover_camera_data_tool` | Discovers manifests, log archives, video files, and feather files under a root dir |

`/log-processing` owns this tool's parameters and full return structure. This skill covers only the one field the output
side reads: each source entry's `timestamps_file`, which carries the path to that camera's feather file. The field is
reported only when `detailed=True` accompanies a listing request, and a source that has not been processed carries no
`timestamps_file` key at all. See the processing completeness section below for its interpretation.

### Analysis tool

| Tool                                   | Purpose                                                                |
|----------------------------------------|------------------------------------------------------------------------|
| `analyze_camera_frame_statistics_tool` | Computes frame timing stats and detects frame drops from feather files |

**Parameters:**

| Parameter            | Type        | Default    | Description                                                       |
|----------------------|-------------|------------|-------------------------------------------------------------------|
| `feather_files`      | `list[str]` | (required) | Absolute paths to `camera_*_timestamps.feather` files             |
| `drop_threshold_us`  | `int`       | `0`        | Gap threshold in microseconds, 0 for auto-detection (1.5x median) |
| `max_drop_locations` | `int`       | `50`       | Maximum number of frame drop locations to include per file        |

Returns a dictionary with a `results` list (one entry per file, each containing `file`, `basic_stats`,
`inter_frame_timing`, and `frame_drop_analysis` keys) and a `total_files` count. Files that cannot be read produce an
entry with `file` and `error` keys instead of statistics.

---

## Recommended query order

1. **`discover_camera_data_tool`**: Find all manifests, archives, video files, and feather files under the root
   directory. Pass `include_items=True` and `detailed=True`, since a bare call lists no source and the feather paths
   this skill consumes are added by detail.
2. **`analyze_camera_frame_statistics_tool`**: Compute statistics for discovered feather files. Pass the
   `timestamps_file` paths from that response as the `feather_files` list.

---

## Output data reference

### Directory structure

The processing pipeline writes all output under a `camera_timestamps/` subdirectory within the output directory provided
by the user:

```text
{output_directory}/
└── camera_timestamps/                               # Processing output subdirectory
    ├── camera_processing_tracker.yaml              # ProcessingTracker state file (job lifecycle)
    ├── camera_{source_id}_timestamps.feather       # Per-camera output (one per source ID)
    ├── camera_{source_id}_timestamps.feather
    └── ...
```

### Feather file schema

Each feather file is a Polars DataFrame serialized in the Feather IPC format with a single column:

| Column          | Dtype  | Description                                                     |
|-----------------|--------|-----------------------------------------------------------------|
| `frame_time_us` | UInt64 | Frame acquisition timestamp in microseconds since the UTC epoch |

Rows are ordered chronologically. Each row corresponds to one acquired frame.

### Naming convention

Files follow the pattern `camera_{source_id}_timestamps.feather` where `source_id` is the unpadded string form of the
integer system ID from the DataLogger archive (e.g., `camera_51_timestamps.feather` for system_id=51,
`camera_112_timestamps.feather` for an MCP session).

### ProcessingTracker file

The `camera_processing_tracker.yaml` file tracks job lifecycle per output directory (inside `camera_timestamps/`). Each
job has:
- **job_id:** Unique hexadecimal identifier
- **job_name:** Always `camera_timestamp_extraction`
- **specifier:** The source ID string
- **status:** `SCHEDULED`, `RUNNING`, `SUCCEEDED`, or `FAILED`
- **started_at / completed_at:** Microsecond UTC timestamps (when available)
- **error_message:** Error details (when status is `FAILED`)
- **executor_id:** Identifier of the executor that ran the job (when set). `get_log_processing_status_tool` and
  `get_log_processing_timing_tool` (see `/log-processing`) surface it in their per-job entries, while
  `get_batch_status_overview_tool` does not report it

---

## Frame statistics reference

### basic_stats

| Field                | Type    | Description                                              |
|----------------------|---------|----------------------------------------------------------|
| `total_frames`       | `int`   | Total number of acquired frames                          |
| `first_timestamp_us` | `int`   | First frame timestamp (microseconds since the UTC epoch) |
| `last_timestamp_us`  | `int`   | Last frame timestamp (microseconds since the UTC epoch)  |
| `duration_us`        | `int`   | Total recording duration in microseconds                 |
| `duration_seconds`   | `float` | Total recording duration in seconds                      |
| `estimated_fps`      | `float` | Estimated frame rate, `(frames - 1) / duration`          |

### inter_frame_timing

| Field       | Type    | Description                                  |
|-------------|---------|----------------------------------------------|
| `mean_us`   | `float` | Mean inter-frame interval in microseconds    |
| `median_us` | `float` | Median inter-frame interval in microseconds  |
| `std_us`    | `float` | Standard deviation in microseconds           |
| `min_us`    | `int`   | Minimum inter-frame interval                 |
| `max_us`    | `int`   | Maximum inter-frame interval                 |
| `mean_ms`   | `float` | Mean inter-frame interval in milliseconds    |
| `median_ms` | `float` | Median inter-frame interval in milliseconds  |
| `std_ms`    | `float` | Standard deviation in milliseconds           |
| `min_ms`    | `float` | Minimum inter-frame interval in milliseconds |
| `max_ms`    | `float` | Maximum inter-frame interval in milliseconds |

### frame_drop_analysis

| Field                            | Type    | Description                                        |
|----------------------------------|---------|----------------------------------------------------|
| `threshold_us`                   | `float` | Gap threshold used for drop detection              |
| `threshold_source`               | `str`   | `"auto_1.5x_median"` or `"user_specified"`         |
| `total_gaps_detected`            | `int`   | Number of inter-frame gaps exceeding the threshold |
| `jitter_compensated_gaps`        | `int`   | Detected gaps the following interval repaid        |
| `total_estimated_dropped_frames` | `int`   | Estimated total frames lost across all gaps        |
| `drop_rate_percent`              | `float` | Percentage of expected frames that were dropped    |
| `longest_gap_us`                 | `int`   | Longest detected gap in microseconds               |
| `longest_gap_ms`                 | `float` | Longest detected gap in milliseconds               |
| `drop_locations`                 | `list`  | Per-gap details (capped by `max_drop_locations`)   |
| `drop_locations_truncated`       | `bool`  | True if more drops exist than `max_drop_locations` |

Each entry in `drop_locations`:

| Field                   | Type    | Description                                 |
|-------------------------|---------|---------------------------------------------|
| `frame_index`           | `int`   | Index of the frame preceding the gap        |
| `gap_us`                | `int`   | Gap duration in microseconds                |
| `gap_ms`                | `float` | Gap duration in milliseconds                |
| `estimated_frames_lost` | `int`   | Estimated number of frames lost in this gap |

**Auto-detection algorithm:** When `drop_threshold_us=0`, the threshold is computed as 1.5x the median inter-frame
interval, which sits below the two-interval span a single lost frame produces. Gaps exceeding the threshold are
classified as frame drops and counted under `total_gaps_detected`.

Each detected gap is then netted against the interval that follows it. A frame whose timestamp arrives late stretches
its own interval and shortens the next one by the same amount. Whatever the following interval falls short of a full
interval is subtracted from the gap before any frame is charged against it. The remainder is divided by the median
interval, rounded, reduced by one, and clamped at zero, because a span of N median intervals accounts for N-1 lost
frames. A recording's last interval has no successor to repay it, so it is netted against a full interval and carries
no compensation.

A gap the netting resolves to zero is jitter rather than loss. It counts under `jitter_compensated_gaps`, contributes
nothing to `total_estimated_dropped_frames`, and still appears in `drop_locations` with an `estimated_frames_lost` of
zero. A file whose `total_gaps_detected` and `jitter_compensated_gaps` are equal lost no frames at all.

**Edge cases:** When `total_frames == 0`, only `basic_stats.total_frames` is returned and `inter_frame_timing` /
`frame_drop_analysis` are empty `{}`. When `total_frames == 1`, `basic_stats` is fully populated but `duration_us`,
`duration_seconds`, and `estimated_fps` are `0`, and the timing and drop sections are again empty `{}`. Whenever
`duration_us` is zero, including a degenerate capture whose timestamps are identical, `estimated_fps` is `0.0`, which is
a "cannot compute" sentinel rather than a measured 0 fps. Check `total_frames >= 2` before indexing into
`inter_frame_timing` or `frame_drop_analysis` to avoid `KeyError` s.

---

## Interpretation guidance

### What "good" frame timing looks like

- **`estimated_fps`** should be close to the configured camera frame rate. A significant shortfall
  (e.g., configured 30fps but estimated 25fps) indicates the camera or encoder cannot keep up.
- **`std_us`** should be small relative to `mean_us`. A coefficient of variation (std/mean) under 5%
  indicates stable timing.
- **`min_us`** and **`max_us`** should bracket `mean_us` tightly. A `max_us` many times larger than
  `mean_us` signals occasional long gaps even if `std_us` appears acceptable.

### Frame drop severity

| Drop Rate | Severity | Typical Impact                                        |
|-----------|----------|-------------------------------------------------------|
| 0%        | None     | Perfect acquisition                                   |
| < 0.1%    | Minimal  | Negligible impact on most analyses                    |
| 0.1%-1%   | Moderate | May affect high-precision timing analyses             |
| 1%-5%     | High     | Likely to affect frame-accurate behavioral scoring    |
| > 5%      | Severe   | Recording quality compromised, investigate root cause |

### Common frame drop patterns

| Pattern                   | Characteristic                             | Likely Cause                              |
|---------------------------|--------------------------------------------|-------------------------------------------|
| Increasing gaps over time | Drops worsen as recording progresses       | Thermal throttling (camera or GPU)        |
| Random isolated drops     | Sporadic single-frame gaps throughout      | Interface bandwidth contention (USB/GigE) |
| Periodic drops            | Drops at regular intervals                 | Competing device or process scheduling    |
| Burst drops               | Clusters of consecutive gaps               | Disk write stalls or buffer overflow      |
| Single large gap          | One very long gap in otherwise stable data | System interruption (sleep, update)       |

### Relating results to camera settings

Encoding cost depends on frame content: high-motion or high-detail frames are expensive to encode, while static or
low-contrast frames are cheap. Drops that correlate with scene changes (e.g., animal entering the field of view)
indicate the encoder cannot handle peak complexity at the current preset.

- If `estimated_fps` << configured frame rate: the camera or encoder cannot sustain the configured rate.
  Reduce resolution, frame rate, or switch from CPU to GPU encoding.
- If `std_us` is high but `estimated_fps` is close to target: jitter is present but throughput is
  maintained. Check cable quality, hub topology (USB), or network configuration (GigE).
- If drops appear only in GPU-encoded sessions: the NVENC encoder may be saturated. Reduce quantization
  quality or use a faster speed preset.
- If drops correlate with high-motion periods: use a faster `encoder_speed_preset` to handle peak
  encoding load, or increase `quantization_parameter` to reduce per-frame encoding cost.

---

## Processing completeness

Read each source entry's `timestamps_file` from a discovery response taken with `include_items=True` and
`detailed=True`:

| `timestamps_file` state | Meaning                                                      |
|-------------------------|--------------------------------------------------------------|
| Path present            | Processing succeeded for this source, feather file available |
| Key absent              | Not yet processed, processing failed, or output was cleaned  |

To determine per-job status (SCHEDULED, RUNNING, SUCCEEDED, FAILED), read the `camera_processing_tracker.yaml` file
through `get_batch_status_overview_tool` with `include_items=True` and `detailed=True`, which together add each
directory's `jobs` entries. That tool reports one entry per `camera_timestamps/` subdirectory rather than per
DataLogger log directory, so its `output_directory` values will not match the `log_directory` values
`discover_camera_data_tool` returns.

---

## Related skills

| Skill                          | Relationship                                                      |
|--------------------------------|-------------------------------------------------------------------|
| `/video-mcp-environment-setup` | Prerequisite: MCP server connectivity for tool access             |
| `/camera-setup`                | Upstream: MCP discovery tools that locate archives and recordings |
| `/camera-interface`            | Owns the encoding parameters the remedies here name               |
| `/post-recording`              | Upstream: verifies session outputs before processing              |
| `/cli-reference`               | Reference: `axvs process` writes the same feather output          |
| `/log-input-format`            | Reference: input archive format and source ID semantics           |
| `/log-processing`              | Owns the processing workflow and the discovery tool contract      |
| `/pipeline`                    | Owns cross-camera comparison of these statistics                  |

---

## Verification checklist

```text
Log Processing Output Completeness, tool-settled (run `discover_camera_data_tool`,
`analyze_camera_frame_statistics_tool`, and `get_batch_status_overview_tool`):
- [ ] Feather files discovered via a `discover_camera_data_tool` call carrying `include_items=True` and `detailed=True`
- [ ] Every expected source ID carries a `timestamps_file` in that listing
- [ ] Processing status verified for all directories
- [ ] Frame statistics analyzed via `analyze_camera_frame_statistics_tool` for each camera

Log Processing Output Completeness, reader-judged:
- [ ] Estimated FPS matches expected camera frame rate
- [ ] Frame drop rate is within acceptable range for the experiment
- [ ] No unexplained large gaps in inter-frame timing
```
