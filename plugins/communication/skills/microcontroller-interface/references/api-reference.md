# API reference

Complete API reference for ataraxis-communication-interface public classes, functions, and constants.

---

## Public API

```python
from ataraxis_communication_interface import (
    # Core classes
    MicroControllerInterface,
    ModuleInterface,
    MQTTCommunication,
    # Data classes
    ModuleData,
    ModuleState,
    ModuleSourceData,
    MicroControllerSourceData,
    # Configuration classes
    MicroControllerManifest,
    ExtractionConfig,
    ControllerExtractionConfig,
    ModuleExtractionConfig,
    KernelExtractionConfig,
    # Functions
    write_microcontroller_manifest,
    create_extraction_config,
    run_log_processing_pipeline,
    # Constants
    MICROCONTROLLER_MANIFEST_FILENAME,
    EXTRACTION_CONFIGURATION_FILENAME,
)
```

---

## MicroControllerInterface

Manages bidirectional serial communication with a single microcontroller running the
ataraxis-micro-controller firmware library.

### Constructor

```python
MicroControllerInterface(
    controller_id: np.uint8,
    data_logger: DataLogger,
    module_interfaces: tuple[ModuleInterface, ...],
    buffer_size: int,
    port: str,
    name: str,
    baudrate: int = 115200,
    keepalive_interval: int = 0,
)
```

| Parameter            | Type                          | Default    | Description                                                      |
|----------------------|-------------------------------|------------|------------------------------------------------------------------|
| `controller_id`      | `np.uint8`                    | (required) | Unique controller ID (1-255). Advised range: 101-150.            |
| `data_logger`        | `DataLogger`                  | (required) | Initialized and started DataLogger instance for message logging. |
| `module_interfaces`  | `tuple[ModuleInterface, ...]` | (required) | Non-empty tuple of ModuleInterface subclass instances.           |
| `buffer_size`        | `int`                         | (required) | Serial buffer size in bytes (from manufacturer spec).            |
| `port`               | `str`                         | (required) | Serial port device path (e.g., `/dev/ttyACM0`, `COM3`).          |
| `name`               | `str`                         | (required) | Human-readable controller name. Written to manifest on init.     |
| `baudrate`           | `int`                         | `115200`   | UART baudrate. Ignored for USB serial.                           |
| `keepalive_interval` | `int`                         | `0`        | Keepalive interval in milliseconds. 0 disables keepalive.        |

### Methods

| Method               | Returns | Description                                                              |
|----------------------|---------|--------------------------------------------------------------------------|
| `start()`            | `None`  | Spawns communication process, verifies identities, enters comm cycle.    |
| `stop()`             | `None`  | Terminates communication process and releases all resources.             |
| `reset_controller()` | `None`  | Resets the microcontroller to its default state.                         |

### Properties

| Property        | Type                          | Description                                            |
|-----------------|-------------------------------|--------------------------------------------------------|
| `controller_id` | `np.uint8`                    | The controller's unique identifier.                    |
| `name`          | `str`                         | Human-readable controller name.                        |
| `modules`       | `tuple[ModuleInterface, ...]` | All ModuleInterface instances bound to this controller.|

---

## ModuleInterface

Abstract base class for custom hardware module interfaces. Must be subclassed for each hardware module.

### Constructor

```python
ModuleInterface(
    module_type: np.uint8,
    module_id: np.uint8,
    name: str,
    error_codes: set[np.uint8] | None = None,
    data_codes: set[np.uint8] | None = None,
)
```

| Parameter     | Type                   | Default | Description                                      |
|---------------|------------------------|---------|--------------------------------------------------|
| `module_type` | `np.uint8`             | -       | Module family code (1-255). Matches firmware.    |
| `module_id`   | `np.uint8`             | -       | Module instance ID (1-255). Matches firmware.    |
| `name`        | `str`                  | -       | Human-readable name. Written to manifest.        |
| `error_codes` | `set[np.uint8] / None` | `None`  | Event codes that trigger RuntimeError.           |
| `data_codes`  | `set[np.uint8] / None` | `None`  | Event codes routed to `process_received_data()`. |

### Abstract methods

| Method                           | Args                       | Description                                          |
|----------------------------------|----------------------------|------------------------------------------------------|
| `initialize_remote_assets()`     | None                       | Initialize non-picklable resources for comm process. |
| `terminate_remote_assets()`      | None                       | Release resources from `initialize_remote_assets`.   |
| `process_received_data(message)` | `ModuleData / ModuleState` | Handle data_codes messages. Keep fast.               |

### Command methods

| Method                  | Key Parameters                                                                                   | Description                       |
|-------------------------|--------------------------------------------------------------------------------------------------|-----------------------------------|
| `send_command()`        | `command: np.uint8, noblock: np.bool_, repetition_delay: np.uint32 = 0`                          | Send command to module.           |
| `send_parameters()`     | `parameter_data: tuple[np.unsignedinteger \| np.signedinteger \| np.bool_ \| np.floating, ...]`  | Send parameters to module.        |
| `reset_command_queue()` | None                                                                                             | Clear the module's command queue. |

### Properties

| Property      | Type            | Description                                      |
|---------------|-----------------|--------------------------------------------------|
| `module_type` | `np.uint8`      | Module family code.                              |
| `module_id`   | `np.uint8`      | Module instance ID.                              |
| `type_id`     | `np.uint16`     | Combined type+id as uint16: (type << 8) OR id.   |
| `name`        | `str`           | Human-readable module name.                      |
| `data_codes`  | `set[np.uint8]` | Event codes routed to `process_received_data()`. |
| `error_codes` | `set[np.uint8]` | Event codes that trigger RuntimeError.           |

---

## MQTTCommunication

Extends serial microcontroller communication by connecting remote producers and consumers to the
microcontroller ecosystem over TCP. Designed for tight integration with `MicroControllerInterface` —
allows separate processes or machines to send commands to or receive data from microcontrollers via
MQTT topics. Can be used standalone, but the library was designed with integrated usage in mind.

### Constructor

```python
MQTTCommunication(
    ip: str = "127.0.0.1",
    port: int = 1883,
    monitored_topics: tuple[str, ...] | None = None,
)
```

| Parameter          | Type                     | Default       | Description                            |
|--------------------|--------------------------|---------------|----------------------------------------|
| `ip`               | `str`                    | `"127.0.0.1"` | MQTT broker IP address.                |
| `port`             | `int`                    | `1883`        | MQTT broker socket port.               |
| `monitored_topics` | `tuple[str, ...] / None` | `None`        | Topics to subscribe for incoming data. |

### Methods

| Method                             | Returns                                    | Description                                                       |
|------------------------------------|--------------------------------------------|-------------------------------------------------------------------|
| `connect()`                        | `None`                                     | Connects to broker and subscribes to monitored topics.            |
| `disconnect()`                     | `None`                                     | Disconnects from broker. Called automatically on garbage collect. |
| `send_data(topic, payload=None)`   | `None`                                     | Publishes data to the specified MQTT topic.                       |
| `get_data()`                       | `tuple[str, bytes \| bytearray] \| None`    | Returns next received `(topic, payload)` or `None` if empty.     |

### Properties

| Property   | Type   | Description                                          |
|------------|--------|------------------------------------------------------|
| `has_data` | `bool` | `True` if there are received messages waiting.       |

---

## Data classes

### ModuleData

Received data message from a hardware module (protocol code 6). Properties are backed by an internal
`message: NDArray[np.uint8]` field; `data_object` holds the deserialized payload.

| Property         | Type                   | Description                               |
|------------------|------------------------|-------------------------------------------|
| `module_type`    | `np.uint8`             | Module family code of the sending module. |
| `module_id`      | `np.uint8`             | Instance ID of the sending module.        |
| `command`        | `np.uint8`             | Command the module was executing.         |
| `event`          | `np.uint8`             | Event code of the message.                |
| `prototype_code` | `np.uint8`             | Prototype code identifying data layout.   |
| `data_object`    | `np.number \| NDArray` | Deserialized data payload.                |

### ModuleState

Received state message from a hardware module (protocol code 8). Properties are backed by an internal
`message: NDArray[np.uint8]` field.

| Property      | Type       | Description                               |
|---------------|------------|-------------------------------------------|
| `module_type` | `np.uint8` | Module family code of the sending module. |
| `module_id`   | `np.uint8` | Instance ID of the sending module.        |
| `command`     | `np.uint8` | Command the module was executing.         |
| `event`       | `np.uint8` | Event code of the message.                |

### ModuleSourceData (frozen dataclass)

Per-module identification metadata for manifests.

| Field         | Type  | Description                    |
|---------------|-------|--------------------------------|
| `module_type` | `int` | Module family code.            |
| `module_id`   | `int` | Module instance ID.            |
| `name`        | `str` | Human-readable module name.    |

### MicroControllerSourceData (frozen dataclass)

Per-controller identification metadata for manifests.

| Field     | Type                           | Description                     |
|-----------|--------------------------------|---------------------------------|
| `id`      | `int`                          | Controller ID.                  |
| `name`    | `str`                          | Human-readable controller name. |
| `modules` | `tuple[ModuleSourceData, ...]` | Managed hardware modules.       |

---

## Configuration classes

### MicroControllerManifest (YamlConfig)

Registry of controllers and their modules within a DataLogger output directory.

| Field         | Type                                  | Description                      |
|---------------|---------------------------------------|----------------------------------|
| `controllers` | `list[MicroControllerSourceData]`     | Registered controller entries.   |

Methods: `save(file_path)`, `load(file_path)` (classmethod).

### ExtractionConfig (YamlConfig)

Configuration controlling which data is extracted from log archives.

| Field         | Type                                  | Description                       |
|---------------|---------------------------------------|-----------------------------------|
| `controllers` | `list[ControllerExtractionConfig]`    | Controller extraction entries.    |

Methods: `save(file_path)`, `load(file_path)` (classmethod).

### ControllerExtractionConfig (frozen dataclass)

| Field           | Type                                 | Description                    |
|-----------------|--------------------------------------|--------------------------------|
| `controller_id` | `int`                                | Controller ID to extract from. |
| `modules`       | `tuple[ModuleExtractionConfig, ...]` | Module extraction entries.     |
| `kernel`        | `KernelExtractionConfig / None`      | Optional kernel extraction.    |

### ModuleExtractionConfig (frozen dataclass)

| Field         | Type              | Description                           |
|---------------|-------------------|---------------------------------------|
| `module_type` | `int`             | Module family code.                   |
| `module_id`   | `int`             | Module instance ID.                   |
| `event_codes` | `tuple[int, ...]` | Event codes to extract (non-empty).   |

### KernelExtractionConfig (frozen dataclass)

| Field         | Type              | Description                           |
|---------------|-------------------|---------------------------------------|
| `event_codes` | `tuple[int, ...]` | Kernel event codes to extract.        |

---

## Constants

| Constant                               | Value                                | Description                         |
|----------------------------------------|--------------------------------------|-------------------------------------|
| `MICROCONTROLLER_MANIFEST_FILENAME`    | `"microcontroller_manifest.yaml"`    | Manifest filename in log dirs.      |
| `EXTRACTION_CONFIGURATION_FILENAME`    | `"extraction_configuration.yaml"`    | Default extraction config filename. |

---

## Functions

### write_microcontroller_manifest

```python
write_microcontroller_manifest(
    log_directory: Path,
    controller_id: int,
    controller_name: str,
    modules: tuple[ModuleSourceData, ...],
) -> None
```

Writes or appends a controller entry to the manifest file. Called automatically by
MicroControllerInterface.__init__().

### create_extraction_config

```python
create_extraction_config(manifest_path: Path) -> ExtractionConfig
```

Generates a precursor extraction config from a manifest with all controllers and modules populated
but with empty event codes that must be filled by the user.

### run_log_processing_pipeline

```python
run_log_processing_pipeline(
    log_directory: Path,
    output_directory: Path,
    config: Path,
    job_id: str | None = None,
    *,
    workers: int = -1,
    display_progress: bool = True,
) -> None
```

Processes log archives from a single DataLogger output directory using the extraction config. Controller
IDs to process are resolved from the config. Prefer MCP batch tools for multi-archive processing.

---

## Dependencies

- Python >=3.12, <3.15
- numpy, polars, paho-mqtt, click, httpx
- ataraxis-time, ataraxis-base-utilities, ataraxis-data-structures, ataraxis-transport-layer-pc
- mcp (for MCP server)
