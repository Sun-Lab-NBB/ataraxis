# Libraries, tools, and project patterns

Conventions for Arduino/PlatformIO development, Python C++ extensions (nanobind), clang-format,
clang-tidy, Doxygen builds, testing, and project-specific constraints in Sun Lab C++ projects.

---

## Embedded constraints

Sun Lab C++ projects target Arduino-compatible microcontrollers (primarily Teensy 4.1) via
PlatformIO. The embedded environment imposes strict constraints that differ from desktop C++.

### Prohibited features

The following C++ features are **prohibited** in Sun Lab embedded code:

| Feature             | Reason                                                               | Alternative                    |
|---------------------|----------------------------------------------------------------------|--------------------------------|
| Exceptions          | Non-deterministic runtime cost; often disabled in embedded compilers | Status codes, boolean returns  |
| Dynamic allocation  | `new`/`delete` cause heap fragmentation and non-deterministic timing | Static allocation, templates   |
| RTTI                | `dynamic_cast`/`typeid` add runtime overhead                         | `static_cast`, `static_assert` |
| STL containers      | `std::vector`, `std::map` etc. use heap allocation                   | Fixed-size arrays, templates   |
| `<iostream>`        | Large binary size, heap allocation                                   | Serial.print(), SendData()     |
| Virtual inheritance | Complex vtable layout, fragile on embedded platforms                 | Single inheritance only        |
| `<type_traits>`     | Not available in all Arduino toolchains                              | Custom type traits (see below) |

### Custom type traits

When `<type_traits>` is not available, Sun Lab libraries define minimal equivalents:

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
inline constexpr bool is_same_v = is_same<T, U>::value;
```

### Memory model

- All objects are statically allocated at file scope or as class members
- No heap allocation during runtime (deterministic memory usage)
- Packed structs for efficient binary serialization over serial communication
- Fixed-size buffers sized via template parameters at compile time

---

## Arduino/PlatformIO conventions

### File organization

For complete directory trees for PlatformIO library and firmware projects, invoke
`/project-layout`.

### Header-only libraries

Sun Lab C++ libraries are **header-only**. All code resides in `.h` files with no separate `.cpp`
implementation files. This simplifies distribution via PlatformIO's library dependency system and
enables full template specialization at compile time.

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

Global object instantiation with non-trivial constructors triggers
`cppcoreguidelines-interfaces-global-init`. Suppress with an inline comment:

```cpp
Communication axmc_communication(Serial);  // NOLINT(*-interfaces-global-init)
TransportLayer<> tl_class(Serial);         // NOLINT(*-interfaces-global-init)
```

### Conditional compilation

Use `#define` / `#ifdef` / `#elif defined` / `#else` / `#endif` for hardware variant selection.
Include a `static_assert(false, ...)` in the `#else` branch to catch missing target definitions:

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

Sun Lab C++ projects use a shared `.clang-format` configuration based on the Google style with
extensive customization for cross-language visual consistency with Python (Black formatter).

The canonical `.clang-format` files are stored in `assets/embedded/` (for PlatformIO projects)
and `assets/extension/` (for nanobind projects). The only differences between them are
`AccessModifierOffset` and `IndentAccessModifiers`.

### Running clang-format

```bash
# Format all files in place
clang-format -i src/*.h src/*.cpp

# Check formatting without modifying (CI mode)
clang-format --dry-run --Werror src/*.h src/*.cpp
```

---

## clang-tidy configuration

Sun Lab C++ projects use clang-tidy with `WarningsAsErrors: '*'`, treating all enabled warnings
as build-breaking errors. The canonical configuration is stored in `assets/.clang-tidy` and is
shared across both embedded and extension archetypes.

### Suppressing warnings

When a clang-tidy warning is a false positive, suppress it with `// NOLINT` and a specific
check pattern:

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

### Running clang-tidy

```bash
# Run clang-tidy on all source files
clang-tidy src/*.h src/*.cpp -- -I include/

# Run with auto-fix for safe transformations
clang-tidy --fix src/*.h src/*.cpp -- -I include/
```

---

## Doxygen documentation builds

Sun Lab C++ libraries use Doxygen for API documentation, integrated with Sphinx via the Breathe
extension. The Doxyfile is at the project root. For Sphinx/Breathe configuration conventions,
invoke `/api-docs`.

### Running Doxygen

```bash
# Generate XML output for Breathe consumption
doxygen Doxyfile

# The XML output is typically directed to docs/source/doxygen/xml/
```

---

## Testing conventions

### PlatformIO native tests

PlatformIO supports native and embedded unit tests:

```text
test/
├── test_cobs/
│   └── test_cobs.cpp
├── test_crc/
│   └── test_crc.cpp
└── test_transport/
    └── test_transport.cpp
```

### Test file naming

Test files use the pattern `test_<component>.cpp`:

```cpp
/**
 * @file
 *
 * @brief Verifies the behavior of the COBSProcessor encode and decode methods.
 */

#include <cobs_processor.h>
#include <unity.h>

void test_encode_empty_payload()
{
    // Arrange, Act, Assert
}

void test_decode_valid_packet()
{
    // Arrange, Act, Assert
}

int main()
{
    UNITY_BEGIN();
    RUN_TEST(test_encode_empty_payload);
    RUN_TEST(test_decode_valid_packet);
    return UNITY_END();
}
```

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
analogReadResolution(12);  // 12-bit resolution (0-4095)
```

### Non-blocking patterns

All runtime code must be non-blocking to allow the kernel to service multiple modules. Never use
blocking waits in the main loop:

```cpp
// Good - non-blocking wait using WaitForMicros
case 2:
    if (!WaitForMicros(_custom_parameters.pulse_duration)) return;
    AdvanceCommandStage();
    return;

// Avoid in runtime - blocking wait
delay(1000);  // Blocks the entire microcontroller for 1 second
```

Exception: `delay()` and `delayMicroseconds()` may be used in `SetupModule()` and calibration
routines where blocking is acceptable.

---

## Python C++ extension conventions

Sun Lab Python libraries use C++ extensions for performance-critical operations. Extensions are
built with nanobind and scikit-build-core, targeting desktop platforms (Windows, Linux, macOS).

### File organization

For the complete directory tree for Python + C++ extension projects, invoke `/project-layout`.

### Naming conventions

Extension-specific naming follows these patterns:

| Element                  | Convention           | Example                      |
|--------------------------|----------------------|------------------------------|
| Extension source file    | `snake_case_ext.cpp` | `precision_timer_ext.cpp`    |
| C++ class (extension)    | `CPascalCase`        | `CPrecisionTimer`            |
| nanobind module name     | `snake_case_ext`     | `precision_timer_ext`        |
| Python wrapper class     | `PascalCase`         | `PrecisionTimer`             |

The `C` prefix on extension classes distinguishes the C++ implementation from the Python wrapper
class that users interact with directly.

### nanobind binding patterns

The nanobind module definition appears at the end of the extension source file, after the class
implementation:

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
        .def("GetPrecision", &CPrecisionTimer::GetPrecision, "Returns the current precision.");
}
```

### Rules

- Use `NOLINTNEXTLINE(performance-unnecessary-value-param)` before the `NB_MODULE` macro
- Use the `"param"_a` literal syntax for keyword argument names (requires `using namespace
  nb::literals`)
- Provide default values matching the C++ constructor/method defaults
- Include a brief docstring for each exposed method
- Bind all public methods; do not expose private implementation details

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

When C++ code performs blocking operations (delays, I/O waits), release the Python GIL to allow
other Python threads to run:

```cpp
void Delay(const int64_t duration, const bool block = false) const
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

- Release the GIL (`nb::gil_scoped_release`) during any operation that blocks for more than a
  trivial duration
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
requires = ["scikit-build-core==0.11.6", "nanobind==2.9.2"]
build-backend = "scikit_build_core.build"
```

Pin exact versions for reproducible builds. The `scikit_build_core.build` backend handles CMake
invocation automatically during `pip install`.

### STL usage in extensions

Unlike embedded code, extension code may use the full C++ standard library:

| Allowed in extensions      | Typical use case                          |
|----------------------------|-------------------------------------------|
| `std::string`              | String parameters from Python             |
| `std::chrono`              | High-resolution timing                    |
| `std::thread`              | Sleep-based delays                        |
| `std::invalid_argument`    | Error propagation to Python via nanobind  |
| `auto` with lambdas        | Callback patterns for GIL management      |

### Differences from embedded C++

| Aspect                | Embedded                       | Extension                             |
|-----------------------|--------------------------------|---------------------------------------|
| Exceptions            | Prohibited                     | Allowed (nanobind translates to Py)   |
| STL containers        | Prohibited (heap allocation)   | Allowed                               |
| Dynamic allocation    | Prohibited                     | Allowed when needed                   |
| RTTI                  | Prohibited                     | Available but rarely needed           |
| Build system          | PlatformIO                     | CMake + scikit-build-core             |
| Include guards        | `#ifndef` / `#define`          | Not needed (single .cpp file)         |
| Target platform       | Teensy / Arduino               | Windows, Linux, macOS                 |
| AccessModifierOffset  | 0                              | -2 (in .clang-format)                 |
| Distribution          | Firmware images                | Binary wheels via cibuildwheel        |
