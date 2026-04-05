---
name: firmware-module
description: >-
  Guides creation of custom hardware Module subclasses in the ataraxis-micro-controller C++ firmware
  library. Covers SetupModule, SetCustomParameters, and RunActiveCommand implementation, stage-based
  command execution, parameter structures with PACKED_STRUCT, event codes, SendData patterns, and
  main.cpp integration with Kernel and Communication. Use when writing or modifying firmware module
  classes for microcontrollers, when the user asks about Module subclassing, or when implementing
  new hardware interfaces on the microcontroller side of ataraxis-communication-interface.
user-invocable: true
---

# Firmware module

Guides implementation of custom hardware Module subclasses in the ataraxis-micro-controller C++ firmware
library. This skill covers the firmware side of the microcontroller communication stack; for the PC-side
Python ModuleInterface counterpart, use `/communication:microcontroller-interface` instead.

---

## Scope

**Covers:**
- Module base class inheritance and constructor pattern
- Three required virtual methods: SetupModule, SetCustomParameters, RunActiveCommand
- Command handler patterns: immediate, multi-stage with non-blocking delay, sensor polling
- Runtime parameter structures with `PACKED_STRUCT` macro
- Event and status code conventions (system 0-50, user 51-250)
- Sending data to PC via `SendData()` overloads
- Template-based module design with compile-time pin configuration
- Static assertions for compile-time validation
- main.cpp integration: Communication, Module, and Kernel wiring

**Does not cover:**
- PC-side ModuleInterface subclassing (see `/communication:microcontroller-interface`)
- Microcontroller discovery or MQTT testing via MCP tools (see `/communication:microcontroller-setup`)
- C++ coding conventions (see `/cpp-style`)
- PlatformIO project directory layout (see `/project-layout`)

---

## Cross-plugin boundary: firmware vs. interface

This skill and `/communication:microcontroller-interface` are counterparts that share a communication
protocol but live in different plugins with distinct responsibilities:

| Concern                                 | Authority                                  |
|-----------------------------------------|--------------------------------------------|
| C++ Module subclass, command handlers   | This skill                                 |
| Parameter structs (`PACKED_STRUCT`)     | This skill                                 |
| main.cpp wiring (Communication, Kernel) | This skill                                 |
| Python ModuleInterface subclass         | `/communication:microcontroller-interface` |
| MicroControllerInterface lifecycle      | `/communication:microcontroller-interface` |
| MQTTCommunication setup                 | `/communication:microcontroller-interface` |

The two sides must agree on **module_type**, **module_id**, **command codes**, **event codes**,
and **parameter struct layout** (field order, types, and sizes). When implementing a new hardware
module, always work both skills together: this skill for the C++ firmware and
`/communication:microcontroller-interface` for the Python interface. If either side's codes or
parameter layout change, the other must be updated to match.

---

## Verification requirements

**Before writing any firmware module code, verify the current state of the library.**

### Step 1: Version verification

Check the locally available ataraxis-micro-controller version:

```bash
cat ../ataraxis-micro-controller/library.json | grep version
```

The current version is **3.0.0**. If a version mismatch exists, ask the user how to proceed.

### Step 2: API verification

Read the source files to confirm the API has not changed since this skill was written:

| File                                                    | What to Check                                     |
|---------------------------------------------------------|---------------------------------------------------|
| `../ataraxis-micro-controller/src/module.h`             | Module base class, virtual methods, utilities     |
| `../ataraxis-micro-controller/src/kernel.h`             | Kernel constructor and lifecycle                  |
| `../ataraxis-micro-controller/src/communication.h`      | Communication constructor                         |
| `../ataraxis-micro-controller/src/axmc_shared_assets.h` | Protocol codes, ResolvePrototype, message structs |

---

## API reference

See [references/api-reference.md](references/api-reference.md) for the complete Module base class API
including constructor parameters, ExecutionControlParameters fields, all protected utility method
signatures, kCoreStatusCodes, Kernel constructor, and Communication constructor.

The prototype code for each `SendData()` call is resolved automatically at compile time from the C++
type of the data object via the `ResolvePrototype` function in `axmc_shared_assets.h`. Users do not
need to specify prototype codes manually.

---

## Module class structure

Modules are header-only classes that inherit from `Module` and override three pure virtual methods.
The template pattern is recommended for compile-time hardware configuration:

```cpp
#pragma once

#include "module.h"

template <const uint8_t kPin>
class CustomModule final : public Module
{
        static_assert(
            kPin != LED_BUILTIN,
            "The LED-connected pin is reserved for error indication. Select a different pin."
        );

    public:
        // Constructor, enums, parameters, virtual method overrides

    private:
        // Command handler methods
};
```

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

### Custom status codes enum

Custom event codes MUST use values 51-250. Values 0-50 are reserved for system use. Each event code
MUST be unique within the module class and MUST carry the same semantic meaning regardless of which
command was executing when the message was sent. The extraction pipeline and PC-side
`process_received_data()` both rely on this invariant:

```cpp
private:
    enum class kStates : uint8_t
    {
        kHigh = 52,
        kLow  = 53,
        kEcho = 54,
    };
```

| Range  | Owner    | Description                                         |
|--------|----------|-----------------------------------------------------|
| 0      | System   | Standby (module idle)                               |
| 1      | System   | Transmission error                                  |
| 2      | System   | Command completed                                   |
| 3      | System   | Command not recognized                              |
| 4-50   | System   | Reserved for future system use                      |
| 51-250 | User     | Custom event codes, unique within each module class |
| 251+   | -------- | Reserved, do not use                                |

### Runtime parameters structure

Parameter structs MUST use the `PACKED_STRUCT` macro to ensure correct binary serialization with the PC.
Field order and types must exactly match the PC-side `send_parameters()` tuple:

```cpp
public:
    struct CustomRuntimeParameters
    {
            uint32_t on_duration  = 2000000;
            uint32_t off_duration = 2000000;
            uint16_t echo_value   = 666;
    } PACKED_STRUCT parameters;
```

| C++ Type   | Size    | Python Equivalent | Typical Use               |
|------------|---------|-------------------|---------------------------|
| `bool`     | 1 byte  | `np.bool_`        | Enable flags              |
| `uint8_t`  | 1 byte  | `np.uint8`        | Small counts, codes       |
| `uint16_t` | 2 bytes | `np.uint16`       | ADC values, medium counts |
| `uint32_t` | 4 bytes | `np.uint32`       | Microsecond durations     |
| `int32_t`  | 4 bytes | `np.int32`        | Signed large values       |
| `float`    | 4 bytes | `np.float32`      | Calibrated sensor values  |

---

## SetupModule()

Initialize hardware pins and reset parameters to defaults. This method is called by Kernel during
`Setup()` and on PC-requested resets. You MUST avoid blocking logic:

```cpp
bool SetupModule() override
{
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
- Set all parameter fields to their default values
- Return `true` on success, `false` on failure (failure bricks the controller until firmware reset)

---

## SetCustomParameters()

Extract the parameter struct from the PC message using the inherited `ExtractParameters()` wrapper:

```cpp
bool SetCustomParameters() override
{
    return ExtractParameters(parameters);
}
```

You MUST use `ExtractParameters()` (the Module base class wrapper), not
`_communication.ExtractModuleParameters()` directly. Post-processing of extracted values is permitted:

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

---

## Command handler patterns

### Immediate command

For commands that complete in a single step:

```cpp
void Echo()
{
    SendData(static_cast<uint8_t>(kStates::kEcho), parameters.echo_value);
    CompleteCommand();
}
```

You MUST call `CompleteCommand()` at the end of every command handler. Failure to do so deadlocks
the module.

### Multi-stage command with non-blocking delay

For commands requiring timed steps. Stages start at 1, not 0:

```cpp
void Pulse()
{
    switch (get_command_stage())
    {
        case 1:
            digitalWrite(kPin, HIGH);
            SendData(static_cast<uint8_t>(kStates::kHigh));
            AdvanceCommandStage();
            break;

        case 2:
            if (WaitForMicros(parameters.on_duration)) AdvanceCommandStage();
            break;

        case 3:
            digitalWrite(kPin, LOW);
            SendData(static_cast<uint8_t>(kStates::kLow));
            AdvanceCommandStage();
            break;

        case 4:
            if (WaitForMicros(parameters.off_duration)) CompleteCommand();
            break;

        default: AbortCommand(); break;
    }
}
```

**Stage-based execution rules:**
- Use `get_command_stage()` to read the current stage (stages start at 1)
- Call `AdvanceCommandStage()` to move to the next stage (also resets the delay timer)
- `WaitForMicros(duration)` returns `true` when the duration has elapsed, `false` while waiting
- In non-blocking mode, `WaitForMicros` returns immediately with `false` if the time has not elapsed,
  allowing other modules to execute. In blocking mode, it blocks in-place until the time has passed.
- Call `CompleteCommand()` on the final stage
- The `default` case should call `AbortCommand()` to handle unexpected stages

### Sensor polling command

For repeated sensor readings:

```cpp
void ReadSensor()
{
    const uint16_t value = AnalogRead(kSensorPin, parameters.pool_size);
    SendData(static_cast<uint8_t>(kStates::kValueRead), value);
    CompleteCommand();
}
```

`AnalogRead(pin, pool_size)` reads and averages `pool_size` samples. Set `pool_size` to 0 or 1
to disable averaging. `DigitalRead(pin, pool_size)` works the same way for digital pins.

---

## Sending data to PC

### State-only (no data payload)

```cpp
SendData(static_cast<uint8_t>(kStates::kHigh));
```

Sends a ModuleState message (protocol 8) containing only the event code. Use when the event itself
carries all needed information.

### With typed data payload

```cpp
SendData(static_cast<uint8_t>(kStates::kEcho), parameters.echo_value);
```

Sends a ModuleData message (protocol 6) containing the event code and a typed data object. The
prototype code for the wire protocol is resolved automatically at compile time from the C++ type
of the data argument. Supported types: all 11 scalars (`bool` through `double`) and C-style
arrays of those types at supported element counts up to the 248-byte payload cap. `uint8_t`
arrays offer the densest count coverage and can serve as a generic bytes buffer for sending
arbitrary packed structures via `uint8_t[sizeof(MyStruct)]`.

The following table lists all supported data types and element counts. An element count of 1 represents
a scalar value; counts greater than 1 require a C-style array declaration (e.g., `uint16_t[24]`).
Unsupported (type, count) combinations trigger a compile-time error via `static_assert`.

| C++ Type   | Size    | Numpy Equivalent | Supported Element Counts                                                         |
|------------|---------|------------------|----------------------------------------------------------------------------------|
| `bool`     | 1 byte  | `np.bool_`       | 1-15, 16, 24, 32, 40, 48, 52, 248                                                |
| `uint8_t`  | 1 byte  | `np.uint8`       | 1-15, 16, 18, 20, 22, 24, 28, 32, 36, 40, 44, 48, 52, 64, 96, 128, 192, 244, 248 |
| `int8_t`   | 1 byte  | `np.int8`        | 1-15, 16, 24, 32, 40, 48, 52, 92, 132, 172, 212, 244, 248                        |
| `uint16_t` | 2 bytes | `np.uint16`      | 1-15, 16, 20, 24, 26, 32, 48, 64, 96, 122, 124                                   |
| `int16_t`  | 2 bytes | `np.int16`       | 1-15, 16, 20, 24, 26, 32, 48, 64, 96, 122, 124                                   |
| `uint32_t` | 4 bytes | `np.uint32`      | 1-15, 16, 20, 24, 32, 48, 62                                                     |
| `int32_t`  | 4 bytes | `np.int32`       | 1-15, 16, 20, 24, 32, 48, 62                                                     |
| `float`    | 4 bytes | `np.float32`     | 1-15, 16, 20, 24, 32, 48, 62                                                     |
| `uint64_t` | 8 bytes | `np.uint64`      | 1-15, 16, 20, 24, 31                                                             |
| `int64_t`  | 8 bytes | `np.int64`       | 1-15, 16, 20, 24, 31                                                             |
| `double`   | 8 bytes | `np.float64`     | 1-15, 16, 20, 24, 31                                                             |

**Error handling:** If transmission fails, `SendData()` automatically attempts to send an error message
and turns on the built-in LED. Do not use the LED-connected pin in your module to avoid interference.

---

## main.cpp integration

Follow this exact instantiation order: Communication, Module(s), Kernel.

```cpp
#include <Arduino.h>
#include "custom_module.h"
#include "communication.h"
#include "kernel.h"
#include "module.h"

static constexpr uint8_t kControllerID     = 222;
static constexpr uint32_t kKeepaliveInterval = 5000;

Communication axmc_communication(Serial);

CustomModule<5> module_1(1, 1, axmc_communication);
CustomModule<6> module_2(1, 2, axmc_communication);

Module* modules[] = {&module_1, &module_2};

Kernel axmc_kernel(kControllerID, axmc_communication, modules, kKeepaliveInterval);

void setup()
{
    Serial.begin(115200);
    axmc_kernel.Setup();
}

void loop()
{
    axmc_kernel.RuntimeCycle();
}
```

**Key points:**
- `kControllerID` must match the `controller_id` used on the PC side (1-255, unique per controller)
- `kKeepaliveInterval` is in milliseconds (0 disables, Kernel internally doubles this value)
- Module constructor arguments: `(module_type, module_id, communication)`
- The `modules[]` array must contain at least one element (enforced by `static_assert`)
- `Serial.begin()` baudrate must match the PC-side `baudrate` parameter

---

## Template parameter guidelines

| Type       | Use Case                          | Example                        |
|------------|-----------------------------------|--------------------------------|
| `uint8_t`  | Pin numbers, counts               | `kPin`, `kEncoderPinA`         |
| `bool`     | Hardware polarity, default states | `kNormallyClosed`, `kStartOff` |
| `uint16_t` | Larger constants (calibration)    | `kDefaultThreshold`            |

Use `255` as a sentinel for optional pins:

```cpp
template <const uint8_t kTonePin = 255>
// In implementation:
if constexpr (kTonePin != 255) { pinMode(kTonePin, OUTPUT); }
```

---

## Static assertions

Place static assertions at the **top of the class body**, before `public:`:

```cpp
template <const uint8_t kPinA, const uint8_t kPinB>
class EncoderModule final : public Module
{
        static_assert(kPinA != kPinB, "Channel A and Channel B pins cannot be the same.");
        static_assert(kPinA != LED_BUILTIN, "Select a different Channel A pin.");
        static_assert(kPinB != LED_BUILTIN, "Select a different Channel B pin.");

    public:
        // ...
};
```

---

## Related skills

| Skill                                      | Relationship                                                                                                                                                              |
|--------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `/communication:microcontroller-interface` | PC-side counterpart: use for Python ModuleInterface subclassing, MicroControllerInterface lifecycle, and MQTTCommunication setup. Codes and parameter layouts must match. |
| `/communication:microcontroller-setup`     | Hardware discovery: MCP tools to verify connected microcontrollers                                                                                                        |
| `/communication:extraction-configuration`  | Downstream: configure extraction for this module's event codes                                                                                                            |
| `/cpp-style`                               | C++ coding conventions for firmware code                                                                                                                                  |
| `/project-layout`                          | Project directory structure for PlatformIO firmware projects                                                                                                              |

---

## Verification checklist

```text
Firmware Module:
- [ ] Verified ataraxis-micro-controller version matches requirements (>=3.0.0)
- [ ] Read module.h source to confirm API has not changed
- [ ] Module header file created with #pragma once or include guard
- [ ] Class inherits from Module (public inheritance)
- [ ] Template parameters use const keyword
- [ ] Static assertions at top of class body (after opening brace, before public:)
- [ ] Constructor calls Module(module_type, module_id, communication)
- [ ] kCommands enum defines commands with values >= 1
- [ ] Custom status codes enum defines event codes with values 51-250
- [ ] CustomRuntimeParameters struct uses PACKED_STRUCT macro
- [ ] SetupModule() configures pins and resets parameters to defaults
- [ ] SetCustomParameters() calls ExtractParameters() (not _communication.ExtractModuleParameters())
- [ ] RunActiveCommand() switches on get_active_command() and returns true/false
- [ ] All command handlers call CompleteCommand() when done
- [ ] Multi-stage commands use get_command_stage() starting at stage 1
- [ ] Multi-stage commands call AdvanceCommandStage() between stages
- [ ] Default case in stage switch calls AbortCommand()
- [ ] Module registered in main.cpp modules[] array
- [ ] Instantiation order: Communication -> Module(s) -> Kernel
- [ ] module_type and module_id match PC-side ModuleInterface values (verify via /communication:microcontroller-interface)
- [ ] Command codes, event codes, and parameter struct layout match PC-side counterpart
- [ ] Firmware compiles without warnings
```
