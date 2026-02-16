# Class design patterns

Conventions for class design, inheritance, templates, enums, structs, and state machines in Sun
Lab C++ projects.

---

## Class hierarchy

Sun Lab C++ libraries use a clear inheritance pattern: abstract base classes define interfaces,
and `final` template classes provide concrete implementations.

### Base classes

Base classes provide pure virtual methods and shared utilities:

```cpp
/**
 * @brief Provides the base class for all hardware module implementations.
 *
 * Derived classes must implement SetCustomParameters(), RunActiveCommand(), and SetupModule()
 * to define their specific hardware interaction logic.
 */
class Module
{
    public:
        /// Overwrites the module's runtime parameters structure with data received from the PC.
        virtual bool SetCustomParameters() = 0;

        /// Resolves and executes the currently active command.
        virtual bool RunActiveCommand() = 0;

        /// Sets the module instance's software and hardware parameters to default values.
        virtual bool SetupModule() = 0;

        /// Default virtual destructor for safe polymorphic deletion.
        virtual ~Module() = default;

    protected:
        /// Sends the specified status code and data to the connected PC.
        template <typename ObjectType>
        void SendData(uint8_t status_code, uint8_t prototype, const ObjectType& object);

        /// Marks the current command as completed.
        void CompleteCommand();

        /// Marks the current command as failed and prevents future executions.
        void AbortCommand();
};
```

### Rules for base classes

- Declare pure virtual methods (`= 0`) for the interface that derived classes must implement
- Provide `virtual ~ClassName() = default` for safe polymorphic deletion
- Place shared utility methods in `protected` scope
- Use `const` member variables for immutable state set at construction time
- Constructor parameters use `const` for value types, `&` for reference types

### Leaf classes

Leaf classes inherit from a base and use `final` to prevent further subclassing:

```cpp
template <const uint8_t kPin, const bool kNormallyEngaged, const bool kStartEngaged = true>
class BrakeModule final : public Module
{
        // static_assert validation of template parameters
        static_assert(
            kPin != LED_BUILTIN,
            "The LED-connected pin is reserved for LED manipulation."
        );

    public:
        // Nested enums for status codes and commands
        enum class kCustomStatusCodes : uint8_t { /* ... */ };
        enum class kModuleCommands : uint8_t { /* ... */ };

        // Constructor delegates to base
        BrakeModule(const uint8_t module_type, const uint8_t module_id, Communication& comm) :
            Module(module_type, module_id, comm)
        {}

        // Virtual overrides
        bool SetCustomParameters() override;
        bool RunActiveCommand() override;
        bool SetupModule() override;

        ~BrakeModule() override = default;

    private:
        // Private implementation
};
```

### Rules for leaf classes

- Always use `final` on classes that should not be subclassed
- Use `override` on all virtual method implementations (enforced by clang-tidy
  `modernize-use-override`)
- Use `~ClassName() override = default` for destructors
- Place `static_assert` statements at the top of the class body, before `public:`
- Constructor should delegate to the base class using the member initializer list

---

## Template patterns

Templates are the primary abstraction mechanism in Sun Lab embedded C++ libraries. Hardware
configuration is specified through template parameters, enabling compile-time specialization.

### Template parameter conventions

Use `const` with value template parameters and descriptive `kPascalCase` names:

```cpp
template <
    const uint8_t kValvePin,
    const bool kNormallyClosed,
    const bool kStartClosed = true,
    const uint8_t kTonePin  = 255,
    const bool kNormallyOff = true,
    const bool kStartOff    = true>
class ValveModule final : public Module
```

For type template parameters, use descriptive PascalCase names:

```cpp
template <typename PolynomialType>
class CRCProcessor
```

### Template parameter validation

Always validate template parameters with `static_assert`:

```cpp
// Validate pin uniqueness
static_assert(kPinA != kPinB, "EncoderModule PinA and PinB cannot be the same!");
static_assert(kPinA != kPinX, "EncoderModule PinA and PinX cannot be the same!");
static_assert(kPinB != kPinX, "EncoderModule PinB and PinX cannot be the same!");

// Validate pin safety
static_assert(
    kPinA != LED_BUILTIN,
    "The LED-connected pin is reserved for LED manipulation. Select a different Channel A "
    "pin for the EncoderModule instance."
);

// Validate type constraints
static_assert(
    is_same_v<PolynomialType, uint8_t> || is_same_v<PolynomialType, uint16_t>
        || is_same_v<PolynomialType, uint32_t>,
    "CRCProcessor only supports uint8_t, uint16_t, or uint32_t polynomial types."
);

// Validate size constraints
static_assert(
    kMaximumTransmittedPayloadSize >= 1,
    "The TransportLayer has to be able to transmit at least 1 byte of payload data."
);
```

### static_assert message format

Messages should clearly state what constraint was violated:
- State the subject and the constraint: `"EncoderModule PinA and PinB cannot be the same!"`
- For remediation advice, include it: `"Select a different Channel A pin for the EncoderModule
  instance."`

### Template instantiation

Templates are instantiated at file scope in `main.cpp` with all configuration parameters:

```cpp
// Instantiate hardware modules with compile-time pin and behavior configuration
BrakeModule<33, false, true> wheel_brake(3, 1, axmc_communication);
ValveModule<35, true, true, 34> reward_valve(5, 1, axmc_communication);
EncoderModule<33, 34, 35, true> wheel_encoder(2, 1, axmc_communication);

// Collect modules into a polymorphic array for the kernel
Module* modules[] = {&wheel_brake, &reward_valve};
```

---

## Enum conventions

### Scoped enums

Always use `enum class` (scoped enums) with an explicit backing type:

```cpp
/// Defines the codes used by each module instance to communicate its runtime state to the PC.
enum class kCustomStatusCodes : uint8_t
{
    kOpen                     = 51,  ///< The valve is open.
    kClosed                   = 52,  ///< The valve is closed.
    kCalibrated               = 53,  ///< The valve has completed a calibration cycle.
    kToneOn                   = 54,  ///< The tone is played.
    kToneOff                  = 55,  ///< The tone is silenced.
    kInvalidToneConfiguration = 56,  ///< The instance is not configured to emit audible tones.
};
```

### Rules

- **Scoped**: Always use `enum class`, never unscoped `enum`
- **Backing type**: Always specify `uint8_t` (or appropriate fixed-width type)
- **Naming**: Both the enum type and its values use `kPascalCase` prefix
- **Values**: Assign explicit integer values when they are part of a communication protocol or
  when specific values are required for interoperability
- **Trailing comma**: Include a trailing comma after the last member
- **Documentation**: Use `///<` trailing comments on each member

### Enum usage

Always use `static_cast` when converting between enum values and their underlying type:

```cpp
switch (static_cast<kModuleCommands>(GetActiveCommand()))
{
    case kModuleCommands::kCheckState: CheckState(); return true;
    case kModuleCommands::kReset: ResetEncoder(); return true;
    default: return false;
}

SendData(static_cast<uint8_t>(kCustomStatusCodes::kOpen));
```

### Nested enums

Module classes nest their status codes and command enums as public members:

```cpp
class EncoderModule final : public Module
{
    public:
        enum class kCustomStatusCodes : uint8_t { /* ... */ };
        enum class kModuleCommands : uint8_t { /* ... */ };
};
```

Shared enums that span multiple classes belong in a dedicated shared assets' namespace:

```cpp
namespace axmc_shared_assets
{
    enum class kCoreStatusCodes : uint8_t { /* ... */ };
}
```

---

## Struct conventions

### Packed structs

Structs used for binary serialization over serial communication must use the `PACKED_STRUCT`
macro:

```cpp
/// Stores the instance's addressable runtime parameters.
struct CustomRuntimeParameters
{
        uint32_t pulse_duration    = 35000;   ///< The time, in microseconds, to keep the valve open.
        uint16_t calibration_count = 500;     ///< The number of times to pulse during calibration.
        uint32_t tone_duration     = 300000;  ///< The time, in microseconds, to keep playing the tone.
} PACKED_STRUCT _custom_parameters;
```

The `PACKED_STRUCT` macro is defined in shared assets headers to ensure cross-compiler
compatibility:

```cpp
#if defined(__GNUC__) || defined(__clang__)
    #define PACKED_STRUCT __attribute__((packed))
#elif defined(_MSC_VER)
    #define PACKED_STRUCT
    #pragma pack(push, 1)
#endif
```

### Rules

- Use `PACKED_STRUCT` for all structs that cross memory boundaries (serial communication,
  DMA transfers)
- Align struct member assignments for readability
- Initialize all members with default values
- Document each member with `///<` trailing comments
- Declare struct instances as private class members with `_snake_case` naming

### Nested structs

Private configuration structs are nested within the class that owns them:

```cpp
class ValveModule final : public Module
{
    private:
        struct CustomRuntimeParameters
        {
                uint32_t pulse_duration = 35000;
                // ...
        } PACKED_STRUCT _custom_parameters;
};
```

---

## Constructor patterns

### Explicit constructors

Use `explicit` on constructors with a single parameter to prevent implicit conversions (enforced
by clang-tidy `google-explicit-constructor`):

```cpp
explicit TransportLayer(Stream& port) :
    _port(port)
{}
```

### Delegating constructors

Use the member initializer list to delegate to the base class:

```cpp
BrakeModule(const uint8_t module_type, const uint8_t module_id, Communication& communication) :
    Module(module_type, module_id, communication)
{}
```

### Constructor initializer formatting

The member initializer list follows the colon on the same line as the constructor signature, with
each initializer on a new line if there are multiple:

```cpp
// Single initializer - same line
explicit TransportLayer(Stream& port) :
    _port(port)
{}

// Multiple initializers - each on new line, aligned
Module(const uint8_t module_type, const uint8_t module_id, Communication& communication) :
    _module_type(module_type),
    _module_id(module_id),
    _communication(communication)
{}
```

---

## Destructor patterns

- Base classes: `virtual ~Module() = default`
- Leaf (final) classes: `~EncoderModule() override = default`
- Place destructors after the last public method, before the `private:` section
- Never implement custom destructors unless the class manages non-RAII resources

---

## State machine pattern

Sun Lab embedded modules use a stage-based state machine for non-blocking command execution. This
allows the kernel to service multiple modules within a single loop iteration:

```cpp
void Pulse()
{
    switch (execution_parameters.stage)
    {
        // Stage 1: Opens the valve
        case 1:
            digitalWriteFast(kValvePin, kOpen);
            SendData(static_cast<uint8_t>(kCustomStatusCodes::kOpen));
            AdvanceCommandStage();
            return;

        // Stage 2: Waits for the requested duration
        case 2:
            if (!WaitForMicros(_custom_parameters.pulse_duration)) return;
            AdvanceCommandStage();
            return;

        // Stage 3: Closes the valve
        case 3:
            digitalWriteFast(kValvePin, kClose);
            SendData(static_cast<uint8_t>(kCustomStatusCodes::kClosed));
            CompleteCommand();
            return;

        default: AbortCommand();
    }
}
```

### Rules

- Each stage performs one atomic action and returns immediately
- Use `AdvanceCommandStage()` to move to the next stage
- Use `CompleteCommand()` to signal successful completion
- Use `AbortCommand()` to signal failure and prevent future executions
- `WaitForMicros()` returns `false` while waiting, allowing the kernel to service other modules
- `default:` always calls `AbortCommand()` to catch invalid stages
- Add a comment before each case label describing the stage's purpose

---

## Command dispatch pattern

The `RunActiveCommand()` method dispatches commands using a switch statement:

```cpp
bool RunActiveCommand() override
{
    switch (static_cast<kModuleCommands>(GetActiveCommand()))
    {
        case kModuleCommands::kSendPulse: Pulse(); return true;
        case kModuleCommands::kToggleOn: Open(); return true;
        case kModuleCommands::kToggleOff: Close(); return true;
        case kModuleCommands::kCalibrate: Calibrate(); return true;
        default: return false;
    }
}
```

### Rules

- Cast the raw command byte to the typed enum using `static_cast`
- Each case delegates to a named private method
- Return `true` for recognized commands, `false` for unrecognized
- Short case labels on a single line (formatted by clang-format)
- Add a `// Comment` before each case label naming the action

---

## Const member variables

Use `const` member variables for values that are set once at construction time and never change:

```cpp
class Module
{
    private:
        /// Stores the module's type identifier.
        const uint8_t _module_type;

        /// Stores the module's unique instance identifier.
        const uint8_t _module_id;
};
```

This pattern ensures immutability of configuration values throughout the object's lifetime,
paralleling the `frozen=True` dataclass convention in Python and `readonly` fields in C#.

---

## Using namespace directives

For shared asset namespaces, use `using namespace` at file scope to bring shared types into the
current scope:

```cpp
#include "axtlmc_shared_assets.h"

using namespace axtlmc_shared_assets;
```

Rules:
- Only use `using namespace` for project-internal shared asset namespaces
- Place `using namespace` after all `#include` directives
- Never use `using namespace std;`
- Never use `using namespace` in header files (pollutes the global namespace for all includers)
