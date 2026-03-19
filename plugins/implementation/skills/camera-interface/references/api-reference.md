# ataraxis-video-system API Reference

Complete API reference for ataraxis-video-system v3.0.0.

---

## Public API

```python
from ataraxis_video_system import (
    # Core
    VideoSystem,
    # Enumerations
    CameraInterfaces,
    VideoEncoders,
    EncoderSpeedPresets,
    InputPixelFormats,
    OutputPixelFormats,
    # Data classes
    CameraInformation,
    GenicamNodeInfo,
    GenicamConfiguration,
    # Discovery and CTI
    discover_camera_ids,
    add_cti_file,
    check_cti_file,
    # Utilities
    check_ffmpeg_availability,
    check_gpu_availability,
    extract_logged_camera_timestamps,
)
```

---

## VideoSystem Class

The main orchestration class for camera acquisition and video encoding.

### Constructor

```python
VideoSystem(
    system_id: np.uint8,
    data_logger: DataLogger,
    output_directory: Path | None,
    camera_interface: CameraInterfaces | str = CameraInterfaces.OPENCV,
    camera_index: int = 0,
    display_frame_rate: int | None = None,
    frame_width: int | None = None,
    frame_height: int | None = None,
    frame_rate: int | None = None,
    gpu: int = -1,
    video_encoder: VideoEncoders | str = VideoEncoders.H265,
    encoder_speed_preset: EncoderSpeedPresets | int = EncoderSpeedPresets.SLOW,
    output_pixel_format: OutputPixelFormats | str = OutputPixelFormats.YUV444,
    quantization_parameter: int = 15,
    *,
    color: bool | None = None,
)
```

### Constructor Parameters

| Parameter                | Type                          | Required | Default                    | Description                                            |
|--------------------------|-------------------------------|----------|----------------------------|--------------------------------------------------------|
| `system_id`              | `np.uint8`                    | Yes      | -                          | Unique identifier for DataLogger timestamp correlation |
| `data_logger`            | `DataLogger`                  | Yes      | -                          | Shared logger instance (must be started)               |
| `output_directory`       | `Path \| None`                | Yes      | -                          | Directory for video output (None disables saving)      |
| `camera_interface`       | `CameraInterfaces \| str`     | No       | `CameraInterfaces.OPENCV`  | Camera backend: HARVESTERS, OPENCV, or MOCK            |
| `camera_index`           | `int`                         | No       | `0`                        | Camera index from discovery functions                  |
| `display_frame_rate`     | `int \| None`                 | No       | `None`                     | Live preview rate in FPS (None disables preview)       |
| `frame_width`            | `int \| None`                 | No       | `None`                     | Override native camera frame width in pixels           |
| `frame_height`           | `int \| None`                 | No       | `None`                     | Override native camera frame height in pixels          |
| `frame_rate`             | `int \| None`                 | No       | `None`                     | Override native camera frame rate in FPS               |
| `gpu`                    | `int`                         | No       | `-1`                       | GPU index for hardware encoding (-1 for CPU only)      |
| `video_encoder`          | `VideoEncoders \| str`        | No       | `VideoEncoders.H265`       | Video codec: H264 or H265                              |
| `encoder_speed_preset`   | `EncoderSpeedPresets \| int`  | No       | `EncoderSpeedPresets.SLOW`  | Encoding speed vs quality tradeoff (1-7)               |
| `output_pixel_format`    | `OutputPixelFormats \| str`   | No       | `OutputPixelFormats.YUV444` | Output color format: YUV420 or YUV444                  |
| `quantization_parameter` | `int`                         | No       | `15`                       | Quality parameter 0-51 (lower = higher quality)        |
| `color`                  | `bool \| None`                | No       | `None`                     | Color mode for OpenCV/Mock (True=BGR, False=MONO). Keyword-only. Harvesters infers from camera config. |

**Notes:**
- `frame_width`, `frame_height`, and `frame_rate` default to the camera's native values when set to None
- `color` is only used by OpenCV and Mock interfaces; Harvesters cameras determine color mode from their GenICam config
- The output video file is named `{system_id:03d}.mp4` in the output directory

### Methods

| Method                 | Returns | Description                                                          |
|------------------------|---------|----------------------------------------------------------------------|
| `start()`              | `None`  | Spawns producer (acquisition) and consumer (encoding) processes      |
| `stop()`               | `None`  | Terminates all processes and releases camera/encoder resources       |
| `start_frame_saving()` | `None`  | Enables writing encoded frames to disk (call after `start()`)        |
| `stop_frame_saving()`  | `None`  | Stops writing frames to disk while keeping acquisition active        |

### Properties

| Property          | Type           | Description                                                   |
|-------------------|----------------|---------------------------------------------------------------|
| `video_file_path` | `Path \| None` | Full path to the output MP4 file (None if saving disabled)    |
| `started`         | `bool`         | True if producer and consumer processes are currently running |
| `system_id`       | `np.uint8`     | The unique system identifier assigned at construction         |

### Lifecycle

```
__init__()
    |
    v
start() ------------------------------------------------+
    |                                                    |
    v                                                    |
[Frame acquisition active, preview if configured]        |
    |                                                    |
    +---- start_frame_saving() ---+                      |
    |                             |                      |
    |                             v                      |
    |              [Frames written to MP4 file]          |
    |                             |                      |
    +---- stop_frame_saving() <---+                      |
    |                                                    |
    v                                                    |
stop() <-------------------------------------------------+
    |
    v
[Resources released]
```

---

## Enumerations

### CameraInterfaces

```python
class CameraInterfaces(StrEnum):
    HARVESTERS = "harvesters"  # GeniCam-compatible cameras (GigE, USB3 Vision)
    OPENCV = "opencv"          # Consumer-grade USB cameras
    MOCK = "mock"              # Testing only (simulated camera)
```

### VideoEncoders

```python
class VideoEncoders(StrEnum):
    H264 = "H264"  # Wider compatibility, larger file size
    H265 = "H265"  # Better compression (recommended for production)
```

### EncoderSpeedPresets

```python
class EncoderSpeedPresets(IntEnum):
    FASTEST = 1   # CPU: veryfast, GPU: p1
    FASTER = 2    # CPU: faster, GPU: p2
    FAST = 3      # CPU: fast, GPU: p3
    MEDIUM = 4    # CPU: medium, GPU: p4
    SLOW = 5      # CPU: slow, GPU: p5 (default)
    SLOWER = 6    # CPU: slower, GPU: p6
    SLOWEST = 7   # CPU: veryslow, GPU: p7
```

### OutputPixelFormats

```python
class OutputPixelFormats(StrEnum):
    YUV420 = "yuv420p"  # Standard, half-bandwidth chrominance
    YUV444 = "yuv444p"  # Better color accuracy (default)
```

### InputPixelFormats

```python
class InputPixelFormats(StrEnum):
    MONOCHROME = "gray"   # Grayscale images
    BGR = "bgr24"         # Color images (BGR channel order)
```

---

## Data Classes

### CameraInformation

Returned by `discover_camera_ids()`:

```python
@dataclass()
class CameraInformation:
    camera_index: int              # Index for VideoSystem constructor
    interface: CameraInterfaces    # OPENCV or HARVESTERS
    frame_width: int               # Native frame width in pixels
    frame_height: int              # Native frame height in pixels
    acquisition_frame_rate: int    # Native frame rate in FPS
    serial_number: str | None      # Harvesters only
    model: str | None              # Harvesters only
```

### GenicamNodeInfo

Stores a single GenICam feature node's value:

```python
@dataclass(frozen=True, slots=True)
class GenicamNodeInfo:
    name: str                              # Feature name (e.g., "Width", "ExposureTime")
    value: int | float | str | bool        # Current value
    unit: str | None = None                # Measurement unit (e.g., "us", "dB"), or None
```

### GenicamConfiguration

Stores a complete camera configuration. Extends `YamlConfig` with `to_yaml()` and `from_yaml()` methods:

```python
@dataclass
class GenicamConfiguration(YamlConfig):
    camera_model: str = ""                          # Model name of the source camera
    camera_serial_number: str = ""                  # Serial number of the source camera
    nodes: list[GenicamNodeInfo] = field(default_factory=list)  # ReadWrite nodes with values
```

**Usage:**
```python
# Save configuration
config.to_yaml(file_path=Path("camera_config.yaml"))

# Load configuration
config = GenicamConfiguration.from_yaml(file_path=Path("camera_config.yaml"))
```

---

## Discovery Functions

### discover_camera_ids

```python
def discover_camera_ids() -> tuple[CameraInformation, ...]
```

Discovers all cameras accessible through both OpenCV and Harvesters interfaces. OpenCV cameras are discovered first.
Harvesters discovery is skipped if no CTI file is configured.

### add_cti_file

```python
def add_cti_file(cti_path: Path) -> None
```

Configures the GenTL Producer file path. Persists across sessions (stored in user data directory via platformdirs).
Must be called before `discover_camera_ids()` or creating a VideoSystem with `CameraInterfaces.HARVESTERS`.

### check_cti_file

```python
def check_cti_file() -> Path | None
```

Returns the configured CTI file path if valid, or None if not configured or the file no longer exists.

---

## Utility Functions

### check_ffmpeg_availability

```python
def check_ffmpeg_availability() -> bool
```

Returns True if FFMPEG is installed and accessible on PATH.

### check_gpu_availability

```python
def check_gpu_availability() -> bool
```

Returns True if NVIDIA GPU hardware encoding is available (checks via `nvidia-smi`).

### extract_logged_camera_timestamps

```python
def extract_logged_camera_timestamps(log_path: Path, n_workers: int = -1) -> tuple[np.uint64, ...]
```

Extracts frame acquisition timestamps from a DataLogger `.npz` archive. Returns timestamps as microseconds since UTC
epoch, in frame order.

| Parameter   | Type   | Default | Description                                         |
|-------------|--------|---------|-----------------------------------------------------|
| `log_path`  | `Path` | -       | Path to the assembled .npz log archive              |
| `n_workers` | `int`  | `-1`    | Parallel workers (-1 for all cores, 1 for sequential) |

---

## Data Logging Format

VideoSystem logs frame acquisition timestamps using the DataLogger class from ataraxis-data-structures.

### Standard Log Entry

Each entry is a 1D numpy uint8 array:

| Offset | Size    | Content                                      |
|--------|---------|----------------------------------------------|
| 0      | 1 byte  | System ID (uint8)                            |
| 1      | 8 bytes | Timestamp (uint64, microseconds since onset) |

### Onset Entry

The first log entry for each VideoSystem uses a special format:

| Offset | Size    | Content                                      |
|--------|---------|----------------------------------------------|
| 0      | 1 byte  | System ID (uint8)                            |
| 1      | 8 bytes | Zero (indicates onset entry)                 |
| 9      | 8 bytes | UTC timestamp (microseconds since epoch)     |

---

## Architecture

```
+------------------------------------------------------------------+
|                          VideoSystem                               |
|  +------------------------------------------------------------+  |
|  |  Main Process                                                |  |
|  |  - Initialization and configuration                          |  |
|  |  - Lifecycle management (start/stop)                         |  |
|  |  - Frame saving control                                      |  |
|  +------------------------------------------------------------+  |
|           |                           |                            |
|           v                           v                            |
|  +--------------------+   +----------------------------------+    |
|  |  Producer Process  |   |       Consumer Process           |    |
|  |  - Camera driver   |   |  - Frame encoding (FFMPEG)       |    |
|  |  - Frame grabbing  |-->|  - MP4 container writing         |    |
|  |  - Timestamp gen   |   |  - Live preview display          |    |
|  +--------------------+   +----------------------------------+    |
|           |                                                        |
|           v                                                        |
|  +--------------------+                                            |
|  |    DataLogger      |                                            |
|  |  - Timestamp I/O   |                                            |
|  |  - .npy file write |                                            |
|  +--------------------+                                            |
+------------------------------------------------------------------+
```

---

## Dependencies

### Runtime

| Package                    | Version       | Purpose                                |
|----------------------------|---------------|----------------------------------------|
| ataraxis-video-system      | >=3.0.0       | This library                           |
| ataraxis-data-structures   | >=6,<7        | DataLogger, SharedMemoryArray          |
| ataraxis-time              | >=6,<7        | Precision timing                       |
| ataraxis-base-utilities    | >=6,<7        | Logging and console output             |
| numpy                      | >=2,<3        | Array operations and system_id type    |
| opencv-python              | >=4.13,<5     | Camera interface and frame display     |
| harvesters                 | >=1,<2        | GeniCam camera support                 |

### External

| Dependency | Required | Purpose                                                |
|------------|----------|--------------------------------------------------------|
| FFMPEG     | Yes      | Backend for H.264/H.265 video encoding                 |
| CTI file   | No       | GenTL Producer for Harvesters cameras                  |
| NVIDIA GPU | No       | Hardware-accelerated encoding (optional)               |

### Python Version

Requires `>=3.12,<3.15`.

---

## Code Examples

### Basic Camera Acquisition

```python
from pathlib import Path
import numpy as np
from ataraxis_data_structures import DataLogger
from ataraxis_video_system import VideoSystem, CameraInterfaces, VideoEncoders

if __name__ == "__main__":
    output_dir = Path("/tmp/camera_test")
    output_dir.mkdir(exist_ok=True)

    logger = DataLogger(output_directory=output_dir, instance_name="test")
    logger.start()

    camera = VideoSystem(
        system_id=np.uint8(51),
        data_logger=logger,
        output_directory=output_dir,
        camera_interface=CameraInterfaces.OPENCV,
        camera_index=0,
        display_frame_rate=15,
        video_encoder=VideoEncoders.H264,
    )

    camera.start()
    # Preview mode - frames displayed but not saved

    camera.start_frame_saving()
    # Recording mode - frames saved to MP4

    # ... acquisition runs ...

    camera.stop_frame_saving()
    camera.stop()
    logger.stop()
```

### GeniCam Camera with Configuration

```python
from pathlib import Path
import numpy as np
from ataraxis_data_structures import DataLogger
from ataraxis_video_system import (
    VideoSystem,
    CameraInterfaces,
    VideoEncoders,
    EncoderSpeedPresets,
    OutputPixelFormats,
    GenicamConfiguration,
    add_cti_file,
    check_cti_file,
    discover_camera_ids,
)

if __name__ == "__main__":
    # Check/configure CTI file
    if check_cti_file() is None:
        add_cti_file(Path("/path/to/your/camera/vendor.cti"))

    # Discover cameras
    cameras = discover_camera_ids()
    harvesters_cameras = [c for c in cameras if c.interface == CameraInterfaces.HARVESTERS]

    # Setup for high-quality recording
    logger = DataLogger(output_directory=Path("/data/session"), instance_name="camera")
    logger.start()

    camera = VideoSystem(
        system_id=np.uint8(51),
        data_logger=logger,
        output_directory=Path("/data/session"),
        camera_interface=CameraInterfaces.HARVESTERS,
        camera_index=harvesters_cameras[0].camera_index,
        display_frame_rate=25,
        video_encoder=VideoEncoders.H265,
        gpu=0,
        encoder_speed_preset=EncoderSpeedPresets.MEDIUM,
        output_pixel_format=OutputPixelFormats.YUV444,
        quantization_parameter=15,
    )

    camera.start()
    camera.start_frame_saving()
    # ... acquisition ...
    camera.stop()
    logger.stop()
```

### Extracting Timestamps

```python
from pathlib import Path
from ataraxis_video_system import extract_logged_camera_timestamps

timestamps = extract_logged_camera_timestamps(Path("051_log.npz"))
frame_times_seconds = [ts / 1e6 for ts in timestamps]  # Convert to seconds
```
