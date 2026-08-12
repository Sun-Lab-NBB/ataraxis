---
name: camera-interface
description: >-
  Guides creation and configuration of VideoSystem instances for camera acquisition. Covers constructor
  parameters, lifecycle methods, system ID allocation, and encoding selection. Use when writing code
  that creates VideoSystem instances or needs to understand the VideoSystem API.
user-invocable: false
---

# Camera interface

Guides creation and configuration of VideoSystem instances.

---

## Scope

**Covers:**
- VideoSystem constructor parameters and lifecycle methods
- The MCP-to-code parameter mapping, which this skill owns for the whole plugin
- Encoding, pixel format, and quantization selection, which this skill owns for the whole plugin
- FFMPEG error interpretation
- DataLogger integration requirements

**Does not cover:**
- Camera discovery, interactive testing, or GenICam node inspection via MCP tools (see `/camera-setup`)
- System ID allocation, DataLogger topology, and the multi-camera lifecycle (see `/pipeline`)
- Archive assembly and output verification after a run (see `/post-recording`)
- Driving log processing from Python. Orchestration runs through MCP or the `axvs` CLI only
- MCP server connectivity issues (see `/video-mcp-environment-setup`)
- Binding class design, configuration dataclasses, or system architecture (consumer's responsibility)

---

## Verification requirements

### Step 1: Version verification

Check the locally installed ataraxis-video-system version against the latest release on GitHub:

```bash
pip show ataraxis-video-system
```

The current version is **5.0.0**. If a version mismatch exists, ask the user how to proceed. 5.0.0 is a breaking release
relative to 4.x: `quantization_parameter` no longer accepts -1, the `video/log_processing.py` module was removed, and
the GenICam interface is no longer installed on macOS.

### Step 2: API verification

| File                                                                       | What to Check                                  |
|----------------------------------------------------------------------------|------------------------------------------------|
| `../ataraxis-video-system/src/ataraxis_video_system/__init__.py`           | Exported classes, functions, and public API    |
| `../ataraxis-video-system/src/ataraxis_video_system/video/video_system.py` | VideoSystem constructor parameters and methods |
| Project `pyproject.toml`                                                   | Current pinned version dependency              |

---

## VideoSystem API reference

See [api-reference.md](references/api-reference.md) for the complete API reference including:

- VideoSystem constructor parameters and their exact types and defaults
- Lifecycle methods (start, stop, start_frame_saving, stop_frame_saving)
- Properties (video_file_path, started, system_id)
- All enumerations (CameraInterfaces, VideoEncoders, EncoderSpeedPresets, OutputPixelFormats, InputPixelFormats)
- Discovery functions (discover_camera_ids, check_cti_file, add_cti_file)
- GenICam configuration classes (GenicamNodeInfo, GenicamConfiguration) and the programmatic config methods
  (set_node_value, get_configuration, apply_configuration) that back the `/camera-setup` tools
- Utility functions (check_ffmpeg_availability, check_gpu_availability, extract_logged_camera_timestamps)

---

## VideoSystem usage

### Creating instances

```python
camera = VideoSystem(
    system_id=np.uint8(51),
    data_logger=data_logger,
    name="behavior_camera",
    output_directory=output_directory,
    camera_interface=CameraInterfaces.HARVESTERS,
    camera_index=0,
    video_encoder=VideoEncoders.H265,
    encoder_speed_preset=EncoderSpeedPresets.SLOW,
    quantization_parameter=15,
    gpu=0,
)
```

Key constructor notes:
- `name` is a required string that identifies the camera for downstream tools. It is written to a
  `camera_manifest.yaml` file in the DataLogger output directory during `__init__()`, associating
  the `system_id` with the human-readable name. The manifest enables `discover_camera_data_tool`
  to identify axvs-produced log archives and match them with video files.
- `data_logger` must be an initialized and started `DataLogger` instance. The VideoSystem sends frame
  timestamps to the logger via its multiprocessing input queue during acquisition.
- `output_directory` accepts `Path | None`. When `None`, frames are acquired but not saved to disk
  (useful for display-only or warm-up without recording).
- `frame_width`, `frame_height`, `frame_rate` default to `None` (use camera native settings). For
  Harvesters cameras, set resolution and frame rate via GenICam configuration (see `/camera-setup`)
  rather than overriding through these parameters. The VideoSystem overrides are primarily intended
  for OpenCV cameras that lack GenICam node control. Exposure, gain, and other GenICam nodes are applied at
  configuration time via the `/camera-setup` tools or, in code, via the `HarvestersCamera` config methods
  (`set_node_value`, `get_configuration`, `apply_configuration`). See the API reference. The deterministic
  acquisition script does not reconfigure nodes at runtime. The camera's `PixelFormat` must be an 8-bit
  format, since the constructor grabs a probe frame and raises `ValueError` on any other frame dtype.
- `display_frame_rate` defaults to `None` (preview disabled). Set to a positive integer FPS not exceeding
  the camera's acquisition frame rate to enable. A value that is neither an int nor `None` raises `TypeError`,
  while an int that is zero, negative, or above the camera's acquisition rate raises `ValueError`. Frame display
  is unsupported on macOS: it is auto-disabled (a warning is emitted), so the value has no effect there.
- `gpu` defaults to `-1` (CPU encoding). Set to `0+` for NVIDIA GPU encoding.
- `quantization_parameter` must be an integer between 0 and 51 inclusive. There is no sentinel for "let the
  encoder decide", and a negative value raises `ValueError`. The parameter is validated only when
  `output_directory` is set, since a system that saves nothing constructs no encoder.
- `camera_interface=CameraInterfaces.HARVESTERS` requires the GenICam runtime, which ataraxis-video-system
  installs on Linux and Windows only. On macOS the constructor raises `NotImplementedError`, as do
  `add_cti_file()` and every GenICam configuration call. Write code that targets GenICam cameras for a
  Linux or Windows host, and use `CameraInterfaces.OPENCV` on macOS.
- `color` is a keyword-only parameter defaulting to `None`. For OpenCV and Mock, `None` resolves to
  monochrome (`False`). A color OpenCV camera left at `None` silently records grayscale, so pass
  `color=True` explicitly to record color. Only Harvesters infers color/mono from the GenICam config.

### Lifecycle

```text
VideoSystem() → start() → [start_frame_saving() → stop_frame_saving()] → stop()
```

- `start()` begins frame acquisition without saving. Useful for preview or warm-up.
- `start_frame_saving()` / `stop_frame_saving()` toggle a flag rather than opening and closing files. One
  VideoSystem writes exactly one `{system_id:03d}.mp4` for its whole lifetime, finalized only when `stop()`
  runs, so toggling saving off and back on resumes appending to that same file. To produce separate video
  files, stop the system and construct a new one.
- `stop()` terminates acquisition and releases all resources. Must be called explicitly. It blocks until
  all buffered frames are encoded, up to a 10-minute (600 s) cap. Beyond that the daemon consumer process is
  abandoned and its remaining frames are discarded, with no error raised.

### System ID allocation

Each VideoSystem instance requires a `system_id` (`np.uint8`, 0-255) that must be unique among all sources sharing one
DataLogger, including sources from sibling libraries. The output video file is named `{system_id:03d}.mp4` (`051.mp4`
for system_id 51), while the log archive uses the bare integer.

`/pipeline` owns the allocation convention, the values the library reserves, and the cross-library coexistence rules.
Invoke it before choosing an ID rather than assuming a band here.

---

## Encoding guidance

### Encoder selection

| Encoder | Use When                                           |
|---------|----------------------------------------------------|
| H265    | Production workloads (default, better compression) |
| H264    | Compatibility with older video players is required |

### Speed preset selection

| Preset  | Value | Use when                                               |
|---------|-------|--------------------------------------------------------|
| FASTEST | 1     | Maximum throughput, large file size acceptable         |
| FASTER  | 2     | High throughput with slightly better compression       |
| FAST    | 3     | Real-time acquisition under CPU/GPU load               |
| MEDIUM  | 4     | Balanced starting point for real-time acquisition      |
| SLOW    | 5     | Default, good compression, may buffer under heavy load |
| SLOWER  | 6     | Higher compression, offline or low-fps recording       |
| SLOWEST | 7     | Maximum compression, offline re-encoding only          |

### Pixel format selection

| Format | Use When                                                              |
|--------|-----------------------------------------------------------------------|
| YUV444 | Default, preserves color accuracy                                     |
| YUV420 | Smaller files acceptable, monochrome cameras gain nothing from YUV444 |

All encoding recommendations in this skill are healthy starting points. Actual parameters must be fine-tuned by the end
user for their specific camera, scene content, and throughput requirements.

---

## Use-case encoding selection

### Recommended configurations by use case

| Use Case                        | Encoder | Preset      | Pixel Format | QP    | GPU | Rationale                                         |
|---------------------------------|---------|-------------|--------------|-------|-----|---------------------------------------------------|
| Scientific imaging (high-speed) | H265    | SLOWEST (7) | YUV444       | 0-5   | 0   | Maximum quality, GPU offloads encoding cost       |
| Behavioral video (color)        | H265    | SLOW (5)    | YUV420       | 15-20 | 0   | Good balance, YUV420 acceptable for scoring       |
| Real-time preview/testing       | H264    | FASTEST (1) | YUV420       | 23    | -1  | Minimum encoding cost, CPU-only for simplicity    |
| Archival (storage-sensitive)    | H265    | SLOWER (6)  | YUV420       | 20-25 | 0   | Best compression, still usable visual quality     |
| Multi-camera rig (bandwidth)    | H265    | FAST (3)    | YUV420       | 15    | 0   | Per-camera encoding must be fast, GPU parallelism |

### GPU vs CPU encoding

| Factor        | CPU                                   | GPU (NVENC)                          |
|---------------|---------------------------------------|--------------------------------------|
| Throughput    | Limited by core speed                 | Higher, especially multi-camera      |
| Quality at QP | Slightly better at same QP            | Slightly lower (NVENC limitation)    |
| Multi-camera  | Competes with other processes for CPU | Dedicated hardware, handles parallel |
| Availability  | Always available                      | Requires NVIDIA GPU                  |

### Quantization parameter cross-encoder equivalence

H265 produces better compression at equivalent visual quality. To match quality across encoders:

| Visual Quality | H265 QP | H264 QP | Notes              |
|----------------|---------|---------|--------------------|
| Near-lossless  | 0-5     | 0-10    | Scientific use     |
| High quality   | 10-15   | 15-20   | Production default |
| Good quality   | 15-20   | 20-25   | Behavioral video   |
| Moderate       | 20-25   | 25-30   | Archival           |

### FFMPEG error interpretation

| Error Pattern                      | Meaning                           | Resolution                                     |
|------------------------------------|-----------------------------------|------------------------------------------------|
| `Unknown encoder 'hevc_nvenc'`     | NVIDIA NVENC not available        | Use CPU encoding (`gpu=-1`) or install drivers |
| `Error initializing output stream` | Resolution or format incompatible | Check frame dimensions are even numbers        |
| `Queue input is backward in time`  | Timestamp ordering issue          | Ensure frames arrive in order                  |
| `CUDA_ERROR_OUT_OF_MEMORY`         | GPU memory exhausted              | Reduce concurrent GPU encoders or resolution   |
| `Too many packets buffered`        | Encoding too slow for frame rate  | Use faster preset or reduce resolution         |

These messages surface on the **stderr of the process that hosts the runtime**, written there directly by the spawned
consumer, because that child re-imports the library with a disabled console. They do not appear in any MCP tool return,
so an agent cannot read them: they land on the terminal for an `axvs run` CLI session, and on the MCP server's stderr
log for an MCP session. Ask the user to check whichever of the two applies.

---

## Bridge from MCP

### What MCP testing reveals for code

| MCP Discovery                              | Informs Code Parameter             | How                                         |
|--------------------------------------------|------------------------------------|---------------------------------------------|
| `list_cameras_tool` output                 | `camera_interface`, `camera_index` | Interface type and index transfer directly  |
| Camera resolution from `list_cameras_tool` | `frame_width`, `frame_height`      | Use native resolution or override           |
| Camera FPS from `list_cameras_tool`        | `frame_rate`                       | Set to `None` for native, or override       |
| `check_runtime_requirements_tool`          | `gpu`                              | If GPU: OK, use `gpu=0`. If None, use `-1`  |
| GenICam node values                        | Camera-level setup                 | Apply same config before VideoSystem init   |
| MCP session success or failure             | Encoding feasibility               | If MCP session works, code session will too |

### Parameter mapping

| MCP Parameter            | VideoSystem Parameter     | Key Difference                                               |
|--------------------------|---------------------------|--------------------------------------------------------------|
| `output_directory` (str) | `output_directory` (Path) | Wrap in `Path()`                                             |
| `interface` (str)        | `camera_interface`        | Use `CameraInterfaces` enum member                           |
| `camera_index` (int)     | `camera_index` (int)      | Same                                                         |
| `width` (int)            | `frame_width` (int)       | **Name change**                                              |
| `height` (int)           | `frame_height` (int)      | **Name change**                                              |
| `frame_rate` (int)       | `frame_rate` (int/None)   | Code default `None` (camera native), MCP requires int        |
| `gpu_index` (int)        | `gpu` (int)               | **Name change**                                              |
| `display_frame_rate`     | `display_frame_rate`      | MCP default 25, code default `None`                          |
| `monochrome` (bool)      | `color` (bool/None)       | **Inverted**: `monochrome=True` → `color=False`              |
| `video_encoder` (str)    | `video_encoder`           | `str` → `VideoEncoders` enum                                 |
| `encoder_speed_preset`   | `encoder_speed_preset`    | `int` → `EncoderSpeedPresets` enum                           |
| `output_pixel_format`    | `output_pixel_format`     | `str` → `OutputPixelFormats` enum                            |
| (fixed at 112)           | `system_id`               | Code uses 51-100, MCP uses 112                               |
| (auto-created)           | `data_logger`             | Code must create and manage DataLogger                       |
| (fixed: `"live_camera"`) | `name`                    | **New required param**, code must provide a descriptive name |

### What MCP cannot test

| Capability               | MCP Limitation                                        | Code Alternative                                    |
|--------------------------|-------------------------------------------------------|-----------------------------------------------------|
| Multi-camera recording   | Single session only                                   | Multiple VideoSystem instances                      |
| Custom DataLogger config | Auto-created with `instance_name="mcp_video_session"` | Full control over DataLogger                        |
| Custom system IDs        | Fixed at 112                                          | Any `np.uint8` value (51-100 recommended)           |
| Custom camera names      | Fixed at `"live_camera"`                              | Descriptive name per camera (e.g., `"face_camera"`) |
| Output directory = None  | Requires directory                                    | Disable saving for preview-only mode                |
| Log archive assembly     | Auto-assembled on stop                                | Call `assemble_log_archives` manually               |

---

## Troubleshooting

### Encoding failures

1. Verify FFMPEG installation: `check_ffmpeg_availability()` must return `True`
2. For GPU encoding: `check_gpu_availability()` must return `True`, and verify NVIDIA drivers
3. Monitor GPU memory and thermal status during acquisition

### Frame drops

Encoding cost depends heavily on frame content: high-motion or high-detail scenes are expensive to encode, while static
or low-contrast scenes are cheap. A preset that works for a stationary camera may cause drops when the scene changes.

1. Reduce `display_frame_rate` or disable preview (set to `None`)
2. Use a faster `encoder_speed_preset` (lower number)
3. Increase `quantization_parameter` (reduces quality but lowers encoding cost)
4. Switch from CPU to GPU encoding (`gpu=0`) if an NVIDIA GPU is available

### Process crashes

1. Verify output directory exists and is writable
2. Check available disk space

---

## Related skills

| Skill                          | Relationship                                                             |
|--------------------------------|--------------------------------------------------------------------------|
| `/camera-setup`                | Upstream: MCP camera discovery and testing that settles these parameters |
| `/post-recording`              | Downstream: verification after recording sessions                        |
| `/cli-reference`               | Reference: `axvs run`, the third parameter set, which no code path uses  |
| `/log-input-format`            | Reference: documents archive format produced by VideoSystem code         |
| `/log-processing`              | Downstream: processes archives from VideoSystem instances                |
| `/log-processing-results`      | Downstream: analyzes frame statistics from processed archives            |
| `/pipeline`                    | Owns system ID allocation, DataLogger topology, and multi-camera code    |
| `/video-mcp-environment-setup` | Prerequisite: MCP server connectivity for API verification               |

---

## Verification checklist

```text
Camera Interface:
- [ ] Verified ataraxis-video-system version matches requirements (>=5.0.0)
- [ ] Verified cameras are discoverable using /camera-setup workflow
- [ ] Allocated system IDs that are unique per DataLogger (51-100 by plugin convention, or the rig's existing
      allocation confirmed with the user)
- [ ] DataLogger initialized and started before VideoSystem creation
- [ ] Encoding configuration selected using use-case guidance table
- [ ] MCP test results translated to code parameters (if applicable)
- [ ] Tested acquisition with /camera-setup interactive session before integration
- [ ] Comments and docstrings fill to 120 characters before wrapping, under the wrap-width rule /python-style defines
- [ ] stop() called explicitly on all VideoSystem instances during shutdown
```
