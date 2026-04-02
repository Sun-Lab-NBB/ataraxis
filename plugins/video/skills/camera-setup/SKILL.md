---
name: camera-setup
description: >-
  Guides use of ataraxis-video-system MCP tools for camera discovery, runtime verification, interactive
  video session testing, and GenICam camera configuration. Use when discovering connected cameras, verifying
  system encoding requirements, testing camera acquisition, or reading and writing GenICam node values.
user-invocable: true
---

# Camera setup

Guides the use of the ataraxis-video-system MCP tools for system verification, camera discovery, interactive testing, 
and GenICam configuration. This skill covers all MCP tool interactions; for writing code that integrates VideoSystem 
into an acquisition system, use `/camera-interface` instead.

---

## Scope

**Covers:**
- Verifying runtime requirements (FFMPEG, GPU, CTI)
- Discovering connected cameras and their properties
- Managing CTI file configuration for Harvesters cameras
- Running interactive video capture sessions via MCP
- Reading, writing, dumping, and loading GenICam node configurations

**Does not cover:**
- Writing VideoSystem integration code (see `/camera-interface`)
- MCP server connectivity issues (see `/mcp-environment-setup`)

---

## MCP tool reference

The ataraxis-video-system MCP server exposes 27 tools. This skill covers the 15 tools most relevant to
camera setup, organized into five groups. Log processing and analysis tools are documented in
`/log-processing` and `/log-processing-results`.

### System verification

| Tool                         | Purpose                                                            |
|------------------------------|--------------------------------------------------------------------|
| `check_runtime_requirements` | Verifies FFMPEG, NVIDIA GPU, and CTI file availability             |
| `get_cti_status`             | Checks whether a GenTL Producer (.cti) file is configured          |
| `set_cti_file`               | Configures the CTI file path for Harvesters camera support         |

`check_runtime_requirements` returns a pipe-separated status line:
```text
FFMPEG: OK | GPU: OK | CTI: OK
```

- **FFMPEG: Missing** means FFMPEG is not installed or not on PATH. Video encoding will fail.
- **GPU: None** means no NVIDIA GPU is available. CPU encoding still works but is slower.
- **CTI: None** means no GenTL Producer file is configured. Harvesters cameras will not be discoverable.

### Camera discovery

| Tool             | Purpose                                                                    |
|------------------|----------------------------------------------------------------------------|
| `list_cameras`   | Discovers all cameras accessible through OpenCV and Harvesters interfaces  |

Output format:
```text
OpenCV #0: 1920x1080@30fps
Harvesters #0: Allied Vision Mako G-040B (DEV_1234) 1936x1216@40fps
```

Each line shows the interface type, camera index, and native resolution/frame rate. Harvesters cameras also show model
and serial number. The camera index is the value to pass to `start_video_session` or to the `VideoSystem` constructor.

### Video session management

| Tool                  | Purpose                                                                        |
|-----------------------|--------------------------------------------------------------------------------|
| `start_video_session` | Starts camera acquisition with configurable encoding                           |
| `stop_video_session`  | Stops active session, assembles log archives, returns output paths             |
| `start_frame_saving`  | Begins recording acquired frames to MP4                                        |
| `stop_frame_saving`   | Stops recording, keeps acquisition active                                      |
| `get_session_status`  | Returns detailed session status including encoding params and output paths     |

Only one video session can be active at a time.

**`start_video_session` parameter details:**

| Parameter                | Type         | Default     | Description                                                  |
|--------------------------|--------------|-------------|--------------------------------------------------------------|
| `output_directory`       | `str`        | (required)  | Path to directory for video output. **Always ask the user.** |
| `interface`              | `str`        | `"opencv"`  | Camera interface: `"opencv"`, `"harvesters"`, or `"mock"`    |
| `camera_index`           | `int`        | `0`         | Camera index from `list_cameras` output                      |
| `width`                  | `int`        | `600`       | Frame width in pixels                                        |
| `height`                 | `int`        | `400`       | Frame height in pixels                                       |
| `frame_rate`             | `int`        | `30`        | Target acquisition frame rate in FPS                         |
| `gpu_index`              | `int`        | `-1`        | GPU index for hardware encoding (-1 for CPU)                 |
| `display_frame_rate`     | `int / None` | `25`        | Preview display rate in FPS (None disables preview)          |
| `monochrome`             | `bool`       | `false`     | Capture in grayscale instead of color                        |
| `video_encoder`          | `str`        | `"H264"`    | Video encoder: `"H264"` or `"H265"`                          |
| `encoder_speed_preset`   | `int`        | `3`         | Speed preset from 1 (fastest) to 7 (slowest)                 |
| `output_pixel_format`    | `str`        | `"yuv420p"` | Pixel format: `"yuv420p"` or `"yuv444p"`                     |
| `quantization_parameter` | `int`        | `15`        | Compression quality (0 = best, 51 = worst)                   |

See the encoding parameter guidance section below for recommendations on encoder, preset, pixel format, and
quantization parameter selection.

**`stop_video_session` return structure:**

```text
{"status": "stopped", "video_file": "/path/to/112.mp4", "log_directory": "/path/to/...",
 "archives_assembled": true, "source_ids": ["112"]}
```

**`get_session_status` return structure:**

Returns `{"status": "inactive"}` when no session exists. When a session is active, returns a dictionary
with interface, resolution, frame rate, encoder, preset, pixel format, quantization parameter, GPU encoding
flag, output directory, video file path, and log directory.

### GenICam configuration

These tools are for Harvesters cameras only. They connect to the camera temporarily, perform the operation, and
disconnect.

| Tool                  | Parameters                                           | Purpose                                         |
|-----------------------|------------------------------------------------------|-------------------------------------------------|
| `read_genicam_node`   | `camera_index`, `node_name`                          | Reads a single node or lists all writable nodes |
| `write_genicam_node`  | `camera_index`, `node_name`, `value`                 | Sets a GenICam node value                       |
| `dump_genicam_config` | `camera_index`, `output_file`                        | Exports full camera config to YAML              |
| `load_genicam_config` | `camera_index`, `config_file`, `strict_identity`     | Applies config from YAML to camera              |

**`read_genicam_node` behavior:**
- With `node_name` provided: returns detailed metadata (type, value, access mode, range, unit, description)
- With `node_name` empty: lists all writable nodes with current values

**`write_genicam_node` behavior:**
- The `value` parameter is always a string; it is automatically coerced to the node's native type (int, float, bool,
  or enum string)

**`dump_genicam_config` / `load_genicam_config`:**
- **Always ask the user** for the `output_file` or `config_file` path before calling these tools
- `strict_identity` (default `false`): when `true`, aborts if camera model/serial does not match the config file;
  when `false`, warns but proceeds

### Camera manifest management

Camera manifests (`camera_manifest.yaml`) identify which log archives in a DataLogger output directory were
produced by ataraxis-video-system and associate each source ID with a human-readable name. Manifests are
written automatically by `VideoSystem.__init__()` and by `start_video_session`. These tools provide manual
manifest management for retroactive tagging or inspection.

| Tool                         | Parameters                           | Purpose                                         |
|------------------------------|--------------------------------------|-------------------------------------------------|
| `read_camera_manifest_tool`  | `manifest_path`                      | Reads a manifest and returns its source entries |
| `write_camera_manifest_tool` | `log_directory`, `source_id`, `name` | Registers a camera source in the manifest       |

**`read_camera_manifest_tool` return structure:**

```text
{"manifest_path": "/path/to/camera_manifest.yaml", "sources": [{"id": 51, "name": "face_camera"}], "total_sources": 1}
```

**`write_camera_manifest_tool` behavior:**
- Creates a new manifest if one does not exist; appends to the existing manifest otherwise
- The `name` parameter must be a non-empty string (e.g., `"face_camera"`, `"body_camera"`)
- Use this tool to retroactively tag log archives from sessions that predate the manifest system

---

## Workflows

### Pre-implementation system check

Run this before any camera work to verify the host system is ready:

1. Call `check_runtime_requirements`
2. If FFMPEG is missing, instruct the user to install FFMPEG n8.0+
3. If GPU is None and hardware encoding is desired, verify NVIDIA drivers
4. If CTI is None and Harvesters cameras are needed, call `set_cti_file` with the user's CTI path

### Camera discovery

1. Call `list_cameras`
2. Record camera indices for configuration
3. If no cameras appear:
   - Check physical USB/GigE connections
   - Verify camera drivers are installed
   - For Harvesters: call `get_cti_status` and `set_cti_file` if needed
   - Check for port conflicts with other applications

### Interactive camera testing

Use this workflow to verify a camera works before writing integration code:

1. Ask the user for an output directory
2. Call `start_video_session` with the camera index from discovery
3. Verify the session starts (check `get_session_status` returns "Running")
4. Call `start_frame_saving` to test recording
5. Call `stop_frame_saving` to end recording
6. Call `stop_video_session` to release resources
7. Verify the output .mp4 file was created in the output directory

### GenICam camera configuration

Use this workflow to inspect or modify Harvesters camera settings:

**Inspect current configuration:**
1. Call `read_genicam_node` with empty `node_name` to list all writable nodes
2. Call `read_genicam_node` with a specific `node_name` for detailed metadata

**Modify a single setting:**
1. Call `read_genicam_node` to check the current value and valid range/entries
2. Call `write_genicam_node` with the new value
3. Call `read_genicam_node` again to confirm the change took effect

**Save and restore configuration:**
1. Ask the user for a YAML file path
2. Call `dump_genicam_config` to export the current configuration
3. On a different session or camera, call `load_genicam_config` to apply the saved configuration

---

## Encoding parameter guidance

### Encoder selection

| Factor         | H264                 | H265                                |
|----------------|----------------------|-------------------------------------|
| Compatibility  | Wider player support | May require newer players           |
| Compression    | Good                 | ~30-50% better at same quality      |
| Encoding speed | Faster               | Slower (use GPU to offset)          |
| Default        | Yes (MCP default)    | Recommended for production via code |

### Speed preset selection

| Use Case                 | Preset     | Rationale                                          |
|--------------------------|------------|----------------------------------------------------|
| Quick camera test        | FAST (3)   | Fast encoding, broad compatibility                 |
| Extended test recording  | MEDIUM (4) | Balance of speed and file size                     |
| Quality evaluation       | SLOW (5)   | Better compression for evaluating camera output    |

All recommendations are healthy starting points. Actual parameters must be fine-tuned by the end user for
their specific camera, scene content, and throughput requirements.

### Pixel format guidance

| Format   | When to Use                                                                    |
|----------|--------------------------------------------------------------------------------|
| YUV420   | Default; monochrome cameras; storage-sensitive; behavioral video               |
| YUV444   | Color-critical scientific imaging; maximum color fidelity                      |

Monochrome cameras gain nothing from YUV444 since all chrominance channels are zero.

### Quantization parameter guidance

| QP Range | Quality Level | Use Case                                           |
|----------|---------------|----------------------------------------------------|
| 0-5      | Near-lossless | Scientific imaging requiring maximum detail        |
| 10-15    | High quality  | Default for production; good balance               |
| 15-20    | Good quality  | Behavioral video where pixel-perfect is not needed |
| 20-25    | Moderate      | Archival; long recordings with storage constraints |
| 25-35    | Low quality   | Preview and testing only                           |
| 35-51    | Very low      | Not recommended                                    |

The default QP of 15 is calibrated for H265. For H264, QP 15-20 produces equivalent visual quality.

---

## Bridge to code integration

When transitioning from MCP-based testing to writing VideoSystem code, use this mapping.

### Parameter mapping

| MCP Parameter            | VideoSystem Parameter    | Key Difference                                                      |
|--------------------------|--------------------------|---------------------------------------------------------------------|
| `output_directory`       | `output_directory`       | `str` → `Path`; wrap in `Path()`                                    |
| `interface`              | `camera_interface`       | `str` → `CameraInterfaces` enum                                     |
| `camera_index`           | `camera_index`           | Same (`int`)                                                        |
| `width`                  | `frame_width`            | **Name differs**                                                    |
| `height`                 | `frame_height`           | **Name differs**                                                    |
| `frame_rate`             | `frame_rate`             | MCP requires explicit value; code default is `None` (camera native) |
| `gpu_index`              | `gpu`                    | **Name differs**                                                    |
| `display_frame_rate`     | `display_frame_rate`     | MCP default: 25; code default: `None`                               |
| `monochrome`             | `color`                  | **Inverted**: `monochrome=True` → `color=False`                     |
| `video_encoder`          | `video_encoder`          | `str` → `VideoEncoders` enum                                        |
| `encoder_speed_preset`   | `encoder_speed_preset`   | `int` → `EncoderSpeedPresets` enum                                  |
| `output_pixel_format`    | `output_pixel_format`    | `str` → `OutputPixelFormats` enum                                   |
| `quantization_parameter` | `quantization_parameter` | Same (`int`)                                                        |
| (fixed at 112)           | `system_id`              | Code should use 51-100 range                                        |
| (auto-created)           | `data_logger`            | Code must create and manage DataLogger                              |
| (fixed: `"live_camera"`) | `name`                   | **New required param**; code must provide a descriptive camera name |

### System ID semantics

| ID     | Assignment                   | Context                                              |
|--------|------------------------------|------------------------------------------------------|
| 51-100 | Camera VideoSystem instances | Advised range; all other IDs used by other assets    |
| 111    | CLI (`axvs run`)             | Fixed; interactive terminal testing                  |
| 112    | MCP server sessions          | Fixed; agent-driven testing                          |

### Workflow: MCP discovery to code integration

1. Use `list_cameras` to discover camera indices and native resolution/FPS
2. Use `start_video_session` to test the camera works at desired parameters
3. Use GenICam tools to find and configure optimal camera settings (Harvesters only)
4. Translate discoveries to VideoSystem constructor parameters using the mapping table above
5. Use production encoding defaults (H265, SLOW, YUV444) or customize per the encoding guidance

---

## Troubleshooting

| Symptom                                       | Likely Cause                       | Resolution                                             |
|-----------------------------------------------|------------------------------------|--------------------------------------------------------|
| `check_runtime_requirements` → FFMPEG Missing | FFMPEG not installed               | Install FFMPEG n8.0+ and ensure it is on PATH          |
| `check_runtime_requirements` → GPU None       | No NVIDIA GPU or drivers           | Install NVIDIA drivers, or use CPU encoding (gpu=-1)   |
| `list_cameras` returns no cameras             | No cameras connected               | Check physical connections, drivers, CTI configuration |
| `start_video_session` → error                 | Session already active             | Call `stop_video_session` first                        |
| `start_video_session` → directory error       | Output directory does not exist    | Create the directory or provide a valid path           |
| GenICam tool errors                           | Camera not Harvesters-compatible   | GenICam tools only work with Harvesters cameras        |
| `write_genicam_node` fails                    | Node is read-only or value invalid | Use `read_genicam_node` to check access mode and range |
| MCP tools unavailable                         | Server not running                 | Use `/mcp-environment-setup` to diagnose               |

---

## Related skills

| Skill                     | Relationship                                                      |
|---------------------------|-------------------------------------------------------------------|
| `/camera-interface`       | Covers writing VideoSystem integration code after testing via MCP |
| `/post-recording`         | Downstream: verification after recording sessions                 |
| `/log-input-format`       | Reference: documents the archive format produced by this workflow |
| `/log-processing`         | Downstream: processes archives from camera sessions               |
| `/log-processing-results` | Downstream: analyzes frame statistics from processed archives     |
| `/pipeline`               | Context: end-to-end orchestration and multi-camera planning       |
| `/mcp-environment-setup`  | Prerequisite: MCP server connectivity for all tool interactions   |

---

## Verification checklist

```text
Camera Setup:
- [ ] Verified runtime requirements (FFMPEG, GPU, CTI) via check_runtime_requirements
- [ ] Configured CTI file if Harvesters cameras are needed
- [ ] Discovered cameras and recorded indices via list_cameras
- [ ] Tested camera with interactive video session
- [ ] Verified recording produces valid MP4 output
- [ ] Configured GenICam nodes if using Harvesters cameras (optional)
- [ ] Encoding parameters understood (see encoding guidance section)
- [ ] Parameter mapping to code reviewed (if transitioning from MCP to code)
```
