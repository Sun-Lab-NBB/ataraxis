---
name: cpp-style
description: >-
  Applies C++ coding conventions when writing, reviewing, or refactoring code. Covers .h, .hpp, and
  .cpp files, Doxygen documentation, naming, formatting, error handling, includes, file ordering,
  template patterns, embedded (Arduino/PlatformIO) conventions, and Python C++ extension
  (nanobind/scikit-build-core) conventions. Use when writing or modifying C++ code, reviewing pull
  requests, or when the user asks about C++ coding standards.
user-invocable: false
---

# C++ code style guide

Applies C++ coding conventions.

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
- Cross-language consistency with Python and C# conventions

**Does not cover:**
- README file conventions (see `/readme-style`)
- Commit message conventions (see `/commit`)
- Skill file and CLAUDE.md conventions (see `/skill-design`)
- Codebase exploration workflows (see `/explore-codebase`)

---

## Workflow

You MUST follow these steps when this skill is invoked.

### Step 1: Read this skill

Read this entire file. The core conventions below apply to ALL C++ code.

### Step 2: Load relevant reference files

Based on the task, load the appropriate reference files:

| Task                                                                                                                | Reference to load                                           |
|---------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------|
| Writing or modifying Doxygen docs / types, or comments                                                              | [doxygen-and-types.md](references/doxygen-and-types.md)     |
| Writing classes, templates, enums, structs, function calls, guard clauses, or formatting (blank lines, line length) | [class-patterns.md](references/class-patterns.md)           |
| Using Arduino/PlatformIO, clang tools, tests, or verifying config files                                             | [libraries-and-tools.md](references/libraries-and-tools.md) |
| Using nanobind extensions, CMake, GIL                                                                               | [libraries-and-tools.md](references/libraries-and-tools.md) |
| Deploying or verifying tool config files                                                                            | [assets/](assets/) directory                                |
| Reviewing code before submission                                                                                    | [anti-patterns.md](references/anti-patterns.md)             |

Load multiple references when the task spans multiple domains.

### Step 3: Apply conventions

Write or modify C++ code following all conventions from this file and the loaded references.

### Step 4: Verify compliance

Complete the verification checklist at the end of this file. Every item must pass before
submitting work. For anti-pattern examples, load
[anti-patterns.md](references/anti-patterns.md).

---

## Cross-language consistency

Projects span Python, C++, and C#. These conventions maximize visual and structural
consistency across languages while respecting each language's idiomatic standards.

**Shared across all languages:**
- 120 character line limit
- 4-space indentation (no tabs)
- Comprehensive documentation on ALL public and private members
- Third-person imperative mood for documentation ("Provides...", "Determines whether...")
- A leading underscore marks a symbol private to the module or class that defines it (`_snake_case`
  for any private Python symbol, `_snake_case` for C++ data members, `_camelCase` for C# fields)
- Full words in identifiers (no abbreviations)
- Guard clauses preferred over deep nesting
- Prose over bullet lists in documentation
- No example/code blocks in documentation (they go stale)
- I/O operations separated from processing logic
- Only full stops and commas separate clauses in documentation prose (no semicolons, no em-dashes)
- State what the code does now, not what it avoids doing or formerly did (positive description)

**Shared between C++ and C# only:**
- Allman brace style (opening braces on new lines; Python uses indentation)

**C++-specific divergences from Python:**
- Methods use PascalCase; accessors use `get_`/`set_` snake_case (not bare snake_case as in Python)
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

**Symbol visibility:**
- A leading underscore marks a symbol private to the translation unit or header that defines it, and a
  symbol referenced from another module is renamed to a public name
- A symbol consumed outside its owning component is exported through that component's public header,
  never by reaching into an internal header, and tests are the sole exception to both rules
- Both rules bind downward as well. A symbol referenced only inside its defining translation unit or
  header keeps the underscore, and a symbol every consumer of which lives inside the owning component
  stays out of that component's public header. A test is not a consumer for either rule
- An asset with no consumer is removed rather than kept, which covers functions, methods, classes,
  constants, enum members, and whole headers. The compiler reports an unused static function and an
  unused local, and it reports nothing about an unused exported declaration, so that case is found by
  reading. A symbol exercised only by its own tests is removed together with those tests

### Project archetypes

C++ code falls into two paradigms with shared style but different constraints:

- **Embedded** (Arduino/PlatformIO): No exceptions, no dynamic allocation, no RTTI, no STL heap
  containers. Uses status codes and boolean returns for error handling. Builds via PlatformIO.
- **Extension** (Python/nanobind): Full STL allowed. Exceptions allowed (nanobind translates them
  to Python exceptions). Builds via CMake + scikit-build-core. Requires GIL management.

All naming, documentation, and tooling conventions apply identically to both. Formatting is identical
except for `AccessModifierOffset` and `IndentAccessModifiers`, which differ per archetype (see
[references/libraries-and-tools.md](references/libraries-and-tools.md)). The verification checklist
marks items that apply to only one paradigm.

---

## Naming conventions

### Variables

Use **full words**, not abbreviations:

| Avoid        | Prefer              |
|--------------|---------------------|
| `pos`, `idx` | `position`, `index` |
| `msg`, `val` | `message`, `value`  |
| `buf`, `cnt` | `buffer`, `count`   |
| `cb`, `sz`   | `callback`, `size`  |

### Identifiers

| Element               | Convention               | Example                                              |
|-----------------------|--------------------------|------------------------------------------------------|
| Classes               | PascalCase               | `TransportLayer`, `EncoderModule`, `COBSProcessor`   |
| Methods               | PascalCase               | `SendData`, `ReceiveData`, `SetupModule`             |
| Accessors             | `get_`/`set_` snake_case | `get_buffer_size`, `set_baud_rate`                   |
| Private members       | `_snake_case`            | `_port`, `_cobs_processor`, `_custom_parameters`     |
| Local variables       | snake_case               | `start_index`, `payload_size`, `new_motion`          |
| Parameters            | snake_case               | `module_type`, `module_id`, `baud_rate`              |
| Constants             | `kPascalCase`            | `kTimeout`, `kSerialBufferSize`, `kCalibrationDelay` |
| Enum types            | `kPascalCase`            | `kCustomStatusCodes`, `kModuleCommands`              |
| Enum values           | `kPascalCase`            | `kStandby`, `kRotatedCW`, `kOpen`                    |
| Template type params  | PascalCase               | `PolynomialType`, `BufferType`                       |
| Template value params | `kPascalCase`            | `kPinA`, `kMaximumTransmittedPayloadSize`            |
| Namespaces            | snake_case               | `axtlmc_shared_assets`, `axmc_communication_assets`  |
| Struct members        | snake_case               | `module_type`, `pulse_duration`, `return_code`       |
| Macros                | UPPER_SNAKE_CASE         | `PACKED_STRUCT`, `ENCODER_USE_INTERRUPTS`            |

### Functions

- Use descriptive verb phrases: `SendData`, `ResetTransmissionBuffer`, `ReadEncoder`
- Private methods use PascalCase (same as public, unlike Python)
- Avoid generic names like `Process`, `Handle`, `DoSomething`

### Accessors (getters and setters)

Accessor methods use `get_`/`set_` snake_case to visually distinguish trivial field access from
methods that perform real work. This follows the Google C++ naming convention:

```cpp
/// Returns the size of the instance's transmission buffer, in bytes.
[[nodiscard]]
static constexpr uint16_t get_transmission_buffer_size()
{
    return kTransmissionBufferSize;
}

/// Returns the size of the payload currently stored in the instance's transmission buffer.
[[nodiscard]]
uint8_t get_bytes_in_transmission_buffer() const
{
    return _transmission_buffer[kBufferLayout::kPayloadSizeIndex];
}
```

At the call site, casing signals intent: PascalCase means work is being done, `get_`/`set_`
snake_case means simple field access.

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

See [class-patterns.md](references/class-patterns.md) for argument-labeling conventions in function calls.

---

## Error handling

### Embedded projects (Arduino/PlatformIO)

Embedded microcontrollers prohibit exceptions, RTTI, and dynamic allocation. Use status codes
and boolean returns:

```cpp
bool RunActiveCommand() override
{
    switch (static_cast<kModuleCommands>(get_active_command()))
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

See [doxygen-and-types.md](references/doxygen-and-types.md) for inline comment conventions and what to avoid.

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

All includes must be at the top of the file, except the variant-gated includes inside a `#ifdef`
target-selection block in `main.cpp`. Include sorting is enforced by **clang-format** — do
not manually reorder. Use angle brackets (`<header.h>`) for library headers and quotes
(`"header.h"`) for local project headers.

---

## File-level ordering

All definitions within a file follow this vertical ordering from top to bottom:

1. **File-level Doxygen comment** (`@file` and `@brief`)
2. **Include guard** (`#ifndef` / `#define`)
3. **Macro definitions** (if any, e.g., `#define ENCODER_USE_INTERRUPTS`)
4. **Include directives**
5. **Using directives** (`using namespace` — allowed in header-only libraries; see the using
   namespace section of [class-patterns.md](references/class-patterns.md))
6. **Namespace declarations** (for shared asset files)
7. **Free constants** (`static constexpr` at file scope)
8. **Enumerations** (`enum class` definitions)
9. **Structs and type definitions**
10. **Class declarations** with members in this order:
    a. `static_assert` statements (compile-time validation)
    b. Public nested enums
    c. Public constructors
    d. Public methods (virtual overrides first, then non-virtual)
    e. Public destructor (`~ClassName() override = default`)
    f. Private nested structs
    g. Private constants (`static constexpr`)
    h. Private member variables
    i. Private methods

### Visibility ordering

Within a class, order by visibility: `public` first, then `private`. Always write access
modifiers explicitly.

### Call-hierarchy ordering

Within each visibility group, definitions should **loosely follow the order in which they are
called** during the class's runtime. When there is no clear call hierarchy, group definitions
**by purpose**. This matches the Python convention of ordering definitions by call sequence
within each visibility group.

For embedded modules, this naturally follows from the lifecycle: `SetupModule()` helpers first,
then `RunActiveCommand()` dispatch helpers, then individual command methods.

### One class per file

Each `.h` file should contain exactly one primary class. The file name must use snake_case and
match the class name converted to snake_case (e.g., `transport_layer.h` contains
`TransportLayer`). Shared asset namespaces with enums and structs may be in a single file (e.g.,
`axtlmc_shared_assets.h`).

---

## Guard clauses and boolean expressions

See [class-patterns.md](references/class-patterns.md) for guard-clause and early-return conventions.

---

## Data member visibility

**Classes** must keep all data members private (`_snake_case`) and expose them through
`get_`/`set_` accessors when external access is needed. This enforces encapsulation and keeps
the class interface explicit.

**Structs** may use public data members (`snake_case`) for passive data holders that have no
invariants or methods beyond simple initialization. Packed structs used for binary serialization
are the primary example.

```cpp
// Class — all data members private, accessed via getters
class TransportLayer
{
    public:
        [[nodiscard]]
        uint8_t get_runtime_status() const
        {
            return _runtime_status;
        }

    private:
        uint8_t _runtime_status = 0;
};

// Struct — public members allowed for passive data
struct CustomRuntimeParameters
{
        uint32_t pulse_duration = 35000;  ///< The time, in microseconds, to keep the valve open.
} PACKED_STRUCT;
```

---

## Blank lines

See [class-patterns.md](references/class-patterns.md) for blank-line placement rules.

---

## Line length and formatting

See [class-patterns.md](references/class-patterns.md) for line length, brace style, and statement formatting.

---

## Configuration files

See [libraries-and-tools.md](references/libraries-and-tools.md) for `.clang-format` and `.clang-tidy` locations.

---

## Related skills

| Skill                | Relationship                                                                                       |
|----------------------|----------------------------------------------------------------------------------------------------|
| `/python-style`      | Provides Python conventions; C++ conventions parallel these                                        |
| `/csharp-style`      | Provides C# conventions; C++ conventions parallel these                                            |
| `/readme-style`      | Provides README conventions; invoke for README tasks                                               |
| `/commit`            | Provides commit message conventions; invoke for commit tasks                                       |
| `/skill-design`      | Provides skill file conventions; invoke for skill authoring tasks                                  |
| `/explore-codebase`  | Provides project context that informs style-compliant code changes                                 |
| `/api-docs`          | Provides Doxygen/Breathe API documentation build conventions                                       |
| `/project-layout`    | Provides the directory tree that C++ source and test files live in                                 |
| `/audit-project`     | Audits the code just written, before it is committed                                               |
| `/platformio-config` | Covers platformio.ini and library.json field/section conventions; cpp-style covers C++ source only |

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

Judgment items. No tool inspects these, so this checklist is their only enforcement. Walk every one
against the code you wrote.
- [ ] Doxygen documentation on all public and private members
- [ ] @brief tags use third-person imperative mood ("Provides..." not "This class provides...")
- [ ] Boolean members documented with "Determines whether..."
- [ ] File-level Doxygen comment with @file and @brief present
- [ ] Doxygen tag order: @brief -> @details -> @warning/@note -> @tparam -> @param -> @returns
- [ ] Sentences in comments and Doxygen blocks stay under 40 words
- [ ] Every member defaults to the @brief line alone, with longer blocks earned by a nameable non-obvious property
- [ ] Each retained sentence survives the cover test (unable to be reconstructed from name, signature, and body)
- [ ] Documentation records current behavior only, never the edit that produced it
- [ ] Edits leave documentation no longer than it started unless the new behavior is harder to derive
- [ ] File-level @brief description is at most 5 sentences, with detail relocated into the members it documents
- [ ] Documentation carries only facts the reader cannot infer from the code (no padding or trivia)
- [ ] Doxygen blocks describe what the code does, with project usage and call-site context left out
- [ ] Peer-format expectations documented only when counter-intuitive enough to mislead the reader
- [ ] Doxygen block length proportional to conceptual difficulty, not to how long the method is
- [ ] @param and @returns descriptions do not restate type information
- [ ] Doxygen accurately describes the method's observable behavior
- [ ] Comments and Doxygen blocks free of typos and grammar errors
- [ ] Inline comments explain why, not what (no narrate-the-code comments)
- [ ] No stale references in comments (closed issues, removed code, outdated TODOs)
- [ ] Prose separators are full stops and commas only, no semicolons or em-dashes (colons and code syntax exempt)
- [ ] Documentation states what the code does, not what it is not or used to be (contrast only when load-bearing)
- [ ] Full words used (no abbreviations like pos, idx, val, buf)
- [ ] Classes use PascalCase
- [ ] Methods use PascalCase (both public and private)
- [ ] Accessors use get_/set_ snake_case (not PascalCase)
- [ ] Class data members are private (_snake_case) with get_/set_ accessors
- [ ] Struct data members may be public (snake_case) for passive data holders
- [ ] Private members use _snake_case prefix
- [ ] Symbols used outside their defining module are public, and cross-component ones use the public header
- [ ] Symbols used only inside their defining translation unit or header keep the underscore, and symbols with
      no consumer outside the owning component stay out of its public header (tests are not consumers)
- [ ] Every asset has a consumer, so functions, methods, classes, constants, enum members, and whole headers
      that nothing outside the test suite references are removed rather than kept
- [ ] Local variables and parameters use snake_case
- [ ] Constants use kPascalCase prefix (static constexpr)
- [ ] Enum types and values use kPascalCase prefix
- [ ] Template type params use PascalCase; value params use kPascalCase
- [ ] Namespaces use snake_case
- [ ] Macros use UPPER_SNAKE_CASE
- [ ] Include guards use LIBRARY_PREFIX_FILE_NAME_H pattern
- [ ] Guard clauses / early returns preferred over deep nesting
- [ ] One primary class per file; file name matches class in snake_case
- [ ] Public members above private members in class definition
- [ ] Inline comments use third person imperative
- [ ] No heavy section separator blocks (// ====== or // ------)
- [ ] No @code/@endcode example blocks in Doxygen documentation
- [ ] Prose used in @brief details (not bullet lists)
- [ ] Accessor docs (get_/set_) are single-sentence /// comments
- [ ] get_/set_ used for trivial field access; PascalCase for methods with side effects
- [ ] Methods ordered by call hierarchy within each visibility group
- [ ] I/O operations separated from processing logic (especially in extension code)
- [ ] Magic numbers replaced with named static constexpr constants
- [ ] Test functions have only @file and @brief (no @param, @returns, or @throws)
- [ ] Linting warnings resolved (not suppressed) unless resolution adds unnecessary complexity
- [ ] NOLINT comments for legitimate clang-tidy false positives only
- [ ] No IDE inspection directives (CLion/ReSharper // noinspection etc.); only clang-tidy // NOLINT suppressions kept

Tooling-enforced items. clang-format and clang-tidy resolve or report each of these, so run them
rather than hand-checking. They stay listed for reviews performed without the tools.
- [ ] All lines <= 120 characters
- [ ] 4-space indentation, no tabs
- [ ] Allman brace style (opening braces on new lines)
- [ ] Include sorting delegated to clang-format (do not manually reorder)
- [ ] Pointer/reference alignment is left (int* pointer, int& reference)
- [ ] Attributes ([[nodiscard]], [[maybe_unused]]) on their own line above the declaration (BreakAfterAttributes: Always)
- [ ] Consecutive assignments aligned (AlignConsecutiveAssignments)
- [ ] Template declarations on separate lines
- [ ] Static methods used when no instance state is accessed (readability-convert-member-functions-to-static)
- [ ] explicit keyword on single-argument constructors (google-explicit-constructor)
- [ ] clang-format applied before commit
- [ ] clang-tidy passes with zero warnings

Embedded-Specific Compliance (skip for extension projects):
- [ ] No exceptions (use status codes and boolean returns)
- [ ] No dynamic memory allocation (no new/delete, no STL containers with heap allocation)
- [ ] No RTTI (no dynamic_cast, no typeid)
- [ ] static_assert used for compile-time template parameter validation
- [ ] Scoped enums (enum class) with explicit backing type (uint8_t)
- [ ] Structs use PACKED_STRUCT macro for binary serialization
- [ ] [[nodiscard]] on const getter methods
- [ ] virtual destructors use = default on leaf and base classes
- [ ] final keyword on leaf classes that should not be subclassed
- [ ] static constexpr for compile-time constants (not #define)

Extension-Specific Compliance (skip for embedded projects):
- [ ] NB_MODULE binding block at end of file with NOLINTNEXTLINE suppression
- [ ] GIL released during blocking operations (nb::gil_scoped_release)
- [ ] CMakeLists.txt uses nanobind_add_module with NB_STATIC
- [ ] Extension class prefixed with C (e.g., CPrecisionTimer) to distinguish from Python wrapper
- [ ] __repr__ method exposed via nanobind using CClassName(key=value) format
- [ ] Multi-line error messages assigned to variable before passing to throw
```
