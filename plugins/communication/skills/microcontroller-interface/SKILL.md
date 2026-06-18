---
name: microcontroller-interface
description: >-
  Guides creation and configuration of MicroControllerInterface, ModuleInterface, and
  MQTTCommunication instances for microcontroller communication. Covers interface initialization
  and lifecycle, ModuleInterface subclassing with command and parameter sending, MQTTCommunication
  setup, controller ID allocation, and DataLogger integration. Use when writing code that creates
  MicroControllerInterface or MQTTCommunication instances or needs the ataraxis-communication-interface API.
user-invocable: false
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
- Controller ID allocation and naming conventions
- DataLogger integration requirements

**Does not cover:**
- Firmware-side Module subclassing, command handlers, parameter structs, or main.cpp
  integration (see `/microcontroller:firmware-module`)
- Microcontroller discovery, MQTT testing, or manifest management via MCP tools (see `/microcontroller-setup`)
- Extraction configuration management (see `/extraction-configuration`)
- MCP server connectivity issues (see `/communication-mcp-environment-setup`)
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

The current version is **6.0.1**. If a version mismatch exists, ask the user how to proceed.

### Step 2: API verification

| File                                                                                                    | What to Check                                    |
|---------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| `../ataraxis-communication-interface/src/ataraxis_communication_interface/__init__.py`                  | Exported classes, functions, and public API      |
| `../ataraxis-communication-interface/src/ataraxis_communication_interface/microcontroller/interface.py` | MicroControllerInterface constructor and methods |
| `../ataraxis-communication-interface/src/ataraxis_communication_interface/communication/mqtt.py`        | MQTTCommunication API                            |
| Project `pyproject.toml`                                                                                | Current pinned version dependency                |

---

## API reference

See [references/api-reference.md](references/api-reference.md) for the complete API reference including:

- MicroControllerInterface constructor parameters and their exact types and defaults
- ModuleInterface constructor parameters and abstract methods
- MQTTCommunication constructor and lifecycle methods
- All public data classes (ModuleData, ModuleState, ModuleSourceData, MicroControllerSourceData)
- Configuration classes (MicroControllerManifest, ExtractionConfig hierarchy)
- Constants, utility functions, the message protocol, data payload types, the keepalive mechanism,
  DataLogger topology, and the MCP-to-code parameter bridge

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
- `port` is the device path from `list_microcontrollers_tool` (e.g., `/dev/ttyACM0`).
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
- `reset_controller()` resets the microcontroller to its default state
- `stop()` terminates the communication process and releases all resources

`start()` and `stop()` are idempotent (each is a no-op if the interface is already in the target state), and
`__del__` calls `stop()` plus the multiprocessing Manager shutdown, so a garbage-collected interface releases
its process automatically.

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

### Implementing process_received_data()

Route on `message.event` to handle different event codes. For `ModuleData` messages, access the
deserialized payload via `message.data_object`. For `ModuleState` messages, the event code itself
carries all information (no payload). Keep this method fast — it runs in the communication process
and blocks message reception while executing.

```python
def process_received_data(self, message: ModuleData | ModuleState) -> None:
    if message.event == np.uint8(51):       # kRotatedCCW from firmware
        delta: np.uint32 = message.data_object
        self._total_pulses -= int(delta)
    elif message.event == np.uint8(52):     # kRotatedCW from firmware
        delta: np.uint32 = message.data_object
        self._total_pulses += int(delta)
```

The type of `message.data_object` matches the numpy equivalent of the C++ type passed to
`SendData()` on the firmware side. For scalar values it is a numpy scalar (e.g., `np.uint32`);
for arrays it is an `NDArray` of the corresponding dtype.

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

`send_command()`, `send_parameters()`, and `reset_command_queue()` raise `RuntimeError` until the module
interface is passed to a `MicroControllerInterface`, which wires the input queue during `__init__()` (not
`start()`). In addition, `send_command()` and `send_parameters()` may only be called from the main runtime
process — calling them from the spawned communication worker process raises `RuntimeError`, because their
message-builder caches are stripped when the interface is pickled into that process (`reset_command_queue()`
works wherever the input queue is wired).

### Parameter struct correspondence

The `parameter_data` tuple must match the firmware's `PACKED_STRUCT` field-by-field — same count, same
order, same types. A mismatch silently corrupts all subsequent fields because the struct is laid out
contiguously with no padding. See `/microcontroller:firmware-module` for the C++ struct side and the full
type mapping table.

```text
C++ struct (firmware)                    Python tuple (PC)
─���───────────────────────────────────    ─���───────────────────────────────────
struct CustomRuntimeParameters           send_parameters(parameter_data=(
{                                            np.uint32(2000000),   # on_duration
        uint32_t on_duration  = ...;         np.uint32(2000000),   # off_duration
        uint32_t off_duration = ...;         np.uint16(666),       # echo_value
        uint16_t echo_value   = ...;     ))
} PACKED_STRUCT parameters;
```

For maintainability, wrap `send_parameters()` in a typed convenience method with named arguments:

```python
def set_parameters(
    self,
    *,
    on_duration: np.uint32,
    off_duration: np.uint32,
    echo_value: np.uint16,
) -> None:
    self.send_parameters(parameter_data=(on_duration, off_duration, echo_value))
```

This makes call sites self-documenting and prevents positional argument ordering mistakes.

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
- `send_data(topic, payload=None)` publishes a payload (`str | bytes | bytearray | float | None`, where
  `None` publishes an empty message) to the specified MQTT topic; raises `ConnectionError` if not connected
- `get_data()` returns the next received `(topic, payload)` tuple or `None` if empty
- `has_data` property returns `True` if there are received messages waiting
- `disconnect()` releases the connection (also called automatically on garbage collection)

---

## Message protocol

See [references/api-reference.md](references/api-reference.md) for the message protocol and code tables.

---

## Supported data payload types

See [references/api-reference.md](references/api-reference.md) for data payload types and element counts.

---

## Keepalive mechanism

See [references/api-reference.md](references/api-reference.md) for the keepalive mechanism and debugging.

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

## Controller ID allocation

Each MicroControllerInterface instance requires a unique `controller_id` (`np.uint8`) for DataLogger
identification. A controller's `controller_id` IS its source ID at the DataLogger level (see
`/log-input-format`). The recommended allocation:

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

See [references/api-reference.md](references/api-reference.md) for DataLogger topology and lifecycle ordering.

---

## Bridge from MCP

See [references/api-reference.md](references/api-reference.md) for the MCP-to-code parameter mapping table.

---

## Troubleshooting

| Symptom                                 | Likely Cause                          | Resolution                                                    |
|-----------------------------------------|---------------------------------------|---------------------------------------------------------------|
| Communication process fails to start    | Wrong port or baudrate                | Verify with `list_microcontrollers_tool` MCP tool             |
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
| MQTT connection refused                 | Broker not running or wrong host/port | Verify broker with `check_mqtt_broker_tool` MCP tool          |

---

## Related skills

| Skill                                  | Relationship                                                                                                                                                      |
|----------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `/microcontroller:firmware-module`     | Firmware-side counterpart: use for C++ Module subclassing, command handlers, parameter structs, and main.cpp integration. Codes and parameter layouts must match. |
| `/microcontroller-setup`               | Covers MCP-based discovery, MQTT testing, and manifest management                                                                                                 |
| `/extraction-configuration`            | Downstream: configure extraction parameters before processing                                                                                                     |
| `/log-input-format`                    | Reference: documents archive format produced by this code                                                                                                         |
| `/log-processing`                      | Downstream: processes archives from MicroControllerInterface data                                                                                                 |
| `/log-processing-results`              | Downstream: analyzes output from processed archives                                                                                                               |
| `/pipeline`                            | Context: end-to-end orchestration and multi-controller planning                                                                                                   |
| `/communication-mcp-environment-setup` | Prerequisite: MCP server connectivity for API verification                                                                                                        |

---

## Verification checklist

```text
Microcontroller Interface:
- [ ] Verified ataraxis-communication-interface version matches requirements (>=6.0.0)
- [ ] Verified microcontrollers are discoverable using /microcontroller-setup workflow
- [ ] Allocated unique controller IDs in the 101-150 advised range
- [ ] DataLogger initialized and started before MicroControllerInterface creation
- [ ] ModuleInterface subclasses implement all three abstract methods
- [ ] Module type/id codes match firmware configuration (verify via /microcontroller:firmware-module)
- [ ] Command codes, event codes, and parameter layout match firmware counterpart
- [ ] stop() called on all MicroControllerInterface instances during shutdown
- [ ] assemble_log_archives() called after DataLogger.stop()
```
