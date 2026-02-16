---
name: csharp-style
description: >-
  Applies Sun Lab C# coding conventions when writing, reviewing, or refactoring code. Covers .cs
  files, XML documentation, naming, formatting, error handling, using directives, file ordering,
  and Unity-specific patterns. Use when writing new C# code, modifying existing code, reviewing
  pull requests, or when the user asks about C# coding standards.
user-invocable: true
---

# C# code style guide

Applies Sun Lab C# coding conventions.

You MUST read this skill and load the relevant reference files before writing or modifying C#
code. You MUST verify your changes against the checklist before submitting.

---

## Scope

**Covers:**
- C# code style (XML documentation, naming, formatting, error handling)
- Using directive conventions and file organization
- Class design, enums, properties, and inheritance patterns
- Unity-specific patterns (MonoBehaviour, ScriptableObject, serialization)
- CSharpier and EditorConfig tooling conventions
- Cross-language consistency with C++ and Python Sun Lab conventions

**Does not cover:**
- README file conventions (invoke `/readme-style`)
- Commit message conventions (invoke `/commit`)
- Skill file and CLAUDE.md conventions (invoke `/skill-design`)
- Codebase exploration workflows (invoke `/explore-codebase`)

---

## Workflow

You MUST follow these steps when this skill is invoked.

### Step 1: Read this skill

Read this entire file. The core conventions below apply to ALL C# code.

### Step 2: Load relevant reference files

Based on the task, load the appropriate reference files:

| Task                                        | Reference to load                                             |
|---------------------------------------------|---------------------------------------------------------------|
| Writing or modifying XML docs / type usage  | [xml-docs-and-types.md](references/xml-docs-and-types.md)     |
| Writing classes, enums, or Unity components | [class-patterns.md](references/class-patterns.md)             |
| Using LINQ, async, IDisposable, or testing  | [libraries-and-tools.md](references/libraries-and-tools.md)   |
| Deploying or verifying tool config files    | [assets/](assets/) directory                                  |
| Reviewing code before submission            | [anti-patterns.md](references/anti-patterns.md)               |

Load multiple references when the task spans multiple domains.

### Step 3: Apply conventions

Write or modify C# code following all conventions from this file and the loaded references.

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
- Private members use underscore prefix (`_camelCase`)
- Full words in identifiers (no abbreviations)
- Guard clauses preferred over deep nesting

**C#-specific divergences from C++:**
- Constants use PascalCase (not `kPrefix` as in C++)
- Enum values use PascalCase (not `kPrefix` as in C++)
- Namespaces use PascalCase (not snake_case as in C++)
- Brace style is Allman (opening braces on new lines, matching the C++ clang-format config)
- Consecutive assignment alignment is NOT used (CSharpier does not support it)
- `#region` blocks are NOT used (prefer blank lines between logical groups)

---

## Naming conventions

### Variables

Use **full words**, not abbreviations:

| Avoid              | Prefer                              |
|--------------------|-------------------------------------|
| `pos`, `idx`       | `position`, `index`                 |
| `msg`, `val`       | `message`, `value`                  |
| `seg`, `trig`      | `segment`, `trigger`                |
| `cfg`, `cnt`       | `configuration`, `count`            |

### Identifiers

| Element           | Convention    | Example                                   |
|-------------------|---------------|-------------------------------------------|
| Classes           | PascalCase    | `StimulusTriggerZone`, `TaskTemplate`     |
| Methods           | PascalCase    | `ResetState`, `GetSegmentLengths`         |
| Public fields     | camelCase     | `trackLength`, `requireLick`, `isActive`  |
| Public properties | PascalCase    | `IsOccupancyMode`, `CorridorSpacingUnity` |
| Private fields    | `_camelCase`  | `_currentSegmentIndex`, `_occupancyTimer` |
| Local variables   | camelCase     | `segmentPath`, `measuredLength`           |
| Parameters        | camelCase     | `configPath`, `qosLevel`                  |
| Constants         | PascalCase    | `LengthComparisonEpsilon`                 |
| Enum types        | PascalCase    | `ControllerTypes`, `StimulusMode`         |
| Enum values       | PascalCase    | `LinearTreadmill`, `OccupancyBased`       |
| Namespaces        | PascalCase    | `Gimbl`, `SL.Config`                      |
| Interfaces        | `IPascalCase` | `IConfigurable`, `IResettable`            |
| Type parameters   | `TPascalCase` | `TMessage`, `TConfig`                     |

### Public fields vs properties

In Unity projects, MonoBehaviour and ScriptableObject classes expose **public fields** using
camelCase for Inspector serialization. These are effectively configuration parameters set via the
Unity Editor:

```csharp
/// <summary>Determines whether the task requires a lick to start a trial.</summary>
public bool requireLick = false;

/// <summary>The length of the track in Unity units.</summary>
public float trackLength = 10f;
```

Use PascalCase **properties** for computed values or encapsulated access:

```csharp
/// <summary>Determines whether this zone uses occupancy-based stimulus triggering.</summary>
public bool IsOccupancyMode => _occupancyZone != null;
```

### Functions

- Use descriptive verb phrases: `CreateNewTask`, `ValidateTemplate`, `ResetState`
- Private methods follow PascalCase (no underscore prefix, unlike Python)
- Avoid generic names like `Process`, `Handle`, `DoSomething`

### Constants and immutability

Use PascalCase for `const` and `static readonly` fields. Prefer immutability by default:

- **`const`**: Compile-time constants (primitives, strings). Inlined at call sites.
- **`static readonly`**: Runtime-initialized immutable values (objects, computed values).
- **`readonly`**: Instance fields assigned only in constructors or field initializers.

```csharp
/// <summary>The tolerance for comparing measured prefab lengths against configured lengths.</summary>
private const float LengthComparisonEpsilon = 0.01f;

/// <summary>The default configuration path resolved at runtime.</summary>
private static readonly string DefaultConfigPath = Path.Combine(Application.dataPath, "config");

/// <summary>The communication port assigned at construction time.</summary>
private readonly SerialPort _port;
```

You MUST mark fields as `readonly` when they are only assigned in the constructor or initializer.
For detailed immutability patterns (`readonly struct`, records, `in` parameters), see
[class-patterns.md](references/class-patterns.md).

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

## Error handling

### Unity projects

Use Unity's logging system for error reporting:

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

### Error message format

Use a structured format: context ("Unable to..."), constraint ("must be..."), actual value
("but [actual state]."). Use `Debug.LogError()` for failures that prevent continuation,
`Debug.LogWarning()` for non-critical issues, and `Debug.Log()` for informational messages.

### Null handling

- Use explicit null checks: `if (template == null)`
- Use `string.IsNullOrEmpty()` for string validation
- Use `TryGetComponent<T>()` for safe component access in Unity
- Prefer null-conditional operator (`?.`) for optional chains
- Prefer null-coalescing operator (`??`) for default values

---

## Comments

### Inline comments

- Use third person imperative ("Configures..." not "This section configures...")
- Place above the code, not at end of line (unless very short)
- Use comments to explain non-obvious logic or provide context

```csharp
// Measures actual prefab lengths and compares with configuration.
float[] measuredSegmentLengths = Utility.GetSegmentLengths(segmentPrefabs);
```

### What to avoid

- Don't reiterate the obvious (e.g., `// Set x to 5` before `x = 5`)
- Don't add XML docs to code you didn't write or modify
- Don't use heavy section separator blocks (e.g., `// ======` or `// ------`)
- Don't use `#region` / `#endregion` blocks (use blank lines between logical groups instead)
- Don't use `this.` qualifier (exception: disambiguating a parameter from a field)

---

## Using directives

- All `using` directives must be at the **top of the file**, outside the namespace
- System directives first, then third-party, then project-local
- Sorting is enforced by EditorConfig (`dotnet_sort_system_directives_first = true`)

```csharp
using System;
using System.Collections.Generic;
using System.IO;
using UnityEngine;
using Gimbl;
using SL.Config;
```

---

## File-level ordering

All definitions within a file follow this vertical ordering from top to bottom:

1. **File-level XML documentation** (`/// <summary>` block describing the file)
2. **Using directives**
3. **Namespace declaration** (file-scoped preferred: `namespace SL.Config;`)
4. **Enumerations** (type definitions that other code depends on)
5. **Class declaration** with members in this order:
   a. Constants (`const` and `static readonly` fields)
   b. Public fields (Unity Inspector-serialized)
   c. Private fields (`_camelCase`)
   d. Properties
   e. Unity lifecycle methods (`Awake`, `Start`, `Update`, `OnDestroy`)
   f. Public methods
   g. Private methods
   h. Nested classes

### Visibility ordering

Within each member kind, order by visibility: `public` -> `internal` -> `protected` ->
`private`. Always write access modifiers explicitly — never rely on C#'s implicit `private`
default. Unity lifecycle methods appear in their natural execution order regardless of
visibility.

### One class per file

Each `.cs` file should contain exactly one public type. The file name must match the class
name (e.g., `OccupancyZone.cs` contains `class OccupancyZone`). Nested helper classes and
message types may remain in the containing class's file.

---

## Guard clauses and boolean expressions

Prefer early returns (guard clauses) over deeply nested conditionals. Use explicit boolean
checks, `string.IsNullOrEmpty()` for strings, and `== null` / `!= null` for null checks:

```csharp
/// <summary>Checks if the occupancy duration has been met while the animal is in the zone.</summary>
void Update()
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

C# does not enforce trailing commas in the same way as Python. Follow CSharpier's output for
comma placement in multi-line constructs.

### Brace rules

Always use braces for control flow statements, even single-line bodies:

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

The `.editorconfig` file enforces naming conventions, brace style, and spacing rules in
IDEs. It is the source of truth for style rules that CSharpier does not cover (naming,
`var` preferences, expression-bodied members).

### CSharpier ignore

The `.csharpierignore` file excludes Unity-generated directories (`Library/`, `Temp/`,
`Logs/`) and third-party packages from formatting.

---

## Configuration files

Canonical configs are stored in [assets/](assets/). When working in a C# project, verify that
`.csharpierrc.yaml`, `.editorconfig`, and `.csharpierignore` in the project root match these:

- [assets/.csharpierrc.yaml](assets/.csharpierrc.yaml)
- [assets/.editorconfig](assets/.editorconfig)
- [assets/.csharpierignore](assets/.csharpierignore)

The `.csharpierignore` contains generic entries only. Individual projects may need additional
project-specific entries (e.g., paths to auto-generated scripts).

---

## Related skills

| Skill               | Relationship                                                       |
|---------------------|--------------------------------------------------------------------|
| `/python-style`     | Provides Python conventions; C# conventions parallel these         |
| `/cpp-style`        | Provides C++ conventions; C# conventions parallel these            |
| `/project-layout`   | Provides C# Unity directory tree; invoke for project structure     |
| `/readme-style`     | Provides README conventions; invoke for README tasks               |
| `/commit`           | Provides commit message conventions; invoke for commit tasks       |
| `/skill-design`     | Provides skill file conventions; invoke for skill authoring tasks  |
| `/explore-codebase` | Provides project context that informs style-compliant code changes |

---

## Proactive behavior

When reviewing or modifying C# code, proactively check for style violations and fix them. When
writing new code, apply all conventions from this skill and its references without being asked.
If you notice existing code near your changes that violates conventions, mention it to the user
but do not fix it unless asked.

---

## Verification checklist

**You MUST verify your edits against this checklist before submitting any changes to C# files.**

```text
C# Style Compliance:
- [ ] XML documentation on all public and private members
- [ ] Summary tags use third-person imperative mood ("Provides..." not "This class provides...")
- [ ] Boolean members documented with "Determines whether..."
- [ ] File-level XML summary comment present
- [ ] All lines ≤ 120 characters
- [ ] 4-space indentation, no tabs
- [ ] Allman brace style (opening braces on new lines)
- [ ] LF line endings
- [ ] Full words used (no abbreviations like pos, idx, val, msg)
- [ ] Private fields use _camelCase prefix
- [ ] Public fields use camelCase (Unity serialized fields)
- [ ] Public properties use PascalCase
- [ ] Constants use PascalCase (const and static readonly)
- [ ] Methods use PascalCase (both public and private)
- [ ] Enum types and values use PascalCase
- [ ] Access modifiers always explicit (never rely on implicit private)
- [ ] Using directives at top of file, outside namespace
- [ ] System directives sorted first
- [ ] String interpolation used (not string.Format or concatenation)
- [ ] Named arguments used for boolean params and ambiguous calls
- [ ] Null checks use explicit comparison (== null, != null)
- [ ] Guard clauses / early returns preferred over deep nesting
- [ ] Fields marked readonly when only assigned in constructor/initializer
- [ ] No #region blocks; no this. qualifier (except disambiguation)
- [ ] One public type per file; file name matches class name
- [ ] Unity logging uses Debug.LogError/LogWarning/Log appropriately
- [ ] CSharpier formatting applied before commit
- [ ] Inline comments use third person imperative

Unity-Specific Compliance:
- [ ] MonoBehaviour fields for Inspector use public camelCase
- [ ] Private backing fields use [SerializeField] when Inspector access needed
- [ ] Lifecycle methods in execution order (Awake, Start, Update, OnDestroy)
- [ ] TryGetComponent used for safe component access
- [ ] No unnecessary GetComponent calls in Update loops
- [ ] No LINQ in Update/FixedUpdate (allocations in hot paths)
- [ ] IDisposable resources cleaned up in OnDestroy
```
