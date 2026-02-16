---
name: cpp-style
description: >-
  Applies Sun Lab C++ coding conventions when writing, reviewing, or refactoring code. Covers .h,
  .hpp, and .cpp files, Doxygen documentation, naming, formatting, error handling, include
  directives, file ordering, template patterns, embedded (Arduino/PlatformIO) conventions, and
  Python C++ extension (nanobind/scikit-build-core) conventions. Use when writing new C++ code,
  modifying existing code, reviewing pull requests, or when the user asks about C++ coding
  standards.
user-invocable: true
---

# C++ code style guide

Applies Sun Lab C++ coding conventions.

You MUST read this skill and load the relevant reference files before writing or modifying C++
code. You MUST verify your changes against the checklist before submitting.

---

## Scope

**Covers:**
- C++ code style (Doxygen documentation, naming, formatting, error handling)
- Include directive conventions and file organization
- Class design, enums, structs, templates, and inheritance patterns
- Embedded-specific patterns (Arduino/PlatformIO, no exceptions, no dynamic allocation)
- Python C++ extension patterns (nanobind bindings, CMake, GIL management)
- clang-format and clang-tidy tooling conventions
- Cross-language consistency with Python and C# Sun Lab conventions

**Does not cover:**
- README file conventions (invoke `/readme-style`)
- Commit message conventions (invoke `/commit`)
- Skill file and CLAUDE.md conventions (invoke `/skill-design`)
- Codebase exploration workflows (invoke `/explore-codebase`)

---

## Workflow

You MUST follow these steps when this skill is invoked.

### Step 1: Read this skill

Read this entire file. The core conventions below apply to ALL C++ code.

### Step 2: Load relevant reference files

Based on the task, load the appropriate reference files:

| Task                                          | Reference to load                                             |
|-----------------------------------------------|---------------------------------------------------------------|
| Writing or modifying Doxygen docs / types     | [doxygen-and-types.md](references/doxygen-and-types.md)       |
| Writing classes, templates, enums, or structs | [class-patterns.md](references/class-patterns.md)             |
| Using Arduino/PlatformIO, clang tools, tests  | [libraries-and-tools.md](references/libraries-and-tools.md)   |
| Using nanobind extensions, CMake, GIL         | [libraries-and-tools.md](references/libraries-and-tools.md)   |
| Deploying or verifying tool config files      | [assets/](assets/) directory                                  |
| Reviewing code before submission              | [anti-patterns.md](references/anti-patterns.md)               |

Load multiple references when the task spans multiple domains.

### Step 3: Apply conventions

Write or modify C++ code following all conventions from this file and the loaded references.

### Step 4: Verify compliance

Complete the verification checklist at the end of this file. Every item must pass before
submitting work. For anti-pattern examples, load
[anti-patterns.md](references/anti-patterns.md).

---

## Cross-language consistency

Sun Lab projects span Python, C++, and C#. These conventions maximize visual and structural
consistency across languages while respecting each language's idiomatic standards.

**Shared across all languages:**
- 120 character line limit
- 4-space indentation (no tabs)
- Comprehensive documentation on ALL public and private members
- Third-person imperative mood for documentation ("Provides...", "Determines whether...")
- Private members use underscore prefix (`_snake_case` in C++, `_camelCase` in C#)
- Full words in identifiers (no abbreviations)
- Guard clauses preferred over deep nesting
- Allman brace style (C++ and C#; not applicable to Python)

**C++-specific divergences from Python:**
- Methods use PascalCase (not snake_case as in Python)
- Constants use `kPascalCase` prefix (not `_UPPER_SNAKE_CASE` as in Python)
- Enum values use `kPascalCase` prefix (not `UPPER_SNAKE_CASE` as in Python)
- Documentation uses Doxygen `@tags` (not Google-style docstrings)
- Error handling uses status codes and boolean returns (not `console.error()`)

**C++-specific divergences from C#:**
- Private members use `_snake_case` (not `_camelCase` as in C#)
- Constants use `kPascalCase` prefix (not bare PascalCase as in C#)
- Enum values use `kPascalCase` prefix (not bare PascalCase as in C#)
- Namespaces use snake_case (not PascalCase as in C#)
- Local variables use snake_case (not camelCase as in C#)
- Consecutive assignment alignment IS used (C# CSharpier does not support it)

### Project archetypes

Sun Lab C++ code falls into two paradigms with shared style but different constraints:

- **Embedded** (Arduino/PlatformIO): No exceptions, no dynamic allocation, no RTTI, no STL heap
  containers. Uses status codes and boolean returns for error handling. Builds via PlatformIO.
- **Extension** (Python/nanobind): Full STL allowed. Exceptions allowed (nanobind translates them
  to Python exceptions). Builds via CMake + scikit-build-core. Requires GIL management.

All naming, documentation, formatting, and tooling conventions apply identically to both. The
verification checklist marks items that apply to only one paradigm.

---

## Naming conventions

### Variables

Use **full words**, not abbreviations:

| Avoid              | Prefer                              |
|--------------------|-------------------------------------|
| `pos`, `idx`       | `position`, `index`                 |
| `msg`, `val`       | `message`, `value`                  |
| `buf`, `cnt`       | `buffer`, `count`                   |
| `cb`, `sz`         | `callback`, `size`                  |

### Identifiers

| Element               | Convention    | Example                                              |
|-----------------------|---------------|------------------------------------------------------|
| Classes               | PascalCase    | `TransportLayer`, `EncoderModule`, `COBSProcessor`   |
| Methods               | PascalCase    | `SendData`, `ReceiveData`, `SetupModule`             |
| Private members       | `_snake_case` | `_port`, `_cobs_processor`, `_custom_parameters`     |
| Local variables       | snake_case    | `start_index`, `payload_size`, `new_motion`          |
| Parameters            | snake_case    | `module_type`, `module_id`, `baud_rate`              |
| Constants             | `kPascalCase` | `kTimeout`, `kSerialBufferSize`, `kCalibrationDelay` |
| Enum types            | `kPascalCase` | `kCustomStatusCodes`, `kModuleCommands`              |
| Enum values           | `kPascalCase` | `kStandby`, `kRotatedCW`, `kOpen`                    |
| Template type params  | PascalCase    | `PolynomialType`, `BufferType`                       |
| Template value params | `kPascalCase` | `kPinA`, `kMaximumTransmittedPayloadSize`            |
| Namespaces            | snake_case    | `axtlmc_shared_assets`, `axmc_communication_assets`  |
| Struct members        | snake_case    | `module_type`, `pulse_duration`, `return_code`       |
| Macros                | UPPER_SNAKE   | `PACKED_STRUCT`, `ENCODER_USE_INTERRUPTS`            |

### Functions

- Use descriptive verb phrases: `SendData`, `ResetTransmissionBuffer`, `ReadEncoder`
- Private methods use PascalCase (same as public, unlike Python)
- Avoid generic names like `Process`, `Handle`, `DoSomething`

### Constants

Use `static constexpr` with the `kPascalCase` prefix and a Doxygen comment:

```cpp
/// Stores the minimum number of bytes required to form a valid packet.
static constexpr uint16_t kMinimumPacketSize = 5;
```

For constants derived from template parameters, add the `// NOLINT(*-dynamic-static-initializers)`
suppression when clang-tidy reports a false positive:

```cpp
static constexpr int32_t kMultiplier = kInvertDirection ? -1 : 1;  // NOLINT(*-dynamic-static-initializers)
```

---

## Function calls

Prefer clarity to brevity. For functions with multiple parameters of the same type or boolean
parameters, use inline comments to label arguments:

```cpp
// Good - labeled arguments clarify meaning
SendData(
    static_cast<uint8_t>(kCustomStatusCodes::kRotatedCW),  // status_code
    kPrototypes::kOneUint32,                                // prototype
    delta                                                   // value
);

// Acceptable - single argument or meaning obvious
CompleteCommand();
```

---

## Error handling

### Embedded projects (Arduino/PlatformIO)

Embedded microcontrollers prohibit exceptions, RTTI, and dynamic allocation. Use status codes
and boolean returns:

```cpp
bool RunActiveCommand() override
{
    switch (static_cast<kModuleCommands>(GetActiveCommand()))
    {
        case kModuleCommands::kCheckState: CheckState(); return true;
        default: return false;
    }
}
```

### Extension projects (Python/nanobind)

Extension code may throw exceptions for error propagation to Python. nanobind automatically
translates C++ exceptions to Python exceptions:

```cpp
throw std::invalid_argument("Unsupported precision. Use 'ns', 'us', 'ms', or 's'.");
```

### Compile-time validation

Use `static_assert` for compile-time constraint checking on template parameters:

```cpp
static_assert(kPinA != kPinB, "EncoderModule PinA and PinB cannot be the same!");
```

### Error message format

Use a structured format: context ("Unable to..."), constraint ("must be..."), actual value
("but received..."). For runtime errors, include the actual value when available.

---

## Comments

### Inline comments

- Use third person imperative ("Configures..." not "This section configures...")
- Place above the code, not at end of line (unless short trailing comments)
- Use comments to explain non-obvious logic or provide hardware-specific context

```cpp
// Resets the overflow tracker. The overflow accumulates insignificant motion between reporting
// cycles to filter sensor noise while preserving real displacement.
_overflow = 0;
```

### What to avoid

- Don't reiterate the obvious (e.g., `// Set x to 5` before `x = 5`)
- Don't add Doxygen comments to code you didn't write or modify
- Don't use heavy section separator blocks (e.g., `// ======` or `// ------`)

---

## Include directives

### Include guards

Use `#ifndef` / `#define` / `#endif` with a library-prefixed identifier:

```cpp
#ifndef AXTLMC_TRANSPORT_LAYER_H
#define AXTLMC_TRANSPORT_LAYER_H

// ... file contents ...

#endif  //AXTLMC_TRANSPORT_LAYER_H
```

The guard identifier follows the pattern: `LIBRARY_PREFIX_FILE_NAME_H`. Use the library's
abbreviated prefix (e.g., `AXTLMC` for ataraxis-transport-layer-mc, `AXMC` for
ataraxis-micro-controller).

### Include ordering

clang-format enforces include sorting (`SortIncludes: CaseSensitive`). The conventional order is:

1. **Arduino/platform headers**: `<Arduino.h>`
2. **Third-party library headers**: `<Encoder.h>`, `<digitalWriteFast.h>`
3. **Project headers**: `<transport_layer.h>`, `<module.h>`, `<kernel.h>`
4. **Local headers**: `"encoder_module.h"`, `"valve_module.h"`

All includes must be at the top of the file. Include sorting is enforced by **clang-format** — do
not manually reorder. Use angle brackets (`<header.h>`) for library headers and quotes
(`"header.h"`) for local project headers.

---

## File-level ordering

All definitions within a file follow this vertical ordering from top to bottom:

1. **File-level Doxygen comment** (`@file` and `@brief`)
2. **Include guard** (`#ifndef` / `#define`)
3. **Macro definitions** (if any, e.g., `#define ENCODER_USE_INTERRUPTS`)
4. **Include directives**
5. **Using directives** (`using namespace`)
6. **Namespace declarations** (for shared asset files)
7. **Free constants** (`static constexpr` at file scope)
8. **Enumerations** (`enum class` definitions)
9. **Structs and type definitions**
10. **Class declarations** with members in this order:
    a. `static_assert` statements (compile-time validation)
    b. Public nested enums
    c. Public constructors and destructors
    d. Public methods (virtual overrides first, then non-virtual)
    e. Public destructor (`~ClassName() override = default`)
    f. Private nested structs
    g. Private constants (`static constexpr`)
    h. Private member variables
    i. Private methods

### Visibility ordering

Within a class, order by visibility: `public` first, then `private`. Always write access
modifiers explicitly.

### One class per file

Each `.h` file should contain exactly one primary class. The file name must use snake_case and
match the class name converted to snake_case (e.g., `transport_layer.h` contains
`TransportLayer`). Shared asset namespaces with enums and structs may be in a single file (e.g.,
`axtlmc_shared_assets.h`).

---

## Guard clauses and boolean expressions

Prefer early returns (guard clauses) over deeply nested conditionals:

```cpp
void SendPulse()
{
    if (!kOutput)
    {
        AbortCommand();
        return;
    }

    // Main logic at minimal indentation level.
    Pulse();
}
```

---

## Blank lines

- **One blank line** between method definitions within a class (enforced by clang-format
  `SeparateDefinitionBlocks: Always`)
- **One blank line** before access modifiers (`public:`, `private:`)
- **No blank line** after an opening brace or before a closing brace
- **One blank line** between logical groups of statements within a method
- **One blank line** after include blocks before code

---

## Line length and formatting

- Maximum line length: **120 characters** (clang-format `ColumnLimit: 120`)
- Formatter: **clang-format** (config in `.clang-format`)
- Linter: **clang-tidy** (config in `.clang-tidy`, `WarningsAsErrors: '*'`)
- Brace style: **Allman** (opening braces on new lines for all constructs)
- Indentation: **4 spaces** (no tabs)
- Pointer/reference alignment: **Left** (`int* pointer`, `int& reference`)

### Aligned assignments

Consecutive assignments and macros are aligned for readability:

```cpp
_custom_parameters.report_CCW      = true;
_custom_parameters.report_CW       = true;
_custom_parameters.delta_threshold = 15;
```

### Template declarations

Template declarations always appear on a separate line:

```cpp
template <const uint8_t kPinA, const uint8_t kPinB, const bool kInvertDirection = false>
class EncoderModule final : public Module
```

### Short statements

Short case labels and simple if/else statements may appear on a single line (enforced by
clang-format `AllowShortCaseLabelsOnASingleLine: true`):

```cpp
case kModuleCommands::kCheckState: CheckState(); return true;
case kModuleCommands::kReset: ResetEncoder(); return true;
default: return false;

if (kTonePin == 255) _custom_parameters.tone_duration = 0;
```

---

## Configuration files

Canonical configs are stored in [assets/](assets/). When working in a C++ project, verify that
`.clang-format` and `.clang-tidy` in the project root match the canonical versions.

- **Embedded** `.clang-format`: [assets/embedded/.clang-format](assets/embedded/.clang-format)
- **Extension** `.clang-format`: [assets/extension/.clang-format](assets/extension/.clang-format)
- **Shared** `.clang-tidy`: [assets/.clang-tidy](assets/.clang-tidy)

The two `.clang-format` variants differ only in `AccessModifierOffset` (`0` vs `-2`) and
`IndentAccessModifiers` (`true` vs `false`). All other settings are identical. The `.clang-tidy`
configuration is shared across both archetypes.

---

## Related skills

| Skill               | Relationship                                                       |
|---------------------|--------------------------------------------------------------------|
| `/python-style`     | Provides Python conventions; C++ conventions parallel these        |
| `/csharp-style`     | Provides C# conventions; C++ conventions parallel these            |
| `/readme-style`     | Provides README conventions; invoke for README tasks               |
| `/commit`           | Provides commit message conventions; invoke for commit tasks       |
| `/skill-design`     | Provides skill file conventions; invoke for skill authoring tasks  |
| `/explore-codebase` | Provides project context that informs style-compliant code changes |
| `/api-docs`         | Provides Doxygen/Breathe API documentation build conventions       |

---

## Proactive behavior

When reviewing or modifying C++ code, proactively check for style violations and fix them. When
writing new code, apply all conventions from this skill and its references without being asked.
If you notice existing code near your changes that violates conventions, mention it to the user
but do not fix it unless asked.

---

## Verification checklist

**You MUST verify your edits against this checklist before submitting any changes to C++ files.**

```text
C++ Style Compliance:
- [ ] Doxygen documentation on all public and private members
- [ ] @brief tags use third-person imperative mood ("Provides..." not "This class provides...")
- [ ] Boolean members documented with "Determines whether..."
- [ ] File-level Doxygen comment with @file and @brief present
- [ ] Doxygen tag order: @brief -> @details -> @warning/@note -> @tparam -> @param -> @returns
- [ ] All lines <= 120 characters
- [ ] 4-space indentation, no tabs
- [ ] Allman brace style (opening braces on new lines)
- [ ] Full words used (no abbreviations like pos, idx, val, buf)
- [ ] Classes use PascalCase
- [ ] Methods use PascalCase (both public and private)
- [ ] Private members use _snake_case prefix
- [ ] Local variables and parameters use snake_case
- [ ] Constants use kPascalCase prefix (static constexpr)
- [ ] Enum types and values use kPascalCase prefix
- [ ] Template type params use PascalCase; value params use kPascalCase
- [ ] Namespaces use snake_case
- [ ] Macros use UPPER_SNAKE_CASE
- [ ] Include guards use LIBRARY_PREFIX_FILE_NAME_H pattern
- [ ] Include sorting delegated to clang-format (do not manually reorder)
- [ ] Pointer/reference alignment is left (int* pointer, int& reference)
- [ ] Consecutive assignments aligned (AlignConsecutiveAssignments)
- [ ] Template declarations on separate lines
- [ ] Guard clauses / early returns preferred over deep nesting
- [ ] One primary class per file; file name matches class in snake_case
- [ ] Public members above private members in class definition
- [ ] Inline comments use third person imperative
- [ ] No heavy section separator blocks (// ====== or // ------)
- [ ] clang-format applied before commit
- [ ] clang-tidy passes with zero warnings

Embedded-Specific Compliance (skip for extension projects):
- [ ] No exceptions (use status codes and boolean returns)
- [ ] No dynamic memory allocation (no new/delete, no STL containers with heap allocation)
- [ ] No RTTI (no dynamic_cast, no typeid)
- [ ] static_assert used for compile-time template parameter validation
- [ ] Scoped enums (enum class) with explicit backing type (uint8_t)
- [ ] Structs use PACKED_STRUCT macro for binary serialization
- [ ] explicit keyword on single-argument constructors
- [ ] [[nodiscard]] on const getter methods
- [ ] virtual destructors use = default on leaf and base classes
- [ ] final keyword on leaf classes that should not be subclassed
- [ ] static constexpr for compile-time constants (not #define)
- [ ] NOLINT comments for legitimate clang-tidy false positives only

Extension-Specific Compliance (skip for embedded projects):
- [ ] NB_MODULE binding block at end of file with NOLINTNEXTLINE suppression
- [ ] GIL released during blocking operations (nb::gil_scoped_release)
- [ ] CMakeLists.txt uses nanobind_add_module with NB_STATIC
- [ ] Extension class prefixed with C (e.g., CPrecisionTimer) to distinguish from Python wrapper
```
