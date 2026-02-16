# XML documentation and type usage

Detailed conventions for C# XML documentation and type usage in Sun Lab projects.

---

## XML documentation

Use XML documentation comments (`///`) for all public and private members. This matches the
Doxygen documentation style used in Sun Lab C++ projects and the Google-style docstrings used in
Python projects.

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

- **Imperative mood**: Use verbs like "Provides...", "Defines...", "Configures...", "Tracks..."
  for ALL members
- **Boolean descriptions**: Use "Determines whether..." for boolean fields and properties
- **Single-line format**: Use single-line `<summary>` for most members
- **Multi-line format**: Use multi-line `<summary>` only when the description exceeds 120
  characters

```csharp
// Single-line (preferred for most members)
/// <summary>Initializes the occupancy timer.</summary>
void Start()

// Multi-line (when description is long)
/// <summary>
/// The duration in milliseconds that the animal must occupy the zone to disarm the boundary.
/// Set at task creation time from the task template.
/// </summary>
public float occupancyDurationMs = 1000f;
```

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

For simple getters, the `<returns>` tag may be omitted if the `<summary>` already describes the
return value.

### Remarks tags

Use `<remarks>` for extended descriptions, implementation notes, or important context:

```csharp
/// <summary>
/// Provides the Task class that manages an infinite corridor VR task.
/// </summary>
/// <remarks>
/// The task creates a looping corridor by instantiating corridor segments and managing
/// probabilistic transitions between them. Each corridor combination is a child GameObject
/// containing the segment prefabs positioned end-to-end.
/// </remarks>
public class Task : MonoBehaviour
```

### Tag ordering

XML documentation tags must appear in this order on every member. This matches the canonical
ordering used by Doxygen in C++ projects and Google-style docstrings in Python projects:

1. `<summary>` — always first
2. `<remarks>` — extended description, notes, warnings
3. `<typeparam>` — type parameters, in declaration order
4. `<param>` — method parameters, in declaration order
5. `<returns>` — return value description
6. `<exception>` — documented exceptions, in alphabetical order by type

This matches Doxygen ordering in C++ (`@brief` → `@details` → `@tparam` → `@param` → `@return`
→ `@throws`) and Google-style Python docstrings (summary → extended description → Args → Returns
→ Raises). Omit tags that do not apply. Never reorder tags within a documentation block.

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

Use `<exception>` tags to document exceptions that a method may throw. This is the C# equivalent
of the `@throws` tag in Doxygen and the `Raises:` section in Google-style Python docstrings:

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

### Inheritdoc

Use `<inheritdoc/>` when overriding a method or implementing an interface where the base
documentation is sufficient:

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

Use `<remarks>` with bold text for warnings and notes (C# XML docs do not have native
`@warning` or `@note` tags like Doxygen):

```csharp
/// <summary>Resets the encoder pulse counter to zero.</summary>
/// <remarks>
/// <b>Warning:</b> This method should only be called when the encoder is stationary.
/// Calling it during active rotation may cause pulse count discontinuities.
/// </remarks>
public void ResetCounter()
```

### Prose over lists in remarks

Use flowing prose in `<remarks>` blocks rather than bullet lists. This matches the Python
convention of using narrative paragraphs in the extended description section of Google-style
docstrings:

```csharp
// Good - prose explains the relationship between concepts
/// <remarks>
/// The corridor is constructed by instantiating segment prefabs end-to-end. Each segment's
/// length is measured from its mesh bounds and compared against the configured length. When
/// the animal reaches the end of the corridor, the first segment is recycled to the back,
/// creating the illusion of an infinite track.
/// </remarks>

// Avoid - bullet lists fragment the explanation
/// <remarks>
/// - Instantiates segment prefabs end-to-end
/// - Measures lengths from mesh bounds
/// - Compares against configured lengths
/// - Recycles first segment when end is reached
/// </remarks>
```

---

## File-level documentation

Every `.cs` file must begin with a file-level XML documentation comment describing the file's
purpose:

```csharp
/// <summary>
/// Provides the OccupancyZone class that tracks whether an animal has occupied a zone for a
/// required duration.
///
/// Used for trial types that require occupancy-based stimulus disarming. The occupancy mode
/// specifies how a stimulus is triggered, not what stimulus is delivered.
/// </summary>
using System.Diagnostics;
using UnityEngine;
```

### Rules

- Place the file-level `<summary>` before all `using` directives
- First sentence describes the primary class or purpose of the file
- Additional sentences provide context for how the file fits into the larger system
- Use third-person imperative mood ("Provides...", "Defines...")

---

## Enum member documentation

Document every enum member with an XML summary:

```csharp
/// <summary>Defines the supported controller types for VR input devices.</summary>
public enum ControllerTypes
{
    /// <summary>A physical linear treadmill connected via serial communication.</summary>
    LinearTreadmill,

    /// <summary>A simulated treadmill for testing without physical hardware.</summary>
    SimulatedLinearTreadmill,
}
```

For enums with explicit integer values (status codes, protocol identifiers), include the value
context in the documentation:

```csharp
/// <summary>Defines the status codes for zone state transitions.</summary>
public enum ZoneStatus
{
    /// <summary>The zone is inactive and not monitoring occupancy.</summary>
    Inactive = 0,

    /// <summary>The zone is active and monitoring for animal entry.</summary>
    Active = 1,

    /// <summary>The zone boundary has been disarmed by meeting the occupancy requirement.</summary>
    Disarmed = 2,
}
```

---

## Property documentation

Properties follow the same XML documentation conventions as fields:

```csharp
/// <summary>Determines whether this zone uses occupancy-based stimulus triggering.</summary>
public bool IsOccupancyMode => _occupancyZone != null;

/// <summary>Returns the corridor spacing in Unity units, converted from centimeters.</summary>
public float CorridorSpacingUnity => corridorSpacingCm / CmPerUnityUnit;
```

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

Prefer explicit type declarations for clarity. Use `var` only when the type is immediately
obvious from the right-hand side:

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

Use nullable reference types (`T?`) when a value may legitimately be null:

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
