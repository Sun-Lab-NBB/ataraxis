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

Guides creation and configuration of MicroControllerInterface, ModuleInterface, and MQTTCommunication instances. This
skill covers the PC-side Python API, for the firmware-side C++ Module counterpart, use
`/microcontroller:firmware-module` instead. For interactive hardware discovery and MQTT broker testing via MCP tools,
use `/microcontroller-setup` instead. Overall system architecture (binding classes, configuration dataclasses, startup
orchestration) is the responsibility of the consuming library or application.

---

## Scope

**Covers:**
- MicroControllerInterface constructor parameters and lifecycle methods
- ModuleInterface abstract base class and subclassing pattern
- MQTTCommunication setup and lifecycle
- Controller ID allocation and naming conventions
- DataLogger integration requirements

**Does not cover:**
- Firmware-side Module subclassing, command handlers, parameter structs, or main.cpp integration (see
  `/microcontroller:firmware-module`)
- Microcontroller discovery, MQTT testing, or manifest management via MCP tools (see `/microcontroller-setup`)
- Extraction configuration management (see `/extraction-configuration`)
- Driving log-processing jobs from code. Orchestration runs through MCP or the `axci` CLI only
- MCP server connectivity issues (see `/communication-mcp-environment-setup`)
- Binding class design, configuration dataclasses, or system architecture (consumer's responsibility)

---

## Cross-plugin boundary: firmware vs. interface

| Concern                                 | Authority                          |
|-----------------------------------------|------------------------------------|
| C++ Module subclass, command handlers   | `/microcontroller:firmware-module` |
| Parameter structs (`PACKED_STRUCT`)     | `/microcontroller:firmware-module` |
| main.cpp wiring (Communication, Kernel) | `/microcontroller:firmware-module` |
| Python ModuleInterface subclass         | This skill                         |
| MicroControllerInterface lifecycle      | This skill                         |
| MQTTCommunication setup                 | This skill                         |

The two sides must agree on **module_type**, **module_id**, **command codes**, **event codes**, **parameter struct
layout** (field order, types, and sizes), and the **data payload prototype** of every firmware `SendData()` call. That
prototype is the call's numpy dtype and element count, which fixes the type of `message.data_object` on this side. When
implementing a new hardware module, always work both skills together: `/microcontroller:firmware-module` for the C++
firmware and this skill for the Python interface. If either side's codes, parameter layout, or payload prototype change,
the other must be updated to match.

**Module command code 0 is reserved.** The firmware uses it to mark the absence of a command, so a module command
message carrying code 0 is rejected before it is queued and answered with module event code 3 (COMMAND_NOT_RECOGNIZED).
On this side that arrival raises `RuntimeError` and terminates the communication process, so
`send_command(command=np.uint8(0), ...)` aborts the runtime instead of doing nothing. Number custom command codes from
1. See `/microcontroller:firmware-module` for the firmware-side rejection path.

---

## Verification requirements

### Step 1: Version verification

Check the locally installed ataraxis-communication-interface version against the latest release:

```bash
pip show ataraxis-communication-interface
```

The current version is **7.0.0**. If a version mismatch exists, ask the user how to proceed.

### Step 2: API verification

| File                                                                                                    | What to Check                                    |
|---------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| `../ataraxis-communication-interface/src/ataraxis_communication_interface/__init__.py`                  | Exported classes, functions, and public API      |
| `../ataraxis-communication-interface/src/ataraxis_communication_interface/microcontroller/interface.py` | MicroControllerInterface constructor and methods |
| `../ataraxis-communication-interface/src/ataraxis_communication_interface/communication/mqtt.py`        | MQTTCommunication API                            |
| Project `pyproject.toml`                                                                                | Current pinned version dependency                |

### Step 3: Firmware-side verification

The PC side cannot be verified alone, three constructor arguments are copies of firmware build settings. Read these two
files in the ataraxis-micro-controller checkout the target board was flashed from:

| File             | What to check                                                                                         |
|------------------|-------------------------------------------------------------------------------------------------------|
| `library.json`   | `version` (the axmc release, currently **4.0.2**) and `platforms` (`atmelavr`, `atmelsam`, `teensy`). |
| `platformio.ini` | The target environment's `monitor_speed`, which the firmware passes to `Serial.begin()`.              |

**`baudrate` must equal the flashed board's `monitor_speed`, and 115200 is not a universal default.** A board whose
`monitor_speed` differs never answers identification when the default is left in place, which surfaces as an
initialization timeout rather than as a configuration error, so suspect the baudrate before the module code. The
per-environment rates live in `/microcontroller:firmware-module`, "Serial speed per board environment". If a project
ships its own firmware `platformio.ini`, read that file instead of the library's.

The two library versions are independent (axci 7.0.0 pairs with axmc 4.0.2), so they cannot be compared numerically.
Record both, and treat an incompatible pair as the first suspect behind error code 5 (INVALID_MESSAGE_PROTOCOL), which
surfaces at runtime rather than at build time. See `/microcontroller:firmware-module` for the firmware build.

---

## API reference

See [references/api-reference.md](references/api-reference.md) for the complete API reference including:

- The full 41-name top-level `__all__`, the per-subpackage export map, and which names are subpackage-only
- MicroControllerInterface constructor parameters and their exact types and defaults
- ModuleInterface constructor parameters, abstract methods, and `set_input_queue()`
- MQTTCommunication constructor, lifecycle methods, and its delivery and connection semantics
- All public data classes (ModuleData, ModuleState, ModuleSourceData, MicroControllerSourceData)
- Configuration classes (MicroControllerManifest, ExtractionConfig hierarchy)
- The five firmware status-code mirror IntEnums, including the two that decode error payload bytes
- `SerialProtocols`, `SerialPrototypes`, `PrototypeType`, and the prototype-code decoders
- Constants, utility functions, the message protocol, data payload types, the keepalive mechanism, DataLogger topology,
  and the MCP-to-code parameter bridge

**Import before transcribing.** The error tables in this file are hand-written subsets. The api-reference names the
`IntEnum` that carries the complete set. Prefer `KernelStatusCodes.KEEPALIVE_TIMEOUT` over the literal 10.

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
- `controller_id` is a `np.uint8` that uniquely identifies this controller within the DataLogger. Allocate it from the
  range the "Controller ID allocation" section below advises.
- `baudrate` is ignored by USB-interface devices and matters only on UART boards, where the Step 3 `monitor_speed` rule
  fixes its value.
- `keepalive_interval` is in milliseconds. Set to 0 to disable keepalive messaging.

### Lifecycle

```text
MicroControllerInterface() → start() → [communication active] → stop()
```

- `__init__()` writes the manifest entry, configures module interfaces, sets up the communication process
- `start()` spawns the communication process, verifies controller and module identities, enters the communication cycle
- `reset_controller()` resets the microcontroller to its default state
- `stop()` terminates the communication process and releases all resources

**Note:** `reset_controller()` is a PC-sent kernel command, so it only works while the controller is still executing its
runtime cycle. A controller that already reported error code 2 (MODULE_SETUP_ERROR) never reads PC messages again, so
neither `reset_controller()` nor a `stop()`/`start()` cycle recovers it. The firmware must be re-uploaded, and every
managed module holds the pin levels it had at the instant of the failure until then. See
`/microcontroller:firmware-module` for the firmware-side suspension.

`start()` and `stop()` are idempotent (each is a no-op if the interface is already in the target state), and `__del__`
calls `stop()` plus the multiprocessing Manager shutdown, so a garbage-collected interface releases its process
automatically.

After `stop()`, the DataLogger can be stopped and archives assembled.

### Properties

| Property        | Type                          | Description                                            |
|-----------------|-------------------------------|--------------------------------------------------------|
| `controller_id` | `np.uint8`                    | The unique identifier of the managed controller        |
| `name`          | `str`                         | Human-readable controller name                         |
| `modules`       | `tuple[ModuleInterface, ...]` | All ModuleInterface instances bound to this controller |

### Runtime architecture

MicroControllerInterface owns the serial communication process but does not expose command or data methods directly. All
runtime interaction, sending commands, sending parameters, dequeuing commands, and receiving data, flows through the
`ModuleInterface` instances passed during initialization.

### Manifest auto-write

During `__init__()`, MicroControllerInterface calls `write_microcontroller_manifest()` to register this controller and
its modules in the DataLogger output directory. This manifest is required for downstream discovery via
`discover_microcontroller_data_tool`.

---

## ModuleInterface subclassing

Every custom hardware module interface must inherit from `ModuleInterface` and implement the three abstract methods.

### Constructor

```python
class EncoderInterface(ModuleInterface):
    def __init__(self) -> None:
        super().__init__(
            module_type=np.uint8(1),
            module_id=np.uint8(1),
            name="encoder",
            error_codes={
                np.uint8(53): "Encoder overflowed its counter.",
                np.uint8(54): "Encoder wiring fault detected.",
            },
            data_codes={np.uint8(51), np.uint8(52)},
        )
```

`error_codes` is a mapping from each error event code to the explanation surfaced when that error arrives, while
`data_codes` is a plain set. The two containers are deliberately different, and passing a set for `error_codes` raises
`TypeError` at construction.

Both `error_codes` and `data_codes` MUST be in the user range 51-250 (`MINIMUM_CUSTOM_STATUS_CODE` through
`MAXIMUM_CUSTOM_STATUS_CODE`, both inclusive). A declared code outside that range raises `ValueError` at construction.
Event codes 0-50 are intercepted as system service messages before per-module routing, so codes in that range never
reach the custom error or data paths.

| Parameter     | Type                          | Default    | Description                                                |
|---------------|-------------------------------|------------|------------------------------------------------------------|
| `module_type` | `np.uint8`                    | (required) | Hardware module family code (matches firmware)             |
| `module_id`   | `np.uint8`                    | (required) | Specific module instance ID (matches firmware)             |
| `name`        | `str`                         | (required) | Human-readable name (written to manifest)                  |
| `error_codes` | `dict[np.uint8, str] \| None` | `None`     | Event code to explanation map. Receipt raises RuntimeError |
| `data_codes`  | `set[np.uint8] \| None`       | `None`     | Event codes routed to `process_received_data()`            |

### Remote asset lifecycle

`initialize_remote_assets()` runs inside the communication process, so an asset **created** there exists only in that
process and the main process can never read it. Create in `__init__()`, connect in `initialize_remote_assets()`.

| Step    | Where                        | What belongs there                                                                |
|---------|------------------------------|-----------------------------------------------------------------------------------|
| Create  | `__init__()`                 | `SharedMemoryArray.create_array()` and anything the main process must also reach. |
| Connect | `initialize_remote_assets()` | `connect()` on that array, `PrecisionTimer` construction, anything non-picklable. |
| Release | `terminate_remote_assets()`  | `disconnect()` and close handles, exactly what the connect step claimed.          |

`examples/example_interface.py` in the axci repository is the reference implementation: it calls
`SharedMemoryArray.create_array(..., exists_ok=True)` in `__init__()`, `connect()` in `initialize_remote_assets()`, and
`disconnect()` in `terminate_remote_assets()`. Read it and `examples/example_runtime.py` before writing a new interface.

### Runtime script structure

**Guard the whole runtime with `if __name__ == "__main__":`.** MicroControllerInterface spawns a process, and under the
spawn and forkserver start methods the child re-imports the entry module, without the guard it re-executes the runtime
instead of only defining it. `example_runtime.py` places the guard near the top of the file and keeps everything below
it. Wrap the started interface in `try` / `finally` and put `stop()`, `DataLogger.stop()`, and `assemble_log_archives()`
in the `finally` block, in that order, so the failure path also drains the logger queue before the archive is assembled.

### Implementing process_received_data()

Route on `message.event` to handle different event codes. For `ModuleData` messages, access the deserialized payload via
`message.data_object`. For `ModuleState` messages, the event code itself carries all information (no payload). This
method runs on the reception path, so its body must not sleep, block on file or network I/O, or wait on a lock. Push
anything heavier onto the main process through a shared asset instead.

```python
def process_received_data(self, message: ModuleData | ModuleState) -> None:
    if message.event == np.uint8(51):       # kRotatedCCW from firmware
        delta: np.uint32 = message.data_object
        self._total_pulses -= int(delta)
    elif message.event == np.uint8(52):     # kRotatedCW from firmware
        delta: np.uint32 = message.data_object
        self._total_pulses += int(delta)
```

The type of `message.data_object` matches the numpy equivalent of the C++ type passed to `SendData()` on the firmware
side. For scalar values it is a numpy scalar (e.g., `np.uint32`). For arrays it is an `NDArray` of the corresponding
dtype.

**Pick the payload prototype while co-designing the module, before either side is written.** The firmware resolves the
prototype code from the C++ type at compile time, so the pairing is fixed by the `SendData()` call and cannot be
renegotiated at runtime. Only certain (type, element count) pairs have a prototype code at all, and the reachable counts
shrink on non-Teensy boards. Read the payload-type matrix in [references/api-reference.md](references/api-reference.md)
with the target board in hand, then annotate `data_object` against the type it fixes.

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

`send_command()`, `send_parameters()`, and `reset_command_queue()` raise `RuntimeError` until the module interface is
passed to a `MicroControllerInterface`, which wires the input queue during `__init__()` (not `start()`).

**`send_command()` and `send_parameters()` may only be called from the main runtime process.** Whether a violation is
*caught* depends on the multiprocessing start method. The guard fires on the message-builder caches, which
`__getstate__` strips only when the interface is pickled into the child. Under spawn (Windows and macOS) and under
forkserver (the Python 3.14 default on Linux) the caches are gone, so the call raises `RuntimeError`. Under fork (the
Python 3.12 and 3.13 default on Linux) the caches are inherited intact and the same call silently succeeds. axci
supports both regimes, since it declares `requires-python >=3.12,<3.15`. Treat the rule as a contract the library states
in its own error text, never as an exception you can rely on to surface the mistake. (`reset_command_queue()` works
wherever the input queue is wired.)

**`noblock` is a controller-wide decision, not a per-module one.** With `noblock=np.bool_(False)` the firmware blocks
its entire runtime cycle inside that command's stage delay. Nothing else on that controller advances meanwhile: every
sibling module stalls, PC messages go unread, and the keepalive check is deferred with them, so a long enough block
trips KEEPALIVE_TIMEOUT (error code 10) and forces an emergency reset. Pass `np.bool_(True)` unless the command must
hold the controller for its whole duration. See `/microcontroller:firmware-module` for the blocking-mode contract.

**The firmware holds exactly one pending command per module.** A module has a single pending slot, so a second
`send_command()` arriving before the module activates the first overwrites it and the first never runs, silently, on
both sides. The currently *active* command still finishes. It is the pending slot that is lost.

There is no online completion signal to wait on: the firmware's `kCommandCompleted` is module event code 2, inside the
system-reserved 0-50 range, so it is logged but never reaches `process_received_data()`. Either express repetition with
`repetition_delay` instead of a burst of one-off sends. The alternative is to have the firmware emit a custom event code
in the 51-250 range at the end of the handler and declare that code in `data_codes`, which gives the PC a completion
callback it can pace against. `/microcontroller:firmware-module` documents the queue's priority chain.

### Parameter struct correspondence

The `parameter_data` tuple must match the firmware's `PACKED_STRUCT` field-by-field, same count, same order, same types.
A mismatch silently corrupts all subsequent fields because the struct is laid out contiguously with no padding. See
`/microcontroller:firmware-module` for the C++ struct side and the full type mapping table.

A struct carrying two or more fields earns a keyword-only wrapper around `send_parameters()`, one argument per field,
because two same-typed neighbors swap places in a positional tuple without raising anything on either side.

---

## Error handling

Kernel errors and module service errors raise `RuntimeError` on the PC side and terminate the communication process.

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

Module messages with event codes 0-50 are system-reserved service messages:

| Event Code | Name                   | Meaning                                                                      | Data Payload                           |
|------------|------------------------|------------------------------------------------------------------------------|----------------------------------------|
| 0          | STANDBY                | Reserved placeholder. Never transmitted. An arriving 0 aborts the runtime    | (none)                                 |
| 1          | TRANSMISSION_ERROR     | Module failed to send data to the PC                                         | communication_status, transport_status |
| 2          | COMMAND_COMPLETED      | Module finished executing its last active command (informational, not error) | (none)                                 |
| 3          | COMMAND_NOT_RECOGNIZED | Module received an unknown command code                                      | (none)                                 |

### User-defined error handling

In addition to the system errors above, `ModuleInterface` subclasses can define custom error codes via the `error_codes`
constructor parameter. When a message arrives with an event code in that mapping, the communication process raises
`RuntimeError` carrying the registered explanation, and terminates.

### Watchdog thread

MicroControllerInterface runs a watchdog thread that monitors the communication process at 20 ms intervals. If the
communication process terminates unexpectedly (e.g., due to an unhandled exception or serial disconnection), the
watchdog detects the process death, cleans up resources, and raises `RuntimeError` inside its own daemon thread. The
traceback therefore surfaces on stderr while the main process keeps running. Main-thread code cannot catch this error.

### Initialization verification errors

During `start()`, the interface verifies the microcontroller configuration before entering the communication cycle.
Verification fails if:
- The microcontroller does not respond within 2 seconds (3 attempts max)
- The reported controller ID does not match the expected `controller_id`
- The microcontroller does not respond to the module identification request at all
- The microcontroller reports two modules sharing the same type+id combination
- The microcontroller has fewer modules than the number of ModuleInterface instances
- Any ModuleInterface's type+id pair has no matching module on the microcontroller
- The overall initialization exceeds 30 seconds

---

## Controller ID allocation

Each MicroControllerInterface instance requires a unique `controller_id` (`np.uint8`, valid range 1-255), which IS its
source ID at the DataLogger level (see `/log-input-format`). Source IDs must be unique across **all** sources sharing a
DataLogger, including sources from other libraries such as ataraxis-video-system. Allocate sequentially from 101 (101,
102, 103 for a 3-controller setup), because the advised 101-150 range avoids collisions with the ranges other libraries
advise. The convention is not enforced.

---

## Troubleshooting

| Symptom                                 | Likely Cause                                            | Resolution                                                                     |
|-----------------------------------------|---------------------------------------------------------|--------------------------------------------------------------------------------|
| Communication process fails to start    | Wrong port or baudrate                                  | Verify with `list_microcontrollers_tool` MCP tool                              |
| Controller ID mismatch on start         | Firmware uses different ID                              | Match controller_id to firmware configuration                                  |
| Module identification fails             | Module type/id mismatch                                 | Match ModuleInterface type/id to firmware modules                              |
| Process crashes on initialization       | DataLogger not started                                  | Start DataLogger before MicroControllerInterface initialization                |
| Serial permission denied                | User not in dialout group                               | Add user to dialout group or run with sudo                                     |
| Initialization timeout (>30s)           | Microcontroller not responding                          | Check serial connection, firmware loaded, correct port                         |
| Error code 2 (MODULE_SETUP_ERROR)       | Module Setup() failed on the microcontroller            | Firmware re-upload required. Check module hardware                             |
| Built-in LED solid HIGH after setup     | The microcontroller hit an error sending data to the PC | Check cable, hub, and buffer_size. See /microcontroller:firmware-module        |
| Built-in LED blinking at ~2 s intervals | Kernel setup failed, controller frozen                  | Unrecoverable from the PC. See /microcontroller:firmware-module                |
| Firmware will not compile for a Mega    | `double` payload on an AVR board                        | Use `np.float32` / `float` on both sides. See /microcontroller:firmware-module |
| Error code 3 (RECEPTION_ERROR)          | The microcontroller failed to parse a PC message        | Check serial cable, buffer_size, communication integrity                       |
| Error code 4 (TRANSMISSION_ERROR)       | The microcontroller failed to send to the PC            | Check serial cable, USB hub, buffer overflow                                   |
| Error code 5 (INVALID_MESSAGE_PROTOCOL) | Unknown protocol code sent                              | Verify library versions match between PC and firmware                          |
| Error code 7 (MODULE_PARAMETERS_ERROR)  | The microcontroller rejected the parameters             | Verify parameter count and types match firmware expectations                   |
| Error code 8 (COMMAND_NOT_RECOGNIZED)   | Unknown command code                                    | Verify command codes match firmware module implementation                      |
| Error code 9 (TARGET_MODULE_NOT_FOUND)  | Module type and id not on the microcontroller           | Verify module_type and module_id match firmware configuration                  |
| Error code 10 (KEEPALIVE_TIMEOUT)       | PC missed keepalive deadline                            | Check CPU load, serial bandwidth, reduce keepalive_interval                    |
| Watchdog: process prematurely shut down | Communication process crashed                           | Check serial connection, inspect stderr for stack trace                        |

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
| `/cli-reference`                       | Reference: the `axci` command surface, the only orchestration path besides MCP                                                                                    |
| `/pipeline`                            | Context: end-to-end orchestration and multi-controller planning                                                                                                   |
| `/communication-mcp-environment-setup` | Prerequisite: MCP server connectivity for API verification                                                                                                        |

---

## Verification checklist

```text
Microcontroller Interface, tool-settled (run `pip show ataraxis-communication-interface`):
- [ ] Verified ataraxis-communication-interface version matches requirements (>=7.0.0)

Microcontroller Interface, reader-judged:
- [ ] Read the firmware's library.json and platformio.ini per Step 3, and set baudrate from the board's monitor_speed
- [ ] Verified microcontrollers are discoverable using /microcontroller-setup workflow
- [ ] Allocated a unique controller_id per interface, from the range the Controller ID allocation section advises
- [ ] DataLogger initialized and started before MicroControllerInterface creation
- [ ] ModuleInterface subclasses implement all three abstract methods
- [ ] Module type/id codes match firmware configuration (verify via /microcontroller:firmware-module)
- [ ] Command codes, event codes, and parameter layout match firmware counterpart
- [ ] Custom command codes start at 1; code 0 is reserved and is rejected before it is queued
- [ ] Every SendData() payload prototype (numpy dtype and element count) is supported on the target board
- [ ] Remote assets are created in __init__() and only connected in initialize_remote_assets()
- [ ] Runtime script body is inside an `if __name__ == "__main__":` guard
- [ ] Comments and docstrings fill to 120 characters before wrapping, under the wrap-width rule /python-style defines
- [ ] stop() called on all MicroControllerInterface instances during shutdown
- [ ] assemble_log_archives() called after DataLogger.stop()
```
