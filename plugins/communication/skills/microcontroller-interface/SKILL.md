---
name: microcontroller-interface
description: >-
  Guides creation and configuration of MicroControllerInterface, ModuleInterface, and MQTTCommunication
  instances for microcontroller communication. Covers MicroControllerInterface initialization and
  lifecycle, ModuleInterface subclassing with command and parameter sending, MQTTCommunication setup,
  system ID allocation, and DataLogger integration. Use when writing code that creates
  MicroControllerInterface or MQTTCommunication instances or needs to understand the AXCI API.
user-invocable: true
---

# Microcontroller interface

Guides creation and configuration of MicroControllerInterface, ModuleInterface, and MQTTCommunication
instances. This skill covers the PC-side Python API; for the firmware-side C++ Module counterpart, use
`/microcontroller:firmware-module` instead. For interactive hardware discovery and MQTT broker testing via
MCP tools, use `/microcontroller-setup` instead. Overall system architecture (binding classes, configuration
dataclasses, startup orchestration) is the responsibility of the consuming library or application.

---

## Scope

**Covers:**
- MicroControllerInterface constructor parameters and lifecycle methods
- ModuleInterface abstract base class and subclassing pattern
- MQTTCommunication setup and lifecycle
- System ID allocation and naming conventions
- DataLogger integration requirements

**Does not cover:**
- Firmware-side Module subclassing, command handlers, parameter structs, or main.cpp
  integration (see `/microcontroller:firmware-module`)
- Microcontroller discovery, MQTT testing, or manifest management via MCP tools (see `/microcontroller-setup`)
- Extraction configuration management (see `/extraction-configuration`)
- MCP server connectivity issues (see `/mcp-environment-setup`)
- Binding class design, configuration dataclasses, or system architecture (consumer's responsibility)

---

## Cross-plugin boundary: firmware vs. interface

This skill and `/microcontroller:firmware-module` are counterparts that share a communication protocol
but live in different plugins with distinct responsibilities:

| Concern                                 | Authority                          |
|-----------------------------------------|------------------------------------|
| C++ Module subclass, command handlers   | `/microcontroller:firmware-module` |
| Parameter structs (`PACKED_STRUCT`)     | `/microcontroller:firmware-module` |
| main.cpp wiring (Communication, Kernel) | `/microcontroller:firmware-module` |
| Python ModuleInterface subclass         | This skill                         |
| MicroControllerInterface lifecycle      | This skill                         |
| MQTTCommunication setup                 | This skill                         |

The two sides must agree on **module_type**, **module_id**, **command codes**, **event codes**,
and **parameter struct layout** (field order, types, and sizes). When implementing a new hardware
module, always work both skills together: `/microcontroller:firmware-module` for the C++ firmware
and this skill for the Python interface. If either side's codes or parameter layout change, the
other must be updated to match.

---

## Verification requirements

**Before writing any microcontroller communication code, verify the current state of the library.**

### Step 1: Version verification

Check the locally installed ataraxis-communication-interface version against the latest release:

```bash
pip show ataraxis-communication-interface
```

The current version is **5.0.0**. If a version mismatch exists, ask the user how to proceed.

### Step 2: API verification

| File                                                                                                    | What to Check                                    |
|---------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| `../ataraxis-communication-interface/src/ataraxis_communication_interface/__init__.py`                  | Exported classes, functions, and public API      |
| `../ataraxis-communication-interface/src/ataraxis_communication_interface/microcontroller_interface.py` | MicroControllerInterface constructor and methods |
| `../ataraxis-communication-interface/src/ataraxis_communication_interface/communication.py`             | MQTTCommunication API                            |
| Project `pyproject.toml`                                                                                | Current pinned version dependency                |

---

## API reference

See [references/api-reference.md](references/api-reference.md) for the complete API reference including:

- MicroControllerInterface constructor parameters and their exact types and defaults
- ModuleInterface constructor parameters and abstract methods
- MQTTCommunication constructor and lifecycle methods
- All public data classes (ModuleData, ModuleState, ModuleSourceData, MicroControllerSourceData)
- Configuration classes (MicroControllerManifest, ExtractionConfig hierarchy)
- Constants and utility functions

---

## MicroControllerInterface usage

### Creating instances

```python
controller = MicroControllerInterface(
    controller_id=np.uint8(101),
    data_logger=data_logger,
    module_interfaces=(encoder_interface, sensor_interface),
    buffer_size=512,
    port="/dev/ttyACM0",
    name="teensy_main",
    baudrate=115200,
    keepalive_interval=0,
)
```

Key constructor notes:
- `controller_id` is a `np.uint8` (1-255) that uniquely identifies this controller within the DataLogger.
  Advised range: **101-150** for production microcontrollers. This range avoids collisions with other
  libraries sharing the same DataLogger. The convention is advised but not enforced.
- `data_logger` must be an initialized and started `DataLogger` instance. The MicroControllerInterface
  sends all communication messages to the logger via its multiprocessing input queue.
- `module_interfaces` must be a non-empty tuple of `ModuleInterface` subclass instances. Each module
  interface binds to the firmware module running on the microcontroller.
- `buffer_size` is the microcontroller's serial buffer size in bytes (from manufacturer spec).
- `port` is the device path from `list_microcontrollers` (e.g., `/dev/ttyACM0`).
- `name` is a required non-empty string written to `microcontroller_manifest.yaml` during `__init__()`,
  associating the `controller_id` with the human-readable name. The manifest enables
  `discover_microcontroller_data_tool` to identify controller-produced log archives.
- `baudrate` defaults to 115200. Only relevant for UART serial; ignored by USB devices.
- `keepalive_interval` is in milliseconds. Set to 0 to disable keepalive messaging.

### Lifecycle

```text
MicroControllerInterface() → start() → [communication active] → stop()
```

- `__init__()` writes the manifest entry, configures module interfaces, sets up the communication process
- `start()` spawns the communication process, verifies controller and module identities, enters the
  communication cycle
- `stop()` terminates the communication process and releases all resources

After `stop()`, the DataLogger can be stopped and archives assembled.

### Properties

| Property        | Type                          | Description                                            |
|-----------------|-------------------------------|--------------------------------------------------------|
| `controller_id` | `np.uint8`                    | The unique identifier of the managed controller        |
| `name`          | `str`                         | Human-readable controller name                         |
| `modules`       | `tuple[ModuleInterface, ...]` | All ModuleInterface instances bound to this controller |

### Runtime architecture

MicroControllerInterface owns the serial communication process but does not expose command or data
methods directly. All runtime interaction — sending commands, sending parameters, dequeuing commands,
and receiving data — flows through the `ModuleInterface` instances passed during initialization. The
controller manages the communication lifecycle; the modules define what is communicated.

```text
User code → ModuleInterface.send_command()    → input queue → communication process → microcontroller
User code → ModuleInterface.send_parameters() → input queue → communication process → microcontroller
Microcontroller → communication process → ModuleInterface.process_received_data() → user code
```

### Manifest auto-write

During `__init__()`, MicroControllerInterface calls `write_microcontroller_manifest()` to register this
controller and its modules in the DataLogger output directory. This manifest is required for downstream
discovery via `discover_microcontroller_data_tool`.

---

## ModuleInterface subclassing

Every custom hardware module interface must inherit from `ModuleInterface` and implement the three
abstract methods.

### Constructor

```python
class EncoderInterface(ModuleInterface):
    def __init__(self) -> None:
        super().__init__(
            module_type=np.uint8(1),
            module_id=np.uint8(1),
            name="encoder",
            error_codes={np.uint8(2), np.uint8(3)},
            data_codes={np.uint8(51), np.uint8(52)},
        )
```

| Parameter     | Type                   | Default   | Description                                      |
|---------------|------------------------|-----------|--------------------------------------------------|
| `module_type` | `np.uint8`             | --------- | Hardware module family code (matches firmware)   |
| `module_id`   | `np.uint8`             | --------- | Specific module instance ID (matches firmware)   |
| `name`        | `str`                  | --------- | Human-readable name (written to manifest)        |
| `error_codes` | `set[np.uint8] / None` | `None`    | Event codes that trigger RuntimeError on receipt |
| `data_codes`  | `set[np.uint8] / None` | `None`    | Event codes routed to `process_received_data()`  |

### Abstract methods to implement

```python
def initialize_remote_assets(self) -> None:
    """Called during communication process setup. Initialize non-picklable resources
    (PrecisionTimer, SharedMemoryArray, etc.) here."""

def terminate_remote_assets(self) -> None:
    """Called during communication process shutdown. Release resources initialized above."""

def process_received_data(self, message: ModuleData | ModuleState) -> None:
    """Called when a message with an event code from data_codes is received.
    Implement custom online data processing here. Keep fast — blocks communication."""
```

### Sending commands and parameters

```python
# Send a one-off command.
module.send_command(command=np.uint8(10), noblock=np.bool_(True))

# Send a repeated command (repetition_delay in microseconds).
module.send_command(command=np.uint8(10), noblock=np.bool_(True), repetition_delay=np.uint32(1000))

# Send parameters.
module.send_parameters(parameter_data=(np.uint16(500), np.float32(1.5)))

# Clear the module's command queue on the microcontroller.
module.reset_command_queue()
```

---

## MQTTCommunication usage

`MQTTCommunication` extends the serial microcontroller communication by connecting remote producers and
consumers to the microcontroller ecosystem over TCP. It is designed for tight integration with
`MicroControllerInterface` — for example, allowing a separate process or machine to send commands to or
receive data from microcontrollers via MQTT topics. It can be used standalone, but the library was
designed with this integrated usage in mind.

### Creating instances

```python
mqtt_client = MQTTCommunication(
    ip="127.0.0.1",
    port=1883,
    monitored_topics=("sensor/data", "control/commands"),
)
```

| Parameter          | Type                     | Default       | Description                              |
|--------------------|--------------------------|---------------|------------------------------------------|
| `ip`               | `str`                    | `"127.0.0.1"` | MQTT broker IP address                   |
| `port`             | `int`                    | `1883`        | MQTT broker socket port                  |
| `monitored_topics` | `tuple[str, ...] / None` | `None`        | Topics to subscribe to for incoming data |

### Lifecycle

```text
MQTTCommunication() → connect() → [publish/subscribe] → disconnect()
```

- `connect()` establishes the broker connection and subscribes to monitored topics
- `disconnect()` releases the connection (also called automatically on garbage collection)

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

## Error handling

Errors are reported through two channels: kernel errors (system-level) and module service errors
(per-module). Both raise `RuntimeError` on the PC side and terminate the communication process.

### Kernel errors

These are sent as KernelState or KernelData messages with the following event codes:

| Event Code | Name                     | Meaning                                                          | Data Payload                           |
|------------|--------------------------|------------------------------------------------------------------|----------------------------------------|
| 2          | MODULE_SETUP_ERROR       | A module's Setup() method failed on the microcontroller          | module_type, module_id                 |
| 3          | RECEPTION_ERROR          | Microcontroller failed to receive/parse a PC-sent message        | communication_status, transport_status |
| 4          | TRANSMISSION_ERROR       | Microcontroller failed to send data to the PC                    | communication_status, transport_status |
| 5          | INVALID_MESSAGE_PROTOCOL | Microcontroller received a message with an unknown protocol code | invalid_protocol_code                  |
| 7          | MODULE_PARAMETERS_ERROR  | Microcontroller failed to apply parameters to a module           | module_type, module_id                 |
| 8          | COMMAND_NOT_RECOGNIZED   | Microcontroller received an unknown kernel command code          | (none)                                 |
| 9          | TARGET_MODULE_NOT_FOUND  | A command/parameter message addressed a non-existent module      | module_type, module_id                 |
| 10         | KEEPALIVE_TIMEOUT        | Microcontroller did not receive a keepalive within expected time | timeout_ms                             |

### Module service errors

Module messages with event codes 1-50 are system-reserved service messages:

| Event Code | Name                   | Meaning                                                                      | Data Payload                           |
|------------|------------------------|------------------------------------------------------------------------------|----------------------------------------|
| 1          | TRANSMISSION_ERROR     | Module failed to send data to the PC                                         | communication_status, transport_status |
| 2          | COMMAND_COMPLETE       | Module finished executing its last active command (informational, not error) | (none)                                 |
| 3          | COMMAND_NOT_RECOGNIZED | Module received an unknown command code                                      | (none)                                 |

### User-defined error handling

In addition to the system errors above, `ModuleInterface` subclasses can define custom error codes
via the `error_codes` constructor parameter. When a message arrives with an event code in this set,
the communication process raises `RuntimeError` and terminates. This allows firmware-specific error
conditions to abort the PC runtime.

### Watchdog thread

MicroControllerInterface runs a watchdog thread that monitors the communication process at 20 ms
intervals. If the communication process terminates unexpectedly (e.g., due to an unhandled exception
or serial disconnection), the watchdog detects the process death, cleans up resources, and raises
`RuntimeError` on the main thread.

### Initialization verification errors

During `start()`, the interface verifies the microcontroller configuration before entering the
communication cycle. Verification fails if:
- The microcontroller does not respond within 2 seconds (3 attempts max)
- The reported controller ID does not match the expected `controller_id`
- The microcontroller has fewer modules than the number of ModuleInterface instances
- Any ModuleInterface's type+id pair has no matching module on the microcontroller
- The overall initialization exceeds 30 seconds

---

## System ID allocation

Each MicroControllerInterface instance requires a unique `controller_id` (`np.uint8`) for DataLogger
identification. The recommended allocation:

| Range   | Assignment                         | Notes                                             |
|---------|------------------------------------|---------------------------------------------------|
| 101-150 | MicroControllerInterface instances | Advised production range; not enforced            |
| 1-255   | Valid range                        | Any np.uint8 value; must be unique per DataLogger |

Allocate controller IDs sequentially starting at 101 (e.g., 101, 102, 103 for a 3-controller setup).
Source IDs must be unique across **all** sources sharing a DataLogger, including sources from other
libraries (e.g., ataraxis-video-system). The 101-150 range avoids collisions with other libraries'
advised ranges.

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

---

## Bridge from MCP

### What MCP testing reveals for code

| MCP Discovery                            | Informs Code Parameter        | How                       |
|------------------------------------------|-------------------------------|---------------------------|
| Device path from `list_microcontrollers` | `port`                        | Pass device path directly |
| Microcontroller ID                       | `controller_id`               | Use as `np.uint8(id)`     |
| Baudrate used in discovery               | `baudrate`                    | Same value                |
| MQTT broker from `check_mqtt_broker`     | `MQTTCommunication(ip, port)` | Pass to MQTT constructor  |

---

## Troubleshooting

| Symptom                                 | Likely Cause                          | Resolution                                                    |
|-----------------------------------------|---------------------------------------|---------------------------------------------------------------|
| Communication process fails to start    | Wrong port or baudrate                | Verify with `list_microcontrollers` MCP tool                  |
| Controller ID mismatch on start         | Firmware uses different ID            | Match controller_id to firmware configuration                 |
| Module identification fails             | Module type/id mismatch               | Match ModuleInterface type/id to firmware modules             |
| Process crashes on initialization       | DataLogger not started                | Start DataLogger before MCI initialization                    |
| Serial permission denied                | User not in dialout group             | Add user to dialout group or run with sudo                    |
| Initialization timeout (>30s)           | Microcontroller not responding        | Check serial connection, firmware loaded, correct port        |
| Error code 2 (MODULE_SETUP_ERROR)       | Module Setup() failed on MCU          | Firmware re-upload required; check module hardware            |
| Error code 3 (RECEPTION_ERROR)          | MCU failed to parse PC message        | Check serial cable, buffer_size, communication integrity      |
| Error code 4 (TRANSMISSION_ERROR)       | MCU failed to send to PC              | Check serial cable, USB hub, buffer overflow                  |
| Error code 5 (INVALID_MESSAGE_PROTOCOL) | Unknown protocol code sent            | Verify library versions match between PC and firmware         |
| Error code 7 (MODULE_PARAMETERS_ERROR)  | MCU rejected parameters               | Verify parameter count and types match firmware expectations  |
| Error code 8 (COMMAND_NOT_RECOGNIZED)   | Unknown command code                  | Verify command codes match firmware module implementation     |
| Error code 9 (TARGET_MODULE_NOT_FOUND)  | Module type+id not on MCU             | Verify module_type and module_id match firmware configuration |
| Error code 10 (KEEPALIVE_TIMEOUT)       | PC missed keepalive deadline          | Check CPU load, serial bandwidth, reduce keepalive_interval   |
| Watchdog: process prematurely shut down | Communication process crashed         | Check serial connection, inspect stderr for stack trace       |
| MQTT connection refused                 | Broker not running or wrong host/port | Verify broker with `check_mqtt_broker` MCP tool               |

---

## Related skills

| Skill                              | Relationship                                                                                                                                                      |
|------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `/microcontroller:firmware-module` | Firmware-side counterpart: use for C++ Module subclassing, command handlers, parameter structs, and main.cpp integration. Codes and parameter layouts must match. |
| `/microcontroller-setup`           | Covers MCP-based discovery, MQTT testing, and manifest management                                                                                                 |
| `/extraction-configuration`        | Downstream: configure extraction parameters before processing                                                                                                     |
| `/log-input-format`                | Reference: documents archive format produced by this code                                                                                                         |
| `/log-processing`                  | Downstream: processes archives from MicroControllerInterface data                                                                                                 |
| `/log-processing-results`          | Downstream: analyzes output from processed archives                                                                                                               |
| `/pipeline`                        | Context: end-to-end orchestration and multi-controller planning                                                                                                   |
| `/mcp-environment-setup`           | Prerequisite: MCP server connectivity for API verification                                                                                                        |

---

## Verification checklist

```text
Microcontroller Interface:
- [ ] Verified ataraxis-communication-interface version matches requirements (>=5.0.0)
- [ ] Verified microcontrollers are discoverable using /microcontroller-setup workflow
- [ ] Allocated unique controller IDs in the 101-150 advised range
- [ ] DataLogger initialized and started before MicroControllerInterface creation
- [ ] ModuleInterface subclasses implement all three abstract methods
- [ ] Module type/id codes match firmware configuration (verify via /microcontroller:firmware-module)
- [ ] Command codes, event codes, and parameter layout match firmware counterpart
- [ ] stop() called on all MicroControllerInterface instances during shutdown
- [ ] assemble_log_archives() called after DataLogger.stop()
```
