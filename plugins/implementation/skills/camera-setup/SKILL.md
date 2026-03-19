---
name: camera-setup
description: >-
  Guides use of ataraxis-video-system MCP tools for camera discovery, runtime verification, interactive
  video session testing, and GenICam camera configuration. Use when discovering connected cameras, verifying
  system encoding requirements, testing camera acquisition, or reading and writing GenICam node values.
---

# Camera Setup

Guides use of the ataraxis-video-system MCP tools for system verification, camera discovery, interactive testing, and
GenICam configuration. This skill covers all MCP tool interactions; for writing code that integrates VideoSystem into an
acquisition system, use `/camera-interface` instead.

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

## MCP Tool Reference

The ataraxis-video-system MCP server exposes 13 tools organized into four groups.

### System Verification

| Tool                         | Purpose                                                            |
|------------------------------|--------------------------------------------------------------------|
| `check_runtime_requirements` | Verifies FFMPEG, NVIDIA GPU, and CTI file availability             |
| `get_cti_status`             | Checks whether a GenTL Producer (.cti) file is configured          |
| `set_cti_file`               | Configures the CTI file path for Harvesters camera support         |

`check_runtime_requirements` returns a pipe-separated status line:
```
FFMPEG: OK | GPU: OK | CTI: OK
```

- **FFMPEG: Missing** means FFMPEG is not installed or not on PATH. Video encoding will fail.
- **GPU: None** means no NVIDIA GPU is available. CPU encoding still works but is slower.
- **CTI: None** means no GenTL Producer file is configured. Harvesters cameras will not be discoverable.

### Camera Discovery

| Tool             | Purpose                                                                    |
|------------------|----------------------------------------------------------------------------|
| `list_cameras`   | Discovers all cameras accessible through OpenCV and Harvesters interfaces  |

Output format:
```
OpenCV #0: 1920x1080@30fps
Harvesters #0: Allied Vision Mako G-040B (DEV_1234) 1936x1216@40fps
```

Each line shows the interface type, camera index, and native resolution/frame rate. Harvesters cameras also show model
and serial number. The camera index is the value to pass to `start_video_session` or to the `VideoSystem` constructor.

### Video Session Management

| Tool                   | Parameters                                                                                        | Purpose                                    |
|------------------------|---------------------------------------------------------------------------------------------------|--------------------------------------------|
| `start_video_session`  | `output_directory`, `interface`, `camera_index`, `width`, `height`, `frame_rate`, `gpu_index`, `display_frame_rate`, `monochrome` | Starts camera acquisition                  |
| `stop_video_session`   | (none)                                                                                            | Stops active session, releases resources   |
| `start_frame_saving`   | (none)                                                                                            | Begins recording acquired frames to MP4    |
| `stop_frame_saving`    | (none)                                                                                            | Stops recording, keeps acquisition active  |
| `get_session_status`   | (none)                                                                                            | Returns "Inactive", "Running", or "Stopped"|

Only one video session can be active at a time.

**`start_video_session` parameter details:**

| Parameter            | Type          | Default    | Description                                                     |
|----------------------|---------------|------------|-----------------------------------------------------------------|
| `output_directory`   | `str`         | (required) | Path to directory for video output. **Always ask the user.**    |
| `interface`          | `str`         | `"opencv"` | Camera interface: `"opencv"`, `"harvesters"`, or `"mock"`       |
| `camera_index`       | `int`         | `0`        | Camera index from `list_cameras` output                         |
| `width`              | `int`         | `600`      | Frame width in pixels                                           |
| `height`             | `int`         | `400`      | Frame height in pixels                                          |
| `frame_rate`         | `int`         | `30`       | Target acquisition frame rate in FPS                            |
| `gpu_index`          | `int`         | `-1`       | GPU index for hardware encoding (-1 for CPU)                    |
| `display_frame_rate` | `int \| None` | `25`       | Preview display rate in FPS (None disables preview)             |
| `monochrome`         | `bool`        | `false`    | Capture in grayscale instead of color                           |

MCP sessions use fixed encoding defaults: H.264, FAST preset, YUV420, quantization 15. These are intentionally
conservative for broad compatibility. For production encoding settings, use the VideoSystem API directly
(see `/camera-interface`).

### GenICam Configuration

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

---

## Workflows

### Pre-implementation System Check

Run this before any camera work to verify the host system is ready:

1. Call `check_runtime_requirements`
2. If FFMPEG is missing, instruct the user to install FFMPEG n8.0+
3. If GPU is None and hardware encoding is desired, verify NVIDIA drivers
4. If CTI is None and Harvesters cameras are needed, call `set_cti_file` with the user's CTI path

### Camera Discovery

1. Call `list_cameras`
2. Record camera indices for configuration
3. If no cameras appear:
   - Check physical USB/GigE connections
   - Verify camera drivers are installed
   - For Harvesters: call `get_cti_status` and `set_cti_file` if needed
   - Check for port conflicts with other applications

### Interactive Camera Testing

Use this workflow to verify a camera works before writing integration code:

1. Ask the user for an output directory
2. Call `start_video_session` with the camera index from discovery
3. Verify the session starts (check `get_session_status` returns "Running")
4. Call `start_frame_saving` to test recording
5. Call `stop_frame_saving` to end recording
6. Call `stop_video_session` to release resources
7. Verify the output .mp4 file was created in the output directory

### GenICam Camera Configuration

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

## Troubleshooting

| Symptom                              | Likely Cause                       | Resolution                                              |
|--------------------------------------|------------------------------------|---------------------------------------------------------|
| `check_runtime_requirements` → FFMPEG Missing | FFMPEG not installed       | Install FFMPEG n8.0+ and ensure it is on PATH           |
| `check_runtime_requirements` → GPU None       | No NVIDIA GPU or drivers   | Install NVIDIA drivers, or use CPU encoding (gpu=-1)    |
| `list_cameras` returns no cameras    | No cameras connected               | Check physical connections, drivers, CTI configuration  |
| `start_video_session` → error        | Session already active             | Call `stop_video_session` first                         |
| `start_video_session` → directory error | Output directory does not exist | Create the directory or provide a valid path            |
| GenICam tool errors                  | Camera not Harvesters-compatible   | GenICam tools only work with Harvesters cameras         |
| `write_genicam_node` fails           | Node is read-only or value invalid | Use `read_genicam_node` to check access mode and range  |
| MCP tools unavailable                | Server not running                 | Use `/mcp-environment-setup` to diagnose                |

---

## Verification Checklist

```text
Camera Setup:
- [ ] Verified runtime requirements (FFMPEG, GPU, CTI) via check_runtime_requirements
- [ ] Configured CTI file if Harvesters cameras are needed
- [ ] Discovered cameras and recorded indices via list_cameras
- [ ] Tested camera with interactive video session
- [ ] Verified recording produces valid MP4 output
- [ ] Configured GenICam nodes if using Harvesters cameras (optional)
```
