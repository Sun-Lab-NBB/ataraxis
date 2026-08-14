---
name: csharp-style
description: >-
  Applies C# coding conventions when writing, reviewing, or refactoring code. Covers .cs files, XML documentation,
  naming, formatting, error handling, using directives, file ordering, and Unity-specific patterns. Use when writing new
  C# code, modifying existing code, reviewing pull requests, or when the user asks about C# coding standards.
user-invocable: false
---

# C# code style guide

Applies C# coding conventions.

You MUST read this skill and load the relevant reference files before writing or modifying C# code. You MUST verify your
changes against the checklist before submitting.

---

## Scope

**Covers:**
- C# code style (XML documentation, naming, formatting, error handling)
- Using directive conventions and file organization
- Class design, enums, properties, and inheritance patterns
- Unity-specific patterns (MonoBehaviour, ScriptableObject, serialization)
- CSharpier and EditorConfig tooling conventions
- Cross-language consistency with C++ and Python conventions

**Does not cover:**
- C# Unity directory structure (see `/project-layout`)
- README file conventions (see `/readme-style`)
- Commit message conventions (see `/commit`)
- Skill file and CLAUDE.md conventions (see `/skill-design`)
- Codebase exploration workflows (see `/explore-codebase`)

---

## Workflow

You MUST follow these steps when this skill is invoked.

### Step 1: Read this skill

Read this entire file. The core conventions below apply to ALL C# code.

### Step 2: Load relevant reference files

Based on the task, load the appropriate reference files:

| Task                                                                                                                                   | Reference to load                                           |
|----------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------|
| Writing or modifying XML docs / type usage / comments                                                                                  | [xml-docs-and-types.md](references/xml-docs-and-types.md)   |
| Writing classes, enums, or Unity components                                                                                            | [class-patterns.md](references/class-patterns.md)           |
| Using LINQ, async, IDisposable, testing, error handling, function calls, blank lines, formatting, tooling, config files, guard clauses | [libraries-and-tools.md](references/libraries-and-tools.md) |
| Deploying or verifying tool config files                                                                                               | [assets/](assets/) directory                                |
| Reviewing code before submission                                                                                                       | [anti-patterns.md](references/anti-patterns.md)             |

Load multiple references when the task spans multiple domains.

### Step 3: Apply conventions

Write or modify C# code following all conventions from this file and the loaded references.

### Step 4: Verify compliance

Complete the verification checklist at the end of this file. Every item must pass before submitting work. For
anti-pattern examples, load [anti-patterns.md](references/anti-patterns.md).

---

## Cross-language consistency

Projects span Python, C++, and C#. These conventions maximize visual and structural consistency across languages while
respecting each language's idiomatic standards.

**Shared across all languages:**
- 120 character line limit
- 4-space indentation (no tabs)
- Comprehensive documentation on ALL public and private members
- Third-person imperative mood for documentation ("Provides...", "Determines whether...")
- A leading underscore marks a symbol private to the module or class that defines it (`_snake_case` for any private
  Python symbol, `_snake_case` for C++ data members, `_camelCase` for C# fields)
- Full words in identifiers (no abbreviations)
- Guard clauses preferred over deep nesting
- Prose over bullet lists in documentation
- No example/code blocks in documentation (they go stale)
- I/O operations separated from processing logic
- Only full stops and commas separate clauses in documentation prose (no semicolons, no em-dashes)
- State what the code does now, not what it avoids doing or formerly did (positive description)

**Shared between C++ and C# only:**
- Allman brace style (opening braces on new lines, where Python uses indentation)

**C#-specific divergences from C++:**
- Constants use PascalCase (not `kPrefix` as in C++)
- Enum values use PascalCase (not `kPrefix` as in C++)
- Namespaces use PascalCase (not snake_case as in C++)
- Consecutive assignment alignment is NOT used (CSharpier does not support it)
- `#region` blocks are NOT used (prefer blank lines between logical groups)

**C#-specific divergences from Python:**
- Methods and properties use PascalCase (not snake_case as in Python)
- Constants use PascalCase (not `_UPPER_SNAKE_CASE` as in Python)
- Enum values use PascalCase (not `UPPER_SNAKE_CASE` as in Python)
- Private fields use `_camelCase` (not `_snake_case` as in Python)
- Public fields use camelCase (not snake_case as in Python)
- Documentation uses XML `<summary>` tags (not Google-style docstrings)

**Symbol visibility:**
- Access modifiers carry the boundary intent that the underscore prefix carries in Python and C++
- `private` marks a member owned by its declaring type, `internal` a member owned by its assembly
- Any symbol consumed outside its declaring assembly or namespace is made `public` deliberately
- No code reaches into another type's internals to work around a missing promotion
- A member consumed only inside its declaring type stays `private` and one consumed only inside its assembly stays
  `internal`, so a real consumer at that boundary earns each access modifier
- An asset with no consumer is removed, which covers methods, properties, fields, types, constants, and enum members
- The compiler reports an unused `private` member and stays silent about an unused `internal` or `public` one, so those
  are found by reading
- Test assemblies are the sole exception and may access the internals of the code under test, so no test counts as a
  consumer and a member exercised only by its own tests is removed with those tests

---

## Naming conventions

### Variables

Use **full words**, not abbreviations:

| Avoid         | Prefer                   |
|---------------|--------------------------|
| `pos`, `idx`  | `position`, `index`      |
| `msg`, `val`  | `message`, `value`       |
| `seg`, `trig` | `segment`, `trigger`     |
| `cfg`, `cnt`  | `configuration`, `count` |

### Identifiers

| Element           | Convention    | Example                                   |
|-------------------|---------------|-------------------------------------------|
| Classes           | PascalCase    | `StimulusTriggerZone`, `TaskTemplate`     |
| Methods           | PascalCase    | `ResetState`, `GetPrefabLength`           |
| Public fields     | camelCase     | `trackLength`, `requireLick`, `isActive`  |
| Public properties | PascalCase    | `IsOccupancyMode`, `CorridorSpacingUnity` |
| Private fields    | `_camelCase`  | `_currentSegmentIndex`, `_occupancyTimer` |
| Local variables   | camelCase     | `segmentPath`, `measuredLength`           |
| Parameters        | camelCase     | `configPath`, `qosLevel`                  |
| Constants         | PascalCase    | `LengthComparisonEpsilon`                 |
| Enum types        | PascalCase    | `ControllerTypes`, `StimulusMode`         |
| Enum values       | PascalCase    | `LinearTreadmill`, `OccupancyBased`       |
| Namespaces        | PascalCase    | `Gimbl`, `Project.Config`                 |
| Interfaces        | `IPascalCase` | `IConfigurable`, `IResettable`            |
| Type parameters   | `TPascalCase` | `TMessage`, `TConfig`                     |

### Public fields vs properties

In Unity projects, MonoBehaviour and ScriptableObject classes expose **public fields** using camelCase for Inspector
serialization. These are effectively configuration parameters set via the Unity Editor:

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

You MUST mark fields as `readonly` when they are only assigned in the constructor or initializer. For detailed
immutability patterns (`readonly struct`, records, `in` parameters), see
[class-patterns.md](references/class-patterns.md).

---

## Function calls

See [libraries-and-tools.md](references/libraries-and-tools.md) for named argument conventions.

---

## Error handling

MonoBehaviour code reports failures through Unity's logging system. Editor tools, static utility classes, and pure C#
libraries that do not inherit from MonoBehaviour raise standard C# exceptions instead. See
[libraries-and-tools.md](references/libraries-and-tools.md) for worked examples of both.

### When to use each approach

| Context                          | Error mechanism           | Example                                  |
|----------------------------------|---------------------------|------------------------------------------|
| MonoBehaviour lifecycle methods  | `Debug.LogError` + return | `Task.Start()`, `OccupancyZone.Update()` |
| Editor tools and menu commands   | Exceptions                | `CreateTask.CreateNewTask()`             |
| Static utility methods           | Exceptions                | `Utility.GetPrefabLength()`              |
| Pure C# classes (non-Unity)      | Exceptions                | `ConfigParser.Parse()`                   |
| MonoBehaviour constructors/Awake | `Debug.LogError` + return | Field validation in `Awake()`            |
| Plain C# class constructors      | Exceptions                | `Communication(portName, baudRate)`      |

### Error message format

Use a structured format: context ("Unable to..."), constraint ("must be..."), actual value ("but [actual state]."). Use
`Debug.LogError()` for failures that prevent continuation, `Debug.LogWarning()` for non-critical issues, and
`Debug.Log()` for informational messages. For multi-line error messages, assign the message to a local variable before
passing it. This matches the Python convention of assigning to a `message` variable before calling `console.error()`.

### Null handling

- Use explicit null checks: `if (template == null)`
- Use `string.IsNullOrEmpty()` for string validation
- Use `TryGetComponent<T>()` for safe component access in Unity
- Prefer null-conditional operator (`?.`) for optional chains
- Prefer null-coalescing operator (`??`) for default values

---

## Comments

Heavy section separator blocks (`// ======` or `// ------`) are not used. Blank lines separate the logical groups
instead, the same replacement `#region` blocks get. See [xml-docs-and-types.md](references/xml-docs-and-types.md) for
inline comment conventions and what to avoid.

### Wrap width

Break an XML doc or comment line only where it would otherwise pass 120 characters, and fill each line to that limit
before breaking. CSharpier reflows code and leaves comment prose as written, so prose wrapped narrower stays a rigid
block that re-wraps badly at every other width. The test is mechanical: a wrapped line that ends before column 100 while
its next word would still fit under 120 is re-flowed. A line ending early because the sentence or the paragraph ends, or
because it holds a list item or a code span, is already correct.

---

## Using directives

- All `using` directives must be at the **top of the file**, outside the namespace
- System directives first, then every remaining directive in alphabetical order
- `dotnet_sort_system_directives_first = true` settles the System-first placement alone. Roslyn has no notion of
  third-party versus project-local, so no EditorConfig key can express a tiered order, and the codebase sorts
  alphabetically after the System block

```csharp
using System;
using System.Collections.Generic;
using System.IO;
using Gimbl;
using SL.Config;
using UnityEngine;
```

### `using static` directives

Do NOT use `using static` directives. Always qualify static method calls with the type name for clarity:

```csharp
// Avoid - using static obscures where methods come from
using static UnityEngine.Mathf;
float result = Clamp(value, 0f, 1f);

// Good - qualified call is explicit
float result = Mathf.Clamp(value, 0f, 1f);
```

### Global usings

Do NOT use C# 10 global `using` directives or implicit usings. Every file must contain its own explicit `using`
directives. This ensures each file is self-contained and matches the C++ convention of explicit `#include` directives
per file.

---

## File-level ordering

All definitions within a file follow this vertical ordering from top to bottom:

1. **File-level XML documentation** (`/// <summary>` block describing the file)
2. **Using directives**
3. **Namespace declaration** (block-scoped is the project convention: `namespace Project.Config { ... }`, with class
   members indented one level inside the block, which `assets/.editorconfig` enforces through
   `csharp_style_namespace_declarations = block_scoped:suggestion`)
4. **Enumerations** (type definitions that other code depends on)
5. **Class declaration** with members in this order: a. Constants (`const` and `static readonly` fields) b. Public
   fields (Unity Inspector-serialized) c. Private fields (`_camelCase`) d. Properties e. Unity lifecycle methods
   (`Awake`, `Start`, `Update`, `OnDestroy`) f. Public methods g. Private methods h. Nested classes

### Visibility ordering

Within each member kind, order by visibility: `public` -> `internal` -> `protected` -> `private`. Always write access
modifiers explicitly, so every member states its visibility in source rather than inheriting C#'s implicit `private`
default. Unity lifecycle methods appear in their natural execution order regardless of visibility.

### Call-hierarchy ordering

Within each visibility group, definitions should **loosely follow the order in which they are called** during the
class's runtime. When there is no clear call hierarchy, group definitions **by purpose**. This matches the Python
convention of ordering definitions by call sequence within each visibility group.

For MonoBehaviour classes, this naturally follows from the lifecycle ordering: `Awake` calls initialization helpers,
`Start` calls setup helpers, `Update` calls per-frame helpers.

### One class per file

Each `.cs` file should contain exactly one public type. The file name must match the class name (e.g.,
`OccupancyZone.cs` contains `class OccupancyZone`). Nested helper classes and message types may remain in the containing
class's file.

---

## Guard clauses and boolean expressions

See [libraries-and-tools.md](references/libraries-and-tools.md) for guard clause and boolean expression conventions.

---

## Blank lines

See [libraries-and-tools.md](references/libraries-and-tools.md) for blank line conventions.

---

## Formatting, tooling, and configuration files

See [libraries-and-tools.md](references/libraries-and-tools.md) for line length, formatting, and brace rules, for
CSharpier and EditorConfig tooling, and for configuration file references.

---

## Related skills

| Skill               | Relationship                                                          |
|---------------------|-----------------------------------------------------------------------|
| `/python-style`     | Provides the Python conventions that these C# conventions parallel    |
| `/cpp-style`        | Provides the C++ conventions that these C# conventions parallel       |
| `/project-layout`   | Provides the C# Unity directory tree, invoke it for project structure |
| `/readme-style`     | Provides README conventions, invoke it for README tasks               |
| `/audit-project`    | Audits the code just written, before it is committed                  |
| `/commit`           | Provides commit message conventions, invoke it for commit tasks       |
| `/skill-design`     | Provides skill file conventions, invoke it for skill authoring tasks  |
| `/explore-codebase` | Provides project context that informs style-compliant code changes    |

---

## Proactive behavior

When reviewing or modifying C# code, proactively check for style violations and fix them. When writing new code, apply
all conventions from this skill and its references without being asked. If you notice existing code near your changes
that violates conventions, mention it to the user but do not fix it unless asked.

---

## Verification checklist

**You MUST verify your edits against this checklist before submitting any changes to C# files.**

```text
C# Style Compliance:

Judgment items. No tool inspects these, so this checklist is their only enforcement. Walk every one
against the code you wrote.
- [ ] XML documentation on all public and private members
- [ ] Summary tags use third-person imperative mood ("Provides..." not "This class provides...")
- [ ] Boolean members documented with "Determines whether..."
- [ ] File-level XML summary comment present
- [ ] Sentences in comments and XML docs stay under 40 words
- [ ] Comment and XML doc prose fills each line to 120 characters, with no line ending before column 100 while its next
      word would still fit
- [ ] Every member defaults to the summary line alone, with a remarks block earned by a nameable non-obvious property
- [ ] Each retained sentence survives the cover test (unable to be reconstructed from name, signature, and body)
- [ ] Documentation records current behavior only, never the edit that produced it
- [ ] Edits leave documentation no longer than it started unless the new behavior is harder to derive
- [ ] File-level summary is at most 5 sentences, with detail relocated into the members it documents
- [ ] Documentation carries only facts the reader cannot infer from the code (no padding or trivia)
- [ ] XML docs describe what the code does, with project usage and call-site context left out
- [ ] Peer-format expectations documented only when counter-intuitive enough to mislead the reader
- [ ] XML doc length proportional to conceptual difficulty, not to how long the method is
- [ ] <param> and <returns> descriptions do not restate type information
- [ ] XML doc accurately describes the method's observable behavior
- [ ] Comments and XML docs free of typos and grammar errors
- [ ] Inline comments explain why, not what (no narrate-the-code comments)
- [ ] No stale references in comments (closed issues, removed code, outdated TODOs)
- [ ] Prose separators are full stops and commas only, no semicolons or em-dashes (colons, hyphen bullets, and code
      syntax exempt)
- [ ] Documentation states what the code does, not what it is not or used to be (contrast only when load-bearing)
- [ ] Full words used (no abbreviations like pos, idx, val, msg)
- [ ] Methods use PascalCase (both public and private)
- [ ] Enum types and values use PascalCase
- [ ] Access modifiers always explicit (never rely on implicit private)
- [ ] Symbols consumed outside their declaring assembly or namespace are public, not reached into
- [ ] Members whose consumers all live inside the declaring type stay private, and those whose consumers all
      live inside the assembly stay internal (tests are not consumers)
- [ ] Every asset has a consumer, so methods, properties, fields, types, constants, and enum members that
      nothing outside the test suite references are removed rather than kept
- [ ] Using directives at top of file, outside namespace
- [ ] String interpolation used (not string.Format or concatenation)
- [ ] Named arguments used for boolean params and ambiguous calls
- [ ] Null checks use explicit comparison (== null, != null)
- [ ] Guard clauses / early returns preferred over deep nesting
- [ ] No #region blocks; no this. qualifier (except disambiguation)
- [ ] No heavy section separator blocks (// ====== or // ------)
- [ ] No IDE suppression comments (ReSharper/Rider // ReSharper disable etc.); use #pragma warning / [SuppressMessage]
- [ ] One public type per file; file name matches class name
- [ ] Unity logging uses Debug.LogError/LogWarning/Log appropriately
- [ ] XML tag ordering: summary -> remarks -> typeparam -> param -> returns -> exception
- [ ] <exception> tags in alphabetical order by type name
- [ ] <inheritdoc/> used only when base documentation fully describes override behavior
- [ ] No <example> or <code> tags in XML documentation
- [ ] No using static directives; no global usings
- [ ] ToString format matches ClassName(key=value, key=value) pattern
- [ ] in used only for large readonly struct parameters (not small types or reference types)
- [ ] out parameters follow TryX pattern (return bool, populate out on success)
- [ ] Error mechanism matches context (Debug.LogError in MonoBehaviour; exceptions elsewhere)
- [ ] Error messages state context, constraint, and actual value ("Unable to...", "must be...", "but [actual state]")
- [ ] Multi-line error messages assigned to variable before passing
- [ ] Prose used in <remarks> blocks (not bullet lists)
- [ ] Property summaries are single sentences (no summary + remarks split)
- [ ] Test methods have only <summary> (no <param>, <returns>, or <exception> tags)
- [ ] Test classes use the Tests suffix and test methods use MethodName_Scenario_ExpectedResult
- [ ] I/O operations separated from processing logic
- [ ] Path.Combine used for path construction (not string concatenation)
- [ ] StringComparison.Ordinal used for internal string matching
- [ ] File sections ordered file summary, using directives, namespace, enums, then class declaration
- [ ] Class members ordered constants, public fields, private fields, properties, lifecycle methods, public methods,
      private methods, nested classes
- [ ] Members ordered public -> internal -> protected -> private within each member kind
- [ ] Methods ordered by call hierarchy within each visibility group
- [ ] Inline comments use third-person imperative mood
- [ ] MonoBehaviour fields for Inspector use public camelCase
- [ ] Private backing fields use [SerializeField] when Inspector access needed
- [ ] Attribute stacking order: serialization, Unity, editor, validation, custom
- [ ] Lifecycle methods in execution order (Awake, Start, Update, OnDestroy)
- [ ] TryGetComponent used for safe component access
- [ ] No unnecessary GetComponent calls in Update loops
- [ ] No LINQ in Update/FixedUpdate (allocations in hot paths)
- [ ] IDisposable resources cleaned up in OnDestroy
- [ ] Event subscriptions have matching unsubscriptions (OnEnable<->OnDisable, Start<->OnDestroy)
- [ ] Coroutines stopped in OnDisable/OnDestroy to prevent orphaned execution
- [ ] Async methods carry the Async suffix, with async void reserved for Unity event handlers and UI callbacks
- [ ] CancellationToken passed last, and ConfigureAwait(false) used in library code outside MonoBehaviours
- [ ] #if UNITY_EDITOR used for editor-only code; [Conditional] preferred over #if DEBUG
- [ ] Switch expressions used for pure value mapping; switch statements for side effects

Tooling-enforced items. CSharpier and the EditorConfig-configured Roslyn analyzers settle each of
these, so run `csharpier --check .` and read the analyzer output rather than hand-checking them.
They stay listed for reviews performed without the tooling.
- [ ] All lines <= 120 characters
- [ ] 4-space indentation, no tabs
- [ ] Allman brace style (opening braces on new lines)
- [ ] LF line endings
- [ ] CSharpier formatting applied before commit
- [ ] Private fields use _camelCase prefix
- [ ] Public fields use camelCase (Unity serialized fields)
- [ ] Public properties use PascalCase
- [ ] Constants use PascalCase (const and static readonly)
- [ ] Namespaces, classes, and structs use PascalCase, interfaces use IPascalCase, type parameters use TPascalCase
- [ ] Local variables and parameters use camelCase
- [ ] System directives sorted first
- [ ] Fields marked readonly when only assigned in constructor/initializer
- [ ] Static methods used when no instance state is accessed
```
