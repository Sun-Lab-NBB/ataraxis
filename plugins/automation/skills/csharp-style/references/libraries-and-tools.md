# Libraries, tools, and patterns

Conventions for LINQ, resource management, async/await, testing, static analysis, path handling, and error handling in
C# projects.

---

## LINQ conventions

Prefer **method syntax** over query syntax:

```csharp
// Good - method syntax
List<float> validLengths = segmentLengths.Where(length => length > 0f).ToList();
float totalLength = segmentLengths.Sum();
int activeCount = zones.Count(zone => zone.isActive);

// Avoid - query syntax
var validLengths = (from length in segmentLengths where length > 0f select length).ToList();
```

### Deferred execution

LINQ queries are lazily evaluated. Materialize with `.ToList()` or `.ToArray()` when the result is used multiple times
or when the source collection may change:

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

You MUST NOT use LINQ in `Update()`, `FixedUpdate()`, or other per-frame methods. LINQ allocates intermediate objects
that trigger garbage collection. Use explicit loops instead:

```csharp
// Good - explicit loop in Update (no allocations)
private void Update()
{
    for (int i = 0; i < _zones.Length; i++)
    {
        if (_zones[i].isActive)
        {
            _zones[i].CheckOccupancy();
        }
    }
}

// Avoid - LINQ in Update (allocates every frame)
private void Update()
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
- Keep lambda expressions to a single line, extracting named methods for complex logic
- Use `Array.Find()` / `Array.FindIndex()` for simple single-element lookups on arrays

---

## Resource management

### Using declarations

Use `using` declarations for resources that implement `IDisposable`:

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
    private void Start()
    {
        _port = new SerialPort(portName, baudRate);
        _port.Open();
    }

    /// <summary>Closes the serial connection when the component is destroyed.</summary>
    private void OnDestroy()
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

Separate I/O operations from processing logic for testability and reuse. This matches the Python convention in
`/python-style`:

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

In Unity, prefer coroutines (`IEnumerator` + `StartCoroutine`) for frame-based async operations. Use `async/await` with
`UniTask` or standard `Task` for true async I/O:

```csharp
// Unity coroutine for frame-based waiting
private IEnumerator WaitAndReset()
{
    yield return new WaitForSeconds(2f);
    ResetState();
}
```

---

## Coroutine conventions

Unity coroutines (`IEnumerator` + `StartCoroutine`) are the standard approach for frame-based async operations.

### Naming

Coroutine methods do not receive a suffix (unlike `Async` for `async/await` methods). Use a descriptive verb phrase that
conveys the timed or staged nature of the operation:

```csharp
/// <summary>Waits for the specified duration then resets the zone state.</summary>
private IEnumerator WaitAndReset()
{
    yield return new WaitForSeconds(2f);
    ResetState();
}

/// <summary>Fades the stimulus intensity over the specified duration.</summary>
/// <param name="duration">The fade duration in seconds.</param>
private IEnumerator FadeStimulus(float duration)
{
    float elapsed = 0f;
    while (elapsed < duration)
    {
        _intensity = Mathf.Lerp(1f, 0f, elapsed / duration);
        elapsed += Time.deltaTime;
        yield return null;
    }
    _intensity = 0f;
}
```

### When to use coroutines vs alternatives

| Approach                       | Use when                                                  |
|--------------------------------|-----------------------------------------------------------|
| Coroutine                      | Frame-based waiting, timed sequences, animation staging   |
| `async/await`                  | True async I/O (file reads, network requests)             |
| State tracking in `Update`     | Simple state that changes every frame without waiting     |

### Guidelines

- Start coroutines with `StartCoroutine()` in lifecycle methods or event handlers
- Store `Coroutine` references when you need to cancel with `StopCoroutine()`
- Stop all coroutines in `OnDisable()` or `OnDestroy()` to prevent orphaned execution
- Prefer `WaitForSeconds` over manual timer tracking for simple delays
- Use `WaitUntil(() => condition)` for condition-based waiting
- Cache `WaitForSeconds` instances when yielding in loops to avoid per-frame allocations:

```csharp
/// <summary>The cached wait instruction for the polling interval.</summary>
private readonly WaitForSeconds _pollWait = new WaitForSeconds(0.1f);

/// <summary>Polls the sensor at a fixed interval until the threshold is met.</summary>
private IEnumerator PollSensor()
{
    while (_sensorValue < _threshold)
    {
        yield return _pollWait;
    }
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

- Use "Verifies..." as the third-person imperative mood for test summaries (matching Python convention)
- Each test method has an XML `<summary>` tag
- Do NOT add `<param>`, `<returns>`, or `<exception>` tags to test methods. The `<summary>` tag is sufficient. This
  matches the Python convention of omitting Args, Returns, and Raises sections from test function docstrings
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

### Key analyzers

Roslyn analyzers configured through EditorConfig carry the automated enforcement in C# projects. The `/cpp-style` skill
owns the matching clang-tidy configuration for C++ projects:

| Roslyn analyzer | Description                  |
|-----------------|------------------------------|
| `CA1822`        | Member can be made static    |
| `IDE0044`       | Make field readonly          |
| `CA1806`        | Do not ignore method results |
| `CS0114`        | Missing override keyword     |
| `CA1834`        | Use StringBuilder            |

### Suppressing warnings

When a warning must be suppressed, use `#pragma warning disable` with a justification comment:

```csharp
// Justification: Unity serialization requires public fields for Inspector access.
#pragma warning disable CA1051
public float trackLength = 10f;
#pragma warning restore CA1051
```

---

## Conditional compilation

### Platform and context directives

Use `#if` directives for code that should only compile in specific contexts:

```csharp
// Good - editor-only code guarded by UNITY_EDITOR
#if UNITY_EDITOR
/// <summary>Validates the task template in the editor before entering play mode.</summary>
[UnityEditor.InitializeOnLoadMethod]
private static void ValidateOnLoad()
{
    // Editor-only validation logic
}
#endif
```

### Guidelines

- Use `#if UNITY_EDITOR` for editor tools, custom inspectors, and menu items
- Use `#if UNITY_STANDALONE` or platform-specific symbols for platform-dependent code
- Prefer `[System.Diagnostics.Conditional("DEBUG")]` over `#if DEBUG` for method-level conditional compilation (cleaner
  and avoids call-site `#if` blocks):

```csharp
// Good - Conditional attribute (callers do not need #if guards)
[System.Diagnostics.Conditional("DEBUG")]
private static void LogDebugInfo(string message)
{
    Debug.Log($"[DEBUG] {message}");
}

// Avoid - #if DEBUG at every call site
#if DEBUG
Debug.Log($"[DEBUG] {message}");
#endif
```

- Do NOT indent code inside `#if`/`#endif` blocks relative to the directive. The `#` must start at column 0, and the
  enclosed code follows normal indentation:

```csharp
// Good - directive at column 0, code at normal indentation
public class Task : MonoBehaviour
{
#if UNITY_EDITOR
    /// <summary>The editor-only validation state.</summary>
    private bool _editorValidated;
#endif

    /// <summary>The runtime task state.</summary>
    private bool _isRunning;
}
```

- Keep `#if` blocks as small as possible. Prefer extracting platform-specific logic into separate methods over wrapping
  large blocks of code.

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

---

## String comparison

Use ordinal comparison for internal string matching. Culture-sensitive comparison introduces locale-dependent behavior
that can cause subtle bugs across machines:

```csharp
// Good - ordinal comparison for internal logic
if (string.Equals(segmentName, "Segment_A", StringComparison.Ordinal))
{
    // Exact match
}

// Good - case-insensitive ordinal for user-facing input
if (string.Equals(input, "yes", StringComparison.OrdinalIgnoreCase))
{
    // Case-insensitive match without locale issues
}

// Good - ordinal for Contains, StartsWith, EndsWith
if (topicName.StartsWith("Gimbl/", StringComparison.Ordinal))
{
    // Prefix check
}

// Avoid - default comparison uses CurrentCulture (locale-dependent)
if (segmentName == "Segment_A")  // Acceptable for simple equality in non-critical paths
if (topicName.Contains("Gimbl"))  // Uses CurrentCulture on some .NET versions
```

### Guidelines

- Use `StringComparison.Ordinal` for protocol strings, topic names, and identifiers
- Use `StringComparison.OrdinalIgnoreCase` for case-insensitive matching
- Simple `==` equality is acceptable for straightforward constant comparisons where locale behavior is irrelevant
- Never use `StringComparison.CurrentCulture` or `StringComparison.InvariantCulture` unless displaying sorted text to
  users

---

## Function calls

Prefer **named arguments** when the meaning is not obvious from the value alone:

```csharp
// Good - named arguments clarify boolean and same-type parameters
CreateChannel(topic: "sensors/encoder", isListener: true, qosLevel: 2);
Instantiate(prefab: segmentPrefab, position: spawnPosition, rotation: Quaternion.identity);

// Acceptable - single argument or meaning obvious from type/name
Mathf.Abs(difference);
Debug.Log(message);
GetComponent<MeshRenderer>();
```

Use named arguments when a method has:
- Boolean parameters (always name them)
- Multiple parameters of the same type
- Parameters whose meaning is unclear from the value

---

## Blank lines

- **One blank line** between method definitions within a class
- **One blank line** after using directive blocks before namespace/class
- **No blank line** after an opening brace or before a closing brace
- **One blank line** between logical groups of statements within a method

---

## Line length and formatting

- Maximum line length: **120 characters**
- Formatter: **CSharpier** (config in `.csharpierrc.yaml`)
- Style enforcement: **EditorConfig** (config in `.editorconfig`)
- Brace style: **Allman** (opening braces on new lines for all constructs)
- Indentation: **4 spaces** (no tabs)
- Line endings: **LF** (Unix-style)

### String formatting

- Use **string interpolation** (`$"..."`) for all string formatting
- Use verbatim strings (`@"..."`) for paths and multi-line strings
- Use **double quotes** for all strings

### Trailing commas

C# does not enforce trailing commas in the same way as Python. Follow CSharpier's output for comma placement in
multi-line constructs.

### Brace rules

Always use braces for control flow statements, with the single-line guard clause as the one exception:

```csharp
// Good
if (template == null)
{
    return;
}

// Acceptable for simple guard clauses
if (!isActive)
    return;
```

---

## Tooling

### CSharpier

CSharpier is the primary formatter. Install and use it before committing:

```bash
dotnet tool install -g csharpier    # Install globally
csharpier .                          # Format all files
csharpier --check .                  # Check without modifying (CI mode)
```

Configuration lives in `.csharpierrc.yaml`:

```yaml
printWidth: 120
useTabs: false
tabWidth: 4
endOfLine: lf
```

### EditorConfig

The `.editorconfig` file enforces naming conventions, brace style, and spacing rules in IDEs. It is the source of truth
for style rules that CSharpier does not cover (naming, `var` preferences, expression-bodied members).

### CSharpier ignore

The `.csharpierignore` file excludes Unity-generated directories (`Library/`, `Temp/`, `Logs/`) and third-party packages
from formatting.

---

## Configuration files

Canonical configs are stored in [../assets/](../assets/). When working in a C# project, verify that `.csharpierrc.yaml`,
`.editorconfig`, and `.csharpierignore` in the project root match these:

- [../assets/.csharpierrc.yaml](../assets/.csharpierrc.yaml)
- [../assets/.editorconfig](../assets/.editorconfig)
- [../assets/.csharpierignore](../assets/.csharpierignore)

The `.csharpierignore` contains generic entries only. Individual projects may need additional project-specific entries
(e.g., paths to auto-generated scripts).

---

## Guard clauses and boolean expressions

Prefer early returns (guard clauses) over deeply nested conditionals. Use explicit boolean checks,
`string.IsNullOrEmpty()` for strings, and `== null` / `!= null` for null checks:

```csharp
/// <summary>Checks if the occupancy duration has been met while the animal is in the zone.</summary>
private void Update()
{
    if (!isActive || boundaryDisarmed)
        return;

    if (string.IsNullOrEmpty(_zoneName))
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

---

## Error handling

MonoBehaviour code reports failures through Unity's logging system and returns without throwing:

```csharp
if (template == null)
{
    Debug.LogError("Failed to load task template from YAML file.");
    return;
}

if (Mathf.Abs(measuredLength - configuredLength) > LengthComparisonEpsilon)
{
    Debug.LogWarning(
        $"For {segmentName}, mismatch between prefab length ({measuredLength}) "
            + $"and configured length ({configuredLength})."
    );
}
```

Editor tools, static utility classes, and pure C# libraries that do not inherit from MonoBehaviour raise standard C#
exceptions instead:

```csharp
/// <summary>Parses the task template from raw YAML content.</summary>
/// <param name="yamlContent">The YAML string to parse.</param>
/// <returns>The deserialized task template.</returns>
/// <exception cref="FormatException">The YAML content does not conform to the template schema.</exception>
public static TaskTemplate ParseTemplate(string yamlContent)
{
    if (string.IsNullOrEmpty(yamlContent))
    {
        throw new ArgumentException("YAML content must not be null or empty.", nameof(yamlContent));
    }

    TaskTemplate template = YamlParser.Deserialize<TaskTemplate>(yamlContent);
    if (template == null)
    {
        throw new FormatException("Failed to deserialize task template from YAML content.");
    }
    return template;
}
```

### Error message format

Assign a multi-line message to a local variable before passing it, and pass a short single-line message directly:

```csharp
// Good - message variable for multi-line errors
string message =
    $"Unable to create corridor for segment '{segmentName}'. "
    + $"The configured length ({configuredLength}) must match the prefab length "
    + $"({measuredLength}), but they differ by {Mathf.Abs(measuredLength - configuredLength)}.";
Debug.LogError(message);

// Acceptable - short single-line messages passed directly
Debug.LogError("Failed to load task template from YAML file.");
```
