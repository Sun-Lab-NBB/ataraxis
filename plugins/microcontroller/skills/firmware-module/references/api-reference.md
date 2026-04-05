# Module base class API reference

Complete API reference for the `Module` base class and supporting classes in ataraxis-micro-controller.
All signatures are sourced from the library's header files at version 3.0.0.

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

Tracks command execution state. Managed by Kernel and utility methods. End users should not modify
fields directly.

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
    uint32_t recurrent_delay = 0;     // Microseconds between command repetitions.
    elapsedMicros recurrent_timer;    // Measures recurrent command activation delays.
    elapsedMicros delay_timer;        // Measures delays between command stages.
};
```

---

## Pure virtual methods

These three methods MUST be overridden by every Module subclass.

### SetupModule

```cpp
virtual bool SetupModule() = 0;
```

Called by Kernel during `Setup()` and on PC-requested resets. Initializes hardware and sets parameter
defaults. Should not contain blocking logic. Returns `true` on success, `false` on failure (failure
bricks the controller until firmware reset).

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

Called during each Kernel runtime cycle when the module has an active command. Should translate the
command code from `get_active_command()` into a call to the command-specific handler method. Returns
`true` if the command was recognized, `false` otherwise. Does NOT indicate command success.

### Virtual destructor

```cpp
virtual ~Module() = default;
```

Module subclasses should declare an overriding destructor (`~MyModule() override = default;`) to ensure
proper cleanup through base class pointers. The Kernel manages modules via `Module*` arrays, so the
virtual destructor is load-bearing.

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

Ends the active command. Sends a kCommandCompleted (event code 2) message to the PC when any of these
conditions hold: a new command is waiting to replace the current one, no next command is queued (including
after an explicit dequeue), or the command is not recurrent. Resets stage to 0, allowing the next command
to activate. **You MUST call this at the end of every command handler.** Failure to call it deadlocks the
module.

```cpp
void AbortCommand();
```

Cancels the active command. If no new command is pending, resets the command queue to prevent
reactivation. Then calls `CompleteCommand()` internally.

```cpp
void AdvanceCommandStage();
```

Increments the stage counter and resets the delay timer. Use to transition between stages of a
multi-stage command.

### Non-blocking delay

```cpp
[[nodiscard]] bool WaitForMicros(const uint32_t delay_duration) const;
```

Delays execution for the specified microseconds, measured relative to the last stage advancement.
- **Blocking mode** (`noblock=false`): blocks in-place until the duration has passed, always
  returns `true`.
- **Non-blocking mode** (`noblock=true`): returns `true` if the duration has elapsed, `false`
  otherwise. The module returns control to the Kernel, allowing other modules to execute.

### Pin reading

```cpp
[[nodiscard]] static uint16_t AnalogRead(const uint8_t pin, const uint16_t pool_size = 0);
```

Reads the analog pin value. If `pool_size >= 2`, reads and averages that many samples using
half-up rounding. Returns the raw or averaged readout.

```cpp
[[nodiscard]] static bool DigitalRead(const uint8_t pin, const uint16_t pool_size = 0);
```

Reads the digital pin value using `digitalReadFast`. If `pool_size >= 2`, reads and averages that
many samples. Returns `true` (HIGH) or `false` (LOW).

### Data transmission

```cpp
template <typename ObjectType>
void SendData(const uint8_t event_code, const ObjectType& object);
```

Sends a ModuleData message (protocol 6) with event code and typed data object. The prototype code
for the wire protocol is resolved automatically at compile time from ObjectType. Supports all 11
scalar types and C-style arrays at type-specific element counts up to the 248-byte payload cap.
`uint8_t` arrays have the densest count support and can be used as a generic bytes buffer. On
failure, automatically attempts to send an error message and turns on the built-in LED.

```cpp
void SendData(const uint8_t event_code) const;
```

Sends a ModuleState message (protocol 8) with event code only. More efficient than the data-carrying
overload when no payload is needed. Same error handling behavior.

### Parameter extraction

```cpp
template <typename ObjectType>
bool ExtractParameters(ObjectType& storage_object);
```

Unpacks the received ModuleParameters message payload into the specified storage object. Internally
delegates to `Communication::ExtractModuleParameters()`. Returns `true` on success, `false` on
error (size mismatch, wrong message type, etc.).

---

## kCoreStatusCodes

System-reserved event codes used by Module base class methods:

| Code | Constant                | Description                                                         |
|------|-------------------------|---------------------------------------------------------------------|
| 0    | `kStandby`              | Default initialization value.                                       |
| 1    | `kTransmissionError`    | SendData failed to transmit to PC.                                  |
| 2    | `kCommandCompleted`     | Active command finished. Sent automatically by `CompleteCommand()`. |
| 3    | `kCommandNotRecognized` | RunActiveCommand returned false. Sent by Kernel.                    |

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

The Kernel internally doubles the keepalive interval to tolerate brief communication lapses.

### Kernel lifecycle

| Method           | Called From | Description                                               |
|------------------|-------------|-----------------------------------------------------------|
| `Setup()`        | `setup()`   | Calls `SetupModule()` on all modules. Sends setup status. |
| `RuntimeCycle()` | `loop()`    | Receives messages, executes commands, checks keepalive.   |

### kKernelStatusCodes

| Code | Constant                  | Description                                            |
|------|---------------------------|--------------------------------------------------------|
| 0    | `kStandby`                | Not used, reserves 0.                                  |
| 1    | `kSetupComplete`          | Setup succeeded.                                       |
| 2    | `kModuleSetupError`       | A module's SetupModule() failed.                       |
| 3    | `kReceptionError`         | Error receiving a PC message.                          |
| 4    | `kTransmissionError`      | Error sending data to PC.                              |
| 5    | `kInvalidMessageProtocol` | Unknown protocol code received.                        |
| 6    | `kModuleParametersSet`    | Parameters applied to module.                          |
| 7    | `kModuleParametersError`  | Failed to apply parameters.                            |
| 8    | `kCommandNotRecognized`   | Unknown kernel command code.                           |
| 9    | `kTargetModuleNotFound`   | No module with requested type+id.                      |
| 10   | `kKeepAliveTimeout`       | Keepalive message not received in time.                |

---

## Communication constructor

```cpp
explicit Communication(Stream& communication_port);
```

| Parameter            | Type      | Description                                               |
|----------------------|-----------|-----------------------------------------------------------|
| `communication_port` | `Stream&` | Arduino Stream (Serial, USB Serial, etc.).                |

Creates a TransportLayer instance with CRC16 (polynomial 0x1021, init 0xFFFF, final XOR 0x0000).
Reserves up to ~1 kB of RAM (~700 bytes on lower-end boards).
