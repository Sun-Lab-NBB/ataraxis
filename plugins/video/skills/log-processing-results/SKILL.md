---
name: log-processing-results
description: >-
  Complete reference for log processing output data formats, feather file discovery, frame statistics analysis,
  and interpretation guidance. Use when evaluating log processing results, when the user asks about frame timing
  data, frame drops, or camera acquisition quality.
user-invocable: true
---

# Log processing results

Complete output data format documentation for the camera timestamp extraction pipeline. Covers feather file
discovery, schema reference, frame statistics analysis, and interpretation guidance for discussing results
with users.

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
- Camera hardware setup or configuration (see `/camera-setup`)
- Writing VideoSystem integration code (see `/camera-interface`)
- MCP server connectivity issues (see `/mcp-environment-setup`)

---

## Available tools

### Discovery tool

| Tool                                   | Purpose                                                                 |
|----------------------------------------|-------------------------------------------------------------------------|
| `discover_processed_camera_logs_tool`  | Discovers feather files, cross-references processing status via tracker |

**Parameters:**

| Parameter        | Type  | Default    | Description                               |
|------------------|-------|------------|-------------------------------------------|
| `root_directory` | `str` | (required) | Absolute path to root directory to search |

### Analysis tool

| Tool                                   | Purpose                                                               |
|----------------------------------------|-----------------------------------------------------------------------|
| `analyze_camera_frame_statistics_tool` | Computes frame timing stats and detects frame drops from feather file |

**Parameters:**

| Parameter            | Type  | Default    | Description                                                      |
|----------------------|-------|------------|------------------------------------------------------------------|
| `feather_file`       | `str` | (required) | Absolute path to a `camera_*_timestamps.feather` file            |
| `drop_threshold_us`  | `int` | `0`        | Gap threshold in microseconds; 0 for auto-detection (2x median)  |
| `max_drop_locations` | `int` | `50`       | Maximum number of drop locations to include in the response      |

---

## Recommended query order

1. **`discover_processed_camera_logs_tool`** — Find all feather files under the root directory and check
   processing completeness per directory.
2. **`analyze_camera_frame_statistics_tool`** — Compute statistics for each camera's feather file. Call
   once per feather file of interest.

---

## Output data reference

### Directory structure

```text
{log_directory}/
├── camera_processing_tracker.yaml          # ProcessingTracker state file (job lifecycle)
├── camera_{source_id}_timestamps.feather   # Per-camera output (one per source ID)
├── camera_{source_id}_timestamps.feather
└── ...
```

### Feather file schema

Each feather file is a Polars DataFrame serialized as Feather IPC format with a single column:

| Column          | Dtype  | Description                                                    |
|-----------------|--------|----------------------------------------------------------------|
| `frame_time_us` | UInt64 | Frame acquisition timestamp in microseconds since UTC epoch    |

Rows are ordered chronologically. Each row corresponds to one acquired frame.

### Naming convention

Files follow the pattern `camera_{source_id}_timestamps.feather` where `source_id` is the zero-padded
system ID string from the DataLogger archive (e.g., `camera_051_timestamps.feather` for system_id=51,
`camera_112_timestamps.feather` for an MCP session).

### ProcessingTracker file

The `camera_processing_tracker.yaml` file tracks job lifecycle per log directory. Each job has:
- **job_id:** Unique hexadecimal identifier
- **job_name:** Always `camera_timestamp_extraction`
- **specifier:** The source ID string
- **status:** `SCHEDULED`, `RUNNING`, `SUCCEEDED`, or `FAILED`
- **started_at / completed_at:** Microsecond UTC timestamps (when available)
- **error_message:** Error details (when status is `FAILED`)

---

## Frame statistics reference

The `analyze_camera_frame_statistics_tool` returns a dictionary with three sections.

### basic_stats

| Field                | Type    | Description                                          |
|----------------------|---------|------------------------------------------------------|
| `total_frames`       | `int`   | Total number of acquired frames                      |
| `first_timestamp_us` | `int`   | First frame timestamp (microseconds since UTC epoch) |
| `last_timestamp_us`  | `int`   | Last frame timestamp (microseconds since UTC epoch)  |
| `duration_us`        | `int`   | Total recording duration in microseconds             |
| `duration_seconds`   | `float` | Total recording duration in seconds                  |
| `estimated_fps`      | `float` | Estimated frame rate: `(frames - 1) / duration`      |

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

| Field                            | Type    | Description                                                    |
|----------------------------------|---------|----------------------------------------------------------------|
| `threshold_us`                   | `float` | Gap threshold used for drop detection                          |
| `threshold_source`               | `str`   | `"auto_2x_median"` or `"user_specified"`                       |
| `total_gaps_detected`            | `int`   | Number of inter-frame gaps exceeding the threshold             |
| `total_estimated_dropped_frames` | `int`   | Estimated total frames lost across all gaps                    |
| `drop_rate_percent`              | `float` | Percentage of expected frames that were dropped                |
| `longest_gap_us`                 | `int`   | Longest detected gap in microseconds                           |
| `longest_gap_ms`                 | `float` | Longest detected gap in milliseconds                           |
| `drop_locations`                 | `list`  | Per-gap details (capped by `max_drop_locations`)               |
| `drop_locations_truncated`       | `bool`  | True if more drops exist than `max_drop_locations`             |

Each entry in `drop_locations`:

| Field                   | Type    | Description                                 |
|-------------------------|---------|---------------------------------------------|
| `frame_index`           | `int`   | Index of the frame preceding the gap        |
| `gap_us`                | `int`   | Gap duration in microseconds                |
| `gap_ms`                | `float` | Gap duration in milliseconds                |
| `estimated_frames_lost` | `int`   | Estimated number of frames lost in this gap |

**Auto-detection algorithm:** When `drop_threshold_us=0`, the threshold is computed as 2x the median
inter-frame interval. Gaps exceeding this threshold are classified as frame drops. The number of lost
frames per gap is estimated by dividing the gap duration by the median interval and rounding.

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
| 0.1% – 1% | Moderate | May affect high-precision timing analyses             |
| 1% – 5%   | High     | Likely to affect frame-accurate behavioral scoring    |
| > 5%      | Severe   | Recording quality compromised; investigate root cause |

### Common frame drop patterns

| Pattern                   | Characteristic                             | Likely Cause                              |
|---------------------------|--------------------------------------------|-------------------------------------------|
| Increasing gaps over time | Drops worsen as recording progresses       | Thermal throttling (camera or GPU)        |
| Random isolated drops     | Sporadic single-frame gaps throughout      | Interface bandwidth contention (USB/GigE) |
| Periodic drops            | Drops at regular intervals                 | Competing device or process scheduling    |
| Burst drops               | Clusters of consecutive gaps               | Disk write stalls or buffer overflow      |
| Single large gap          | One very long gap in otherwise stable data | System interruption (sleep, update)       |

### Relating results to camera settings

Encoding cost depends on frame content: high-motion or high-detail frames are expensive to encode,
while static or low-contrast frames are cheap. Drops that correlate with scene changes (e.g., animal
entering the field of view) indicate the encoder cannot handle peak complexity at the current preset.

- If `estimated_fps` << configured frame rate: the camera or encoder cannot sustain the configured rate.
  Reduce resolution, frame rate, or switch from CPU to GPU encoding.
- If `std_us` is high but `estimated_fps` is close to target: jitter is present but throughput is
  maintained. Check cable quality, hub topology (USB), or network configuration (GigE).
- If drops appear only in GPU-encoded sessions: the NVENC encoder may be saturated. Reduce quantization
  quality or use a faster speed preset.
- If drops correlate with high-motion periods: use a faster `encoder_speed_preset` to handle peak
  encoding load, or increase `quantization_parameter` to reduce per-frame encoding cost.

---

## Processing status classification

The `discover_processed_camera_logs_tool` classifies each directory into one of four statuses by
cross-referencing feather files with the ProcessingTracker:

| Status                | Meaning                                                                  |
|-----------------------|--------------------------------------------------------------------------|
| `fully_processed`     | All tracker jobs succeeded and feather files present for all source IDs  |
| `partially_processed` | Some feather files present but tracker shows incomplete jobs             |
| `unprocessed`         | Tracker exists but no feather files found                                |
| `no_tracker`          | Feather files found without a tracker file (manual or legacy processing) |

The discovery tool also reports aggregate counts in `status_summary` and per-directory details including
`file_count`, `total_size_bytes`, `source_ids`, and individual feather file paths with sizes.

---

## Related skills

| Skill                    | Relationship                                                       |
|--------------------------|--------------------------------------------------------------------|
| `/log-input-format`      | Reference: input archive format and source ID semantics            |
| `/log-processing`        | Upstream: processing workflow that produces this output            |
| `/camera-interface`      | Context: VideoSystem configuration determines expected frame rates |
| `/mcp-environment-setup` | Prerequisite: MCP server connectivity for tool access              |

---

## Verification checklist

```text
Log Processing Output Completeness:
- [ ] Feather files discovered via `discover_processed_camera_logs_tool`
- [ ] All expected source IDs have corresponding feather files
- [ ] Processing status is `fully_processed` for all directories
- [ ] Frame statistics analyzed via `analyze_camera_frame_statistics_tool` for each camera
- [ ] Estimated FPS matches expected camera frame rate
- [ ] Frame drop rate is within acceptable range for the experiment
- [ ] No unexplained large gaps in inter-frame timing
```
