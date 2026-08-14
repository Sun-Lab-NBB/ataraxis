# Doxygen documentation and type usage

Detailed conventions for C++ Doxygen documentation and type usage across projects.

---

## Doxygen documentation

Use Doxygen documentation comments for all public and private members. C++ projects use the `@tag` syntax (not `\tag`).

### Comment styles

Use `/** ... */` blocks for multi-line documentation and `///` for single-line member docs:

```cpp
/**
 * @brief Wraps an Encoder class instance and provides access to its pulse counter to monitor the direction and
 * magnitude of the rotation measured by the managed quadrature encoder.
 *
 * @warning Both Pin A and Pin B must be hardware interrupt pins to achieve the maximum encoder readout resolution.
 *
 * @tparam kPinA the digital interrupt pin connected to the 'A' channel of the quadrature encoder.
 * @tparam kPinB the digital interrupt pin connected to the 'B' channel of the quadrature encoder.
 * @tparam kPinX the digital interrupt pin connected to the 'X' index channel of the quadrature encoder.
 */
template <const uint8_t kPinA, const uint8_t kPinB, const uint8_t kPinX>
class EncoderModule final : public Module

/// Stores the previous readout of the analog pin.
static uint16_t previous_readout = 0;
```

### Rules

- **Third-person imperative mood**: Use verbs like "Provides...", "Wraps...", "Monitors...", "Tracks..." for ALL members
- **Boolean descriptions**: Use "Determines whether..." for boolean members
- **`///` for inline**: Use single-line `///` for brief member docs (fields, constants, enum values)
- **`/** ... */` for blocks**: Use multi-line blocks for classes, methods, and complex members
- **`@brief` required**: Every `/** ... */` block must begin with `@brief`

### Documentation quality

Beyond structural rules, every Doxygen comment must meet quality criteria that govern information density, readability,
and accuracy.

**Necessary minimalism**: Documentation exists to convey information the reader cannot infer from the code itself. Each
`///` summary, `/** ... */` block, and inline comment keeps only the sentences that survive the cover test below. Do not
pad with restatements, motivational prose, or implementation trivia.

The default for every member is the `@brief` line by itself, followed by the `@tparam`, `@param`, and `@returns` tags
the signature requires. A `@details` paragraph, a second paragraph after `@brief`, or a `@note` is an exception the code
has to earn. It is earned when the member carries a specific property the reader is unable to derive. Such a property is
a non-obvious algorithm, an invariant the signature does not express, a unit or pin convention, a timing characteristic
that constrains call sites, or a hardware failure mode. Name that property to yourself before writing the extra prose.
When no such property can be named, the `@brief` line was already complete.

**The cover test**: Before keeping a documentation sentence, cover it and try to reconstruct it from the member name,
the signature, and the first few lines of the body. A sentence you are able to reconstruct carries no information, so
delete it. Apply the test to one sentence at a time rather than to the block as a whole, because a compliant `@brief`
line frequently sits above three sentences that each fail.

**Behavioral scope**: A Doxygen block describes what the method does, and it stops there. Leave out how the method is
deployed in the project, which runtime stage calls it, which feature depends on it, and why it was introduced. That
context belongs to the `@file` block, the README, or the API documentation, and it goes stale the moment the call sites
change. A method that packs a payload documents the payload it produces, leaving the transmission routine that consumes
it undocumented here.

One exception applies. A block may state that an input arrives in a specific format produced by a named peer method.
That statement is allowed only when the expectation is genuinely counter-intuitive, contradicts the usual convention, or
is exceptional enough that the reader is lost without it. State the constraint and its reason in one sentence. An input
that behaves the way a reader already expects needs no such note.

**Sentence length**: Sentences over 40 words are difficult for humans to parse and must be broken into smaller sentences
at natural clause boundaries. Long sentences in `@brief`, `@details`, and inline comments are a strong signal of
over-explanation.

**Wrap width**: Break a comment or Doxygen line only where it would otherwise pass 120 characters, and fill each line to
that limit before breaking, counting the leading `///` or ` * ` prefix toward the column. Prose wrapped at a narrower
width reads as a rigid block, re-wraps badly at any other viewport, and advertises a limit the project does not set. The
test is mechanical: a wrapped line that ends before column 100 while its next word would still fit under 120 is
re-flowed. A line ending early because the sentence or the paragraph ends, or because it starts a new Doxygen tag, is
already correct.

**Typo-free and grammatical**: Every comment, Doxygen block, and inline annotation must be free of typos and grammatical
errors.

**Length proportionality**: Doxygen block length must be proportional to how hard the code is to understand, which is
independent of how many lines it occupies. A long method that carries out one straightforward task needs a short block,
because its size alone gives the reader nothing extra to learn. A short method warrants a longer description when its
behavior is counter-intuitive or hard to derive, such as one built on dense bit manipulation, register-level access, or
a non-obvious timing invariant. Judge the documentation against the difficulty of the idea and keep it to what the
reader is unable to work out from the code.

**No type-signature restating**: `@param` and `@returns` descriptions must not restate information already conveyed by
the type signature or the parameter names. Replace `@param status_code the uint8_t status code` with `@param status_code
the status code to include in the packet header.` The type is already on the parameter.

**No narrate-the-code comments**: Inline comments must explain non-obvious context, intent, or constraints rather than
narrate what the code already says. Replace `// increment counter` above `counter++` with either no comment, or a
comment that explains why the increment matters at that point.

**No change narration**: Documentation describes the code as it currently stands, never the edit that produced it. Do
not record that a behavior was added, that a case is now handled, that a parameter was renamed, or that a defect was
corrected. The commit message and the pull request body carry that history and are the only place it belongs. When an
edit changes behavior, rewrite the affected sentence to state the new behavior and leave the change itself unrecorded.

**No documentation ratchet**: Editing a member is not a reason to lengthen its documentation. When a change leaves the
documented behavior intact, leave the block exactly as it stands. When a change alters the behavior, rewrite the
affected sentences and delete the ones the change made redundant, so the block ends no longer than it started unless the
new behavior is genuinely harder to derive than the old.

**No stale references**: Comments must not reference closed issue numbers, removed code, deprecated firmware versions,
or outdated TODOs. When the code referenced by a comment is removed, the comment must be removed or rewritten.

**Implementation accuracy**: `@brief` and `@returns` claims must accurately describe the method's observable behavior,
signature, parameter semantics, and return value. A `@returns` that says "the absolute path" for a method that returns a
relative path is a defect, even when the code itself is correct.

**Separator punctuation**: Within Doxygen and comment prose, only the full stop and the comma separate clauses. Do not
use a semicolon, and do not use an em-dash as a separator whether it is typed `--`, `—`, or `–`. A colon is allowed
where it is lexically appropriate, such as introducing an explanation or list. A single hyphen in a compound word, a
list marker, or a numeric range is not an em-dash and is fine. This rule governs prose only. Code stays exempt, so a
statement-terminating `;`, a decrement `--`, or a `--flag` in a CLI reference is left as written.

**Positive description**: State what the code does and what is currently true. Do not define behavior by contrast with
what it does not do ("does X, not Y", "works by X rather than Y"), and do not frame it against former behavior
("previously", "used to", "no longer"). The one exception is a contrast that is load-bearing because it corrects a
counter-intuitive but likely assumption, and it must carry its reason. For example, "Iterates over columns rather than
rows, because the columnar store keeps each column contiguous in memory." Without that reason, drop the contrast and
keep only the positive statement.

### Worked reductions

The rules above name the defects. This pair shows the size of the correction that follows from them. The "Avoid" block
is a realistic over-documentation pattern.

**A self-evident method padded with call-site context and restated types:**

```cpp
// Avoid
/**
 * @brief Resets the overflow tracker.
 *
 * This method resets the overflow tracker to zero. It is called by RunActiveCommand() at the end of every reporting
 * cycle, and the value it clears is later read by the PC-side interface when it reconstructs the traveled distance.
 * The method takes no parameters and returns nothing.
 */
void ResetOverflow();

// Good
/// Clears the accumulated sub-threshold motion so the next reporting cycle starts from zero.
void ResetOverflow();
```

The reduction keeps the one fact the name omits, which is what the tracker accumulates. It drops the caller, the
downstream consumer, and the sentence restating the empty signature.

### Tag ordering

Doxygen tags must appear in this order on every member:

1. `@file`: file identification (file-level comments only)
2. `@brief`: one-line summary (always first in class/method blocks)
3. `@details`: extended description (rarely used, prefer adding paragraphs after `@brief`)
4. `@section`: named sections within file-level or class-level docs
5. `@warning`: critical usage warnings
6. `@note`: important notes that are not warnings
7. `@attention`: attention markers for special considerations
8. `@tparam`: template parameters, in declaration order
9. `@param`: function parameters, in declaration order
10. `@returns`: return value description

Do **not** include `@code` / `@endcode` example blocks in Doxygen documentation. Examples go stale as APIs evolve and
create maintenance debt. The `@brief`, `@param`, and `@returns` tags are sufficient.

Omit tags that do not apply. Never reorder tags within a documentation block.

```cpp
/**
 * @brief Sends the specified data to the connected PC via the serial port.
 *
 * Packages the data into a valid transport layer packet by prepending the preamble, encoding the payload using COBS,
 * and appending the CRC checksum.
 *
 * @warning This method is NOT thread-safe. Do not call from interrupt handlers.
 *
 * @tparam ObjectType the type of the data object to send.
 * @param event_code the event code to include in the packet header.
 * @param object the data object to serialize and send.
 */
template <typename ObjectType>
void SendData(const uint8_t event_code, const ObjectType& object);
```

---

## File-level documentation

Every `.h`, `.hpp`, and `.cpp` file must begin with a file-level Doxygen comment:

```cpp
/**
 * @file
 *
 * @brief Provides the EncoderModule class that monitors and records the data produced by a quadrature encoder.
 *
 * @warning This file is written in a way that is @b NOT compatible with any other library or class that uses
 * AttachInterrupt(). Disable the 'ENCODER_USE_INTERRUPTS' macro defined at the top of the file to make this file
 * compatible with other interrupt libraries.
 */
```

### Rules

- Place the file-level comment before all `#include` directives and the include guard
- `@file` tag with no filename argument (Doxygen auto-detects the filename)
- `@brief` describes the primary class or purpose of the file
- Additional `@warning` or `@note` tags provide important file-level context
- Use third-person imperative mood ("Provides...", "Defines...")
- The `@brief` description is a lean, cohesive chunk of at most 5 sentences. Methodology, caveats, and rationale belong
  in the blocks of the classes, methods, enums, and constants the file defines, so relocate each detail to the member it
  concerns rather than accumulating it here

---

## Class documentation

Every class must have a `@brief` tag describing its purpose. Include `@tparam` tags for template classes and `@warning`
/ `@note` tags for important usage constraints:

```cpp
/**
 * @brief Controls the electromagnetic brake by sending digital or analog Pulse-Width-Modulated (PWM) currents through
 * the brake.
 *
 * @tparam kPin the analog pin connected to the logic terminal of the managed brake's FET-gated power relay.
 * @tparam kNormallyEngaged determines whether the brake is engaged (active) or disengaged (inactive) when unpowered.
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
 * @param object the reference to the object whose memory will be overwritten with the received bytes.
 * @returns true if the requested number of bytes was successfully read, false otherwise.
 */
template <typename ReadObject>
[[nodiscard]]
bool ReadData(ReadObject& object) const
```

### When to use block vs inline

- Use `///` when the summary alone is sufficient (most methods)
- Use `/** ... */` when the method has `@tparam`, `@param`, `@returns`, `@warning`, or `@note` tags
- Virtual method overrides with unchanged semantics may use a brief `///` comment

### Accessor documentation

Accessor methods (`get_`/`set_` snake_case) should use single-line `///` documentation. Keep the summary to a single
sentence:

```cpp
// Good - single-sentence accessor docs
/// Returns the size of the instance's transmission buffer, in bytes.
[[nodiscard]]
static constexpr uint16_t get_transmission_buffer_size()

/// Returns the runtime status of the most recently called method.
[[nodiscard]]
uint8_t get_runtime_status() const

// Avoid - multi-sentence accessor docs (move details to the class @brief instead)
/// Returns the runtime status of the most recently called method. The status is updated after each call to SendData
/// or ReceiveData, and tracks whether the operation succeeded.
[[nodiscard]]
uint8_t get_runtime_status() const
```

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
- Align trailing comments, which clang-format applies through `AlignTrailingComments`
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

Use flowing prose in documentation descriptions rather than bullet lists:

```cpp
// Good - prose explains the relationship between concepts
/**
 * @brief Packages the contents of the transmission buffer and sends them to the connected PC.
 *
 * The method prepends the preamble byte to mark the start of a new packet, encodes the payload using COBS to
 * eliminate delimiter bytes, appends the CRC checksum for error detection, and transmits the complete packet through
 * the serial port. The transmission buffer is reset after each successful send operation.
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

Prefer explicit types over `auto` in most cases. Use `auto` when the type is already stated on the right-hand side
(cast, constructor, template factory) and restating it would be redundant, or when working with complex template types:

```cpp
// Good - explicit types when the type is not obvious from the initializer
const int32_t new_motion = _encoder.readAndReset() * kMultiplier;
const uint16_t signal = AnalogRead<kPin>(_custom_parameters.average_pool_size);
uint8_t test_array[4] = {0, 0, 0, 0};

// Good - auto avoids trivially restating the type already visible in the initializer
const auto delta = static_cast<uint32_t>(abs(_overflow));
const auto start_index = static_cast<uint16_t>(_transmission_buffer[kPayloadSizeIndex]);

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
[[nodiscard]]
bool ReadData(ReadObject& object) const
```

### Integer types

Use fixed-width integer types from `<cstdint>` (or Arduino equivalents) for all integer variables. Never use bare `int`,
`short`, or `long`:

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
template <const uint8_t kPinA, const uint8_t kPinB, const uint8_t kPinX, const bool kInvertDirection = false>
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

Exception: `#define` is required for Arduino library configuration macros (e.g., `ENCODER_USE_INTERRUPTS`) that must
precede header inclusion.

---

## Comments

### Inline comments

- Use third-person imperative mood ("Configures..." not "This section configures...")
- Place above the code, not at end of line (unless short trailing comments)
- Use comments to explain non-obvious logic or provide hardware-specific context

```cpp
// Resets the overflow tracker. The overflow accumulates insignificant motion between reporting cycles to filter
// sensor noise while preserving real displacement.
_overflow = 0;
```

### What to avoid

- Don't reiterate the obvious (e.g., `// Set x to 5` before `x = 5`)
- Don't add Doxygen comments to code you didn't write or modify
- Don't use heavy section separator blocks (e.g., `// ======` or `// ------`)
