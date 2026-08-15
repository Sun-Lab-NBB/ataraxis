# XML documentation and type usage

Detailed conventions for C# XML documentation and type usage across projects.

---

## XML documentation

Use XML documentation comments (`///`) for all public and private members. The `/cpp-style` and `/python-style` skills
own the equivalent requirement for their own languages.

### Summary tags

Every class, method, field, property, and enum member must have a `<summary>` tag:

```csharp
/// <summary>Tracks animal occupancy duration within a zone and manages boundary arm/disarm state.</summary>
public class OccupancyZone : MonoBehaviour
{
    /// <summary>The duration in milliseconds that the animal must occupy the zone to disarm the boundary.</summary>
    public float occupancyDurationMs = 1000f;

    /// <summary>Determines whether the animal is currently inside this zone.</summary>
    [HideInInspector]
    public bool inZone = false;
}
```

### Rules

- **Third-person imperative mood**: Use verbs like "Provides...", "Defines...", "Configures...", "Tracks..." for ALL
  members
- **Boolean descriptions**: Use "Determines whether..." for boolean fields and properties
- **Single-line format**: Use single-line `<summary>` for most members
- **Multi-line format**: Use multi-line `<summary>` only when the description exceeds 120 characters
- **Wrap width**: Fill each `///` line to 120 characters before breaking it, because prose wrapped narrower stays a
  rigid block that CSharpier never reflows. A wrapped line ending before column 100 while its next word would still fit
  under 120 is re-flowed, and a line ending because its sentence or its paragraph ends is already correct

```csharp
// Single-line (preferred for most members)
/// <summary>Initializes the occupancy timer.</summary>
private void Start()

// Multi-line (when description is long)
/// <summary>
/// The duration in milliseconds that the animal must occupy the zone to disarm the boundary.
/// Set at task creation time from the task template.
/// </summary>
public float occupancyDurationMs = 1000f;
```

### Documentation quality

Beyond structural rules, every XML doc comment must meet quality criteria that govern information density, readability,
and accuracy.

**Necessary minimalism**: Documentation exists to convey information the reader cannot infer from the code itself. Each
`<summary>`, `<remarks>`, and inline comment must be as short as possible while still conveying every necessary fact. Do
not pad with restatements, motivational prose, or implementation trivia.

The default for every member is the single-line `<summary>` by itself, followed by the `<typeparam>`, `<param>`,
`<returns>`, and `<exception>` tags the signature requires. A `<remarks>` block is an exception the code has to earn. It
is earned when the member carries a specific property the reader is unable to derive. Such a property is a non-obvious
algorithm, an invariant the signature does not express, a unit or coordinate convention, a per-frame cost that
constrains call sites, or a lifecycle requirement. Name that property to yourself before writing the extra prose. When
no such property can be named, the `<summary>` line was already complete.

**The cover test**: Before keeping a documentation sentence, cover it and try to reconstruct it from the member name,
the signature, and the first few lines of the body. A sentence you are able to reconstruct carries no information, so
delete it. Apply the test to one sentence at a time rather than to the block as a whole, because a compliant `<summary>`
line frequently sits above three sentences that each fail.

**Behavioral scope**: An XML doc describes what the member does, and it stops there. Leave out how the member is
deployed in the project, which scene or task calls it, which feature depends on it, and why it was introduced. That
context belongs to the file-level summary, the README, or the API documentation, and it goes stale the moment the call
sites change. A method that computes a position documents the position it computes, leaving the component that later
applies it undocumented here.

One exception applies. An XML doc may state that an input arrives in a specific format produced by a named peer method.
That note is warranted only when the expectation is genuinely counter-intuitive, contradicts the usual convention, or is
exceptional enough that the reader is lost without it. State the constraint and its reason in one sentence. An input
that behaves the way a reader already expects needs no such note.

**Sentence length**: Sentences over 40 words are difficult for humans to parse and must be broken into smaller sentences
at natural clause boundaries. Long sentences in `<summary>`, `<remarks>`, and inline comments are a strong signal of
over-explanation.

**Typo-free and grammatical**: Every comment, XML doc tag, and inline annotation must be free of typos and grammatical
errors.

**Length proportionality**: XML doc length must be proportional to how hard the code is to understand, which is
independent of how many lines it occupies. A long method that carries out one straightforward task needs a short
`<summary>`, because its size alone gives the reader nothing extra to learn. A short method warrants a longer
description when its behavior is counter-intuitive or hard to derive, such as one built on dense bit manipulation, an
unusual algorithm, or a non-obvious invariant. Judge the documentation against the difficulty of the idea and keep it to
what the reader is unable to work out from the code.

**No type-signature restating**: `<param>` and `<returns>` descriptions must not restate information already conveyed by
the type signature or the parameter names. Replace `<param name="count">The integer count.</param>` with `<param
name="count">The number of trials to sample.</param>`. The type is already on the parameter.

**No narrate-the-code comments**: Inline comments must explain non-obvious context, intent, or constraints, leaving what
the code already says unwritten. Replace `// increment counter` above `counter++` with either no comment, or a comment
that explains why the increment matters at that point.

**No change narration**: Documentation describes the code as it currently stands, never the edit that produced it. Do
not record that a behavior was added, that a case is now handled, that a parameter was renamed, or that a defect was
corrected. The commit message and the pull request body carry that history and are the only place it belongs. When an
edit changes behavior, rewrite the affected sentence to state the new behavior and leave the change itself unrecorded.

**No documentation ratchet**: Editing a member is not a reason to lengthen its documentation. When a change leaves the
documented behavior intact, leave the XML doc exactly as it stands. When a change alters the behavior, rewrite the
affected sentences and delete the ones the change made redundant, so the block ends no longer than it started unless the
new behavior is genuinely harder to derive than the old.

**No stale references**: Comments must not reference closed issue numbers, removed code, deprecated package versions, or
outdated TODOs. When the code referenced by a comment is removed, the comment must be removed or rewritten.

**Implementation accuracy**: `<summary>` and `<returns>` claims must accurately describe the method's observable
behavior, signature, parameter semantics, and return value. A `<summary>` that says "Returns the absolute path" for a
method that returns a relative path is a defect, even when the code itself is correct.

**Separator punctuation**: Within XML doc and comment prose, only the full stop and the comma separate clauses. Do not
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

The rules above name the defects. These pairs show the size of the correction that follows from them.

**A self-evident method padded with call-site context and restated types:**

```csharp
// Avoid
/// <summary>Resets the occupancy timer.</summary>
/// <remarks>
/// This method resets the occupancy timer back to zero. It is called by the Task component when a new trial begins, and
/// the timer it clears is later read by the reward controller to decide whether the animal has met the occupancy
/// requirement. The method takes no parameters.
/// </remarks>
public void ResetTimer()

// Good
/// <summary>Clears the accumulated occupancy time so the next trial starts from zero.</summary>
public void ResetTimer()
```

The reduction keeps the one fact the name omits, which is what the timer accumulates. It drops the caller, the
downstream consumer, and the sentence restating the empty signature.

**A field documented with its own type:**

```csharp
// Avoid
/// <summary>A float value that stores the occupancy duration.</summary>
public float occupancyDurationMs = 1000f;

// Good
/// <summary>The duration in milliseconds that the animal must occupy the zone to disarm the boundary.</summary>
public float occupancyDurationMs = 1000f;
```

**Comments that narrate the code and record the edit:**

```csharp
// Avoid
// Loop over the zones.
foreach (OccupancyZone zone in _zones)
{
    // Now also skips disabled zones, which was added to fix the null reference.
    if (!zone.enabled) continue;
}

// Good
foreach (OccupancyZone zone in _zones)
{
    // Disabled zones keep stale timer state, so including them would disarm the boundary early.
    if (!zone.enabled) continue;
}
```

---

### Parameter tags

Use `<param>` tags for method parameters:

```csharp
/// <summary>Samples an index from a probability distribution.</summary>
/// <param name="probabilities">The array of probabilities that must sum to 1.0.</param>
/// <param name="random">The random number generator instance.</param>
/// <returns>The sampled index.</returns>
private int SampleFromDistribution(float[] probabilities, System.Random random)
```

### Returns tags

Use `<returns>` for methods that return a value:

```csharp
/// <summary>Returns the elapsed time in milliseconds since the occupancy timer started.</summary>
/// <returns>The elapsed time in milliseconds.</returns>
public float GetElapsedMs()
{
    return _occupancyTimer.ElapsedMilliseconds;
}
```

For simple getters, the `<returns>` tag may be omitted if the `<summary>` already describes the return value.

### Remarks tags

Use `<remarks>` for extended descriptions, implementation notes, or important context:

```csharp
/// <summary>
/// Provides the Task class that manages an infinite corridor VR task.
/// </summary>
/// <remarks>
/// The task creates a looping corridor by instantiating corridor segments and managing probabilistic transitions
/// between them. Each corridor combination is a child GameObject containing the segment prefabs positioned end-to-end.
/// </remarks>
public class Task : MonoBehaviour
```

### Tag ordering

XML documentation tags must appear in this order on every member:

1. `<summary>`: always first
2. `<remarks>`: extended description, notes, warnings
3. `<typeparam>`: type parameters, in declaration order
4. `<param>`: method parameters, in declaration order
5. `<returns>`: return value description
6. `<exception>`: documented exceptions, in alphabetical order by type

The `/cpp-style` and `/python-style` skills define the equivalent ordering for their own documentation blocks. Omit tags
that do not apply. Never reorder tags within a documentation block.

```csharp
/// <summary>Loads and parses a task template from the specified YAML file.</summary>
/// <remarks>
/// The YAML file must conform to the task template schema defined in the project wiki.
/// </remarks>
/// <typeparam name="TTemplate">The template type to deserialize into.</typeparam>
/// <param name="configPath">The path to the YAML configuration file.</param>
/// <param name="validate">Determines whether to validate the template after loading.</param>
/// <returns>The deserialized task template, or null if loading fails.</returns>
/// <exception cref="FileNotFoundException">
/// The file at <paramref name="configPath"/> does not exist.
/// </exception>
public TTemplate? LoadTemplate<TTemplate>(string configPath, bool validate = true)
```

### Exception tags

Use `<exception>` tags to document exceptions that a method may throw, the tag `/cpp-style` and `/python-style` require
in their own documentation blocks:

```csharp
/// <summary>Opens the serial port connection to the microcontroller.</summary>
/// <exception cref="InvalidOperationException">
/// The port is already open.
/// </exception>
/// <exception cref="IOException">
/// The serial port could not be opened due to a hardware or driver error.
/// </exception>
public void OpenConnection()
```

Rules:
- Document all exceptions that callers should handle or be aware of
- Use `cref` to reference the exception type (enables IDE navigation)
- Use `<paramref>` within exception descriptions to reference parameters
- Order multiple `<exception>` tags alphabetically by exception type name
- In Unity MonoBehaviour methods, prefer `Debug.LogError` over throwing (see SKILL.md)

### See cref references

Use `<see cref="MemberOrType"/>` inside `<summary>` and `<remarks>` to link to other fields, constants, methods,
classes, interfaces, and generic types. The tag accepts every member kind, and exception types are only one of them.
This enables IDE navigation between related members:

```csharp
/// <summary>Resets the corridor to the configured <see cref="trackSeed"/> for reproducible runs.</summary>
public void ResetZone()
```

### Inheritdoc

Use `<inheritdoc/>` when overriding a method or implementing an interface where the base documentation is sufficient:

```csharp
/// <inheritdoc/>
public override void ResetState()
{
    _occupancyTimer.Reset();
    inZone = false;
}
```

Rules:
- Use `<inheritdoc/>` only when the base documentation fully describes the override's behavior
- Add a new `<summary>` if the override changes behavior beyond what the base documents
- Use `<inheritdoc cref="InterfaceName.MethodName"/>` for explicit interface implementations

### Warning and note patterns

Use `<remarks>` with bold text for warnings and notes (C# XML docs do not have native `@warning` or `@note` tags like
Doxygen):

```csharp
/// <summary>Resets the encoder pulse counter to zero.</summary>
/// <remarks>
/// <b>Warning:</b> This method should only be called when the encoder is stationary.
/// Calling it during active rotation may cause pulse count discontinuities.
/// </remarks>
public void ResetCounter()
```

### Prose over lists in remarks

Use flowing prose in `<remarks>` blocks rather than bullet lists, the same narrative form `/python-style` requires of an
extended description:

```csharp
// Good - prose explains the relationship between concepts
/// <remarks>
/// The corridor is constructed by instantiating segment prefabs end-to-end. Each segment's length is measured from its
/// mesh bounds and compared against the configured length. When the animal reaches the end of the corridor, the first
/// segment is recycled to the back, creating the illusion of an infinite track.
/// </remarks>

// Avoid - bullet lists fragment the explanation
/// <remarks>
/// - Instantiates segment prefabs end-to-end
/// - Measures lengths from mesh bounds
/// - Compares against configured lengths
/// - Recycles first segment when end is reached
/// </remarks>
```

### Example tags

Do NOT use `<example>` or `<code>` tags in XML documentation. Examples go stale and create maintenance debt. The
`/cpp-style` and `/python-style` skills state the same prohibition for their languages:

```csharp
// Avoid - <example> tag in documentation
/// <summary>Samples an index from a probability distribution.</summary>
/// <example>
/// <code>
/// int result = SampleFromDistribution(new[] { 0.5f, 0.3f, 0.2f }, random);
/// </code>
/// </example>

// Good - describe behavior in <summary> or <remarks>, no examples
/// <summary>Samples an index from a probability distribution.</summary>
/// <remarks>
/// The probabilities array must sum to 1.0. The method uses inverse transform sampling to select an index weighted by
/// the given distribution.
/// </remarks>
```

---

## File-level documentation

Every `.cs` file must begin with a file-level XML documentation comment describing the file's purpose:

```csharp
/// <summary>
/// Provides the OccupancyZone class that tracks whether an animal has occupied a zone for a required duration.
///
/// Used for trial types that require occupancy-based stimulus disarming. The occupancy mode specifies how a stimulus is
/// triggered, not what stimulus is delivered.
/// </summary>
using System.Diagnostics;
using UnityEngine;
```

### Rules

- Place the file-level `<summary>` before all `using` directives
- First sentence describes the primary class or purpose of the file
- Additional sentences provide context for how the file fits into the larger system. This is the one place that context
  belongs, and it stays subject to the cover test like any other prose
- The whole description is a lean, cohesive chunk of at most 2 sentences. Methodology, caveats, and rationale belong in
  the XML docs of the classes, methods, and enums the file defines, so relocate each detail to the member it concerns
  rather than accumulating it here
- Use third-person imperative mood ("Provides...", "Defines...")

---

## Enum member documentation

Document every enum member with an XML summary. For enums with explicit integer values (status codes, protocol
identifiers), include the value context in the documentation. For enum declaration examples, load `class-patterns.md`
from SKILL.md.

---

## Property documentation

Property summaries should ideally be a single sentence, even if it spans multiple lines. Do not split a property summary
into a one-line `<summary>` plus a `<remarks>` block. Keep it as one continuous sentence, the same shape `/python-style`
requires of a property docstring. For property declaration examples, load `class-patterns.md` from SKILL.md.

For properties with backing fields, document both the field and the property:

```csharp
/// <summary>The serialized reference to the display configuration object.</summary>
[SerializeField]
private DisplayObject _display;

/// <summary>Returns the display configuration object.</summary>
public DisplayObject Display
{
    get { return _display; }
    set { _display = value; }
}
```

---

## Type usage conventions

### Explicit types vs var

Prefer explicit type declarations for clarity. Use `var` only when the type is immediately obvious from the right-hand
side:

```csharp
// Good - explicit types
Dictionary<string, byte> cueIds = new Dictionary<string, byte>();
float[] segmentLengths = template.GetSegmentLengthsUnity();
GameObject task = new GameObject(taskName);

// Acceptable - type obvious from constructor
var meshRenderer = gameObject.GetComponent<MeshRenderer>();
var random = new System.Random(seed);

// Avoid - type not obvious
var result = ProcessData(input);
var value = GetConfiguration();
```

This matches the EditorConfig settings:
- `csharp_style_var_for_built_in_types = false` (always spell out `int`, `float`, `string`)
- `csharp_style_var_when_type_is_apparent = true` (allow `var` when type is obvious)
- `csharp_style_var_elsewhere = false` (spell out type when not obvious)

### Nullable types

Nullable reference type annotations require an enabled nullable annotation context. Turn it on per file with
`#nullable enable`, or project-wide by adding `-nullable:enable` to `Assets/csc.rsp`. Without it the compiler raises
CS8632 on every `T?` annotation. Once the context is on, use nullable reference types (`T?`) when a value may
legitimately be null:

```csharp
/// <summary>The optional occupancy zone component attached to this trigger zone.</summary>
private OccupancyZone? _occupancyZone;
```

### Generic types

Use descriptive type parameter names prefixed with `T`:

```csharp
/// <summary>A typed MQTT channel that deserializes messages to the specified type.</summary>
/// <typeparam name="TMessage">The message type for deserialization.</typeparam>
public class MQTTChannel<TMessage> : MQTTChannel
```

### Null analysis attributes

Use null analysis attributes from `System.Diagnostics.CodeAnalysis` to express nullability contracts that the compiler
cannot infer. This is the C# equivalent of C++ `[[nodiscard]]` and `static_assert` for null safety:

```csharp
using System.Diagnostics.CodeAnalysis;

/// <summary>Attempts to find the zone with the specified name.</summary>
/// <param name="zoneName">The name of the zone to find.</param>
/// <param name="zone">The found zone, or null if not found.</param>
/// <returns>True if the zone was found, false otherwise.</returns>
public bool TryFindZone(string zoneName, [NotNullWhen(true)] out OccupancyZone? zone)
{
    zone = _zones.FirstOrDefault(z => z.name == zoneName);
    return zone != null;
}

/// <summary>Returns the validated template. Throws if validation fails.</summary>
/// <returns>The validated template instance.</returns>
[return: NotNull]
public TaskTemplate GetValidatedTemplate()
{
    if (_template == null)
    {
        throw new InvalidOperationException("Template has not been loaded.");
    }
    return _template;
}
```

Common attributes:

| Attribute                | Meaning                                           |
|--------------------------|---------------------------------------------------|
| `[NotNull]`              | Output is never null (on return or out parameter) |
| `[NotNullWhen(true)]`    | Output is non-null when method returns true       |
| `[NotNullWhen(false)]`   | Output is non-null when method returns false      |
| `[MaybeNullWhen(false)]` | Output may be null when method returns false      |
| `[DoesNotReturn]`        | Method never returns (always throws)              |
| `[MemberNotNull]`        | Specified member is non-null after method returns |

Use these attributes only when they convey information the compiler cannot determine from the method body. Do not add
them speculatively.

### Array vs List

- Use arrays (`T[]`) for fixed-size collections or performance-critical code
- Use `List<T>` for dynamically-sized collections
- Use `Dictionary<TKey, TValue>` for key-value lookups

```csharp
// Fixed size known at creation
float[] segmentLengths = new float[segmentCount];

// Dynamic size
List<GameObject> corridors = new List<GameObject>();

// Key-value mapping
Dictionary<string, byte> cueIdentifiers = new Dictionary<string, byte>();
```

---

## Comments

### Inline comments

- Use third-person imperative mood ("Configures..." not "This section configures...")
- Place above the code, not at end of line (unless very short)
- Use comments to explain non-obvious logic or provide context

```csharp
// Measures actual prefab lengths and compares with configuration.
float measuredSegmentLength = Utility.GetPrefabLength(segmentPrefab);
```

### What to avoid

- Don't reiterate the obvious (e.g., `// Set x to 5` before `x = 5`)
- Don't add XML docs to code you didn't write or modify
- Don't use heavy section separator blocks (e.g., `// ======` or `// ------`)
- Don't use `#region` / `#endregion` blocks (use blank lines between logical groups instead)
- Don't use `this.` qualifier (exception: disambiguating a parameter from a field)
- Don't use IDE-specific suppression comments such as ReSharper/Rider `// ReSharper disable` or `// noinspection`, and
  suppress a genuine analyzer finding with `#pragma warning disable CODE` or `[SuppressMessage]` alone.
