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

| Method               | Returns | Description                                                                    |
|----------------------|---------|--------------------------------------------------------------------------------|
| `start()`            | `None`  | Spawns communication process, verifies identities, enters communication cycle. |
| `stop()`             | `None`  | Terminates communication process and releases all resources.                   |
| `reset_controller()` | `None`  | Resets the microcontroller to its default state.                               |

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

| Parameter     | Type                    | Default    | Description                                      |
|---------------|-------------------------|------------|--------------------------------------------------|
| `module_type` | `np.uint8`              | (required) | Module family code (1-255). Matches firmware.    |
| `module_id`   | `np.uint8`              | (required) | Module instance ID (1-255). Matches firmware.    |
| `name`        | `str`                   | (required) | Human-readable name. Written to manifest.        |
| `error_codes` | `set[np.uint8] \| None` | `None`     | Event codes that trigger RuntimeError.           |
| `data_codes`  | `set[np.uint8] \| None` | `None`     | Event codes routed to `process_received_data()`. |

### Abstract methods

| Method                           | Args                        | Description                                                    |
|----------------------------------|-----------------------------|----------------------------------------------------------------|
| `initialize_remote_assets()`     | `None`                      | Initializes non-picklable resources for communication process. |
| `terminate_remote_assets()`      | `None`                      | Releases resources from `initialize_remote_assets`.            |
| `process_received_data(message)` | `ModuleData \| ModuleState` | Handles `data_codes` messages. Keeps fast.                     |

### Command methods

| Method                  | Key Parameters                                                                                  | Description                        |
|-------------------------|-------------------------------------------------------------------------------------------------|------------------------------------|
| `send_command()`        | `command: np.uint8, *, noblock: np.bool_, repetition_delay: np.uint32 = 0`                      | Sends command to module.           |
| `send_parameters()`     | `parameter_data: tuple[np.unsignedinteger \| np.signedinteger \| np.bool_ \| np.floating, ...]` | Sends parameters to module.        |
| `reset_command_queue()` | `None`                                                                                          | Clears the module's command queue. |

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

| Parameter          | Type                      | Default       | Description                            |
|--------------------|---------------------------|---------------|----------------------------------------|
| `ip`               | `str`                     | `"127.0.0.1"` | MQTT broker IP address.                |
| `port`             | `int`                     | `1883`        | MQTT broker socket port.               |
| `monitored_topics` | `tuple[str, ...] \| None` | `None`        | Topics to subscribe for incoming data. |

### Methods

| Method                           | Returns                     | Description                                                                                                                                         |
|----------------------------------|-----------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------|
| `connect()`                      | `None`                      | Connects to broker and subscribes to monitored topics.                                                                                              |
| `disconnect()`                   | `None`                      | Disconnects from broker. Called automatically on garbage collect.                                                                                   |
| `send_data(topic, payload=None)` | `None`                      | Publishes a payload (`str \| bytes \| bytearray \| float \| None`; `None` = empty message) to the topic. Raises `ConnectionError` if not connected. |
| `get_data()`                     | `tuple[str, bytes] \| None` | Returns next received `(topic, payload)` or `None` if empty.                                                                                        |

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

| Field         | Type                              | Description                    |
|---------------|-----------------------------------|--------------------------------|
| `controllers` | `list[MicroControllerSourceData]` | Registered controller entries. |

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
| `kernel`        | `KernelExtractionConfig \| None`     | Optional kernel extraction.    |

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

---

## Message protocol

All PC-microcontroller communication uses a structured message protocol with typed messages identified
by protocol codes. Understanding this protocol is essential for debugging communication issues.

### Outgoing messages (PC → microcontroller)

| Protocol Code | Message Type          | Description                                               |
|---------------|-----------------------|-----------------------------------------------------------|
| 1             | RepeatedModuleCommand | Module command that executes recurrently at a cycle delay |
| 2             | OneOffModuleCommand   | Module command that executes once                         |
| 3             | DequeueModuleCommand  | Removes all queued commands from a module                 |
| 4             | KernelCommand         | System-level command (reset, identify, keepalive)         |
| 5             | ModuleParameters      | Sets runtime parameters on a module                       |

### Incoming messages (microcontroller → PC)

| Protocol Code | Message Type             | Description                                                    |
|---------------|--------------------------|----------------------------------------------------------------|
| 6             | ModuleData               | Module event with a typed data payload (command, event, data)  |
| 7             | KernelData               | Kernel event with a typed data payload (command, event, data)  |
| 8             | ModuleState              | Module event without data (command, event only)                |
| 9             | KernelState              | Kernel event without data (command, event only)                |
| 10            | ReceptionCode            | Acknowledgement that the microcontroller received a PC message |
| 11            | ControllerIdentification | Microcontroller ID response during initialization              |
| 12            | ModuleIdentification     | Module type+id response during initialization                  |

### Event code ranges

| Range | Owner  | Description                                                                    |
|-------|--------|--------------------------------------------------------------------------------|
| 1-50  | System | Reserved service codes for internal module status (errors, command completion) |
| 51+   | User   | User-defined event codes for application-specific data and state messages      |

Event codes are unique within each module and within the kernel — the same code always carries the
same semantic meaning regardless of which command was executing when the message was sent. This means
event codes identify the *type* of event, not a command-specific response. The extraction pipeline
and `process_received_data()` both rely on this invariant.

Messages with event codes in the user range (51+) and matching a module's `data_codes` set are
routed to `process_received_data()`. Messages with event codes matching `error_codes` raise
`RuntimeError` and abort the runtime.

---

## Supported data payload types

The firmware resolves prototype codes at compile time for all data transmitted via `SendData()`. The
PC side deserializes them into numpy values. The `data_object` field in `ModuleData` and the
`dtype`/`data` columns in processed feather files use numpy types from this table.

| Numpy Type   | C++ Equivalent | Size    | Supported Element Counts                                                         |
|--------------|----------------|---------|----------------------------------------------------------------------------------|
| `np.bool_`   | `bool`         | 1 byte  | 1-15, 16, 24, 32, 40, 48, 52, 248                                                |
| `np.uint8`   | `uint8_t`      | 1 byte  | 1-15, 16, 18, 20, 22, 24, 28, 32, 36, 40, 44, 48, 52, 64, 96, 128, 192, 244, 248 |
| `np.int8`    | `int8_t`       | 1 byte  | 1-15, 16, 24, 32, 40, 48, 52, 92, 132, 172, 212, 244, 248                        |
| `np.uint16`  | `uint16_t`     | 2 bytes | 1-15, 16, 20, 24, 26, 32, 48, 64, 96, 122, 124                                   |
| `np.int16`   | `int16_t`      | 2 bytes | 1-15, 16, 20, 24, 26, 32, 48, 64, 96, 122, 124                                   |
| `np.uint32`  | `uint32_t`     | 4 bytes | 1-15, 16, 20, 24, 32, 48, 62                                                     |
| `np.int32`   | `int32_t`      | 4 bytes | 1-15, 16, 20, 24, 32, 48, 62                                                     |
| `np.float32` | `float`        | 4 bytes | 1-15, 16, 20, 24, 32, 48, 62                                                     |
| `np.uint64`  | `uint64_t`     | 8 bytes | 1-15, 16, 20, 24, 31                                                             |
| `np.int64`   | `int64_t`      | 8 bytes | 1-15, 16, 20, 24, 31                                                             |
| `np.float64` | `double`       | 8 bytes | 1-15, 16, 20, 24, 31                                                             |

An element count of 1 represents a scalar value. For arrays, `ModuleData.data_object` is a numpy
array of the corresponding dtype. The maximum payload is 248 bytes; array element counts are
constrained by `floor(248 / element_size)`. `uint8` arrays have the densest count coverage and can
serve as a generic bytes buffer for packed structures.

---

## Keepalive mechanism

The keepalive system detects communication failures between the PC and the microcontroller during
runtime. It is optional and controlled by the `keepalive_interval` constructor parameter.

**How it works:**
1. When `keepalive_interval > 0`, the PC sends a KernelCommand (command code 5) to the
   microcontroller at the specified interval (in milliseconds)
2. The microcontroller's Kernel tracks the time since the last received keepalive message
3. If the microcontroller does not receive a keepalive within its configured timeout, it performs
   an emergency reset (all modules return to default state) and reports error code 10
   (KEEPALIVE_TIMEOUT) via a KernelData message
4. In the other direction, if the microcontroller does not return the keepalive acknowledgement
   (a ReceptionCode message with reception_code 255) within one `keepalive_interval`, the PC
   communication process issues a reset command and raises `RuntimeError`, terminating the interface.
   This PC-side failure is distinct from and additional to the MCU-reported error code 10

**When to enable:**
- Enable keepalive for safety-critical hardware that must be reset if the PC loses communication
  (e.g., actuators, valves, motors)
- Disable (`keepalive_interval=0`) for passive sensors or when the microcontroller firmware does
  not implement keepalive handling

**Debugging keepalive issues:**
- Error code 10 with a timeout duration in the data payload indicates the microcontroller
  triggered an emergency reset due to missed keepalive messages
- Common causes: PC process stalled, serial buffer overflow, USB disconnection, CPU contention
  preventing the communication process from sending keepalive messages on time

---

## Bridge from MCP

### What MCP testing reveals for code

| MCP Discovery                                 | Informs Code Parameter        | How                       |
|-----------------------------------------------|-------------------------------|---------------------------|
| Device path from `list_microcontrollers_tool` | `port`                        | Pass device path directly |
| Microcontroller ID                            | `controller_id`               | Use as `np.uint8(id)`     |
| Baudrate used in discovery                    | `baudrate`                    | Same value                |
| MQTT broker from `check_mqtt_broker_tool`     | `MQTTCommunication(ip, port)` | Pass to MQTT constructor  |

---

## DataLogger topology

A single shared DataLogger is the preferred topology:

```python
logger = DataLogger(output_directory=session_directory, instance_name="session")
logger.start()

ctrl1 = MicroControllerInterface(controller_id=np.uint8(101), data_logger=logger, ...)
ctrl2 = MicroControllerInterface(controller_id=np.uint8(102), data_logger=logger, ...)
```

All controllers sharing one logger: correlated timestamps, single archive assembly, single processing batch.

### Coordinated lifecycle ordering

```text
Startup:  DataLogger.start() → MCI.__init__() → MCI.start()
Shutdown: MCI.stop() → DataLogger.stop() → assemble_log_archives()
```
