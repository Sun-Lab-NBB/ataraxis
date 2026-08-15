# API reference

Complete API reference for ataraxis-communication-interface public classes, functions, and constants.

---

## Public API

The top-level `__all__` exports exactly these 42 names. Anything not listed here must be imported from a subpackage.

```python
from ataraxis_communication_interface import (
    # Interface classes
    MicroControllerInterface,
    ModuleInterface,
    MQTTCommunication,
    # Message classes
    ModuleData,
    ModuleState,
    # Firmware code mirrors (IntEnum)
    KernelCommandCodes,
    KernelStatusCodes,
    ModuleStatusCodes,
    CommunicationStatusCodes,
    TransportStatusCodes,
    # Manifest and extraction configuration
    MicroControllerManifest,
    MicroControllerSourceData,
    ModuleSourceData,
    ExtractionConfig,
    ControllerExtractionConfig,
    ModuleExtractionConfig,
    KernelExtractionConfig,
    create_extraction_config,
    write_microcontroller_manifest,
    # Extracted-table schema and access primitives
    ExtractedDataColumns,
    get_event_data,
    get_event_timestamps,
    partition_events,
    # Constants
    EXTRACTION_CONFIGURATION_FILENAME,
    MICROCONTROLLER_MANIFEST_FILENAME,
    MAXIMUM_CUSTOM_STATUS_CODE,
    MINIMUM_CUSTOM_STATUS_CODE,
    # Output path resolvers, documented in /log-processing-results
    find_kernel_paths,
    find_module_paths,
    parse_kernel_path,
    parse_module_path,
    resolve_kernel_path,
    resolve_module_path,
    # Orchestration, driven through the MCP tools and the axci CLI
    CONTROLLER_EXTRACTION_JOB_CORES,
    CONTROLLER_EXTRACTION_JOB_NAME,
    JobSizing,
    JobSource,
    JobUniverse,
    execute_job,
    resolve_jobs,
    run_log_processing_pipeline,
    size_archive_job,
)
```

### Subpackage map

Every subpackage declares its own `__all__`, and for the names it owns that list is wider than the curated top-level
one. Import a name through the package that exports it, never from the module file inside it.

| Import path                                        | Exports | Contents                                                                      |
|----------------------------------------------------|---------|-------------------------------------------------------------------------------|
| `ataraxis_communication_interface`                 | 42      | The curated public API listed above.                                          |
| `ataraxis_communication_interface.communication`   | 17      | MQTT and serial transports, all 12 message classes, protocol/prototype enums. |
| `ataraxis_communication_interface.microcontroller` | 30      | Interfaces, dataclasses, status-code mirrors, extraction and table access.    |
| `ataraxis_communication_interface.orchestration`   | 41      | Job identity, sizing, discovery, the single-job runner, and the batch engine. |
| `ataraxis_communication_interface.interfaces`      | 0       | CLI, MCP entry points, response machinery. `__all__` is empty by design.      |

**Note:** all 15 names the `.orchestration` subpackage contributes are re-exported at top level, but this plugin
deliberately does not document the job-scheduling contract they form. Orchestration runs through the MCP tools or
through the `axci` CLI a user invokes by hand, and there is no third path. Do not write code against these symbols.

Names a caller reaches for by symmetry that are **not** top level: `build_message_dataframe`, `evaluate_port`,
`extract_logged_microcontroller_data`, `ExtractedMessages`, `ExtractedModuleData`, and `ExtractedControllerData` (all
`.microcontroller`), plus `prepare_jobs`, `JobDescriptor`, `JobSet`, and `size_job` (all `.orchestration`). Importing
any of them from the top level raises `ImportError`.

---

## MicroControllerInterface

Manages bidirectional serial communication with a single microcontroller running the ataraxis-micro-controller firmware
library.

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
| `buffer_size`        | `int`                         | (required) | Serial buffer size in bytes (from manufacturer spec). Minimum 9. |
| `port`               | `str`                         | (required) | Serial port device path (e.g., `/dev/ttyACM0`, `COM3`).          |
| `name`               | `str`                         | (required) | Human-readable controller name. Written to manifest on init.     |
| `baudrate`           | `int`                         | `115200`   | UART baudrate. Ignored for USB serial.                           |
| `keepalive_interval` | `int`                         | `0`        | Keepalive interval in milliseconds. 0 disables keepalive.        |

`buffer_size` below 9 raises `TypeError` in `__init__()`, not later in the spawned process: 9 bytes is the width of the
TransportLayer packet preamble and postamble plus one payload byte. The value bounds only what the **PC transmits**.
Reception is bounded separately by the 254-byte COBS ceiling. It is the PC-side mirror of the firmware's
`kSerialBufferSize`, so a value larger than the board's real buffer produces parameter messages the controller cannot
receive. See `/microcontroller:firmware-module` for the per-board payload budgets that constant yields.

### Methods

| Method               | Returns | Description                                                                    |
|----------------------|---------|--------------------------------------------------------------------------------|
| `start()`            | `None`  | Spawns communication process, verifies identities, enters communication cycle. |
| `stop()`             | `None`  | Terminates communication process and releases all resources.                   |
| `reset_controller()` | `None`  | Resets the microcontroller to its default state.                               |

### Properties

| Property        | Type                          | Description                                             |
|-----------------|-------------------------------|---------------------------------------------------------|
| `controller_id` | `np.uint8`                    | The controller's unique identifier.                     |
| `name`          | `str`                         | Human-readable controller name.                         |
| `modules`       | `tuple[ModuleInterface, ...]` | All ModuleInterface instances bound to this controller. |

---

## ModuleInterface

Abstract base class for custom hardware module interfaces.

### Constructor

```python
ModuleInterface(
    module_type: np.uint8,
    module_id: np.uint8,
    name: str,
    error_codes: dict[np.uint8, str] | None = None,
    data_codes: set[np.uint8] | None = None,
)
```

| Parameter     | Type                          | Default    | Description                                                 |
|---------------|-------------------------------|------------|-------------------------------------------------------------|
| `module_type` | `np.uint8`                    | (required) | Module family code (1-255). Matches firmware.               |
| `module_id`   | `np.uint8`                    | (required) | Module instance ID (1-255). Matches firmware.               |
| `name`        | `str`                         | (required) | Human-readable name. Written to manifest.                   |
| `error_codes` | `dict[np.uint8, str] \| None` | `None`     | Event code to explanation map. Receipt raises RuntimeError. |
| `data_codes`  | `set[np.uint8] \| None`       | `None`     | Event codes routed to `process_received_data()`.            |

Every declared `error_codes` key and `data_codes` member must fall in the user range the "Event code ranges" section
below defines. A code outside that range raises `ValueError` at construction, and a non-dict `error_codes` raises
`TypeError`.

### Abstract methods

| Method                           | Args                        | Description                                                    |
|----------------------------------|-----------------------------|----------------------------------------------------------------|
| `initialize_remote_assets()`     | `None`                      | Initializes non-picklable resources for communication process. |
| `terminate_remote_assets()`      | `None`                      | Releases resources from `initialize_remote_assets()`.          |
| `process_received_data(message)` | `ModuleData \| ModuleState` | Handles `data_codes` messages. Must not sleep or block on I/O. |

### Command methods

| Method                  | Key Parameters                                                                                  | Description                                              |
|-------------------------|-------------------------------------------------------------------------------------------------|----------------------------------------------------------|
| `send_command()`        | `command: np.uint8, *, noblock: np.bool_, repetition_delay: np.uint32 = 0`                      | Sends command to module.                                 |
| `send_parameters()`     | `parameter_data: tuple[np.unsignedinteger \| np.signedinteger \| np.bool_ \| np.floating, ...]` | Sends parameters to module.                              |
| `reset_command_queue()` | `None`                                                                                          | Cancels the module's queued command.                     |
| `set_input_queue()`     | `input_queue: MPQueue`                                                                          | Wires the send path. Called by MicroControllerInterface. |

`set_input_queue()` is the mechanism behind the "not usable until passed to a MicroControllerInterface" rule:
`MicroControllerInterface.__init__()` calls it on every interface in `module_interfaces`, and until it runs the
`_input_queue` attribute is `None`, so all three command methods raise `RuntimeError`. Never call it yourself, an
interface wired to a queue no controller owns accepts messages that reach no microcontroller.

`send_command()` and `send_parameters()` additionally require their message-builder LRU caches, which `__getstate__`
strips to `None` when the interface is pickled. Both guards raise the same `RuntimeError`, whose text states the
contract: only the main runtime process may construct and send messages. See the SKILL.md note on start methods.

`reset_command_queue()` does not abort a running command, the active command finishes. It clears the module's one
pending slot and cancels any recurrence, and the firmware reports a completion message for a recurrent command that was
idle between repetitions.

### Properties

| Property      | Type                  | Description                                                        |
|---------------|-----------------------|--------------------------------------------------------------------|
| `module_type` | `np.uint8`            | Module family code.                                                |
| `module_id`   | `np.uint8`            | Module instance ID.                                                |
| `type_id`     | `np.uint16`           | Combined type+id as uint16: (type << 8) OR id.                     |
| `name`        | `str`                 | Human-readable module name.                                        |
| `data_codes`  | `set[np.uint8]`       | Event codes routed to `process_received_data()`.                   |
| `error_codes` | `dict[np.uint8, str]` | Event code to explanation map for codes that trigger RuntimeError. |

---

## MQTTCommunication

Provides bidirectional publish and subscribe messaging with the other clients connected to the same MQTT broker over
TCP. It is independent of `MicroControllerInterface`, which neither accepts nor constructs an instance, so routing MQTT
traffic to or from a microcontroller is the consuming application's work. Inside this library, only the `axci mqtt`
health check and the MQTT discovery tool construct it.

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

The instance listens to the `monitored_topics` it was built with for its whole lifetime, because no method adds a topic
afterwards. When the broker cannot be reached, confirm it is running on the expected host and port with
`check_mqtt_broker_tool` before suspecting the code (see `/microcontroller-setup`).

### Methods

| Method                           | Returns                     | Description                                                                                                                                             |
|----------------------------------|-----------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------|
| `connect()`                      | `None`                      | Connects to broker and subscribes to monitored topics. Raises `ConnectionError` if the broker cannot be reached, including on a failed name resolution. |
| `disconnect()`                   | `None`                      | Disconnects from broker. Called automatically on garbage collect.                                                                                       |
| `send_data(topic, payload=None)` | `None`                      | Publishes a payload (`str \| bytes \| bytearray \| float \| None`. `None` = empty message) to the topic. Raises `ConnectionError` if not connected.     |
| `get_data()`                     | `tuple[str, bytes] \| None` | Returns next received `(topic, payload)` or `None` if empty. Raises `ConnectionError` if not connected.                                                 |

### Properties

| Property   | Type   | Description                                    |
|------------|--------|------------------------------------------------|
| `has_data` | `bool` | `True` if there are received messages waiting. |

### Delivery and connection semantics

| Behavior                      | Contract                                                                                                   |
|-------------------------------|------------------------------------------------------------------------------------------------------------|
| QoS                           | Hardcoded to 0 on both publish and subscribe. At-most-once, no delivery guarantee, no retry.               |
| `monitored_topics=None`       | Subscribes to nothing and starts no listener thread. `get_data()` returns `None` forever.                  |
| Broker-side link loss         | The disconnect callback clears the tracked connection state, with no exception at that moment.             |
| Publish failure               | `send_data()` raises `ConnectionError` on any non-success publish return code, not just when disconnected. |
| `connect()` after a link loss | Not a no-op. It early-returns only while the tracked state says connected, so it reconnects.               |

The QoS choice assumes a loopback or LAN TCP socket. Never use `MQTTCommunication` as the sole transport for a command
that must not be lost. Acknowledge such a command over a second topic, or send it over the serial path instead.

---

## Data classes

### ModuleData

Received data message from a hardware module (protocol code 6). Properties are backed by an internal `message:
NDArray[np.uint8]` field. `data_object` holds the deserialized payload.

| Property         | Type            | Description                                                                  |
|------------------|-----------------|------------------------------------------------------------------------------|
| `module_type`    | `np.uint8`      | Module family code of the sending module.                                    |
| `module_id`      | `np.uint8`      | Instance ID of the sending module.                                           |
| `type_id`        | `np.uint16`     | Combined type and id of the sending module.                                  |
| `command`        | `np.uint8`      | Command the module was executing.                                            |
| `event`          | `np.uint8`      | Event code of the message.                                                   |
| `prototype_code` | `np.uint8`      | Prototype code identifying data layout.                                      |
| `data_object`    | `PrototypeType` | Deserialized payload: a numpy scalar (including `np.bool_`) or an `NDArray`. |

### ModuleState

Received state message from a hardware module (protocol code 8). Properties are backed by an internal `message:
NDArray[np.uint8]` field.

| Property      | Type        | Description                                 |
|---------------|-------------|---------------------------------------------|
| `module_type` | `np.uint8`  | Module family code of the sending module.   |
| `module_id`   | `np.uint8`  | Instance ID of the sending module.          |
| `type_id`     | `np.uint16` | Combined type and id of the sending module. |
| `command`     | `np.uint8`  | Command the module was executing.           |
| `event`       | `np.uint8`  | Event code of the message.                  |

### ModuleSourceData (frozen dataclass)

Per-module identification metadata for manifests.

| Field         | Type  | Description                 |
|---------------|-------|-----------------------------|
| `module_type` | `int` | Module family code.         |
| `module_id`   | `int` | Module instance ID.         |
| `name`        | `str` | Human-readable module name. |

### MicroControllerSourceData (frozen dataclass)

Per-controller identification metadata for manifests.

| Field     | Type                           | Description                     |
|-----------|--------------------------------|---------------------------------|
| `id`      | `int`                          | Controller ID.                  |
| `name`    | `str`                          | Human-readable controller name. |
| `modules` | `tuple[ModuleSourceData, ...]` | Managed hardware modules.       |

---

## Configuration classes

The two `YamlConfig` subclasses below inherit `to_yaml(file_path)` and the `from_yaml(file_path)` classmethod from it.

### MicroControllerManifest (YamlConfig)

Registry of controllers and their modules within a DataLogger output directory.

| Field         | Type                              | Description                    |
|---------------|-----------------------------------|--------------------------------|
| `controllers` | `list[MicroControllerSourceData]` | Registered controller entries. |

### ExtractionConfig (YamlConfig)

Configuration controlling which data is extracted from log archives.

| Field         | Type                               | Description                    |
|---------------|------------------------------------|--------------------------------|
| `controllers` | `list[ControllerExtractionConfig]` | Controller extraction entries. |

### ControllerExtractionConfig (frozen dataclass)

| Field           | Type                                 | Description                    |
|-----------------|--------------------------------------|--------------------------------|
| `controller_id` | `int`                                | Controller ID to extract from. |
| `modules`       | `tuple[ModuleExtractionConfig, ...]` | Module extraction entries.     |
| `kernel`        | `KernelExtractionConfig \| None`     | Optional kernel extraction.    |

### ModuleExtractionConfig (frozen dataclass)

| Field         | Type              | Description                         |
|---------------|-------------------|-------------------------------------|
| `module_type` | `int`             | Module family code.                 |
| `module_id`   | `int`             | Module instance ID.                 |
| `event_codes` | `tuple[int, ...]` | Event codes to extract (non-empty). |

### KernelExtractionConfig (frozen dataclass)

| Field         | Type              | Description                    |
|---------------|-------------------|--------------------------------|
| `event_codes` | `tuple[int, ...]` | Kernel event codes to extract. |

---

## Constants

| Constant                            | Value                             | Description                          |
|-------------------------------------|-----------------------------------|--------------------------------------|
| `MICROCONTROLLER_MANIFEST_FILENAME` | `"microcontroller_manifest.yaml"` | Manifest filename in log dirs.       |
| `EXTRACTION_CONFIGURATION_FILENAME` | `"extraction_configuration.yaml"` | Default extraction config filename.  |
| `MINIMUM_CUSTOM_STATUS_CODE`        | `51`                              | Lowest event code a module may use.  |
| `MAXIMUM_CUSTOM_STATUS_CODE`        | `250`                             | Highest event code a module may use. |

Two further top-level constants, `CONTROLLER_EXTRACTION_JOB_NAME` and `CONTROLLER_EXTRACTION_JOB_CORES`, belong to the
orchestration layer, which this plugin does not document. `/log-processing` names the declared per-job core allocation
where a reader needs it.

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

Writes or appends a controller entry to the manifest file. Called automatically by MicroControllerInterface.__init__().

### create_extraction_config

```python
create_extraction_config(manifest_path: Path) -> ExtractionConfig
```

Generates a precursor extraction config from a manifest with all controllers and modules populated but with empty event
codes that must be filled by the user.

### run_log_processing_pipeline

```python
run_log_processing_pipeline(
    log_directory: Path,
    output_directory: Path,
    config: Path,
    job_id: str | None = None,
    source_ids: Sequence[str] | None = None,
    *,
    workers: int = -1,
    display_progress: bool = True,
) -> None
```

Processes log archives from a single DataLogger output directory using the extraction config. Controller IDs to process
are resolved from the config, and `source_ids` narrows that set further in local mode (it is ignored when `job_id`
selects the work). Prefer MCP batch tools for multi-archive processing.

### Top-level functions documented elsewhere

| Function                                                                                                                         | Owning skill                       |
|----------------------------------------------------------------------------------------------------------------------------------|------------------------------------|
| `get_event_data`, `get_event_timestamps`, `partition_events`                                                                     | `/log-processing-results`          |
| `find_kernel_paths`, `find_module_paths`, `parse_kernel_path`, `parse_module_path`, `resolve_kernel_path`, `resolve_module_path` | `/log-processing-results`          |
| `size_archive_job`                                                                                                               | `/log-processing`                  |
| `execute_job`, `resolve_jobs`, `JobSource`, `JobUniverse`                                                                        | Not documented, use MCP or the CLI |

---

## Dependencies

- Python >=3.12, <3.15
- numpy, polars, paho-mqtt, pyserial, click, psutil, filelock
- ataraxis-time, ataraxis-base-utilities, ataraxis-data-structures, ataraxis-transport-layer-pc
- mcp (for MCP server)

---

## Firmware status code mirrors

Five top-level `IntEnum` classes mirror the firmware enumerations. Import them instead of hand-transcribing codes into
comparisons, dictionaries, or extraction configs, a literal drifts, a member does not.

| Enum                       | Values | Mirrors                               | Use for                                              |
|----------------------------|--------|---------------------------------------|------------------------------------------------------|
| `KernelCommandCodes`       | 0-5    | `kKernelCommands`                     | Naming the command a Kernel message was sent under.  |
| `KernelStatusCodes`        | 0-10   | `kKernelStatusCodes`                  | Kernel event codes, including the non-error members. |
| `ModuleStatusCodes`        | 0-3    | Module service codes                  | Module event codes 0-3, below the custom range.      |
| `CommunicationStatusCodes` | 51-62  | firmware `Communication` class        | Byte 1 of a reception/transmission error payload.    |
| `TransportStatusCodes`     | 11-29  | microcontroller-side `TransportLayer` | Byte 2 of a reception/transmission error payload.    |

The first three cover the **complete** firmware sets, not the error-only subsets the SKILL.md tables list. They include
the success members the runtime never raises on, `KernelStatusCodes.SETUP_COMPLETE` (1) and
`KernelStatusCodes.MODULE_PARAMETERS_SET` (6) on the Kernel side, `ModuleStatusCodes.COMMAND_COMPLETED` (2) on the
module side. Those three are the codes to select when building an extraction config that captures runtime progress
rather than faults.

### CommunicationStatusCodes and TransportStatusCodes

These two never arrive as event codes. The firmware attaches them as the two-byte payload of every Kernel
RECEPTION_ERROR (3) and TRANSMISSION_ERROR (4) message and every module TRANSMISSION_ERROR (1) message: byte 1 is the
`Communication` status, byte 2 is the `TransportLayer` status.

**axci 7.0.0 already decodes both bytes into the `RuntimeError` text it raises.** Read the raised message first. The two
enumerations are the decoder for the other path: the raw pair the extraction pipeline writes undecoded into the `data`
column of a processed kernel or module feather. Index byte 1 into `CommunicationStatusCodes` (51-62) and byte 2 into
`TransportStatusCodes` (11-29), which mirrors the microcontroller's own TransportLayer rather than the PC's. For the
firmware-side meaning of every member, and the corrective action each byte pair points at, see
`/microcontroller:firmware-module`.

---

## Message protocol

### Outgoing messages (PC → microcontroller)

| Protocol Code | Message Type          | Description                                               |
|---------------|-----------------------|-----------------------------------------------------------|
| 1             | RepeatedModuleCommand | Module command that executes recurrently at a cycle delay |
| 2             | OneOffModuleCommand   | Module command that executes once                         |
| 3             | DequeueModuleCommand  | Cancels a module's queued command                         |
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

### Protocol and prototype enums

The `.communication` subpackage exports the codes above as enums, plus the decoders that turn a prototype code into a
concrete numpy description.

```python
from ataraxis_communication_interface.communication import PrototypeType, SerialProtocols, SerialPrototypes
```

| Symbol                                      | Kind       | Description                                                             |
|---------------------------------------------|------------|-------------------------------------------------------------------------|
| `SerialProtocols`                           | `IntEnum`  | Protocol codes 0-12. Code 0 is `UNDEFINED` and never valid on the wire. |
| `SerialPrototypes`                          | `IntEnum`  | Prototype codes naming the dtype and element count of a data payload.   |
| `PrototypeType`                             | type alias | Union of the 11 numpy scalar types and their `NDArray` forms.           |
| `SerialProtocols.as_uint8()`                | method     | Returns the member as `np.uint8`, the type the message classes expect.  |
| `SerialPrototypes.get_prototype_for_code()` | static     | Code to a shared read-only prototype object, or `None` if unknown.      |
| `SerialPrototypes.get_byte_size_for_code()` | static     | Code to the payload byte size, or `None` if unknown.                    |
| `SerialPrototypes.get_dtype_for_code()`     | static     | Code to a numpy dtype string (`'float32'`), or `None` if unknown.       |

`get_dtype_for_code()` is the bridge between a logged message's `prototype_code` byte and the `dtype` column of a
processed feather. That column stores exactly this string, which is why a consumer can rebuild the payload with
`np.frombuffer(data, dtype=dtype_str)` without importing this library. A code the library does not recognize yields
`None` from all three decoders, and the extraction pipeline writes null into both the `dtype` and `data` columns for
that message.

**Note:** all three decoders return objects from a module-level table shared by every caller. Treat a returned prototype
as read-only and never write into it.

### Subpackage-only names

`ModuleData` and `ModuleState` are the only message classes exported at top level, because they are the only two a
`process_received_data()` implementation annotates against. The other ten, `ControllerIdentification`,
`DequeueModuleCommand`, `KernelCommand`, `KernelData`, `KernelState`, `ModuleIdentification`, `ModuleParameters`,
`OneOffModuleCommand`, `ReceptionCode`, `RepeatedModuleCommand`, plus `SerialCommunication`, `SerialProtocols`,
`SerialPrototypes`, and `PrototypeType`, are importable from `.communication` only.

`SerialCommunication` is internal: `MicroControllerInterface` owns the one instance, inside the spawned communication
process. If you read it anyway, note that `receive_message()` parses into a **reused instance attribute** and returns a
reference to it, so each call overwrites the previous message. Finish with a message before receiving the next one, and
copy anything you intend to keep.

### Event code ranges

| Range  | Owner  | Description                                                                    |
|--------|--------|--------------------------------------------------------------------------------|
| 0-50   | System | Reserved service codes for internal module status (errors, command completion) |
| 51-250 | User   | User-defined event codes for application-specific data and state messages      |

Event codes are unique within each module and within the kernel, so the same code always carries the same semantic
meaning regardless of which command was executing when the message was sent. The extraction pipeline and
`process_received_data()` both rely on this invariant.

Messages with event codes in the user range and matching a module's `data_codes` set are routed to
`process_received_data()`. Messages with event codes matching `error_codes` raise `RuntimeError` and abort the runtime.

---

## Supported data payload types

The firmware resolves prototype codes at compile time for all data transmitted via `SendData()`. The PC side
deserializes them into numpy values. The `data_object` field in `ModuleData` and the `dtype`/`data` columns in processed
feather files use numpy types from this table.

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
| `np.float64` | `double` *     | 8 bytes | 1-15, 16, 20, 24, 31                                                             |

An element count of 1 represents a scalar value. For arrays, `ModuleData.data_object` is a numpy array of the
corresponding dtype. `uint8` arrays have the densest count coverage and can serve as a generic bytes buffer for packed
structures.

**The counts above are the wire-protocol prototype codes, not a per-board guarantee.** The bytes a board can actually
carry are capped separately by its serial buffer, and only the Teensy budget reaches the 248-byte top of this table. The
Arduino Due and Arduino Mega budgets are smaller, for parameters as well as for data. A payload sized against this table
therefore compiles on Teensy and can fail the firmware's `static_assert` on another board. Get the per-board figures
from `/microcontroller:firmware-module`, and size `parameter_data` against the target board before writing the tuple.

\* **`np.float64` does not reach an AVR board such as the Arduino Mega by default.** The mismatch surfaces as a firmware
build failure rather than at runtime, so use `np.float32` / `float` on both sides for those boards. Teensy and Arduino
Due are unaffected, and `/microcontroller:firmware-module` owns the build-flag alternative.

---

## Keepalive mechanism

The keepalive system detects communication failures between the PC and the microcontroller during runtime. It is
optional and controlled by the `keepalive_interval` constructor parameter.

**How it works:**
1. When `keepalive_interval > 0`, the PC sends a KernelCommand (command code 5) to the microcontroller at the specified
   interval (in milliseconds)
2. The microcontroller's Kernel tracks the time since the last received keepalive message
3. If the microcontroller does not receive a keepalive within its own timeout, it performs an emergency reset (all
   modules return to default state) and reports error code 10 (KEEPALIVE_TIMEOUT) via a KernelData message
4. In the other direction, if the microcontroller does not return the keepalive acknowledgement (a ReceptionCode message
   with reception_code 255) within one `keepalive_interval`, the PC communication process issues a reset command and
   raises `RuntimeError`, terminating the interface. This PC-side failure is distinct from and additional to the
   microcontroller-reported error code 10

**The two sides do not use the same deadline.** The PC allows one `keepalive_interval` for the acknowledgement, while
the firmware Kernel **doubles** the interval it was constructed with to derive its own timeout, deliberately tolerating
a brief lapse. The microcontroller therefore holds hardware for roughly twice as long as the PC does before either side
reacts, so `keepalive_interval` is a PC-side deadline and a firmware-side half-deadline. Size the interval so that twice
its value is still an acceptable time for a valve, solenoid, or motor to stay energized after the PC stops talking.

**The firmware watchdog is disarmed until the first keepalive arrives, and every reset disarms it again.** It arms on
the first `kKeepAlive` command the Kernel receives, not at boot, and the firmware's setup routine clears the armed flag.
Setup runs at boot, on a PC-sent reset, and after a keepalive-timeout emergency reset. Two consequences for the PC: the
window between `start()` and the first keepalive is unprotected, and `reset_controller()` leaves the controller
unprotected until the next keepalive re-arms it. Both windows close on their own within one `keepalive_interval`,
because the communication process keeps sending. The firmware side of this contract belongs to
`/microcontroller:firmware-module`.

**When to enable:**
- Enable keepalive for safety-critical hardware that must be reset if the PC loses communication (e.g., actuators,
  valves, motors)
- Disable (`keepalive_interval=0`) for passive sensors or when the microcontroller firmware does not implement keepalive
  handling

**Debugging keepalive issues:** the common causes are a stalled PC process, serial buffer overflow, USB disconnection,
and CPU contention preventing the communication process from sending keepalive messages on time.

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

All controllers sharing one logger get correlated timestamps, one archive assembly, and one processing batch.

### Coordinated lifecycle ordering

```text
Startup:  DataLogger.start() → MicroControllerInterface.__init__() → MicroControllerInterface.start()
Shutdown: MicroControllerInterface.stop() → DataLogger.stop() → assemble_log_archives()
```
