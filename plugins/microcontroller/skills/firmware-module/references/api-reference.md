# Module base class API reference

All signatures are sourced from the library's header files at version 4.0.1.

---

## Module constructor

```cpp
Module(const uint8_t module_type, const uint8_t module_id, Communication& communication)
```

| Parameter       | Type             | Description                                                    |
|-----------------|------------------|----------------------------------------------------------------|
| `module_type`   | `uint8_t`        | Family code (1-255). All instances of the same class share it. |
| `module_id`     | `uint8_t`        | Instance code (1-255). Unique within the module type.          |
| `communication` | `Communication&` | Shared Communication instance for PC messaging.                |

---

## ExecutionControlParameters

Tracks command execution state. Managed by Kernel and utility methods. End users should not modify fields directly.

```cpp
struct ExecutionControlParameters
{
    uint8_t command          = 0;      // Currently executed command (0 = none).
    uint8_t stage            = 0;      // Stage of current command (0 = inactive, starts at 1).
    bool noblock             = false;  // True = non-blocking, false = blocking execution.
    uint8_t next_command     = 0;      // Queued command waiting to execute.
    bool next_noblock        = false;  // Noblock flag for the queued command.
    bool new_command         = false;  // True if next_command is newly queued (not recurrent repeat).
    bool run_recurrently     = false;  // True if the command repeats after completion.
    uint32_t recurrent_delay = 0;      // Microseconds between command repetitions.
    elapsedMicros recurrent_timer;     // Measures recurrent command activation delays.
    elapsedMicros delay_timer;         // Measures delays between command stages.
};
```

---

## Pure virtual methods

These three methods MUST be overridden by every Module subclass.

### SetupModule

```cpp
virtual bool SetupModule() = 0;
```

Called by Kernel during `Setup()` and on PC-requested resets. Initializes hardware and sets parameter defaults. Should
not contain blocking logic. Returns `true` on success, `false` on failure (failure bricks the controller until firmware
reset).

Design this method so it cannot fail, push any unavoidable failure to compile time with `static_assert`, and deactivate
all managed hardware as the first statement of the body, before any logic that can return `false`. A `false` return
aborts `Kernel::Setup()` and reduces `RuntimeCycle()` to blinking the LED, so no module command ever runs again and
every managed module stays frozen in its current physical state until the firmware is reset. See the SetupModule design
requirements in SKILL.md for the full rationale and the per-module consequences.

### SetCustomParameters

```cpp
virtual bool SetCustomParameters() = 0;
```

Called when the PC sends a ModuleParameters message (protocol 5) addressed to this module. Should call
`ExtractParameters()` to unpack the received data into the parameter struct. Returns `true` on success.

### RunActiveCommand

```cpp
virtual bool RunActiveCommand() = 0;
```

Called during each Kernel runtime cycle when the module has an active command. Should translate the command code from
`get_active_command()` into a call to the command-specific handler method. Returns `true` if the command was recognized,
`false` otherwise. Does NOT indicate command success.

Returning `false` makes the Kernel call `SendCommandActivationError()`, which reports event code 3, and then
`DiscardActiveCommand()`, which clears the active command and its stage without a `kCommandCompleted` message. That
method also resets the command queue when the queue still holds the rejected command, so one-off and recurrent commands
alike are dropped after a single error report. A command the PC queued while the rejected one was running is left
untouched and activates on the next cycle.

### Virtual destructor

```cpp
virtual ~Module() = default;
```

Module subclasses should declare an overriding destructor (`~MyModule() override = default;`) to ensure proper cleanup
through base class pointers. The Kernel manages modules via `Module*` arrays, so the virtual destructor is load-bearing.

---

## Kernel-facing public methods

The Kernel drives each module through these public methods. Command handlers must not call them, because a handler
manages its own lifecycle through the protected utilities below, and subclasses must not override or shadow them.

```cpp
void QueueCommand(const uint8_t command, const bool noblock, const uint32_t cycle_delay);
void QueueCommand(const uint8_t command, const bool noblock);
void ResetCommandQueue();
bool ResolveActiveCommand();
void ResetExecutionParameters();
[[nodiscard]] uint8_t get_module_id() const;
[[nodiscard]] uint8_t get_module_type() const;
[[nodiscard]] uint16_t get_module_type_id() const;
void SendCommandActivationError() const;
void SendCommandRejection(const uint8_t command) const;
void DiscardActiveCommand();
```

| Method                       | The Kernel calls it when                                                                      |
|------------------------------|-----------------------------------------------------------------------------------------------|
| `QueueCommand` (recurrent)   | A RepeatedModuleCommand (protocol 1) arrives. `cycle_delay` is the delay between repetitions. |
| `QueueCommand` (one-off)     | A OneOffModuleCommand (protocol 2) arrives. Queues the command as non-recurrent.              |
| `ResetCommandQueue`          | A DequeueModuleCommand (protocol 3) arrives. A running command still finishes gracefully.     |
| `ResolveActiveCommand`       | Once per runtime cycle, before `RunActiveCommand()`. `false` skips the module for that cycle. |
| `ResetExecutionParameters`   | Inside `Kernel::Setup()`, on every module, before the `SetupModule()` pass.                   |
| `get_module_id`              | Resolving the target of a module-addressed message and building error payloads.               |
| `get_module_type`            | Resolving the target of a module-addressed message and building error payloads.               |
| `get_module_type_id`         | The `kIdentifyModules` kernel command (code 4). Returns `module_type << 8 \| module_id`.      |
| `SendCommandActivationError` | `RunActiveCommand()` returned `false`. Reports event code 3 against the active command.       |
| `SendCommandRejection`       | A module command message carries command code 0. Reports event code 3 against that code.      |
| `DiscardActiveCommand`       | Immediately after `SendCommandActivationError()`. Drops the queue entry, sends no message.    |

`ResolveActiveCommand()` applies the command priority chain: finish the active command, then activate a newly queued
command, then repeat a recurrent command whose delay has elapsed. The recurrent comparison is inclusive
(`recurrent_timer >= recurrent_delay`), so the largest representable delay still repeats.

Both `QueueCommand()` overloads and `ResetCommandQueue()` first report the retirement of a recurrent command that is
idle between repetitions, sending `kCommandCompleted` attributed to that command's own code. `CompleteCommand()` cannot
cover that case, because none of the retired command's stages are running at the moment it is replaced or dequeued.

---

## Protected utility methods

### Command state accessors

```cpp
[[nodiscard]] uint8_t get_active_command() const;
```

Returns the active command code, or 0 if no command is active.

```cpp
[[nodiscard]] uint8_t get_command_stage() const;
```

Returns the execution stage of the active command (starts at 1), or 0 if no command is active.

### Command lifecycle

```cpp
void CompleteCommand();
```

Ends the active command. Sends a kCommandCompleted (event code 2) message to the PC under three conditions. A new
command is waiting to replace the current one, no next command is queued (including after an explicit dequeue), or the
command is not recurrent. Resets stage to 0, allowing the next command to activate. **You MUST call this at the end of
every command handler.** Failure to call it deadlocks the module.

```cpp
void AbortCommand();
```

Cancels the active command. If no new command is pending, resets the command queue to prevent reactivation. Then calls
`CompleteCommand()` internally.

```cpp
void AdvanceCommandStage();
```

Increments the stage counter and resets the delay timer. Use to transition between stages of a multi-stage command.

### Non-blocking delay

```cpp
[[nodiscard]] bool WaitForMicros(const uint32_t delay_duration) const;
```

Delays execution for the specified microseconds, measured relative to the last stage advancement.
- **Blocking mode** (`noblock=false`): blocks in-place until the duration has passed, always returns `true`.
- **Non-blocking mode** (`noblock=true`): returns `true` if the duration has elapsed, `false` otherwise. The module
  returns control to the Kernel, allowing other modules to execute.
- **Duration clamp**: `delay_duration` is clamped to `UINT32_MAX - 1` microseconds, roughly 71.6 minutes. The 32-bit
  stage timer has to exceed the duration for the blocking wait to end, so an unclamped `UINT32_MAX` would hang the
  controller until it is power-cycled. Build a stage delay longer than that bound out of several stages.

### Pin reading

```cpp
template <const uint8_t kPin>
[[nodiscard]] static uint16_t AnalogRead(const uint16_t pool_size = 0);
```

Reads the analog pin named by the `kPin` template argument, so the call form is `AnalogRead<kSensorPin>(pool_size)`. If
`pool_size >= 2`, reads and averages that many samples using half-up rounding. Returns the raw or averaged readout.

```cpp
template <const uint8_t kPin>
[[nodiscard]] static bool DigitalRead(const uint16_t pool_size = 0);
```

Reads the digital pin named by the `kPin` template argument through `digitalReadFast`, so the call form is
`DigitalRead<kSensorPin>(pool_size)`. If `pool_size >= 2`, reads and averages that many samples. Returns `true` (HIGH)
or `false` (LOW).

`kPin` is a template parameter on both helpers, so every pin passed to them has to be a compile-time constant, either a
class template parameter or a `static constexpr uint8_t` member. A pin held in a runtime parameter struct cannot be read
through them. For `DigitalRead`, the compile-time constant also resolves the register-level read path that AVR and
Teensy boards expose, and Arduino Due exposes no such path and reads through `digitalRead`. `AnalogRead` delegates to
`analogRead()` on every board.

### Data transmission

```cpp
template <typename ObjectType>
void SendData(const uint8_t event_code, const ObjectType& object);
```

Sends a ModuleData message (protocol 6) with event code and typed data object. The prototype code for the wire protocol
is resolved automatically at compile time from ObjectType. Supports all 11 scalar types and C-style arrays at
type-specific element counts, up to a platform-dependent payload cap. `uint8_t` arrays have the densest count support
and can be used as a generic bytes buffer. On failure, automatically attempts to send an error message and turns on the
built-in LED.

The following table lists all supported data types and element counts. An element count of 1 is a scalar, and counts
greater than 1 require a C-style array declaration (e.g., `uint16_t[24]`). Unsupported (type, count) combinations
trigger a compile-time `static_assert` error.

| C++ type   | Size    | Numpy equivalent | Supported element counts                                                         |
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

The counts above are the prototype codes the wire protocol defines. The number of bytes a board can actually transmit is
capped separately by its serial buffer, because the data object has to satisfy
`sizeof(ObjectType) <= kMaximumPayloadSize - sizeof(ModuleData)`. That yields 248 bytes on Teensy, 244 bytes on Arduino
Due, and 52 bytes on Arduino Mega, so counts such as `uint8_t[248]` or `uint16_t[124]` compile only on Teensy. The
`SendDataMessage` static assertion rejects an oversized object at compile time for the board being built.

```cpp
void SendData(const uint8_t event_code) const;
```

Sends a ModuleState message (protocol 8) with event code only. More efficient than the data-carrying overload when no
payload is needed. Same error handling behavior.

### Parameter extraction

```cpp
template <typename ObjectType>
bool ExtractParameters(ObjectType& storage_object);
```

Unpacks the received ModuleParameters message payload into the specified storage object. Internally delegates to
`Communication::ExtractModuleParameters()`, which `static_assert`s that the struct is at least one byte and still fits
into the payload alongside the 3-byte ModuleParameters header and the protocol code. That upper bound is 250 bytes on
Teensy, 246 bytes on Arduino Due, and 54 bytes on Arduino Mega, so a struct that builds for one board can fail to build
for another. Returns `true` on success, `false` on one of three conditions: `kExtractionForbidden` (message is not a
ModuleParameters message), `kParameterMismatch` (received byte count does not equal `sizeof(struct)`), or
`kParsingError` (payload parsing failed).

---

## kCoreStatusCodes

System-reserved event codes used by Module base class methods:

| Code | Constant                | Description                        |
|------|-------------------------|------------------------------------|
| 0    | `kStandby`              | Default initialization value.      |
| 1    | `kTransmissionError`    | SendData failed to transmit to PC. |
| 2    | `kCommandCompleted`     | A command finished.                |
| 3    | `kCommandNotRecognized` | A command could not be executed.   |

Code 2 has two emitters. `CompleteCommand()` sends it for the command that just ran, under the conditions listed for
that method, and `QueueCommand()` or `ResetCommandQueue()` sends it for a recurrent command retired while idle between
repetitions, attributed to that command's own code. `DiscardActiveCommand()` sends nothing.

Code 3 also has two emitters, both invoked by the Kernel. `SendCommandActivationError()` sends it against the active
command when `RunActiveCommand()` returns `false`, and `SendCommandRejection()` sends it against the rejected code when
the PC addresses the module with command code 0, leaving the active command undisturbed.

---

## Kernel constructor

```cpp
template <const size_t kModuleNumber>
Kernel(
    const uint8_t controller_id,
    Communication& communication,
    Module* (&module_array)[kModuleNumber],
    const uint32_t keepalive_interval = 0
);
```

| Parameter            | Type             | Description                                                   |
|----------------------|------------------|---------------------------------------------------------------|
| `controller_id`      | `uint8_t`        | Unique controller identifier (1-255). Must match PC side.     |
| `communication`      | `Communication&` | Shared Communication instance.                                |
| `module_array`       | `Module*[]`      | Array of Module pointers. Must contain >= 1 element.          |
| `keepalive_interval` | `uint32_t`       | Milliseconds between expected keepalive messages. 0=disabled. |

The Kernel internally doubles the keepalive interval to tolerate brief communication lapses, saturating at the largest
representable millisecond value rather than wrapping, so an interval above `UINT32_MAX / 2` yields the maximum timeout.
Keepalive tracking stays inert until the PC sends its first keepalive kernel command (code 5), which arms tracking when
the configured interval is non-zero and resets the timer. A controller the PC has not yet contacted therefore never
times out. Every `Setup()` run disarms tracking again, so the PC has to re-arm the watchdog after a requested reset or
after a keepalive-triggered emergency reset.

### kKernelCommands

The kernel command codes. The PC addresses codes 2 through 5 in a KernelCommand message (protocol 4), while codes 0 and
1 mark internal Kernel states, which the Kernel reports as kernel status 8 when a PC message carries either of them.
Firmware authors implement none of the six, as the Kernel handles them all internally.

| Code | Constant              | Description                                                      |
|------|-----------------------|------------------------------------------------------------------|
| 0    | `kStandby`            | Class initialization value.                                      |
| 1    | `kReceiveData`        | Receives PC-sent data. Not externally addressable.               |
| 2    | `kResetController`    | Reruns `Setup()`, resetting all managed modules and hardware.    |
| 3    | `kIdentifyController` | Sends the controller ID to the PC.                               |
| 4    | `kIdentifyModules`    | Sends each managed module's combined type and ID code to the PC. |
| 5    | `kKeepAlive`          | Arms the keepalive watchdog and restarts its timer.              |

### Kernel lifecycle

| Method           | Called from | Description                                               |
|------------------|-------------|-----------------------------------------------------------|
| `Setup()`        | `setup()`   | Calls `SetupModule()` on all modules. Sends setup status. |
| `RuntimeCycle()` | `loop()`    | Receives messages, executes commands, checks keepalive.   |

### kKernelStatusCodes

| Code | Constant                  | Description                             |
|------|---------------------------|-----------------------------------------|
| 0    | `kStandby`                | Not used, reserves 0.                   |
| 1    | `kSetupComplete`          | Setup succeeded.                        |
| 2    | `kModuleSetupError`       | A module's SetupModule() failed.        |
| 3    | `kReceptionError`         | Error receiving a PC message.           |
| 4    | `kTransmissionError`      | Error sending data to PC.               |
| 5    | `kInvalidMessageProtocol` | Unknown protocol code received.         |
| 6    | `kModuleParametersSet`    | Parameters applied to module.           |
| 7    | `kModuleParametersError`  | Failed to apply parameters.             |
| 8    | `kCommandNotRecognized`   | Unknown kernel command code.            |
| 9    | `kTargetModuleNotFound`   | No module with requested type+id.       |
| 10   | `kKeepAliveTimeout`       | Keepalive message not received in time. |

---

## Communication constructor

```cpp
explicit Communication(Stream& communication_port);
```

| Parameter            | Type      | Description                                |
|----------------------|-----------|--------------------------------------------|
| `communication_port` | `Stream&` | Arduino Stream (Serial, USB Serial, etc.). |

Creates a TransportLayer instance with a non-reflected CRC16 (polynomial 0x1021, init 0xFFFF, final XOR 0x0000). The PC
interface has to use the same CRC parameters. Reserves up to ~1 kB of RAM (~700 bytes on lower-end boards).

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
- In non-blocking mode, `WaitForMicros` returns immediately with `false` if the time has not elapsed, allowing other
  modules to execute. In blocking mode, it blocks in-place until the time has passed.
- Call `CompleteCommand()` on the final stage
- The `default` case should call `AbortCommand()` to handle unexpected stages

### Sensor polling command

For repeated sensor readings:

```cpp
void ReadSensor()
{
    const uint16_t value = AnalogRead<kSensorPin>(parameters.pool_size);
    SendData(static_cast<uint8_t>(kStates::kValueRead), value);
    CompleteCommand();
}
```

---

## Implementation hints

**Constexpr pin logic for polarity-configurable modules:** When a template parameter controls whether hardware is
normally-open vs. normally-closed (or similar polarity inversion), compute the active/inactive logic levels as constexpr
booleans rather than branching at runtime:

```cpp
template <const uint8_t kPin, const bool kNormallyClosed>
class ValveModule final : public Module
{
        static constexpr bool kOpen  = kNormallyClosed ? HIGH : LOW;
        static constexpr bool kClose = kNormallyClosed ? LOW  : HIGH;
    // ...
    // Then use kOpen/kClose directly: digitalWriteFast(kPin, kOpen);
};
```

**Sensor hysteresis for polling commands:** When a sensor-polling command runs recurrently, tracking the previous
reading avoids flooding the PC with redundant zero or steady-state messages. Report only when the value crosses a
meaningful threshold or when the state changes:

```cpp
void CheckSensor()
{
    const uint16_t value = AnalogRead<kPin>(parameters.pool_size);
    const bool above_threshold = value >= parameters.signal_threshold;

    if (above_threshold)
    {
        SendData(static_cast<uint8_t>(kStates::kDetected), value);
        _previous_zero = false;
    }
    else if (!_previous_zero)
    {
        SendData(static_cast<uint8_t>(kStates::kDetected), static_cast<uint16_t>(0));
        _previous_zero = true;
    }
    CompleteCommand();
}
```

This reduces serial bandwidth and log archive size without losing transition information.

---

## Template parameter guidelines

| Type       | Use case                          | Example                        |
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

## Build and upload

Build, upload, and monitor the firmware with PlatformIO. The board environment(s) are defined in `platformio.ini` (see
`/platformio-config`).

```bash
# Build only (compile, no upload)
pio run

# Build and upload to a specific board environment
pio run --environment teensy41 --target upload

# Build and upload every environment platformio.ini defines
pio run --target upload

# List the environments defined in platformio.ini
pio project config

# Open the serial monitor after upload (match the Serial.begin() baudrate)
pio device monitor --baud 115200
```

A command that names no environment covers every environment `platformio.ini` defines, unless the file sets
`default_envs`, which narrows the set to the environments it names. Prefer the `--environment` form when the project
targets several boards, because an upload to a board that is not connected fails that environment.

After uploading, the controller runs the deterministic `RuntimeCycle()` loop and is ready for the PC-side
`MicroControllerInterface` to connect (see `/communication:microcontroller-interface`). Re-upload whenever the firmware
module, its command/event codes, or its parameter struct change.
