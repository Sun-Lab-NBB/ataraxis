# Libraries, tools, and patterns

Conventions for LINQ, resource management, async/await, testing, static analysis, and path
handling in Sun Lab C# projects.

---

## LINQ conventions

LINQ is the C# equivalent of Python comprehensions. Prefer **method syntax** over query syntax:

```csharp
// Good - method syntax
List<float> validLengths = segmentLengths.Where(length => length > 0f).ToList();
float totalLength = segmentLengths.Sum();
int activeCount = zones.Count(zone => zone.isActive);

// Avoid - query syntax
var validLengths = (from length in segmentLengths where length > 0f select length).ToList();
```

### Deferred execution

LINQ queries are lazily evaluated. Materialize with `.ToList()` or `.ToArray()` when the
result is used multiple times or when the source collection may change:

```csharp
// Good - materialize when reused
List<GameObject> activeZones = allZones.Where(zone => zone.isActive).ToList();
Debug.Log($"Active zones: {activeZones.Count}");
ProcessZones(activeZones);

// Avoid - evaluating the query twice
IEnumerable<GameObject> activeZones = allZones.Where(zone => zone.isActive);
Debug.Log($"Active zones: {activeZones.Count()}");  // First evaluation
ProcessZones(activeZones);                           // Second evaluation
```

### LINQ in hot paths

You MUST NOT use LINQ in `Update()`, `FixedUpdate()`, or other per-frame methods. LINQ
allocates intermediate objects that trigger garbage collection. Use explicit loops instead:

```csharp
// Good - explicit loop in Update (no allocations)
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

// Wrong - LINQ in Update (allocates every frame)
void Update()
{
    foreach (OccupancyZone zone in _zones.Where(z => z.isActive))
    {
        zone.CheckOccupancy();
    }
}
```

### Guidelines

- Use LINQ for collection initialization, one-time queries, and editor scripts
- Use explicit loops for per-frame logic and performance-critical paths
- Prefer `.Select()` over manual list building in non-hot paths
- Keep lambda expressions short (one line); extract named methods for complex logic
- Use `Array.Find()` / `Array.FindIndex()` for simple single-element lookups on arrays

---

## Resource management

### Using declarations

Use `using` declarations for resources that implement `IDisposable`. This is the C# equivalent
of Python's context managers (`with` statements):

```csharp
// Good - using declaration (C# 8+, preferred)
using var reader = new StreamReader(configPath);
string content = reader.ReadToEnd();

// Good - using block (when scope must be explicit)
using (var writer = new StreamWriter(outputPath))
{
    writer.Write(serializedData);
}
```

### IDisposable in Unity

MonoBehaviour classes that hold disposable resources must clean them up in `OnDestroy()`:

```csharp
/// <summary>Manages a serial connection to the microcontroller.</summary>
public class SerialController : MonoBehaviour
{
    /// <summary>The serial port connection.</summary>
    private SerialPort _port;

    /// <summary>Opens the serial connection.</summary>
    void Start()
    {
        _port = new SerialPort(portName, baudRate);
        _port.Open();
    }

    /// <summary>Closes the serial connection when the component is destroyed.</summary>
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

### Guidelines

- Prefer `using` declarations over `using` blocks (less nesting)
- Always dispose `Stream`, `StreamReader`, `StreamWriter`, `SerialPort`, `HttpClient`
- In Unity, clean up resources in `OnDestroy()` (not finalizers)
- Never implement `~Finalizer()` unless wrapping unmanaged resources directly

---

## I/O separation

Separate I/O operations from processing logic for testability and reuse. This matches the
Python convention in `/python-style`:

```csharp
// Good - I/O separated from logic
public static TaskTemplate LoadTemplate(string configPath)
{
    string yamlContent = File.ReadAllText(configPath);
    return ParseTemplate(yamlContent);
}

private static TaskTemplate ParseTemplate(string yamlContent)
{
    // Pure processing - no I/O, easy to test
    return YamlParser.Deserialize<TaskTemplate>(yamlContent);
}

// Avoid - I/O mixed with logic
public static TaskTemplate LoadAndParseTemplate(string configPath)
{
    string yamlContent = File.ReadAllText(configPath);  // I/O
    return YamlParser.Deserialize<TaskTemplate>(yamlContent);  // Logic
}
```

---

## Async/await conventions

### Naming

Async methods must have the `Async` suffix:

```csharp
/// <summary>Loads the task configuration asynchronously.</summary>
public async Task<TaskTemplate> LoadConfigAsync(string configPath)
{
    string content = await File.ReadAllTextAsync(configPath);
    return ParseTemplate(content);
}
```

### Rules

- **`async Task`** for methods that return a value or complete asynchronously
- **`async Task<T>`** for methods that return a value asynchronously
- **Never use `async void`** except for Unity event handlers and UI callbacks
- Pass `CancellationToken` as the last parameter for cancellable operations
- Use `ConfigureAwait(false)` in library code (not in Unity MonoBehaviours)

### Unity-specific

In Unity, prefer coroutines (`IEnumerator` + `StartCoroutine`) for frame-based async
operations. Use `async/await` with `UniTask` or standard `Task` for true async I/O:

```csharp
// Unity coroutine for frame-based waiting
IEnumerator WaitAndReset()
{
    yield return new WaitForSeconds(2f);
    ResetState();
}
```

---

## Testing conventions

### Test class naming

Test classes use the `Tests` suffix and mirror the source class name:

```csharp
/// <summary>Verifies the behavior of the ConfigLoader class.</summary>
[TestFixture]
public class ConfigLoaderTests
{
    // Tests for ConfigLoader
}
```

### Test method naming

Use the pattern `MethodName_Scenario_ExpectedResult`:

```csharp
/// <summary>Verifies that LoadTemplate returns null for an invalid YAML path.</summary>
[Test]
public void LoadTemplate_InvalidPath_ReturnsNull()
{
    TaskTemplate result = ConfigLoader.LoadTemplate("nonexistent.yaml");
    Assert.IsNull(result);
}

/// <summary>Verifies that GetSegmentLengths returns correct lengths for valid prefabs.</summary>
[Test]
public void GetSegmentLengths_ValidPrefabs_ReturnsCorrectLengths()
{
    // Arrange, Act, Assert
}
```

### Test documentation

- Use "Verifies..." as the imperative mood for test summaries (matching Python convention)
- Each test method has an XML `<summary>` tag
- Use the Arrange-Act-Assert pattern for test body organization

---

## Static analysis

### EditorConfig diagnostic severity

Configure Roslyn analyzer severity in `.editorconfig` for automated enforcement:

```ini
# Make specific analyzers warnings or errors
dotnet_diagnostic.CA1822.severity = suggestion   # Member can be made static
dotnet_diagnostic.IDE0044.severity = suggestion  # Make field readonly
dotnet_diagnostic.CA1051.severity = suggestion   # Do not declare visible instance fields
```

### Key analyzer equivalents

The C++ projects use clang-tidy with `WarningsAsErrors: '*'`. The C# equivalents are Roslyn
analyzers configured via EditorConfig:

| clang-tidy Check                                 | Roslyn Equivalent | Description                  |
|--------------------------------------------------|-------------------|------------------------------|
| `readability-convert-member-functions-to-static` | `CA1822`          | Member can be made static    |
| `readability-make-member-function-const`         | `IDE0044`         | Make field readonly          |
| `bugprone-unused-return-value`                   | `CA1806`          | Do not ignore method results |
| `modernize-use-override`                         | `CS0114`          | Missing override keyword     |
| `performance-inefficient-string-concatenation`   | `CA1834`          | Use StringBuilder            |

### Suppressing warnings

When a warning must be suppressed, use `#pragma warning disable` with a justification comment:

```csharp
// Justification: Unity serialization requires public fields for Inspector access.
#pragma warning disable CA1051
public float trackLength = 10f;
#pragma warning restore CA1051
```

---

## Path handling

Use `System.IO.Path` methods for all path manipulation. Never use string concatenation:

```csharp
// Good - Path.Combine for cross-platform paths
string configPath = Path.Combine(Application.dataPath, "Configurations", "task.yaml");
string directory = Path.GetDirectoryName(configPath);
string extension = Path.GetExtension(configPath);

// Avoid - string concatenation
string configPath = Application.dataPath + "/Configurations/" + "task.yaml";
```

### Guidelines

- Use `Path.Combine()` to join path segments (handles separators correctly)
- Use `Path.GetFullPath()` for path normalization
- Use `@"..."` verbatim strings for literal Windows paths in constants
- Use `Application.dataPath`, `Application.persistentDataPath` for Unity paths
