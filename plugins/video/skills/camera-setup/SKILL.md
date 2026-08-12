---
name: camera-setup
description: >-
  Guides use of ataraxis-video-system MCP tools for camera discovery, runtime verification, interactive
  video session testing, and GenICam camera configuration. Use when discovering connected cameras, verifying
  system encoding requirements, testing camera acquisition, or reading and writing GenICam node values.
user-invocable: false
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
- Writing VideoSystem integration code, and the MCP-to-code parameter mapping (see `/camera-interface`)
- The use-case encoding table and the FFMPEG error catalog (see `/camera-interface`)
- System ID allocation and DataLogger topology (see `/pipeline`)
- The `axvs` command-line surface (see `/cli-reference`)
- Archive assembly and video file validation after a session (see `/post-recording`)
- MCP server connectivity issues (see `/video-mcp-environment-setup`)

**Handoff rules:** After a session stops, invoke `/post-recording` for output verification. When the user moves from
testing to production code, invoke `/camera-interface`. For a multi-camera rig, invoke `/pipeline` before allocating
system IDs. If the user asks what an `axvs` command does, invoke `/cli-reference`. If MCP tools are unavailable,
invoke `/video-mcp-environment-setup`.

---

## MCP tool reference

The ataraxis-video-system MCP server exposes 27 tools. This skill covers the 15 tools most relevant to
camera setup, organized into five groups. Log processing and analysis tools are documented in
`/log-processing` and `/log-processing-results`.

### System verification

| Tool                              | Purpose                                                    |
|-----------------------------------|------------------------------------------------------------|
| `check_runtime_requirements_tool` | Verifies FFMPEG, NVIDIA GPU, and CTI file availability     |
| `get_cti_status_tool`             | Checks whether a GenTL Producer (.cti) file is configured  |
| `set_cti_file_tool`               | Configures the CTI file path for Harvesters camera support |

`set_cti_file_tool` takes one parameter, `file_path` (`str`, required): the absolute path to the vendor-supplied
`.cti` GenTL Producer file. **Always ask the user** for it. It returns `CTI configured: {path}` on success, or a
string beginning `Error:` when the path is missing, is not a file, or the library rejects the Producer. The tool
never raises, so inspect the returned string.

`check_runtime_requirements_tool` returns a pipe-separated status line:
```text
FFMPEG: OK | GPU: OK | CTI: OK
```

- **FFMPEG: Missing** means FFMPEG is not installed or not on PATH. Video encoding will fail.
- **GPU: None** means no NVIDIA GPU is available. CPU encoding still works but is slower.
- **CTI: None** means no GenTL Producer file is configured. Harvesters cameras will not be discoverable.
- **CTI: Unsupported** means the GenICam camera runtime is absent, so Harvesters cameras cannot be used at all.
  Calling `set_cti_file_tool` on such a host is pointless: it returns a string beginning `Error:` that names the
  limitation rather than configuring anything. See the GenICam platform support section below.

`get_cti_status_tool` returns one of three lines: `CTI: {path}` when a valid Producer is configured,
`CTI: Not configured` when none is set or the stored path no longer resolves, and `CTI: Unavailable.` followed by
the reason when the GenICam runtime is absent. Treat `Unavailable` the same way as `Unsupported` above.

The `AXVS_CTI_PATH` environment variable supplies the Producer path for a single runtime and takes precedence over
the path `set_cti_file_tool` persists. A host with that variable set reports a configured CTI without the tool
having been called, so check it whenever the reported path is not the one you expect.

### GenICam platform support

The GenICam (Harvesters) camera interface requires the `harvesters` and `genicam` distributions, which
ataraxis-video-system installs on Linux and Windows only. macOS never has them, because the `genicam`
distribution publishes no macOS wheel for every Python version the library supports. On a host without the
runtime:

- `check_runtime_requirements_tool` reports `CTI: Unsupported` and `get_cti_status_tool` reports `CTI: Unavailable.`
- `list_cameras_tool` returns only the OpenCV cameras and appends `Harvesters discovery skipped.` with the reason
- `set_cti_file_tool` and all four GenICam configuration tools return an error naming the limitation
- Starting a session with `interface="harvesters"` fails for the same reason

None of this is a wiring, driver, or configuration fault, and no camera-side change fixes it. The reason the
tools report names one of two causes, so read it rather than assuming:

- **macOS** — the wheel does not exist, so this is permanent. Use the `opencv` interface, or drive GenICam
  cameras from a Linux or Windows host. Every other library feature, including video encoding and log
  processing, works normally there. The only other macOS restriction is the preview window, which is always
  disabled.
- **Linux or Windows** — the distributions install with the library, so an absent runtime means a damaged
  installation. The reported reason instructs the user to reinstall the library. See
  `/video-mcp-environment-setup`.

### Camera discovery

| Tool                | Purpose                                                                   |
|---------------------|---------------------------------------------------------------------------|
| `list_cameras_tool` | Discovers all cameras accessible through OpenCV and Harvesters interfaces |

Output format:
```text
OpenCV #0: 1920x1080@30fps
Harvesters #0: Allied Vision Mako G-040B (DEV_1234) 1936x1216@40fps
```

Each line shows the interface type, camera index, and native resolution/frame rate. Harvesters cameras also show model
and serial number. The camera index is the value to pass to `start_video_session_tool`
or to the `VideoSystem` constructor.

When nothing is found, the tool returns `No cameras discovered on the system.` instead of a list. Either response
carries a trailing `Harvesters discovery skipped.` note with the reason when the GenICam runtime is absent, which
means the host cannot enumerate GenICam hardware at all rather than that none is attached.

### Video session management

| Tool                       | Purpose                                                                    |
|----------------------------|----------------------------------------------------------------------------|
| `start_video_session_tool` | Starts camera acquisition with configurable encoding                       |
| `stop_video_session_tool`  | Stops active session, assembles log archives, returns output paths         |
| `start_frame_saving_tool`  | Begins recording acquired frames to MP4                                    |
| `stop_frame_saving_tool`   | Stops recording, keeps acquisition active                                  |
| `get_session_status_tool`  | Returns detailed session status including encoding params and output paths |

Only one video session can be active at a time.

A session writes exactly one `{system_id:03d}.mp4` (`112.mp4` for MCP sessions) for its whole lifetime, and the
file is finalized only when the session stops. `stop_frame_saving_tool` clears a flag rather than closing the
file, so a later `start_frame_saving_tool` call resumes appending to the same file. Several save/stop cycles in
one session therefore produce one video, not several. To get separate files, stop the session and start a new one.

`start_video_session_tool`, `start_frame_saving_tool`, and `stop_frame_saving_tool` return plain strings and
never raise. Success reads `Session started: ...`, `Recording started`, and `Recording stopped` respectively,
while every failure is a string beginning `Error:`. Check that prefix on every call, since a failed start
returns a value rather than an exception and a subsequent recording call would otherwise look successful.

**`start_video_session_tool` parameter details:**

| Parameter                | Type         | Default     | Description                                                  |
|--------------------------|--------------|-------------|--------------------------------------------------------------|
| `output_directory`       | `str`        | (required)  | Path to directory for video output. **Always ask the user.** |
| `interface`              | `str`        | `"opencv"`  | Camera interface: `"opencv"`, `"harvesters"`, or `"mock"`    |
| `camera_index`           | `int`        | `0`         | Camera index from `list_cameras_tool` output                 |
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

- `monochrome` and every parameter listed after it (`video_encoder`, `encoder_speed_preset`,
  `output_pixel_format`, `quantization_parameter`) are keyword-only. Pass them by name.
- `interface`: `"mock"` produces synthetic frames with no hardware involvement and is intended only for
  pipeline/library testing; to test a real camera use `"opencv"` or `"harvesters"`.
- `monochrome` applies to the `"opencv"` and `"mock"` interfaces only. The `"harvesters"` interface takes its
  color mode from the camera's own GenICam configuration, so the flag has no effect there.
- `display_frame_rate`: on macOS, preview display is automatically disabled regardless of this value, so the
  absence of a preview window on macOS is expected and not a session failure. On other platforms it must not
  exceed `frame_rate`.
- `quantization_parameter` must be between 0 and 51 inclusive. There is no sentinel value.

The encoding parameter guidance section below covers what is specific to these MCP defaults. For choosing an
encoder, preset, pixel format, and quantization parameter for a given use case, invoke `/camera-interface`.

**`stop_video_session_tool` return structure:**

```text
{"status": "stopped", "video_file": "/path/to/112.mp4", "log_directory": "/path/to/...",
 "archives_assembled": true, "source_ids": ["112"]}
```

**`get_session_status_tool` return structure:**

Returns `{"status": "inactive"}` when no session exists. While a session object exists, `status` is `"running"`
or `"stopped"`, and the dictionary also carries `name` (always `"live_camera"` for MCP sessions), `interface`,
`camera_index`, `resolution`, `frame_rate`, `monochrome`, `encoder`, `encoder_speed_preset`,
`output_pixel_format`, `quantization_parameter`, `gpu_encoding`, `output_directory`, `display_frame_rate`,
`video_file`, and `log_directory`.

### GenICam configuration

These tools are for Harvesters cameras only. They connect to the camera temporarily, perform the operation, and
disconnect.

| Tool                       | Parameters                                                            | Purpose                                         |
|----------------------------|-----------------------------------------------------------------------|-------------------------------------------------|
| `read_genicam_node_tool`   | `camera_index`, `node_name`, `blacklisted_nodes`                      | Reads a single node or lists all writable nodes |
| `write_genicam_node_tool`  | `camera_index`, `node_name`, `value`                                  | Sets a GenICam node value                       |
| `dump_genicam_config_tool` | `camera_index`, `output_file`, `blacklisted_nodes`                    | Exports full camera config to YAML              |
| `load_genicam_config_tool` | `camera_index`, `config_file`, `strict_identity`, `blacklisted_nodes` | Applies config from YAML to camera              |

All four tools return plain strings and never raise. Every failure comes back as a string beginning `Error:`,
so check that prefix on every call.

**`read_genicam_node_tool` behavior:**
- `camera_index` defaults to `0` and `node_name` defaults to `""`
- With `node_name` provided: returns detailed metadata (type, value, access mode, range, unit, description)
- With `node_name` empty: returns `Found {N} writable GenICam nodes:` followed by one `  {name} = {value}` line
  per node. A node that cannot be read renders as `<unreadable>` rather than aborting the listing

**`write_genicam_node_tool` behavior:**
- The `value` parameter is always a string; it is automatically coerced to the node's native type (int, float, bool,
  or enum string)

**`dump_genicam_config_tool` / `load_genicam_config_tool`:**
- **Always ask the user** for the `output_file` or `config_file` path before calling these tools
- `strict_identity` (default `false`): when `true`, aborts if camera model/serial does not match the config file;
  when `false`, warns but proceeds. It is keyword-only, as is `load_genicam_config_tool`'s `blacklisted_nodes`
- The dump confirmation counts one entry per selector combination for selector-addressed nodes, so the reported
  node count can exceed the number of distinct feature names

**`blacklisted_nodes` (read/dump/load tools):**
- Optional `list[str] | None`; when omitted, defaults to `{"CustomerIDKey", "CustomerValueKey", "TestPattern"}`
- Pass an empty list (`[]`) to disable blacklisting and operate on all matching nodes

### Camera manifest management

Camera manifests (`camera_manifest.yaml`) identify which log archives in a DataLogger output directory were
produced by ataraxis-video-system and associate each source ID with a human-readable name. Manifests are
written automatically by `VideoSystem.__init__()` and by `start_video_session_tool`. These tools provide manual
manifest management for retroactive tagging or inspection.

| Tool                         | Parameters                           | Purpose                                         |
|------------------------------|--------------------------------------|-------------------------------------------------|
| `read_camera_manifest_tool`  | `manifest_path`                      | Reads a manifest and returns its source entries |
| `write_camera_manifest_tool` | `log_directory`, `source_id`, `name` | Registers a camera source in the manifest       |

**`read_camera_manifest_tool` return structure:**

```text
{"manifest_path": "/path/to/camera_manifest.yaml", "sources": [{"id": 51, "name": "face_camera"}], "total_sources": 1}
```

**`write_camera_manifest_tool` return structure:**

```text
{"manifest_path": "/path/to/camera_manifest.yaml", "registered_source": {"id": 51, "name": "face_camera"},
 "status": "success"}
```

Returns an `error` dictionary instead when the log directory is absent or is not a directory, when `name` is
empty, or when the write fails. Confirm `status` is `success` before running discovery against the directory.

**`write_camera_manifest_tool` behavior:**
- Creates a new manifest if one does not exist. Otherwise it replaces the entry already registered under the same
  `source_id`, and appends a new entry only when the manifest carries none for that ID
- The write runs under a `.lock` file beside the manifest and aborts if the lock cannot be taken within 10 seconds
- The `name` parameter must be a non-empty string (e.g., `"face_camera"`, `"body_camera"`)
- Use this tool to retroactively tag log archives from sessions that predate the manifest system

---

## Workflows

### Pre-implementation system check

Run this before any camera work to verify the host system is ready:

1. Call `check_runtime_requirements_tool`
2. If FFMPEG is missing, instruct the user to install FFMPEG n8.1
3. If GPU is None and hardware encoding is desired, verify NVIDIA drivers
4. If CTI is None and Harvesters cameras are needed, call `set_cti_file_tool` with the user's CTI path. If the
   user reports a CTI path you did not configure, check whether `AXVS_CTI_PATH` is set in their environment
5. If CTI is Unsupported, stop all Harvesters work. The host has no GenICam runtime, so `set_cti_file_tool` and
   every GenICam tool can only return an error. Steer the user to the `opencv` interface or a Linux/Windows host

### Camera discovery

1. Call `list_cameras_tool`
2. Record camera indices for configuration
3. If no cameras appear:
   - Check physical USB/GigE connections
   - Verify camera drivers are installed
   - For Harvesters: call `get_cti_status_tool` and `set_cti_file_tool` if needed
   - Check for port conflicts with other applications

### Interactive camera testing

Use this workflow to verify a camera works before writing integration code:

1. Ask the user for an output directory
2. Call `start_video_session_tool` with the camera index from discovery
3. Verify the session starts (check `get_session_status_tool` returns "running")
4. Call `start_frame_saving_tool` to test recording
5. Call `stop_frame_saving_tool` to end recording
6. Call `stop_video_session_tool` to release resources
7. Verify the output .mp4 file was created in the output directory

### GenICam camera configuration

Use this workflow to inspect or modify Harvesters camera settings:

**Inspect current configuration:**
1. Call `read_genicam_node_tool` with empty `node_name` to list all writable nodes
2. Call `read_genicam_node_tool` with a specific `node_name` for detailed metadata

**Modify a single setting:**
1. Call `read_genicam_node_tool` to check the current value and valid range/entries
2. Call `write_genicam_node_tool` with the new value. Success returns `Node '{node_name}' set to {value}`, so
   confirm the string does not begin `Error:` before treating the write as applied
3. Call `read_genicam_node_tool` again to confirm the change took effect

Set `PixelFormat` to an 8-bit format (Mono8, BGR8, RGB8) before recording. The VideoSystem constructor grabs a
probe frame and rejects the camera with a ValueError when the frames are not 8-bit, so a camera left on Mono12
or Mono16 fails to start rather than being down-converted.

**Save and restore configuration:**
1. Ask the user for a YAML file path
2. Call `dump_genicam_config_tool` to export the current configuration
3. On a different session or camera, call `load_genicam_config_tool` to apply the saved configuration

---

## Encoding parameter guidance

This section covers only what is specific to an MCP session. `/camera-interface` owns the use-case encoding table, the
encoder and pixel format trade-offs, the H264-to-H265 quantization equivalence, and the FFMPEG error catalog. Read
those from there rather than from a restatement here.

The MCP defaults (`H264`, preset `3`, `yuv420p`, QP `15`) are tuned for a quick compatibility-first test, not for
production. Two of them deserve attention while testing:

- **Preset.** `3` (FAST) suits a quick camera test. Raise it to `4` for an extended test recording, and to `5` when the
  point of the session is evaluating the camera's own image quality rather than proving the pipeline runs.
- **Quantization parameter.** The default of `15` is calibrated for H265 and is likely too low for the H264 default a
  session starts with. Around 15-20 is a reasonable place to begin for H264. Tune it for the scene rather than treating
  it as a documented equivalence.

Every recommendation is a healthy starting point. Actual parameters must be fine-tuned by the end user for their
specific camera, scene content, and throughput requirements.

---

## Bridge to code integration

Testing through MCP is the intended way to settle a camera's parameters before writing code. Once the session works:

1. Use `list_cameras_tool` to discover camera indices and native resolution and frame rate
2. Use `start_video_session_tool` to confirm the camera works at the desired parameters
3. Use the GenICam tools to find and apply the camera's own optimal settings (Harvesters only)
4. Invoke `/camera-interface` and translate the settled parameters through its MCP-to-code mapping table

**The mapping table lives in `/camera-interface` alone.** Several MCP parameters change name (`width` becomes
`frame_width`), one inverts (`monochrome` becomes `color`), several change type from string to enum, and three code
parameters have no MCP counterpart at all. Do not translate them from memory.

An MCP session is fixed at `system_id=112` and `name="live_camera"`, so neither value transfers to code. `/pipeline`
owns system ID allocation and the DataLogger topology that constrains it.

---

## Troubleshooting

| Symptom                                            | Likely Cause                       | Resolution                                                  |
|----------------------------------------------------|------------------------------------|-------------------------------------------------------------|
| `check_runtime_requirements_tool` → FFMPEG Missing | FFMPEG not installed               | Install FFMPEG n8.1 and ensure it is on PATH                |
| `check_runtime_requirements_tool` → GPU None       | No NVIDIA GPU or drivers           | Install NVIDIA drivers, or use CPU encoding (gpu=-1)        |
| `check_runtime_requirements_tool` → CTI Unsupported | GenICam runtime absent (macOS)    | Use the `opencv` interface or a Linux/Windows host          |
| `list_cameras_tool` returns no cameras             | No cameras connected               | Check physical connections, drivers, CTI configuration      |
| `list_cameras_tool` → "Harvesters discovery skipped" | GenICam runtime absent           | Not a wiring fault; no camera-side fix exists               |
| `start_video_session_tool` → error                 | Session already active             | Call `stop_video_session_tool` first                        |
| `start_video_session_tool` → directory error       | Output directory does not exist    | Create the directory or provide a valid path                |
| GenICam tool errors                                | Camera not Harvesters-compatible   | GenICam tools only work with Harvesters cameras             |
| GenICam error naming the interface as unsupported  | GenICam runtime absent             | Host limitation; see the GenICam platform support section   |
| Session start fails on frame data type             | Camera set to a wider-than-8-bit format | Write `PixelFormat` to Mono8, BGR8, or RGB8            |
| `write_genicam_node_tool` fails                    | Node is read-only or value invalid | Use `read_genicam_node_tool` to check access mode and range |
| MCP tools unavailable                              | Server not running                 | Use `/video-mcp-environment-setup` to diagnose              |

---

## CLI equivalents

Every tool in this skill has an `axvs` command a user can run by hand, and `/cli-reference` is canonical for all of
them. **Agents must never invoke `axvs` commands.** Invoke `/cli-reference` when the user asks what a command does, or
when `/video-mcp-environment-setup` has established that the server cannot be restored and the work must move to the
terminal.

| Tool group                  | User-facing command                                                          |
|-----------------------------|------------------------------------------------------------------------------|
| CTI configuration           | `axvs cti set`, `axvs cti check`                                             |
| Runtime and discovery       | `axvs check compatibility`, `axvs check devices`                             |
| Video session               | `axvs run`, which is keypress-driven and records at fixed encoding           |
| GenICam configuration       | `axvs configure read`, `write`, `dump`, `load`                               |
| Camera manifest management  | No CLI path. `read_camera_manifest_tool` and `write_camera_manifest_tool` only |

---

## Related skills

| Skill                          | Relationship                                                      |
|--------------------------------|-------------------------------------------------------------------|
| `/camera-interface`            | Owns the MCP-to-code mapping and the use-case encoding guidance   |
| `/post-recording`              | Downstream: verification after recording sessions                 |
| `/cli-reference`               | Reference: the `axvs` commands equivalent to these tools          |
| `/log-input-format`            | Reference: documents the archive format produced by this workflow |
| `/log-processing`              | Downstream: processes archives from camera sessions               |
| `/log-processing-results`      | Downstream: analyzes frame statistics from processed archives     |
| `/pipeline`                    | Context: end-to-end orchestration and multi-camera planning       |
| `/video-mcp-environment-setup` | Prerequisite: MCP server connectivity for all tool interactions   |

---

## Verification checklist

```text
Camera Setup:
- [ ] Verified runtime requirements (FFMPEG, GPU, CTI) via check_runtime_requirements_tool
- [ ] Configured CTI file if Harvesters cameras are needed
- [ ] Discovered cameras and recorded indices via list_cameras_tool
- [ ] Tested camera with interactive video session
- [ ] Verified recording produces valid MP4 output
- [ ] Configured GenICam nodes if using Harvesters cameras (optional)
- [ ] Every tool return checked for an `Error:` prefix or an `error` key before treating it as success
- [ ] Session stopped and handed to /post-recording for output verification
- [ ] /camera-interface invoked for the parameter mapping, if transitioning from MCP to code
```
