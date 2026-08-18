# Module base class API reference

All signatures are sourced from the library's header files at version 4.0.2, which pins `ataraxis-transport-layer-mc` at
`^4.0.1`.

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

### SetupModule

```cpp
virtual bool SetupModule() = 0;
```

Called by Kernel during `Setup()`, which runs at controller startup, on a PC-requested reset, and when the keepalive
monitor detects a lost PC connection. Initializes hardware and sets parameter defaults. Should not contain blocking
logic. Returns `true` on success, `false` on failure (failure bricks the controller until firmware reset).

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

Returning `false` makes the Kernel call `SendCommandActivationError()` and then `DiscardActiveCommand()`, which clears
the active command and its stage without a `kCommandCompleted` message. That method also resets the command queue when
the queue still holds the rejected command, so one-off and recurrent commands alike are dropped after a single error
report. A command the PC queued while the rejected one was running is left untouched and activates on the next cycle.

### Virtual destructor

```cpp
virtual ~Module() = default;
```

Module subclasses should declare an overriding destructor (`~MyModule() override = default;`) to ensure proper cleanup
through base class pointers. The Kernel manages modules via `Module*` arrays, so the virtual destructor is load-bearing.

---

## Kernel-facing public methods

Command handlers must not call these methods, because a handler manages its own lifecycle through the protected
utilities below, and subclasses must not override or shadow them.

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
| `SendCommandActivationError` | `RunActiveCommand()` returned `false` for the active command.                                 |
| `SendCommandRejection`       | A module command message carries command code 0.                                              |
| `DiscardActiveCommand`       | Immediately after `SendCommandActivationError()`. Drops the queue entry.                      |

`ResolveActiveCommand()` applies the command priority chain: finish the active command, then activate a newly queued
command, then repeat a recurrent command whose delay has elapsed. The recurrent comparison is inclusive
(`recurrent_timer >= recurrent_delay`), so the largest representable delay still repeats.

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

Ends the active command. Sends a kCommandCompleted message, which is core event code 2, to the PC under three
conditions. A new command is waiting to replace the current one, no next command is queued (including after an explicit
dequeue), or the command is not recurrent. Resets stage to 0, allowing the next command to activate. **You MUST call
this at the end of every command handler.** Failure to call it deadlocks the module.

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
type-specific element counts. `uint8_t` arrays have the densest count support and can be used as a generic bytes buffer.

An element count of 1 is a scalar, and counts greater than 1 require a C-style array declaration (e.g., `uint16_t[24]`).
Unsupported (type, count) combinations trigger a compile-time `static_assert` error.

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
capped separately by its serial buffer, because the data object has to satisfy `sizeof(ObjectType) <=
kMaximumPayloadSize - sizeof(ModuleData)`. That yields 248 bytes on Teensy, 244 bytes on Arduino Due, and 52 bytes on
Arduino Mega, so counts such as `uint8_t[248]` or `uint16_t[124]` compile only on Teensy. The `SendDataMessage` static
assertion rejects an oversized object at compile time for the board being built.

**`double` does not build for an AVR board by default.** avr-gcc compiles `double` to 4 bytes unless the build passes
`-mdouble=64`, and `axmc_shared_assets.h` rejects the narrower width at compile time rather than tagging a 4-byte
payload with a prototype code the PC would decode as 8 bytes. An Arduino Mega therefore fails to compile rather than
failing at runtime. Add `-mdouble=64` to that board's `build_flags`, or use `float` and `np.float32` on both sides.
Teensy and Arduino Due are unaffected. `/communication:microcontroller-interface` carries the PC-side statement of the
same constraint.

```cpp
void SendData(const uint8_t event_code) const;
```

Sends a ModuleState message (protocol 8) with event code only. More efficient than the data-carrying overload when no
payload is needed. Both overloads report a failed transfer through the communication error payload below.

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
ModuleParameters message), `kParameterMismatch` (the reception buffer does not hold `sizeof(struct)` plus the 3-byte
ModuleParameters header plus the protocol code), or `kParsingError` (payload parsing failed).

---

## kCoreStatusCodes

System-reserved event codes used by Module base class methods:

| Code | Constant                | Description                        |
|------|-------------------------|------------------------------------|
| 0    | `kStandby`              | Default initialization value.      |
| 1    | `kTransmissionError`    | SendData failed to transmit to PC. |
| 2    | `kCommandCompleted`     | A command finished.                |
| 3    | `kCommandNotRecognized` | A command could not be executed.   |

Core event code 2 has two emitters. `CompleteCommand()` sends it for the command that just ran, under the conditions
listed for that method, and `QueueCommand()` or `ResetCommandQueue()` sends it for a recurrent command retired while
idle between repetitions, attributed to that command's own code. `DiscardActiveCommand()` sends nothing.

Core event code 3 also has two emitters, both invoked by the Kernel. `SendCommandActivationError()` sends it against the
active command when `RunActiveCommand()` returns `false`, and `SendCommandRejection()` sends it against the rejected
code when the PC addresses the module with command code 0, leaving the active command undisturbed.

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

The library README recommends enabling keepalive for most use cases and gives starting bands chosen by link speed and
CPU frequency rather than by board name. The bands are 100-500 ms for a fast controller on USB such as the Teensy 4.1,
and 2-5 s for a slower controller on UART such as the Arduino Mega. The README pairs the slow band with a 115200 UART
link while the `mega` environment in `platformio.ini` runs faster, so treat a band as a starting point to confirm
against the link the project actually uses. A band names the PC's ping period, and the silence the controller tolerates
follows from the doubling above.

### kKernelCommands

The PC addresses codes 2 through 5 in a KernelCommand message (protocol 4), while codes 0 and 1 mark internal Kernel
states, which the Kernel reports as kernel status 8 when a PC message carries either of them. Firmware authors implement
none of the six, as the Kernel handles them all internally.

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

## Communication error payload

Neither status enumeration below is ever sent as an event code. Every firmware path that fails to send or receive calls
`Communication::SendCommunicationErrorMessage()`, which packs the two-byte array `{Communication status, TransportLayer
status}` as the data object of an error message, then drives `LED_BUILTIN` HIGH. Nothing on the error path clears that
LED. Only `SetupKernel()` drives it LOW again. A `Kernel::Setup()` pass reaches it once no module fails, and the first
`RuntimeCycle()` to observe an incomplete setup runs it as well, before the 2-second toggle takes over. A lit LED
therefore means at least one transfer failed since the last successful setup, not that one is failing right now, and a
failed setup leaves the LED blinking rather than off. The two bytes are the whole firmware-side diagnosis of the
failure, so read them as a pair.

| Failing call                                 | Message emitted | Event code reported                     |
|----------------------------------------------|-----------------|-----------------------------------------|
| `Module::SendData()`, `Module::SendEvent()`  | ModuleData (6)  | Core event code 1, `kTransmissionError` |
| `Kernel::SendData()` and its service senders | KernelData (7)  | Kernel status 4, `kTransmissionError`   |
| `Kernel::ReceiveData()`                      | KernelData (7)  | Kernel status 3, `kReceptionError`      |

One reception failure carries no such payload. When the PC sends a protocol code the firmware does not accept, the
Kernel reports kernel status 5 (`kInvalidMessageProtocol`) with the rejected protocol code as its data object instead.

**Note:** the PC-side mirrors of both enumerations use the same numbering, and every code in either mirror has a
firmware counterpart, so no PC-side code is orphaned. Only the firmware side knows which method sets each code, which is
what the two tables below add.

### kCommunicationStatusCodes

Defined in `axmc_shared_assets.h` and returned by `Communication::get_communication_status()`. Fills the first byte of
the error payload, which in practice carries only 52, 53, 54, and 62, because every call site snapshots the status after
a failed send or receive has already overwritten it. "A sender" below means any of `SendDataMessage()`,
`SendStateMessage()`, and `SendServiceMessage()`, all three of which set codes 54, 55, and 62 identically.

| Code | Constant               | Firmware meaning, with the method that sets the code                                       |
|------|------------------------|--------------------------------------------------------------------------------------------|
| 51   | `kStandby`             | Initializer value. No Communication method has completed yet.                              |
| 52   | `kReceptionError`      | `ReceiveMessage()` failed on packet reception. Byte 2 names the reason.                    |
| 53   | `kParsingError`        | `ReceiveMessage()` or `ExtractModuleParameters()` could not read the arrived payload.      |
| 54   | `kPackingError`        | A sender could not stage the message. Nothing was sent, and the buffer was reset.          |
| 55   | `kMessageSent`         | A sender wrote the whole packet to the port.                                               |
| 56   | `kMessageReceived`     | `ReceiveMessage()` read the protocol code. A header read can still fail after it.          |
| 57   | `kInvalidProtocol`     | The received protocol code is not accepted on reception. Kernel reports kernel status 5.   |
| 58   | `kNoBytesToReceive`    | No message was waiting. The idle path, not an error, and nothing is reported.              |
| 59   | `kParameterMismatch`   | Received bytes are not `sizeof(struct)` plus header. The PC tuple and the struct disagree. |
| 60   | `kParametersExtracted` | `ExtractModuleParameters()` wrote the parameter struct.                                    |
| 61   | `kExtractionForbidden` | `ExtractParameters()` ran while the stored message was not ModuleParameters.               |
| 62   | `kTransmissionError`   | The serial interface accepted only part of the packet a sender wrote.                      |

### kTransportStatusCodes

Defined by the `ataraxis-transport-layer-mc` dependency in `axtlmc_shared_assets.h` and set by `TransportLayer` in
`transport_layer.h`. `Communication::get_transport_layer_status()` returns it, and it fills the second byte of the error
payload. Codes 14, 16, and 27 are governed by the TransportLayer's private `kTimeout` stall window, which firmware
cannot configure.

| Code | Constant                       | Firmware meaning, with the method that sets the code                       |
|------|--------------------------------|----------------------------------------------------------------------------|
| 11   | `kStandby`                     | Initializer value. No packet operation has completed yet.                  |
| 12   | `kDecodingFailed`              | `ValidatePacket()`: CRC passed, but COBS decoding of the payload failed.   |
| 13   | `kPacketSent`                  | `SendData()` wrote the full packet to the port.                            |
| 14   | `kPayloadSizeByteNotFound`     | `ParsePacket()`: start byte arrived, no size byte within the stall window. |
| 15   | `kInvalidPayloadSize`          | `ParsePacket()`: announced size is under the minimum or over the cap.      |
| 16   | `kPacketTimeoutError`          | `ParsePacket()`: the packet body stalled before the delimiter arrived.     |
| 17   | `kNoBytesToParse`              | Nothing available, or no start byte in what was. The idle path.            |
| 18   | `kPacketParsed`                | `ParsePacket()` read the body and the CRC postamble.                       |
| 19   | `kCRCCheckFailed`              | `ValidatePacket()`: CRC failed. Corruption or mismatched CRC parameters.   |
| 20   | `kPacketReceived`              | `ReceiveData()` parsed, validated, and decoded the packet.                 |
| 21   | `kWriteObjectBufferError`      | `WriteData()`: the object exceeds the free transmission payload space.     |
| 22   | `kObjectWrittenToBuffer`       | `WriteData()` staged the object for transmission.                          |
| 23   | `kReadObjectBufferError`       | `ReadData()`: fewer unread payload bytes remain than the object needs.     |
| 24   | `kObjectReadFromBuffer`        | `ReadData()` read the object out of the reception payload.                 |
| 25   | `kDelimiterNotFoundError`      | `ParsePacket()`: a full packet arrived with no delimiter. Corrupted.       |
| 26   | `kDelimiterFoundTooEarlyError` | `ParsePacket()`: the delimiter arrived before the announced packet end.    |
| 27   | `kPostambleTimeoutError`       | `ParsePacket()`: the CRC postamble missed the stall window.                |
| 28   | `kEmptyPayloadError`           | `SendData()` was called with an empty transmission payload.                |
| 29   | `kPacketPartiallySent`         | `SendData()`: the interface accepted only part of the packet.              |

### Reading a pair

```text
{58, 17}  Idle. No message was waiting. Never reaches the PC.
{52, 19}  CRC failure. Suspect line noise, a baud mismatch, or PC-side CRC parameters.
{52, 15}  The PC announced a payload this board's reception buffer cannot hold. Shrink the PC-side message.
{52, 16}  Stalled body. {52, 27} is the same fault in the CRC postamble. The link dropped bytes mid-packet.
{52, 25}  Framing corruption, delimiter never arrived. {52, 26} is the delimiter arriving early.
{53, 23}  A header or parameter read ran past the payload. The message layout and the firmware disagree.
{54, 21}  The outgoing object does not fit the payload. Shrink the data object or split it across messages.
{62, 29}  The port accepted only part of the packet. The host is not draining the link, or the link dropped.
```

Codes 59 and 61 never appear in this payload. `SendCommunicationErrorMessage()` snapshots the status on entry, and
every call site reaches it after a failed send or receive has already replaced a parameter code with 52, 53, 54, or 62.
A parameter fault reports instead as kernel status 7, `kModuleParametersError`, carrying `{module_type, module_id}`.
Align the `PACKED_STRUCT` with the PC-side `send_parameters()` tuple, and call `ExtractParameters()` only from
`SetCustomParameters()`.

A second byte that reports success (18, 20, 22, or 24) means the packet layer was healthy and the fault lies entirely in
the Communication layer.

---

## Command handler patterns

### Immediate command

```cpp
void Echo()
{
    SendData(static_cast<uint8_t>(kStates::kEcho), parameters.echo_value);
    CompleteCommand();
}
```

### Multi-stage command with non-blocking delay

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
- Call `CompleteCommand()` on the final stage
- The `default` case should call `AbortCommand()` to handle unexpected stages

### Sensor polling command

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

`/cpp-style` owns the value template parameter conventions and the static-assertion placement these patterns rely on, in
its class patterns reference.

---

## Build and upload

`/platformio-config` owns the `pio` command set and the per-board `monitor_speed` values `platformio.ini` defines.
Prefer `pio run --environment <board> --target upload` when the project targets several boards, because a command that
names no environment covers every environment `platformio.ini` defines and an upload to a board that is not connected
fails that environment. Re-upload whenever the firmware module, its command/event codes, or its parameter struct change,
after which the flashed controller is ready for the PC-side `MicroControllerInterface` to connect.

### Serial speed

The firmware's `Serial.begin()` rate has to match the `monitor_speed` of the environment being built, and the PC side
has to open the port at that same speed, so hold the rate in a named constant rather than a literal. Nothing reports a
baud-rate fault when the two disagree. The controller runs `RuntimeCycle()` normally, and the PC-side
`MicroControllerInterface` raises a "did not respond to the identification request" error once its retry budget expires.
Read that identification failure as a baud mismatch first, before suspecting the module code. A project that targets
more than one board resolves the rate at compile time, so the wrong constant cannot reach the wrong board.
