---
name: explore-dependencies
description: >-
  Explores installed ataraxis and Sun Lab dependency source code to build a live API snapshot.
  Discovers public classes, functions, and constants exported by each dependency, identifies
  replacement opportunities where project code reimplements existing library functionality, and
  produces a structured dependency API snapshot. Use when starting a session on a project with
  ataraxis dependencies, before writing code that uses ataraxis library features, or when the user
  asks about available library APIs.
user-invocable: true
---

# Dependency exploration

Explores installed ataraxis and Sun Lab library source code to build a live API snapshot for the
current project.

You MUST run this skill before writing code that uses ataraxis library features. Static reference
tables go stale; reading the actual installed source is the only reliable way to know what APIs are
available and how they work.

---

## Scope

**Covers:**
- Discovering ataraxis and Sun Lab dependencies from `pyproject.toml`
- Locating installed package source code on disk
- Reading `__init__.py` and `__all__` exports to enumerate public API surfaces
- Reading source signatures, docstrings, and return types for each public asset
- Cross-referencing project code against discovered APIs to find replacement opportunities
- Producing a structured "Dependency API snapshot" organized by library

**Does not cover:**
- Third-party library exploration (NumPy, Click, etc.) — read their docs directly
- Modifying dependency versions or adding new dependencies (invoke `/pyproject-style`)
- Applying Python coding conventions (invoke `/python-style`)
- Exploring the project's own codebase structure (invoke `/explore-codebase`)

---

## Workflow

You MUST follow these steps when this skill is invoked.

### Step 1: Load the library catalog

Read [library-catalog.md](references/library-catalog.md) to understand the ataraxis library
ecosystem, domain-to-library mappings, and import names.

### Step 2: Identify project dependencies

Read the project's `pyproject.toml` and extract all ataraxis and Sun Lab dependencies from
`[project.dependencies]` and `[project.optional-dependencies]`. Match package names that start
with `ataraxis-` or `sl-`.

If `pyproject.toml` is not found, check for `setup.cfg`, `setup.py`, or `requirements.txt` as
fallbacks.

Record each dependency with its package name and corresponding import name (replace hyphens with
underscores).

### Step 3: Locate installed source code

For each identified dependency, resolve the installed package location:

```bash
python -c "import <import_name>; print(<import_name>.__file__)"
```

This returns the path to the package's `__init__.py`. The parent directory contains all source
modules.

If a package is not installed, note it as unavailable and skip to the next dependency.

### Step 4: Enumerate public APIs

For each installed dependency:

1. Read the package's `__init__.py` file
2. Extract the `__all__` list to identify all public exports
3. For each exported name, identify whether it is a class, function, constant, or enum
4. Group exports by category (classes, functions, constants/enums)

### Step 5: Read API details

For each public class and function discovered in Step 4:

1. Locate the source module that defines it (follow the import in `__init__.py`)
2. Read the definition to extract:
   - **Signature**: Full function/method signature with type annotations
   - **Docstring summary**: First line of the docstring
   - **Key parameters**: Parameter names, types, and defaults
   - **Return type**: The annotated return type
3. For classes, also extract:
   - **`__init__` signature**: Constructor parameters
   - **Public methods**: Method names and their signatures
   - **Class/static methods**: Any `@classmethod` or `@staticmethod` methods

Focus on signatures and summaries. Do not read full method bodies unless needed to understand
behavior.

### Step 6: Identify replacement opportunities

Scan the project's source code for patterns that duplicate functionality provided by the
discovered dependency APIs:

| Pattern in project code                    | Likely replacement                          |
|--------------------------------------------|---------------------------------------------|
| `print()` or `click.echo()` for output     | `console.echo()` (with `raw=True` if needed)|
| `raise` for error reporting                | `console.error()`                           |
| `isinstance` chains to normalize to list   | `ensure_list()`                             |
| Manual slicing loops for batching          | `chunk_iterable()`                          |
| `os.cpu_count()` arithmetic               | `resolve_worker_count()`                    |
| `time.sleep()` or `time.time()` patterns   | `PrecisionTimer` methods                    |
| `datetime.now().strftime()` calls          | `get_timestamp()`                           |
| Manual `yaml.dump()`/`yaml.load()` calls   | `YamlConfig` subclass                       |
| `multiprocessing.Array` or `Value` usage   | `SharedMemoryArray`                         |
| Direct file writes in acquisition loops    | `DataLogger` + `LogPackage`                 |
| `os.makedirs()` or `Path.mkdir()` patterns | `ensure_directory_exists()`                 |

Report each replacement opportunity with the file location and the suggested library alternative.

### Step 7: Produce the dependency API snapshot

Present the results using the output format below. This snapshot gives the agent (and user) a
complete picture of what the project's ataraxis dependencies provide.

---

## Output format

Organize the snapshot by library, with sections for each dependency.

### Per-library section

```markdown
## <library-name> (v<version>)

**Import:** `from <import_name> import ...`
**Source:** `<installed_path>`

### Classes

| Class | Summary | Key methods |
|-------|---------|-------------|
| `ClassName` | Docstring summary | `method_one()`, `method_two()` |

### Functions

| Function | Signature | Summary |
|----------|-----------|---------|
| `function_name` | `(param: type, ...) -> return_type` | Docstring summary |

### Constants and enums

| Name | Type | Summary |
|------|------|---------|
| `CONSTANT_NAME` | `type` | Description |
| `EnumName` | `enum` | Members: `MEMBER_A`, `MEMBER_B`, ... |
```

### Replacement opportunities section

After all per-library sections, include a summary of replacement opportunities:

```markdown
## Replacement opportunities

| File | Current pattern | Suggested replacement |
|------|----------------|----------------------|
| `src/module.py:42` | `print(table)` | `console.echo(message=table, raw=True)` |
| `src/utils.py:15` | `time.sleep(0.1)` | `PrecisionTimer.delay()` |
```

If no replacement opportunities are found, state: "No replacement opportunities identified."

---

## Handling large dependencies

For dependencies with many exports (15+ public names), use the Task tool with
`subagent_type: Explore` to parallelize the API reading. Assign one subagent per large library
to read source files concurrently.

For small dependencies (fewer than 15 exports), read the APIs directly without subagents.

---

## Related skills

| Skill               | Relationship                                                          |
|---------------------|-----------------------------------------------------------------------|
| `/python-style`     | Requires this skill before writing code that uses ataraxis features   |
| `/pyproject-style`  | Manages dependency versions and additions; defer dependency changes   |
| `/explore-codebase` | Explores project structure; complements dependency exploration        |
| `/commit`           | Invoke after completing code changes informed by the API snapshot     |

---

## Proactive behavior

This skill should be invoked at session start alongside `/explore-codebase` when the project has
ataraxis or Sun Lab dependencies. The `/python-style` skill explicitly requires this skill before
writing code that touches ataraxis library domains.

When invoked proactively, present the dependency API snapshot and replacement opportunities before
proceeding to code changes. Wait for user acknowledgment before modifying code.

---

## Verification checklist

**You MUST verify the exploration output against this checklist before presenting it to the user.**

```text
Dependency Exploration Compliance:
- [ ] All ataraxis/Sun Lab dependencies identified from pyproject.toml
- [ ] Each installed dependency's source location resolved
- [ ] Unavailable packages noted and skipped
- [ ] __all__ exports read for each installed dependency
- [ ] Public classes documented with constructors and public methods
- [ ] Public functions documented with signatures and summaries
- [ ] Constants and enums documented with types and members
- [ ] Replacement opportunities scanned and reported
- [ ] Output organized by library with consistent table format
- [ ] Snapshot includes version numbers where available
- [ ] No code modifications made during exploration
```
