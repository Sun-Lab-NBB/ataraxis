# Example exploration output

A complete worked example of the codebase-exploration output format for a medium or large project.
Reproduce this structure (headings, tables, lists) when presenting findings. Include all sections for
medium and large projects; for small projects, omit sections that do not apply.

```markdown
## Project purpose

Provides a camera interface library for OpenCV and GenICam cameras with real-time FFMPEG video
encoding, including an MCP server and CLI tools for camera management.

## Entry points and CLI commands

| Entry point    | Location               | Description                         |
|----------------|------------------------|-------------------------------------|
| `axvs`         | pyproject.toml scripts | Main CLI entry point                |
| `list-cameras` | cli.py:list_cameras    | Enumerates connected cameras        |
| `live`         | cli.py:live            | Opens a live camera preview session |
| `mcp`          | cli.py:mcp             | Starts the MCP server               |

## Key components

| Component       | Location                                  | Purpose                                |
|-----------------|-------------------------------------------|----------------------------------------|
| VideoSystem     | src/ataraxis_video_system/video_system.py | Top-level camera acquisition lifecycle |
| Camera backends | src/ataraxis_video_system/camera/         | OpenCV and GenICam camera interfaces   |
| Savers          | src/ataraxis_video_system/saver/          | FFMPEG-based video and image encoders  |
| Configuration   | src/ataraxis_video_system/configuration/  | Acquisition and encoding settings      |
| MCP Server      | src/ataraxis_video_system/interfaces/     | AI agent integration via MCP           |

## Call chain summary

`axvs live` → `cli.py:live` → `video_system.py:VideoSystem.start`
→ `camera/opencv_camera.py:OpenCVCamera.connect` → `saver/video_saver.py:VideoSaver.create_encoder`
→ writes encoded frames to the output directory.

## Import dependency map

Central components (imported by 5+ modules):
- `configuration/acquisition_config.py` — imported by video_system, camera, saver
- `camera/camera_base.py` — base class imported by all camera backends
- `types.py` — imported by all modules for shared type definitions

Dependency direction: cli → video_system → camera + saver → configuration.

## Public API surface

Exported from `__init__.py` via `__all__`:
- `VideoSystem` (class)
- `CameraBackends` (enum)
- `SaverBackends` (enum)

## MCP tools

| Tool                | Location                      | Parameters       | Returns              |
|---------------------|-------------------------------|------------------|----------------------|
| `list_cameras_tool` | interfaces/discovery_tools.py | (none)           | list of camera dicts |
| `check_camera_tool` | interfaces/discovery_tools.py | `camera_id: int` | camera status dict   |

## Configuration

| Mechanism           | Location                            | Purpose                                  |
|---------------------|-------------------------------------|------------------------------------------|
| `AcquisitionConfig` | configuration/acquisition_config.py | Dataclass with camera + encoder settings |
| CLI arguments       | cli.py                              | Override config values at runtime        |
| YAML config         | User-provided path                  | Full acquisition configuration file      |

## Test coverage

| Source module            | Test file                   | Coverage |
|--------------------------|-----------------------------|----------|
| video_system.py          | tests/video_system_test.py  | Yes      |
| camera/opencv_camera.py  | tests/opencv_camera_test.py | Yes      |
| saver/video_saver.py     | tests/video_saver_test.py   | Yes      |
| interfaces/mcp_server.py | (none)                      | Gap      |

## Notable patterns

- Multiprocessing for concurrent acquisition and encoding
- Configuration dataclasses with validation
- MyPy strict mode with full type annotations
- Camera backends follow a consistent connect → grab → release interface

## Areas of concern

- `interfaces/mcp_server.py` lacks test coverage
- GenICam backend requires a vendor SDK that is unavailable in CI
- Saver module has high cyclomatic complexity
- Several `# type: ignore` suppressions in the camera backend
```
