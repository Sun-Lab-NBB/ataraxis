---
name: camera-interface
description: >-
  Guides implementation of camera hardware interfaces using ataraxis-video-system. Covers VideoSystem
  API, binding class patterns, configuration dataclasses, and system ID allocation. Use when writing
  code that integrates cameras into an acquisition system.
---

# Camera Interface Implementation

Guides the implementation of camera hardware interfaces using the ataraxis-video-system library. This skill covers
writing integration code; for interactive camera discovery and testing via MCP tools, use `/camera-setup` instead.

---

## Scope

**Covers:**
- VideoSystem constructor parameters and lifecycle methods
- Binding class patterns for wrapping VideoSystem
- Configuration dataclass design
- System ID allocation
- Encoding and pixel format selection
- GenICam configuration API (programmatic, not MCP)

**Does not cover:**
- Camera discovery, interactive testing, or GenICam node inspection via MCP tools (see `/camera-setup`)
- MCP server connectivity issues (see `/mcp-environment-setup`)

---

## Verification Requirements

**Before writing any camera code, verify the current state of the library.**

### Step 1: Version Verification

Check the locally installed ataraxis-video-system version against the latest release on GitHub:

```bash
pip show ataraxis-video-system
```

The current version is **3.0.0**. If a version mismatch exists, ask the user how to proceed.

### Step 2: API Verification

| File                                                                 | What to Check                                   |
|----------------------------------------------------------------------|-------------------------------------------------|
| `../ataraxis-video-system/src/ataraxis_video_system/__init__.py`     | Exported classes, functions, and public API     |
| `../ataraxis-video-system/src/ataraxis_video_system/video_system.py` | VideoSystem constructor parameters and methods  |
| Project `pyproject.toml`                                             | Current pinned version dependency               |

---

## VideoSystem API Reference

See [references/api-reference.md](references/api-reference.md) for the complete API reference including:

- VideoSystem constructor parameters and their exact types and defaults
- Lifecycle methods (start, stop, start_frame_saving, stop_frame_saving)
- Properties (video_file_path, started, system_id)
- All enumerations (CameraInterfaces, VideoEncoders, EncoderSpeedPresets, OutputPixelFormats, InputPixelFormats)
- Discovery functions (discover_camera_ids, check_cti_file, add_cti_file)
- GenICam configuration classes (GenicamNodeInfo, GenicamConfiguration)
- Utility functions (check_ffmpeg_availability, check_gpu_availability, extract_logged_camera_timestamps)

---

## Binding Class Patterns

When implementing camera support in a binding class, follow these patterns:

### Basic Structure

```python
class VideoSystems:
    """Manages video acquisition from cameras.

    Args:
        data_logger: DataLogger instance for timestamp logging.
        output_directory: Directory path for video file output.
        camera_configuration: Camera settings from system configuration.

    Attributes:
        _camera: VideoSystem instance for frame acquisition.
        _camera_started: Tracks whether acquisition has started.
    """

    def __init__(
        self,
        data_logger: DataLogger,
        output_directory: Path,
        camera_configuration: CameraConfig,
    ) -> None:
        self._camera: VideoSystem = VideoSystem(
            system_id=np.uint8(51),
            data_logger=data_logger,
            output_directory=output_directory,
            camera_interface=CameraInterfaces.HARVESTERS,
            camera_index=camera_configuration.camera_index,
            video_encoder=VideoEncoders.H265,
            encoder_speed_preset=EncoderSpeedPresets(camera_configuration.camera_preset),
            quantization_parameter=camera_configuration.camera_quantization,
            gpu=0,
        )
        self._camera_started: bool = False

    def start(self) -> None:
        """Starts frame acquisition (does not save frames)."""
        if self._camera_started:
            return
        self._camera.start()
        self._camera_started = True

    def start_saving(self) -> None:
        """Begins saving frames to disk."""
        self._camera.start_frame_saving()

    def stop(self) -> None:
        """Stops acquisition and releases resources."""
        if self._camera_started:
            self._camera.stop_frame_saving()
        self._camera.stop()
        self._camera_started = False
```

### Key Patterns

| Pattern              | Purpose                                                   |
|----------------------|-----------------------------------------------------------|
| Idempotency guards   | Prevent double-start with `_started` flags                |
| Destructor cleanup   | `__del__` method calls `stop()` for safety                |
| Separate start/save  | Preview mode before recording                             |
| Configuration inject | Pass dataclass from shared assets                         |
| DataLogger first     | DataLogger must be initialized and started before VideoSystem |

### System ID Allocation

| ID Range | Purpose                                      |
|----------|----------------------------------------------|
| 1-49     | Reserved for non-camera hardware systems     |
| 50-99    | Camera and video acquisition systems         |
| 100+     | Reserved for future expansion                |

Each camera in an acquisition system must have a unique `system_id` (as `np.uint8`) for DataLogger timestamp
correlation. Check the project's existing system ID allocations before assigning new ones. The output video file is
named `{system_id:03d}.mp4` (e.g., `051.mp4` for system_id 51).

---

## Configuration Requirements

Camera configuration should be defined in project shared assets before implementation.

### Required Configuration Fields

| Field                    | Type  | Range | Description                                |
|--------------------------|-------|-------|--------------------------------------------|
| `*_camera_index`         | `int` | 0+    | Camera index from discovery                |
| `*_camera_quantization`  | `int` | 0-51  | Encoding quality (lower = higher quality)  |
| `*_camera_preset`        | `int` | 1-7   | Encoding speed preset (maps to IntEnum)    |

### Configuration Dataclass Pattern

```python
@dataclass()
class SystemCameras:
    """Camera configuration for the acquisition system."""

    camera_index: int = 0
    """Camera index from the discovery function output."""

    camera_quantization: int = 15
    """Quantization parameter (0-51). Lower values produce higher quality."""

    camera_preset: int = 5
    """Encoding speed preset (1-7). Maps to EncoderSpeedPresets enum (SLOW=5 default)."""
```

---

## Encoding Guidance

### Encoder Selection

| Encoder | Use When                                              |
|---------|-------------------------------------------------------|
| H265    | Production workloads (default, better compression)    |
| H264    | Compatibility with older video players is required    |

### Speed Preset Selection

| Preset          | Value | Use When                                                  |
|-----------------|-------|-----------------------------------------------------------|
| FASTEST (1)     | 1     | Maximum throughput, large file size acceptable            |
| FAST (3)        | 3     | Real-time acquisition under CPU/GPU load                  |
| MEDIUM (4)      | 4     | Balanced starting point for real-time acquisition         |
| SLOW (5)        | 5     | Default; good compression, may buffer under heavy load    |
| SLOWEST (7)     | 7     | Offline re-encoding only; will cause frame buffering      |

### Pixel Format Selection

| Format | Use When                                                             |
|--------|----------------------------------------------------------------------|
| YUV444 | Default; preserves color accuracy                                    |
| YUV420 | Smaller files acceptable; monochrome cameras gain nothing from YUV444 |

---

## Troubleshooting

### Encoding Failures

1. Verify FFMPEG installation: `check_ffmpeg_availability()` must return `True`
2. For GPU encoding: `check_gpu_availability()` must return `True`; verify NVIDIA drivers
3. Monitor GPU memory and thermal status during acquisition

### Frame Drops

1. Reduce `display_frame_rate` or disable preview (set to `None`)
2. Use a faster `encoder_speed_preset` (lower number)
3. Increase `quantization_parameter` (reduces quality but lowers encoding cost)

### Process Crashes

1. Ensure DataLogger is initialized and started before VideoSystem
2. Verify output directory exists and is writable
3. Check available disk space
4. Ensure `system_id` is unique across all active VideoSystem instances

---

## Implementation Checklist

```text
Camera Interface Implementation:
- [ ] Verified ataraxis-video-system version matches requirements (>=3.0.0)
- [ ] Verified cameras are discoverable using /camera-setup workflow
- [ ] Created configuration dataclass in shared assets
- [ ] Allocated unique system IDs for each camera (checked existing allocations)
- [ ] Implemented binding class with lifecycle methods
- [ ] DataLogger initialized before VideoSystem in startup sequence
- [ ] Selected appropriate encoder, speed preset, and pixel format
- [ ] Tested acquisition with /camera-setup interactive session before integration
```
