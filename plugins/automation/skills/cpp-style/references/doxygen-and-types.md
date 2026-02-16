# Doxygen documentation and type usage

Detailed conventions for C++ Doxygen documentation and type usage in Sun Lab projects.

---

## Doxygen documentation

Use Doxygen documentation comments for all public and private members. Sun Lab C++ projects use
the `@tag` syntax (not `\tag`). This matches the Doxygen style used across all Sun Lab C++
libraries and parallels the Google-style docstrings in Python and XML documentation in C#.

### Comment styles

Use `/** ... */` blocks for multi-line documentation and `///` for single-line member docs:

```cpp
/**
 * @brief Wraps an Encoder class instance and provides access to its pulse counter to monitor the
 * direction and magnitude of the rotation measured by the managed quadrature encoder.
 *
 * @warning Both Pin A and Pin B must be hardware interrupt pins to achieve the maximum encoder
 * state readout resolution.
 *
 * @tparam kPinA the digital interrupt pin connected to the 'A' channel of the quadrature encoder.
 * @tparam kPinB the digital interrupt pin connected to the 'B' channel of the quadrature encoder.
 */
template <const uint8_t kPinA, const uint8_t kPinB>
class EncoderModule final : public Module

/// Stores the previous readout of the analog pin.
static uint16_t previous_readout = 0;
```

### Rules

- **Imperative mood**: Use verbs like "Provides...", "Wraps...", "Monitors...", "Tracks..." for
  ALL members
- **Boolean descriptions**: Use "Determines whether..." for boolean members
- **`///` for inline**: Use single-line `///` for brief member docs (fields, constants, enum
  values)
- **`/** ... */` for blocks**: Use multi-line blocks for classes, methods, and complex members
- **`@brief` required**: Every `/** ... */` block must begin with `@brief`

### Tag ordering

Doxygen tags must appear in this order on every member. This matches the Doxygen convention used
in C++ reference documentation and parallels the Python and C# tag ordering:

1. `@file` — file identification (file-level comments only)
2. `@brief` — one-line summary (always first in class/method blocks)
3. `@details` — extended description (rarely used; prefer adding paragraphs after `@brief`)
4. `@section` — named sections within file-level or class-level docs
5. `@warning` — critical usage warnings
6. `@note` — important notes that are not warnings
7. `@attention` — attention markers for special considerations
8. `@tparam` — template parameters, in declaration order
9. `@param` — function parameters, in declaration order
10. `@returns` — return value description
11. `@code` / `@endcode` — usage examples (rare; prefer keeping docs concise)

This matches Python ordering (summary → extended description → Notes → Args → Returns) and
C# ordering (summary → remarks → typeparam → param → returns → exception). Omit tags that
do not apply. Never reorder tags within a documentation block.

```cpp
/**
 * @brief Sends the specified data to the connected PC via the serial port.
 *
 * Packages the data into a valid transport layer packet by prepending the preamble, encoding
 * the payload using COBS, and appending the CRC checksum.
 *
 * @warning This method is NOT thread-safe. Do not call from interrupt handlers.
 *
 * @tparam ObjectType the type of the data object to send.
 * @param status_code the status code to include in the packet header.
 * @param prototype the data prototype identifier for the receiving parser.
 * @param object the data object to serialize and send.
 * @returns true if the data was successfully sent, false otherwise.
 */
template <typename ObjectType>
bool SendData(uint8_t status_code, uint8_t prototype, const ObjectType& object);
```

---

## File-level documentation

Every `.h`, `.hpp`, and `.cpp` file must begin with a file-level Doxygen comment:

```cpp
/**
 * @file
 *
 * @brief Provides the EncoderModule class that monitors and records the data produced by a
 * quadrature encoder.
 *
 * @warning This file is written in a way that is @b NOT compatible with any other library or
 * class that uses AttachInterrupt(). Disable the 'ENCODER_USE_INTERRUPTS' macro defined at the
 * top of the file to make this file compatible with other interrupt libraries.
 */
```

### Rules

- Place the file-level comment before all `#include` directives and the include guard
- `@file` tag with no filename argument (Doxygen auto-detects the filename)
- `@brief` describes the primary class or purpose of the file
- Additional `@warning` or `@note` tags provide important file-level context
- Use third-person imperative mood ("Provides...", "Defines...")

---

## Class documentation

Every class must have a `@brief` tag describing its purpose. Include `@tparam` tags for template
classes and `@warning` / `@note` tags for important usage constraints:

```cpp
/**
 * @brief Controls the electromagnetic brake by sending digital or analog Pulse-Width-Modulated
 * (PWM) currents through the brake.
 *
 * @tparam kPin the analog pin connected to the logic terminal of the managed brake's FET-gated
 * power relay.
 * @tparam kNormallyEngaged determines whether the brake is engaged (active) or disengaged
 * (inactive) when unpowered.
 * @tparam kStartEngaged determines the initial state of the brake during class initialization.
 */
template <const uint8_t kPin, const bool kNormallyEngaged, const bool kStartEngaged = true>
class BrakeModule final : public Module
```

---

## Method documentation

Methods use `///` for single-line summaries or `/** ... */` blocks for complex methods:

```cpp
/// Initializes the base Module class.
EncoderModule(const uint8_t module_type, const uint8_t module_id, Communication& communication) :
    Module(module_type, module_id, communication)
{}

/// Overwrites the module's runtime parameters structure with the data received from the PC.
bool SetCustomParameters() override

/// Resolves and executes the currently active command.
bool RunActiveCommand() override
```

For methods with parameters that need documentation, use `/** ... */` blocks:

```cpp
/**
 * @brief Reads the specified number of bytes from the reception buffer into the provided object.
 *
 * @tparam ReadObject the type of the object to overwrite with the received data.
 * @param object the reference to the object whose memory will be overwritten with the received
 * bytes.
 * @returns true if the requested number of bytes was successfully read, false otherwise.
 */
template <typename ReadObject>
[[nodiscard]] bool ReadData(ReadObject& object) const
```

### When to use block vs inline

- Use `///` when the summary alone is sufficient (most methods)
- Use `/** ... */` when the method has `@tparam`, `@param`, `@returns`, `@warning`, or `@note`
  tags
- Virtual method overrides with unchanged semantics may use a brief `///` comment

---

## Enum member documentation

Document every enum member with an inline `///` comment:

```cpp
/// Defines the codes used by each module instance to communicate its runtime state to the PC.
enum class kCustomStatusCodes : uint8_t
{
    kRotatedCCW = 51,  ///< The encoder was rotated in the counterclockwise (CCW) direction.
    kRotatedCW  = 52,  ///< The encoder was rotated in the clockwise (CW) direction.
    kPPR        = 53,  ///< Communicates the estimated Pulse-Per-Revolution (PPR) value.
};
```

Rules:
- Use `///` before the enum type declaration
- Use `///<` (trailing Doxygen) after each enum member value
- Align trailing comments for readability
- Include explicit integer values when they are part of a communication protocol

---

## Struct member documentation

Document struct members with trailing `///<` comments:

```cpp
/// Stores the instance's addressable runtime parameters.
struct CustomRuntimeParameters
{
        uint32_t pulse_duration    = 35000;   ///< The time, in microseconds, to keep the valve open.
        uint16_t calibration_count = 500;     ///< The number of times to pulse the valve during calibration.
        uint32_t tone_duration     = 300000;  ///< The time, in microseconds, to keep playing the tone.
} PACKED_STRUCT _custom_parameters;
```

---

## Prose over lists

Use flowing prose in documentation descriptions rather than bullet lists. This matches the Python
convention of using narrative paragraphs in extended descriptions and the C# convention of using
prose in `<remarks>`:

```cpp
// Good - prose explains the relationship between concepts
/**
 * @brief Packages the contents of the transmission buffer and sends them to the connected PC.
 *
 * The method prepends the preamble byte to mark the start of a new packet, encodes the payload
 * using COBS to eliminate delimiter bytes, appends the CRC checksum for error detection, and
 * transmits the complete packet through the serial port. The transmission buffer is reset after
 * each successful send operation.
 */

// Avoid - bullet lists fragment the explanation
/**
 * @brief Packages the contents of the transmission buffer and sends them to the connected PC.
 *
 * - Prepends the preamble byte
 * - Encodes payload using COBS
 * - Appends CRC checksum
 * - Transmits through serial port
 * - Resets transmission buffer
 */
```

---

## Type usage conventions

### Explicit types

Prefer explicit types over `auto` in most cases. Use `auto` only when the type is immediately
obvious or when working with complex template types:

```cpp
// Good - explicit types
const int32_t new_motion = _encoder.readAndReset() * kMultiplier;
const uint16_t signal = AnalogRead(kPin, _custom_parameters.average_pool_size);
uint8_t test_array[4] = {0, 0, 0, 0};

// Acceptable - type obvious from cast
auto delta = static_cast<uint32_t>(abs(_overflow));

// Acceptable - complex iterator types
auto it = container.begin();
```

### const correctness

Mark variables, parameters, and methods `const` wherever possible:

```cpp
// const local variables for values that don't change
const int32_t new_motion = _encoder.readAndReset() * kMultiplier;
const bool data_received = tl_class.ReceiveData();

// const parameters
EncoderModule(const uint8_t module_type, const uint8_t module_id, Communication& communication)

// [[nodiscard]] on const methods
[[nodiscard]] bool ReadData(ReadObject& object) const
```

### Integer types

Use fixed-width integer types from `<cstdint>` (or Arduino equivalents) for all integer
variables. Never use bare `int`, `short`, or `long`:

```cpp
// Good - fixed-width types
uint8_t status_code = 0;
uint16_t signal_threshold = 300;
int32_t overflow = 0;
uint32_t pulse_duration = 35000;

// Avoid - platform-dependent sizes
int status_code = 0;
short signal_threshold = 300;
long overflow = 0;
```

### Template parameter types

Use `const` with value template parameters to prevent accidental modification:

```cpp
template <const uint8_t kPinA, const uint8_t kPinB, const bool kInvertDirection = false>
class EncoderModule final : public Module
```

### static_assert for type validation

Use `static_assert` with type traits to validate template type parameters at compile time:

```cpp
static_assert(
    is_same_v<PolynomialType, uint8_t> || is_same_v<PolynomialType, uint16_t>
        || is_same_v<PolynomialType, uint32_t>,
    "CRCProcessor only supports uint8_t, uint16_t, or uint32_t polynomial types."
);
```

### static constexpr

Prefer `static constexpr` over `#define` for compile-time constants:

```cpp
// Good - type-safe, scoped constant
static constexpr uint32_t kCalibrationDelay = 300000;

// Avoid - untyped, unscoped macro
#define CALIBRATION_DELAY 300000
```

Exception: `#define` is required for Arduino library configuration macros (e.g.,
`ENCODER_USE_INTERRUPTS`) that must precede header inclusion.
