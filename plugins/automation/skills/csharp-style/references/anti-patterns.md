# Anti-patterns

Common C# style violations and how to fix them.

---

## Quick reference

Common transformations at a glance:

| Wrong                              | Correct                            | Rule                                      |
|------------------------------------|------------------------------------|-------------------------------------------|
| `public void Start() {`            | Brace on the following line        | Allman brace style                        |
| `/// This class manages...`        | `/// Manages...`                   | Third-person imperative mood, no preamble |
| `/// Is the zone active.`          | `/// Determines whether...`        | Boolean documentation                     |
| `GetComponent<T>()` in `Update`    | Cache in `Awake`/`Start`           | No allocations in hot paths               |
| `_zones.Where()` in `Update`       | Explicit `for` loop                | No LINQ in hot paths                      |
| Missing `using`/`Dispose()`        | `using var` or `OnDestroy` cleanup | IDisposable resources                     |
| `private int _count;` (assignable) | `private readonly int _count;`     | Readonly when single-assign               |

---

## Naming violations

### Abbreviated identifiers

```csharp
// Wrong - abbreviations
float pos;
int idx;
string msg;
GameObject seg;

// Correct - full words
float position;
int index;
string message;
GameObject segment;
```

### Incorrect casing

```csharp
// Wrong - private field without underscore prefix
private int currentIndex;

// Correct
private int _currentIndex;

// Wrong - constant with kPrefix (C++ style, not C#)
private const float kEpsilon = 0.01f;

// Correct - PascalCase for constants
private const float Epsilon = 0.01f;

// Wrong - public property with camelCase
public bool isReady => _initialized;

// Correct - PascalCase for properties
public bool IsReady => _initialized;

// Wrong - enum values with kPrefix
public enum Status
{
    kActive,
    kInactive,
}

// Correct - PascalCase for enum values
public enum Status
{
    Active,
    Inactive,
}
```

---

## Documentation violations

### Missing documentation

```csharp
// Wrong - no XML documentation
public class TaskManager : MonoBehaviour
{
    public float speed;
    private int _count;
    private void Start() { }
}

// Correct - all members documented
/// <summary>Manages task execution and corridor transitions.</summary>
public class TaskManager : MonoBehaviour
{
    /// <summary>The movement speed in Unity units per second.</summary>
    public float speed;

    /// <summary>The number of completed laps.</summary>
    private int _count;

    /// <summary>Initializes task state and subscribes to MQTT channels.</summary>
    private void Start() { }
}
```

### Wrong documentation mood

```csharp
// Wrong - "This class" / "This method" phrasing
/// <summary>This class manages task state.</summary>

// Correct - third-person imperative mood
/// <summary>Manages task state and corridor transitions.</summary>

// Wrong - not using "Determines whether" for booleans
/// <summary>Is the zone active.</summary>
public bool isActive;

// Correct
/// <summary>Determines whether the zone is active.</summary>
public bool isActive;
```

### Missing file-level documentation

```csharp
// Wrong - file starts with using directives
using UnityEngine;

public class Task : MonoBehaviour { }

// Correct - file-level summary before using directives
/// <summary>
/// Provides the Task class that manages infinite corridor VR task execution.
/// </summary>
using UnityEngine;

public class Task : MonoBehaviour { }
```

---

## Documentation quality violations

| Wrong                                      | Correct                                   | Rule                         |
|--------------------------------------------|-------------------------------------------|------------------------------|
| Sentences over 40 words in prose           | Split into shorter sentences              | Sentence length cap          |
| XML doc length driven by method length     | Match length to conceptual difficulty     | Length proportionality       |
| XML doc explains where it is called        | Describe what the member does             | Behavioral scope             |
| `<param>` restates parameter type          | Describe semantics, not types             | No type-signature restating  |
| XML doc contradicts method behavior        | Update XML doc to match implementation    | Implementation accuracy      |
| Stale issue numbers in comments            | Remove or update with current reference   | No stale references          |
| Typos and grammar errors in comments       | Proofread before submission               | Typo-free and grammatical    |
| Comments narrate what code obviously does  | Remove or explain why                     | No narrate-the-code comments |
| `<remarks>` on a self-evident method       | Single-line `<summary>` alone             | Summary line is the default  |
| `// Now also skips disabled zones`         | State current behavior only               | No change narration          |
| XML doc grown on every edit                | Rewrite in place, delete what is moot     | No documentation ratchet     |

---

## Comment violations

| Wrong                                      | Correct                           | Rule                               |
|--------------------------------------------|-----------------------------------|------------------------------------|
| `// This function sends...`                | `// Sends...`                     | Third-person imperative mood       |
| `// ========================`              | (remove separator)                | No heavy separator blocks          |
| `// ---- Section ----`                     | (remove separator)                | No heavy separator blocks          |
| End-of-line comment on complex logic       | Comment above the code block      | Place above, not at end            |
| `x = 5;  // Set x to 5`                    | `x = 5;`                          | Don't state the obvious            |
| Adding XML docs to code not written by you | Only document your changes        | Don't add docs to others' code     |
| `#region Setup` / `#endregion`             | Blank line between logical groups | No `#region` blocks                |
| `this.fieldName`                           | `fieldName`                       | No `this.` (except disambiguation) |

---

## Formatting violations

### Wrong brace style

```csharp
// Wrong - K&R style (opening brace on same line)
public void Start() {
    if (isReady) {
        Initialize();
    }
}

// Correct - Allman style (opening brace on new line)
public void Start()
{
    if (isReady)
    {
        Initialize();
    }
}
```

### Line length exceeded

```csharp
// Wrong - exceeds 120 characters
Debug.Log($"Warning: For {template.segments[i].name}, there is a mismatch between the prefab length ({measuredSegmentLengths[i]}) and the configured length ({segmentLengths[i]}).");

// Correct - broken across multiple lines
Debug.Log(
    $"Warning: For {template.segments[i].name}, mismatch between prefab length "
        + $"({measuredSegmentLengths[i]}) and configured length ({segmentLengths[i]})."
);
```

### Tabs instead of spaces

The EditorConfig and CSharpier both enforce 4-space indentation. Never use tabs.

---

## Unity-specific violations

### GetComponent in Update

```csharp
// Wrong - GetComponent called every frame
private void Update()
{
    MeshRenderer renderer = GetComponent<MeshRenderer>();
    renderer.enabled = isVisible;
}

// Correct - cache in Awake/Start
private MeshRenderer _meshRenderer;

private void Awake()
{
    _meshRenderer = GetComponent<MeshRenderer>();
}

private void Update()
{
    _meshRenderer.enabled = isVisible;
}
```

### Unsafe component access

```csharp
// Wrong - assumes component exists
private void Start()
{
    OccupancyZone zone = GetComponent<OccupancyZone>();
    zone.ResetState();
}

// Correct - safe access with TryGetComponent
private void Start()
{
    if (TryGetComponent(out OccupancyZone zone))
    {
        zone.ResetState();
    }
}
```

### String concatenation in hot paths

```csharp
// Wrong - string concatenation in Update
private void Update()
{
    Debug.Log("Position: " + transform.position.x + ", " + transform.position.y);
}

// Correct - string interpolation (if logging is needed at all)
private void Update()
{
    Debug.Log($"Position: {transform.position.x}, {transform.position.y}");
}
```

---

## Error handling violations

### Silent failures

```csharp
// Wrong - silently returns without explanation
if (template == null)
    return;

// Correct - log the error before returning
if (template == null)
{
    Debug.LogError("Failed to load task template from YAML file.");
    return;
}
```

### Exception throwing in Unity

```csharp
// Wrong - throwing exceptions in MonoBehaviour methods
private void Start()
{
    if (configPath == null)
        throw new ArgumentNullException(nameof(configPath));
}

// Correct - use Debug.LogError for Unity components
private void Start()
{
    if (string.IsNullOrEmpty(configPath))
    {
        Debug.LogError("No configuration path specified for task.");
        return;
    }
}
```

---

## Structural violations

### Deep nesting

```csharp
// Wrong - deeply nested conditionals
private void Update()
{
    if (isActive)
    {
        if (!boundaryDisarmed)
        {
            if (_occupancyTimer.IsRunning)
            {
                if (inZone)
                {
                    if (_occupancyTimer.ElapsedMilliseconds >= occupancyDurationMs)
                    {
                        OnOccupancyMet();
                    }
                }
            }
        }
    }
}

// Correct - guard clauses reduce nesting
private void Update()
{
    if (!isActive || boundaryDisarmed)
        return;

    if (_occupancyTimer.IsRunning && inZone)
    {
        if (_occupancyTimer.ElapsedMilliseconds >= occupancyDurationMs)
        {
            OnOccupancyMet();
        }
    }
}
```

### Implicit access modifiers

```csharp
// Wrong - relies on implicit private default
int _count;
void Initialize() { }

// Correct - always explicit
private int _count;
private void Initialize() { }
```

---

## Immutability violations

### Missing readonly on single-assignment fields

```csharp
// Wrong - field assigned once but not marked readonly
private SerialPort _port;

public SerialController(string portName, int baudRate)
{
    _port = new SerialPort(portName, baudRate);
}

// Correct - readonly enforces single assignment
private readonly SerialPort _port;

public SerialController(string portName, int baudRate)
{
    _port = new SerialPort(portName, baudRate);
}
```

### Using static readonly for compile-time constants

```csharp
// Wrong - static readonly for a value known at compile time
private static readonly float Epsilon = 0.01f;

// Correct - const for compile-time constants (primitives, strings)
private const float Epsilon = 0.01f;

// Correct - static readonly for runtime-initialized values
private static readonly string DefaultConfigPath = Path.Combine(Application.dataPath, "config");
```

### Mutable struct passed by value

```csharp
// Wrong - mutable struct loses changes when passed by value
public struct ZoneState
{
    public float timer;
    public bool isActive;
}

private void UpdateState(ZoneState state)
{
    state.timer += Time.deltaTime;  // Modifies a copy, not the original
}

// Correct - use readonly struct for value semantics, or use class for mutability
public readonly struct ZoneState
{
    public readonly float Timer;
    public readonly bool IsActive;

    public ZoneState(float timer, bool isActive)
    {
        Timer = timer;
        IsActive = isActive;
    }
}
```

---

## Cross-language consistency violations

These violations drift toward C++ or Python conventions that do not apply in C#:

### Naming drift

```csharp
// Wrong - kPrefix from C++ convention
private const float kEpsilon = 0.01f;
private const int kMaxRetries = 3;

// Correct - PascalCase for C# constants
private const float Epsilon = 0.01f;
private const int MaxRetries = 3;

// Wrong - snake_case namespace from C++ convention
namespace project_config { }

// Correct - PascalCase namespace
namespace Project.Config { }

// Wrong - get_/set_ accessor methods from C++ convention
public float get_Position() { return _position; }
public void set_Position(float value) { _position = value; }

// Correct - C# property
public float Position
{
    get { return _position; }
    set { _position = value; }
}

// Wrong - _snake_case private members from C++/Python convention
private int _current_index;
private float _segment_length;

// Correct - _camelCase private members in C#
private int _currentIndex;
private float _segmentLength;
```

### Documentation drift

```csharp
// Wrong - Doxygen @brief tag from C++ convention
/// @brief Manages the task state.

// Correct - XML <summary> tag
/// <summary>Manages the task state.</summary>

// Wrong - Google-style docstring phrasing from Python convention
/// <summary>A class that manages task state.</summary>

// Correct - third-person imperative mood without articles
/// <summary>Manages task state and corridor transitions.</summary>

// Wrong - trailing /// comment from C++ Doxygen convention
private int _count; ///< The number of completed laps.

// Correct - XML summary above the member
/// <summary>The number of completed laps.</summary>
private int _count;
```

### Structural drift

```csharp
// Wrong - using static (analogous to C++ "using namespace")
using static UnityEngine.Mathf;
float result = Clamp(value, 0f, 1f);

// Correct - qualified call
float result = Mathf.Clamp(value, 0f, 1f);

// Wrong - Python-style enum (UPPER_SNAKE_CASE values)
public enum Status
{
    ZONE_ACTIVE,
    ZONE_INACTIVE,
}

// Correct - PascalCase enum values
public enum Status
{
    ZoneActive,
    ZoneInactive,
}
```
