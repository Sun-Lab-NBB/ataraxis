---
name: project-layout
description: >-
  Applies Sun Lab project directory structure conventions when creating new projects, adding
  directories, or verifying project layout. Covers the five project archetypes (Python-only,
  Python+C++ extension, C++ PlatformIO library, C++ PlatformIO firmware, C# Unity), common root
  files, environment directories, test directories, and documentation directory placement. Use
  when creating a new project, adding top-level directories, restructuring a project, or when the
  user asks about project directory conventions.
user-invocable: true
---

# Project layout

Applies Sun Lab conventions for project directory structure across all five project archetypes.

You MUST read this skill before creating, restructuring, or verifying any Sun Lab project's
directory layout. You MUST verify your changes against the checklist before submitting.

---

## Scope

**Covers:**
- Project directory trees for all five Sun Lab archetypes
- Common root files and their purposes
- Environment directories (`envs/`) with OS-specific files
- Test directory conventions (`tests/` vs `test/`)
- Documentation directory placement (defers to `/api-docs` for internal structure)
- `.github/` directory structure
- Archetype identification criteria

**Does not cover:**
- File-level ordering within source files (see `/python-style`, `/cpp-style`, `/csharp-style`)
- Documentation file contents and Sphinx configuration (see `/api-docs`)
- pyproject.toml structure and fields (see `/pyproject-style`)
- README file contents (see `/readme-style`)
- Code style or formatting rules (see language-specific style skills)

---

## Workflow

You MUST follow these steps when this skill is invoked.

### Step 1: Identify the project archetype

Determine which archetype applies using this table:

| Archetype               | Key indicators                                                       |
|-------------------------|----------------------------------------------------------------------|
| Python-only             | `pyproject.toml` + `src/` layout, no `.clang-format` or `CMakeLists` |
| Python + C++ extension  | `pyproject.toml` + `CMakeLists.txt` + `src/c_extensions/`            |
| C++ PlatformIO library  | `platformio.ini` + `library.json` + `src/*.h`                        |
| C++ PlatformIO firmware | `platformio.ini` + `src/main.cpp`, no `library.json`                 |
| C# Unity                | `Assets/` + `ProjectSettings/` + `*.slnx`                            |

### Step 2: Load the reference tree

Read [archetype-trees.md](references/archetype-trees.md) and locate the section matching the
identified archetype. The reference tree is the authoritative source for directory structure.

### Step 3: Apply conventions

Create or verify the project structure against the reference tree and the rules below. When
creating a new project, generate all required directories and files. When verifying, report any
deviations from the expected structure.

### Step 4: Verify compliance

Complete the verification checklist at the end of this file. Every item must pass before
submitting work.

---

## Common root files

These files appear at the root of all (or most) Sun Lab projects:

| File         | All archetypes | Purpose                                           |
|--------------|----------------|---------------------------------------------------|
| `LICENSE`    | Yes            | GPL-3.0 license                                   |
| `README.md`  | Yes            | Project documentation (see `/readme-style`)       |
| `.gitignore` | Yes            | Git ignore patterns                               |
| `CLAUDE.md`  | Yes            | Claude Code project instructions                  |
| `tox.ini`    | Python + C++   | Automation orchestration (lint, type, test, docs) |

### Python-specific root files

| File              | Archetypes                   | Purpose                                      |
|-------------------|------------------------------|----------------------------------------------|
| `pyproject.toml`  | Python-only, Python+C++ ext  | Build config, metadata, tool settings        |
| `CMakeLists.txt`  | Python+C++ ext               | CMake build config for nanobind extension    |
| `Doxyfile`        | Python+C++ ext               | Doxygen documentation configuration          |
| `.clang-format`   | Python+C++ ext               | C++ formatting configuration                 |
| `.clang-tidy`     | Python+C++ ext               | C++ linting configuration                    |

### PlatformIO-specific root files

| File              | Archetypes                         | Purpose                              |
|-------------------|------------------------------------|--------------------------------------|
| `platformio.ini`  | PlatformIO lib, PlatformIO fw      | PlatformIO build configuration       |
| `library.json`    | PlatformIO lib only                | PlatformIO library manifest          |
| `Doxyfile`        | PlatformIO lib, PlatformIO fw      | Doxygen documentation configuration  |
| `.clang-format`   | PlatformIO lib, PlatformIO fw      | C++ formatting configuration         |
| `.clang-tidy`     | PlatformIO lib, PlatformIO fw      | C++ linting configuration            |

### Unity-specific root files

| File                  | Purpose                                       |
|-----------------------|-----------------------------------------------|
| `*.slnx`              | Unity solution file                           |
| `.editorconfig`       | Editor configuration (indentation, encoding)  |
| `.csharpierrc.yaml`   | CSharpier formatter configuration             |
| `.csharpierignore`    | CSharpier formatter ignore patterns           |

---

## Environment directories

Python projects (Python-only and Python+C++ extension) include an `envs/` directory with
OS-specific conda/mamba environment files:

```text
envs/
├── {abbr}_dev_lin.yml            # Linux conda environment specification
├── {abbr}_dev_lin_spec.txt       # Linux explicit package list
├── {abbr}_dev_osx.yml            # macOS conda environment specification
├── {abbr}_dev_osx_spec.txt       # macOS explicit package list
├── {abbr}_dev_win.yml            # Windows conda environment specification
└── {abbr}_dev_win_spec.txt       # Windows explicit package list
```

The `{abbr}` placeholder is a short project abbreviation (e.g., `axa` for ataraxis-automation,
`axbu` for ataraxis-base-utilities). Each platform has two files:
- `.yml` — human-readable conda environment specification used by `mamba env create`
- `_spec.txt` — explicit package list generated by `mamba list --explicit`

PlatformIO and Unity projects do NOT have `envs/` directories.

---

## Test directories

| Archetype               | Directory | Framework | File pattern                |
|-------------------------|-----------|-----------|-----------------------------|
| Python-only             | `tests/`  | pytest    | `module_test.py`            |
| Python + C++ extension  | `tests/`  | pytest    | `module_test.py`            |
| C++ PlatformIO library  | `test/`   | Unity (C) | `test_component.cpp`        |
| C++ PlatformIO firmware | (none)    | —         | No test directory           |
| C# Unity                | (none)    | —         | Unity Play Mode / Edit Mode |

### Python test structure

The `tests/` directory mirrors the `src/package_name/` subpackage structure:

```text
tests/
├── submodule/
│   └── module_test.py
└── standalone_test.py
```

### PlatformIO test structure

The `test/` directory (singular, not `tests/`) follows PlatformIO's native test convention:

```text
test/
└── test_component.cpp
```

---

## Documentation directory

All Python and C++ projects include a `docs/` directory for Sphinx documentation. For the
complete internal structure, Sphinx configuration, and RST templates, invoke `/api-docs`.

C# Unity projects do NOT have a `docs/` directory.

---

## `.github/` directory

Python projects include a `.github/` directory with issue templates:

```text
.github/
└── ISSUE_TEMPLATE/
    ├── bug_report.md
    └── feature_request.md
```

PlatformIO and Unity projects may or may not include `.github/` depending on whether they are
published to GitHub as standalone repositories.

---

## Source directory conventions

### Python-only (`src/` layout)

```text
src/
└── package_name/
    ├── __init__.py
    ├── module.py
    └── py.typed
```

### Python + C++ extension (`src/` flat namespace)

```text
src/
├── c_extensions/
│   └── module_ext.cpp
├── python_wrapper/
│   ├── __init__.py
│   └── wrapper_module.py
├── __init__.py
├── module_ext.pyi
└── py.typed
```

### PlatformIO library (header-only `src/`)

```text
src/
├── main.cpp                  # Development entry point (excluded from library)
├── primary_header.h
└── shared_assets.h
```

### PlatformIO firmware (`src/`)

```text
src/
├── main.cpp                  # Firmware entry point (setup/loop)
└── custom_module.h
```

### C# Unity (`Assets/`)

```text
Assets/
├── TaskName/
│   ├── Configurations/
│   ├── Materials/
│   ├── Prefabs/
│   ├── Scripts/
│   │   └── TaskScript.cs
│   ├── Sounds/
│   └── Textures/
├── Plugins/
└── Scenes/
```

---

## Related skills

| Skill              | Relationship                                                             |
|--------------------|--------------------------------------------------------------------------|
| `/api-docs`        | Owns the internal `docs/` structure; this skill owns directory placement |
| `/python-style`    | Owns file-level ordering within Python source files                      |
| `/cpp-style`       | Owns file-level ordering within C++ source files                         |
| `/csharp-style`    | Owns file-level ordering within C# source files                          |
| `/pyproject-style` | Owns `pyproject.toml` structure; references `src/` layout convention     |
| `/tox-config`      | Owns `tox.ini` conventions; `tox.ini` is a common root file             |
| `/readme-style`    | Owns `README.md` content conventions                                     |
| `/skill-design`    | Owns `plugins/automation/skills/` directory structure conventions         |

---

## Proactive behavior

You should proactively offer to invoke this skill when:
- Creating a new Sun Lab project from scratch
- The user asks where to place a new file or directory
- A project structure appears to deviate from conventions during exploration
- The user asks about project directory conventions or archetypes

---

## Verification checklist

You MUST verify your work against this checklist before submitting any layout changes.

```text
Project Layout Compliance:

Archetype Identification:
- [ ] Project archetype correctly identified from key indicators
- [ ] Reference tree loaded from archetype-trees.md

Common Root Files:
- [ ] LICENSE present (GPL-3.0)
- [ ] README.md present
- [ ] .gitignore present
- [ ] CLAUDE.md present
- [ ] Archetype-specific root files present (pyproject.toml, platformio.ini, etc.)

Source Directory:
- [ ] Python projects use src/ layout with package_name/ subdirectory
- [ ] Python+C++ extension uses flat namespace under src/ (c_extensions/, wrapper/, etc.)
- [ ] PlatformIO projects use src/ with header-only .h files
- [ ] Unity projects use Assets/ with task-specific subdirectories

Environment Directory:
- [ ] Python projects have envs/ with 6 files (3 platforms x 2 files each)
- [ ] envs/ file names use correct abbreviation prefix
- [ ] PlatformIO and Unity projects do NOT have envs/

Test Directory:
- [ ] Python projects use tests/ (plural) with _test.py suffix
- [ ] PlatformIO library projects use test/ (singular) with test_ prefix
- [ ] PlatformIO firmware and Unity projects have no dedicated test directory

Documentation Directory:
- [ ] Python and C++ projects have docs/ directory
- [ ] Unity projects do NOT have docs/

GitHub Directory:
- [ ] .github/ISSUE_TEMPLATE/ present for published GitHub repositories

No Duplicates:
- [ ] Directory trees not duplicated in other skills (api-docs owns docs/ internals)
- [ ] File-level ordering not specified (owned by language style skills)
```
