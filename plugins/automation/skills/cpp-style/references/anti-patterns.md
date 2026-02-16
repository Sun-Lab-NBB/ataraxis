# Anti-patterns

Common C++ style violations in Sun Lab projects and their corrections. Use this reference when
reviewing code before submission.

---

## Naming violations

| Wrong                          | Correct                                          | Rule                            |
|--------------------------------|--------------------------------------------------|---------------------------------|
| `int pos`                      | `int32_t position`                               | Full words, fixed-width types   |
| `uint8_t idx`                  | `uint8_t index`                                  | Full words                      |
| `float val`                    | `float value`                                    | Full words                      |
| `char* buf`                    | `char* buffer`                                   | Full words                      |
| `int cnt`                      | `int32_t count`                                  | Full words, fixed-width types   |
| `void sendData()`              | `void SendData()`                                | Methods use PascalCase          |
| `void setup_module()`          | `void SetupModule()`                             | Methods use PascalCase          |
| `int _myVar`                   | `int32_t _my_var`                                | Private members use _snake_case |
| `int _myCounter`               | `int32_t _my_counter`                            | Private members use _snake_case |
| `const int TIMEOUT = 100`      | `static constexpr int32_t kTimeout = 100`        | Constants use kPascalCase       |
| `const int MAX_SIZE = 256`     | `static constexpr uint16_t kMaxSize = 256`       | Constants use kPascalCase       |
| `#define TIMEOUT 100`          | `static constexpr int32_t kTimeout = 100`        | Prefer constexpr over define    |
| `enum StatusCodes { Standby }` | `enum class kStatusCodes : uint8_t { kStandby }` | kPascalCase enums               |
| `enum class Codes { Open }`    | `enum class kCodes : uint8_t { kOpen }`          | kPascalCase prefix on both      |
| `namespace MyAssets {}`        | `namespace my_assets {}`                         | Namespaces use snake_case       |
| `int* ptr`                     | `int32_t* pointer`                               | Full words, fixed-width types   |

---

## Documentation violations

| Wrong                                           | Correct                                         | Rule                          |
|-------------------------------------------------|-------------------------------------------------|-------------------------------|
| No documentation on class                       | `/** @brief Provides... */`                     | Document ALL members          |
| No documentation on private method              | `/// Reads the encoder pulse counter.`          | Document ALL members          |
| `// This class manages...`                      | `/** @brief Manages... */`                      | Use Doxygen syntax            |
| `/** \brief Sends data */`                      | `/** @brief Sends data. */`                     | Use @tag not \tag syntax      |
| `@param pin - the pin number`                   | `@param pin the pin number.`                    | No dash after @param          |
| `@brief This method sends...`                   | `@brief Sends...`                               | Third-person imperative       |
| `@brief A class that manages...`                | `@brief Manages...`                             | No articles in brief          |
| `@return the value`                             | `@returns the value.`                           | Use @returns not @return      |
| Missing `@file` at top of file                  | `/** @file @brief Provides... */`               | Every file needs @file        |
| Bullet list in @brief details                   | Flowing prose paragraph                         | Prose over lists              |
| `@param` before `@tparam`                       | `@tparam` before `@param`                       | Follow tag ordering           |
| `bool is_enabled`  `///< Checks if enabled.`    | `bool is_enabled`  `///< Determines whether...` | Boolean "Determines whether"  |

---

## Formatting violations

| Wrong                             | Correct                            | Rule                            |
|-----------------------------------|------------------------------------|---------------------------------|
| `if (x) {`                        | `if (x)\n{`                        | Allman brace style              |
| `class Foo {`                     | `class Foo\n{`                     | Allman brace style              |
| Tabs for indentation              | 4 spaces                           | No tabs                         |
| 80-char line limit                | 120-char line limit                | Match Python/C# limit           |
| `int *pointer`                    | `int* pointer`                     | Left pointer alignment          |
| `int &reference`                  | `int& reference`                   | Left reference alignment        |
| Unaligned consecutive assignments | Aligned `=` operators              | AlignConsecutiveAssignments     |
| Template on same line as class    | Template on separate line          | AlwaysBreakTemplateDeclarations |
| `#ifndef TRANSPORT_LAYER_H`       | `#ifndef AXTLMC_TRANSPORT_LAYER_H` | Library-prefixed include guards |
| `#pragma once`                    | `#ifndef` / `#define` / `#endif`   | Include guards, not pragma once |
| Manual include ordering           | Let clang-format sort              | SortIncludes: CaseSensitive     |

---

## Type and const violations

| Wrong                                  | Correct                                         | Rule                             |
|----------------------------------------|-------------------------------------------------|----------------------------------|
| `int status`                           | `uint8_t status`                                | Fixed-width integer types        |
| `short value`                          | `int16_t value`                                 | Fixed-width integer types        |
| `long duration`                        | `uint32_t duration`                             | Fixed-width integer types        |
| `unsigned int size`                    | `uint32_t size`                                 | Fixed-width integer types        |
| Missing `const` on unchanged local     | `const int32_t new_motion = ...`                | const correctness                |
| Missing `const` on value parameter     | `void Foo(const uint8_t pin)`                   | const value parameters           |
| Missing `explicit` on constructor      | `explicit TransportLayer(Stream& port)`         | Prevent implicit conversions     |
| Missing `[[nodiscard]]` on getter      | `[[nodiscard]] bool ReadData(...) const`        | Mark pure query methods          |
| Missing `override` on virtual          | `bool RunActiveCommand() override`              | Enforced by clang-tidy           |
| Missing `final` on leaf class          | `class EncoderModule final : public Module`     | Prevent unintended subclassing   |
| `auto result = Process()`              | `uint8_t result = Process()`                    | Explicit types preferred         |

---

## Template and class violations

| Wrong                                  | Correct                                | Rule                          |
|----------------------------------------|----------------------------------------|-------------------------------|
| No `static_assert` on template params  | `static_assert(kPinA != kPinB, "...")` | Validate template params      |
| Unscoped `enum` without backing type   | `enum class kCodes : uint8_t`          | Scoped enum with backing type |
| Private struct without `PACKED_STRUCT` | `} PACKED_STRUCT _params;`             | Pack serialized structs       |
| `dynamic_cast<Derived*>(base)`         | `static_cast<Derived*>(base)`          | No RTTI in embedded           |
| `new Object()` in runtime code         | Static allocation                      | No dynamic allocation         |
| `std::vector<int> data`                | Fixed-size array or template param     | No STL heap containers        |
| `throw std::runtime_error("...")`      | Return status code / boolean           | No exceptions in embedded     |
| `try { ... } catch (...) { ... }`      | If/else with error codes               | No exceptions in embedded     |
| Missing virtual destructor on base     | `virtual ~Module() = default`          | Safe polymorphic deletion     |
| Custom destructor on leaf class        | `~EncoderModule() override = default`  | Default unless managing RAII  |
| `using namespace std;`                 | Use qualified names                    | Never use namespace std       |
| `using namespace` in header file       | Only in .cpp files                     | Avoid namespace pollution     |

---

## Embedded-specific violations

| Wrong                                         | Correct                                     | Rule                          |
|-----------------------------------------------|---------------------------------------------|-------------------------------|
| `delay(1000)` in `loop()`                     | `WaitForMicros()` with state machine        | Non-blocking runtime          |
| Blocking `while()` in runtime command         | Stage-based state machine pattern           | Non-blocking runtime          |
| Missing `CompleteCommand()` at end of command | Always call `CompleteCommand()` or `return` | Command lifecycle             |
| Missing `default: AbortCommand()` in switch   | Always include default abort                | Catch invalid commands/stages |
| Missing `// NOLINT` on global init            | `// NOLINT(*-interfaces-global-init)`       | Suppress known false positive |
| Blanket `// NOLINT` without pattern           | `// NOLINT(*-specific-check)`               | Use specific suppression      |
| `float` for microsecond timing                | `uint32_t` for microsecond values           | Integer microsecond timing    |
| Missing `analogReadResolution()` call         | Set in `setup()` before any analog reads    | Explicit ADC configuration    |

---

## Comment violations

| Wrong                                  | Correct                      | Rule                           |
|----------------------------------------|------------------------------|--------------------------------|
| `// This function sends...`            | `// Sends...`                | Third-person imperative        |
| `// ========================`          | (remove separator)           | No heavy separator blocks      |
| `// ---- Section ----`                 | (remove separator)           | No heavy separator blocks      |
| End-of-line comment on complex logic   | Comment above the code block | Place above, not at end        |
| `x = 5;  // Set x to 5`                | `x = 5;`                     | Don't state the obvious        |
| Adding docs to code not written by you | Only document your changes   | Don't add docs to others' code |

---

## Cross-language consistency violations

These violations break the visual consistency across Python, C#, and C++:

| Wrong                                               | Correct                                       | Rule                           |
|-----------------------------------------------------|-----------------------------------------------|--------------------------------|
| 80-char line limit                                  | 120-char line limit                           | All languages use 120          |
| 2-space indentation                                 | 4-space indentation                           | All languages use 4 spaces     |
| Tab indentation                                     | Space indentation                             | All languages use spaces       |
| K&R brace style                                     | Allman brace style                            | C++ and C# use Allman          |
| `Process()` as method name                          | `ProcessData()` or specific verb phrase       | Descriptive verb phrases       |
| `int p` as parameter name                           | `int32_t position` with full word             | Full words across all langs    |
| No documentation on private member                  | Document all members                          | All languages document all     |
| "This class manages..." in doc                      | "Manages..." in doc                           | Imperative mood in all langs   |

---

## Extension-specific violations

Common violations in Python C++ extension (nanobind) code:

| Wrong                                               | Correct                                                    | Rule                               |
|-----------------------------------------------------|------------------------------------------------------------|------------------------------------|
| `NB_MODULE(...)` without NOLINTNEXTLINE             | `// NOLINTNEXTLINE(performance-unnecessary-value-param)`   | Suppress known nanobind warning    |
| Blocking without GIL release                        | `nb::gil_scoped_release release;` before blocking call     | Release GIL during blocking ops    |
| Class named `PrecisionTimer` (same as Python)       | `CPrecisionTimer` with C prefix                            | Distinguish C++ from Py wrapper    |
| Missing `explicit` on single-arg constructor        | `explicit CPrecisionTimer(const std::string& precision)`   | Prevent implicit conversions       |
| Accessing Python objects with GIL released          | Only access C++ state while GIL is released                | Thread safety                      |
| `#include "nanobind/nanobind.h"`                    | `#include <nanobind/nanobind.h>`                           | Angle brackets for library headers |
| Inline binding code mixed with class implementation | NB_MODULE block at end of file, after class                | Separation of concerns             |
| Unpinned nanobind/scikit-build-core versions        | Pin exact versions in pyproject.toml build-system.requires | Reproducible builds                |
