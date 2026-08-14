# Libraries, tools, and project patterns

Conventions for Arduino/PlatformIO development, Python C++ extensions (nanobind), clang-format, clang-tidy, Doxygen
builds, testing, and project-specific constraints in C++ projects.

---

## Embedded constraints

C++ projects target Arduino-compatible microcontrollers (primarily Teensy 4.1) via PlatformIO. The embedded environment
imposes strict constraints that differ from desktop C++.

### Prohibited features

The following C++ features are **prohibited** in embedded code:

| Feature             | Reason                                                               | Alternative                    |
|---------------------|----------------------------------------------------------------------|--------------------------------|
| Exceptions          | Non-deterministic runtime cost, often disabled in embedded compilers | Status codes, boolean returns  |
| Dynamic allocation  | `new`/`delete` cause heap fragmentation and non-deterministic timing | Static allocation, templates   |
| RTTI                | `dynamic_cast`/`typeid` add runtime overhead                         | `static_cast`, `static_assert` |
| STL containers      | `std::vector`, `std::map` etc. use heap allocation                   | Fixed-size arrays, templates   |
| `<iostream>`        | Large binary size, heap allocation                                   | Serial.print(), SendData()     |
| Virtual inheritance | Complex vtable layout, fragile on embedded platforms                 | Single inheritance only        |
| `<type_traits>`     | Not available in all Arduino toolchains                              | Custom type traits (see below) |

### Custom type traits

When `<type_traits>` is not available, libraries define minimal equivalents:

```cpp
/// Checks if two types are the same.
template <typename T, typename U>
struct is_same
{
    static constexpr bool value = false;
};

template <typename T>
struct is_same<T, T>
{
    static constexpr bool value = true;
};

/// Helper variable template for is_same.
template <typename T, typename U>
constexpr bool is_same_v = is_same<T, U>::value;  // NOLINT(*-dynamic-static-initializers)
```

### Memory model

- All objects are statically allocated at file scope or as class members
- No heap allocation during runtime (deterministic memory usage)
- Packed structs for efficient binary serialization over serial communication
- Fixed-size buffers sized via template parameters at compile time

---

## Arduino/PlatformIO conventions

### File organization

For complete directory trees for PlatformIO library and firmware projects, invoke `/project-layout`.

### Header-only libraries

C++ libraries are **header-only**. All code resides in `.h` files with no separate `.cpp` implementation files. This
simplifies distribution via PlatformIO's library dependency system and enables full template specialization at compile
time.

### Distribution boundary

An asset with no consumer is removed rather than kept, and a published library measures that consumption against its
distribution boundary rather than against this repository. `library.json` fixes the boundary as `export.include` minus
`export.exclude`, so an auditor reads both fields before calling any symbol dead, and `/platformio-config` owns those
two fields.

Every public declaration in an exported header reaches consumers this repository cannot see, so it is consumed. Three
shapes recur. A published class keeps its whole public API, including the accessors that only the test suite calls
inside this repository. A mock or a fake the library exports for downstream projects to test against is kept, and
`stream_mock.h` is the worked case. A member carrying an `@warning` or `@note` that names it a testing or debugging aid
is kept, because that block is the recorded rationale that marks the member as consumed.

Removal governs everything the boundary leaves behind. A private member of an exported header stays under removal,
because no downstream consumer is able to name it. A path `export.exclude` drops stays under removal, `main.cpp` being
the standing case. A firmware project carries no `library.json` and publishes no API, so every symbol in it stays under
removal, and a symbol only its own tests exercise is removed together with those tests. The compiler reports an unused
static function and an unused local, and it reports nothing about an unused exported declaration, so that case is found
by reading.

### main.cpp conventions

The `main.cpp` entry point follows a standard structure:

```cpp
// File-level comment explaining the project and hardware configuration.

// Dependencies
#include <Arduino.h>
#include <communication.h>
#include <kernel.h>
#include <module.h>

// Global object instantiation
Communication axmc_communication(Serial);  // NOLINT(*-interfaces-global-init)

// Compile-time configuration
static constexpr uint32_t kKeepaliveInterval = 500;

// Conditional compilation for target microcontroller variants
#define ACTOR
#ifdef ACTOR
#include "brake_module.h"
#include "valve_module.h"
// Module instantiation with template parameters
BrakeModule<33, false, true> wheel_brake(3, 1, axmc_communication);
ValveModule<35, true, true, 34> reward_valve(5, 1, axmc_communication);
Module* modules[] = {&wheel_brake, &reward_valve};
#endif

// Kernel instantiation
Kernel axmc_kernel(kControllerID, axmc_communication, modules, kKeepaliveInterval);

void setup()
{
    Serial.begin(115200);
    analogReadResolution(12);
    axmc_kernel.Setup();
}

void loop()
{
    axmc_kernel.RuntimeCycle();
}
```

### NOLINT for global initialization

Global object instantiation with non-trivial constructors triggers `cppcoreguidelines-interfaces-global-init`. Suppress
with an inline comment:

```cpp
Communication axmc_communication(Serial);  // NOLINT(*-interfaces-global-init)
TransportLayer<> tl_class(Serial);         // NOLINT(*-interfaces-global-init)
```

### Conditional compilation

Use `#define` / `#ifdef` / `#elif defined` / `#else` / `#endif` for hardware variant selection. Include a
`static_assert(false, ...)` in the `#else` branch to catch missing target definitions:

```cpp
#ifdef ACTOR
// ACTOR configuration
#elif defined SENSOR
// SENSOR configuration
#else
static_assert(false, "Define one of the supported microcontroller targets.");
#endif
```

---

## clang-format configuration

C++ projects use a shared `.clang-format` configuration based on the Google style with extensive customization for
cross-language visual consistency with Python (Black formatter).

The canonical `.clang-format` files are stored in `assets/embedded/` (for PlatformIO projects) and `assets/extension/`
(for nanobind projects). The only differences between them are `AccessModifierOffset` and `IndentAccessModifiers`.

### Running clang-format

```bash
# Format all files in place
clang-format -i src/*.h src/*.cpp

# Check formatting without modifying (CI mode)
clang-format --dry-run --Werror src/*.h src/*.cpp
```

---

## clang-tidy configuration

C++ projects use clang-tidy with `WarningsAsErrors: '*'`, treating all enabled warnings as build-breaking errors. The
canonical configuration is stored in `assets/.clang-tidy` and is shared across both embedded and extension archetypes.

### Every check is named explicitly

The `Checks` list opens with `-*` and then names every enabled check in full. It carries NO globs, so the enabled set is
identical on every clang-tidy version. This matters because clang-tidy is host-provided rather than pinned by the
project environments, so two contributors routinely run different releases. A glob adopts each check a new release adds,
and `WarningsAsErrors: '*'` turns that adoption into a build failure across every project sharing this file.

Naming each check also makes the list auditable. A name clang-tidy has dropped is visible as a dead entry, which a glob
hides. Verify the list against the installed toolchain with `clang-tidy --checks='*' --list-checks` when a new release
lands, and remove the entries it no longer ships.

Adopting a new check is therefore a deliberate edit, which adds its name to the list in its family's alphabetical
position rather than widening a pattern.

### A check that contradicts this skill is disabled, never suppressed

Where an enabled check forbids a construct this skill PRESCRIBES, the check is removed from the `Checks` list and the
reason is recorded in the configuration file header. Do NOT suppress such a check with per-site `// NOLINT` comments,
because the construct is correct everywhere the skill calls for it, so a suppression would have to be repeated at every
occurrence and would read as an exception to a rule the project actually follows.

Two checks are absent for this reason, and both concern the file-scope `using namespace` directive that the "Using
namespace directives" section of [class-patterns.md](class-patterns.md) prescribes for project-internal shared asset
namespaces, and that the nanobind pattern above prescribes for the literals and chrono namespaces:

| Check                             | What it forbids                                            |
|-----------------------------------|------------------------------------------------------------|
| `google-build-using-namespace`    | Any file-scope `using namespace` directive                 |
| `google-global-names-in-headers`  | The same directive specifically inside a header file       |

The second one binds the header-only PlatformIO libraries, where every such directive lives in a `.h` file.

### Suppressing warnings

When a clang-tidy warning is a false positive, suppress it with `// NOLINT` and a specific check pattern:

```cpp
// Suppress specific check
static constexpr bool kOpen = kNormallyClosed ? HIGH : LOW;  // NOLINT(*-dynamic-static-initializers)

// Suppress at global scope (global init with non-trivial constructor)
Communication axmc_communication(Serial);  // NOLINT(*-interfaces-global-init)
```

Rules:
- Only suppress warnings that are confirmed false positives
- Use the most specific suppression pattern possible (e.g., `*-dynamic-static-initializers`)
- Never use blanket `// NOLINT` without a check pattern

### IDE inspection directives

IDE-specific inspection-suppression comments (e.g., CLion/ReSharper `// noinspection` or `// ReSharper disable` lines)
are NOT used and MUST be removed when encountered. clang-tidy is the authoritative linter. Only its `// NOLINT(<check>)`
suppressions bear weight and MUST be preserved.

### Resolution policy

Prefer resolving clang-tidy warnings over suppressing them, unless the resolution would:
- Add a branch, a cast, or a helper that exists only to satisfy the check
- Hurt performance by adding redundant checks
- Force a rename or a restructuring that leaves the code harder to read than the warning it removes

### Magic numbers

When clang-tidy flags magic numbers, prefer defining named `static constexpr` constants:

```cpp
// Good - named constant with documentation
/// The minimum number of bytes required to form a valid packet.
static constexpr uint16_t kMinimumPacketSize = 5;
if (buffer_size < kMinimumPacketSize) { ... }

// Avoid - magic number inline
if (buffer_size < 5) { ... }
```

For array indices, bit shifts, and common factors, suppression with `// NOLINT` and a comment is acceptable.

### Running clang-tidy

```bash
# Run clang-tidy on all source files
clang-tidy src/*.h src/*.cpp -- -I include/

# Run with auto-fix for safe transformations
clang-tidy --fix src/*.h src/*.cpp -- -I include/
```

---

## Doxygen documentation builds

C++ libraries use Doxygen for API documentation, integrated with Sphinx via the Breathe extension. For the Doxyfile, the
build command, and the Sphinx/Breathe configuration conventions, invoke `/api-docs`.

---

## Testing conventions

### PlatformIO test layout

Teensy boards drop the serial connection between separately built test suites, so every test in a library lives in one
flat test file directly under `test/`, named after the component it covers. For that directory tree, invoke
`/project-layout`.

### Test file naming

Test files use the pattern `test_<component>.cpp` and run under the Arduino framework, which supplies its own `main()`.
Tests are therefore registered in `RunUnityTests()` and executed from `setup()`:

```cpp
/**
 * @file
 *
 * @brief Verifies the behavior of the COBSProcessor encode and decode methods.
 */

#include <Arduino.h>
#include <unity.h>  // This is the C testing framework, no connection to the Unity game engine
#include "cobs_processor.h"

/// Called automatically before each test function. Currently unused.
void setUp()
{}

/// Called automatically after each test function. Currently unused.
void tearDown()
{}

/// Verifies the COBSProcessor EncodePayload() method.
void test_encode_empty_payload()
{
    // Arrange, Act, Assert
}

/// Verifies the COBSProcessor DecodePayload() method.
void test_decode_valid_packet()
{
    // Arrange, Act, Assert
}

/// Specifies the test functions to be executed and controls their runtime.
int RunUnityTests()
{
    UNITY_BEGIN();

    // COBS Processor
    RUN_TEST(test_encode_empty_payload);
    RUN_TEST(test_decode_valid_packet);

    return UNITY_END();
}

// Defines the baud rates for different boards.

// For Arduino Due, the maximum non-doubled stable rate is 5.25 Mbps at 84 MHz cpu clock.
#if defined(ARDUINO_SAM_DUE)
static constexpr uint32_t kSerialBaudRate = 5250000;

// For Uno, Mega, and other 16 MHz AVR boards, the maximum stable non-doubled rate is 1 Mbps.
#elif defined(ARDUINO_AVR_UNO) || defined(ARDUINO_AVR_MEGA2560) || defined(ARDUINO_AVR_MEGA) ||  \
    defined(__AVR_ATmega328P__) || defined(__AVR_ATmega32U4__) || defined(__AVR_ATmega2560__) || \
    defined(__AVR_ATmega168__) || defined(__AVR_ATmega1280__) || defined(__AVR_ATmega16U4__)
static constexpr uint32_t kSerialBaudRate = 1000000;

// For all other boards the default 9600 rate is used.
#else
static constexpr uint32_t kSerialBaudRate = 9600;
#endif

/// Runs all tests inside setup() as required by the Arduino framework for one-shot testing.
void setup()
{
    // Starts the serial connection.
    Serial.begin(kSerialBaudRate);

    // Waits ~2 seconds for the Unity test runner to establish the connection with the board Serial interface. For
    // teensy, this is less important, since it uses a USB interface which does not reset the board on connection.
    delay(2000);

    // Runs the required tests.
    RunUnityTests();

    // Stops the serial communication interface.
    Serial.end();
}

/// Intentionally empty. All tests run in setup() as one-shot operations.
void loop()
{}
```

### Harness rules

- Declare `setUp()` and `tearDown()` even when both are empty, as Unity calls them around every test
- Register every test with `RUN_TEST` inside `RunUnityTests()`, grouped by the class or method under test
- Resolve the baud rate through a per-board preprocessor block and pass the result to `Serial.begin()`
- Call `RunUnityTests()` from `setup()` and leave `loop()` empty
- Do NOT declare `int main()` in Arduino test files, where it belongs to a native test environment only. The Arduino
  core supplies `main()` from a static archive, so a test file that declares its own replaces it and the suite reports
  zero executed tests instead of failing to build

### Test naming

Use descriptive snake_case names with the pattern `test_<action>_<scenario>`:

```cpp
void test_encode_empty_payload()
void test_decode_valid_packet()
void test_send_data_exceeds_buffer_size()
```

### Test documentation

- Use `@file` and `@brief` at the file level with "Verifies..." imperative
- Use comments to describe test intent when not obvious from the name
- Do NOT add `@param`, `@returns`, or `@throws` tags to test functions. The `@file` and `@brief` tags are sufficient

---

## I/O separation

Separate I/O operations from processing logic for testability and reuse. This is most relevant for extension code:

```cpp
// Good - I/O separated from logic
/// Reads timer data from the hardware counter.
int64_t ReadHardwareCounter() const
{
    return static_cast<int64_t>(
        std::chrono::steady_clock::now().time_since_epoch().count()
    );
}

/// Computes the elapsed time from the given counter values.
int64_t ComputeElapsed(int64_t start_count, int64_t end_count) const
{
    return end_count - start_count;
}

// Avoid - I/O mixed with logic
/// Reads the hardware counter and computes elapsed time.
int64_t ReadAndComputeElapsed() const
{
    int64_t now = /* hardware read */;
    return now - _start_count;  // I/O and processing combined
}
```

### Guidelines

- I/O functions should only perform I/O (read hardware, write serial, access files)
- Processing functions should take standard data types and return standard data types
- In embedded code, hardware constraints limit this separation, so apply it wherever the hardware access can be lifted
  into its own method without changing the order the hardware requires
- In extension code, this separation enables easier unit testing without hardware dependencies

---

## Serial communication patterns

### Baud rate

Teensy boards ignore the baud rate parameter, but it must be specified for API compatibility:

```cpp
Serial.begin(115200);  // The baudrate is ignored for teensy boards.
```

### ADC resolution

Set ADC resolution explicitly at startup:

```cpp
// AVR boards have a fixed 10-bit ADC and provide no analogReadResolution(), so guard the call.
#if !defined(__AVR__)
analogReadResolution(12);  // 12-bit resolution (0-4095)
#endif
```

### Non-blocking patterns

All runtime code must be non-blocking to allow the kernel to service multiple modules. Never use blocking waits in the
main loop:

```cpp
// Good - non-blocking wait using WaitForMicros
case 2:
    if (!WaitForMicros(_custom_parameters.pulse_duration)) return;
    AdvanceCommandStage();
    return;

// Avoid in runtime - blocking wait
delay(1000);  // Blocks the entire microcontroller for 1 second
```

Exception: `delay()` and `delayMicroseconds()` may be used in `SetupModule()` and calibration routines where blocking is
acceptable.

---

## Python C++ extension conventions

Python libraries use C++ extensions for performance-critical operations. Extensions are built with nanobind and
scikit-build-core, targeting desktop platforms (Windows, Linux, macOS).

### File organization

For the complete directory tree for Python + C++ extension projects, invoke `/project-layout`.

### Naming conventions

Extension-specific naming follows these patterns:

| Element               | Convention           | Example                   |
|-----------------------|----------------------|---------------------------|
| Extension source file | `snake_case_ext.cpp` | `precision_timer_ext.cpp` |
| C++ class (extension) | `CPascalCase`        | `CPrecisionTimer`         |
| nanobind module name  | `snake_case_ext`     | `precision_timer_ext`     |
| Python wrapper class  | `PascalCase`         | `PrecisionTimer`          |

The `C` prefix on extension classes distinguishes the C++ implementation from the Python wrapper class that users
interact with directly.

### nanobind binding patterns

The nanobind module definition appears at the end of the extension source file, after the class implementation:

```cpp
// NOLINTNEXTLINE(performance-unnecessary-value-param)
NB_MODULE(precision_timer_ext, m)
{
    m.doc() = "A sub-microsecond-precise timer and non/blocking delay module.";
    nb::class_<CPrecisionTimer>(m, "CPrecisionTimer")
        .def(nb::init<const std::string&>(), "precision"_a = "us")
        .def("Reset", &CPrecisionTimer::Reset, "Resets the reference point.")
        .def(
            "Delay",
            &CPrecisionTimer::Delay,
            "duration"_a,
            "allow_sleep"_a = false,
            "block"_a = false,
            "Delays for the requested period of time."
        )
        .def("get_precision", &CPrecisionTimer::get_precision, "Returns the current precision of the timer.");
}
```

### Rules

- Use `NOLINTNEXTLINE(performance-unnecessary-value-param)` before the `NB_MODULE` macro
- Use the `"param"_a` literal syntax for keyword argument names (requires `using namespace nb::literals`)
- Provide default values matching the C++ constructor/method defaults
- Include a brief docstring for each exposed method
- Bind all public methods, and leave private implementation details unbound

### Namespace aliases and using directives

Extension files use namespace aliases and using directives for readability:

```cpp
namespace nb = nanobind;

/// Provides the ability to work with Python literal string-options.
using namespace nb::literals;

/// Provides the binding for various clock-related operations.
using namespace std::chrono;
```

### GIL management

When C++ code performs blocking operations (delays, I/O waits), release the Python GIL to allow other Python threads to
run:

```cpp
void Delay(const int64_t duration, const bool allow_sleep = false, const bool block = false) const
{
    auto perform_delay = [&]() {
        // ... blocking operation ...
    };

    if (!block)
    {
        nb::gil_scoped_release release;  // Releases the GIL for the entire scope
        perform_delay();
    }
    else
    {
        perform_delay();  // Keeps GIL held
    }
}
```

### Rules

- Release the GIL (`nb::gil_scoped_release`) during any operation that blocks for more than a trivial duration
- Provide a `block` parameter to let callers choose GIL behavior when appropriate
- Never access Python objects while the GIL is released
- Use lambdas to share delay logic between GIL-released and GIL-held paths

### CMake configuration

Extension projects use CMake with scikit-build-core as the Python build backend:

```cmake
cmake_minimum_required(VERSION 4.1.0)
project(project-name LANGUAGES CXX)
set(LIBRARY_NAME library_name)

# Import nanobind
find_package(Python 3.12 REQUIRED COMPONENTS Interpreter Development.Module)
execute_process(
    COMMAND "${Python_EXECUTABLE}" -m nanobind --cmake_dir
    OUTPUT_STRIP_TRAILING_WHITESPACE OUTPUT_VARIABLE NB_DIR)
list(APPEND CMAKE_PREFIX_PATH "${NB_DIR}")
find_package(nanobind CONFIG REQUIRED)

# Compile extension
nanobind_add_module(module_ext NB_STATIC src/c_extensions/module_ext.cpp)

# Install extension alongside Python package
install(TARGETS module_ext LIBRARY DESTINATION ${LIBRARY_NAME})
```

### pyproject.toml build configuration

The build system section declares scikit-build-core and nanobind as build dependencies:

```toml
[build-system]
requires = ["scikit-build-core>=1,<2", "nanobind>=2,<3"]
build-backend = "scikit_build_core.build"
```

For the version constraints that govern these two entries, invoke `/pyproject-style`. For the matching PlatformIO
dependency constraints, invoke `/platformio-config`. The `scikit_build_core.build` backend handles CMake invocation
automatically during `pip install`.

### `__repr__` for extension classes

Extension classes that are exposed to Python via nanobind should provide a `__repr__` method formatted as
`ClassName(key=value, key=value)`:

```cpp
/// Returns a string representation of the timer instance.
std::string Repr() const
{
    return "CPrecisionTimer(precision=" + get_precision() + ")";
}
```

In the nanobind binding, expose `Repr` as `__repr__`:

```cpp
.def("__repr__", &CPrecisionTimer::Repr)
```

### Rules for `__repr__`

- Format: `CClassName(key_attr=value, key_attr=value)`
- Include only the most important attributes, not every internal field
- Use the C++ class name (with `C` prefix), not the Python wrapper name

### Error message variable pattern

For extension code that throws exceptions with multi-line messages, assign the message to a local variable before
passing it:

```cpp
// Good - message variable for multi-line errors
std::string message = "Unable to start the timer with the requested precision '" + precision +
                      "'. Use one of the supported precision options: 'ns', 'us', 'ms', or 's'.";
throw std::invalid_argument(message);

// Acceptable - short single-line messages passed directly
throw std::invalid_argument("Timer has not been started.");
```

### STL usage in extensions

Unlike embedded code, extension code may use the full C++ standard library:

| Allowed in extensions   | Typical use case                         |
|-------------------------|------------------------------------------|
| `std::string`           | String parameters from Python            |
| `std::chrono`           | High-resolution timing                   |
| `std::thread`           | Sleep-based delays                       |
| `std::filesystem::path` | Cross-platform path construction         |
| `std::invalid_argument` | Error propagation to Python via nanobind |
| `auto` with lambdas     | Callback patterns for GIL management     |

Compose filesystem paths from `std::filesystem::path` values joined with `operator/`. Appending a literal that contains
`/` or `\\` to a path value hard-codes one platform's separator, so that form is replaced with the `operator/`
composition. Embedded code has no filesystem, so this rule binds to extension sources only.

### Differences from embedded C++

| Aspect               | Embedded                     | Extension                           |
|----------------------|------------------------------|-------------------------------------|
| Exceptions           | Prohibited                   | Allowed (nanobind translates to Py) |
| STL containers       | Prohibited (heap allocation) | Allowed                             |
| Dynamic allocation   | Prohibited                   | Allowed when needed                 |
| RTTI                 | Prohibited                   | Available but rarely needed         |
| Build system         | PlatformIO                   | CMake + scikit-build-core           |
| Include guards       | `#ifndef` / `#define`        | Not needed (single .cpp file)       |
| Target platform      | Teensy / Arduino             | Windows, Linux, macOS               |
| AccessModifierOffset | 0                            | -2 (in .clang-format)               |
| Distribution         | Firmware images              | Binary wheels via cibuildwheel      |

---

## Configuration files

Canonical configs are stored in [../assets/](../assets/). When working in a C++ project, verify that `.clang-format` and
`.clang-tidy` in the project root match the canonical versions.

- **Embedded** `.clang-format`: [../assets/embedded/.clang-format](../assets/embedded/.clang-format)
- **Extension** `.clang-format`: [../assets/extension/.clang-format](../assets/extension/.clang-format)
- **Shared** `.clang-tidy`: [../assets/.clang-tidy](../assets/.clang-tidy)

The two `.clang-format` variants differ only in `AccessModifierOffset` (`0` vs `-2`) and `IndentAccessModifiers` (`true`
vs `false`). All other settings are identical. The `.clang-tidy` configuration is shared across both archetypes.
