# Anti-patterns

Common C# style violations in Sun Lab projects and how to fix them.

---

## Quick reference

Common transformations at a glance:

| Wrong                              | Correct                            | Rule                         |
|------------------------------------|------------------------------------|------------------------------|
| `float pos;`                       | `float position;`                  | Full words, no abbreviations |
| `private int currentIndex;`        | `private int _currentIndex;`       | Private fields use `_prefix` |
| `private const float kEpsilon`     | `private const float Epsilon`      | PascalCase constants         |
| `public bool isReady => ...`       | `public bool IsReady => ...`       | PascalCase properties        |
| `public void Start() {`            | `public void Start()\n{`           | Allman brace style           |
| `/// This class manages...`        | `/// Manages...`                   | Imperative mood, no preamble |
| `/// Is the zone active.`          | `/// Determines whether...`        | Boolean documentation        |
| `GetComponent<T>()` in `Update`    | Cache in `Awake`/`Start`           | No allocations in hot paths  |
| `_zones.Where()` in `Update`       | Explicit `for` loop                | No LINQ in hot paths         |
| Missing `using`/`Dispose()`        | `using var` or `OnDestroy` cleanup | IDisposable resources        |
| `private int _count;` (assignable) | `private readonly int _count;`     | Readonly when single-assign  |

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
    void Start() { }
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
    void Start() { }
}
```

### Wrong documentation mood

```csharp
// Wrong - "This class" / "This method" phrasing
/// <summary>This class manages task state.</summary>

// Correct - imperative mood
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

## Unity-specific anti-patterns

### GetComponent in Update

```csharp
// Wrong - GetComponent called every frame
void Update()
{
    MeshRenderer renderer = GetComponent<MeshRenderer>();
    renderer.enabled = isVisible;
}

// Correct - cache in Awake/Start
private MeshRenderer _meshRenderer;

void Awake()
{
    _meshRenderer = GetComponent<MeshRenderer>();
}

void Update()
{
    _meshRenderer.enabled = isVisible;
}
```

### Unsafe component access

```csharp
// Wrong - assumes component exists
void Start()
{
    OccupancyZone zone = GetComponent<OccupancyZone>();
    zone.ResetState();
}

// Correct - safe access with TryGetComponent
void Start()
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
void Update()
{
    Debug.Log("Position: " + transform.position.x + ", " + transform.position.y);
}

// Correct - string interpolation (if logging is needed at all)
void Update()
{
    Debug.Log($"Position: {transform.position.x}, {transform.position.y}");
}
```

---

## Error handling anti-patterns

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
void Start()
{
    if (configPath == null)
        throw new ArgumentNullException(nameof(configPath));
}

// Correct - use Debug.LogError for Unity components
void Start()
{
    if (string.IsNullOrEmpty(configPath))
    {
        Debug.LogError("No configuration path specified for task.");
        return;
    }
}
```

---

## Structural anti-patterns

### Deep nesting

```csharp
// Wrong - deeply nested conditionals
void Update()
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
void Update()
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

## LINQ anti-patterns

### LINQ in hot paths

```csharp
// Wrong - LINQ allocates in Update (garbage collection pressure)
void Update()
{
    foreach (OccupancyZone zone in _zones.Where(z => z.isActive))
    {
        zone.CheckOccupancy();
    }
}

// Correct - explicit loop with no allocations
void Update()
{
    for (int i = 0; i < _zones.Length; i++)
    {
        if (_zones[i].isActive)
        {
            _zones[i].CheckOccupancy();
        }
    }
}
```

### Multiple enumeration

```csharp
// Wrong - evaluates the query twice
IEnumerable<GameObject> activeZones = allZones.Where(zone => zone.isActive);
Debug.Log($"Active zones: {activeZones.Count()}");  // First evaluation
ProcessZones(activeZones);                           // Second evaluation

// Correct - materialize with ToList when reused
List<GameObject> activeZones = allZones.Where(zone => zone.isActive).ToList();
Debug.Log($"Active zones: {activeZones.Count}");
ProcessZones(activeZones);
```

### Query syntax instead of method syntax

```csharp
// Wrong - query syntax
var validLengths = (from length in segmentLengths where length > 0f select length).ToList();

// Correct - method syntax
List<float> validLengths = segmentLengths.Where(length => length > 0f).ToList();
```

---

## Resource management anti-patterns

### Missing IDisposable cleanup

```csharp
// Wrong - disposable resource never cleaned up
public class SerialController : MonoBehaviour
{
    private SerialPort _port;

    void Start()
    {
        _port = new SerialPort(portName, baudRate);
        _port.Open();
    }
    // _port is never disposed!
}

// Correct - clean up in OnDestroy
public class SerialController : MonoBehaviour
{
    private SerialPort _port;

    void Start()
    {
        _port = new SerialPort(portName, baudRate);
        _port.Open();
    }

    void OnDestroy()
    {
        if (_port != null && _port.IsOpen)
        {
            _port.Close();
            _port.Dispose();
        }
    }
}
```

### Missing using declaration

```csharp
// Wrong - StreamReader never disposed
string content;
StreamReader reader = new StreamReader(configPath);
content = reader.ReadToEnd();

// Correct - using declaration ensures disposal
using var reader = new StreamReader(configPath);
string content = reader.ReadToEnd();
```

---

## Immutability anti-patterns

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

void UpdateState(ZoneState state)
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
