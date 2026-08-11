---
name: firmware-module
description: >-
  Guides creation of custom hardware Module subclasses in the ataraxis-micro-controller C++ firmware library. Covers
  SetupModule, SetCustomParameters, and RunActiveCommand implementation, stage-based command execution, PACKED_STRUCT
  parameter structures, event codes, SendData patterns, and main.cpp integration with Kernel and Communication. Use when
  writing or modifying firmware Module subclasses, or when implementing the microcontroller side of an
  ataraxis-communication-interface module.
user-invocable: false
---

# Firmware module

Guides implementation of custom hardware Module subclasses in the ataraxis-micro-controller C++ firmware library.

---

## Scope

**Covers:**
- Module base class inheritance and constructor pattern
- Three required virtual methods: SetupModule, SetCustomParameters, RunActiveCommand
- SetupModule design requirements: unfailable body, compile-time validation, and the idle-state guarantee
- Command handler patterns: immediate, multi-stage with non-blocking delay, sensor polling
- Runtime parameter structures with `PACKED_STRUCT` macro
- Event code conventions (system 0-50, user 51-250)
- Sending data to PC via `SendData()` overloads
- Template-based module design with compile-time pin configuration
- Static assertions for compile-time validation
- main.cpp integration: Communication, Module, and Kernel wiring
- Firmware build configuration: per-board serial speed, supported platforms, and the keepalive interval
- Firmware-side diagnosis of a failed transfer via the Communication and TransportLayer status code pair

**Does not cover:**
- PC-side ModuleInterface subclassing (see `/communication:microcontroller-interface`)
- Microcontroller discovery or MQTT testing via MCP tools (see `/communication:microcontroller-setup`)
- C++ coding conventions (see `/cpp-style`)
- PlatformIO project directory layout (see `/project-layout`)

---

## Cross-plugin boundary: firmware vs. interface

This skill and `/communication:microcontroller-interface` are counterparts that share a communication protocol but live
in different plugins with distinct responsibilities:

| Concern                                 | Authority                                  |
|-----------------------------------------|--------------------------------------------|
| C++ Module subclass, command handlers   | This skill                                 |
| Parameter structs (`PACKED_STRUCT`)     | This skill                                 |
| main.cpp wiring (Communication, Kernel) | This skill                                 |
| Python ModuleInterface subclass         | `/communication:microcontroller-interface` |
| MicroControllerInterface lifecycle      | `/communication:microcontroller-interface` |
| MQTTCommunication setup                 | `/communication:microcontroller-interface` |

The two sides must agree on **module_type**, **module_id**, **command codes**, **event codes**, and **parameter struct
layout** (field order, types, and sizes). When implementing a new hardware module, always work both skills together,
this skill for the C++ firmware and `/communication:microcontroller-interface` for the Python interface. If either
side's codes or parameter layout change, the other must be updated to match.

---

## Verification requirements

**Before writing any firmware module code, verify the current state of the library.**

### Step 1: Version verification

Check the locally available ataraxis-micro-controller version:

```bash
grep version ../ataraxis-micro-controller/library.json
```

The current version is **4.0.2**, which requires `ataraxis-transport-layer-mc` at `^4.0.1`. Check that pin in the
project's `platformio.ini` as well. If a version mismatch exists, ask the user how to proceed.

`library.json` is the single source of the C++ library version, and its `platforms` field names the three PlatformIO
platforms the firmware supports: `atmelavr`, `atmelsam`, and `teensy`. A board outside those platforms is unsupported,
so resolve the target board's platform before writing any board-conditional code. The same file excludes
`./src/main.cpp` from the published package, which is why the library's own main.cpp is a development harness that
mirrors `examples/module_integration.cpp` rather than shipped library code.

### Step 2: API verification

Read the source files to confirm the API has not changed since this skill was written:

| File                                                    | What to check                                     |
|---------------------------------------------------------|---------------------------------------------------|
| `../ataraxis-micro-controller/src/module.h`             | Module base class, virtual methods, utilities     |
| `../ataraxis-micro-controller/src/kernel.h`             | Kernel constructor and lifecycle                  |
| `../ataraxis-micro-controller/src/communication.h`      | Communication constructor                         |
| `../ataraxis-micro-controller/src/axmc_shared_assets.h` | Protocol codes, ResolvePrototype, message structs |
| `../ataraxis-micro-controller/platformio.ini`           | Board environments, per-board `monitor_speed`     |

---

## API reference

See [references/api-reference.md](references/api-reference.md) for the complete Module base class API including
constructor parameters, ExecutionControlParameters fields, the Kernel-facing public methods, all protected utility
method signatures, kCoreStatusCodes, the Kernel constructor and its status and command codes, and the Communication
constructor. It also documents the two-byte communication error payload and the two enumerations that fill it,
kCommunicationStatusCodes (51-62) and kTransportStatusCodes (11-29), with the firmware method that sets each code. The
same reference also holds the command handler patterns (immediate, multi-stage non-blocking delay, sensor polling) and
the optional implementation hints.

---

## Module class structure

Modules are header-only classes that inherit from `Module` and override three pure virtual methods. The template pattern
is recommended for compile-time hardware configuration:

```cpp
#ifndef CUSTOM_MODULE_H
#define CUSTOM_MODULE_H

#include <Arduino.h>
#include <module.h>

template <const uint8_t kPin>
class CustomModule final : public Module
{
        static_assert(
            kPin != LED_BUILTIN,
            "The LED-connected pin is reserved for error indication. Select a different pin."
        );

    public:
        // Constructor, parameters, virtual method overrides

    private:
        // Enums, command handler methods
};

#endif  // CUSTOM_MODULE_H
```

Header guards, include form, and include ordering follow `/cpp-style`.

### Constructor

You MUST call the base Module constructor with the type, id, and communication reference:

```cpp
CustomModule(
    const uint8_t module_type,
    const uint8_t module_id,
    Communication& communication
) : Module(module_type, module_id, communication) {}
```

- `module_type` identifies the module family. All instances of the same class share this code.
- `module_id` identifies the specific instance within the family. Must be unique per type.
- `communication` is the shared Communication instance created before any modules.

Also declare an overriding virtual destructor, `~CustomModule() override = default;`, because the Kernel manages modules
through `Module*` pointers, which makes the base virtual destructor load-bearing (see
[references/api-reference.md](references/api-reference.md)).

---

## Code definitions

### Command enum

Define commands with values starting at 1. Value 0 is reserved for "no active command":

```cpp
private:
    enum class kCommands : uint8_t
    {
        kPulse = 1,
        kEcho  = 2,
    };
```

### Custom event codes enum

Custom event codes MUST use values 51-250. Each event code MUST be unique within the module class and MUST carry the
same semantic meaning regardless of which command was executing when the message was sent. The extraction pipeline and
PC-side `process_received_data()` both rely on this invariant:

```cpp
private:
    enum class kStates : uint8_t
    {
        kHigh = 52,
        kLow  = 53,
        kEcho = 54,
    };
```

| Range  | Owner  | Description                                         |
|--------|--------|-----------------------------------------------------|
| 0      | System | Standby (module idle)                               |
| 1      | System | Transmission error                                  |
| 2      | System | Command completed                                   |
| 3      | System | Command not recognized                              |
| 4-50   | System | Reserved for future system use                      |
| 51-250 | User   | Custom event codes, unique within each module class |
| 251+   | System | Reserved, do not use                                |

### Runtime parameters structure

Parameter structs MUST use the `PACKED_STRUCT` macro to ensure correct binary serialization with the PC. The struct MUST
be at least one byte and MUST fit into the target board's payload, which is 250 bytes on Teensy, 246 bytes on Arduino
Due, and 54 bytes on Arduino Mega. A `static_assert` in `ExtractModuleParameters` enforces this per board, so a struct
that builds for Teensy can still fail to build for Mega. Field order and types must exactly match the PC-side
`send_parameters()` tuple:

```cpp
public:
    struct CustomRuntimeParameters
    {
            uint32_t on_duration  = 2000000;
            uint32_t off_duration = 2000000;
            uint16_t echo_value   = 666;
    } PACKED_STRUCT parameters;
```

| C++ type   | Size    | Numpy equivalent | Typical use               |
|------------|---------|------------------|---------------------------|
| `bool`     | 1 byte  | `np.bool_`       | Enable flags              |
| `uint8_t`  | 1 byte  | `np.uint8`       | Small counts, codes       |
| `uint16_t` | 2 bytes | `np.uint16`      | ADC values, medium counts |
| `uint32_t` | 4 bytes | `np.uint32`      | Microsecond durations     |
| `int32_t`  | 4 bytes | `np.int32`       | Signed large values       |
| `float`    | 4 bytes | `np.float32`     | Calibrated sensor values  |

**Cross-language correspondence:** The PC sends parameters as a numpy-typed tuple via `send_parameters()`, and each
tuple element maps to the struct field at the same position. The struct above corresponds to the tuple
`(np.uint32(2000000), np.uint32(2000000), np.uint16(666))`.

The C++ type of each field determines the required numpy dtype on the Python side. A mismatch that changes the struct's
total size, such as `np.uint16` sent for a `uint32_t` field, is caught at runtime. `ExtractParameters()` checks the
received payload against the expected size and returns `false` without writing anything, which the Kernel reports as
kernel status 7. A mismatch that preserves the total size, such as `np.int32` for a `uint32_t` field or two reordered
same-width fields, passes that check and silently corrupts the parsed values, because `PACKED_STRUCT` lays fields out
contiguously with no padding. Always verify field count, order, and types match across both sides.

---

## SetupModule()

Initialize hardware pins and reset parameters to defaults. This method is called by Kernel during `Setup()` and on
PC-requested resets. Deactivate the module's hardware first, then avoid blocking or fail-prone logic:

```cpp
bool SetupModule() override
{
    // Deactivates the managed hardware first, so the module is idle before anything else runs.
    pinMode(kPin, OUTPUT);
    digitalWrite(kPin, LOW);

    parameters.on_duration  = 2000000;
    parameters.off_duration = 2000000;
    parameters.echo_value   = 666;

    return true;
}
```

**Rules:**
- Configure all GPIO pins (`pinMode`, `digitalWriteFast`, etc.)
- Deactivate every managed output before anything else, so the module ends the method idle
- Set all parameter fields to their default values
- Return `true` on success, `false` on failure (failure bricks the controller until firmware reset)

On keepalive timeout the Kernel emits kernel status 10 (`kKeepAliveTimeout`) and re-runs `Setup()`, which clears every
module's command buffers and re-invokes its `SetupModule()`, so `SetupModule()` must be safe to call repeatedly. The
Kernel clears only the software command state, leaving the return of the hardware to its idle level as this method's
own job.

### Design requirements

A setup failure is unrecoverable at runtime, which makes `SetupModule()` the one virtual method whose failure modes you
MUST design out rather than merely report. Apply these three requirements in order.

**1. Write the method so that it cannot fail.** Its body should always return `true`. Restrict it to operations that
cannot fail: `pinMode` calls, pin state writes, and parameter default assignments. Do NOT place connection handshakes,
calibration routines, homing sweeps, or sensor readiness polling in this method. Move that logic into a command handler,
where a failure is reported to the PC as an event code and leaves the controller able to respond.

**2. When a failure mode is unavoidable, move it to compile time.** Validate pin assignments, template arguments,
parameter struct sizes, and every other statically known property with `static_assert` at the top of the class body, so
an invalid configuration fails `pio run` instead of bricking a controller in the field. A compilation error is always
recoverable, while a runtime setup failure needs physical access to the microcontroller.

**3. Leave the module in the idle state, and reach that state before anything that can fail.** Deactivate every hardware
asset the module manages, driving each output pin to the level that turns its attached device off, as the first
statement of the method body. Any step that can return `false` MUST come after that point.

Requirement 3 is the load-bearing one. `Kernel::Setup()` calls `SetupModule()` on each managed module in `modules[]`
array order and returns immediately on the first `false`, leaving its setup tracker inactive. `RuntimeCycle()` then only
blinks the built-in LED at ~2-second intervals: it never parses PC messages and never calls `RunModuleCommands()`. The
controller cannot receive the PC-sent `kResetController` command in that state, so nothing will change the hardware
state again until the firmware is reset. Whatever each module is driving at the instant of the failure, it keeps driving
indefinitely: a solenoid stays open, a valve keeps dispensing, a motor keeps turning.

The freeze hits every managed module, not only the one that failed:

| Module position           | State after a setup failure                                                         |
|---------------------------|-------------------------------------------------------------------------------------|
| Before the failing module | Already ran `SetupModule()`, so it is frozen where that method left it.             |
| The failing module        | Frozen part-way through its own `SetupModule()`, at the step that returned `false`. |
| After the failing module  | Never ran `SetupModule()` at all, so it holds the state it had before `Setup()`.    |

The last row is why this matters beyond first boot. `Setup()` also re-runs when the PC sends the reset kernel command
and when the keepalive monitor times out, both of which fire mid-experiment while modules can be actively driving
hardware. The Kernel calls `ResetExecutionParameters()` on every module before the setup pass, but that clears only
software command state and never touches pins, so each module's own `SetupModule()` is the only thing that can return
its hardware to a safe level. A module whose turn never comes keeps driving whatever its aborted command left active.

Idle is defined by the attached hardware, not by the pin level. Active-low and normally-closed devices idle at `HIGH`,
so never hardcode `LOW` as the idle level. Derive it from the polarity template parameter with the constexpr pin logic
pattern in [references/api-reference.md](references/api-reference.md), then write that constant in `SetupModule()`:

```cpp
template <const uint8_t kPin, const bool kNormallyClosed>
class ValveModule final : public Module
{
        static constexpr bool kOpen  = kNormallyClosed ? HIGH : LOW;
        static constexpr bool kClose = kNormallyClosed ? LOW  : HIGH;

    public:
        bool SetupModule() override
        {
            // Closes the valve first, which is this module's idle state, before anything else runs.
            pinMode(kPin, OUTPUT);
            digitalWriteFast(kPin, kClose);
            // ...
        }
};
```

---

## SetCustomParameters()

Extract the parameter struct from the PC message using the inherited `ExtractParameters()` wrapper:

```cpp
bool SetCustomParameters() override
{
    return ExtractParameters(parameters);
}
```

You MUST use `ExtractParameters()` (the Module base class wrapper), not `_communication.ExtractModuleParameters()`
directly. Post-processing of extracted values is permitted:

```cpp
bool SetCustomParameters() override
{
    if (ExtractParameters(parameters))
    {
        if (kOptionalPin == 255) parameters.optional_duration = 0;
        return true;
    }
    return false;
}
```

---

## RunActiveCommand()

Dispatch the active command to a handler method. Use `get_active_command()` to read the command code:

```cpp
bool RunActiveCommand() override
{
    switch (static_cast<kCommands>(get_active_command()))
    {
        case kCommands::kPulse: Pulse(); return true;
        case kCommands::kEcho:  Echo();  return true;
        default: return false;
    }
}
```

**Rules:**
- Cast `get_active_command()` to your command enum
- Each case calls a private handler method and returns `true`
- The `default` case returns `false` (triggers system error code 3: command not recognized)
- Do NOT evaluate whether the command ran successfully here, only whether it was recognized
- Do NOT call `CompleteCommand()` or `AbortCommand()` from the `default` case, because returning `false` is sufficient

Returning `false` makes the Kernel report error code 3 and then discard the active command, so an unrecognized command
clears itself after a single runtime cycle. The discard also drops the command from the queue when the queue still holds
it, because a code the module does not recognize cannot become recognized on a later repetition. A recurrent command
therefore reports the error once rather than on every repetition, and the module stays free to run whatever the PC
queues next.

---

## Command handler patterns

See [references/api-reference.md](references/api-reference.md) for the immediate, multi-stage non-blocking delay, and
sensor polling command handler patterns with full code examples.

### Recurrent vs one-off commands

The PC queues each command as either **one-off** (`kOneOffModuleCommand`) or **recurrent** (`kRepeatedModuleCommand`,
carrying a `cycle_delay`). Write handlers identically for both, always calling `CompleteCommand()` when the work is
done, but note the runtime difference. A recurrent command auto-reactivates at stage 1 after its `cycle_delay`, and
`CompleteCommand()` deliberately suppresses the `kCommandCompleted` (status 2) message for recurrent commands until they
are dequeued or replaced, while one-off commands always report completion. A recurrent command retired while idle
between repetitions still reports completion, sent by the dequeue or by the command that replaces it. Expect repeated
activation for recurrent commands and use the multi-stage non-blocking-delay and sensor-polling patterns to avoid
flooding the PC. The PC sets the recurrence interval via `send_command(repetition_delay=...)`, covered by
`/communication:microcontroller-interface`.

**Output-device note:** handlers that drive a pin with `digitalWrite(HIGH/LOW)` control level-driven (active) devices,
so an active buzzer sounds at HIGH and is silent at LOW, and a valve or LED switches on and off. A device that needs a
generated waveform, such as a **passive** piezo or a servo, needs a `tone()` or `analogWrite` (PWM) handler instead,
because `digitalWrite` alone leaves it silent or unmoving.

---

## Sending data to PC

### State-only (no data payload)

```cpp
SendData(static_cast<uint8_t>(kStates::kHigh));
```

Sends a ModuleState message (protocol 8) containing only the event code. Use when the event itself carries all needed
information.

### With typed data payload

```cpp
SendData(static_cast<uint8_t>(kStates::kEcho), parameters.echo_value);
```

Sends a ModuleData message (protocol 6) containing the event code and a typed data object. The prototype code for the
wire protocol is resolved automatically at compile time from the C++ type of the data argument, via the
`ResolvePrototype` function in `axmc_shared_assets.h`, so callers never specify a prototype code. Supported types: all
11 scalars (`bool` through `double`) and C-style arrays of those types at supported element counts. `uint8_t` arrays
offer the densest count coverage and can serve as a generic bytes buffer for sending arbitrary packed structures via
`uint8_t[sizeof(MyStruct)]`.

Each scalar type supports a type-specific set of array element counts, where a count of 1 is a scalar. Unsupported
(type, count) combinations trigger a compile-time `static_assert`. The transmittable size is capped separately by the
board's serial buffer, at 248 bytes on Teensy, 244 bytes on Arduino Due, and 52 bytes on Arduino Mega. Sizing a data
object for Teensy therefore does not guarantee it compiles for the other two. For the full supported data-type and
element-count table, see [api-reference.md](references/api-reference.md).

**Error handling:** If transmission fails, `SendData()` automatically attempts to send an error message and turns on the
built-in LED. Do not use the LED-connected pin in your module to avoid interference.

That error message is a ModuleData message carrying core event code 1 (`kTransmissionError`) and a two-byte payload,
`{Communication status, TransportLayer status}`. Those two bytes are the firmware's entire diagnosis of a failed
transfer, so read them as a pair. See the communication error payload section in
[references/api-reference.md](references/api-reference.md) for both code tables and the fix each pair points at.

---

## main.cpp integration

Follow this exact instantiation order: Communication, Module(s), Kernel.

```cpp
#include <Arduino.h>
#include <communication.h>
#include <kernel.h>
#include <module.h>
#include "custom_module.h"

static constexpr uint8_t kControllerID       = 222;
static constexpr uint32_t kKeepaliveInterval = 5000;
static constexpr uint32_t kSerialBaudRate    = 115200;  // Matches the teensy41 monitor_speed.

Communication axmc_communication(Serial);

CustomModule<5> module_1(1, 1, axmc_communication);
CustomModule<6> module_2(1, 2, axmc_communication);

Module* modules[] = {&module_1, &module_2};

Kernel axmc_kernel(kControllerID, axmc_communication, modules, kKeepaliveInterval);

void setup()
{
    Serial.begin(kSerialBaudRate);

#if !defined(__AVR__)
    analogReadResolution(12);
#endif

    axmc_kernel.Setup();
}

void loop()
{
    axmc_kernel.RuntimeCycle();
}
```

**Key points:**
- `kControllerID` must match the `controller_id` used on the PC side (1-255, unique per controller)
- `kKeepaliveInterval` is in milliseconds and 0 disables the mechanism. The Kernel doubles it to derive the effective
  timeout, saturating rather than wrapping, so 5000 ms fires after about 10 s of silence
- Keepalive monitoring arms only once the PC sends its first keepalive command, and every `Setup()` run disarms it
  again, so the controller never times out before the PC starts pinging it
- Module constructor arguments: `(module_type, module_id, communication)`
- The `modules[]` array must contain at least one element (enforced by `static_assert`)
- `Serial.begin()` baudrate must match both the target board environment's `monitor_speed` and the PC-side `baudrate`
  parameter, so keep it in a named constant rather than a literal (see below)
- Modules that perform analog reads require 12-bit resolution via `analogReadResolution(12)`. AVR boards have a fixed
  10-bit ADC and no `analogReadResolution()`, so the call must be guarded with `#if !defined(__AVR__)`

### Serial speed per board environment

`platformio.ini` sets `monitor_speed` separately for every board environment, and the firmware's `Serial.begin()` rate
has to match the `monitor_speed` of the environment being built. The library's own main.cpp holds the rate in a named
constant and states that the value it carries matches the `teensy41` environment only.

| Environment | Board        | `monitor_speed` |
|-------------|--------------|-----------------|
| `teensy41`  | Teensy 4.1   | 115200          |
| `due`       | Arduino Due  | 5250000         |
| `mega`      | Arduino Mega | 1000000         |

**Note:** these rates are a link contract, not a tunable. The PC side has to open the port at the same speed, and if it
does not, nothing reports a baud-rate fault. The controller runs `RuntimeCycle()` normally, and the PC-side
`MicroControllerInterface` raises a "did not respond to the identification request" error once its retry budget
expires. Read that identification failure as a baud mismatch first, before suspecting the module code. A project that
targets more than one board should resolve the rate at compile time, so the wrong constant cannot reach the wrong
board.

### Keepalive interval

`kKeepaliveInterval` is the firmware-side knob. It is passed to the Kernel constructor as milliseconds, and a value of
0 disables keepalive monitoring outright. The library README recommends enabling it for most use cases and gives these
starting bands, chosen by link speed and CPU frequency rather than by board name:

| Board profile                            | README starting band |
|------------------------------------------|----------------------|
| Fast controller on USB (Teensy 4.1)      | 100-500 ms           |
| Slower controller on UART (Arduino Mega) | 2-5 s                |

**Note:** the bands are the README's own authored guidance, and the README pairs the slow-board band with a 115200
UART link while the `mega` environment in `platformio.ini` runs at 1000000. Treat a band as a starting point to
confirm against the link the project actually uses. A band names the PC's ping period, not the silence the controller
tolerates, because the Kernel doubles the interval to derive the timeout.

---

## Build and upload

Build, upload, and monitor the firmware with PlatformIO, whose board environments are defined in `platformio.ini` and
covered by `/platformio-config`. For the full `pio` command set, see [api-reference.md](references/api-reference.md).
Re-upload whenever the firmware module, its command/event codes, or its parameter struct change.

---

## Implementation hints

See [references/api-reference.md](references/api-reference.md) for template parameter type guidelines, static-assertion
placement, and optional efficiency patterns (constexpr pin logic for polarity-configurable modules and sensor hysteresis
for polling commands).

---

## Related skills

| Skill                                      | Relationship                                                       |
|--------------------------------------------|--------------------------------------------------------------------|
| `/communication:microcontroller-interface` | PC-side counterpart: ModuleInterface, lifecycle, and MQTT setup    |
| `/communication:microcontroller-setup`     | Hardware discovery: MCP tools to verify connected microcontrollers |
| `/communication:extraction-configuration`  | Downstream: configure extraction for this module's event codes     |
| `/cpp-style`                               | C++ coding conventions for firmware code                           |
| `/project-layout`                          | Project directory structure for PlatformIO firmware projects       |
| `/platformio-config`                       | PlatformIO config conventions for platformio.ini and library.json  |

---

## Verification checklist

```text
Firmware Module, tool-settled (run `pio run` and `clang-format --dry-run --Werror src/*.h`):
- [ ] Firmware compiles without warnings
- [ ] Doxygen and inline comments fill to 120 characters before wrapping, under the wrap-width rule /cpp-style defines

Firmware Module, reader-judged:
- [ ] Verified ataraxis-micro-controller >=4.0.2 and ataraxis-transport-layer-mc >=4.0.1
- [ ] Read module.h source to confirm API has not changed
- [ ] Module header file created with an include guard, and library headers included with angle brackets
- [ ] Class inherits from Module (public inheritance)
- [ ] Template parameters use const keyword
- [ ] Static assertions at top of class body (after opening brace, before public:)
- [ ] Constructor calls Module(module_type, module_id, communication)
- [ ] Declares an overriding virtual destructor (~CustomModule() override = default;)
- [ ] kCommands enum defines commands with values >= 1
- [ ] Custom event codes enum defines event codes with values 51-250
- [ ] CustomRuntimeParameters struct uses PACKED_STRUCT macro
- [ ] SetupModule() configures pins and resets parameters to defaults
- [ ] SetupModule() deactivates every managed output as its first statement, leaving the module idle
- [ ] SetupModule() contains no logic that can return false, and no blocking, handshake, calibration, or polling step
- [ ] Any unavoidable SetupModule() failure mode is caught by a static_assert instead of a runtime false return
- [ ] The idle pin level is a named constexpr verified against the device wiring, not a hardcoded LOW
- [ ] SetCustomParameters() calls ExtractParameters() (not _communication.ExtractModuleParameters())
- [ ] RunActiveCommand() switches on get_active_command() and returns true/false
- [ ] All command handlers call CompleteCommand() when done
- [ ] Multi-stage commands use get_command_stage() starting at stage 1
- [ ] Multi-stage commands call AdvanceCommandStage() between stages
- [ ] Default case in stage switch calls AbortCommand()
- [ ] Pin reads use the templated form AnalogRead<kPin>(pool_size) and DigitalRead<kPin>(pool_size)
- [ ] Module registered in main.cpp modules[] array
- [ ] Instantiation order: Communication -> Module(s) -> Kernel
- [ ] Serial.begin() receives the monitor_speed of the board environment being built, held in a named constant
- [ ] kKeepaliveInterval is either 0 by deliberate choice or a value inside the README band for the board's link
- [ ] module_type and module_id match PC-side ModuleInterface values (see /communication:microcontroller-interface)
- [ ] Command codes, event codes, and parameter struct layout match PC-side counterpart
```
